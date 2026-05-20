# Geofence.java

**Role:** Represents a geographic boundary (circle, polygon, or polyline). `@StorageName("tc_geofences")` → `tc_geofences`. `GeofenceHandler` tests every incoming position against all geofences visible to the device's users, firing `TYPE_GEOFENCE_ENTER`/`EXIT` events.
**Fits in:** Extends `ExtendedModel` (has attributes for `floor`, `ceiling`, `polylineDistance`). Implements `Schedulable` (can be time-restricted via a calendar). Linked to devices via `tc_device_geofence`; linked to users via `tc_user_geofence`.
**Read next:** [[ExtendedModel.java]] (parent), [[Schedulable.java]], [[Event.java]] (geofence events), [[Calendar.java]] (time-gating)

## Public API

### DB-mapped fields
- `calendarId long` (line 33) — FK to `tc_calendars`; 0 = always active.
- `name String` (line 43) — display name.
- `description String` (line 53) — optional free text.
- `area String` (line 63) — WKT-like geometry string: `"CIRCLE (lat lon, radius)"`, `"POLYGON ((lat lon, ...))"`, or `"LINESTRING (lat lon, ...)"`. **This is the persisted representation.**
- Inherited: `attributes AttributeMap` — keys used: `floor` (m), `ceiling` (m), `polylineDistance` (m, default 25).

### Not DB-mapped (`@QueryIgnore @JsonIgnore`)
- `geometry GeofenceGeometry` (line 74) — lazily parsed from `area` on first `getGeometry()` call. Cached; invalidated by `setArea()`.

### Key methods
- `getGeometry()` (line 78) — lazy-parse `area` into `GeofenceCircle`, `GeofencePolygon`, or `GeofencePolyline` (the last uses `attributes.polylineDistance`). Throws `RuntimeException` on bad WKT.
- `setGeometry(GeofenceGeometry)` (line 99) — converts geometry back to WKT via `toWkt()`, calls `setArea()`.
- `containsPosition(Position)` (line 103) — **the hot path**. Checks altitude floor/ceiling first, then delegates to `geometry.containsPoint(lat, lon)`.

## Key flows

### `GeofenceHandler.onPosition()`
For each incoming position, iterates all cached geofences visible to the device's users, calls `geofence.containsPosition(position)` for each, then compares result to prior containment state to generate ENTER/EXIT events.

## Gotchas / non-obvious

- `setArea()` (line 69) nulls out `geometry` cache — any subsequent `containsPosition()` call re-parses. This is intentional but means editing `area` via the API causes the next position evaluation to re-parse.
- `polylineDistance` attribute default is 25 metres (line 86) — corridor width on each side of the linestring.
- Floor/ceiling of `0.0` means "no check" (lines 105-111) — altitude filtering only applies when explicitly set to non-zero.

## Line index

- 28 — `@StorageName("tc_geofences")`
- 29 — `class Geofence extends ExtendedModel implements Schedulable`
- 33-41 — calendarId
- 43-51 — name
- 53-61 — description
- 63-72 — area (WKT string, invalidates geometry cache)
- 74-101 — geometry (lazy parse, @QueryIgnore @JsonIgnore)
- 103-115 — containsPosition (altitude check + geometry test)
