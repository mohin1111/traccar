# OutdatedHandler.java

**Role:** Fills in missing GPS coordinates for "outdated" positions (positions the device sends without a valid GPS fix, containing only sensor/event data). Copies last known position's location fields into the current position.
**Fits in:** Extends `BasePositionHandler`. Step 2 in the pipeline (after `ComputedAttributesHandler.Early`, before `TimeHandler`). Must run early so subsequent handlers see valid coordinates.
**Read next:** [[TimeHandler.java.md]] (step after), [[ComputedAttributesHandler.java.md]] (step before)

## Public API

### Constant `GPS_EPOCH` (line 27)
`1980-01-06T00:00:00Z` in epoch millis — used as the default fixTime when there is no last position to copy from.

### `onPosition(Position, Callback)` (lines 37-57)
- Only acts if `position.getOutdated() = true`.
- Copies `fixTime`, `valid`, `lat`, `lon`, `altitude`, `speed`, `course`, `accuracy` from the last position.
- If no last position exists: sets `fixTime = GPS_EPOCH` (a clearly-wrong sentinel).
- Ensures `deviceTime` is set (falls back to `serverTime`).

## Gotchas / non-obvious

- **Outdated ≠ filtered** — an outdated position still flows through the pipeline. `FilterHandler.filterOutdated` (step 8) can drop it afterward if configured.
- **The position is mutable** — callers should not assume coordinates are the raw device values after this handler.
- **`GPS_EPOCH` sentinel** — if you see a position dated 1980-01-06, it means the device sent a position without GPS and had no prior fix to copy from.

## Line index

- 27 — `GPS_EPOCH` constant
- 37 — `onPosition`
- 38-54 — guard and coordinate copy
- 49 — fallback to GPS_EPOCH
- 52-54 — deviceTime fallback
