# `storage/` — Custom JDBC "ORM"

No Hibernate. No JPA. A hand-rolled reflection-based persistence layer using HikariCP + raw JDBC, with Liquibase for schema migrations. The design is intentionally minimal: one annotation (`@StorageName`) maps a class to a table; another (`@QueryIgnore`) excludes methods from column scanning; everything else is derived from JavaBean getter/setter conventions via `ReflectionCache`.

## File index

| File | One-liner | Annotation |
|---|---|---|
| `Storage.java` | Abstract CRUD + permission contract | [Storage.java.md](Storage.java.md) |
| `DatabaseStorage.java` | Production impl: SQL builder + reflection execution | [DatabaseStorage.java.md](DatabaseStorage.java.md) |
| `MemoryStorage.java` | In-memory impl for tests | [MemoryStorage.java.md](MemoryStorage.java.md) |
| `QueryBuilder.java` | JDBC connection + PreparedStatement wrapper; reflection hydration | [QueryBuilder.java.md](QueryBuilder.java.md) |
| `StorageName.java` | `@StorageName("tc_xxx")` — class-to-table mapping annotation | [StorageName.java.md](StorageName.java.md) |
| `QueryIgnore.java` | `@QueryIgnore` — excludes a getter from column scan | [QueryIgnore.java.md](QueryIgnore.java.md) |
| `StorageException.java` | Checked exception wrapping SQL/logic failures | [StorageException.java.md](StorageException.java.md) |
| `DatabaseModule.java` | Guice provider: HikariCP setup + Liquibase migration | [DatabaseModule.java.md](DatabaseModule.java.md) |
| `query/Request.java` | Immutable query spec bundling Columns + Condition + Order | [query/Request.java.md](query/Request.java.md) |
| `query/Columns.java` | Column strategy: All / Include / Exclude | [query/Columns.java.md](query/Columns.java.md) |
| `query/Condition.java` | WHERE predicate hierarchy (9 types) | [query/Condition.java.md](query/Condition.java.md) |
| `query/Order.java` | ORDER BY + LIMIT/OFFSET (dialect-aware) | [query/Order.java.md](query/Order.java.md) |

## Architecture: how `@StorageName` + reflection replace an ORM

```
Model class (e.g. Device.java)
  @StorageName("tc_devices")     ← table name
  getId() / setId()              ← column "id"
  getName() / setName()          ← column "name"
  @QueryIgnore getStatus()       ← NOT a column
         ↓
DatabaseStorage.getObjectsStream(Device.class, request)
  → getStorageName(Device.class) → "tc_devices"
  → formatCondition(request.getCondition()) → "WHERE id = ?"
  → QueryBuilder.create(..., "SELECT * FROM tc_devices WHERE id = ?")
  → ReflectionCache.getProperties(Device.class, "set") → {id: handle, name: handle, ...}
  → bind result set columns to setter handles by lowercased name
  → stream of Device objects
```

No mapping files, no XML, no annotations on fields — just class-level `@StorageName`, method-level `@QueryIgnore`, and JavaBean naming conventions.

## Supported databases

All five are bundled in `build.gradle` and handled by dialect branches in `DatabaseStorage.formatOrder`:

| DB | Notes |
|---|---|
| H2 | Default for development (`debug.xml`); in-memory mode |
| MySQL | Production primary; uses `LIMIT … OFFSET …` |
| MariaDB | Identical SQL to MySQL |
| PostgreSQL | Same `LIMIT … OFFSET …` syntax |
| Microsoft SQL Server | Uses `OFFSET n ROWS FETCH FIRST m ROWS ONLY` |

DB type is detected once at `DatabaseStorage` construction via `connection.getMetaData().getDatabaseProductName()`.

## The `Request` / `Columns` / `Condition` / `Order` query model

Every `Storage` method takes a `Request`. Reading from left to right:

```
Request(Columns.Exclude("id"), Condition.Equals("id", 42), new Order("fixTime", true, 100))
        └─ which columns          └─ WHERE clause              └─ ORDER BY + LIMIT
```

`DatabaseStorage` disassembles the Request and assembles the SQL string using three formatters:
- `formatColumns` — comma-joined column list
- `formatCondition` — recursive condition tree → SQL fragment + `?` placeholders
- `formatOrder` — ORDER BY/LIMIT/OFFSET with dialect switch

Then `QueryBuilder` binds values and executes.

## Liquibase migration

Runs on every startup before the Guice injector is fully initialized (inside `DatabaseModule.provideDataSource`). Master changelog: `schema/changelog-master.xml`. Current tip: `schema/changelog-6.13.0.xml`.

- `liquibase.clearCheckSums()` is called before each `update` — handles checksum drift across Liquibase versions.
- Lock timeout: 1 minute. If the lock is stuck from a previous crash, run: `UPDATE DATABASECHANGELOGLOCK SET locked = 0`.
- **All schema changes must go through Liquibase changesets.** Never run DDL directly against the DB.

## Permission model

Permissions use junction tables named `tc_<owner>_<property>` (e.g., `tc_user_device`, `tc_group_geofence`). These tables have no `@StorageName` entity class; they are queried via `Storage.getPermissions` / `addPermission` / `removePermission` using the `Permission` model class which derives the table name from class names.

`Condition.Permission` generates a correlated subquery that transparently expands group membership (up to 2 levels deep via UNION + self-JOIN) so a user with access to a group automatically sees all group-member devices.

## How to add a new entity

1. Create `model/Foo.java extends BaseModel` (or `ExtendedModel`, `GroupedModel`).
2. Annotate with `@StorageName("tc_foos")`.
3. Add getters/setters for each column. Apply `@QueryIgnore` to computed getters.
4. Create a Liquibase changeset in `schema/changelog-next.xml` with `CREATE TABLE tc_foos (...)`.
5. Reference `storage.addObject(foo, new Request(new Columns.Exclude("id")))` to insert.
6. Create API resource class in `api/resource/FooResource.java` if REST exposure is needed.

## How to add a new query condition

1. Add inner class in `Condition.java` implementing `Condition`.
2. Add `case NewCondition ->` branch in `DatabaseStorage.formatCondition` (SQL fragment).
3. Add corresponding `case NewCondition ->` in `DatabaseStorage.getConditionVariables` (bind values, in the same positional order).
4. Add `case NewCondition ->` in `MemoryStorage.checkCondition` for test support (or document limitation).
