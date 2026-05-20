# `handler/` — Position-Processing Pipeline

The central business-logic layer of Traccar. Every GPS position decoded by a protocol decoder flows through this handler chain before being persisted. The chain is orchestrated by `ProcessingHandler` (root package, annotated separately).

## Position pipeline — ordered chain

| Step | Handler | Purpose | Note |
|---|---|---|---|
| 1 | `ComputedAttributesHandler.Early` | JEXL expressions with priority < 0 | Extension point |
| 2 | `OutdatedHandler` | Fill missing GPS coords from last position | — |
| 3 | `TimeHandler` | GPS rollover correction + time override | — |
| 4 | `GeolocationHandler` | Cell/WiFi → GPS (if no fix) | Optional, async |
| 5 | `HemisphereHandler` | Hemisphere sign fix for unsigned lat/lon | Optional |
| 6 | `MapMatcherHandler` | Road snap (GraphHopper) | Optional, async |
| 7 | `DistanceHandler` | Incremental + cumulative odometer | — |
| 8 | `FilterHandler` | Quality gate — drops invalid/duplicate | Only `processed(true)` here |
| 9 | `GeofenceHandler` | Geofence membership → `position.geofenceIds` | — |
| 10 | `GeocoderHandler` | Reverse-geocode → `position.address` | Optional, async |
| 11 | `SpeedLimitHandler` | Road speed limit → `KEY_SPEED_LIMIT` | Optional, async |
| 12 | `MotionHandler` | Derive `KEY_MOTION` from speed | — |
| 13 | `ComputedAttributesHandler.Late` | JEXL expressions with priority >= 0 | Extension point |
| 14 | `DriverHandler` | Fallback RFID → linked driver | Optional |
| 15 | `CopyAttributesHandler` | Carry forward sticky attributes | — |
| 16 | `EngineHoursHandler` | Accumulate engine run time | — |
| 17 | `PositionForwardingHandler` | Push to external systems (Kafka/HTTP/MQTT…) | Fire-and-forget |
| 18 | `DatabaseHandler` | Persist to `tc_positions` | Sets `position.id` |
| 19 | `PostProcessHandler` | Update device pointer, cache, WebSocket push | — |

After step 19, **event handlers** fire (in `handler/events/`) — see `events/PACKAGE.md`.

## File index

| File | One-liner | Annotation |
|---|---|---|
| `BasePositionHandler.java` | Abstract base: `Callback` contract, exception wrapper | [BasePositionHandler.java.md](BasePositionHandler.java.md) |
| `ComputedAttributesHandler.java` | JEXL evaluation (Early + Late inner classes) | [ComputedAttributesHandler.java.md](ComputedAttributesHandler.java.md) |
| `ComputedAttributesProvider.java` | Singleton JEXL engine + sandbox + context builder | [ComputedAttributesProvider.java.md](ComputedAttributesProvider.java.md) |
| `CopyAttributesHandler.java` | Carry-forward sticky attributes from last position | [CopyAttributesHandler.java.md](CopyAttributesHandler.java.md) |
| `DatabaseHandler.java` | Persist position; sets `position.id` | [DatabaseHandler.java.md](DatabaseHandler.java.md) |
| `DistanceHandler.java` | Incremental + cumulative odometer; jitter filter | [DistanceHandler.java.md](DistanceHandler.java.md) |
| `DriverHandler.java` | Fallback driver ID from linked Driver entity | [DriverHandler.java.md](DriverHandler.java.md) |
| `EngineHoursHandler.java` | Accumulate engine hours from ignition state | [EngineHoursHandler.java.md](EngineHoursHandler.java.md) |
| `FilterHandler.java` | 13-rule quality gate; only handler that drops positions | [FilterHandler.java.md](FilterHandler.java.md) |
| `GeocoderHandler.java` | Async reverse-geocode → address | [GeocoderHandler.java.md](GeocoderHandler.java.md) |
| `GeofenceHandler.java` | Geofence containment → `position.geofenceIds` | [GeofenceHandler.java.md](GeofenceHandler.java.md) |
| `GeolocationHandler.java` | Cell/WiFi geolocation for GPS-less positions | [GeolocationHandler.java.md](GeolocationHandler.java.md) |
| `HemisphereHandler.java` | Force hemisphere sign on unsigned coordinates | [HemisphereHandler.java.md](HemisphereHandler.java.md) |
| `MapMatcherHandler.java` | Async road snap via MapMatcher | [MapMatcherHandler.java.md](MapMatcherHandler.java.md) |
| `MotionHandler.java` | Derive `KEY_MOTION` boolean from speed | [MotionHandler.java.md](MotionHandler.java.md) |
| `OutdatedHandler.java` | Fill GPS coords from last position for outdated messages | [OutdatedHandler.java.md](OutdatedHandler.java.md) |
| `PositionForwardingHandler.java` | External forwarding with exponential-backoff retry | [PositionForwardingHandler.java.md](PositionForwardingHandler.java.md) |
| `PostProcessHandler.java` | Update device `positionId`, cache, WebSocket push | [PostProcessHandler.java.md](PostProcessHandler.java.md) |
| `SpeedLimitHandler.java` | Async road speed limit lookup | [SpeedLimitHandler.java.md](SpeedLimitHandler.java.md) |
| `TimeHandler.java` | GPS week rollover + time override | [TimeHandler.java.md](TimeHandler.java.md) |

## How handlers are wired (Guice / ProcessingHandler)

`ProcessingHandler` (root package) is injected with all handler instances via Guice. In `MainModule.java`, optional handlers (`GeocoderHandler`, `GeolocationHandler`, `MapMatcherHandler`, `SpeedLimitHandler`) are bound conditionally — only if the corresponding provider is configured. The chain is assembled in `ProcessingHandler` as a list; each handler's `handlePosition` is called with a callback that advances to the next.

`ComputedAttributesHandler` appears twice — `Early` (injected as step 1) and `Late` (step 13). Both are distinct Guice bindings for the same class.

## BasePositionHandler contract

```java
abstract void onPosition(Position position, Callback callback);
// callback.processed(true)  → position is dropped; chain stops
// callback.processed(false) → position continues to next handler
// Must call callback exactly once, including in async paths
```

## Dependency graph (intra-package)

```
BasePositionHandler
  ├── ComputedAttributesHandler → ComputedAttributesProvider (singleton JEXL engine)
  ├── OutdatedHandler
  ├── TimeHandler
  ├── GeolocationHandler (optional)
  ├── HemisphereHandler
  ├── MapMatcherHandler (optional)
  ├── DistanceHandler
  ├── FilterHandler → StatisticsManager (daily limit counter)
  ├── GeofenceHandler → GeofenceUtil
  ├── GeocoderHandler → Geocoder (optional)
  ├── SpeedLimitHandler → SpeedLimitProvider (optional)
  ├── MotionHandler
  ├── DriverHandler
  ├── CopyAttributesHandler
  ├── EngineHoursHandler
  ├── PositionForwardingHandler → PositionForwarder (optional)
  ├── DatabaseHandler → PositionBatchWriter
  └── PostProcessHandler → ConnectionManager (WebSocket push)
```

All handlers share: `CacheManager` (device-level attribute lookup and last-position cache).

## Multi-file flows

### Per-position flow (normal path)
1. `ProcessingHandler.onMessage(position)` → calls `ComputedAttributesHandler.Early.handlePosition`.
2. Chain executes steps 2–17.
3. `DatabaseHandler` assigns `position.id` (async DB write).
4. `PostProcessHandler` updates device pointer + WebSocket push.
5. Event handlers iterate in parallel: each `BaseEventHandler.analyzePosition(position, callback)`.
6. `NotificationManager` receives events, evaluates user notification rules, dispatches via notificators.

### Filtered position path
`FilterHandler` calls `callback.processed(true)` → `ProcessingHandler` stops the chain. No DB write. No events.

## Conventions

- All handlers call `callback.processed(false)` unconditionally except `FilterHandler` (which may call `true`).
- Async handlers (geocoder, geolocation, map matcher, speed limit) call callback from an external thread pool — the chain pauses at that step.
- All per-device config is read via `AttributeUtil.lookup(cacheManager, key, deviceId)` — cascades device → group → server.
- `CacheManager.getPosition(deviceId)` returns the last known position for a device (before the current one is persisted).

## How to add a new handler

1. Create `MyHandler extends BasePositionHandler`.
2. Implement `onPosition(Position, Callback)`. Call `callback.processed(false)` in all paths.
3. Add `@Inject` constructor; inject `CacheManager`, `Config`, or whatever you need.
4. Add a Guice binding in `MainModule.java` (or conditionally in a sub-module).
5. Add the handler instance to the chain in `ProcessingHandler` at the desired position.
6. If the handler modifies a position field that downstream handlers depend on, confirm pipeline order.
7. Document config keys in `config/Keys.java`.
