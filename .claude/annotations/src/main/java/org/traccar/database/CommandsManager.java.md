# CommandsManager.java

**Role:** Orchestrates outbound device command dispatch — tries live TCP connection first, falls back to SMS, external senders (Firebase/FindHub), or queues commands to `tc_queued_commands` for deferred pickup.
**Fits in:** Called by `api/resource/CommandResource` (API-initiated commands). Also implements `BroadcastInterface` to receive cross-node `updateCommand` events when a queued command should be flushed.
**Read next:** [[CommandSenderManager.java]], [[BroadcastService.java]], [[NotificationManager.java]]

## Public API

- `sendCommand(Command command)` (line 78) — main entry. Returns `QueuedCommand` if queued, `null` if sent immediately. Throws on failure.
- `readQueuedCommands(long deviceId)` (lines 121-144) — dequeues all queued commands for device, fires `TYPE_QUEUED_COMMAND_SENT` events, returns them as `Command` objects.
- `readQueuedCommands(long deviceId, int count)` — same, capped at `count`.
- `updateCommand(boolean local, long deviceId)` (lines 147-157) — `BroadcastInterface` callback; on remote signal, flushes the queue to any active session.
- `updateNotificationToken(long deviceId, String token)` (lines 159-174) — stores Firebase/push token in device attributes and invalidates cache.

## Key flows

### `sendCommand` dispatch decision tree (lines 78-118)
```
if command.textChannel:
    → protocol.sendTextCommand(phone, command)   (SMS via protocol encoder)
    → OR smsManager.sendMessage(phone, data)     (custom SMS via HTTP/SNS)
else:
    sender = commandSenderManager.getSender(device)
    if sender != null:
        → sender.sendCommand(device, command)    (Firebase / FindHub / Traccar push)
    else:
        session = connectionManager.getDeviceSession(deviceId)
        if session != null && session.supportsLiveCommands():
            → session.sendCommand(command)       (live TCP write)
        else if !command.noQueue:
            → storage.addObject(queuedCommand)   (persist to tc_queued_commands)
            → broadcastService.updateCommand(true, deviceId)  (notify other nodes)
        else:
            → throw RuntimeException("Failed to send command")
```

### Queue pickup (`readQueuedCommands`)
Called by `BaseProtocolDecoder.getDeviceSession` when a device reconnects — each decoder that supports queued commands calls this. Fires `TYPE_QUEUED_COMMAND_SENT` events for each dequeued command.

## Gotchas / non-obvious

- **`smsManager` is `@Nullable`** — SMS channel throws if not configured.
- **Live commands require `supportsLiveCommands()`** — only `DeviceSession` with an active writable channel returns true.
- **`noQueue` flag** (`Command.KEY_NO_QUEUE`) forces immediate failure instead of queuing — useful for time-sensitive commands that shouldn't be sent stale.
- **`updateNotificationToken` does cache invalidation** via `broadcastService` — all nodes get the updated token atomically.

## Line index

- 50-76 — `@Inject` constructor + `broadcastService.registerListener(this)`
- 78-119 — `sendCommand` dispatch tree
- 121-145 — `readQueuedCommands` (dequeue + fire events)
- 147-157 — `updateCommand` BroadcastInterface callback
- 159-174 — `updateNotificationToken` + cache invalidation
