# GeofenceResource.java

**Role:** REST resource at `/api/geofences` for geofence CRUD. Minimal — all logic in ExtendedObjectResource. Sorted by name, searchable by name.
**Fits in:** Entity: `Geofence` (tc_geofences). Extends ExtendedObjectResource.
**Read next:** [[ExtendedObjectResource.java]], `Geofence.java` model, `MapGeofenceEdit.js` (web client)

## Public API

Full CRUD + filtered list via base classes. Constructor: `super(Geofence.class, "name", List.of("name"))`.
