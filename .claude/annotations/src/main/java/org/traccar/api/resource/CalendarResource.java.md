# CalendarResource.java

**Role:** REST resource at `/api/calendars` for iCalendar-based schedule objects. Minimal — all logic in SimpleObjectResource. Sorted by name.
**Fits in:** Entity: `Calendar` (tc_calendars). Extends SimpleObjectResource (no device/group cross-filter — calendars referenced by notifications).
**Read next:** [[SimpleObjectResource.java]], `Calendar.java` model

## Public API

Full CRUD + user-filtered list. Constructor: `super(Calendar.class, "name", List.of("name"))`.
