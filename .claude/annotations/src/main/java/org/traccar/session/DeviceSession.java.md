# DeviceSession.java

**Role:** Per-device connection context for the duration of a TCP/UDP session. Holds device identity (ID, uniqueId, model), the Netty `Channel` for sending commands, and a local key-value store for protocol-specific state (e.g., timezone offsets).
**Fits in:** Created and stored by [[ConnectionManager.java]]; retrieved by protocol decoders via `BaseProtocolDecoder.getDeviceSession(...)`. Protocol decoders use it to look up the device ID without hitting the DB on every message.
**Read next:** [[ConnectionManager.java]] (creates and manages DeviceSession), `BaseProtocolDecoder.java` (calls `getDeviceSession`)

## Public API

### Constructor (lines 39-47)
`DeviceSession(deviceId, uniqueId, model, protocol, channel, remoteAddress)` — all fields are final except `lastUpdate`.

### Identity accessors (lines 49-67)
- `getDeviceId()` → `long` — the DB row ID; used for all storage operations
- `getUniqueId()` → `String` — IMEI or other device identifier as registered
- `getModel()` → `String` — device model string (for model-specific protocol quirks)
- `getChannel()` → `Channel` — the Netty channel; used for sending commands

### Session management (lines 49-55, 73-75)
- `getLastUpdate()` / `setLastUpdate()` — epoch ms; updated on every message to prevent session timeout
- `getConnectionKey()` → `ConnectionKey` — derives from channel + remoteAddress; used for map lookup

### Command sending (lines 77-83)
- `supportsLiveCommands()` — returns `false` if channel has an `HttpRequestDecoder` (i.e., it's an HTTP long-poll connection, not a persistent TCP socket). HTTP connections cannot receive async push commands.
- `sendCommand(Command)` — delegates to `protocol.sendDataCommand(channel, remoteAddress, command)`

### Local state store (lines 85-104)
`Map<String, Object> locals` — protocol decoders store per-session state here (e.g., timezone, accumulated odometer, sequence numbers). `KEY_TIMEZONE = "timezone"` is the only defined constant.

Methods: `contains(key)`, `set(key, value)` (removes key if value is null), `get(key)` (unchecked cast).

## Gotchas / non-obvious

- **`lastUpdate` is `volatile`** — accessed from both the Netty I/O thread and the `sweepIdleSessions` scheduled task without synchronization. Volatile is sufficient for a single `long` write.
- **`locals` is not thread-safe** — `HashMap` is used. All protocol processing for a given device is single-threaded (Netty per-channel ordering), so this is safe as long as locals are only accessed from the pipeline.
- **`supportsLiveCommands` HTTP check** — if a device communicates via HTTP (some protocols do periodic HTTP POSTs rather than persistent TCP), `HttpRequestDecoder` will be in the pipeline. Such sessions can send data but cannot receive real-time commands.
- `sendCommand` throws no checked exception but may fail silently if the channel is disconnected — callers should check channel state before calling.

## Line index

- 28-47 — fields + constructor
- 49-55 — `lastUpdate` get/set
- 57-67 — `deviceId`, `uniqueId`, `model`, `channel` accessors
- 73-75 — `getConnectionKey`
- 77-83 — `supportsLiveCommands`, `sendCommand`
- 85-104 — local state store (`KEY_TIMEZONE`, `contains`, `set`, `get`)
