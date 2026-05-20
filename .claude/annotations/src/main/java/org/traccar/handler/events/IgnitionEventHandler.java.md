# IgnitionEventHandler.java

**Role:** Detects ignition state transitions by comparing `KEY_IGNITION` between current and last position. Emits `TYPE_IGNITION_ON` or `TYPE_IGNITION_OFF`.
**Fits in:** Extends `BaseEventHandler`. Runs in the event-detection phase.
**Read next:** [[BaseEventHandler.java.md]]

## Public API

### `onPosition(Position, Callback)` (lines 36-57)
- Guards: device must exist, position must be latest.
- Only acts if both current and last positions have `KEY_IGNITION`.
- `false → true` transition: fire `TYPE_IGNITION_ON`.
- `true → false` transition: fire `TYPE_IGNITION_OFF`.

## Gotchas / non-obvious

- **Both positions must have `KEY_IGNITION`** — if the protocol starts reporting ignition mid-stream, the first position with ignition does not trigger an event (no prior state). Only the second position where the state has changed fires.
- **`isLatest` guard** — ignition events should not fire for replayed historical data.
- Ignition state is also used by `EngineHoursHandler` in the main pipeline; they share `KEY_IGNITION` but are independent.

## Line index

- 37-38 — device + isLatest guard
- 42 — read current ignition
- 45-48 — read last ignition
- 49-53 — transition detection + event emission
