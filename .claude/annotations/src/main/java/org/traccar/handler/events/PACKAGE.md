# `handler/events/` — Event Detection Layer

Event detectors run after the full position pipeline chain completes. They analyze the enriched position (with geofences, motion, speed limit, etc.) and emit `Event` objects which are then persisted and dispatched to notification subscribers.

## File index

| File | One-liner | Annotation |
|---|---|---|
| `BaseEventHandler.java` | Abstract base: `Callback` contract, exception wrapper | [BaseEventHandler.java.md](BaseEventHandler.java.md) |
| `AlarmEventHandler.java` | Device alarm codes → `TYPE_ALARM` events | [AlarmEventHandler.java.md](AlarmEventHandler.java.md) |
| `BehaviorEventHandler.java` | Speed delta → harsh acceleration/braking alarms | [BehaviorEventHandler.java.md](BehaviorEventHandler.java.md) |
| `CommandResultEventHandler.java` | `KEY_RESULT` attribute → `TYPE_COMMAND_RESULT` | [CommandResultEventHandler.java.md](CommandResultEventHandler.java.md) |
| `DriverEventHandler.java` | Driver ID change → `TYPE_DRIVER_CHANGED` | [DriverEventHandler.java.md](DriverEventHandler.java.md) |
| `FuelEventHandler.java` | Fuel level delta → refuel / drop events | [FuelEventHandler.java.md](FuelEventHandler.java.md) |
| `GeofenceEventHandler.java` | Geofence set diff → enter / exit events | [GeofenceEventHandler.java.md](GeofenceEventHandler.java.md) |
| `IgnitionEventHandler.java` | Ignition boolean transition → on / off events | [IgnitionEventHandler.java.md](IgnitionEventHandler.java.md) |
| `MaintenanceEventHandler.java` | Odometer/hours threshold crossing → maintenance due | [MaintenanceEventHandler.java.md](MaintenanceEventHandler.java.md) |
| `MediaEventHandler.java` | IMAGE/VIDEO/AUDIO attributes → `TYPE_MEDIA` | [MediaEventHandler.java.md](MediaEventHandler.java.md) |
| `MotionEventHandler.java` | Sustained motion state → trip start/stop events | [MotionEventHandler.java.md](MotionEventHandler.java.md) |
| `OverspeedEventHandler.java` | Sustained overspeed (state machine) → overspeed event | [OverspeedEventHandler.java.md](OverspeedEventHandler.java.md) |
| `ProximityEventHandler.java` | Device-to-device distance → proximity enter/exit/unaccompanied | [ProximityEventHandler.java.md](ProximityEventHandler.java.md) |

## Event types produced

| Event type constant | Handler |
|---|---|
| `TYPE_ALARM` | `AlarmEventHandler`, `BehaviorEventHandler` |
| `TYPE_COMMAND_RESULT` | `CommandResultEventHandler` |
| `TYPE_DEVICE_OVERSPEED` | `OverspeedEventHandler` |
| `TYPE_DEVICE_FUEL_INCREASE` / `TYPE_DEVICE_FUEL_DROP` | `FuelEventHandler` |
| `TYPE_DRIVER_CHANGED` | `DriverEventHandler` |
| `TYPE_GEOFENCE_ENTER` / `TYPE_GEOFENCE_EXIT` | `GeofenceEventHandler` |
| `TYPE_IGNITION_ON` / `TYPE_IGNITION_OFF` | `IgnitionEventHandler` |
| `TYPE_MAINTENANCE` | `MaintenanceEventHandler` |
| `TYPE_MEDIA` | `MediaEventHandler` |
| `TYPE_MOTION_MOVING` / `TYPE_MOTION_STOPPED` | `MotionEventHandler` |
| `TYPE_PROXIMITY_ENTER` / `TYPE_PROXIMITY_EXIT` | `ProximityEventHandler` |
| `TYPE_UNACCOMPANIED_MOTION` | `ProximityEventHandler` |

## BaseEventHandler contract

```java
abstract void onPosition(Position position, Callback callback);
// callback.eventDetected(Event event) — call zero or more times
// Handlers are synchronous — no async callbacks
// Exceptions are caught by analyzePosition() wrapper — failure skips detection, does not halt pipeline
```

**Key difference from `BasePositionHandler`**: event handlers do not filter positions (no `processed(filtered)` concept). They only detect and emit events.

## Dependency graph (intra-package)

```
BaseEventHandler
  ├── AlarmEventHandler         → CacheManager (last position for dedup)
  ├── BehaviorEventHandler      → CacheManager
  ├── CommandResultEventHandler (stateless)
  ├── DriverEventHandler        → CacheManager
  ├── FuelEventHandler          → CacheManager, AttributeUtil
  ├── GeofenceEventHandler      → CacheManager (geofence objects + calendars)
  ├── IgnitionEventHandler      → CacheManager
  ├── MaintenanceEventHandler   → CacheManager (maintenance objects)
  ├── MediaEventHandler         (stateless)
  ├── MotionEventHandler        → CacheManager, Storage (writes to tc_devices)
  ├── OverspeedEventHandler     → CacheManager, Storage (writes to tc_devices)
  └── ProximityEventHandler     → CacheManager, DistanceCalculator
```

## State-mutating handlers

Two event handlers write device state to the DB on every state change:
- **`MotionEventHandler`** — persists `motionStreak`, `motionTime`, `motionLatitude`, `motionLongitude`.
- **`OverspeedEventHandler`** — persists `overspeedState`, `overspeedTime`, `overspeedGeofenceId`.

These handlers use the state-machine pattern: `XxxState.fromDevice(device)` → `XxxProcessor.updateState(...)` → `state.toDevice(device)` → `storage.updateObject(device)`.

## Multi-file flows

### Event → notification flow
1. `ProcessingHandler` calls `eventHandler.analyzePosition(position, event -> notificationManager.handleEvent(event))`.
2. `NotificationManager` evaluates notification rules for the device's users.
3. Matching `Notification` objects dispatch via configured `Notificator` channels (mail, SMS, Firebase push, Telegram, etc.).

### GeofenceEventHandler pipeline dependency
`GeofenceEventHandler` reads `position.geofenceIds` — which is set by `GeofenceHandler` in the main pipeline (step 9). This dependency requires `GeofenceHandler` to run before events are evaluated.

### OverspeedEventHandler pipeline dependency
`OverspeedEventHandler` reads `KEY_SPEED_LIMIT` (from `SpeedLimitHandler`, step 11) and `position.geofenceIds` (from `GeofenceHandler`, step 9).

## `isLatest` guard pattern

Many handlers call `PositionUtil.isLatest(cacheManager, position)` early and return if false. This prevents event storms during historical position imports (bulk replay). Events should only fire for real-time positions, not re-processed historical data.

## How to add a new event type

1. Create `MyEventHandler extends BaseEventHandler`.
2. Implement `onPosition(Position, Callback)` — call `callback.eventDetected(new Event(Event.TYPE_MY_EVENT, position))` when the condition is met.
3. Add `@Inject` constructor.
4. Add a Guice binding in `MainModule.java`.
5. Add the handler to the event handler list in `ProcessingHandler`.
6. Add the new event type constant to `model/Event.java` (e.g., `TYPE_MY_EVENT = "myEvent"`).
7. Add a notification type in the web UI and `NotificationManager` if user-facing alerting is needed.
8. If the handler reads a pipeline-produced attribute (like `geofenceIds`, `KEY_SPEED_LIMIT`), confirm that the relevant pipeline handler runs first.
