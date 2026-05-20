# EngineHoursHandler.java

**Role:** Accumulates engine-hours (run time in milliseconds) by tracking ignition-on durations between positions. Only acts when the device does not report `KEY_HOURS` natively.
**Fits in:** Extends `BasePositionHandler`. Step 16 in the pipeline (after `CopyAttributesHandler`).
**Read next:** [[CopyAttributesHandler.java.md]], [[BasePositionHandler.java.md]]

## Public API

### `onPosition(Position, Callback)` (lines 33-47)
- Skips if position already has `KEY_HOURS` (device-side counter takes priority).
- Reads `hours` from the last position.
- If both `last` and current position have `KEY_IGNITION = true`, adds the time delta (`deviceTime` difference in ms) to accumulated hours.
- Sets `KEY_HOURS` only if non-zero (avoids polluting positions before the first ignition event).

## Gotchas / non-obvious

- **Double-ignition guard** — both `last.KEY_IGNITION` AND `position.KEY_IGNITION` must be true. If ignition transitioned on this position, no hours are added until the next position. This avoids counting the start-up delta.
- **Device time, not fix time** — uses `getDeviceTime()` for the delta. Device time reflects the actual hardware clock; fix time may lag for buffered positions.
- **Device-reported hours always win** — guard at line 34 (`!position.hasAttribute(KEY_HOURS)`).

## Line index

- 34 — guard: skip if device-reported hours present
- 36-37 — fetch last position and accumulated hours
- 38 — double-ignition check
- 39 — add time delta in ms
- 41-43 — set hours if non-zero
