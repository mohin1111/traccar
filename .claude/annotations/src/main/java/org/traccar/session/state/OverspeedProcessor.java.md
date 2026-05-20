# OverspeedProcessor.java

**Role:** Stateless utility implementing the overspeed state machine. Detects when a device exceeds a speed limit for a minimum duration and fires a `DEVICE_OVERSPEED` event.
**Fits in:** Called by `OverspeedHandler` (position pipeline). Works with [[OverspeedState.java]].
**Read next:** [[OverspeedState.java]], [[MotionProcessor.java]] (structural parallel), `handler/OverspeedHandler.java`

## Public API

### `updateState` (lines 27-69)
```java
public static void updateState(
    OverspeedState state, Position position,
    double speedLimit, double multiplier, long minimalDuration, long geofenceId)
```
- `speedLimit` — knots (Traccar standard)
- `multiplier` — tolerance factor (e.g., 1.1 = 10% grace above limit)
- `minimalDuration` — ms the device must exceed the limit before event fires
- `geofenceId` — geofence source of the limit (0 if device/server config)

## Key flows

### Entry into overspeed (lines 42-49)
If not currently in overspeed state AND speed > limit×multiplier:
1. Set `overspeedState = true`, record `overspeedTime`, `overspeedGeofenceId`.
2. Immediately call `checkEvent` — event may fire on the very first speeding position if duration threshold is 0.

### While speeding (lines 34-41)
If already in overspeed state:
- If still speeding → `checkEvent` (may fire if duration threshold crossed)
- If no longer speeding → reset state (clear time and geofenceId)

### Event generation: `checkEvent` (lines 51-66)
If `newTime - overspeedTime >= minimalDuration`:
1. Create `EVENT_DEVICE_OVERSPEED` with position, speed, speedLimit, geofenceId.
2. Reset `overspeedTime = null` and `overspeedGeofenceId = 0` — ready to fire again if speeding continues (allows repeated events for long overspeed periods).

## Gotchas / non-obvious

- **Reset after firing** — `overspeedTime` is cleared after each event but `overspeedState` stays `true`. This means if the device keeps speeding, a new timing window starts immediately → another event fires after another `minimalDuration`. Repeated events for sustained speeding are intentional.
- **Speed comparison uses `multiplier`** — configured via `event.overspeed.minimalSpeed` or similar; allows a 10% grace so 65kph in a 60 zone doesn't trigger if multiplier is 1.1.
- **`OverspeedProcessor` is final with private constructor** — same utility-class pattern as `MotionProcessor`.

## Line index

- 23 — `ATTRIBUTE_SPEED` constant
- 27-49 — main `updateState` logic (entry + while-speeding)
- 51-66 — `checkEvent` (duration check + event creation)
