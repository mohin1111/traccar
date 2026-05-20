# LoginResult.java

**Role:** Immutable value object returned by all `LoginService.login(...)` overloads. Carries the authenticated `User` and an optional token `expiration` date.
**Fits in:** Returned by [[LoginService.java]], consumed by [[SecurityRequestFilter.java]] and `SessionResource.java`.
**Read next:** [[LoginService.java]] (produces), [[SecurityRequestFilter.java]] (consumes)

## Public API

- `LoginResult(User user)` (line 12) — no expiration (session-based login).
- `LoginResult(User user, Date expiration)` (line 16) — with expiration (Bearer token login).
- `getUser()` / `getExpiration()` — accessors.

## Gotchas / non-obvious

- `expiration` is null for session-based logins (email+password, LDAP). It is populated only for Bearer token logins where the token carries its own expiry.
- `UserPrincipal` in `SecurityRequestFilter` carries this expiration forward so resources can query it for token-refresh logic.
