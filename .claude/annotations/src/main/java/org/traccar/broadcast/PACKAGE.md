# `broadcast/` — Multi-node event fan-out bus

Provides a publish-subscribe bus for propagating device/position/event/command/cache-invalidation signals across multiple Traccar nodes running in a cluster.

## File index

| File | One-liner |
|---|---|
| `BroadcastService.java` | Interface extending `BroadcastInterface` + `LifecycleObject`; adds `singleInstance()` and `registerListener()` |
| `BroadcastInterface.java` | Listener interface with default-no-op methods: `updateDevice`, `updatePosition`, `updateEvent`, `updateCommand`, `invalidateObject`, `invalidatePermission` |
| `BroadcastMessage.java` | JSON-serializable envelope for wire messages (type + payload) |
| `NullBroadcastService.java` | Single-node stub; `singleInstance()=true`, all updates forwarded to local listeners only |
| `MulticastBroadcastService.java` | UDP multicast implementation; sends `BroadcastMessage` as JSON datagram to a multicast group |
| `RedisBroadcastService.java` | Redis pub/sub implementation using Lettuce; publishes messages to a channel, subscribes for incoming |
| `BaseBroadcastService.java` | Base class with listener registry and local-dispatch logic |

## Dependency graph

```
BroadcastService (interface)
 └── BaseBroadcastService (abstract, manages listeners)
      ├── NullBroadcastService   (single-node, no network)
      ├── MulticastBroadcastService  (UDP, Keys.BROADCAST_ADDRESS + PORT)
      └── RedisBroadcastService  (Lettuce, Keys.BROADCAST_REDIS_URL)

Listeners registered via registerListener():
  - CommandsManager (updateCommand)
  - CacheManager (invalidateObject, invalidatePermission)
  - ConnectionManager (updateDevice, updatePosition, updateEvent)
```

## Configuration

In `traccar.xml`:
```xml
<!-- Single node (default — NullBroadcastService) -->
<!-- no broadcast keys needed -->

<!-- Redis -->
<entry key="broadcast.type">redis</entry>
<entry key="broadcast.redis.url">redis://localhost:6379</entry>

<!-- UDP multicast -->
<entry key="broadcast.type">multicast</entry>
<entry key="broadcast.address">239.0.0.1</entry>
<entry key="broadcast.port">5175</entry>
```

Set `broadcast.secondary=true` on nodes that should not run scheduled tasks (prevents duplicate report generation / stat uploads).

## How the `local` flag works

Every `updateX(boolean local, ...)` method has a `local` parameter:
- `local=true` → originated on this node; the implementation publishes to the bus AND dispatches to local listeners.
- `local=false` → arrived from another node; dispatches to local listeners only (no re-publish, avoids loops).

## Key flow: cross-node command delivery

1. Node A receives `sendCommand(command)` API call.
2. Device session not found on Node A → command queued to DB.
3. `CommandsManager.sendCommand` calls `broadcastService.updateCommand(true, deviceId)`.
4. `RedisBroadcastService` publishes JSON message to Redis channel.
5. Node B (which has the device's TCP connection) receives message.
6. Node B's `CommandsManager.updateCommand(false, deviceId)` fires.
7. Node B dequeues commands from DB and sends to live device session.

## How to add a new broadcast transport

1. Extend `BaseBroadcastService`.
2. Override `start()`/`stop()` to set up the connection.
3. Implement `send(BroadcastMessage)` to serialize + publish.
4. On incoming message: deserialize and call `super.updateX(false, ...)` to dispatch locally.
5. Add a `case "mytype":` in `MainModule` broadcast binding.
