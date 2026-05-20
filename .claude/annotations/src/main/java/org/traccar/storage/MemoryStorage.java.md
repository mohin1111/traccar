# MemoryStorage.java

**Role:** In-memory `Storage` implementation backed by `HashMap<Class<?>, Map<Long, Object>>`. Used in unit tests and as a lightweight alternative when no DB is configured. Pre-seeds a default `Server` object.
**Fits in:** Swapped in for [[DatabaseStorage.java]] in test harness. Implements the same [[Storage.java]] contract.
**Read next:** [[Storage.java]] (the contract), [[DatabaseStorage.java]] (the production impl for comparison)

## Public API

Implements all `Storage` abstract methods. Adds no public API of its own.

### Data stores
- `objects: Map<Class<?>, Map<Long, Object>>` — keyed by entity class then by `long` ID
- `permissions: Map<Pair<Class<?>,Class<?>>, Set<Pair<Long,Long>>>` — keyed by (ownerClass, propertyClass) pair
- `increment: AtomicLong` — auto-increment ID generator

## Key flows

### Construction (lines 43-48)
Pre-seeds `Server` with `id=1, registration=true` so the minimal Traccar UI flow works without a DB.

### `getObjectsStream` (lines 58-62)
Streams all objects of the class, filtering via `checkCondition`. No SQL; pure Java predicate.

### `checkCondition` (lines 64-108)
Java sealed `switch` over `Condition` subtypes:
- `Compare` — uses `Comparable.compareTo` with operator string switch
- `Between` — two comparisons via `Comparable`
- `Binary (And/Or)` — recursive boolean short-circuit
- `Permission` — iterates the permissions set; matches ownerId or propertyId
- Default → `false` (unsupported conditions like `Contains`, `LatestPositions` are NOT implemented)

### `updateObject` (lines 127-149)
Reads getter values from the entity, then applies setter to matching stored object(s). Uses `ReflectionCache.getProperties` for both getter and setter handles. If condition is set, finds exact-ID object; else updates all.

### `addObject` (lines 120-123)
Returns `increment.incrementAndGet()` — IDs start at 1, monotonically increasing. Unlike `DatabaseStorage`, IDs are not DB-generated; they are in-process counters.

## Gotchas / non-obvious

- **`Contains` and `LatestPositions` conditions are NOT supported** — `checkCondition` default branch returns `false`. Using these conditions in MemoryStorage silently returns no results. Tests using these conditions need `DatabaseStorage`.
- **`removeObject` and `updateObject` assume `Condition.Equals` on `id`** — directly casts `request.getCondition()` to `Condition.Equals` without type check. Passing any other condition type causes `ClassCastException`.
- **Not thread-safe for writes** — `HashMap` is not concurrent. Tests are single-threaded; production uses `DatabaseStorage`.
- **Permission queries return correct results** but use `stream().anyMatch()` (O(n) per check); fine for tests, not production.

## Line index

- 38-39 — field declarations (objects, permissions maps)
- 43-48 — constructor with Server seed
- 58-62 — `getObjectsStream`
- 64-108 — `checkCondition` recursive evaluator
- 110-117 — `retrieveValue` (reflection-based getter invocation)
- 120-123 — `addObject`
- 127-149 — `updateObject` (setter-based property copy)
- 152-155 — `removeObject`
- 157-159 — `getPermissionsSet` helper
- 162-170 — `getPermissions`
- 173-180 — `addPermission` / `removePermission`
