# PermissionsService.java

**Role:** The permission-enforcement service. Every resource method that touches an entity calls one or more methods here. Implements the graph-based access model: admin full access, manager via `tc_user_user`, regular user via `tc_user_device` etc. `@RequestScoped` — one instance per HTTP request, caches User and Server within the request.
**Fits in:** Injected into all resource classes via [[BaseResource.java]]. Called by [[SecurityRequestFilter.java]] (to check disabled status). The most-called service in the entire API layer.
**Read next:** [[BaseResource.java]] (the `permissionsService` field), [[BaseObjectResource.java]] (calls `checkEdit`, `checkPermission`), `Permission.java` (the permission row model), `ManagedUser.java` (User→User proxy)

## Public API

- `getServer()` (line 54) — lazy-loads and caches the `Server` config row.
- `getUser(userId)` (line 62) — lazy-loads and caches the authenticated user; handles `ServiceAccountUser.ID` sentinel.
- `notAdmin(userId)` (line 74) — returns true if not administrator; used for conditional filtering.
- `checkAdmin(userId)` (line 78) — throws `SecurityException` if not admin.
- `checkManager(userId)` (line 84) — throws if not admin AND not a manager (userLimit != 0).
- `checkRestriction(userId, callback)` (line 94) — checks a `UserRestrictions` lambda against both server and user settings; used for `readonly`, `disableReports`, `limitCommands`.
- `checkEdit(userId, clazz, addition, skipReadonly)` (line 102) — checks write access; enforces device limits on addition.
- `checkEdit(userId, entity, addition, skipReadonly)` (line 127) — object-level variant; also validates group hierarchy and calendar/command references.
- `checkUser(userId, managedUserId)` (line 171) — admin OR self OR manager with `tc_user_user` row.
- `checkUserUpdate(userId, before, after)` (line 180) — guards privilege escalation on user PUT (admin flags, expiration time, restrictions).
- `checkPermission(clazz, userId, objectId)` (line 212) — THE core permission check: admin short-circuits, else queries the join table for a row linking `(userId → objectId)`.

## Key flows

### `checkPermission` (lines 212-227) — the most critical method
1. If user is admin → return immediately.
2. If checking `User.class` and `userId == objectId` → allow (users always access themselves).
3. Otherwise: query `storage.getObject(queryClass, ...)` with `Condition.And(Equals(id, objectId), Permission(User, userId, queryClass))`. If null → throw `SecurityException`.
4. Special: `LinkedDevice.class` is queried as `Device.class` (proxy pattern for device-device links).

### `checkEdit` — device limit enforcement (lines 109-115)
When adding a Device: if `user.deviceLimit > 0`, counts existing linked devices. If `count >= limit` → deny. `-1` means unlimited.

### `checkUserUpdate` — privilege escalation prevention (lines 180-210)
Fields that require admin: `administrator`, `deviceLimit`, `userLimit`, `expirationTime` (extension beyond caller's own expiry).
Fields that require manager or self: `readonly`, `deviceReadonly`, `disabled`, `limitCommands`, `disableReports`, `fixedEmail`.
`fixedEmail` users cannot change their own email (line 207).

## Gotchas / non-obvious

- **`@RequestScoped`** — User and Server are cached per-request. If the DB is modified mid-request (e.g., in the same transaction), the cached values won't reflect it.
- **`ServiceAccountUser`** always returns `getAdministrator() = true` — all `checkAdmin` calls pass for service account.
- **`ManagedUser` proxy** — querying user-to-user permissions uses `ManagedUser.class` which maps to `tc_user_user` table via `@StorageName`. Regular `User.class` maps to `tc_users`.
- **`checkPermission` does a DB query every time** — not cached in the current request. For endpoints that check the same permission multiple times (rare), this is repeated DB work.

## Line index

- 54-59 — `getServer()` lazy load
- 62-73 — `getUser()` lazy load + ServiceAccount
- 74-99 — `notAdmin`, `checkAdmin`, `checkManager`, `checkRestriction`
- 102-125 — `checkEdit(clazz)` — device limits + command limits
- 127-168 — `checkEdit(entity)` — group/calendar/command reference validation
- 171-178 — `checkUser` (admin/self/manager check)
- 180-210 — `checkUserUpdate` privilege escalation guards
- 212-227 — `checkPermission` (the core join-table lookup)
