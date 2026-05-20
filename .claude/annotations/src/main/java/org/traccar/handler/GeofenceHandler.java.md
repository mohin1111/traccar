# GeofenceHandler.java

**Role:** Evaluates geofence membership for a position and sets `position.geofenceIds` to the list of geofences containing the point. This is what `GeofenceEventHandler` reads to detect entry/exit.
**Fits in:** Extends `BasePositionHandler`. Step 9 in the pipeline (after `FilterHandler`, before `GeocoderHandler`). Delegates geometry containment to `GeofenceUtil.getCurrentGeofences`.
**Read next:** [[GeocoderHandler.java.md]] (step after), [[events/GeofenceEventHandler.java.md]] (consumes geofenceIds), [[FilterHandler.java.md]] (step before)

## Public API

### `onPosition(Position, Callback)` (lines 35-42)
- Calls `GeofenceUtil.getCurrentGeofences(cacheManager, position)` — returns list of geofence IDs whose geometry contains the position's coordinates.
- If non-empty, sets `position.setGeofenceIds(geofenceIds)`.
- Always passes `callback.processed(false)`.

## Key flows

`GeofenceUtil.getCurrentGeofences` (in `helper/model/`) iterates geofences linked to the device (via `CacheManager`) and tests each geometry (circle, polygon, polyline) against the position. Only geofences the device is linked to are evaluated — not all server geofences.

## Gotchas / non-obvious

- **Only sets the list if non-empty** — if the device is outside all geofences, `position.getGeofenceIds()` is `null`, not an empty list. Consumers must null-check.
- **Geometry is evaluated in-memory** — no DB query per position; all geofence shapes are in `CacheManager`. Cache invalidation is handled by `CacheManager` on geofence update.
- **Only geofences linked to the device** (via `tc_device_geofence` permission rows) are evaluated. Adding a geofence without linking it to a device produces no events.

## Line index

- 35 — `onPosition`
- 37 — `GeofenceUtil.getCurrentGeofences` call
- 38-40 — set `geofenceIds` if non-empty
- 41 — `callback.processed(false)`
