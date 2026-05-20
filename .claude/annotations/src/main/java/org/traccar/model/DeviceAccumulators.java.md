# DeviceAccumulators.java

**Role:** DTO for updating a device's persisted odometer and engine-hours accumulators. Not a full entity (no `@StorageName`, no `extends BaseModel`) — used as the request body for `PUT /api/devices/{id}/accumulators`. `DatabaseStorage` has a dedicated update path for this.
**Fits in:** Used by `DeviceResource.updateAccumulators()`. Values are written directly into the device's latest `Position` attributes in `tc_positions`.
**Read next:** [[Device.java]], [[Position.java]] (KEY_TOTAL_DISTANCE, KEY_HOURS)

## Public API

- `deviceId long` (line 21) — identifies the target device.
- `totalDistance Double` (line 30) — new total distance in **meters** (nullable; null = no change).
- `hours Long` (line 40) — new engine hours in **milliseconds** (nullable; null = no change).

## Gotchas / non-obvious

- This is NOT an `ExtendedModel` — it has no `id`, no `attributes`, no `@StorageName`. It is a bare DTO.
- `totalDistance` and `hours` are nullable boxed types; callers must check null before applying.

## Line index

- 19 — `class DeviceAccumulators` (no inheritance, no @StorageName)
- 21-28 — deviceId
- 30-38 — totalDistance (meters, nullable)
- 40-48 — hours (milliseconds, nullable)
