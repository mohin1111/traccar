# BaseModel.java

**Role:** Root of every persisted domain object. Holds the single `long id` field that maps to the DB primary key.
**Fits in:** All 39 model classes ultimately extend `BaseModel` (directly or via `ExtendedModel`/`GroupedModel`/`Message`). The custom ORM in `DatabaseStorage`/`QueryBuilder` reflects on `getId()`/`setId()` to read/write the PK column.
**Read next:** [[ExtendedModel.java]] (adds attributes map), [[GroupedModel.java]] (adds groupId), [[Permission.java]] (does not extend BaseModel but interacts with BaseModel subclasses)

## Public API

- `long id` (line 21) — surrogate PK; 0 means "not yet persisted".
- `getId()` / `setId(long)` (lines 23-28)

## Key flows

### ORM mapping
`QueryBuilder` iterates JavaBean getters; `getId()` maps to the `id` column in the `@StorageName` table. After INSERT, `setId()` is called with the generated key.

## Gotchas / non-obvious

- `id = 0` is a valid sentinel meaning "not yet assigned" — never store 0 as a real FK without checking.
- `Permission` uses `BaseModel` subclasses as type tokens in its static `CLASSES` map, but `Permission` itself does NOT extend `BaseModel`.

## Line index

- 19 — class declaration
- 21 — `private long id`
- 23-25 — `getId()`
- 27-29 — `setId()`
