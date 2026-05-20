# DriverResource.java

**Role:** REST resource at `/api/drivers` for driver CRUD. Minimal — all logic in ExtendedObjectResource. Sorted by name, searchable by name and uniqueId (RFID/iButton identifier).
**Fits in:** Entity: `Driver` (tc_drivers). Extends ExtendedObjectResource.
**Read next:** [[ExtendedObjectResource.java]], `Driver.java` model, `DriverHandler.java` (links drivers to positions)

## Public API

Full CRUD + filtered list. Constructor: `super(Driver.class, "name", List.of("name", "uniqueId"))`.
