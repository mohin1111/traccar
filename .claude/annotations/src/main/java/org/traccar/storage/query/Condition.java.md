# Condition.java

**Role:** Sealed condition hierarchy representing SQL WHERE clause predicates. Nine inner classes cover equality, comparison, range, boolean composition (AND/OR), full-text contains, permission-based subquery, and latest-positions subquery. The most important type in the query layer.
**Fits in:** Component of [[Request.java]]; consumed by [[DatabaseStorage.java]] `formatCondition` and `getConditionVariables`. [[MemoryStorage.java]] evaluates conditions in Java.
**Read next:** [[DatabaseStorage.java]] (SQL generation), [[Request.java]], [[Order.java]]

## Public API

### `Condition.merge(List<Condition>)` (lines 24-34)
Static helper: AND-chains a list of conditions. Returns null if list is empty, a single condition if one element, nested `And` nodes for more.

### Condition hierarchy

| Type | SQL output | Parameters |
|---|---|---|
| `Equals(column, value)` | `col = ?` | 1 |
| `Compare(column, op, value)` | `col op ?` where op ∈ `<, <=, >, >=, =` | 1 |
| `Between(column, from, to)` | `col BETWEEN ? AND ?` | 2 |
| `And(first, second)` | `first AND second` | sum of parts |
| `Or(first, second)` | `(first OR second)` | sum of parts |
| `Contains(columns, value)` | `(LOWER(c1) LIKE ? OR LOWER(c2) LIKE ?)` | N (one per column) |
| `Permission(ownerClass, ownerId, propertyClass)` | `id IN (SELECT ... FROM junction WHERE key=? [UNION group-expansion])` | 1 or 2 |
| `LatestPositions(deviceId)` | `deviceId=? AND id IN (SELECT positionId FROM tc_devices WHERE id=?)` | 2 (or 0/1 if deviceId=0) |

### `Condition.Permission` detail (lines 126-175)
Two constructors for the two directions of a permission join:
- `Permission(ownerClass, ownerId, propertyClass)` — "give me all X owned by user Y"
- `Permission(ownerClass, propertyClass, propertyId)` — "give me all users who own X with id Z"

`getIncludeGroups()` (line 170-174): returns `true` when either side is a `GroupedModel` subclass AND `excludeGroups()` was not called. When true, `DatabaseStorage` emits the 3-level UNION group-expansion query.

`excludeGroups()` returns a new `Permission` with `excludeGroups=true` — used in contexts where group-transitive access should be ignored.

### `Condition.LatestPositions` (lines 195-209)
Filters to only the positions currently stored in `tc_devices.positionId` (the device's latest position). Zero `deviceId` means "all devices"; non-zero scopes to one device. Also optionally filters by `fixTime` if `DATABASE_POSITION_PERIOD` config is set.

## Key flows

### Building conditions at call sites
```java
// Get device by ID
new Condition.Equals("id", deviceId)

// Get positions for device in time range
new Condition.And(
    new Condition.Equals("deviceId", deviceId),
    new Condition.Between("fixTime", from, to)
)

// Get devices user can see (with group expansion)
new Condition.Permission(User.class, userId, Device.class)

// Search devices by name or uniqueId
new Condition.Contains(List.of("name", "uniqueId"), searchText)

// Get current positions
new Condition.LatestPositions()
```

## Gotchas / non-obvious

- **`Or` wraps in parentheses, `And` does not** (lines 321-328 in DatabaseStorage) — because AND has higher precedence, Or must parenthesize to be safe.
- **`Contains` value is lowercased and wrapped in `%…%`** in `DatabaseStorage.getConditionVariables` — the `value` stored here is raw; the `%` wrapping and lowercasing happen at binding time.
- **`Permission` with both `ownerId > 0` and `propertyId > 0` is not supported** — one must be 0. The constructor enforces this by only taking one ID value.
- **`LatestPositions` with `deviceId=0`** may optionally prepend a `fixTime > ?` filter if `DATABASE_POSITION_PERIOD` config is non-zero — this prevents returning ancient positions for devices that haven't reported recently.

## Line index

- 24-34 — `merge` static helper
- 36-63 — `Equals`, `Compare` classes
- 66-88 — `Between` class
- 90-124 — `Binary`, `And`, `Or` classes
- 126-175 — `Permission` class (most important)
- 177-193 — `Contains` class
- 195-209 — `LatestPositions` class
