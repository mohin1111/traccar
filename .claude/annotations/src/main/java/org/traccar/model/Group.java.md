# Group.java

**Role:** Represents a device group. `@StorageName("tc_groups")` → `tc_groups`. Groups nest via `groupId` (a `Group` can have a parent `Group`). Devices belong to a group via `Device.groupId`. Group membership enables inherited attributes, notifications, geofences, and commands to cascade down.
**Fits in:** Extends `GroupedModel` (has its own `groupId` → parent group). One-liner entity: only adds `name`. Linked to users via `tc_user_group`.
**Read next:** [[GroupedModel.java]] (parent), [[Device.java]] (references groupId), [[ExtendedModel.java]] (provides attributes column)

## Public API

### DB-mapped fields (`tc_groups` columns)
- Inherited `groupId long` — FK to parent group; 0 = root group.
- Inherited `attributes AttributeMap` — JSON column; can store group-level device defaults.
- `name String` (line 23) — display name.

## Key flows

### Group chain walk (`CacheManager`)
Starting from a device's `groupId`, `CacheManager.getDeviceObjects()` follows `group.groupId` up the chain, collecting `Attribute`, `Notification`, `Geofence`, `Command` objects at each level. There is no cycle detection.

## Line index

- 20 — `@StorageName("tc_groups")`
- 21 — `class Group extends GroupedModel`
- 23-31 — name
