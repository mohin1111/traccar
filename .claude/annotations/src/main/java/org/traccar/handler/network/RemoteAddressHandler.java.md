# RemoteAddressHandler.java

**Role:** Stamps the device's IP address (`KEY_IP`) onto `Position` objects as they flow through the Netty pipeline. Runs after the protocol decoder produces positions.
**Fits in:** Netty pipeline, inbound direction. Located after protocol handlers (step 9) and before `ProcessingHandler`. `@Singleton @Sharable`.
**Read next:** [[StandardLoggingHandler.java.md]], [[NetworkMessageHandler.java.md]]

## Public API

### `channelRead` (lines 41-53)
- Only acts if `enabled = true` (config `PROCESSING_REMOTE_ADDRESS_ENABLE`).
- If the message is a `Position`: reads `channel.remoteAddress()` as `InetSocketAddress` and sets `KEY_IP` to the host address string.
- Always fires `ctx.fireChannelRead(msg)` — never consumes the message.

## Gotchas / non-obvious

- **`@Sharable`** — safe to share because it has no mutable per-channel state; `enabled` is final after construction.
- **UDP channels**: `remoteAddress()` on a UDP channel may return null for connectionless transports. The `hostAddress = remoteAddress != null ? ... : null` guard (line 45) handles this.
- **Disabled by default** — `PROCESSING_REMOTE_ADDRESS_ENABLE` is false unless explicitly set. This avoids unnecessary data in positions.

## Line index

- 36 — `enabled` flag from config
- 41 — `channelRead`
- 44-45 — extract host address with null guard
- 47-49 — set `KEY_IP` if message is Position
- 52 — `fireChannelRead` passthrough
