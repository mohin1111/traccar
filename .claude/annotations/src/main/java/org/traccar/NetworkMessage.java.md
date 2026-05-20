# NetworkMessage.java

**Role:** Simple value object pairing a payload `Object` with a `SocketAddress`. Carries the remote address through the pipeline so UDP-sourced messages and TCP messages have consistent routing for responses.
**Fits in:** Created by `NetworkMessageHandler` for UDP datagrams; passed through the pipeline between handlers. `WrapperInboundHandler` / `WrapperOutboundHandler` unwrap and re-wrap it for non-decoder handlers.
**Read next:** [[WrapperInboundHandler.java]], [[ExtendedObjectDecoder.java]] (unwraps it), [[BaseProtocol.java]] (creates it for command sending)

## Public API

- `NetworkMessage(Object message, SocketAddress remoteAddress)` — constructor.
- `getMessage()` — the payload (`ByteBuf`, `String`, `HttpRequest`, `Command`, `Position`, etc.)
- `getRemoteAddress()` — the peer address.

## Gotchas / non-obvious

- The `message` field is untyped — downstream handlers must cast. Frame decoders receive `ByteBuf` (unwrapped by `WrapperInboundHandler`); `ExtendedObjectDecoder` receives the unwrapped payload.
- For TCP, `remoteAddress` comes from `channel.remoteAddress()` (set at `NetworkMessageHandler`). For UDP, it comes from `DatagramPacket.sender()`.
