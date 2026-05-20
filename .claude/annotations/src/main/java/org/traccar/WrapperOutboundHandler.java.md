# WrapperOutboundHandler.java

**Role:** Outbound counterpart to `WrapperInboundHandler`. Wraps a `ChannelOutboundHandler` so it receives raw payloads in `write()` while the surrounding pipeline uses `NetworkMessage`.
**Fits in:** Created by `BasePipelineFactory.initChannel` for outbound handlers (e.g., `StringEncoder`) in the protocol handler section.
**Read next:** [[WrapperInboundHandler.java]], [[WrapperContext.java]], [[BasePipelineFactory.java]]

## Key flows

### `write` (lines 69-75)
```java
if (msg instanceof NetworkMessage nm) {
    handler.write(new WrapperContext(ctx, nm.getRemoteAddress()), nm.getMessage(), promise);
} else {
    handler.write(ctx, msg, promise);
}
```
Unwraps `NetworkMessage` → raw message for the wrapped encoder handler; `WrapperContext` ensures any re-fire wraps back.

## Gotchas / non-obvious

- `getWrappedHandler()` (line 28) used by `BasePipelineFactory.getHandler()` for type-safe pipeline inspection.
