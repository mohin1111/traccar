# Storage.java

**Role:** Abstract base defining the entire persistence contract — the "interface" every caller programs to. Five CRUD operations plus permission operations.
**Fits in:** Extended by [[DatabaseStorage.java]] (production) and [[MemoryStorage.java]] (tests). Injected via Guice everywhere that touches the DB.
**Read next:** [[DatabaseStorage.java]] (concrete SQL impl), [[DatabaseModule.java]] (Guice wiring), [[storage/query/Request.java]] (the parameter object passed to every method)

## Public API

### Abstract methods (lines 28-52)
- `getObjects(Class<T>, Request)` → `List<T>` — SELECT all matching rows
- `getObjectsStream(Class<T>, Request)` → `Stream<T>` — streaming SELECT (caller must close stream)
- `addObject(T, Request)` → `long` — INSERT; returns generated ID
- `addObjects(List<T>, Request)` → `List<Long>` — batch INSERT (default = loop over `addObject`; `DatabaseStorage` overrides with `addBatch`)
- `updateObject(T, Request)` — UPDATE
- `removeObject(Class<?>, Request)` — DELETE
- `getPermissions(ownerClass, ownerId, propertyClass, propertyId)` → `List<Permission>` — query junction tables
- `addPermission(Permission)` / `removePermission(Permission)` — insert/delete junction-table rows

### Convenience methods (lines 54-64)
- `getPermissions(ownerClass, propertyClass)` — no ID filters (returns all links)
- `getObject(Class<T>, Request)` → `T | null` — shortcut using `getObjectsStream().findFirst()`; **closes the stream automatically** via try-with-resources

## Key flows

### The `getObject` shortcut (lines 60-64)
Opens a stream, finds the first element, closes the stream in the `try` block. Safe pattern — callers never need to close streams from `getObject`.

### Batch insert default (lines 34-40)
`addObjects` loops over `addObject` with no batching. `DatabaseStorage` overrides this with JDBC `addBatch` / `executeBatch` for performance.

## Gotchas / non-obvious

- **Streams from `getObjectsStream` hold a live JDBC connection** — callers must close them (use try-with-resources). `getObjects` does this internally.
- **`getObject` returns `null` if not found** — not Optional. Every call site should null-check.
- `Request` can have null `Condition`, null `Order`, null `Columns` — `DatabaseStorage` handles each null gracefully.

## Line index

- 28-44 — core CRUD abstract methods
- 46-52 — permission abstract methods
- 54-58 — `getPermissions` convenience (no ID filter)
- 60-64 — `getObject` stream-shortcut
