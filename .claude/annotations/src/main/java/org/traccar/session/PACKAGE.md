# `session/` — Device session lifecycle, cache, and motion/overspeed state

Three subsystems: (1) the device-connection registry that maps IMEI→DeviceSession and fans out real-time updates to WebSocket clients; (2) the in-memory object graph cache for zero-DB-query position processing; (3) two motion-detection algorithms and an overspeed detector, each as state+processor pairs.

## File index

| File | One-liner | Annotation |
|---|---|---|
| `ConnectionKey.java` | (localAddress, remoteAddress) record — unique connection identifier | [ConnectionKey.java.md](ConnectionKey.java.md) |
| `DeviceSession.java` | Per-connection context: deviceId, channel, command dispatch, local kv store | [DeviceSession.java.md](DeviceSession.java.md) |
| `ConnectionManager.java` | Device registry + WebSocket fan-out hub + idle sweep | [ConnectionManager.java.md](ConnectionManager.java.md) |
| `cache/CacheManager.java` | In-memory entity cache: getObject/getDeviceObjects/getPosition | [cache/CacheManager.java.md](cache/CacheManager.java.md) |
| `cache/CacheGraph.java` | Bidirectional weak-ref object graph | [cache/CacheGraph.java.md](cache/CacheGraph.java.md) |
| `cache/CacheNode.java` | Graph node: value + typed forward/back links | [cache/CacheNode.java.md](cache/CacheNode.java.md) |
| `cache/CacheKey.java` | (Class, id) record as graph map key | [cache/CacheKey.java.md](cache/CacheKey.java.md) |
| `cache/WeakValueMap.java` | HashMap with WeakReference values (utility) | [cache/WeakValueMap.java.md](cache/WeakValueMap.java.md) |
| `state/MotionState.java` | Classic motion state: streak + transition timing | [state/MotionState.java.md](state/MotionState.java.md) |
| `state/MotionProcessor.java` | Classic motion algorithm (duration+distance confirmation) | [state/MotionProcessor.java.md](state/MotionProcessor.java.md) |
| `state/NewMotionState.java` | New-logic state: position deque + event list + anchor | [state/NewMotionState.java.md](state/NewMotionState.java.md) |
| `state/NewMotionProcessor.java` | New-logic algorithm (distance-from-anchor with history window) | [state/NewMotionProcessor.java.md](state/NewMotionProcessor.java.md) |
| `state/OverspeedState.java` | Overspeed state: active flag, start time, geofence source | [state/OverspeedState.java.md](state/OverspeedState.java.md) |
| `state/OverspeedProcessor.java` | Overspeed algorithm: speed×multiplier for minimalDuration | [state/OverspeedProcessor.java.md](state/OverspeedProcessor.java.md) |

## Dependency graph

```
Netty pipeline (protocol decoders)
  └── ConnectionManager.getDeviceSession(protocol, channel, remoteAddress, uniqueIds)
        ├── DeviceLookupService.lookup()   [DB, once per new connection]
        ├── DeviceSession (created, stored)
        └── CacheManager.addDevice()       [DB queries for graph init, once per connection]

ProcessingHandler (position pipeline, per position)
  ├── CacheManager.getDeviceObjects()      [NO DB — graph traversal]
  ├── CacheManager.getPosition()           [NO DB — deque peek]
  ├── MotionHandler
  │    ├── MotionState.fromDevice() / NewMotionState + CacheManager.getPositions()
  │    ├── MotionProcessor.updateState() / NewMotionProcessor.updateState()
  │    └── storage.updateObject(device) if changed
  ├── OverspeedHandler
  │    ├── OverspeedState.fromDevice()
  │    ├── OverspeedProcessor.updateState()
  │    └── storage.updateObject(device) if changed
  └── ConnectionManager.updatePosition()   [→ WebSocket listeners]

AsyncSocket (WebSocket)
  └── ConnectionManager.addListener(userId, this)
       ← onUpdateDevice / onUpdatePosition / onUpdateEvent / onUpdateLog
```

## The three subsystems

### 1. Connection registry (`ConnectionKey`, `DeviceSession`, `ConnectionManager`)

`ConnectionManager` maintains two concurrent maps:
- `sessionsByDeviceId: Map<Long, DeviceSession>` — fast lookup by DB device ID
- `sessionsByEndpoint: Map<ConnectionKey, Map<String, DeviceSession>>` — session(s) per network endpoint (a single TCP connection can serve multiple devices, e.g., multiplexer)

On protocol decoder call to `getDeviceSession(protocol, channel, addr, uniqueIds)`:
1. Fast path: return existing session if found by endpoint+uniqueId.
2. Slow path: DB lookup → create session → cache → `cacheManager.addDevice`.
3. Unknown device: log WARN, optionally auto-register if `DATABASE_REGISTER_UNKNOWN` is set.

On disconnect (`deviceDisconnected`): removes session, optionally marks OFFLINE, calls `cacheManager.removeDevice`.

WebSocket clients register as `UpdateListener` via `addListener`. Each listener receives `onUpdateDevice / onUpdatePosition / onUpdateEvent / onUpdateLog` callbacks which serialize to JSON over the WebSocket.

### 2. In-memory cache (`cache/`)

A bidirectional object graph where Device nodes are strong roots and all other nodes (Geofence, User, Notification, etc.) are weak-ref. The graph grows as devices connect and shrinks as they disconnect and the GC runs. See [cache/PACKAGE.md](cache/PACKAGE.md) for full detail.

### 3. Motion + overspeed state machines (`state/`)

Two motion algorithms (classic timer-based and new position-history-based) controlled by `Keys.REPORT_TRIP_NEW_LOGIC`. One overspeed algorithm. Each state machine is a pair: a mutable state object (serialized to/from the `Device` row) and a stateless processor (pure static function). See [state/PACKAGE.md](state/PACKAGE.md) for full detail.

## Multi-file flows

### New device connects for the first time ever
1. Decoder calls `ConnectionManager.getDeviceSession(..., uniqueId)`.
2. `DeviceLookupService.lookup` returns null.
3. If `DATABASE_REGISTER_UNKNOWN = true`: `addUnknownDevice` → INSERT into tc_devices → returns new Device.
4. `DeviceSession` created with new Device.
5. `CacheManager.addDevice(deviceId, connectionKey)` → loads all linked entities from DB into graph.
6. Device status set to ONLINE → `updateDevice` → WebSocket listeners notified.

### Position arrives from connected device
1. Decoder gets `DeviceSession` from `ConnectionManager` (fast path — session already cached).
2. Sets `session.setLastUpdate(now)` (prevents idle timeout).
3. Creates `Position` with `deviceId` from session.
4. `ProcessingHandler` runs pipeline handlers, all using `CacheManager` for entity lookups.
5. `DatabaseHandler` persists position; `CacheManager.updatePosition` updates deque.
6. `ConnectionManager.updatePosition(local=true, position)` → fans to WebSocket listeners.

### Admin changes a geofence assignment
1. REST `PUT /api/permissions` → `PermissionsResource` → `storage.addPermission(permission)`.
2. `CacheManager.invalidatePermission(local=true, Device.class, deviceId, Geofence.class, geofenceId, link=true)`.
3. `BroadcastService` propagates to all cluster nodes.
4. Each node's `CacheManager.invalidatePermission` updates its graph in-memory.
5. Next position for that device: `GeofenceHandler` sees the new geofence immediately.

## Conventions

- **`ConnectionManager` is the only class that calls `cacheManager.addDevice/removeDevice`** — never call these from protocol decoders or handlers.
- **Never call `Storage` from a hot position handler** — use `CacheManager.getDeviceObjects()` instead. DB queries in the pipeline will cause throughput collapse under load.
- **`BroadcastInterface`** is implemented by both `ConnectionManager` and `CacheManager` — both register with `BroadcastService`. When a mutation arrives from a remote node, both get notified independently. Order is not guaranteed.
