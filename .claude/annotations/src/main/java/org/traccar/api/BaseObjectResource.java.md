# BaseObjectResource.java

**Role:** Generic CRUD base for all entity resources. Provides `GET /{id}`, `POST`, `PUT /{id}`, `DELETE /{id}` with permission checks, cache invalidation, audit logging, and automatic owner-linking on creation.
**Fits in:** Extended by [[ExtendedObjectResource.java]] and [[SimpleObjectResource.java]], which add the list (`GET /`) endpoint. All 24 REST resources extend one of these three classes.
**Read next:** [[ExtendedObjectResource.java]] (adds groupId/deviceId-filtered list), [[SimpleObjectResource.java]] (adds simple user-filtered list), [[BaseResource.java]] (parent), [[security/PermissionsService.java]] (all permission checks), `CacheManager.java` (invalidation on write)

## Public API

- `getSingle(id)` → `GET /{id}` (line 65): permission check then storage fetch; 404 if not found.
- `add(entity)` → `POST` (line 78): checkEdit → addObject → add owner-link Permission row → cache/socket invalidation → audit.
- `update(entity)` → `PUT /{id}` (line 96): checkPermission → special User update checks → checkEdit → updateObject → (extra password update if User) → cache invalidation → audit.
- `remove(id)` → `DELETE /{id}` (line 132): checkPermission → checkEdit → removeObject → cache invalidation → audit.

## Key flows

### POST (create) flow (lines 78-91)
1. `permissionsService.checkEdit(userId, entity, addition=true, skipReadonly=false)` — verifies not readonly, respects device limit quotas.
2. `storage.addObject(entity, ...)` — inserts row; entity gets its new `id`.
3. Unless caller is `ServiceAccountUser` (ID=9e18), adds a `Permission(User, userId, EntityClass, entityId)` row so the creator owns the new object.
4. Invalidates cache and notifies `ConnectionManager`.
5. Logs to audit.

### PUT (update) flow — User special-casing (lines 99-128)
- If entity is a `User`: loads before-state, calls `checkUserUpdate` to prevent privilege escalation. Also skips readonly for `notificationTokens`/`termsAccepted` fields (users can update those on themselves).
- If entity is a `Group`: guards against circular group hierarchy (`group.id == group.groupId`).
- After update, if `hashedPassword != null` (password was changed), does a second update targeting only `hashedPassword`+`salt` columns.

### Audit logging
Every write calls `actionLogger.create/edit/remove/link` — writes to `tc_actions` for the audit trail.

## Gotchas / non-obvious

- **`ServiceAccountUser.ID` bypass** (line 84) — service account creates entities without being linked as owner. Useful for automation that shouldn't pollute the user→entity graph.
- **Two-pass password update** (lines 117-122) — password hash is stored in separate columns; Columns.Exclude("id") on the main update would skip hashedPassword, so a second targeted update is needed.
- **`cacheManager.invalidateObject`** propagates to cluster via `BroadcastService` if multi-node.
- **`connectionManager.invalidatePermission`** (line 87) causes connected WebSocket clients to receive updated device lists.

## Line index

- 44 — class declaration (generic `T extends BaseModel`)
- 60 — `baseClass` field (the entity class passed by subclass constructor)
- 65-75 — `getSingle`
- 78-91 — `add` (POST)
- 96-128 — `update` (PUT) with User/Group special-casing
- 132-143 — `remove` (DELETE)
