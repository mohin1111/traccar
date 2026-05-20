# `handler/network/` — Netty Channel Pipeline Stages

Infrastructure-layer Netty handlers that form the protocol-agnostic portion of the channel pipeline. These run before and after the protocol-specific decoders in `BasePipelineFactory.initChannel`.

## File index

| File | One-liner | Annotation |
|---|---|---|
| `AcknowledgementHandler.java` | Holds outbound ACK until all decoded positions are pipeline-processed | [AcknowledgementHandler.java.md](AcknowledgementHandler.java.md) |
| `MainEventHandler.java` | Connection lifecycle: connect/disconnect/timeout/exception logging | [MainEventHandler.java.md](MainEventHandler.java.md) |
| `NetworkForwarderHandler.java` | Mirror raw bytes to secondary server (passive tap) | [NetworkForwarderHandler.java.md](NetworkForwarderHandler.java.md) |
| `NetworkMessageHandler.java` | Normalize UDP/TCP to `NetworkMessage`; denormalize on write | [NetworkMessageHandler.java.md](NetworkMessageHandler.java.md) |
| `OpenChannelHandler.java` | Register/deregister channel in `TrackerConnector.channelGroup` | [OpenChannelHandler.java.md](OpenChannelHandler.java.md) |
| `RemoteAddressHandler.java` | Stamp `KEY_IP` (device IP) onto decoded `Position` objects | [RemoteAddressHandler.java.md](RemoteAddressHandler.java.md) |
| `StandardLoggingHandler.java` | Hex/text log of all inbound/outbound bytes + live log streaming | [StandardLoggingHandler.java.md](StandardLoggingHandler.java.md) |

## Netty channel pipeline order

Built by `BasePipelineFactory.initChannel()` for every new protocol connection:

```
[inbound direction: device → server]

1. (Netty) IdleStateHandler              — close idle TCP connections
2. OpenChannelHandler                     — register channel in TrackerConnector group
3. NetworkForwarderHandler                — mirror raw bytes (optional)
4. NetworkMessageHandler                  — wrap to NetworkMessage (inbound)
5. StandardLoggingHandler                 — hex/text log (inbound)
6. AcknowledgementHandler                 — queue outbound ACK (outbound, positioned here)
7. [protocol-specific frame decoders]     — e.g., LineBasedFrameDecoder, LengthFieldFrameDecoder
8. [protocol-specific string codecs]      — e.g., StringDecoder/StringEncoder
9. [*ProtocolDecoder]                     — device-specific parser → produces Position objects
10. RemoteAddressHandler                  — stamp KEY_IP on Position
11. ProcessingHandler                     — full position pipeline (steps 1–19 + events)
12. MainEventHandler                      — connection lifecycle events (last inbound handler)

[outbound direction: server → device]

NetworkMessageHandler                     — unwrap NetworkMessage to DatagramPacket/ByteBuf
StandardLoggingHandler                    — hex/text log (outbound)
AcknowledgementHandler                    — release queued ACK when pipeline done
[protocol encoder]                        — format command for wire
```

## Key design patterns

### Dual-direction handlers
`NetworkMessageHandler`, `StandardLoggingHandler`, and `AcknowledgementHandler` are `ChannelDuplexHandler` — they handle both reads and writes. `OpenChannelHandler` uses `ChannelDuplexHandler` for lifecycle events.

### Singleton `@Sharable` handlers
`MainEventHandler` and `RemoteAddressHandler` are `@Singleton @Sharable` — one instance shared across all protocol channels of all protocols. They are stateless (or protected by thread-safe structures).

### Per-channel handlers
`StandardLoggingHandler`, `AcknowledgementHandler`, `OpenChannelHandler`, `NetworkForwarderHandler` are instantiated per-channel (per-connection). They may hold per-connection state (`logCount`, queue).

### `NetworkMessage` as the common message type
After `NetworkMessageHandler`, all inbound messages are `NetworkMessage(ByteBuf, SocketAddress)`. Protocol decoders receive this. After decoding, positions flow as `Position` objects. `RemoteAddressHandler` and `ProcessingHandler` only act on `Position` objects.

## Multi-file flows

### New TCP connection
1. Netty fires `channelActive` → `OpenChannelHandler` adds to group → `MainEventHandler` logs connect.
2. Device sends first frame → `NetworkForwarderHandler` copies bytes → `NetworkMessageHandler` wraps → `StandardLoggingHandler` logs → `AcknowledgementHandler` opens queue → protocol decoders run → `ProtocolDecoder` produces `Position` → `RemoteAddressHandler` stamps IP → `ProcessingHandler` runs pipeline → `MainEventHandler` at end.

### Protocol ACK sequencing (via AcknowledgementHandler)
1. `ProcessingHandler` writes `EventReceived` to channel → `AcknowledgementHandler` opens queue.
2. Decoder writes `EventDecoded({positions})` → handler tracks all pending.
3. Each position completes pipeline → `EventHandled(position)` written → removed from pending.
4. Protocol encoder tries to write ACK → held in queue.
5. All positions done → queue flushed → ACK reaches device.

### Raw byte mirroring
`NetworkForwarderHandler` calls `networkForwarder.forward(...)` with a non-consuming copy of the buffer. The original buffer still flows to `NetworkMessageHandler`. The forwarder opens its own TCP/UDP connection to the mirror destination.

## How to add a new pipeline stage

1. Create a class extending `ChannelInboundHandlerAdapter`, `ChannelOutboundHandlerAdapter`, or `ChannelDuplexHandler`.
2. If stateless: add `@Sharable` and bind as `@Singleton` in `MainModule.java`.
3. If per-channel: instantiate in `BasePipelineFactory.initChannel()` or in the protocol's `addTransportHandlers`/`addProtocolHandlers`.
4. Insert at the correct pipeline position relative to the existing handlers (see order above).
5. For inbound: override `channelRead`. Always call `ctx.fireChannelRead(msg)` or `super.channelRead(ctx, msg)` to pass the message along (unless intentionally consuming it).
6. For outbound: override `write`. Always call `super.write(ctx, msg, promise)` unless queuing.
