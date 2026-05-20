# Calendar.java

**Role:** An iCal calendar used to time-gate features (geofences, notifications, scheduled reports). `@StorageName("tc_calendars")` → `tc_calendars`. Stores raw iCal (`data` bytes) and provides `checkMoment(Date)` to test whether a given timestamp falls in an active period.
**Fits in:** Extends `ExtendedModel`. Referenced by `Geofence`, `Notification`, `Report`, `Device` via `calendarId`. `ScheduleManager` uses `checkMoment` to decide whether to fire scheduled reports.
**Read next:** [[Schedulable.java]] (interface), [[Geofence.java]], [[Notification.java]], [[Report.java]]

## Public API

### DB-mapped fields
- `name String` (line 47) — display name.
- `data byte[]` (line 56) — raw iCal bytes. `setData()` parses immediately via ical4j `CalendarBuilder`; throws `IOException`/`ParserException` on bad data.

### Not DB-mapped
- `getCalendar()` (line 70) — the parsed ical4j `Calendar` object; `@QueryIgnore @JsonIgnore`.
- `findPeriods(Date date)` (line 75) — returns all `VEVENT` periods active at the given instant; handles `LocalDate`, `LocalDateTime`, `ZonedDateTime`, `OffsetDateTime` temporal types.
- `checkMoment(Date date)` (line 91) — convenience wrapper; returns true if `findPeriods` is non-empty.

## Gotchas / non-obvious

- `setData(byte[])` does a **full parse** on every call (line 61-64) — do not call repeatedly in hot paths.
- Timezone handling in `convertToMatchingTemporal` (line 95-101) matches the VEVENT's own timezone, not a server-wide timezone.
- If `calendar == null` (e.g., `data` was never set), `findPeriods` returns `Set.of()` — `checkMoment` returns `false`, effectively disabling the schedule gate. Callers must guard.

## Line index

- 42 — `@StorageName("tc_calendars")`
- 47-54 — name
- 56-65 — data (raw iCal bytes; parsed on set)
- 68-73 — getCalendar (@QueryIgnore @JsonIgnore)
- 75-89 — findPeriods (VEVENT recurrence expansion)
- 91-93 — checkMoment
- 95-113 — private temporal conversion helpers
