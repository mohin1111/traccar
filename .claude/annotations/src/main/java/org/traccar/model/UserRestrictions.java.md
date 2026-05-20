# UserRestrictions.java

**Role:** Interface capturing the five capability-restriction flags shared by both `User` (per-user caps) and `Server` (global defaults). `PermissionsService` evaluates both and applies OR logic (most restrictive wins).
**Fits in:** Implemented by `User` and `Server`. `PermissionsService.checkEdit()`, `checkAdmin()`, etc. query these flags to determine what an authenticated user may do.
**Read next:** [[User.java]], [[Server.java]]

## Public API

Five boolean accessors (no setters defined in interface — implementations provide them):
- `getReadonly()` — user can only read, not write via API.
- `getDeviceReadonly()` — user cannot edit device configuration.
- `getLimitCommands()` — user cannot send arbitrary commands to devices.
- `getDisableReports()` — user cannot generate or view reports.
- `getFixedEmail()` — user cannot change their own email address.

## Gotchas / non-obvious

- When `Server.limitCommands == true`, ALL non-admin users are affected, regardless of individual `User.limitCommands`.
- These are additive restrictions — there is no "grant override" mechanism at individual user level if `Server` restricts it.

## Line index

- 18 — `public interface UserRestrictions`
- 19-23 — five boolean getter declarations
