# MotionProcessor.java

**Role:** Pure stateless utility class containing the motion state-machine logic. Takes a `MotionState` + two consecutive positions + config, updates the state in place, and optionally sets an event on the state object.
**Fits in:** Called by `MotionHandler` (position pipeline) with `MotionState` loaded from the device. Logic is separate from state to keep it testable.
**Read next:** [[MotionState.java]] (the mutable state), [[NewMotionProcessor.java]] (the newer alternative logic), `handler/MotionHandler.java`

## Public API

### `updateState` (lines 26-98)
```java
public static void updateState(
    MotionState state, Position last, Position position, boolean newState, TripsConfig tripsConfig)
```
Parameters:
- `state` — mutable; modified in place
- `last` — previous position (may be null on first position)
- `position` — current position
- `newState` — whether the current position is "moving" (speed > threshold, or from device motion flag)
- `tripsConfig` — thresholds: `minimalTripDuration`, `minimalTripDistance`, `minimalParkingDuration`, `minimalNoDataDuration`, `useIgnition`

## Key flows

### No-data gap detection (lines 31-43)
If `last != null` and `newTime - oldTime >= minimalNoDataDuration` AND the device was on a motion streak: force a STOPPED event at `last` position. Resets all pending state. This handles devices that lose signal while moving.

### State unchanged (lines 45-84)
If `oldState == newState` and there is a pending transition (`motionTime != null`):
- For moving (`newState=true`): fire DEVICE_MOVING if duration ≥ `minimalTripDuration` OR distance ≥ `minimalTripDistance`
- For stopped (`newState=false`): fire DEVICE_STOPPED if duration ≥ `minimalParkingDuration` OR ignition=false (if ignition-based trips enabled)

When fired: event gets `eventTime` from `motionTime` (the transition start time, not the current time).

### State changed (lines 85-96)
If `oldState != newState`:
- If `motionStreak == newState`: already confirmed that direction before; clear the pending transition markers
- If `motionStreak != newState`: start timing the transition; record `motionPositionId`, `motionTime`, `motionDistance`

### Key insight: two-phase confirmation
Motion state changes require sustained evidence before a trip/stop event is fired. The device must be in the new state for `minimalTripDuration` ms AND/OR travel `minimalTripDistance` meters before `DEVICE_MOVING` is emitted. Similarly, parking requires `minimalParkingDuration` before `DEVICE_STOPPED`. This prevents false trips from brief stops or GPS jitter.

## Gotchas / non-obvious

- **Event time is the transition START time**, not the current position's time. If a trip started 5 minutes ago but the confirmation threshold was just crossed, the event's `eventTime` is 5 minutes ago.
- **`newState` (is currently moving)** is computed outside this method — typically from `position.getBoolean(Position.KEY_MOTION)` or a speed-threshold check in `MotionHandler`. This processor doesn't know about speed directly.
- **Ignition shortcut** (line 65): if ignition=false and the device was moving, immediately confirm STOPPED regardless of duration. This prevents long "stopping" periods when the engine is off.
- **`MotionProcessor` is final with private constructor** — enforces utility-class pattern; cannot be instantiated or subclassed.

## Line index

- 26 — `updateState` signature
- 31-43 — no-data gap forced stop
- 45-84 — same-state: check thresholds, possibly fire event
- 85-96 — state-change: start or cancel pending transition
