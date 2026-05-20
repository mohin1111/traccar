# BaseResource.java

**Role:** Root base class for all 24 REST resource classes. Provides the two injected dependencies every resource needs: `Storage` (DB access) and `PermissionsService` (auth/permission checks), plus `getUserId()` which extracts the authenticated user ID from the JAX-RS `SecurityContext`.
**Fits in:** Extended directly by non-CRUD resources (`SessionResource`, `PositionResource`, `EventResource`, `PermissionsResource`, `ServerResource`, `StatisticsResource`, `AuditResource`, `VideoStreamResource`, `HealthResource`, `ShareResource`, `PasswordResource`). CRUD resources extend [[BaseObjectResource.java]] which extends this.
**Read next:** [[BaseObjectResource.java]] (adds CRUD endpoints), [[security/PermissionsService.java]] (the `permissionsService` field), [[security/UserPrincipal.java]] (extracted from SecurityContext)

## Public API

- `getUserId()` (line 37) — returns the authenticated user's ID from `securityContext.getUserPrincipal()`. Returns `0` if unauthenticated (only relevant on `@PermitAll` endpoints).
- `storage` (line 33) — `@Inject protected` — direct DB access for all subclasses.
- `permissionsService` (line 36) — `@Inject protected` — permission/auth checks.

## Gotchas / non-obvious

- `securityContext` is `@Context private` — injected per-request by Jersey from the context set by [[security/SecurityRequestFilter.java]].
- **`getUserId()` returns `0` for unauthenticated requests** on `@PermitAll` endpoints. Resources using this in permission checks must handle the 0 case.
- `PermissionsService` is `@RequestScoped` (one instance per request), while `Storage` is a singleton. Both are safe for multi-threaded use at the resource level.

## Line index

- 28-29 — `securityContext` injection
- 32-33 — `storage` injection
- 35-36 — `permissionsService` injection
- 37-43 — `getUserId()` implementation
