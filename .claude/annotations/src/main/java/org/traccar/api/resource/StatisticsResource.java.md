# StatisticsResource.java

**Role:** REST resource at `/api/statistics` for server usage statistics. Admin-only. Returns `Statistics` records (daily counters of positions, devices, users, requests) for time range.
**Fits in:** Entity: `Statistics` (tc_statistics). Extends BaseResource directly.
**Read next:** [[BaseResource.java]], `Statistics.java` model, `StatisticsManager.java` (collects the counters)

## Public API

- `GET /api/statistics?from=&to=` — admin only; streams `Statistics` objects ordered by `captureTime`.
