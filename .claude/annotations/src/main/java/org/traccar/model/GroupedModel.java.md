# GroupedModel.java

**Role:** Third tier of the model hierarchy. Adds `groupId` — a FK to `tc_groups` — to any entity that can belong to a device group. Enables attribute/command inheritance from group to device.
**Fits in:** `Device` and `Group` extend this. `CacheManager` walks the group chain to apply inherited settings.
**Read next:** [[ExtendedModel.java]] (parent), [[Device.java]] (primary user), [[Group.java]] (is itself a GroupedModel, enabling nested groups)

## Public API

- `long groupId` (line 21) — FK to `tc_groups.id`; 0 means "no group".
- `getGroupId()` / `setGroupId(long)` (lines 23-28)

## Key flows

### Group inheritance
`CacheManager.getDeviceObjects()` walks from `device.groupId` → `group.groupId` up the chain, collecting inherited `Attribute`s, `Notification`s, `Geofence`s, `Command`s. Depth is unbounded in code; cycles would loop forever (not guarded).

## Gotchas / non-obvious

- `Group` itself extends `GroupedModel`, making groups nestable: a group can have a parent group.
- `groupId = 0` means "root" (no parent); do not treat it as a valid FK.

## Line index

- 19 — `class GroupedModel extends ExtendedModel`
- 21 — `private long groupId`
- 23-25 — `getGroupId()`
- 27-29 — `setGroupId()`
