# BasePositionHandler.java

**Role:** Abstract base class and contract for every handler in the position-processing chain. Defines the `Callback` interface and wraps `onPosition` in exception-safe `handlePosition`.
**Fits in:** All per-position pipeline handlers (`FilterHandler`, `DatabaseHandler`, etc.) extend this. `ProcessingHandler` (root package) calls `handlePosition` on each in sequence.
**Read next:** [[ProcessingHandler.java]] (orchestrates the chain), [[FilterHandler.java.md]], [[DatabaseHandler.java.md]]

## Public API

### Interface `Callback` (line 26-28)
- `void processed(boolean filtered)` — must be called exactly once per position. Pass `true` to drop the position (filtered); `false` to continue the chain.

### Abstract method `onPosition` (line 30)
- Subclasses implement this. Must call `callback.processed(...)` in all code paths, including async callbacks.

### `handlePosition(Position, Callback)` (lines 32-39)
- Safe wrapper: catches `RuntimeException`, logs a warning, and calls `callback.processed(false)` to keep the chain flowing despite handler bugs.

## Key flows

1. `ProcessingHandler` calls `handlePosition` on the first handler.
2. Each handler calls `onPosition`, performs its work (sync or async), then calls `callback.processed(filtered)`.
3. `ProcessingHandler` wires callbacks so a `false` value advances to the next handler; `true` means the position is dropped (no DB write, no events).

## Gotchas / non-obvious

- **Async handlers must still call `callback.processed`** — it is not optional. A handler that never calls it deadlocks the chain for that device.
- **`RuntimeException` handling in `handlePosition`** — only `RuntimeException` is caught, not `Error`. Any unchecked exception in an async callback escape `handlePosition` and can propagate up.
- `LOGGER` is the only state here — the class is intentionally stateless so subclasses can be singletons.

## Line index

- 22 — class declaration
- 26-28 — `Callback` interface
- 30 — abstract `onPosition`
- 32-39 — `handlePosition` with exception guard
