# ShareResource.java

**Role:** REST resource at `/api/share` for generating temporary shareable Bearer tokens for a Device or Group. Creates a synthetic temporary user with read-only access, then issues a token for that user.
**Fits in:** Extends BaseResource. Uses TokenManager for token generation.
**Read next:** [[BaseResource.java]], [[signature/TokenManager.java]], `User.java` (temporary user fields)

## Public API

- `POST /api/share/device` — form params: `deviceId`, `expiration`. Returns Bearer token string.
- `POST /api/share/group` — form params: `groupId`, `expiration`. Returns Bearer token string.

## Key flows

`share(user, clazz, id, expiration)` (lines 56-101):
1. Check sharing not disabled (`Keys.DEVICE_SHARE_DISABLE`).
2. Check caller is not already a temporary user.
3. Cap expiration to caller's own token expiration if shorter.
4. Verify caller has access to the object.
5. Construct a synthetic email: `user@email:deviceUniqueId` or `user@email:group:groupId`.
6. Find or create a temporary User with `readonly=true`, `temporary=true`, the access permission row.
7. Generate token for the temporary user's ID.

## Gotchas / non-obvious

- Share users are real `tc_users` rows with synthetic emails — they persist until explicitly deleted or expire.
- `limitCommands` and `disableReports` on the share user inherit from the creating user's settings, potentially further restricted by config keys `WEB_SHARE_DEVICE_COMMANDS`/`WEB_SHARE_DEVICE_REPORTS`.
- Calling the endpoint again for the same object reuses the existing share user (idempotent).

## Line index

- 56-101 — `share` private method (core logic)
- 103-110 — `POST /share/device`
- 112-119 — `POST /share/group`
