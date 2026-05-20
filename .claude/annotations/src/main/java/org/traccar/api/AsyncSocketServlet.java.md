# AsyncSocketServlet.java

**Role:** Jetty `JettyWebSocketServlet` that handles the WebSocket upgrade for `/api/socket`. Authenticates the upgrading request (token param or session cookie) and, if valid, returns a new [[AsyncSocket.java]] instance.
**Fits in:** Registered as a servlet in `WebServer.java` at path `/api/socket`. Acts as the factory for all WebSocket connections.
**Read next:** [[AsyncSocket.java]] (the handler returned), [[security/LoginService.java]] (used for Bearer token auth on upgrade), `SessionHelper.java` (session origin validation)

## Public API

- `configure(JettyWebSocketServletFactory)` (line 59) — overrides Jetty hook; sets idle timeout and registers the WebSocket creator lambda.

## Key flows

### WebSocket upgrade (lines 61-78)
1. Check if `token` query parameter present → call `loginService.login(token)` → extract `userId`.
2. Else check `SessionHelper.isSessionOriginValid(req)` (CSRF origin check) and read `userId` from HTTP session attribute.
3. If `userId != null` → return `new AsyncSocket(...)`.
4. Else → return `null` (Jetty rejects the upgrade with 403).

### Idle timeout (line 60)
Configured from `Keys.WEB_TIMEOUT` config value (milliseconds). Default in Traccar config is typically 15 minutes.

## Gotchas / non-obvious

- **Token auth on WebSocket** — WebSocket upgrade requests cannot carry custom headers in browser JS. The `?token=...` query parameter is the workaround for Bearer-token clients (mobile apps, scripted clients).
- **Session origin validation** — `SessionHelper.isSessionOriginValid` checks the `Origin` or `Referer` header against the server's configured origin. Prevents CSRF on WS upgrades from foreign origins.
- **`@Singleton`** — the servlet is a singleton; the `AsyncSocket` instances it creates are per-connection (not singletons).

## Line index

- 39 — `@Singleton` + class declaration
- 59 — `configure()` entry
- 60 — idle timeout from config
- 61-78 — creator lambda: token → session → null fallback
