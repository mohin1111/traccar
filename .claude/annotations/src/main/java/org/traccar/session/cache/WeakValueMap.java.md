# WeakValueMap.java

**Role:** A `HashMap`-backed map with `WeakReference` values. Allows the GC to reclaim values when no strong references remain — the map entry is then effectively stale (returns null on get).
**Fits in:** Used conceptually in the cache; `CacheGraph.nodes` uses `ConcurrentWeakValueMap` (from `helper/`) which extends this pattern with thread safety.
**Read next:** [[CacheGraph.java]], [[CacheNode.java]]

## Public API

- `put(K, V)` — wraps value in `WeakReference`, stores in map
- `get(K)` → `V | null` — dereferences the weak ref; returns null if key absent OR if the referent was GC'd
- `remove(K)` → `V | null` — removes entry, dereferences; returns null if already GC'd

Private `clean()` — removes stale entries (where `WeakReference.get() == null`); exists but is never called internally. Manual cleanup only.

## Gotchas / non-obvious

- **`clean()` is never called automatically** — stale `WeakReference` entries accumulate in the map's backing `HashMap` until the map itself is GC'd or `clean()` is called explicitly. For long-lived maps this can cause memory leak of the key objects (values are GC'd but keys remain). The production `ConcurrentWeakValueMap` in `helper/` may handle this differently.
- **Non-thread-safe** — plain `HashMap`; this class appears to be a utility/reference; the actual cache graph uses `ConcurrentWeakValueMap`.
- The weak-value pattern here means: if a `CacheNode` is only reachable via `nodes` (the weak map) and not via `roots` (strong map), it can be GC'd. Root nodes (active devices) are kept in `roots` with strong references.
