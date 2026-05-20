# MainEventHandler.java

**Role:** The final inbound Netty channel handler. Manages connection lifecycle events — logs connects/disconnects, handles idle timeouts, surfaces exceptions, and notifies `ConnectionManager` of device disconnections.
**Fits in:** Last inbound stage of the Netty pipeline (after `ProcessingHandler`). `@Singleton @Sharable` — one instance shared across all protocol channels.
**Read next:** [[OpenChannelHandler.java.md]] (earlier in pipeline), [[NetworkMessageHandler.java.md]]

## Public API

### `channelActive` (lines 59-62)
Logs TCP connect. Skipped for UDP (`DatagramChannel`).

### `channelInactive` (lines 65-73)
- Logs disconnect.
- Closes the channel.
- Calls `connectionManager.deviceDisconnected(channel, supportsOffline)`.
- `supportsOffline = true` for TCP non-HTTP protocols not in `connectionlessProtocols` list. HTTP and UDP are treated as connectionless (no persistent state on disconnect).

### `exceptionCaught` (lines 75-81)
Unwraps the exception cause chain, logs, closes channel.

### `userEventTriggered` (lines 83-88)
Handles `IdleStateEvent` (from `IdleStateHandler` earlier in pipeline) — logs timeout, closes channel.

## Gotchas / non-obvious

- **`supportsOffline` affects device status** — when `supportsOffline=true`, `ConnectionManager` marks the device as offline. For UDP/HTTP devices this is not done (they are inherently connectionless).
- **`connectionlessProtocols`** config: protocols listed in `STATUS_IGNORE_OFFLINE` are treated as not supporting persistent connections even if TCP.
- **Exception chaining unwrap** (lines 77-79) — Netty wraps exceptions; this loop extracts the original cause for cleaner logging.

## Line index

- 47-52 — `connectionlessProtocols` set from config
- 59-62 — `channelActive` (TCP only)
- 65-73 — `channelInactive` + `deviceDisconnected`
- 70-72 — `supportsOffline` determination
- 75-81 — `exceptionCaught`
- 83-88 — `IdleStateEvent` timeout
