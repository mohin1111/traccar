# GeocoderHandler.java

**Role:** Reverse-geocodes the position's lat/lon to a human-readable address string and sets `position.address`. Async — the callback fires from the geocoder's thread pool.
**Fits in:** Extends `BasePositionHandler`. Step 10 in the pipeline (after `GeofenceHandler`, before `SpeedLimitHandler`). The `Geocoder` implementation is injected (22+ providers supported; see `geocoder/` package).
**Read next:** [[GeofenceHandler.java.md]] (step before), [[SpeedLimitHandler.java.md]] (step after), [[BasePositionHandler.java.md]]

## Public API

### `onPosition(Position, Callback)` (lines 43-73)
- If `ignorePositions = true` (config `GEOCODER_IGNORE_POSITIONS`): skip and continue.
- If `reuseDistance > 0`: reuse the last position's address if the incremental distance is within `reuseDistance` meters (avoids API calls for stationary devices).
- Otherwise: calls `geocoder.getAddress(lat, lon, callback)` — async. On success or failure, calls `callback.processed(false)` from the geocoder's thread.

## Gotchas / non-obvious

- **Async callback** — the position chain pauses here until the geocoder resolves. Under load this is the most common pipeline latency source.
- **`reuseDistance` optimization** — depends on `KEY_DISTANCE` being set by `DistanceHandler`. Ensure pipeline order is maintained.
- **Failures are soft** — geocoding failure logs a warning but does not drop the position. Address remains null.
- **Not `@Inject`** — the constructor takes a concrete `Geocoder`; Guice injects via `MainModule` conditional binding (only wired if a geocoder provider is configured).

## Line index

- 33 — `ignorePositions` flag
- 34 — `reuseDistance` meters
- 43 — `onPosition`
- 44-53 — `reuseDistance` shortcut
- 55-68 — async geocoder call + callbacks
