# MediaFilter.java

**Role:** Servlet `Filter` (not a JAX-RS filter) that guards the `/api/media/` path. Checks the HTTP session for an authenticated userId, then verifies the user has permission to access the device whose `uniqueId` is in the URL path. Only session-cookie auth is supported (no Bearer/Basic).
**Fits in:** Registered as a servlet filter in `WebServer.java` at path `/api/media/*`. Runs before Jersey.
**Read next:** [[security/PermissionsService.java]] (`checkPermission`), `MediaManager.java` (serves the media files), `StatisticsManager.java` (request counting)

## Public API

- `doFilter(request, response, chain)` (line 59) — authentication + device permission check; passes through on success, sends 401/403 on failure.

## Key flows

1. Validate session origin (`SessionHelper.isSessionOriginValid`).
2. Extract `userId` from HTTP session; register request stats.
3. Parse path: `/api/media/{uniqueId}/...` — extract `parts[1]` as device uniqueId.
4. Lookup device by `uniqueId`; call `permissionsService.checkPermission(Device.class, userId, device.getId())`.
5. On pass: `chain.doFilter`. On fail: 403.

## Gotchas / non-obvious

- **Session-only auth** — Bearer tokens and Basic auth are not checked here. Media URLs are meant for browser playback where session is established.
- **`Provider<PermissionsService>`** (line 48) — `PermissionsService` is request-scoped in Guice servlet context; using a `Provider` ensures a fresh instance per filter invocation.
- `SecurityException` from `checkPermission` is caught and mapped to 403 (not re-thrown as 401).

## Line index

- 43 — `@Singleton` filter
- 59-95 — `doFilter` body
- 65-70 — session userId extraction + stats
- 79-87 — device lookup and permission check
