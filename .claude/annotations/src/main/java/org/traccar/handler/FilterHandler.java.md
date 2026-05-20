# FilterHandler.java

**Role:** Quality gate for the position pipeline. Evaluates up to 13 independent filter rules and drops the position if any pass. The only handler in the chain that passes `callback.processed(true)`.
**Fits in:** Extends `BasePositionHandler`. Step 8 in the pipeline, after `DistanceHandler` (which sets `KEY_DISTANCE` needed by `filterDistance`) and before `GeofenceHandler`. All filter predicates read config via `AttributeUtil.lookup` (device → group → server cascade).
**Read next:** [[DistanceHandler.java.md]] (dependency), [[GeofenceHandler.java.md]] (next step), [[BasePositionHandler.java.md]]

## Public API

### `filter(Position)` (lines 166-230) — protected, overridable
The core logic. Returns `true` if the position should be dropped. Each filter appends its name to `filterTypes`; if non-empty, the position is dropped and a `LOGGER.info` lists which filters triggered.

### `onPosition(Position, Callback)` (lines 232-235)
Thin wrapper: calls `filter(position)` and passes the result to `callback.processed(...)`.

## Filter rules

| Method | Config key | Logic |
|---|---|---|
| `filterInvalid` | `FILTER_INVALID` | Drops if `valid=false` or coords out of WGS84 range |
| `filterZero` | `FILTER_ZERO` | Drops if lat=0, lon=0 |
| `filterDuplicate` | `FILTER_DUPLICATE` | Drops if `fixTime` equals last, unless new attributes present |
| `filterOutdated` | `FILTER_OUTDATED` | Drops if `position.outdated = true` |
| `filterFuture` | `FILTER_FUTURE` | Drops if `fixTime` is more than N seconds in the future |
| `filterPast` | `FILTER_PAST` | Drops if `fixTime` is more than N seconds in the past |
| `filterAccuracy` | `FILTER_ACCURACY` | Drops if GPS accuracy > threshold meters |
| `filterApproximate` | `FILTER_APPROXIMATE` | Drops cell-tower-geolocated positions (`KEY_APPROXIMATE`) |
| `filterStatic` | `FILTER_STATIC` | Drops if speed == 0 |
| `filterDistance` | `FILTER_DISTANCE` | Drops if incremental distance < threshold meters |
| `filterMaxSpeed` | `FILTER_MAX_SPEED` | Drops if implied speed > threshold knots (GPS jump guard) |
| `filterMinPeriod` | `FILTER_MIN_PERIOD` | Drops if time since last < threshold seconds |
| `filterDailyLimit` | `FILTER_DAILY_LIMIT` | Drops if daily message count exceeded, unless interval passes |
| Calendar check | device calendar | Drops if device's assigned calendar does not cover `fixTime` |

### Skip conditions (override filters)
- `skipLimit` (lines 142-147): if `serverTime` gap since last exceeds `FILTER_SKIP_LIMIT`, always passes through (even if other filters trigger). Prevents perpetual silence after a gap.
- `skipAttributes` (lines 150-163): if any attribute listed in `FILTER_SKIP_ATTRIBUTES` is present, the position bypasses `Duplicate`, `Static`, and `Distance` filters. Allows events (alarm, ignition) to always pass.

## Gotchas / non-obvious

- **Filter predicates are checked even when disabled** — each method reads its config key and returns `false` if the key is unset. No short-circuit at `filter()`.
- **`filterDuplicate` has a nuance**: a position with the same `fixTime` is NOT dropped if it carries a new attribute key not present in the last position (line 61-64). This allows event-enriched positions to pass through.
- **MaxSpeed uses computed `KEY_DISTANCE`** from `DistanceHandler`, not raw Haversine — ensure `DistanceHandler` runs first.
- **`statisticsManager`** is injected but only used by `filterDailyLimit` (line 132).

## Line index

- 46-51 — `filterInvalid`
- 53-56 — `filterZero`
- 58-68 — `filterDuplicate` (with new-attribute escape)
- 71-74 — `filterOutdated`
- 76-79 — `filterFuture`
- 82-85 — `filterPast`
- 87-90 — `filterAccuracy`
- 92-95 — `filterApproximate`
- 97-100 — `filterStatic`
- 102-108 — `filterDistance`
- 110-118 — `filterMaxSpeed`
- 120-127 — `filterMinPeriod`
- 129-140 — `filterDailyLimit`
- 142-147 — `skipLimit` (gap override)
- 150-163 — `skipAttributes` (event-pass override)
- 166-230 — `filter()` orchestrator
- 232-235 — `onPosition`
