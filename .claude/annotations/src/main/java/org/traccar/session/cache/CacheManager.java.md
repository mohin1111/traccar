# CacheManager.java

**Role:** The public-facing in-memory cache for the active-device object graph. Exposes `getObject`, `getDeviceObjects`, `getPosition`, `getServer` to the position-processing pipeline. Manages device lifecycle (add/remove on connect/disconnect) and keeps the cache consistent with DB changes via `BroadcastInterface` callbacks.
**Fits in:** `@Singleton`; injected into every handler in the position pipeline that needs entity lookups without DB queries (`GeofenceHandler`, `NotificationManager`, `ComputedAttributesHandler`, `MotionHandler`, `OverspeedHandler`, etc.). Also injected into [[ConnectionManager.java]].
**Read next:** [[CacheGraph.java]] (the underlying graph), [[session/ConnectionManager.java]] (calls `addDevice`/`removeDevice`)

## Public API

### Entity lookups (lines 97-133)
- `getObject(Class<T>, id)` → `T | null` — direct node lookup in the graph
- `getDeviceObjects(deviceId, Class<T>)` → `Set<T>` — all objects of a type linked to a device (traverses via Group proxy)
- `getPosition(deviceId)` → `Position | null` — most recent position for a device
- `getPositions(deviceId)` → `Deque<Position>` — position history deque (for new trip logic)
- `getServer()` → `Server` — the singleton server configuration object (updated on DB change)
- `getNotificationUsers(notificationId, deviceId)` → `Set<User>` — users to notify for a given notification+device combo
- `getDeviceNotifications(deviceId)` → `Set<Notification>` — all notifications applicable to device (direct + "always" flags)

### Device lifecycle (lines 135-182)
- `addDevice(deviceId, key)` — called by [[ConnectionManager.java]] when a device connects. On first reference: loads device from DB, builds graph links via `initializeCache`, optionally loads position history. `key` is a `ConnectionKey` (reference counting — same device on multiple connections keeps one cache entry).
- `removeDevice(deviceId, key)` — called on disconnect. Decrements reference count; evicts from graph only when count reaches zero.
- `updatePosition(Position)` — called by `DatabaseHandler` after persisting; appends to deque, prunes old positions.

### Cache invalidation (lines 214-318)
`invalidateObject(boolean local, Class<T>, id, ObjectOperation)` — called when an entity is created/updated/deleted (via REST API). Handles:
- DELETE → `graph.removeObject`
- UPDATE Server → reload from DB
- UPDATE GroupedModel → detect group change → re-link to new group
- UPDATE Schedulable → detect calendar change → re-link
- All cases → `graph.updateObject(after)`

`invalidatePermission(...)` — called when a permission link is added/removed (e.g., assigning a device to a user). Routes to `invalidatePermission` private method which selectively re-links.

## Key flows

### Device connect: `addDevice` (lines 135-171)
1. Check `deviceReferences` — if first connection for this device (empty references):
   a. Load `Device` from storage.
   b. `graph.addObject(device)` — strong root node.
   c. `initializeCache(device)` — loads all linked entities (group, users, geofences, notifications, attributes, drivers, maintenances, calendars) from storage and adds graph links.
   d. Load last position(s) from DB into `devicePositions` deque.
2. Add `key` (ConnectionKey) to references set.
3. Log debug with reference count.

### Graph initialization: `initializeCache` (lines 320-363)
Runs once per device-add. For a `GroupedModel` (Device, Driver, etc.):
- Links to parent `Group` (if `groupId > 0`).
- Links to each `User` who has permission to this object.
- For each class in `GROUPED_CLASSES` {Attribute, Device, Driver, Geofence, Maintenance, Notification}: loads all permissions of (device → class) and adds links.
For a `Schedulable` (Notification, etc.): links to `Calendar`.
For a `User`: loads Notification permissions.

### Position history management: `updatePosition` (lines 184-212)
Only runs if device has active references (i.e., is connected). Appends new position to deque. In new-trip mode: prunes positions older than `minDuration` from the front. In classic mode: keeps only the last position (deque size 1). This history is used by `NewMotionProcessor` to determine trip/stop transitions.

### Cache coherence with broadcast (lines 214-318)
When the REST API updates/deletes an entity, it calls `BroadcastService.invalidateObject`. This reaches `CacheManager.invalidateObject` on all nodes in the cluster. The manager re-fetches from DB and updates the in-memory graph — so the next position-processing cycle sees the new state without a DB query.

## Gotchas / non-obvious

- **Reference counting prevents premature eviction** — a device connected on two protocols simultaneously (e.g., TCP + HTTP) has two keys in references. Only when both disconnect is the cache evicted.
- **`addDevice` is `synchronized`** — prevents race between two connections for the same device arriving concurrently. `removeDevice` is also synchronized.
- **`getDeviceObjects` traverses via `Group` as proxy** — `graph.getObjects(Device.class, deviceId, Geofence.class, Set.of(Group.class), true)` returns geofences linked directly AND via the device's group hierarchy.
- **`getDeviceNotifications` two-pass logic** (lines 126-133): first gets all notification IDs directly linked to device (for "direct" check); then gets all notifications including via Group/User, but filters to keep only those that are `always=true` OR directly linked. This prevents group-level notifications from firing for devices they weren't directly assigned to.
- **`Server` is `volatile`** (line 75) — read without the lock; `invalidateObject` updates it under the lock.
- **`createObjectSupplier`** (lines 365-374) — wraps storage lookup in a `Supplier<T>`; passed to `graph.addLink` which calls it lazily only if the node doesn't already exist.

## Line index

- 66-67 — `GROUPED_CLASSES` constant (the 6 types that support group inheritance)
- 75-76 — `server` (volatile) + `devicePositions` + `deviceReferences`
- 80-86 — constructor: loads server, registers with broadcast
- 97-99 — `getObject` direct lookup
- 101-104 — `getDeviceObjects` (group-proxy traversal)
- 106-113 — `getPosition`, `getPositions`
- 115 — `getServer`
- 119-133 — `getNotificationUsers`, `getDeviceNotifications`
- 135-171 — `addDevice` (connect + init)
- 173-182 — `removeDevice` (disconnect + evict)
- 184-212 — `updatePosition` (deque management)
- 214-274 — `invalidateObject` (BroadcastInterface, object-level cache sync)
- 277-318 — `invalidatePermission` (BroadcastInterface, link-level cache sync)
- 320-363 — `initializeCache` (graph link initialization)
- 365-374 — `createObjectSupplier`
