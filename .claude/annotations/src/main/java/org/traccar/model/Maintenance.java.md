# Maintenance.java

**Role:** Defines a maintenance schedule rule. `@StorageName("tc_maintenances")` → `tc_maintenances`. A `Maintenance` triggers a `TYPE_MAINTENANCE` event when a device's `type` accumulator (odometer, hours) crosses a threshold (`start`) and then repeats every `period` units.
**Fits in:** Extends `ExtendedModel`. Linked to devices/groups/users. `MaintenanceEventHandler` in the pipeline compares the position's accumulator value against each linked `Maintenance` rule.
**Read next:** [[Event.java]] (TYPE_MAINTENANCE), [[ExtendedModel.java]] (parent)

## Public API

### DB-mapped fields
- `name String` (line 24) — display label.
- `type String` (line 34) — which accumulator to watch: matches a `Position.KEY_*` string, e.g., `"totalDistance"` (meters), `"hours"` (milliseconds).
- `start double` (line 44) — initial threshold (absolute value in the accumulator's units).
- `period double` (line 54) — repeat interval after `start` is crossed. E.g., `start=5000000` (5000 km), `period=5000000` → alerts every 5000 km.

## Gotchas / non-obvious

- `type = "hours"` units are **milliseconds** (matching `Position.KEY_HOURS`). UI converts to hours for display.
- `type = "totalDistance"` units are **meters**. UI converts to km/miles.

## Line index

- 21 — `@StorageName("tc_maintenances")`
- 22 — `class Maintenance extends ExtendedModel`
- 24-32 — name
- 34-42 — type (accumulator key)
- 44-52 — start (initial threshold)
- 54-62 — period (repeat interval)
