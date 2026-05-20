# CommandResultEventHandler.java

**Role:** Converts a command response in `KEY_RESULT` attribute (sent back by the device after an SMS/TCP command) into an `Event.TYPE_COMMAND_RESULT` event.
**Fits in:** Extends `BaseEventHandler`. Runs in the event-detection phase.
**Read next:** [[BaseEventHandler.java.md]]

## Public API

### `onPosition(Position, Callback)` (lines 28-35)
- Reads `position.getAttributes().get(KEY_RESULT)`.
- If non-null: emits `TYPE_COMMAND_RESULT` event with `KEY_RESULT` set.

## Gotchas / non-obvious

- **Cast to `String`** at line 31 — if a protocol decoder stores a non-String result, this throws `ClassCastException` (caught by `analyzePosition` wrapper).
- Simple stateless handler — no cache reads, no config, just attribute → event mapping.

## Line index

- 28 — `onPosition`
- 29 — read `KEY_RESULT`
- 30-34 — conditional event emission
