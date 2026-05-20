# MaintenanceResource.java

**Role:** REST resource at `/api/maintenance` for maintenance schedule CRUD. Minimal — all logic in ExtendedObjectResource. Sorted by name.
**Fits in:** Entity: `Maintenance` (tc_maintenances). Extends ExtendedObjectResource.
**Read next:** [[ExtendedObjectResource.java]], `Maintenance.java` model, `MaintenanceHandler.java` (fires maintenance events)

## Public API

Full CRUD + filtered list. Constructor: `super(Maintenance.class, "name", List.of("name"))`.
