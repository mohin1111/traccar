# SpeedLimitHandler.java

**Role:** Looks up the posted road speed limit at the position's coordinates and writes it to `KEY_SPEED_LIMIT`. `OverspeedEventHandler` uses this value as a speed threshold. Async.
**Fits in:** Extends `BasePositionHandler`. Step 11 in the pipeline (after `GeocoderHandler`, before `MotionHandler`). Only active if a `SpeedLimitProvider` is configured (default: `OverpassSpeedLimitProvider` using OpenStreetMap Overpass API).
**Read next:** [[MotionHandler.java.md]] (step after), [[events/OverspeedEventHandler.java.md]] (consumer of `KEY_SPEED_LIMIT`)

## Public API

### `onPosition(Position, Callback)` (lines 36-53)
- Calls `speedLimitProvider.getSpeedLimit(lat, lon, callback)` — async.
- On success: sets `KEY_SPEED_LIMIT` in knots.
- On failure: logs warning, continues without the attribute.

## Gotchas / non-obvious

- **Async** — same pattern as `GeocoderHandler`. Chain pauses until the provider resolves.
- **`KEY_SPEED_LIMIT` in knots** — the Overpass provider converts from km/h. `OverspeedEventHandler` compares directly with position speed (also in knots).
- **Optional** — not present in pipeline if no `SpeedLimitProvider` is bound. `OverspeedEventHandler` falls back to the configured per-device limit.

## Line index

- 36 — `onPosition`
- 38-50 — async provider call + success/failure callbacks
