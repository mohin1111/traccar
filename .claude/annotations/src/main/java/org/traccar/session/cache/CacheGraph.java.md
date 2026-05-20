# CacheGraph.java

**Role:** The in-memory object graph for connected devices. Root nodes (devices) are held with strong references; all other linked objects (geofences, notifications, users, groups, attributes, etc.) are held in a `ConcurrentWeakValueMap` and reachable via bidirectional [[CacheNode.java]] links. Supports traversal with proxy-class expansion (Device→Group→Geofence).
**Fits in:** Created and managed by [[CacheManager.java]]; never accessed directly from outside the cache package.
**Read next:** [[CacheManager.java]], [[CacheNode.java]], [[CacheKey.java]]

## Public API

All methods are package-private (no `public` modifier on the class-level methods except the constructor and `toString`).

- `addObject(BaseModel)` — adds as a root node (strong reference); also registers in `nodes` weak map
- `removeObject(Class, id)` — removes from both maps; severs all backlinks from children
- `getObject(Class<T>, id)` → `T | null` — lookup by (class, id) in weak map
- `getObjects(fromClass, fromId, Class<T>, proxies, forward)` → `Stream<T>` — graph traversal from a start node, through optional proxy classes (e.g., Group as intermediary)
- `updateObject(BaseModel)` — replaces value in existing node (does not change links)
- `addLink(fromClass, fromId, toClass, toId, objectSupplier)` → `boolean` — creates directional link; if `toNode` doesn't exist yet, creates it using `objectSupplier` and initializes via `CacheManager.initializeCache`
- `removeLink(fromClass, fromId, toClass, toId)` — severs the bidirectional link
- `toString()` — debug tree print of all root nodes and their descendants

## Key flows

### Graph structure
```
roots: Map<CacheKey, CacheNode>   ← strong refs (only Device nodes)
nodes: ConcurrentWeakValueMap<CacheKey, CacheNode>  ← all nodes (weak refs for non-roots)
```

A `Device` node holds forward links to: `Geofence`, `Notification`, `Attribute`, `Driver`, `Maintenance`, `User`, `Group`.
Each linked node holds backlinks to its parents.

### Traversal with proxy expansion (lines 54-82)
`getObjects(fromClass, fromId, Geofence.class, Set.of(Group.class), forward=true)`:
1. Find root node for (fromClass, fromId).
2. Stream direct links to `Geofence.class` from that node.
3. For each proxy class (Group): stream links to Group from root, then recursively `getObjectStream` for Geofence from each Group node.
4. Result = UNION of direct and proxy-transitive results.

This is how "all geofences for a device, including those inherited from the device's group" is resolved without a DB query.

### Object removal (lines 39-46)
```java
node.getAllLinks(true)  // all forward-linked child nodes
    .forEach(child -> child.removeLink(key.clazz(), false, node));  // remove backlinks
roots.remove(key);
// node is then only weakly reachable → GC eligible
```

### `addLink` return value (line 91, 106)
Returns `false` if the `toNode` was newly created (did not exist in `nodes` yet). `CacheManager` uses this to trigger `initializeCache` on the new node — loading its own links from DB.

## Gotchas / non-obvious

- **Root nodes (Devices) survive GC; non-root nodes may not** — a Geofence linked to no active device can be GC'd since it's only in the weak map. This is the memory-management feature: cache grows with active connections and shrinks automatically.
- **`getObjects` may return duplicates** if a Geofence is linked both directly to a Device and via a Group. Callers must deduplicate if needed (e.g., use `collect(Collectors.toUnmodifiableSet())`).
- **`proxies` set prevents infinite recursion** — when traversing through Group, Group is added to `proxies` so the recursion stops at Group and doesn't re-enter itself.
- **`toString()` for debug only** — used in `CacheManager.toString()`. Can be called on production instances to dump the entire graph for troubleshooting.

## Line index

- 29-30 — `roots` (strong) and `nodes` (weak) maps
- 32-37 — `addObject`
- 39-46 — `removeObject`
- 48-52 — `getObject`
- 54-64 — `getObjects` (public traversal entry point)
- 66-82 — `getObjectStream` (recursive proxy expansion)
- 84-89 — `updateObject`
- 91-109 — `addLink` (creates node if absent, returns stop flag)
- 111-122 — `removeLink`
- 124-140 — `toString` / `printNode` debug
