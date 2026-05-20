# StorageException.java

**Role:** Checked exception wrapping any persistence failure (SQL errors, missing annotations, key-count mismatches). The single checked exception type in the storage layer.
**Fits in:** Thrown by [[Storage.java]] abstract methods and [[DatabaseStorage.java]]; caught by API resource classes and service layer.
**Read next:** [[Storage.java]] (declares throws clauses), [[DatabaseStorage.java]] (throws sites)

## Public API

Three constructors (lines 20-32):
- `StorageException(String message)` — for logic errors (e.g., "StorageName annotation is missing")
- `StorageException(Throwable cause)` — wraps a `SQLException`
- `StorageException(String message, Throwable cause)` — both

## Gotchas / non-obvious

- **Checked exception** — all `Storage` method signatures declare `throws StorageException`. API resource classes must either propagate or catch and return an error response.
- Wrapping `SQLException` in `StorageException` keeps JDBC details out of the API layer.
- `MemoryStorage` overrides the abstract methods but its signature also says `throws StorageException` even though it never throws — allows a `MemoryStorage` instance to be used wherever `Storage` is expected without cast.
