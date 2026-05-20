# NetworkMessageHandler.java

**Role:** Normalizes inbound messages to `NetworkMessage` objects (which carry both the payload `ByteBuf` and the remote `SocketAddress`). Also handles outbound `NetworkMessage` unwrapping for UDP. Bridges the gap between Netty's datagram/stream duality.
**Fits in:** Early inbound/outbound Netty pipeline stage (step 5). All downstream protocol decoders receive `NetworkMessage`, not raw `ByteBuf` or `DatagramPacket`.
**Read next:** [[StandardLoggingHandler.java.md]] (next in pipeline), [[AcknowledgementHandler.java.md]]

## Public API

### `channelRead` (lines 31-38)
- UDP: wraps `DatagramPacket` → `NetworkMessage(content, sender)`.
- TCP: wraps `ByteBuf` → `NetworkMessage(buffer, channel.remoteAddress())`.
- Fires wrapped message downstream.

### `write` (lines 41-54)
- If outbound message is `NetworkMessage`:
  - UDP: wraps back to `DatagramPacket(buf, recipient, sender)`.
  - TCP: unwraps to raw `ByteBuf`.
- Non-`NetworkMessage` outbound messages pass through unchanged.

## Gotchas / non-obvious

- **`ChannelDuplexHandler`** — handles both inbound and outbound directions.
- **Sender vs recipient in UDP**: inbound `DatagramPacket.sender()` → `NetworkMessage.remoteAddress`. Outbound: `NetworkMessage.remoteAddress` → `DatagramPacket.recipient`. The local address is `ctx.channel().localAddress()`.
- **Protocol decoders always receive `NetworkMessage`** — they do not need to know whether the transport is TCP or UDP.

## Line index

- 31-38 — `channelRead` (inbound normalization)
- 41-54 — `write` (outbound denormalization)
