# BaseFrameDecoder.java

**Role:** Abstract Netty `ByteToMessageDecoder` wrapper that adds automatic `TooLongFrameException` protection. Protocol-specific frame decoders extend this and implement `decode(ctx, channel, buf)` returning a complete frame or null.
**Fits in:** First handler in the protocol section of the pipeline (`addProtocolHandlers`). Converts the TCP byte stream into discrete frames that the `*ProtocolDecoder` can parse.
**Read next:** [[CharacterDelimiterFrameDecoder.java]] (pre-built delimiter-based variant), [[BasePipelineFactory.java]] (where frame decoders are added), [[ItsFrameDecoder.java]] (concrete example)

## Public API

- `BaseFrameDecoder()` — default max frame = `BaseProtocol.MAX_FRAME_LENGTH` (1024 bytes).
- `BaseFrameDecoder(int maxFrameLength)` — custom max.
- `decode(ChannelHandlerContext ctx, Channel channel, ByteBuf buf)` — abstract; called by Netty. Return a `ByteBuf` slice (complete frame) or null (more data needed). Do NOT release the input buffer.

## Key flows

### Frame size guard (lines 39-46)
If `decode()` returns null AND `in.readableBytes() > maxFrameLength`, throws `TooLongFrameException`, which Netty routes to `exceptionCaught`. This prevents unbounded buffer growth on malformed connections.

## Gotchas / non-obvious

- The `ctx` parameter is forwarded to the abstract `decode` for compatibility; some protocol implementations ignore it.
- `ByteToMessageDecoder` is NOT `@Sharable` — a new instance per channel. Every protocol creates a fresh frame decoder per connection.
- Use `buf.readRetainedSlice(n)` in implementations to return a slice without releasing the underlying buffer; Netty's ref-counting handles cleanup.

## Line index

- 30-33 — constructors (default and custom max)
- 39-46 — `decode` override (null guard + TooLong check)
- 48 — abstract `decode` signature
