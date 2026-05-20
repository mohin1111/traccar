# MapMatcherHandler.java

**Role:** Snaps the raw GPS coordinate to the nearest road using a `MapMatcher` implementation (typically GraphHopper via `mapmatcher/`). Replaces position lat/lon with the snapped point. Async.
**Fits in:** Extends `BasePositionHandler`. Step 6 in the pipeline (after `HemisphereHandler`, before `DistanceHandler`). Snapping runs before distance computation so that odometer uses road-network distances.
**Read next:** [[DistanceHandler.java.md]] (step after), [[HemisphereHandler.java.md]] (step before)

## Public API

### `onPosition(Position, Callback)` (lines 34-50)
- Calls `mapMatcher.getPoint(lat, lon, callback)` — async.
- On success: sets new lat/lon on the position.
- On failure: logs a warning, continues with original coordinates.

## Gotchas / non-obvious

- **Optional** — `MapMatcherHandler` is only added to the pipeline if a `MapMatcher` is configured (`MainModule` conditional). When not configured, this step is absent.
- **Replaces coordinates** — downstream handlers (including `DistanceHandler`) see snapped coordinates. If snapping is wrong, all subsequent geo-computations drift.
- **No `@Inject`** — constructed manually in `ProcessingHandler` with the injected `MapMatcher`.

## Line index

- 34 — `onPosition`
- 35-48 — async map-matcher call + success/failure callbacks
