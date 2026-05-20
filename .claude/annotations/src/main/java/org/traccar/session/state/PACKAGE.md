# `session/state/` — Motion and overspeed state machines

Six files implementing two independent state machines (motion detection and overspeed detection) each with a state object and a stateless processor. The state persists between positions via the `Device` row in DB; the processor is a pure function called per position in the pipeline.

## File index

| File | One-liner | Annotation |
|---|---|---|
| `MotionState.java` | Classic motion state: streak, timing, transition markers | [MotionState.java.md](MotionState.java.md) |
| `MotionProcessor.java` | Classic motion algorithm: duration+distance threshold confirmation | [MotionProcessor.java.md](MotionProcessor.java.md) |
| `NewMotionState.java` | New-logic motion state: position deque + event list + anchor | [NewMotionState.java.md](NewMotionState.java.md) |
| `NewMotionProcessor.java` | New-logic algorithm: distance-from-anchor with history window | [NewMotionProcessor.java.md](NewMotionProcessor.java.md) |
| `OverspeedState.java` | Overspeed state: active flag, start time, geofence source | [OverspeedState.java.md](OverspeedState.java.md) |
| `OverspeedProcessor.java` | Overspeed algorithm: speed×multiplier for minimalDuration | [OverspeedProcessor.java.md](OverspeedProcessor.java.md) |

## Dependency graph

```
MotionHandler (handler/)
 ├── MotionState.fromDevice(device)
 │    └── MotionProcessor.updateState(state, last, pos, newState, config)
 │         └── state.setEvent(event)
 └── NewMotionState (if REPORT_TRIP_NEW_LOGIC)
      └── NewMotionProcessor.updateState(state, pos, minDist, minDur, stopGap)
           └── state.setEvents(events)

OverspeedHandler (handler/)
 └── OverspeedState.fromDevice(device)
      └── OverspeedProcessor.updateState(state, pos, limit, mult, dur, geofenceId)
           └── state.setEvent(event)
```

## State machine design pattern

All six files follow the same pattern:

```
StateObject            Processor (final, private ctor)
 ├── fromDevice()       └── updateState(state, ...) — pure function
 ├── field get/set           mutates state
 │    └── sets changed       sets state.event(s)
 ├── toDevice()
 └── getEvent(s)
```

The handler:
1. Calls `StateObject.fromDevice(device)` to load state.
2. Calls `Processor.updateState(state, ...)`.
3. If `state.isChanged()`: calls `toDevice(device)` + `storage.updateObject(device, ...)`.
4. If `state.getEvent() != null` (or `state.getEvents()` non-empty): passes to notification manager.

## Classic vs. new motion algorithm

| | Classic (`MotionProcessor`) | New (`NewMotionProcessor`) |
|---|---|---|
| Enabled by | default | `Keys.REPORT_TRIP_NEW_LOGIC = true` |
| Mechanism | Real-time duration + distance timers | Distance from last-stop anchor; position history window |
| Data source | Two positions (last, current) | Full position deque from `CacheManager` |
| Events per call | 0 or 1 | 0, 1, 2, or 3 |
| Offline gap | Forces STOPPED if gap > `minimalNoDataDuration` | Gap detection via `stopGap` threshold |
| Jitter resistance | Moderate (duration threshold) | Better (requires sustained displacement from anchor) |

## Conventions

- **Processors are final utility classes** with private constructors — never instantiated, only `updateState` static method.
- **State setters auto-set `changed = true`** except `setEvent` / `setEvents` — events are side outputs, not persisted state.
- **`event` field on state objects** is `null` if no event this position; processor clears it at the start of each call (`state.setEvent(null)`).
- **Overspeed processor resets start time after firing** — allows repeated events during prolonged speeding.
