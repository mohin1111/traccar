# CacheNode.java

**Role:** A node in the bidirectional object graph. Holds a `BaseModel` value plus forward links (`links`) and backward links (`backlinks`) to other nodes, keyed by the linked class type.
**Fits in:** Populated and traversed by [[CacheGraph.java]]; never accessed directly by [[CacheManager.java]].
**Read next:** [[CacheGraph.java]], [[CacheKey.java]]

## Public API

- `getValue()` / `setValue(BaseModel)` — the wrapped entity; `volatile` for visibility across threads
- `linkStream(Class<? extends BaseModel>, boolean forward)` → `Stream<CacheNode>` — streams all linked nodes of a given class, either forward (parent→children) or backward (child→parents)
- `getAllLinks(boolean forward)` → `Stream<CacheNode>` — all linked nodes regardless of class
- `addLink(clazz, forward, node)` — adds to `links` (forward=true) or `backlinks` (forward=false) map; creates set on demand via `computeIfAbsent`
- `removeLink(clazz, forward, node)` — removes from the appropriate set

## Key flows

### Bidirectional link structure
`links: Map<Class, Set<CacheNode>>` — "what objects of each type does this node point to?"
`backlinks: Map<Class, Set<CacheNode>>` — "what objects point back to this node?"

When `CacheGraph.addLink(Device, deviceId, Geofence, geofenceId)`:
- Device node adds Geofence node to `links[Geofence.class]`
- Geofence node adds Device node to `backlinks[Device.class]`

This bidirectionality enables traversal in both directions: "all geofences for a device" and "all devices for a geofence".

## Gotchas / non-obvious

- **`value` is `volatile`** — allows `setValue` from any thread (e.g., cache invalidation on broadcast) while `getValue` is being called from a pipeline handler thread.
- **Sets are `ConcurrentHashMap.newKeySet()`** — thread-safe; multiple connections can modify links concurrently.
- **No deduplication guard in `addLink`** — if called twice with the same target node, it's added once (Set semantics handle deduplication by object identity/equals).
