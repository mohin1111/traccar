# MaintenanceEventHandler.java

**Role:** Checks configured maintenance schedules against current position values (odometer, hours, or time-based) and fires `TYPE_MAINTENANCE` events when a service interval threshold is crossed.
**Fits in:** Extends `BaseEventHandler`. Runs in the event-detection phase.
**Read next:** [[BaseEventHandler.java.md]]

## Public API

### `onPosition(Position, Callback)` (lines 35-57)
- Guards: last position must exist and not be newer than current.
- Iterates all `Maintenance` objects linked to the device via `CacheManager`.
- For each maintenance: reads `type` (the metric key), `start` (initial threshold), `period` (recurring interval).
- Computes `oldValue` and `newValue` from the relevant metric.
- Fires event if the value crosses a maintenance boundary: either first crossing of `start`, or crossing of `start + N*period`.

### `getValue(Position, String type)` (lines 59-65)
- `"serverTime"`, `"deviceTime"`, `"fixTime"` → timestamp in ms.
- Anything else → `position.getDouble(type)` (e.g., `"totalDistance"`, `"hours"`).

## Key flows

### Boundary crossing logic (lines 45-49)
```
fires if: newValue >= start AND
          (oldValue < start  → first crossing
           OR floor((oldValue-start)/period) < floor((newValue-start)/period) → recurring)
```

## Gotchas / non-obvious

- **`period = 0` skips** (line 42 guard) — an unconfigured maintenance entry is ignored.
- **`oldValue == 0` skips** — prevents false positives on the very first position.
- **Calendar-independent** — maintenance events are not gated by any calendar.
- **Multiple maintenance schedules** per device are all evaluated per position.

## Line index

- 37 — time-ordering guard
- 41-55 — maintenance loop
- 43-48 — values and boundary condition
- 49-52 — event emission
- 59-65 — `getValue` metric resolver
