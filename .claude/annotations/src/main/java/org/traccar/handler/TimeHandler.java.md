# TimeHandler.java

**Role:** Corrects two categories of timestamp issues: (1) GPS week-number rollover and (2) device/server time override. Step 3 in the pipeline.
**Fits in:** Extends `BasePositionHandler`. Step 3 (after `OutdatedHandler`, before `GeolocationHandler`). Timestamp corrections must happen before any time-based filtering in `FilterHandler`.
**Read next:** [[OutdatedHandler.java.md]] (step before), [[GeolocationHandler.java.md]] (step after)

## Public API

### Static constant `ROLLOVER_CYCLE` (line 31)
1024 weeks in millis (GPS rollover period). GPS epoch rolls over every 1024 weeks (≈ 19.7 years).

### `onPosition(Position, Callback)` (lines 50-54)
Calls `handleRollover` then `handleOverride`.

### `adjustRollover(long currentTime, Date time)` (lines 62-67) — public static
Used by `OutdatedHandler` too. Adds `ROLLOVER_CYCLE` to the position time until it falls within `ROLLOVER_THRESHOLD` of current time. Handles devices with GPS chips that have not received a rollover correction.

### `handleOverride(Position)` (lines 70-87)
If `TIME_OVERRIDE` is configured:
- `"serverTime"` → sets both `deviceTime` and `fixTime` to `serverTime`. Useful for devices with unreliable clocks.
- `"deviceTime"` (default) → copies `deviceTime` into `fixTime`.

Optionally scoped to specific protocols via `TIME_PROTOCOLS` config key.

## Gotchas / non-obvious

- **Rollover only fires when the timestamp is `> GPS_EPOCH`** (post-1980) and far in the past. Timestamps before GPS epoch are not adjusted.
- **`ROLLOVER_THRESHOLD = ROLLOVER_CYCLE - 90 days`** — if the position time is within 90 days of the rollover boundary, we assume the rollover occurred. Prevents over-correction for old cached data.
- **`TIME_PROTOCOLS` restricts override to listed protocols** — useful when only some device families have bad clocks.

## Line index

- 31 — `ROLLOVER_CYCLE` constant
- 32 — `ROLLOVER_THRESHOLD`
- 38-46 — constructor: read override and protocol list
- 50-54 — `onPosition` dispatcher
- 56-59 — `handleRollover`
- 62-67 — `adjustRollover` (public static, also used by OutdatedHandler)
- 70-87 — `handleOverride`
