# GroupResource.java

**Role:** REST resource at `/api/groups` for device group CRUD. Minimal — all logic in SimpleObjectResource. Sorted by name.
**Fits in:** Entity: `Group` (tc_groups). Extends SimpleObjectResource (no device/group cross-filter — groups are user-owned).
**Read next:** [[SimpleObjectResource.java]], `Group.java` model

## Public API

Full CRUD + user-filtered list. Constructor: `super(Group.class, "name", List.of("name"))`.

## Gotchas / non-obvious

Group hierarchy cycle check is in BaseObjectResource.update (line 106-108) — prevents a group from being its own parent.
