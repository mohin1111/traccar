# Device.java

**Role:** Represents a physical GPS tracker. `@StorageName("tc_devices")` → `tc_devices` table. One row per registered device. Holds the IMEI/identifier (`uniqueId`), live state (status, positionId), and motion/overspeed transient state (in-memory only, never persisted).
**Fits in:** Central entity. `Permission` links it to `User` via `tc_user_device`. `Position` references it via `deviceId`. `Group` can contain it via `groupId`. `DeviceResource` serves CRUD.
**Read next:** [[GroupedModel.java]] (parent — provides groupId), [[Disableable.java]] (interface), [[Schedulable.java]] (interface), [[Position.java]] (references deviceId), [[LinkedDevice.java]] (permission proxy)

## Public API

### DB-mapped fields (`tc_devices` columns)
- `calendarId long` (line 27) — FK to `tc_calendars`; used via `Schedulable`; 0 = no calendar.
- `name String` (line 39) — human display name.
- `uniqueId String` (line 49) — the device identifier (IMEI, serial, etc.); trimmed on set; **unique** in DB. This is what protocol decoders use to look up the device session.
- `phone String` (line 96) — optional phone number; used for SMS commands.
- `model String` (line 106) — device model description (free text).
- `contact String` (line 116) — contact info (free text).
- `category String` (line 126) — device category (maps to map icon in web UI: `truck`, `car`, `boat`, etc.).
- `disabled boolean` (line 136) — from `Disableable`; disables login/connection when true.
- `expirationTime Date` (line 148) — from `Disableable`; device expires at this time.
- Inherited from `GroupedModel`: `groupId long`.
- Inherited from `ExtendedModel`: `attributes AttributeMap` (JSON column).

### Computed / not DB-mapped (`@QueryIgnore`)
- `status String` (line 63) — `"online"` / `"offline"` / `"unknown"`; derived at runtime, default `STATUS_OFFLINE` when null (line 68).
- `lastUpdate Date` (line 74) — last position receive time; populated by `ConnectionManager`.
- `positionId long` (line 85) — FK to latest position in `tc_positions`; updated by `DatabaseHandler`.

### Transient motion/overspeed state (all `@QueryIgnore @JsonIgnore`)
These live **only in memory** in `CacheManager`; never written to DB or sent to API.
- `motionStreak boolean` (line 160) — motion start/stop streak tracking.
- `motionState boolean` (line 173) — currently moving?
- `motionPositionId long` (line 185) — positionId when motion state last changed.
- `motionTime Date` (line 198) — time of last motion state change.
- `motionDistance double` (line 211) — distance accumulated since motion start.
- `motionLatitude/Longitude double` (lines 224, 237) — position at motion start.
- `overspeedState boolean` (line 250) — currently overspeeding?
- `overspeedTime Date` (line 263) — time overspeed started.
- `overspeedGeofenceId long` (line 276) — geofence whose speed limit is being exceeded.

## Key flows

### Device lookup by IMEI
`BaseProtocolDecoder.getDeviceSession(channel, remoteAddress, identifiers...)` calls `CacheManager.findDeviceSession(uniqueId)` → finds the `Device` whose `uniqueId` matches the identifier sent by the tracker.

### Status lifecycle
`ConnectionManager` sets `status = STATUS_ONLINE` when a Netty channel connects and `STATUS_OFFLINE` when it closes. Pushed to WebSocket clients via `AsyncSocket`.

## Gotchas / non-obvious

- `setUniqueId()` trims the input (line 57) — important for some devices that pad with whitespace.
- `status`, `lastUpdate`, `positionId` are `@QueryIgnore` — they are **not** stored by the ORM. They are set programmatically at runtime. The DB columns for `lastupdate` and `positionid` exist in the schema but are managed by `DatabaseHandler`/`ConnectionManager` via direct SQL (not `updateObject`).
- The `motion*` and `overspeed*` fields are pure in-memory state; a server restart resets them. They are reconstructed from the last known position on startup via `PositionBatchWriter`.

## Line index

- 24 — `@StorageName("tc_devices")`
- 25 — `class Device extends GroupedModel implements Disableable, Schedulable`
- 27-37 — calendarId (Schedulable)
- 39-57 — name, uniqueId
- 59-82 — status constants + status field (@QueryIgnore)
- 74-82 — lastUpdate (@QueryIgnore)
- 85-95 — positionId (@QueryIgnore)
- 96-134 — phone, model, contact, category
- 136-157 — disabled, expirationTime (Disableable)
- 160-288 — all motion/overspeed transient fields (@QueryIgnore @JsonIgnore)
