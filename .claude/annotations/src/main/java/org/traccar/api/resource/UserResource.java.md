# UserResource.java

**Role:** REST resource for `tc_users` at `/api/users`. Manages user creation (including self-registration and manager-created sub-users), listing, update, delete, and TOTP key generation.
**Fits in:** Entity: `User`. Extends [[BaseObjectResource.java]]. Overrides `POST` with registration logic.
**Read next:** [[BaseObjectResource.java]] (base CRUD), [[security/PermissionsService.java]] (`checkUserUpdate` for privilege escalation prevention), `UserUtil.java`

## Public API (endpoints)

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `GET` | `/api/users` | Required | List users (admin: all; manager: sub-users; self: empty unless `?userId`) |
| `GET` | `/api/users/{id}` | Required | Single user (base class) |
| `POST` | `/api/users` | `@PermitAll` | Register/create user |
| `PUT` | `/api/users/{id}` | Required | Update user (base class) |
| `DELETE` | `/api/users/{id}` | Required | Delete user; self-delete clears session |
| `POST` | `/api/users/totp` | `@PermitAll` | Generate a new TOTP secret key |

## Key flows

### `GET /api/users` (lines 73-96)
- `?userId=N`: lists sub-users of N (manager/admin).
- Admin (no userId): list all users.
- Non-admin (no userId): lists only managed sub-users via `tc_user_user`.
- `?deviceId=N`: lists users who have access to device N (manager+ only).
- `?keyword=X`: searches name + email.

### `POST /api/users` — registration flow (lines 101-142)
Complex branching based on who is calling:
1. **Admin creating**: no restrictions, sets whatever they specify.
2. **Manager creating sub-user** (`currentUser.userLimit != 0`): checks manager's sub-user count against `userLimit`; links new user via `tc_user_user`.
3. **Self-registration** (`currentUser == null` or `currentUser` is not admin/manager):
   - If DB empty (first user ever): sets `administrator=true` on the new user.
   - Else: checks `server.registration` setting; throws if disabled.
   - Checks `WEB_TOTP_FORCE` — if TOTP is forced, `totpKey` must be provided.
   - Applies `UserUtil.setUserDefaults` (default language, timezone etc. from config).
4. Always: insert user row, then second update for `hashedPassword`+`salt`.

### `DELETE /api/users/{id}` (lines 144-152)
Calls `super.remove(id)`, then if the deleted user is the current requester: removes `userId` from the session (self-delete = auto-logout).

### `POST /api/users/totp` (lines 154-163)
`@PermitAll` — generates a random TOTP key using `GoogleAuthenticator.createCredentials()`. Client stores this key and sets it on the user via PUT. Server doesn't store it here; the PUT to update the user with `totpKey` is the commit step.

## Gotchas / non-obvious

- **`@PermitAll` on `POST /api/users`** — registration must be reachable before login. If `server.registration=false`, the server enforces the restriction but the endpoint is still reachable (returns 403 via SecurityException).
- **First-user bootstrap** (line 117) — `UserUtil.isEmpty(storage)` checks if `tc_users` has any rows. The very first POST creates an admin regardless of what the client sends for `administrator`.
- **TOTP force check** (line 122) — server attribute `web.totpForce=true` means any new user must supply a TOTP key at registration time.
- **Two DB writes on create** (lines 130-133) — first without password, then with `hashedPassword`+`salt`. The `id` is needed before hashing can be stored.

## Line index

- 57 — class + extends `BaseObjectResource<User>`
- 73-96 — `GET /api/users` (list with filters)
- 101-142 — `POST /api/users` (overrides base; registration logic)
- 144-152 — `DELETE /api/users/{id}` (self-delete logout)
- 154-163 — `POST /api/users/totp`
