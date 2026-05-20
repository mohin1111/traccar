# `storage/query/` — Query value objects

The four immutable value objects that form a query specification. Callers construct a `Request` combining `Columns`, `Condition`, and `Order`, then pass it to any `Storage` method. No SQL is written in this package — that lives in `DatabaseStorage`.

## File index

| File | One-liner | Annotation |
|---|---|---|
| `Request.java` | Bundles Columns + Condition + Order into one query argument | [Request.java.md](Request.java.md) |
| `Columns.java` | Specifies which columns: All, Include(whitelist), Exclude(blacklist) | [Columns.java.md](Columns.java.md) |
| `Condition.java` | WHERE predicate hierarchy: Equals, Compare, Between, And, Or, Permission, Contains, LatestPositions | [Condition.java.md](Condition.java.md) |
| `Order.java` | ORDER BY col [DESC] [LIMIT n [OFFSET m]] | [Order.java.md](Order.java.md) |

## Dependency graph

```
Request
 ├── Columns (All / Include / Exclude)
 ├── Condition (Equals / Compare / Between / And / Or / Permission / Contains / LatestPositions)
 └── Order
```

All four are pure value objects with no dependencies on storage infrastructure.

## Condition hierarchy (the most important type)

```
Condition (interface)
 ├── Compare
 │    └── Equals  (shorthand for Compare with "=")
 ├── Between
 ├── Binary
 │    ├── And
 │    └── Or
 ├── Permission   (generates sub-SELECT with optional group-expansion UNION)
 ├── Contains     (LOWER(col) LIKE '%val%' across multiple columns)
 └── LatestPositions  (filters to device's current positionId in tc_devices)
```

## Conventions

- All constructors take plain types (String column names, Object values, Class references) — no dependency on model layer except `Condition.Permission` which references `GroupedModel`.
- Column names passed to conditions are the **camelCase JavaBean property names** (e.g., `"deviceId"`, `"fixTime"`), not SQL column names. `ReflectionCache` and `DatabaseStorage` handle the mapping.
- `Condition.merge(List)` is the idiomatic way to AND-chain a variable-length list of conditions. Returns `null` for empty list (= no WHERE clause).

## How to add a new condition type

1. Add a new inner class implementing `Condition` in `Condition.java`.
2. Add a `case MyCondition condition ->` branch in `DatabaseStorage.formatCondition` to emit the SQL fragment.
3. Add a corresponding `case MyCondition condition ->` in `DatabaseStorage.getConditionVariables` to extract bind values in the same positional order as the SQL fragment.
4. Add the same case in `MemoryStorage.checkCondition` for test support (or document that `MemoryStorage` does not support it).
