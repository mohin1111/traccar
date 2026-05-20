# EventResource.java

**Role:** Minimal REST resource at `/api/events/{id}` for reading a single event. Extends BaseResource directly. Permission checked via the event's deviceId.
**Fits in:** Entity: `Event`. Read-only (no CRUD hierarchy).
**Read next:** [[BaseResource.java]], `Event.java` model, `NotificationManager.java` (event creation)

## Public API

- `GET /api/events/{id}` — loads event by id; checks `Device` permission on `event.deviceId`; 404 if not found.

## Line index

- 40 — `@Path("events")` + extends `BaseResource`
- 42-49 — `get(id)` body
