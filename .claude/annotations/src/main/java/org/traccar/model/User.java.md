# User.java

**Role:** Represents a Traccar user account. `@StorageName("tc_users")` → `tc_users`. The tenancy root: every device, group, geofence, etc. is accessible only via `tc_user_*` link tables. Implements `UserRestrictions` (capability caps) and `Disableable` (account enable/expiry).
**Fits in:** Central auth entity. `SessionResource`/`LoginService` authenticate against it. `PermissionsService` checks `administrator` flag. `ManagedUser` is a same-table proxy used in `tc_user_user` link rows.
**Read next:** [[ManagedUser.java]] (permission proxy), [[UserRestrictions.java]] (restriction interface), [[Disableable.java]], [[Permission.java]] (link-table mechanics), [[ExtendedModel.java]] (parent)

## Public API

### DB-mapped fields (`tc_users` columns)
- `name String` (line 31) — display name.
- `login String` (line 41) — LDAP login / alternate login identifier.
- `email String` (line 51) — primary login credential; trimmed; **unique** in DB.
- `phone String` (line 61) — optional; used for SMS notifications.
- `readonly boolean` (line 71) — from `UserRestrictions`; user cannot write via API.
- `administrator boolean` (line 82) — superuser; bypasses all permission checks.
- `map String` (line 98) — preferred map tile provider key.
- `latitude/longitude/zoom` (lines 108-129) — map view state (user's last map center).
- `coordinateFormat String` (line 138) — `"dd"` / `"ddm"` / `"dms"`.
- `disabled boolean` (line 148) — from `Disableable`; blocks login.
- `expirationTime Date` (line 160) — from `Disableable`; account expires.
- `deviceLimit int` (line 172) — max devices this user can register; `-1` = unlimited.
- `userLimit int` (line 183) — max managed sub-users; `0` = regular user (cannot manage); non-zero = manager.
- `deviceReadonly boolean` (line 193) — from `UserRestrictions`; cannot edit device config.
- `limitCommands boolean` (line 203) — from `UserRestrictions`; cannot send arbitrary commands.
- `disableReports boolean` (line 213) — from `UserRestrictions`.
- `fixedEmail boolean` (line 223) — from `UserRestrictions`; cannot change own email.
- `poiLayer String` (line 233) — custom POI layer URL.
- `totpKey String` (line 243) — TOTP secret for 2FA; stored in DB.
- `temporary boolean` (line 253) — marks temporary/shared sessions.

### Not DB-mapped (`@QueryIgnore`)
- `getPassword()` always returns `null` (line 267) — password is write-only.
- `setPassword(String)` (line 271) — hashes via `Hashing.createHash()`, stores into `hashedPassword` + `salt`.
- `hashedPassword String` (line 280) — `@JsonIgnore @QueryIgnore` — never exposed via API, read separately for auth.
- `salt String` (line 292) — `@JsonIgnore @QueryIgnore`.
- `getManager()` (line 84) — computed: `userLimit != 0`.

### Key methods
- `isPasswordValid(String)` (line 306) — delegates to `Hashing.validatePassword`.
- `compare(User other, String... exclusions)` (line 310) — reflection equality ignoring hashed creds + named attribute keys; used to detect meaningful config changes.

## Key flows

### Auth flow
`LoginService.login(email, password)` → loads `User` from DB → `user.isPasswordValid(password)` → session or reject.

### Role determination
Three effective roles derived from `User` fields:
1. `administrator == true` → full system access.
2. `userLimit != 0` → manager; can create/manage sub-users (limited by `userLimit`).
3. Neither → regular user; sees only explicitly linked entities.

### Device limit enforcement
`DeviceResource.add()` calls `PermissionsService.checkDeviceLimit(userId)` → counts `tc_user_device` rows for user, compares against `deviceLimit`.

## Gotchas / non-obvious

- `getHashedPassword()` and `getSalt()` are `@QueryIgnore` but the underlying columns **do exist** in `tc_users` (`hashedpassword`, `salt`). They are loaded by a specialized query in `LoginService`, not by the generic `getObject(User.class)` path.
- `userLimit = 0` means "not a manager" (NOT "unlimited managers"). `-1` for `deviceLimit` means unlimited devices. The semantics are intentionally asymmetric.
- `getManager()` (line 84) is `@QueryIgnore @JsonIgnore` — it is a pure computed convenience method, not in any API response or DB column.

## Line index

- 28 — `@StorageName("tc_users")`
- 29 — `class User extends ExtendedModel implements UserRestrictions, Disableable`
- 82-96 — administrator + getManager() computed
- 172-181 — deviceLimit (-1 = unlimited)
- 183-192 — userLimit (0 = normal, non-zero = manager)
- 266-303 — password handling (write-only, hashed)
- 306-308 — isPasswordValid
- 310-320 — compare (with exclusions)
