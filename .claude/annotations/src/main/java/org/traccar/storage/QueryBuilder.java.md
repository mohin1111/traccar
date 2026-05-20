# QueryBuilder.java

**Role:** The JDBC execution layer. Owns a `Connection` + `PreparedStatement` pair for the lifetime of one query. Reflection-driven setter binding (`setObject`) and result-set hydration (`executeQueryStreamed`). Implements `AutoCloseable` so callers use try-with-resources.
**Fits in:** Created by [[DatabaseStorage.java]] for every query; never exposed to layers above `DatabaseStorage`. Relies on `ReflectionCache` for cached `MethodHandle` lookups.
**Read next:** [[DatabaseStorage.java]] (sole creator), [[storage/query/Columns.java]] (determines which getters are iterated)

## Public API

### Factory methods (lines 82-91)
- `QueryBuilder.create(config, dataSource, objectMapper, query)` — no generated keys
- `QueryBuilder.create(..., query, returnGeneratedKeys: true)` — for INSERT with key return

### Parameter binding
- `setBoolean/setInteger/setLong/setDouble/setString/setDate/setBlob` — 0-indexed wrappers over `PreparedStatement.setX(index+1, ...)`
- `setLong(index, value, nullIfZero)` — when `nullIfZero=true` and value=0, sets SQL NULL (used for FK columns ending in `Id`)
- `setValue(index, Object)` — dispatch switch over type; no JSON fallback
- `setObject(Object, List<String> columns)` — **the main binding method**; iterates column names, resolves getter via `ReflectionCache`, dispatches to typed setter; JSON-serializes unknown types via `ObjectMapper`

### Execution
- `executeQueryStreamed(Class<T>)` → `Stream<T>` — lazy stream; **keeps connection open until stream closed**
- `executeUpdate()` → `long` — runs DML; returns generated key if `returnGeneratedKeys=true`
- `addBatch()` / `executeBatch()` → `List<Long>` — batch INSERT support
- `executePermissionsQuery()` → `List<Permission>` — reads raw `LinkedHashMap` columns into `Permission` objects

### Lifecycle
- `close()` — closes `statement` then `connection`; called in stream `onClose` or try-with-resources

## Key flows

### `setObject` reflection binding (lines 153-181)
For each column name:
1. Get `ReflectionCache.Property` for `get+column` on the entity class.
2. Read `property.type()` (return type of getter).
3. Invoke `property.handle().invokeExact(object)` → raw value.
4. Dispatch to typed `setX` based on return type.
5. Fallback: `objectMapper.writeValueAsString(value)` → `setString` (handles `Map<String,Object>` attributes, enums, etc.).

### `executeQueryStreamed` RS hydration (lines 237-307)
1. Execute query; get `ResultSetMetaData` → build `columnIndexes` map (lowercased label → 1-based index).
2. For each setter property in `ReflectionCache.getProperties(clazz, "set")`: look up column index; build a `ResultSetProcessor<T>` lambda that reads the right JDBC type.
3. Create a `Spliterator` that calls `resultSet.next()` and constructs a new instance via `ReflectionCache.getConstructor(clazz)` per row.
4. Stream is lazy — JDBC cursor advances only as elements are consumed.
5. `onClose` → `resultSet.close()` then `this.close()`.

### Logging (lines 231-235)
If `Keys.LOGGER_QUERIES` is enabled in config, logs the raw SQL at INFO level via SLF4J. Useful for debugging but can be noisy; disabled by default.

## Gotchas / non-obvious

- **0-indexed parameters externally, 1-indexed in JDBC** — all `setX` methods do `index + 1`. Pass 0-based indices from callers.
- **`executeQueryStreamed` must be called at most once** — `PreparedStatement.executeQuery()` can only be called once per `PreparedStatement`. After the stream is consumed/closed, the `QueryBuilder` is also closed.
- **Date → `Timestamp` mapping** (line 129): `new Timestamp(value.getTime())` — `Date` is stored as `TIMESTAMP` in SQL. NULL dates are stored as SQL NULL.
- **ResultSet column matching is case-insensitive** (line 246: `toLowerCase(Locale.ROOT)`) — column aliases in SQL must match property names when lowercased. H2 uppercases column names; this normalization handles that.
- **Timestamp null check on read** (line 213): if the DB column is NULL, `resultSet.getTimestamp()` returns null; the processor skips `invokeExact` entirely, leaving the field at its default (null for `Date`).
- **Complex type JSON round-trip** (line 175 write / line 224 read): any getter return type not in the primitive/String/Date/byte[] list is serialized as JSON on write and deserialized via `objectMapper.readValue(value, parameterType)` on read. This is how `Map<String,Object> attributes` is persisted.

## Line index

- 62-80 — private constructor (connection + prepareStatement)
- 82-91 — static factory methods
- 93-151 — typed `setX` methods
- 153-181 — `setObject` (reflection-based batch binding)
- 183-229 — `ResultSetProcessor` interface + `addProcessors` builder
- 231-235 — `logQuery`
- 237-307 — `executeQueryStreamed` (lazy stream hydration)
- 309-316 — `close`
- 318-330 — `executeUpdate` + generated-key return
- 331-347 — `addBatch` / `executeBatch`
- 349-364 — `executePermissionsQuery`
