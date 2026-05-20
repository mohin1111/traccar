# PositionForwardingHandler.java

**Role:** Forwards each position to external systems (HTTP, Kafka, MQTT, AMQP, Redis, Wialon) via a `PositionForwarder`. Implements exponential-backoff retry. Step 17 in the pipeline.
**Fits in:** Extends `BasePositionHandler`. Runs after `EngineHoursHandler` (step 16), before `DatabaseHandler` (step 18). The actual forwarding mechanism is injected as `PositionForwarder` (may be null if no forwarder is configured).
**Read next:** [[DatabaseHandler.java.md]] (step after), [[BasePositionHandler.java.md]]

## Public API

### `onPosition(Position, Callback)` (lines 127-135)
- If no `positionForwarder` configured (`null`): no-op, chain continues.
- Builds `PositionData` (position + device), creates `AsyncRequestAndCallback`, calls `send()`.
- `callback.processed(false)` is called immediately — forwarding is fire-and-forget.

## Key flows

### `AsyncRequestAndCallback` (lines 69-124)
- Wraps a single delivery attempt with retry logic.
- `deliveryPending` (AtomicInteger) tracks in-flight deliveries globally.
- On success: decrements `deliveryPending`.
- On failure: if `retryEnabled && pending <= retryLimit && retries < retryCount`: schedules a retry via `timer.newTimeout()` with exponential delay `retryDelay * 2^retries`.
- Implements `TimerTask.run` for the scheduled retry.

## Gotchas / non-obvious

- **Non-blocking** — `callback.processed(false)` fires before the forward completes. DB write happens in parallel with the forward attempt.
- **`@Nullable PositionForwarder`** — Guice injects `null` if no forwarder module is bound. Handler silently skips.
- **Retry limit `retryLimit`** is a global cap on concurrent pending deliveries, not per-position. Under sustained failure, new retries stop being scheduled once `deliveryPending > retryLimit`.
- **Exponential backoff**: delay = `retryDelay * 2^attempt`. With default `retryDelay=100ms`, attempt 0 = 100ms, attempt 1 = 200ms, etc.

## Line index

- 44-66 — constructor: config + deliveryPending init
- 69 — `AsyncRequestAndCallback` inner class
- 80-95 — `retry()` with exponential scheduling
- 102-108 — `onResult` callback (success/failure dispatch)
- 111-123 — `TimerTask.run` retry execution
- 127-135 — `onPosition`: build PositionData + fire-and-forget
