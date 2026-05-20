# Schedulable.java

**Role:** Marker interface for entities that can be time-restricted via a linked `Calendar`. Implemented by `Device`, `Geofence`, `Notification`, and `Report`.
**Fits in:** `ScheduleManager` uses `Schedulable.getCalendarId()` → loads `Calendar` → calls `checkMoment(Date)` to decide whether to activate the entity at the current time.
**Read next:** [[Calendar.java]], [[Geofence.java]], [[Notification.java]], [[Report.java]], [[Device.java]]

## Public API

- `getCalendarId() / setCalendarId(long calendarId)` — FK to `tc_calendars.id`; `0` = no schedule restriction (always active).

## Line index

- 19 — `public interface Schedulable`
- 20 — `long getCalendarId()`
- 21 — `void setCalendarId(long calendarId)`
