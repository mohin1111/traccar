# DriverEventHandler.java

**Role:** Fires `Event.TYPE_DRIVER_CHANGED` when the `KEY_DRIVER_UNIQUE_ID` changes between the current and last position (RFID/iButton driver identification change).
**Fits in:** Extends `BaseEventHandler`. Runs in the event-detection phase.
**Read next:** [[BaseEventHandler.java.md]], [[handler/DriverHandler.java.md]] (sets KEY_DRIVER_UNIQUE_ID in the pipeline)

## Public API

### `onPosition(Position, Callback)` (lines 35-53)
- Guards: only fires if position `isLatest` (not a historical replay).
- Reads `KEY_DRIVER_UNIQUE_ID` from current and last positions.
- If changed (including null → non-null): emits `TYPE_DRIVER_CHANGED` event.

## Gotchas / non-obvious

- **`isLatest` guard** — driver-changed events should not fire for historical position imports. Only real-time positions trigger this.
- **`KEY_DRIVER_UNIQUE_ID` is set by `DriverHandler`** in the pipeline — that handler can fall back to the linked-driver if the device does not report RFID. So this event fires on both hardware RFID changes and admin reassignments.

## Line index

- 36 — `isLatest` guard
- 39 — read current `KEY_DRIVER_UNIQUE_ID`
- 41-44 — read last driver ID
- 45-50 — emit if changed
