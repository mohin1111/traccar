# DatabaseHandler.java

**Role:** Persists a position to `tc_positions` via `PositionBatchWriter` (async batching). Step 18 — the terminal write step of the position pipeline. Also registers the message with `StatisticsManager` for daily counters.
**Fits in:** Extends `BasePositionHandler`. Final step before `PostProcessHandler`. Delegates actual SQL to `PositionBatchWriter` which batches writes.
**Read next:** [[PostProcessHandler.java.md]] (runs after, updates device's lastPositionId), [[BasePositionHandler.java.md]]

## Public API

### `onPosition(Position, Callback)` (lines 39-48)
- Calls `batchWriter.submit(position)` — returns a `CompletableFuture<Long>`.
- On success: sets `position.setId(id)` (the newly assigned DB row ID) and calls `statisticsManager.registerMessageStored`.
- On error: logs a warning.
- In both cases: `callback.processed(false)` — `DatabaseHandler` never drops positions at this stage (they already passed `FilterHandler`).

## Key flows

1. `FilterHandler` has passed the position (`filtered=false`).
2. `DatabaseHandler.onPosition` submits to `PositionBatchWriter`.
3. `PositionBatchWriter` accumulates positions and flushes via JDBC `INSERT`.
4. The `CompletableFuture` resolves with the generated position ID.
5. `position.setId(id)` is set — subsequent handlers (events, forwarding, post-process) can reference the DB ID.

## Gotchas / non-obvious

- **Async** — the callback fires on the `PositionBatchWriter`'s thread, not the Netty I/O thread. This is safe because the callback chain is thread-safe by design.
- **Position ID is set here** — event handlers that run after (in `ProcessingHandler`) can rely on `position.getId()` being non-zero.
- **Write errors are not fatal** — the pipeline continues with `callback.processed(false)` even on DB failure. The position is lost silently except for the WARN log.

## Line index

- 29 — `PositionBatchWriter` injection
- 39 — `onPosition`
- 40 — async `batchWriter.submit(position)`
- 42 — success branch: set ID, register stats
- 45 — error branch: warn log
- 47 — `callback.processed(false)` in both paths
