# UserSecurityContext.java

**Role:** JAX-RS `SecurityContext` implementation. Wraps a [[UserPrincipal.java]] and installed by [[SecurityRequestFilter.java]] via `requestContext.setSecurityContext(...)`. Jersey then injects it via `@Context SecurityContext` into [[BaseResource.java]].
**Fits in:** Created in [[SecurityRequestFilter.java]] on successful auth. Consumed by [[BaseResource.java]] `getUserId()`.
**Read next:** [[UserPrincipal.java]] (the principal it wraps), [[SecurityRequestFilter.java]] (creates it), [[BaseResource.java]] (reads it)

## Public API

- `getUserPrincipal()` (line 31) — returns the wrapped `UserPrincipal`.
- `isUserInRole(String)` (line 36) — always returns `true` (role-based access not used; permission-graph used instead).
- `isSecure()` (line 41) — returns `false` (TLS termination handled upstream/by Jetty; not tracked here).
- `getAuthenticationScheme()` (line 46) — returns `BASIC_AUTH` constant (informational only).

## Gotchas / non-obvious

- `isUserInRole` always true — Traccar does not use JAX-RS role annotations (`@RolesAllowed`). All access control goes through `PermissionsService` methods called explicitly in resource methods.
