# MotionState.java

**Role:** Mutable state object holding the current motion detection state for one device. Persisted to the `Device` row and loaded back on reconnect. The state machine's data — separate from the logic in [[MotionProcessor.java]].
**Fits in:** Created by `MotionHandler` (loaded from `Device` via `fromDevice`), mutated by [[MotionProcessor.java]], persisted back via `toDevice` then a storage update.
**Read next:** [[MotionProcessor.java]], [[NewMotionState.java]] (the new-logic counterpart)

## Public API

### Factory / persistence bridge (lines 25-41)
- `fromDevice(Device)` → `MotionState` — loads all five state fields from Device attributes
- `toDevice(Device)` — writes all five fields back to the Device for persistence

### State fields (lines 43-113)
All setters set `changed = true` automatically:
- `motionStreak: boolean` — whether the device has been consistently in the current motion state long enough for it to be "confirmed"
- `motionState: boolean` — current raw motion flag (true = moving)
- `motionPositionId: long` — position ID at which the potential state transition began
- `motionTime: Date` — time at which the potential transition began
- `motionDistance: double` — total distance accumulator at transition start
- `event: Event | null` — set by `MotionProcessor` when a `DEVICE_MOVING` or `DEVICE_STOPPED` event should be fired; null = no event this position

### Change tracking (lines 43-46)
`isChanged()` — set to `true` on any setter call. `MotionHandler` uses this to skip the DB `updateObject` call when nothing changed.

## Key flows

### Motion event lifecycle
1. New position arrives → `MotionHandler` calls `MotionProcessor.updateState(state, last, position, newMotionFlag, config)`.
2. `MotionProcessor` may call `state.setEvent(event)` if a trip-start or trip-stop is confirmed.
3. `MotionHandler` calls `toDevice(device)` + `storage.updateObject(device, ...)` if `state.isChanged()`.
4. If `state.getEvent() != null`, the event is passed to `notificationManager`.

## Gotchas / non-obvious

- **`changed` is never reset to `false`** — once any setter is called, `isChanged()` returns true forever. This is fine because the state is discarded after each position (a new one is created from `fromDevice` for the next).
- **`event` setter does NOT set `changed`** — the event is a side output, not persisted state. Only the five motion fields are persisted via `toDevice`.
- `motionPositionId` and `motionTime` being zero/null means no pending transition — the device hasn't crossed the threshold yet.
