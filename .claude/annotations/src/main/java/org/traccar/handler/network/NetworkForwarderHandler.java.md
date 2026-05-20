# NetworkForwarderHandler.java

**Role:** Copies raw inbound bytes from a device to a second network destination (raw TCP/UDP mirror). Enables passive protocol monitoring or forwarding to a secondary server without affecting the main pipeline.
**Fits in:** Early inbound stage of the Netty pipeline (step 4, before `NetworkMessageHandler`). Uses `@Inject` field injection for `NetworkForwarder`.
**Read next:** [[NetworkMessageHandler.java.md]] (next in pipeline), [[MainEventHandler.java.md]]

## Public API

### `channelRead` (lines 45-62)
- Determines if UDP or TCP.
- Extracts `remoteAddress` and `ByteBuf` data.
- Copies bytes to a `byte[]` without consuming the buffer (`getBytes` vs `readBytes`).
- Calls `networkForwarder.forward(remoteAddress, port, datagram, data)`.
- Passes the original message downstream (`super.channelRead`).

### `channelInactive` (lines 64-70)
- On TCP disconnect: notifies `networkForwarder.disconnect(remoteAddress)`.

## Gotchas / non-obvious

- **Non-consuming read** — uses `buffer.getBytes(buffer.readerIndex(), data)` not `readBytes`. The buffer is not consumed, so downstream handlers still see the full original message.
- **`@Inject` on `setNetworkForwarder`** — field injection on a setter method (unusual Guice pattern). The `NetworkForwarder` is optional; if not configured, `networkForwarder` remains null and `forward()` would NPE. In practice, this handler is only added to the pipeline when forwarding is configured.
- **Port** is the local server listen port (not the forwarding destination port). The forwarder maps local port → forwarding destination in its own config.

## Line index

- 35-41 — constructor: store local port + null forwarder
- 40 — `@Inject setNetworkForwarder`
- 45-62 — `channelRead`: copy bytes + forward
- 58 — `buffer.getBytes` (non-consuming)
- 64-70 — `channelInactive`: disconnect notification
