# ConnectionManager.java

**Role:** The device-connection registry and real-time update fan-out hub. Maintains three concurrent maps tracking active sessions, dispatches device/position/event updates to WebSocket listeners, handles automatic device registration, and sweeps idle sessions. Implements both `BroadcastInterface` (for multi-node clustering) and defines the `UpdateListener` interface (implemented by WebSocket sessions).
**Fits in:** `@Singleton`; injected into `BaseProtocolDecoder` (session lookup), `ProcessingHandler` (position/event publishing), `AsyncSocket` (listener registration), and `ScheduleManager` (idle sweep). The central bus connecting Netty pipeline → WebSocket clients.
**Read next:** [[DeviceSession.java]], [[session/cache/CacheManager.java]], `api/AsyncSocket.java` (implements `UpdateListener`)

## Public API

### Session lookup (lines 110-185)
- `getDeviceSession(deviceId)` → `DeviceSession` — fast lookup by DB device ID
- `getDeviceSession(protocol, channel, remoteAddress, uniqueIds...)` → `DeviceSession | null` — **the main entry point called by every protocol decoder**. Returns existing session or creates one by looking up the device in DB. Returns null for unknown devices (logged as WARN).

### Session lifecycle (lines 208-241)
- `deviceDisconnected(channel, supportsOffline)` — called on Netty `channelInactive`; removes sessions, optionally marks device OFFLINE
- `deviceUnknown(deviceId)` — marks device STATUS_UNKNOWN and removes session (called by idle sweep)
- `sweepIdleSessions()` — called by scheduler; expires sessions older than `STATUS_TIMEOUT` ms

### Device status updates (lines 243-282)
- `updateDevice(deviceId, status, time)` — updates status in DB + cache + notifies all WebSocket listeners for this device's users. Fires `Event.TYPE_DEVICE_ONLINE/OFFLINE/UNKNOWN` if status changed.

### Keepalive (lines 284-289)
- `sendKeepalive()` — calls `onKeepalive()` on every registered listener; used to keep WebSocket connections alive.

### WebSocket listener management (lines 378-403)
- `addListener(userId, UpdateListener)` — registers a WebSocket session; on first add for a userId, loads the user's device list from DB and populates `userDevices`/`deviceUsers` maps.
- `removeListener(userId, UpdateListener)` — deregisters; cleans up maps if no more listeners for that user.

### `UpdateListener` interface (lines 370-376)
```java
interface UpdateListener {
    void onKeepalive();
    void onUpdateDevice(Device device);
    void onUpdatePosition(Position position);
    void onUpdateEvent(Event event);
    void onUpdateLog(LogRecord record);
}
```
Implemented by `AsyncSocket` (WebSocket). Each callback serializes to JSON and pushes over the WebSocket.

### `BroadcastInterface` implementations (lines 293-347)
- `updateDevice(boolean local, Device)` — if local, fans out to broadcast bus; dispatches to all `UpdateListener`s for this device's users
- `updatePosition(boolean local, Position)` — same fan-out pattern
- `updateEvent(boolean local, userId, Event)` — fan-out per userId
- `invalidatePermission(...)` — on `User→Device` link: adds device to user's tracking set

### Log update (lines 348-368)
- `updateLog(LogRecord)` — enriches log record with uniqueId/deviceId from session maps; fans out to appropriate WebSocket listeners. Also delivers logs for unknown devices to all listeners if `WEB_SHOW_UNKNOWN_DEVICES` is enabled.

## Key flows

### Device identification on connection (lines 114-185)
1. Check `sessionsByEndpoint.get(connectionKey)` — if session exists and uniqueId matches, update `lastUpdate` and return cached session (fast path).
2. If uniqueIds provided but no cached session: `deviceLookupService.lookup(uniqueIds)` — DB lookup.
3. If not found and `DATABASE_REGISTER_UNKNOWN` is enabled and uniqueId matches regex: `addUnknownDevice` — auto-registers in DB.
4. If device found: remove any old session from the other maps, create new `DeviceSession`, store in both maps, call `cacheManager.addDevice`.
5. If not found: log WARN, store in `unknownByEndpoint` for logging purposes.

### Update fan-out (updatePosition, lines 310-322)
1. Called from `ProcessingHandler` after a position is saved to DB.
2. `deviceUsers.getOrDefault(position.getDeviceId(), emptySet())` — get all user IDs watching this device.
3. For each userId → get their `Set<UpdateListener>` → call `listener.onUpdatePosition(position)`.
4. Also forwards to `BroadcastService` (Redis/Multicast) for multi-node clusters.

### Idle session sweep (lines 100-108)
Called periodically by `ScheduleManager`. Compares `session.getLastUpdate()` against `deviceTimeout` cutoff (from `Keys.STATUS_TIMEOUT`). Sessions older than the cutoff are marked UNKNOWN via `deviceUnknown`.

## Gotchas / non-obvious

- **`listeners`, `userDevices`, `deviceUsers` are `HashMap` protected by `synchronized`** — the three update methods (`updateDevice`, `updatePosition`, `updateEvent`) and listener add/remove are all `synchronized`. The session maps (`sessionsByDeviceId`, `sessionsByEndpoint`) are `ConcurrentHashMap` — they can be read without the lock.
- **`userDevices` and `deviceUsers` are only populated when a WebSocket listener is registered** — if no client is connected for a user, their devices are not tracked in these maps. This is intentional — no fan-out cost when nobody is watching.
- **`invalidatePermission` only handles `User→Device` links** (line 340) — other permission types (User→Group, Group→Device) are not reflected here; those go through `CacheManager.invalidatePermission` instead.
- **Auto-registration regex** (line 147) — `DATABASE_REGISTER_UNKNOWN_REGEX` config key; defaults to `.*` (match all). Can be restricted to IMEI pattern etc.
- **`updateDevice(true, device)` → broadcast then local fan-out** (line 296-306) — "local=true" means this node generated the event; broadcast it to other nodes first, then deliver locally. "local=false" means received from another node; only deliver locally, and if it's an ONLINE status, remove the local session (device has connected to another node).

## Line index

- 58-98 — class fields + constructor
- 100-108 — `sweepIdleSessions`
- 110-113 — `getDeviceSession(deviceId)` fast lookup
- 114-185 — `getDeviceSession(...)` full lookup + session creation
- 187-206 — `addUnknownDevice`
- 208-224 — `deviceDisconnected`
- 226-240 — `deviceUnknown` + `removeDeviceSession`
- 243-282 — `updateDevice(deviceId, status, time)` — DB + cache + listeners
- 284-289 — `sendKeepalive`
- 293-307 — `updateDevice(boolean local, Device)` — BroadcastInterface impl
- 310-322 — `updatePosition` — BroadcastInterface impl
- 325-335 — `updateEvent` — BroadcastInterface impl
- 338-346 — `invalidatePermission` — BroadcastInterface impl
- 348-368 — `updateLog`
- 370-376 — `UpdateListener` interface
- 378-403 — `addListener` / `removeListener`
