# GeofenceEventHandler.java

**Role:** Compares geofence membership between current and last position (computed by `GeofenceHandler` in the pipeline) and emits `TYPE_GEOFENCE_ENTER` or `TYPE_GEOFENCE_EXIT` events for geofence boundary crossings.
**Fits in:** Extends `BaseEventHandler`. Runs in the event-detection phase. Depends on `position.geofenceIds` being populated by `GeofenceHandler`.
**Read next:** [[BaseEventHandler.java.md]], [[handler/GeofenceHandler.java.md]] (sets geofenceIds)

## Public API

### `onPosition(Position, Callback)` (lines 39-78)
- Guards: position must be latest.
- Builds `oldGeofences` set from last position's `geofenceIds`.
- Builds `newGeofences` set from current position's `geofenceIds`.
- `exited = oldGeofences - newGeofences` → fire `TYPE_GEOFENCE_EXIT` for each.
- `entered = newGeofences - oldGeofences` → fire `TYPE_GEOFENCE_ENTER` for each.
- Each event is gated by the geofence's calendar (if set): if the calendar does not cover the current `fixTime`, the event is skipped.

## Key flows

### Set arithmetic (lines 51-55)
- `newGeofences.removeAll(oldGeofences)` → entered geofences.
- `position.getGeofenceIds().forEach(oldGeofences::remove)` → exited geofences.

## Gotchas / non-obvious

- **Calendar gating** — both enter and exit events check the geofence calendar. This allows time-restricted geofences (e.g., "alert only during working hours").
- **Null safety** — if `lastPosition.geofenceIds` is null (device was outside all geofences), `oldGeofences` starts empty. Entry into the first geofence is correctly detected.
- **Multiple events per position** — if the device crosses N geofence boundaries in one position jump, N events fire.
- **`isLatest` guard** prevents duplicate events from historical replays.

## Line index

- 40 — `isLatest` guard
- 44-48 — build `oldGeofences` from last position
- 50-55 — compute entered and exited sets
- 57-67 — emit EXIT events (with calendar check)
- 68-77 — emit ENTER events (with calendar check)
