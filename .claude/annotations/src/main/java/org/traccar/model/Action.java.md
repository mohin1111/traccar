# Action.java

**Role:** Audit log entry for user actions on the system. `@StorageName("tc_actions")` → `tc_actions`. Records who (userId/userEmail), when (actionTime), what (actionType), and which object (objectType + objectId) was affected.
**Fits in:** Extends `ExtendedModel`. Written by `AuditResource` or directly by service layer on create/update/delete operations. Read via `GET /api/actions` (admin only).
**Read next:** [[ExtendedModel.java]] (parent), [[ObjectOperation.java]] (action type enum values)

## Public API

### DB-mapped fields
- `actionTime Date` (line 26) — default `new Date()` at construction; timestamp of the action.
- `address String` (line 36) — client IP address at time of action.
- `userId long` (line 46) — FK to `tc_users.id`; who performed the action.
- `actionType String` (line 56) — action string: typically `"add"`, `"update"`, `"delete"`.
- `objectType String` (line 66) — the entity class name (e.g., `"Device"`, `"User"`).
- `objectId long` (line 76) — the entity's `id`.

### Not DB-mapped
- `userEmail String` (line 86) — `@QueryIgnore`; populated from user join at query time, not stored.

## Line index

- 23 — `@StorageName("tc_actions")`
- 24 — `class Action extends ExtendedModel`
- 26 — `actionTime = new Date()` (default)
- 56-64 — actionType
- 66-74 — objectType
- 76-84 — objectId
- 86-93 — userEmail (@QueryIgnore)
