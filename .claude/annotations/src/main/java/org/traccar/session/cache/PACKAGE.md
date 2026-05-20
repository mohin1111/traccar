# `session/cache/` — In-memory object graph for connected devices

A reference-counted, bidirectional object graph that caches all entities related to currently-connected devices. Eliminates DB queries from the hot position-processing path. Entries are added on device connection and evicted on disconnect, with weak references for non-root (non-device) nodes allowing GC reclamation.

## File index

| File | One-liner | Annotation |
|---|---|---|
| `CacheManager.java` | Public API: `getObject`, `getDeviceObjects`, `getPosition`, `addDevice`, cache invalidation | [CacheManager.java.md](CacheManager.java.md) |
| `CacheGraph.java` | Bidirectional node graph with root-strong / non-root-weak references | [CacheGraph.java.md](CacheGraph.java.md) |
| `CacheNode.java` | Single node: value + typed forward links + typed backlinks | [CacheNode.java.md](CacheNode.java.md) |
| `CacheKey.java` | `(Class, long id)` record used as map key | [CacheKey.java.md](CacheKey.java.md) |
| `WeakValueMap.java` | HashMap with WeakReference values (utility/reference impl) | [WeakValueMap.java.md](WeakValueMap.java.md) |

## Dependency graph

```
CacheManager (public, @Singleton)
 └── CacheGraph
      ├── roots: Map<CacheKey, CacheNode>  ← strong refs (Device nodes)
      └── nodes: ConcurrentWeakValueMap<CacheKey, CacheNode>  ← weak refs (all)
           └── CacheNode
                ├── value: volatile BaseModel
                ├── links: Map<Class, Set<CacheNode>>  ← forward
                └── backlinks: Map<Class, Set<CacheNode>>  ← reverse
```

`CacheKey`, `WeakValueMap` are internal utilities.

## What is cached per connected device

When `CacheManager.addDevice(deviceId, key)` is called:
```
Device (strong root)
 ├── Group (if groupId > 0)
 │    ├── Geofence(s) linked to group
 │    ├── Notification(s) linked to group
 │    ├── Attribute(s) linked to group
 │    └── ... (other GROUPED_CLASSES)
 ├── User(s) who have permission to device
 │    └── Notification(s) linked to user
 ├── Geofence(s) linked directly to device
 ├── Notification(s) linked directly
 ├── Attribute(s) / Driver(s) / Maintenance(s)
 └── Calendar(s) for schedulable entities
```
Position history deque is stored separately in `devicePositions` map.

## The weak-reference eviction strategy

Root nodes (Devices) → `ConcurrentHashMap roots` (strong). Non-root nodes → `ConcurrentWeakValueMap nodes` (weak). A Geofence linked to no active device can be GC'd automatically because:
- It's not in `roots`
- The only reference in `nodes` is a `WeakReference`
- When all device root nodes that linked to it are evicted, no more strong paths exist

This means the cache is self-managing: no manual eviction for non-device entities.

## Cache coherence with DB changes

`CacheManager implements BroadcastInterface`. When the REST API modifies an entity:
1. API resource calls `BroadcastService.invalidateObject(local=true, ...)` / `invalidatePermission`.
2. `BroadcastService` fans out to all cluster nodes (Redis pub/sub or UDP multicast).
3. Each node's `CacheManager.invalidateObject/invalidatePermission` is called.
4. Manager re-fetches the updated entity from DB and updates the graph node, or removes/re-links as appropriate.

This ensures all pipeline handlers read fresh data without their own DB queries.

## Multi-file flows

### Position pipeline reads (every position, hot path)
```
GeofenceHandler → cacheManager.getDeviceObjects(deviceId, Geofence.class)  // no DB
NotificationManager → cacheManager.getDeviceNotifications(deviceId)        // no DB
ComputedAttributesHandler → cacheManager.getDeviceObjects(deviceId, Attribute.class)  // no DB
OverspeedHandler → cacheManager.getDeviceObjects(deviceId, Geofence.class) // for speed limit geofences
```

### Device connect (ConnectionManager.getDeviceSession, new session)
```
ConnectionManager → cacheManager.addDevice(deviceId, connectionKey)
  → graph.addObject(device)
  → initializeCache(device)  → storage queries to load all linked entities
  → load position history from storage
```

### Device disconnect
```
ConnectionManager → cacheManager.removeDevice(deviceId, connectionKey)
  → if references empty: graph.removeObject(Device.class, deviceId)
  → devicePositions.remove(deviceId)
```

## How to add a new cached relationship

1. Add the new `Class` to `GROUPED_CLASSES` set in `CacheManager` if it follows the Device→Group→Entity inheritance pattern.
2. If it needs a custom link relationship, add handling in `CacheManager.initializeCache` and `CacheManager.invalidatePermission`.
3. Add a `getDeviceObjects(deviceId, NewClass.class)` consumer in the relevant handler.
