# LifecycleObject.java

**Role:** Minimal marker interface defining `start()` / `stop()` with checked exceptions. Implemented by `ServerManager`, `WebServer`, `ScheduleManager`, and `BroadcastService`.
**Fits in:** `Main.run()` starts and stops a `List<LifecycleObject>` in order.
**Read next:** [[Main.java]] (orchestrator), [[ServerManager.java]] (primary implementor)

## Public API

```java
void start() throws Exception;
void stop() throws Exception;
```

Both declare `throws Exception` — callers in `Main` must either catch or rethrow. The shutdown hook wraps `stop()` in a `RuntimeException` rethrow.

## Gotchas / non-obvious

- Intentionally minimal — no `isRunning()`, no health check, no restart support. Transport OS wrapper should add these if needed.
