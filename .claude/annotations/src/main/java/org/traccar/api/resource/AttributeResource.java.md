# AttributeResource.java

**Role:** REST resource at `/api/attributes/computed` for JEXL computed attributes. Extends ExtendedObjectResource for CRUD. Adds a test endpoint that evaluates an expression against the latest position of a device. All write operations restricted to admins.
**Fits in:** Entity: `Attribute` (tc_attributes). Admin-only for writes.
**Read next:** [[ExtendedObjectResource.java]], `ComputedAttributesProvider.java` (JEXL evaluator), `Attribute.java` model

## Public API

- `POST /api/attributes/computed/test?deviceId=N` — admin only; evaluates expression against device's latest position; returns result or 204 if null.
- All CRUD from base class restricted to admin via override.

## Gotchas / non-obvious

- The `test` endpoint uses `cacheManager.addDevice` + `removeDevice` in try/finally to set up the cache context needed by `ComputedAttributesProvider.compute`.
- Type cast for result: number/boolean returned as-is; others as `toString()`.

## Line index

- 46 — extends `ExtendedObjectResource<Attribute>`
- 59-83 — `POST /test` (JEXL evaluation)
- 85-103 — admin-gated add/update/remove overrides
