# mplex

> This repo contains the spec of mplex, the friendly Stream Multiplexer (that works in 3 languages!)

Mplex is a Stream Multiplexer protocol used by js-ipfs and go-ipfs in their implementations. The origins of this protocol are based in [multiplex](https://github.com/maxogden/multiplex), the JavaScript only Stream Multiplexer. After many battle field tests, we felt the need to improve and fix some of its bugs and mechanics, resulting on this new version used by libp2p.

This document will attempt to define a specification for the wire protocol and algorithm used in both implementations.

Mplex is a very simple protocol that does not provide many features offered by other stream multiplexers. Notably, mplex does not provide backpressure at the protocol level.

Implementations in:

- [JavaScript](https://github.com/libp2p/js-libp2p-mplex)
- [Go](https://github.com/libp2p/go-mplex)
- [Rust](https://github.com/libp2p/rust-libp2p/tree/master/multiplex-rs)

## Message format

Every communication in mplex consists of a header, and a length prefixed data segment.

The header is an unsigned base128 varint, as defined in the [protocol buffers spec](https://developers.google.com/protocol-buffers/docs/encoding#varints). The lower three bits are the message flags, and the rest of the bits (shifted down by three bits) are the stream ID this message pertains to:

```
header = readUvarint()
flag = header & 0x07
id = header >> 3
```

### Flag Values

```
| NewStream        | 0 |
| MessageReceiver  | 1 |
| MessageInitiator | 2 |
| CloseReceiver    | 3 |
| CloseInitiator   | 4 |
| ResetReceiver    | 5 |
| ResetInitiator   | 6 |
```

The data segment is length prefixed by another unsigned varint. This results in one message looking like:

```
| header  | length  | data           |
| uvarint | uvarint | 'length' bytes |
```

## Protocol

Mplex operates over a reliable ordered pipe between two peers, such as a TCP socket, or a unix pipe.

### Opening a new stream

To open a new stream, first allocate a new stream ID. Then, send a message with the flag set to `NewStream`, the ID set to the newly allocated stream ID, and the data of the message set to the name of the stream.

Stream names are purely for interfaces and are not otherwise considered by the protocol. An empty string may also be used for the stream name, and they may also be repeated (using the same stream name for every stream is valid). Reusing a stream ID after closing a stream may result in undefined behaviour.

The party that opens a stream is called the stream initiator. This is used to identify whether the message comes from a channel opened locally or remotely. Thus, the stream initiator always uses even flags and stream receivers uses odd flags.

### Writing to a stream

To write data to a stream, one must send a message with the flag `MessageReceiver` (1) or `MessageInitiator` (2) (depending on whether or not the writer is the receiver or sender). The data field should contain the data you wish to write to the stream, up to 1MiB per message.

### Closing a stream

Mplex supports half-closed streams. Closing a stream closes it for writing and closes the remote end for reading but allows writing in the other direction.

To close a stream, send a message with a zero length body and a `CloseReceiver` (3) or `CloseInitiator` (4) flag (depending on whether or not the closer is the receiver or sender). Writing to a stream after it has been closed should result
in an error. Reading from a remote-closed stream should return all data send before closing the stream and then EOF thereafter.

### Resetting a stream

To immediately close a stream for both reading and writing, use reset. This should generally only be used on error; during normal operation, both sides should close instead.

To reset a stream, send a message with a zero length body and a `ResetReceiver` (5) or `ResetInitiator` (6) flag. Reset must not block and must immediately close both ends of the stream for both reading and writing. All current and future reads and writes must return errors (*not* EOF) and any data queued or in flight should be dropped.

## Implementation notes

If a stream is being actively written to, the reader must take care to keep up with inbound data. Due to the lack of back pressure at the protocol level, the implementation must handle slow readers by doing one or both of:

1. Blocking the entire connection until the offending stream is read.
2. Resetting the offending stream.

For example, the go-mplex implementation blocks for a short period of time and then resets the stream if necessary.
