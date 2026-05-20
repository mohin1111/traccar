# CopyAttributesHandler.java

**Role:** Carries forward selected attributes from the previous position into the current one when the device does not report them. Implements a "sticky attribute" pattern for attributes like sensor readings that do not change every fix.
**Fits in:** Extends `BasePositionHandler`. Step 15 in the pipeline (after `DriverHandler`, before `EngineHoursHandler`). Reads config key `PROCESSING_COPY_ATTRIBUTES`.
**Read next:** [[BasePositionHandler.java.md]], [[DriverHandler.java.md]]

## Public API

### `onPosition(Position, Callback)` (lines 36-48)
- Looks up `PROCESSING_COPY_ATTRIBUTES` (a comma/space-separated attribute name list) per device via `AttributeUtil.lookup`.
- Gets the last stored position from `CacheManager`.
- For each attribute name: copies the value from `last` → `position` only if `last` has it **and** `position` does not already have it (non-destructive).

## Key flows

1. Config value `PROCESSING_COPY_ATTRIBUTES = "fuel,door"` (example) set at server/group/device level.
2. On each position, if the device did not report `fuel` but the last position had it, the value carries forward.
3. `callback.processed(false)` — never filters.

## Gotchas / non-obvious

- **One-way, non-destructive** — only copies when the incoming position is missing the attribute. Device-reported values always win.
- **`last` may be `null`** (first ever position for device) — guarded at line 40; no copy happens.
- **Attribute list is per-device via `AttributeUtil.lookup`** — can be overridden at group or server level.

## Line index

- 37-38 — look up `PROCESSING_COPY_ATTRIBUTES` and last position
- 40-47 — conditional per-attribute copy loop
