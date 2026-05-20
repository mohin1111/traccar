# Report.java

**Role:** Saved scheduled report configuration. `@StorageName("tc_reports")` → `tc_reports`. Implements `Schedulable` (linked to a `Calendar`). `ScheduleManager` runs saved reports on their calendar schedule and emails results.
**Fits in:** Extends `ExtendedModel`. Linked to devices/groups/users via link tables. `ScheduleManager.handleReport()` evaluates the calendar, runs the appropriate report type, and delivers it.
**Read next:** [[Schedulable.java]], [[Calendar.java]], [[ExtendedModel.java]] (parent)

## Public API

### DB-mapped fields
- `calendarId long` (line 23) — FK to `tc_calendars`; schedule trigger.
- `type String` (line 33) — type key matching report providers: `"route"`, `"events"`, `"summary"`, `"trips"`, `"stops"`, `"combined"`, `"chart"`, `"geofences"`.
- `description String` (line 43) — display name.
- Inherited `attributes AttributeMap` — parameters (date range, device IDs, etc.).

## Line index

- 20 — `@StorageName("tc_reports")`
- 21 — `class Report extends ExtendedModel implements Schedulable`
- 23-31 — calendarId
- 33-41 — type
- 43-51 — description
