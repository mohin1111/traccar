# DatabaseStorage.java

**Role:** The production `Storage` implementation. Translates `Request(Columns, Condition, Order)` into SQL strings, uses [[QueryBuilder.java]] for parameter binding and result-set hydration via reflection. Handles permissions via subquery generation. Detects DB type at construction to emit dialect-specific SQL.
**Fits in:** Bound as `@Singleton` by [[DatabaseModule.java]]; injected everywhere via `Storage`. Sits above [[QueryBuilder.java]] (which owns the JDBC level) and below API resource classes.
**Read next:** [[QueryBuilder.java]] (JDBC binding + RS hydration), [[storage/query/Condition.java]] (the condition hierarchy it formats), [[DatabaseModule.java]] (Guice provider)

## Public API

All methods implement `Storage` abstract methods. The key four:

- `getObjectsStream(Class<T>, Request)` → `Stream<T>` — builds SELECT; returns lazy stream keeping JDBC open
- `addObject(T, Request)` → `long` — INSERT; returns generated key
- `addObjects(List<T>, Request)` → `List<Long>` — batch INSERT via `addBatch`/`executeBatch`
- `updateObject(T, Request)` — UPDATE with condition
- `removeObject(Class<?>, Request)` — DELETE with condition
- `getPermissions(ownerClass, ownerId, propertyClass, propertyId)` — SELECT from junction table
- `addPermission(Permission)` / `removePermission(Permission)` — junction-table INSERT/DELETE

## Key flows

### SELECT path (lines 72-103)
1. Build `"SELECT * | col1, col2 FROM <table>"` using `getStorageName(clazz)` (reads `@StorageName`).
2. Append `formatCondition(request.getCondition())` → SQL fragment with `?` placeholders.
3. Append `formatOrder(request.getOrder())` → `ORDER BY col [DESC] [LIMIT n OFFSET m]`.
4. Create `QueryBuilder`; bind condition values positionally via `builder.setValue(index, value)`.
5. Return `builder.executeQueryStreamed(clazz)` — **lazy stream; QueryBuilder closes itself in stream's `onClose`**.
6. If QueryBuilder creation fails, close it in `finally`; the stream retains the builder reference and nulls it on success (line 90 `builder = null`) to prevent double-close.

### INSERT path (lines 106-115)
1. `getColumns(entity.getClass(), "get")` — all getter-based column names excluding `@QueryIgnore`.
2. `formatInsert` builds `INSERT INTO <table>(col1,col2) VALUES (?,?)`.
3. `builder.setObject(entity, columns)` — reflection over getters; maps types to JDBC set methods; JSON-serializes complex types.
4. `builder.executeUpdate()` returns generated key.

### Batch INSERT path (lines 118-136)
Reuses one `QueryBuilder` / one `PreparedStatement`. Per-entity: `setObject` + `addBatch`. Final `executeBatch` returns all generated keys. Validates count matches input size.

### UPDATE path (lines 150-167)
Column values bound first (positions 0..n-1), then condition values appended at positions n..m. This ordering is critical — condition comes **after** SET values in the `?` positional sequence.

### Condition formatting (lines 302-369)
Recursive `formatCondition` converts the `Condition` sealed hierarchy to SQL strings:
- `Compare` → `col op ?`
- `Between` → `col BETWEEN ? AND ?`
- `Binary (And/Or)` → recursive with parentheses for Or
- `Permission` → `id IN (SELECT ... FROM junction_table WHERE key = ? [UNION ...group expansion...])`
- `Contains` → `(LOWER(col1) LIKE ? OR LOWER(col2) LIKE ?)`
- `LatestPositions` → `deviceId = ? AND id IN (SELECT positionId FROM tc_devices [WHERE id = ?])`

### Permission sub-query (lines 399-481)
The `formatPermissionQuery` method is the most complex in the file. For group-enabled entity types (`GroupedModel` subclasses), it emits a 3-part UNION query:
1. Direct `tc_user_X` link
2. Group-level link through `tc_user_group` + recursive group ancestry CTE (via self-JOINs up to 2 levels)
3. If devices, expand group → devices via `tc_devices.groupId`

This allows "user has access to device via group membership" without any application-level expansion.

### Order formatting (lines 371-397)
Dialect-aware: MSSQL uses `OFFSET n ROWS FETCH FIRST m ROWS ONLY`; all others use `LIMIT m [OFFSET n]`. DB type detected once at construction from `connection.getMetaData().getDatabaseProductName()`.

## Gotchas / non-obvious

- **`builder = null` trick (line 90)** — the stream's `onClose` lambda will call `builder.close()`. If `executeQueryStreamed` succeeds, the local `builder` reference is nulled out so the `finally` block (which would close on exception) doesn't double-close.
- **Condition variables are extracted in the same recursive order as `formatCondition`** — `getConditionVariables` (lines 254-292) must traverse the condition tree in the same order the SQL placeholders appear. If you add a new `Condition` type, update both methods in parallel.
- **`column.endsWith("Id")` → `nullIfZero = true`** (line 165 in QueryBuilder, triggered from `setObject`) — foreign key fields named `*Id` are stored as SQL NULL when value is 0, preserving referential integrity.
- **Streamed results hold a live connection** — forgetting try-with-resources on the returned stream leaks JDBC connections from HikariCP pool.
- `getPermissions` (line 186) uses `Condition.merge()` to AND-chain optional owner/property conditions, then calls `builder.executePermissionsQuery()` which returns raw `Permission` objects (not entity hydration).

## Line index

- 52-62 — constructor; DB type detection
- 64-68 — `getObjects` (wraps stream)
- 72-103 — `getObjectsStream` (main SELECT path)
- 106-115 — `addObject`
- 118-136 — `addObjects` (batch INSERT)
- 138-147 — `formatInsert` helper
- 150-167 — `updateObject`
- 170-183 — `removeObject`
- 186-209 — `getPermissions`
- 212-227 — `addPermission`
- 230-244 — `removePermission`
- 246-252 — `getStorageName` (@StorageName lookup)
- 254-292 — `getConditionVariables` (positional binding)
- 294-296 — `formatColumns` (JOIN with commas)
- 298-369 — `formatCondition` (SQL fragment generator)
- 371-397 — `formatOrder` (dialect-aware ORDER BY + LIMIT)
- 399-481 — `formatPermissionQuery` (group-expansion sub-query)
