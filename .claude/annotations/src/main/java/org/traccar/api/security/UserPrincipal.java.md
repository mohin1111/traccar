# UserPrincipal.java

**Role:** JAX-RS `Principal` implementation. Carries the authenticated `userId` and optional token `expiration`. Stored in `UserSecurityContext` and retrieved by `BaseResource.getUserId()`.
**Fits in:** Created in [[SecurityRequestFilter.java]] on successful auth. Read by [[BaseResource.java]] `getUserId()`.
**Read next:** [[UserSecurityContext.java]] (wraps this), [[BaseResource.java]] (unwraps via `securityContext.getUserPrincipal()`)

## Public API

- `getUserId()` → long (line 32)
- `getExpiration()` → Date (line 36) — null for session-based auth; non-null for token-based.
- `getName()` → null (line 41) — `Principal` interface; not used.
