# PositionBatchWriter.java

**Role:** Optional async batch inserter for `tc_positions`. When `DATABASE_POSITION_BATCH_INTERVAL` > 0, accumulates positions in a `ConcurrentLinkedQueue` and flushes them in bulk on a scheduled executor, reducing per-position DB round trips under high load.
**Fits in:** Injected into `DatabaseHandler` (in `handler/`) which calls `submit()` instead of inserting directly. Falls back to immediate single-insert when batching is disabled.
**Read next:** [[DatabaseHandler]] (caller), [[Storage.java]] (underlying DB layer)

## Public API

- `submit(Position position)` (line 65) — submits a position for insertion. Returns a `CompletableFuture<Long>` that resolves to the generated DB id. Caller must `join()` or chain on the future before using the id.

## Key flows

### Batching enabled (`interval > 0`, lines 53-62)
A single-thread `ScheduledExecutorService` named `"PositionBatchWriter"` runs `flush()` every `interval` ms.

### `flush()` (lines 79-97)
1. Drains up to `batchSize` entries from the queue.
2. Calls `storage.addObjects(positions, INSERT_REQUEST)` — batch insert returning list of generated ids.
3. Completes each future with its corresponding id.
4. On exception: all futures in the batch are `completeExceptionally`.

### Batching disabled (`interval == 0`, lines 67-71)
`queue` is null. `submit` calls `storage.addObject` synchronously and completes the future immediately.

## Gotchas / non-obvious

- **`queue = null` when disabled** — code checks `queue == null` to branch. Do not NPE-check `queue` unnecessarily; the null is intentional.
- **Daemon thread** — the executor thread is daemon (`thread.setDaemon(true)`) so it won't prevent JVM shutdown.
- **`addObjects` is a bulk insert** — `Storage` / `QueryBuilder` must support it. In `DatabaseStorage`, this is a single `executeBatch()` JDBC call.
- **No retry** — if the batch insert fails, all positions in that batch are lost (futures exceptionally completed). The caller (`DatabaseHandler`) logs the error.
- **`batchSize = 0`** (default from `Keys.DATABASE_POSITION_BATCH_SIZE`) means the flush loop drains the entire queue — use a positive value to cap memory use under burst traffic.

## Line index

- 39 — `INSERT_REQUEST` static constant
- 41 — `Entry` record (position + future pair)
- 47-62 — `@Inject` constructor: reads batch config, optionally starts scheduler
- 65-77 — `submit` (null-queue path = immediate; queue path = enqueue)
- 79-97 — `flush` (drain queue, batch insert, complete futures)
