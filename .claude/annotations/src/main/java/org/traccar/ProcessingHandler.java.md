# ProcessingHandler.java

**Role:** THE critical path. Receives decoded `Position` objects from any protocol decoder and runs them through an ordered chain of enrichment handlers (geocoding, filtering, geofencing, etc.), then event handlers (overspeed, ignition, alarms, etc.), then persists to DB and notifies clients.
**Fits in:** `@Singleton @Sharable` — one instance shared across all Netty channels. Added last (before `MainEventHandler`) in `BasePipelineFactory.initChannel`. All per-device ordering is managed internally via a per-device queue.
**Read next:** [[BasePipelineFactory.java]] (where this is added), [[MainModule.java]] (where all handler singletons are conditionally provided), [[BaseProtocolDecoder.java]] (upstream producer)

## Public API

- `ProcessingHandler(Injector injector, Config config, CacheManager cacheManager, NotificationManager notificationManager, PositionLogger positionLogger)` (line 93) — builds the ordered `positionHandlers` and `eventHandlers` lists via Guice `getInstance`; null handlers (disabled features) are filtered out.
- `channelRead(ChannelHandlerContext ctx, Object msg)` (line 145) — entry point from Netty; accepts `Position` objects only; passes others to super.
- `onReleased(ChannelHandlerContext context, Position position)` (line 155) — `BufferingManager.Callback`; called when a buffered/delayed position is ready for processing.

## Key flows

### Handler chain construction (lines 101-139)
Position handlers (ordered):
1. `ComputedAttributesHandler.Early` — JEXL expressions before filter
2. `OutdatedHandler` — drop positions older than device's last position
3. `TimeHandler` — fix/adjust device timestamps
4. `GeolocationHandler` — cell/WiFi → GPS coords (if no GPS fix)
5. `HemisphereHandler` — fix hemisphere sign from protocol quirks
6. `MapMatcherHandler` — snap coordinates to nearest road
7. `DistanceHandler` — compute odometer delta
8. `FilterHandler` — drop invalid/duplicate positions
9. `GeofenceHandler` — compute geofence membership
10. `GeocoderHandler` — reverse geocode lat/lon → address string
11. `SpeedLimitHandler` — attach posted speed limit from Overpass
12. `MotionHandler` — update motion state
13. `ComputedAttributesHandler.Late` — JEXL expressions after enrichment
14. `DriverHandler` — match RFID/iButton to driver record
15. `CopyAttributesHandler` — propagate defaults from group/device attributes
16. `EngineHoursHandler` — accumulate engine hours
17. `PositionForwardingHandler` — push to external forwarders (Kafka, MQTT, webhook...)
18. `DatabaseHandler` — persist `Position` to `tc_positions`

Event handlers (run after all position handlers, in parallel per-position):
`MediaEvent`, `CommandResult`, `Overspeed`, `Behavior`, `Fuel`, `Motion`, `Geofence`, `Proximity`, `Alarm`, `Ignition`, `Maintenance`, `Driver`

### Per-device ordering guarantee (lines 155-222)
Each device has a `Queue<QueuedPosition>`. When `onReleased` is called:
- If queue was empty → process immediately
- Otherwise → enqueue; previous position's `finishedProcessing()` dequeues and starts the next

This ensures positions for the same device are processed strictly in order, even though Netty channels are concurrent.

### Async handler support (lines 167-189)
`processPositionHandlers` uses a recursive callback (`BasePositionHandler.Callback.processed(filtered)`) that continues on the **Netty event loop executor** if called from off-thread (e.g., geocoder async HTTP response). This keeps ordering without blocking Netty threads.

### `finishedProcessing` (lines 198-210)
- Not filtered: calls `postProcessHandler` then `positionLogger`, then writes `AcknowledgementHandler.EventHandled` (triggers TCP ACK), then dequeues next position.
- Filtered: just sends `EventHandled` and dequeues (no persistence, no forwarding).
- Always: removes device from `cacheManager`.

## Gotchas / non-obvious

- **`@Singleton @Sharable`** — the handler is shared across all channels and threads. Never store per-connection state as instance fields. Per-device state is in `queues`.
- **Null handlers** — disabled features (filter off, no geocoder, etc.) are silently absent from the list. The `Stream.filter(Objects::nonNull)` at lines 121-122 handles this. The pipeline length varies by config.
- **`BufferingManager`** — sits between `channelRead` and `onReleased`. It may hold positions briefly (e.g., for ordering across protocol reconnects). Its callback is `this`.
- **Event handlers fire on every non-filtered position** — they don't stop the chain. Each event handler calls `notificationManager.updateEvents` independently.
- **`FilterHandler` drops a position** by calling `callback.processed(true)` — this short-circuits the remainder of the chain. No further enrichment, no DB write, but `EventHandled` still fires.

## Line index

- 74-86 — field declarations
- 84 — `QueuedPosition` record
- 86 — `queues` map (per-device FIFO)
- 93 — constructor
- 101-122 — `positionHandlers` list (18 handlers)
- 124-139 — `eventHandlers` list (12 handlers)
- 141 — `postProcessHandler`
- 145-152 — `channelRead` entry
- 155-165 — `onReleased` + queue logic
- 167-189 — `processPositionHandlers` (async callback chain)
- 192-196 — `processEventHandlers` (fire all 12 in parallel)
- 198-210 — `finishedProcessing` (ACK + dequeue)
- 212-222 — `processNextPosition`
