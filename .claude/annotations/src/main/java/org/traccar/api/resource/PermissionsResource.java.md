# PermissionsResource.java

**Role:** REST resource for managing permission rows (link table entries) at `/api/permissions`. POST adds, DELETE removes. Supports bulk operations. The canonical way to link entities (e.g., user→device, group→geofence) in the Traccar permission graph.
**Fits in:** Extends [[BaseResource.java]] directly. Operates on `Permission` objects which are `LinkedHashMap<String, Long>` payloads (not model entities with CRUD lifecycle).
**Read next:** [[BaseResource.java]], `Permission.java` model (how type names map to join tables), [[security/PermissionsService.java]] (`checkPermission` called for both owner and property)

## Public API (endpoints)

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `GET` | `/api/permissions` | Admin only | Query link rows (admin debug tool) |
| `POST` | `/api/permissions` | Required | Add a single link row |
| `POST` | `/api/permissions/bulk` | Required | Add multiple link rows |
| `DELETE` | `/api/permissions` | Required | Remove a single link row |
| `DELETE` | `/api/permissions/bulk` | Required | Remove multiple link rows |

## Key flows

### POST payload format
JSON: `{"userId": 5, "deviceId": 12}` — keys must end in `Id`. `Permission(entity)` constructor parses these key strings via `Permission.getKeyClass(key)` to derive the owner and property classes, then the join table name.

### `checkPermission(permission)` (lines 59-64)
Non-admins must already have access to BOTH the owner and the property object to link them. Prevents privilege escalation (can't grant access to a device you don't own).

### `checkPermissionTypes(entities)` (lines 66-73)
Validates all entries in a bulk request have the same key set (same entity types). Prevents mixed-type bulk requests.

### Cache invalidation (lines 98-101, 126-129)
Every add/remove calls `cacheManager.invalidatePermission(...)` which propagates to connected WebSocket clients so they see immediate permission changes.

## Gotchas / non-obvious

- **`GET /api/permissions` is admin-only** (line 78) — it's a raw join-table query tool for debugging. Normal users only POST/DELETE.
- **`@GET` reads query params** (line 77) — the GET form takes `?userId=5&deviceId=12` as query params (two `*Id` params required) to look up specific permission rows.
- **Non-admins cannot grant access they don't have** (lines 60-63) — the checkPermission in `checkPermission(permission)` ensures the caller owns both sides of the link.
- `actionLogger.link`/`unlink` records the grant/revoke in the audit log.

## Line index

- 48 — class extends `BaseResource`
- 59-64 — `checkPermission` (double-check both sides)
- 66-73 — `checkPermissionTypes` (bulk validation)
- 77-87 — `GET /api/permissions` (admin query)
- 89-108 — `POST /api/permissions/bulk`
- 110-113 — `POST /api/permissions` (single, delegates to bulk)
- 115-134 — `DELETE /api/permissions/bulk`
- 136-139 — `DELETE /api/permissions` (single, delegates)
