# BasePipelineFactory.java

**Role:** Abstract Netty `ChannelInitializer` that builds the full handler pipeline for every accepted connection. Defines the fixed infrastructure layers (idle timeout, logging, ACK, address stamping) and delegates the transport-level and protocol-level handlers to subclasses.
**Fits in:** Instantiated once per `TrackerServer`/`TrackerClient`, passed as the `childHandler`/`handler` to the Netty bootstrap. Called by Netty for each new channel. The most important structural file after `ProcessingHandler`.
**Read next:** [[ProcessingHandler.java]] (the last significant handler added), [[TrackerServer.java]] (instantiates this as anonymous subclass), [[WrapperInboundHandler.java]] / [[WrapperOutboundHandler.java]] (wrap non-decoder handlers to inject `NetworkMessage`)

## Public API

- `BasePipelineFactory(TrackerConnector connector, Config config, String protocol)` (line 46) — resolves per-protocol or global timeout.
- `addTransportHandlers(PipelineBuilder pipeline)` — abstract; subclass adds SSL, codec, etc.
- `addProtocolHandlers(PipelineBuilder pipeline)` — abstract; subclass adds frame decoder, string codec, `*ProtocolDecoder`.
- `getHandler(ChannelPipeline pipeline, Class<T> clazz)` (line 64) — static utility; walks the pipeline unwrapping `WrapperInboundHandler`/`WrapperOutboundHandler` to find a handler by type. Used by `BaseProtocol.sendDataCommand` to detect `StringEncoder`.

## Key flows

### `initChannel(Channel channel)` — the full pipeline (lines 85-121)

Pipeline assembly order (top = first handler for inbound, last = first handler for outbound):

1. **`addTransportHandlers`** (subclass) — SSL, HTTP codec, etc.
2. **`IdleStateHandler(timeout, 0, 0)`** (line 91) — fires `IdleStateEvent` on read-idle; `OpenChannelHandler` uses this to close the connection. Skipped for datagrams.
3. **`OpenChannelHandler`** (line 93) — registers channel with `TrackerConnector.channelGroup`; closes on idle.
4. **`NetworkForwarderHandler`** (line 96) — optional raw-byte mirror to another host (`server.forward` config).
5. **`NetworkMessageHandler`** (line 98) — wraps `DatagramPacket` into `NetworkMessage` for uniform downstream handling.
6. **`StandardLoggingHandler`** (line 99) — hex-dump of raw bytes (debug log level).
7. **`AcknowledgementHandler`** (line 101) — delays TCP ACK until the position is processed by `ProcessingHandler`; only for TCP when `server.delayAck=true`.
8. **`addProtocolHandlers`** (lines 105-116) — subclass handlers. `BaseProtocolDecoder`/`BaseProtocolEncoder` receive Guice injection; all other handlers are wrapped in `WrapperInboundHandler`/`WrapperOutboundHandler`.
9. **`RemoteAddressHandler`** (line 118) — stamps `Position.PROTOCOL_*` and stores remote IP.
10. **`ProcessingHandler`** (line 119) — receives decoded `Position` objects; runs the full enrichment/persistence chain.
11. **`MainEventHandler`** (line 120) — writes protocol-level ACK response back to device.

### Handler injection pattern (lines 105-116)
Only `BaseProtocolDecoder` and `BaseProtocolEncoder` receive full `injector.injectMembers()`. All other handlers in `addProtocolHandlers` are wrapped in `WrapperInboundHandler`/`WrapperOutboundHandler` so they receive `NetworkMessage` unwrapped to raw payload + remote address.

## Gotchas / non-obvious

- **`WrapperInboundHandler` is transparent** — it unwraps `NetworkMessage` before delegating. Frame decoders (e.g., `ItsFrameDecoder`) never see `NetworkMessage`; they see raw `ByteBuf`.
- **`getHandler` walks wrappers** (lines 67-72) — use this utility, not direct `pipeline.get(class)`, when you need to find a handler that might be wrapped.
- `ProcessingHandler` is `@Sharable` (added via `injector.getInstance`) — it is the same instance across all channels. All per-device state is keyed on `deviceId`.
- Timeout is per-protocol first (`<protocol>.timeout`), then falls back to global `server.timeout` (default 600s). Datagrams never get an idle handler.

## Line index

- 46-57 — constructor; resolves timeout
- 59 — `addTransportHandlers` abstract
- 61 — `addProtocolHandlers` abstract
- 64-77 — `getHandler` static utility (unwraps wrappers)
- 79-82 — `injectMembers` helper
- 85 — `initChannel` entry
- 88 — `addTransportHandlers` call
- 90-92 — `IdleStateHandler` (TCP only)
- 93 — `OpenChannelHandler`
- 94-97 — `NetworkForwarderHandler` (optional)
- 98 — `NetworkMessageHandler`
- 99 — `StandardLoggingHandler`
- 101-103 — `AcknowledgementHandler` (optional)
- 105-116 — `addProtocolHandlers` with injection/wrapping logic
- 118 — `RemoteAddressHandler`
- 119 — `ProcessingHandler` (THE handler)
- 120 — `MainEventHandler`
