# DatabaseModule.java

**Role:** Guice `AbstractModule` that provides the singleton `DataSource`. Performs three tasks at startup: optional JDBC driver loading from external JAR, HikariCP pool configuration, and Liquibase schema migration.
**Fits in:** Installed in `MainModule.java` during app boot. Produces the `DataSource` singleton consumed by [[DatabaseStorage.java]] (via `QueryBuilder`) and [[DatabaseStorage.java]] constructor.
**Read next:** [[DatabaseStorage.java]] (consumes `DataSource`), `config/Keys.java` (all config key constants referenced here)

## Public API

### `provideDataSource(Config)` → `DataSource` (lines 45-112)
`@Singleton @Provides` Guice method. Called once at startup. Throws checked `ReflectiveOperationException`, `IOException`, `LiquibaseException`.

## Key flows

### Driver loading (lines 49-61)
If `Keys.DATABASE_DRIVER_FILE` is set, dynamically adds an external JAR to the classloader using `addURL` (JDK 8) or `appendToClassPathForInstrumentation` (JDK 9+ workaround). This allows deploying with a custom JDBC driver JAR not bundled in `target/lib/`.

### HikariCP pool config (lines 63-81)
- Driver class name, JDBC URL, user, password from config keys
- `connectionInitSql` for connection validation (e.g., `SET NAMES utf8mb4` for MySQL)
- `idleTimeout = 600000ms` (10 min) hardcoded
- `maxLifetime` from `Keys.DATABASE_MAX_LIFETIME` (0 = HikariCP default)
- `maximumPoolSize` from `Keys.DATABASE_MAX_POOL_SIZE`

### Liquibase migration (lines 83-109)
Only runs if `Keys.DATABASE_CHANGELOG` is set (non-empty). Steps:
1. Resolve absolute path; use parent dir as `DirectoryResourceAccessor`.
2. Set system properties to cap lock wait at 1 minute and disable analytics.
3. Open Liquibase `Database` connection (separate from HikariCP pool).
4. Call `liquibase.clearCheckSums()` then `liquibase.update(new Contexts())` — applies all pending changesets.
5. If `LockException` caught → throw `DatabaseLockException` with instructions to run `UPDATE DATABASECHANGELOGLOCK SET locked = 0`.

### `DatabaseLockException` (lines 116-123)
Package-private `RuntimeException` with a human-readable message explaining how to unlock the Liquibase lock table. Thrown during startup so the server refuses to start with a stale lock.

## Gotchas / non-obvious

- **Migration blocks startup** — Liquibase runs synchronously in `provideDataSource`; server does not accept connections until migration completes.
- **`liquibase.clearCheckSums()`** is called before `update` — this forces Liquibase to re-validate checksums on every startup. Needed because checksum algorithms change between Liquibase versions.
- **Two separate connections for Liquibase and HikariCP** — Liquibase opens its own connection via `DatabaseFactory.openDatabase`, separate from the pool. The pool is created first; Liquibase runs against the same URL independently.
- **Lock timeout is 1 minute** — if a previous server process died mid-migration, the lock stays. The `DatabaseLockException` message gives the fix: manual SQL UPDATE to release the lock.
- **Supported DBs** are not determined here — they depend on which JDBC drivers are bundled in `build.gradle` (H2, MySQL, MariaDB, PostgreSQL, MSSQL).

## Line index

- 41 — class declaration (`AbstractModule`)
- 43-46 — `@Singleton @Provides` method signature
- 49-61 — external driver JAR loading
- 63-81 — HikariCP configuration
- 83-109 — Liquibase migration
- 106-108 — `LockException` → `DatabaseLockException`
- 116-123 — `DatabaseLockException` class
