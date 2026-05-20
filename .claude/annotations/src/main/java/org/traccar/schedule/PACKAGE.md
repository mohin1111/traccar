# `schedule/` — Scheduled background tasks

11 files: a `ScheduleManager` that owns a 4-thread pool and runs 8 periodic `ScheduleTask` implementations on startup.

## File index

| File | One-liner |
|---|---|
| `ScheduleManager.java` | Starts/stops the `ScheduledExecutorService`; instantiates and schedules all tasks |
| `ScheduleTask.java` | Abstract base: `schedule(ScheduledExecutorService)` + `multipleInstances()` |
| `SingleScheduleTask.java` | Variant for tasks that should run only on the primary node |
| `TaskHealthCheck.java` | Pings a configured URL (health check endpoint); useful for uptime monitors |
| `TaskClearStatus.java` | Clears `online/offline` device status for devices inactive beyond threshold |
| `TaskExpirations.java` | Disables users + devices past their expiration date |
| `TaskDeleteTemporary.java` | Deletes temporary shares (`tc_shares`) past their expiry |
| `TaskReports.java` | Generates and emails scheduled reports (daily) |
| `TaskDeviceInactivityCheck.java` | Fires `deviceInactive` events for devices not seen in > configured period |
| `TaskSessionTimeout.java` | Expires idle HTTP sessions from the Jetty JDBC session store |
| `TaskWebSocketKeepalive.java` | Sends WebSocket ping frames to keep connections alive through proxies |

## Dependency graph

```
ScheduleManager (LifecycleObject) → started by Main.java
 ├── TaskHealthCheck         (runs on all nodes, multipleInstances=true)
 ├── TaskClearStatus         (runs on primary only)
 ├── TaskExpirations         (primary only)
 ├── TaskDeleteTemporary     (primary only)
 ├── TaskReports             (primary only)
 ├── TaskDeviceInactivityCheck (primary only)
 ├── TaskSessionTimeout      (primary only)
 └── TaskWebSocketKeepalive  (all nodes)
```

## Primary vs. secondary nodes

`ScheduleManager` reads `Keys.BROADCAST_SECONDARY`. On secondary nodes (`broadcast.secondary=true`), only tasks with `multipleInstances()=true` are scheduled. This prevents duplicate scheduled report emails, duplicate expiration processing, etc. in a cluster.

## Adding a new scheduled task

1. Extend `ScheduleTask` (or `SingleScheduleTask` if primary-only).
2. Implement `schedule(executor)` — call `executor.scheduleWithFixedDelay(this::run, initial, period, unit)`.
3. Implement `run()` with the task logic.
4. Add to `ScheduleManager.start()` stream.

## Key task periods (approximate)

- `TaskHealthCheck` — every 60s
- `TaskClearStatus` — every 5 min
- `TaskExpirations` — every 6 hours
- `TaskWebSocketKeepalive` — every 55s (just under typical 60s proxy timeout)
- `TaskReports` — daily

## Transport OS relevance

`TaskDeviceInactivityCheck` is directly useful: fires `deviceInactive` events that can trigger alerts when a truck has not reported for > N minutes. Configurable per device via `deviceInactivityStart` + `deviceInactivityPeriod` attributes. Essential for AIS-140 monitoring SLA.

`TaskSessionTimeout` is important for security — ensures expired HTTP sessions are cleaned up. In a clustered deployment with Redis session store, ensure `TaskSessionTimeout` runs on only one node.
