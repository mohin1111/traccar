# Order.java

**Role:** Immutable value object specifying SQL ORDER BY with optional DESC, LIMIT, and OFFSET. The pagination/sorting component of [[Request.java]].
**Fits in:** Optional component of [[Request.java]]; consumed by [[DatabaseStorage.java]] `formatOrder` which emits dialect-appropriate SQL (MSSQL vs all others).
**Read next:** [[Request.java]], [[DatabaseStorage.java]] (formatOrder, lines 371-397)

## Public API

### Constructors
- `Order(String column)` — ascending, no limit/offset
- `Order(String column, boolean descending, int limit)` — with limit, no offset
- `Order(String column, boolean descending, int limit, int offset)` — full control

### Accessors
- `getColumn()` — the column name to sort by
- `getDescending()` — true → append `DESC`
- `getLimit()` — 0 = no limit; > 0 → append `LIMIT n` (or MSSQL equivalent)
- `getOffset()` — 0 = no offset; used for pagination

## Key flows

### Standard pagination pattern
```java
new Order("fixTime", true, 100, 200)
// → ORDER BY fixTime DESC LIMIT 100 OFFSET 200   (MySQL/PostgreSQL/H2)
// → ORDER BY fixTime DESC OFFSET 200 ROWS FETCH FIRST 100 ROWS ONLY   (MSSQL)
```

## Gotchas / non-obvious

- **MSSQL requires `OFFSET 0 ROWS` even when no offset is wanted** — `DatabaseStorage.formatOrder` unconditionally emits `OFFSET <offset> ROWS` for MSSQL when limit > 0. Always pass an explicit offset (even 0) when using limits with MSSQL.
- **0 limit = no LIMIT clause** — passing `limit=0` means unlimited results. This is intentional for queries that return all rows.
- Column name is inserted directly into SQL without quoting — must be a valid DB column name, not user input.

## Line index

- 25-38 — constructors
- 40-54 — accessors
