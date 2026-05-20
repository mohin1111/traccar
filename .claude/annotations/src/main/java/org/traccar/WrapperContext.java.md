# WrapperContext.java

**Role:** `ChannelHandlerContext` decorator that overrides `fireChannelRead` and `write(msg, promise)` to ensure raw payloads are wrapped in `NetworkMessage` with the correct remote address before being propagated.
**Fits in:** Created by `WrapperInboundHandler.channelRead` and `WrapperOutboundHandler.write` for each message. Ensures non-decoder protocol handlers (frame decoders, string codecs) see raw payloads while the rest of the pipeline sees `NetworkMessage`.
**Read next:** [[WrapperInboundHandler.java]], [[WrapperOutboundHandler.java]], [[NetworkMessage.java]]

## Key flows

- `fireChannelRead(msg)` (line 99): if msg is not already a `NetworkMessage`, wraps it with stored `remoteAddress`.
- `write(msg, promise)` (line 187): same wrapping on the outbound path.

## Gotchas / non-obvious

- All other methods delegate unchanged to the wrapped `context`. Only the two message-propagating methods are overridden.
- `remoteAddress` is fixed at construction — if the channel's remote address changes (shouldn't happen for TCP), this wrapper would carry stale data.
