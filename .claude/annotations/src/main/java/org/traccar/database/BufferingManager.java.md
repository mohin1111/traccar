# BufferingManager.java

**Role:** Re-orders out-of-sequence positions from a single device. Holds positions in a `TreeSet<Holder>` sorted by fix/device/server time and releases them after a configurable threshold delay via Netty's `HashedWheelTimer`.
**Fits in:** Instantiated inside `ProcessingHandler` when `Keys.SERVER_BUFFERING_THRESHOLD > 0`. Wraps the normal position processing callback so positions arrive in order.
**Read next:** `handler/ProcessingHandler.java` (caller)

## Public API

- `BufferingManager(Config config, Callback callback)` — constructor. Reads `SERVER_BUFFERING_THRESHOLD`. If threshold is 0, `accept()` calls through immediately.
- `accept(ChannelHandlerContext context, Position position)` — queues position if threshold > 0, otherwise fires callback immediately.
- `Callback` interface: `onReleased(ChannelHandlerContext context, Position position)` — called when a position is released from the buffer.

## Key flows

### Buffering logic `accept()` (lines 101-117)
1. Inserts position into per-device `TreeSet<Holder>` (sorted by fix time, then device time, then server time).
2. Schedules a timeout for the new holder.
3. Any existing holders AFTER this new holder in the sorted set have their timeouts **reset** — they must wait at least `threshold` ms from the most recent insertion that precedes them in time.

Effect: if positions arrive out of order, earlier positions re-sort ahead and their timeouts are pushed out until all earlier positions have been flushed, ensuring ordered delivery.

### Timeout callback (lines 89-97)
On expiry: removes holder from set, re-executes `callback.onReleased` on the device's Netty executor thread.

## Gotchas / non-obvious

- **`threshold = 0` disables buffering** — `accept` calls `callback.onReleased` synchronously. No timer overhead.
- **Per-device isolation** — each device has its own `TreeSet`. Ordering is only within a device.
- **`HashedWheelTimer` is shared** across all devices — one timer for all timeout tasks.
- **Comparison uses server time as tiebreaker** (line 72) — guarantees total order even when fix/device times are equal or null.
- **Possible memory growth** if devices send many positions quickly with a large threshold. No size cap.

## Line index

- 38-40 — `Callback` interface
- 42-74 — `Holder` inner class (context + position + timeout, `compareTo` by time)
- 76-85 — fields: timer, callback, threshold, buffer map
- 82-85 — constructor
- 87-98 — `scheduleTimeout` helper
- 101-117 — `accept` (buffer or pass through)
