# HealthResource.java

**Role:** REST resource at `/api/health` (actually at `/api/server/health` when registered — check WebServer). @PermitAll health check endpoint for load balancers. Checks message throughput and DB connectivity.
**Fits in:** Extends BaseResource. Used by monitoring infrastructure.
**Read next:** [[BaseResource.java]], `StatisticsManager.java` (message count source)

## Public API

- `GET /api/health` — `@PermitAll`; returns `200 OK` (text `"OK"`) or `500` if checks fail.

## Key flows

`checkMessages()` (lines 65-78): compares message count delta between calls; if ratio < `Keys.WEB_HEALTH_CHECK_DROP_THRESHOLD`, throws (500). Stateful with class-level static fields.

`checkDatabase()` (lines 80-83): loads the `Server` row — any DB error causes 500.

## Gotchas / non-obvious

- `messageLastTotal` and `messageLastCheck` are static — shared across all requests. Not thread-safe in strict sense but benign (both monotonically increasing; occasional race is acceptable for health checks).
- First call always passes (messageLastCheck = 0 → ratio check skipped).
- Threshold = 0 disables the message drop check.

## Line index

- 39 — class + @Path("health")
- 54-63 — `health()` endpoint
- 65-78 — `checkMessages` (drop ratio)
- 80-83 — `checkDatabase`
