# BaseEventHandler.java

**Role:** Abstract base class for all event detectors. Defines the `Callback` interface and wraps `onPosition` in exception-safe `analyzePosition`. Does not extend `BasePositionHandler` — event handlers run after the position pipeline, not as pipeline steps.
**Fits in:** All event handlers in `handler/events/` extend this. `ProcessingHandler` (root package) iterates the registered event handlers after the main chain completes, calling `analyzePosition` on each.
**Read next:** [[AlarmEventHandler.java.md]], [[GeofenceEventHandler.java.md]], [[OverspeedEventHandler.java.md]]

## Public API

### Interface `Callback` (lines 27-29)
- `void eventDetected(Event event)` — called once per detected event. A single position may trigger multiple events (e.g., multiple geofences).

### `analyzePosition(Position, Callback)` (lines 31-37)
- Exception-safe wrapper: catches `RuntimeException`, logs a warning. If an event handler throws, detection is skipped but the pipeline is unaffected.

### Abstract `onPosition(Position, Callback)` (line 42)
- Subclasses implement detection logic. Must call `callback.eventDetected` for each event found (zero or more times).

## Gotchas / non-obvious

- **Comment at line 41**: "Event handlers should be processed synchronously." Unlike `BasePositionHandler` which supports async (callback-based), event handlers are expected to be synchronous. Async event handlers are not currently supported by the framework.
- Event handlers **do not** call `processed(filtered)` — they emit events via `eventDetected`. Different contract from position handlers.

## Line index

- 23 — class declaration (abstract)
- 27-29 — `Callback` interface
- 31-37 — `analyzePosition` wrapper
- 42 — abstract `onPosition`
