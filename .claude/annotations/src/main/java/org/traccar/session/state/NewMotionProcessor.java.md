# NewMotionProcessor.java

**Role:** Stateless utility implementing the "new" trip-detection algorithm based on position history rather than duration timers. Uses distance from anchor point and average-speed analysis over the position window to determine trip start/stop.
**Fits in:** Called by `MotionHandler` when `Keys.REPORT_TRIP_NEW_LOGIC` is enabled. Works with [[NewMotionState.java]] and uses position history from `CacheManager.getPositions`.
**Read next:** [[NewMotionState.java]], [[MotionProcessor.java]] (classic algorithm), `handler/MotionHandler.java`

## Public API

### `updateState` (lines 31-97)
```java
public static void updateState(
    NewMotionState state, Position position,
    double minDistance, long minDuration, long stopGap)
```
- `minDistance` — meters a device must move from the stop anchor to confirm a trip
- `minDuration` — milliseconds the device must remain within `minDistance` of its history window to confirm a stop
- `stopGap` — milliseconds; if a gap between positions is larger than this AND average speed during gap > threshold, treat it as move+stop

## Key flows

### Empty position history (lines 37-39)
If the deque is empty, return immediately. Processor requires at least one historical position.

### Gap detection (lines 43-56)
If the time gap from the last history position to current is > `stopGap`:
- Compute `gapAverageSpeed = gapDistance / gap`
- If speed > `minAverageSpeed` (minDistance/minDuration): the device moved significantly during the gap → fire DEVICE_STOPPED on last + DEVICE_MOVING on last + DEVICE_STOPPED on current. Reset `motionStreak = false`. This handles devices that go offline during a trip and report a large jump.

### Currently on a motion streak (lines 59-73)
Scan back through position history (descending). If any candidate position is ≥ `minDistance` from current → device is still moving → return (no event).
If ALL positions in the window are within `minDistance` AND the oldest position is ≥ `minDuration` old: fire DEVICE_STOPPED; set `motionStreak = false`.

### Not on a motion streak (lines 74-96)
Compute distance from the stop anchor (`state.getEventLatitude/Longitude()`) to current position.
If ≥ `minDistance`: device has moved — fire DEVICE_MOVING. Walk back through history to find the earliest position that's still part of the moving segment (for accurate trip-start time). Set `motionStreak = true`.

### Private `addEvent` (lines 99-105)
Creates event with `positionId`, `eventTime` from the triggering position, and calls `state.setEventPosition(position)` (which updates anchor + sets `changed=true`).

## Key insight vs. classic algorithm

Classic (`MotionProcessor`): uses real-time duration/distance timers; relies on the device sending motion flags or speed above threshold; can fire false trips with GPS jitter.

New (`NewMotionProcessor`): uses a position history window; detects trips by whether the device has moved away from its stopping location; more robust to GPS jitter because it requires sustained distance from the anchor. Requires `REPORT_TRIP_NEW_LOGIC = true` AND position history populated in `CacheManager`.

## Gotchas / non-obvious

- **Multiple events per call** — can emit 0, 1, 2, or 3 events. Caller must iterate `state.getEvents()`.
- **Position history window management** is in `CacheManager.updatePosition` — prunes positions older than `minDuration`. The processor assumes the deque contains positions within the relevant window.
- **`DistanceCalculator.distance(candidate, position)`** is Haversine — computes distance between position pairs directly.
- **Stop anchor vs. history window** — the anchor (eventLatitude/Longitude) is the last confirmed stop location; the history window is a rolling deque. They are different: anchor persists across the window pruning.

## Line index

- 31-97 — `updateState` main logic
- 37-39 — empty history guard
- 43-56 — gap detection (offline period)
- 59-73 — motion streak: check all-within-distance → confirm stop
- 74-96 — stopped: check distance from anchor → confirm trip
- 99-105 — `addEvent` helper
