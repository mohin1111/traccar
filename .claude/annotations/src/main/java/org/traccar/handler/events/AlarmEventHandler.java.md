# AlarmEventHandler.java

**Role:** Converts `KEY_ALARM` attribute values (comma-separated alarm codes from the device) into `Event.TYPE_ALARM` events. Optionally suppresses duplicate alarms that were present in the last position.
**Fits in:** Extends `BaseEventHandler`. Runs in the event-detection phase after the main pipeline.
**Read next:** [[BaseEventHandler.java.md]]

## Public API

### `onPosition(Position, Callback)` (lines 41-62)
- Reads `position.getString(KEY_ALARM)` — may be a comma-separated list (e.g., `"sos,powerCut"`).
- Splits into a `Set<String>` of alarm codes.
- If `ignoreDuplicates=true`: removes alarm codes already present in the last position's `KEY_ALARM`. Prevents repeated events for a sustained alarm.
- Emits one `Event.TYPE_ALARM` per alarm code, with the alarm code stored as `KEY_ALARM` in the event.

## Gotchas / non-obvious

- **Multiple alarms per position** — a single position can carry multiple comma-separated alarm codes, each generating its own event.
- **`ignoreDuplicates`** requires comparing with the last cached position. If the cache is empty (first position), no dedup occurs.
- **Config key `EVENT_IGNORE_DUPLICATE_ALERTS`** is global, not per-device.

## Line index

- 37 — `ignoreDuplicates` flag from config
- 41 — `onPosition`
- 43-44 — split alarm string into set
- 45-53 — duplicate suppression
- 54-59 — emit one event per alarm code
