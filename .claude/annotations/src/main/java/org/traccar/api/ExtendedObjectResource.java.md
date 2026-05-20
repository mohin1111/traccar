# ExtendedObjectResource.java

**Role:** Adds a rich `GET /` list endpoint to [[BaseObjectResource.java]] with `?all`, `?userId`, `?groupId`, `?deviceId`, `?keyword`, `?excludeAttributes`, `?limit`, `?offset` filtering. Used by resources that need cross-entity filtering (by group or device).
**Fits in:** Extended by `AttributeResource`, `CommandResource`, `DriverResource`, `GeofenceResource`, `MaintenanceResource`, `NotificationResource`. Contrast with [[SimpleObjectResource.java]] which only supports user-level filtering.
**Read next:** [[SimpleObjectResource.java]] (simpler variant), [[BaseObjectResource.java]] (CRUD methods), `Condition.Permission` (the join-table query mechanism)

## Public API

- `get(all, userId, groupId, deviceId, excludeAttributes, limit, offset, keyword)` → `GET /` (line 47)

## Key flows

### Filtering logic (lines 54-84)
1. **`?all=true`**: admin sees everything; non-admin adds `Condition.Permission(User, self, EntityClass)`.
2. **`?userId=N`**: `checkUser(getUserId(), N)` then filter by `N`'s permissions (excludeGroups = direct links only).
3. **`?groupId=N`**: checkPermission on group, then add group-linked condition.
4. **`?deviceId=N`**: checkPermission on device, then add device-linked condition.
5. **`?keyword=X`**: `Condition.Contains(searchColumns, X)` — SQL `LIKE %X%` on configured columns.
6. Conditions are AND-merged; `Order` applies sort + pagination.

### Sort and pagination
`sortField` is set by each subclass constructor (e.g., `"name"` for GeofenceResource). `limit`/`offset` map to SQL LIMIT/OFFSET. Default sort field falls back to `"id"` if null.

## Gotchas / non-obvious

- **`excludeGroups()`** on `Condition.Permission` — when querying by `userId`, group-inherited permissions are excluded (direct links only). This matches the UI expectation that "user's geofences" means explicitly linked, not inherited through group membership.
- **`groupId` and `deviceId` can be combined** — both conditions AND-merged, so caller gets intersection.

## Line index

- 40 — constructor (sortField, searchColumns)
- 47-85 — `get()` implementation
- 56-67 — `?all` / `?userId` branch
- 69-75 — `?groupId` + `?deviceId` conditions
- 78-80 — keyword search
- 82-84 — column selection + order + query
