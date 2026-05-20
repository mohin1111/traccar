# ResourceErrorHandler.java

**Role:** JAX-RS `ExceptionMapper<Exception>` — global exception handler for the REST API. Converts any uncaught exception into an HTTP response whose body is the full stack trace as a string.
**Fits in:** Registered as a Jersey provider. Catches all exceptions not handled by more specific mappers.
**Read next:** No strong dependencies; purely infrastructure.

## Public API

- `toResponse(exception)` (line 28) — returns the original response status if the exception is a `WebApplicationException`; otherwise returns 400 BAD_REQUEST. Body is always the stack trace string.

## Gotchas / non-obvious

- **Stack traces in HTTP responses** — exposes internal implementation details to API callers. Acceptable for an internal tool like Traccar; in a multi-tenant production context consider sanitizing.
- `WebApplicationException` preserves the original HTTP status (e.g., 404, 403) while wrapping the stack trace as the body entity. Useful for debugging.
- `SecurityException` (thrown by `PermissionsService`) is NOT a `WebApplicationException`, so it lands here as 400. Callers must check whether they need 403 vs 400 behavior.

## Line index

- 28-38 — `toResponse` body
- 33 — WebApplicationException passthrough
- 36 — fallback 400
