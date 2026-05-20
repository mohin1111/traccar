# CorsResponseFilter.java

**Role:** JAX-RS `ContainerResponseFilter` that adds CORS headers to every API response. Allows cross-origin browser access to the API. Configured via `Keys.WEB_ORIGIN`.
**Fits in:** Registered as a Jersey provider in the application config. Runs after every resource method.
**Read next:** `Config.java` + `Keys.java` (the `WEB_ORIGIN` key)

## Public API

- `filter(request, response)` (line 44) — adds `Access-Control-Allow-*` headers if not already set.

## Key flows

- If `WEB_ORIGIN` is `null` or `"*"`: responds with `Access-Control-Allow-Origin: *`.
- If `WEB_ORIGIN` contains the request's `Origin` header value: echoes it back (allows specific origins).
- `Access-Control-Allow-Credentials: true` — required for cookie-based auth across origins.
- Headers are: `origin, content-type, accept, authorization` / Methods: `GET, POST, PUT, DELETE, OPTIONS`.

## Gotchas / non-obvious

- Uses `if (!response.getHeaders().containsKey(...))` — existing headers win. Resources can set their own CORS headers if needed.
- `Access-Control-Allow-Credentials: true` combined with a wildcard origin is technically invalid per spec (browsers block it). The `WEB_ORIGIN` config should be set to a specific origin in production with credential-bearing requests.

## Line index

- 39-41 — constants (ORIGIN_ALL, HEADERS_ALL, METHODS_ALL)
- 44-65 — `filter()` body
- 61 — origin matching logic
