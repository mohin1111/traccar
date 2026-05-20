# ServerResource.java

**Role:** REST resource at `/api/server` for global server configuration. Read is public; write is admin-only. Also exposes utility endpoints: reverse geocoding, timezone list, file upload (for custom branding), GC trigger, cache dump, reboot.
**Fits in:** Entity: `Server`. Extends BaseResource directly.
**Read next:** [[BaseResource.java]], `Server.java` model, `CacheManager.java`

## Public API

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `GET` | `/api/server` | `@PermitAll` | Server config (augmented with runtime capability flags) |
| `PUT` | `/api/server` | Admin | Update server settings |
| `GET` | `/api/server/geocode?latitude=&longitude=` | Required | On-demand reverse geocode |
| `GET` | `/api/server/timezones` | Required | All Java timezone IDs |
| `POST` | `/api/server/file/{path}` | Admin | Upload branding file to web root |
| `GET` | `/api/server/gc` | Admin | System.gc() trigger |
| `GET` | `/api/server/cache` | Admin | CacheManager debug dump |
| `POST` | `/api/server/reboot` | Admin | System.exit(130) restart |

## Key flows

`GET /api/server` (lines 96-112): loads `Server` from DB, decorates with runtime flags (`emailEnabled`, `textEnabled`, `geocoderEnabled`, `openIdEnabled`, `openIdForce`). For admin: adds `storageSpace`. For unauthenticated: adds `newServer` flag (used to show first-run wizard).

`POST /api/server/file/{path}` (lines 142-163): path traversal prevention via `outputPath.startsWith(rootPath)` check. Creates parent directories if needed.

## Gotchas / non-obvious

- `GET /api/server` is `@PermitAll` — the web UI calls it before login to learn server capabilities and whether registration is open.
- `System.exit(130)` on reboot (line 184) — relies on the host process manager (systemd, Docker restart policy) to restart the JVM.

## Line index

- 65 — class extends `BaseResource`
- 95-112 — `GET /api/server`
- 114-123 — `PUT /api/server`
- 125-133 — geocode
- 135-138 — timezones
- 140-163 — file upload
- 165-178 — gc/cache/reboot
