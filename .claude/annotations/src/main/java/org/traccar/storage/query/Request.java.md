# Request.java

**Role:** Immutable parameter object bundling `Columns`, `Condition`, and `Order` into a single argument for all `Storage` method calls. Acts as the query spec — analogous to a query builder output in other ORMs.
**Fits in:** Constructed at every call site (API resources, handlers, managers) and passed to [[Storage.java]] methods. Consumed by [[DatabaseStorage.java]] to build SQL.
**Read next:** [[Columns.java]], [[Condition.java]], [[Order.java]] (the three components)

## Public API

### Constructors (lines 24-44)
- `Request(Columns)` — SELECT all with no filter/order
- `Request(Condition)` — DELETE/UPDATE/SELECT with condition only (null columns)
- `Request(Columns, Condition)` — SELECT with filter
- `Request(Columns, Order)` — SELECT with sort
- `Request(Columns, Condition, Order)` — full query

### Accessors (lines 46-56)
- `getColumns()` → `Columns` — may be null; `DatabaseStorage` uses `*` for null
- `getCondition()` → `Condition` — may be null (no WHERE clause)
- `getOrder()` → `Order` — may be null (no ORDER BY)

## Key flows

### Typical usage patterns
```java
// SELECT * FROM tc_devices WHERE id = 42
new Request(new Columns.All(), new Condition.Equals("id", deviceId))

// INSERT INTO tc_positions (all columns except id)
new Request(new Columns.Exclude("id"))

// SELECT id FROM tc_devices WHERE user_permission JOIN ... ORDER BY name
new Request(new Columns.Include("id"), new Condition.Permission(User.class, userId, Device.class))

// SELECT ... ORDER BY fixTime DESC LIMIT 100
new Request(new Columns.All(), condition, new Order("fixTime", true, 100))
```

## Gotchas / non-obvious

- **Null `Columns` on DELETE** — `removeObject` ignores columns, so `new Request(new Condition.Equals("id", id))` is valid for delete. `DatabaseStorage.removeObject` doesn't call `getColumns()`.
- `Request` is immutable (all fields final) — safe to share across threads, but in practice new instances are created per call.

## Line index

- 24-44 — all constructors
- 46-56 — accessors
