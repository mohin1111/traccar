# BaseProtocolDecoder.java

**Role:** Abstract base for all protocol decoders. Provides the `getDeviceSession()` contract (resolves IMEI → DB device ID, creates `DeviceSession`), speed/timezone helpers, and the `onMessageEvent` hook that marks devices online and flushes queued commands.
**Fits in:** Extends `ExtendedObjectDecoder`. All 265 protocol-specific decoders extend this class and implement `decode(Channel, SocketAddress, Object)`. Receives Guice field injection for `CacheManager`, `ConnectionManager`, `StatisticsManager`, etc.
**Read next:** [[ExtendedObjectDecoder.java]] (handles `channelRead`, calls `decode`), [[BaseProtocol.java]] (protocol back-ref), [[ProcessingHandler.java]] (downstream consumer of decoded `Position`)

## Public API

- `getDeviceSession(Channel channel, SocketAddress remoteAddress, String... uniqueIds)` (line 133) — THE most important method. Resolves one or more candidate identifiers (IMEI, serial number, etc.) to a `DeviceSession`. Returns null if device is unknown and auto-registration is disabled. Subclasses call this once per message and exit early if it returns null.
- `getLastLocation(Position position, Date deviceTime)` (line 149) — populates a `Position` with the device's last known location and marks it `outdated=true`. Used when a protocol sends non-position messages (heartbeats, events) that should create a position record at the last known location.
- `getProtocolName()` (line 97) — returns the protocol name string (e.g., `"its"`).
- `getServer(Channel channel, char delimiter)` (line 101) — returns `<protocol>.server` config or the local bind address, formatted with `delimiter`.
- `convertSpeed(double value, String defaultUnits)` (line 110) — converts to knots from kph/mps/mph per `<protocol>.speed` config.
- `getTimeZone(long deviceId)` / `getTimeZone(long deviceId, String defaultTimeZone)` (lines 119-131) — looks up per-device timezone attribute; falls back to default.
- `sendQueuedCommands(Channel channel, SocketAddress remoteAddress, long deviceId)` (line 186) — called from `onMessageEvent`; flushes any pending commands for the device.

## Key flows

### `getDeviceSession` → `ConnectionManager.getDeviceSession` (line 135)
`ConnectionManager` checks if any of the `uniqueIds` matches a known device. If found, it creates or returns the existing `DeviceSession`, links the channel to the device, and marks the device online. If unknown and `database.registerUnknown=true`, a new device is auto-created. Returns null otherwise.

### `onMessageEvent` (lines 159-183)
Called by `ExtendedObjectDecoder.channelRead` after `decode()` returns. Regardless of whether decode returned a `Position`:
1. Increments `statisticsManager` message counter.
2. Extracts device IDs from the decoded result (`Position`, `Collection<Position>`, or null).
3. If no device ID found: calls `getDeviceSession` for the session device.
4. For each device ID: calls `connectionManager.updateDevice(deviceId, STATUS_ONLINE, now)` and `sendQueuedCommands()`.

This is why heartbeats (decode returning null) still keep devices online.

### Empty message handling (lines 193-203)
If `decode()` returns null AND `database.saveEmpty=true`, creates a position at the last known location and returns it — useful for heartbeat-only protocols.

## Gotchas / non-obvious

- **Always check `getDeviceSession() != null`** before creating a `Position`. Returning null from `decode()` is correct when the device is not registered. The standard pattern:
  ```java
  DeviceSession deviceSession = getDeviceSession(channel, remoteAddress, imei);
  if (deviceSession == null) return null;
  ```
- **Multiple `uniqueIds`** — pass all candidates (e.g., IMEI + serial number). `ConnectionManager` checks each in order.
- **`convertSpeed` returns knots** — Traccar's `Position` stores speed in knots. Always call `convertSpeed` or `UnitsConverter.knotsFromKph()` before setting position speed.
- **Timezone lookup** (line 122) — reads `decoder.timezone` attribute from device hierarchy (device → group → server). Always UTC if not set.
- **`@Inject` methods are called by Guice after construction** — the decoder constructor cannot use injected fields. All decoder logic that needs injection should be in `decode()` or `init()`.

## Line index

- 42-55 — field declarations (all Guice-injected)
- 56-58 — constructor (stores protocol ref)
- 64-86 — `@Inject` setter methods (`setCacheManager`, `setConnectionManager`, etc.)
- 97-99 — `getProtocolName`
- 101-108 — `getServer` (local address or config override)
- 110-117 — `convertSpeed` (kph/mps/mph → knots)
- 119-131 — `getTimeZone`
- 133-139 — `getDeviceSession` (delegates to `ConnectionManager`)
- 149-156 — `getLastLocation` (outdated position)
- 159-183 — `onMessageEvent` (device online + queued commands)
- 186-190 — `sendQueuedCommands`
- 193-203 — `handleEmptyMessage` (save heartbeat as empty position)
