# WrapperInboundHandler.java

**Role:** Wraps a `ChannelInboundHandler` so it receives raw payloads (not `NetworkMessage`) while the surrounding pipeline passes `NetworkMessage`. Unwraps `NetworkMessage` in `channelRead` before delegating to the wrapped handler.
**Fits in:** Created by `BasePipelineFactory.initChannel` for all non-`BaseProtocolDecoder` handlers in `addProtocolHandlers`. This is what lets frame decoders and string codecs work in a pipeline that uses `NetworkMessage` as its universal envelope.
**Read next:** [[WrapperContext.java]], [[WrapperOutboundHandler.java]], [[BasePipelineFactory.java]]

## Key flows

### `channelRead` (lines 54-60)
```java
if (msg instanceof NetworkMessage nm) {
    handler.channelRead(new WrapperContext(ctx, nm.getRemoteAddress()), nm.getMessage());
} else {
    handler.channelRead(ctx, msg);
}
```
Passes `nm.getMessage()` (raw `ByteBuf`/`String`) to the wrapped handler inside a `WrapperContext` that re-wraps any outbound messages.

## Gotchas / non-obvious

- `getWrappedHandler()` (line 25) is used by `BasePipelineFactory.getHandler()` to find handlers by type through the wrapper.
- All lifecycle callbacks (`channelActive`, `handlerAdded`, etc.) delegate without modification.
