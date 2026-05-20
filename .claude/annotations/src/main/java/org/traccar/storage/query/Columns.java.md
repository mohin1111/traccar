# Columns.java

**Role:** Specifies which columns appear in a query — analogous to the SELECT column list and the INSERT/UPDATE field list. Three concrete strategies: all, include-list, exclude-list.
**Fits in:** Component of [[Request.java]]; consumed by [[DatabaseStorage.java]] which calls `getColumns(clazz, type)`. Uses `ReflectionCache` to enumerate JavaBean properties.
**Read next:** [[Request.java]], [[DatabaseStorage.java]], [[QueryIgnore.java]] (excludes annotated methods)

## Public API

### Abstract base (lines 26-35)
- `getColumns(Class<?>, String type)` → `List<String>` — `type` is `"get"` for INSERT/UPDATE (getter scan) or `"set"` for SELECT (setter scan).
- `getAllColumns(clazz, type)` — reads `ReflectionCache.getProperties(clazz, type)`, filters out `@QueryIgnore`, returns property names as column names.

### Concrete subclasses

**`Columns.All` (lines 37-42)**
Returns `getAllColumns` — every mapped property. Used for `SELECT *` equivalents when the column names are also needed (e.g., for INSERT).

**`Columns.Include` (lines 44-55)**
Constructor takes vararg `String... columns`. Returns the explicit list regardless of the class. Used when you know exactly which columns to touch:
```java
new Columns.Include("status", "lastUpdate")
```

**`Columns.Exclude` (lines 57-70)**
Constructor takes vararg columns to skip. Returns `getAllColumns` minus the excluded set. Primary use: `new Columns.Exclude("id")` for INSERT (ID is DB-generated).

## Key flows

### INSERT without ID (most common pattern)
```java
new Request(new Columns.Exclude("id"))
// → getColumns(Device.class, "get") → all getter-mapped columns except "id"
// → INSERT INTO tc_devices(name, uniqueId, groupId, ...) VALUES (?,?,?,...)
```

### Selective UPDATE
```java
new Columns.Include("status", "lastUpdate")
// → UPDATE tc_devices SET status = ?, lastUpdate = ? WHERE id = ?
```

## Gotchas / non-obvious

- **`type` parameter is `"get"` for write operations, `"set"` for read** — `ReflectionCache.getProperties(clazz, "get")` scans getters (for binding params); `"set"` scans setters (for hydrating from RS).
- `Columns.All` used in `getObjects` for SELECT also drives the INSERT column list — the same property set.
- `Columns.Include` column names must exactly match the lowercased JavaBean property names (e.g., `"deviceId"` not `"device_id"`). The DB column name matching is done via reflection; the column list is passed as-is into SQL.

## Line index

- 26-35 — abstract class + `getAllColumns`
- 37-42 — `Columns.All`
- 44-55 — `Columns.Include`
- 57-70 — `Columns.Exclude`
