# Position.java

**Role:** The central telemetry entity. One row = one GPS fix received from a device. `@StorageName("tc_positions")` maps to `tc_positions`. This is the highest-volume table in the system.
**Fits in:** Created by protocol decoders (`BaseProtocolDecoder.decode()`), flows through `ProcessingHandler`'s 18-step chain, persisted by `DatabaseHandler`. Pushed to WebSocket clients via `ConnectionManager`. Linked from `tc_devices.positionid` (latest position FK).
**Read next:** [[Message.java]] (immediate parent), [[Device.java]] (holds FK positionId), [[Event.java]] (created from Position), [[Network.java]] (embedded field)

## Public API

### DB-mapped fields (all in `tc_positions`)
- `protocol` (line 161) — protocol name string, e.g. `"its"`, `"teltonika"`.
- `serverTime Date` (line 171) — when server received the packet; default = `new Date()` at construction.
- `deviceTime Date` (line 181) — time as reported by device.
- `fixTime Date` (line 191) — GPS fix time; used for all ordering/reporting.
- `valid boolean` (line 221) — GPS fix validity flag; `false` = estimated/approximate.
- `latitude double` (line 231) — validated [-90, 90]; throws `IllegalArgumentException` out of range.
- `longitude double` (line 244) — validated [-180, 180]; throws `IllegalArgumentException` out of range.
- `altitude double` (line 257) — meters above sea level.
- `speed double` (line 267) — **knots** (not km/h, not m/s). Convert via `UnitsConverter`.
- `course double` (line 277) — heading in degrees 0-360.
- `address String` (line 287) — reverse-geocoded street address; populated by `GeocoderHandler`.
- `accuracy double` (line 297) — GPS accuracy radius in meters.
- `network Network` (line 307) — embedded cell/WiFi network data (serialized as JSON or separate columns depending on DB).
- `geofenceIds List<Long>` (line 317) — geofence IDs this position falls within; set by `GeofenceHandler`.
- Inherited from `Message`: `deviceId long`, `type String` (type is `@QueryIgnore` + `@JsonIgnore` in Position).
- Inherited from `ExtendedModel`: `attributes AttributeMap` — the main bag for all device-specific sensor data.

### Not DB-mapped (`@QueryIgnore`)
- `outdated boolean` (line 207) — transient; `OutdatedHandler` sets true for duplicate-device-time positions. Also `@JsonIgnore`.
- `setTime(Date)` (line 202) — convenience setter; sets both `deviceTime` and `fixTime`.
- `getType()`/`setType()` overrides (lines 343-353) — suppressed with `@QueryIgnore` + `@JsonIgnore`; Position uses `type` field only via `Message`.

### Attribute key constants (lines 28-114)
~90 `KEY_*` and `PREFIX_*` constants. Critical ones:
- `KEY_ALARM` — comma-separated alarm string; `addAlarm()` appends.
- `KEY_IGNITION` — boolean; triggers ignition events.
- `KEY_ODOMETER` / `KEY_TOTAL_DISTANCE` — meters; used by `DistanceHandler` + reports.
- `KEY_HOURS` — milliseconds; engine hours accumulator.
- `KEY_DRIVER_UNIQUE_ID` — RFID/iButton; triggers driver assignment.
- `KEY_FUEL`, `KEY_FUEL_LEVEL`, `KEY_FUEL_CONSUMPTION` — fuel monitoring.
- `PREFIX_TEMP`, `PREFIX_ADC`, `PREFIX_IO`, `PREFIX_COUNT`, `PREFIX_IN`, `PREFIX_OUT` — numbered prefixes (start at 1): e.g., `temp1`, `adc2`, `io3`.

### Alarm constants (lines 116-153)
~35 `ALARM_*` string constants matching the `alarm` attribute values. Key AIS-140-relevant ones: `ALARM_SOS`, `ALARM_TAMPERING`, `ALARM_POWER_CUT`, `ALARM_GPS_ANTENNA_CUT`, `ALARM_OVERSPEED`.

## Key flows

### Protocol decoder → Position construction
```
new Position("protocolName")
→ setDeviceId(session.getDeviceId())
→ setTime(date)   // sets deviceTime + fixTime
→ setLatitude / setLongitude / setSpeed / setCourse / setValid
→ set(KEY_IGNITION, true)
→ set(KEY_ALARM, ALARM_SOS)
→ setNetwork(new Network(CellTower.from(mcc, mnc, lac, cid)))
→ return position
```

### `addAlarm(String)` (line 331)
Appends alarm strings comma-delimited; allows multiple alarms per position packet (common in AIS-140 devices that report geofence + overspeed in one message).

## Gotchas / non-obvious

- **Speed is in knots.** Every UI/report must convert. `UnitsConverter.knotsFromKph()` etc. in `helper/`.
- **Three timestamps.** `fixTime` = GPS fix time (use for reporting/ordering), `deviceTime` = device clock (may drift), `serverTime` = arrival time (used for latency metrics). They can differ significantly for buffered/offline positions.
- `geofenceIds` is populated **after** `GeofenceHandler` runs in `ProcessingHandler` — not set by protocol decoders.
- `network` field is **not** in the model hierarchy `attributes` bag — it is a top-level field that gets serialized to its own column (or JSON column depending on DB schema version).

## Line index

- 25 — `@StorageName("tc_positions")`
- 26 — `class Position extends Message`
- 28-114 — all KEY_* and PREFIX_* constants
- 116-153 — ALARM_* constants
- 155-158 — constructors
- 161-168 — protocol field
- 171-178 — serverTime
- 181-188 — deviceTime
- 191-205 — fixTime + setTime convenience
- 207-219 — outdated (transient)
- 221-229 — valid
- 231-254 — latitude (with range check)
- 244-255 — longitude (with range check)
- 257-265 — altitude
- 267-275 — speed (knots!)
- 277-285 — course
- 287-295 — address
- 297-305 — accuracy
- 307-314 — network
- 317-329 — geofenceIds
- 331-339 — addAlarm
- 341-354 — type override (@QueryIgnore + @JsonIgnore)
