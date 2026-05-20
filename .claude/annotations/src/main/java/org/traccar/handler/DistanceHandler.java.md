# DistanceHandler.java

**Role:** Computes `distance` (incremental leg) and `totalDistance` (cumulative odometer) for each position. Optionally filters out jitter by clamping improbable coordinate jumps.
**Fits in:** Extends `BasePositionHandler`. Step 7 in the pipeline (after `MapMatcherHandler`, before `FilterHandler`). `FilterHandler` uses `position.KEY_DISTANCE` for its `filterDistance` check, so this must run first.
**Read next:** [[FilterHandler.java.md]] (uses `KEY_DISTANCE`), [[BasePositionHandler.java.md]]

## Public API

### `onPosition(Position, Callback)` (lines 43-73)
- If the device already sent `KEY_DISTANCE` (device-side odometer), uses it as-is.
- Otherwise computes via `DistanceCalculator.distance(position, last)` (Haversine).
- Reads `last.KEY_TOTAL_DISTANCE` and accumulates.
- If `filter=true` and the computed distance is outside `[minError, maxError]`, snaps coordinates back to the last position and sets distance = 0 (treats the point as a jitter artifact).
- Sets `KEY_DISTANCE` and `KEY_TOTAL_DISTANCE` on the position.

## Key flows

### Jitter filter (lines 56-65)
Config keys `COORDINATES_FILTER`, `COORDINATES_MIN_ERROR`, `COORDINATES_MAX_ERROR` (global, not per-device). When enabled:
- Distance < `minError` → coordinate jump too small (duplicate/jitter) → snap to last.
- Distance > `maxError` → coordinate jump too large (GPS glitch) → snap to last.
- In both cases `distance = 0`, but the position is **not** filtered — it is kept with corrected coordinates.

## Gotchas / non-obvious

- **Runs before `FilterHandler`** — so the `KEY_DISTANCE` it sets is available to `filterDistance` rule in `FilterHandler`.
- **Coordinate snapping is silent** — the position is kept in the pipeline but its lat/lon are replaced with the last known good values. No flag is set to mark this.
- **`last == null`** (first position): `totalDistance = 0`, no jitter filtering (no prior reference).

## Line index

- 31 — `filter`, `minError`, `maxError` config
- 43 — `onPosition`
- 46-48 — use device-reported distance if present
- 51-55 — compute Haversine distance
- 56-65 — coordinate jitter filter
- 69-70 — set `KEY_DISTANCE`, `KEY_TOTAL_DISTANCE`
