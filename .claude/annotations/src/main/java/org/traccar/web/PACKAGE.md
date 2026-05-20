# `web/` — Jetty web server setup

11 files configuring the embedded Jetty server: HTTP/HTTPS, session management, Jersey JAX-RS, GuiceFilter, static file serving, WebSocket, MCP (Model Context Protocol) endpoint, and request logging.

## File index

| File | One-liner |
|---|---|
| `WebServer.java` | Central Jetty `Server` builder: binds port, sets up handlers, wires GuiceFilter + Jersey |
| `WebModule.java` | Guice `ServletModule` — binds Jersey `ServletContainer` at `/api/*` |
| `WebInjectionManagerFactory.java` | Bridges Jersey's injection with Guice (allows `@Inject` in JAX-RS resources) |
| `WebRequestLog.java` | Jetty `RequestLog` implementation writing to SLF4J via `StatisticsManager.registerRequest` |
| `McpServerHolder.java` | Holds the MCP (Model Context Protocol) server instance for AI tool integration |
| `McpAuthFilter.java` | Servlet filter protecting the MCP endpoint |
| `WellKnownServlet.java` | Serves `.well-known/openid-configuration` for OIDC discovery |
| `ConsoleServlet.java` | Optional debug console servlet |
| `OverrideFileFilter.java` | Intercepts static file requests to allow per-file server-side overrides |
| `OverrideTextFilter.java` | Replaces `${title}`, `${colorPrimary}` etc. placeholders in served HTML/JS |
| `ResponseWrapper.java` | Servlet response wrapper used by `OverrideTextFilter` for in-flight text substitution |

## Dependency graph

```
Main.java → WebServer.start()
WebServer
 ├── Jetty Server (eclipse.jetty)
 ├── CompressionHandler (gzip)
 ├── SessionHandler (in-memory or JDBC session store)
 ├── GuiceFilter → WebModule → Jersey + Guice resources
 ├── ResourceServlet (static files from web.path)
 ├── AsyncProxyServlet (if web.debug.proxyUrl configured)
 └── JettyWebSocketServletContainerInitializer → AsyncSocketServlet

WebRequestLog → StatisticsManager.registerRequest()
McpServerHolder → optional MCP server
```

## Configuration

```xml
<entry key="web.port">8082</entry>
<entry key="web.address">0.0.0.0</entry>        <!-- bind address -->
<entry key="web.path">/opt/traccar/web</entry>  <!-- static files directory -->
<entry key="web.origin">*</entry>               <!-- CORS origin, restrict in production -->
<entry key="web.cacheControl">no-store</entry>  <!-- HTTP cache headers -->
<entry key="web.debug.proxyUrl">http://localhost:3000</entry>  <!-- dev: proxy to Vite -->
```

## Session management

By default Jetty uses in-memory sessions. For clustered deployments, set JDBC session store:
```xml
<entry key="web.session.driver">...</entry>
<entry key="web.session.url">...</entry>
```
`WebServer` detects these keys and configures `JDBCSessionDataStoreFactory` with `DatabaseAdaptor`. Session table is `jettysessions` — created automatically by Jetty.

## Static file serving + placeholder substitution

`ResourceServlet` serves the React build from `web.path`. `OverrideTextFilter` intercepts `index.html` (and `.js` files) to replace:
- `${title}` → `server.attributes.title`
- `${colorPrimary}` → `server.attributes.colorPrimary`
- `${description}` → `server.attributes.description`

This is how the web frontend is re-branded without rebuilding.

## WebSocket

`JettyWebSocketServletContainerInitializer` registers `AsyncSocketServlet` at `/api/socket`. Jetty upgrades the HTTP connection to WebSocket for live position/event streaming.

## MCP endpoint

`McpServerHolder` + `McpAuthFilter` expose a Model Context Protocol endpoint for AI assistant integration. Not part of standard Traccar — present in this fork. Protect with `mcpAuthFilter.token` config key.

## Transport OS note

The `OverrideTextFilter` placeholder mechanism is the correct way to inject server-specific branding (fleet company name, primary color) without React rebuilds — reuse this pattern in the Transport OS frontend build.

For multi-tenant deployments, consider a reverse proxy (nginx/Traefik) in front of Jetty rather than extending `WebServer` — keeps Traccar as a single-tenant service per Transport OS architecture decision.
