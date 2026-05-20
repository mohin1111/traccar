# AuditResource.java

**Role:** REST resource at `/api/audit` for the action audit log. Admin-only. Returns `Action` records with time range filtering, enriched with user email.
**Fits in:** Entity: `Action` (tc_actions). Extends BaseResource directly.
**Read next:** [[BaseResource.java]], `Action.java` model, `LogAction.java` (creates audit records)

## Public API

- `GET /api/audit?from=&to=` — admin only; streams `Action` objects ordered by `actionTime`; each action gets `userEmail` resolved from a pre-loaded user map.

## Key flows

Pre-loads all users into a `Map<Long, String>` (userId → email) using try-with-resources stream to avoid holding a DB cursor while streaming output. Then streams `Action` records in time range.

## Gotchas / non-obvious

- `userEmail` on each Action is resolved in Java, not SQL — a deliberate choice to avoid a JOIN on potentially large `tc_actions` table.
- No pagination — returns full time range. Large ranges may be slow on busy servers.

## Line index

- 41 — class extends `BaseResource`
- 45-63 — `get(from, to)` with user email map
