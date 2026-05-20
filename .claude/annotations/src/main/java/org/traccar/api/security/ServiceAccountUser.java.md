# ServiceAccountUser.java

**Role:** A synthetic `User` subclass used when the API is called with the service account token (`Keys.WEB_SERVICE_ACCOUNT_TOKEN`). Has `administrator=true`, a sentinel ID (`9000000000000000000L`), and no DB row. Used for machine-to-machine automation.
**Fits in:** Returned by [[LoginService.java]] `login(token)` when the token matches `serviceAccountToken`. Checked in [[BaseObjectResource.java]] `add()` to skip owner-link creation.
**Read next:** [[LoginService.java]] (creates this), [[PermissionsService.java]] (`getUser` handles this ID)

## Public API

- `ID = 9000000000000000000L` (line 22) — static constant; used as a sentinel in `BaseObjectResource` and `PermissionsService`.

## Gotchas / non-obvious

- **No DB row** — `PermissionsService.getUser(ID)` returns `new ServiceAccountUser()` directly (line 65 in PermissionsService) without a DB query.
- `getAdministrator()` returns true (set in constructor) — all permission checks pass.
- **Never linked as owner** — `BaseObjectResource.add()` checks `getUserId() != ServiceAccountUser.ID` before creating the owner Permission row (line 84). This means objects created by the service account are not owned by any user — admins must grant access explicitly.
