# NewMotionState.java

**Role:** State object for the "new" position-history-based motion logic. Unlike [[MotionState.java]] (which stores transition timing), this holds a reference to the device's position history deque and outputs a list of events rather than a single event.
**Fits in:** Used by [[NewMotionProcessor.java]]; populated from `CacheManager.getPositions(deviceId)` in `MotionHandler` when `REPORT_TRIP_NEW_LOGIC` is enabled.
**Read next:** [[NewMotionProcessor.java]], [[MotionState.java]] (classic counterpart), `session/cache/CacheManager.java` (provides position deque)

## Public API

- `isChanged()` / `setChanged(boolean)` — whether the state was modified (triggers device DB update)
- `getMotionStreak()` / `setMotionStreak(boolean)` — confirmed motion direction
- `getPositions()` / `setPositions(Deque<Position>)` — the sliding position window from CacheManager
- `getEvents()` / `setEvents(List<Event>)` — zero or more events generated per position (vs. exactly one in classic)
- `getEventTime()`, `getEventLatitude()`, `getEventLongitude()` — coordinates of the last motion state change
- `setEventPosition(Date, lat, lon)` — set all three in one call
- `setEventPosition(Position)` — convenience; also sets `changed = true`

## Key flows

### Multiple events per position
`NewMotionProcessor` can emit 0, 1, 2, or 3 events per position (e.g., a stop→move→stop gap creates both DEVICE_STOPPED + DEVICE_MOVING + DEVICE_STOPPED). The classic `MotionState` supports at most one event.

### Event position tracking
`eventTime/eventLatitude/eventLongitude` store where the last confirmed stop occurred — the "anchor point" for detecting the next trip start. If the current position is ≥ `minDistance` away from this anchor, a trip has started.

## Gotchas / non-obvious

- `setEventPosition(Position)` is the only setter that auto-sets `changed = true`. The other setters and `setChanged(boolean)` are explicit.
- This state is NOT persisted via `fromDevice`/`toDevice` like `MotionState` — it is derived from the position deque each time. The `motionStreak` boolean is the only thing that needs persistence.
