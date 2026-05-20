# SecurityRequestFilter.java

**Role:** The authentication gate for every JAX-RS request. A `ContainerRequestFilter` that runs before any resource method. Tries two auth paths (Authorization header or session cookie), sets a `UserSecurityContext` if successful, or rejects with 401 if the endpoint is not `@PermitAll`.
**Fits in:** Registered as a Jersey request filter. The central entry point of the auth system. Works alongside [[LoginService.java]] (credential validation) and [[PermissionsService.java]] (post-auth permission checks in resources).
**Read next:** [[LoginService.java]] (called for Bearer + Basic auth), [[UserSecurityContext.java]] + [[UserPrincipal.java]] (what gets set on success), [[BaseResource.java]] (reads the SecurityContext via `getUserId()`)

## Public API

- `filter(ContainerRequestContext)` (line 62) — sole method; sets `securityContext` or throws 401.

## Key flows

### OPTIONS passthrough (line 64)
`OPTIONS` preflight requests skip auth entirely (required for CORS preflight to work without credentials).

### Authorization header path (lines 72-86)
1. Split `Authorization` header into `[scheme, credentials]`.
2. Delegate to `LoginService.login(scheme, credentials)` which handles `"Bearer"` (token) and `"Basic"` (base64 email:password).
3. On success: register stats, wrap in `UserSecurityContext(UserPrincipal(userId, expiration))`.

### Session cookie path (lines 88-104)
1. `SessionHelper.isSessionOriginValid(request)` — CSRF check (Origin/Referer must match configured origin).
2. Read `userId` and `expiration` from `HttpSession`.
3. If `expiration` is past → invalidate session (forces re-login).
4. Load user via `PermissionsService.getUser(userId)` → call `user.checkDisabled()` (throws if account disabled).
5. Register stats, create `UserSecurityContext`.

### Rejection path (lines 112-124)
If `securityContext == null` and the resource method lacks `@PermitAll`: build 401 response. For HTML-accepting clients (browsers), adds `WWW-Authenticate: Basic realm="api"` to trigger browser basic-auth dialog.

## Gotchas / non-obvious

- **`@PermitAll` is checked on the method** (line 115) — not on the class. A resource class can mix public and protected methods.
- **`injector.getInstance(PermissionsService.class)`** (line 97) — needed because `PermissionsService` is `@RequestScoped` in Guice; cannot be `@Inject` directly into a filter (which has a different scope). This is the one place Guice's `Injector` is used directly.
- **`SecurityException` from `checkDisabled` is caught** (lines 108-110) and logged as a warning; the request falls through to the `@PermitAll` check. A disabled-user hit on a non-public endpoint gets 401, not 403.
- **Two-step expiration check**: session has its own `expiration` attribute (from token-based sessions) separate from the session timeout. Past expiration invalidates session even if the Jetty session is still alive.

## Line index

- 62-127 — `filter()` body
- 64-66 — OPTIONS shortcut
- 72-86 — Authorization header (Bearer + Basic)
- 88-104 — Session cookie path
- 112-124 — 401 rejection + WWW-Authenticate
