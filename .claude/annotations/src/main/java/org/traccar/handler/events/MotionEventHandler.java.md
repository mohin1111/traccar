# MotionEventHandler.java

**Role:** Detects trip start/stop events by tracking sustained motion state changes. Supports two logic paths: new logic (`REPORT_TRIP_NEW_LOGIC`) using `NewMotionProcessor` and old logic using `MotionProcessor` + `TripsConfig`. Persists motion state to `tc_devices`.
**Fits in:** Extends `BaseEventHandler`. The most complex event handler — it writes device state to DB on every state change.
**Read next:** [[BaseEventHandler.java.md]], [[OverspeedEventHandler.java.md]] (similar state-machine pattern)

## Public API

### `onPosition(Position, Callback)` (lines 57-75)
- Guards: device must exist and position must be latest.
- Dispatches to `handleNewLogic` or `handleOldLogic` based on config.

### `handleNewLogic` (lines 77-116)
Uses `NewMotionState` (loaded from device fields `motionStreak`, `motionTime`, `motionLatitude`, `motionLongitude`) and `cacheManager.getPositions(deviceId)` (recent position window). Calls `NewMotionProcessor.updateState`. Persists state changes to DB. Emits all events from `state.getEvents()`.

### `handleOldLogic` (lines 118-136)
Uses `MotionState.fromDevice(device)` and `MotionProcessor.updateState(state, last, position, motion, tripsConfig)`. Persists state to DB. Emits at most one event from `state.getEvent()`.

## Gotchas / non-obvious

- **Writes to DB on every state change** — unlike event handlers that only read, this handler mutates `tc_devices` (motionStreak, motionTime, motionLatitude, motionLongitude or motionState, motionPositionId, etc.).
- **TODO migration path** (lines 83-88) — old `motionTime`/`motionLat`/`motionLon` device attributes are migrated to typed columns on first encounter. This suggests the new logic was introduced mid-version.
- **`cacheManager.getPositions`** in new logic — requires a short position window (not just last), implying cache stores recent history.
- **Two logic paths** — `REPORT_TRIP_NEW_LOGIC` flag switches the algorithm. Old logic uses speed threshold + time windows. New logic uses distance + duration + stop-gap parameters.

## Line index

- 57 — `onPosition`
- 66-70 — new logic branch
- 71-74 — old logic branch
- 77-116 — `handleNewLogic`
- 97-112 — DB persist + error handling
- 113-115 — event emission (can be multiple)
- 118-136 — `handleOldLogic`
- 125-131 — DB persist
