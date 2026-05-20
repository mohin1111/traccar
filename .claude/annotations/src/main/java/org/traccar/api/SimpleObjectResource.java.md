# SimpleObjectResource.java

**Role:** Adds a `GET /` list endpoint to [[BaseObjectResource.java]] supporting only user-level filtering (`?all`, `?userId`). Simpler than [[ExtendedObjectResource.java]] — no groupId/deviceId cross-filtering.
**Fits in:** Extended by `CalendarResource`, `GroupResource`, `OrderResource`, `ReportResource`. Used for entities that are purely user-owned with no device/group cross-linkage needed.
**Read next:** [[ExtendedObjectResource.java]] (richer variant), [[BaseObjectResource.java]] (CRUD)

## Public API

- `get(all, userId, excludeAttributes, limit, offset, keyword)` → `GET /` (line 45)

## Key flows

### Filtering logic (lines 53-65)
1. `?all=true`: admin sees everything; non-admin adds self-permission condition.
2. `?userId=N`: `checkUser` then filter by N's permissions (no `excludeGroups` — groups included for user-level queries here).
3. Default (`userId=0`): uses the authenticated user's own ID.

## Gotchas / non-obvious

- Unlike `ExtendedObjectResource`, no `?groupId` or `?deviceId` filter — these entities don't have device/group cross-links meaningful for list queries.
- `userId = getUserId()` default assignment (line 61) — when `userId==0` and not `all`, query filters by the requester's own userId rather than an explicit check.

## Line index

- 39 — constructor (sortField, searchColumns)
- 45-74 — `get()` implementation
- 53-65 — `?all` / `?userId` / default branch
- 67-69 — keyword search
- 71-73 — query execution
