# TrackerClient.java

**Role:** `TrackerConnector` for outbound TCP connections. Server initiates connection to a remote device/service, reconnects on close, and runs the same `BasePipelineFactory` pipeline as inbound servers.
**Fits in:** Used by protocols where Traccar polls the device (not device-initiated). `BaseProtocol.addClient()` registers these.
**Read next:** [[TrackerConnector.java]] (interface), [[BasePipelineFactory.java]] (pipeline), [[TrackerServer.java]] (inbound counterpart)

## Public API

- `TrackerClient(Config config, String protocol)` — reads `<protocol>.ssl`, `<protocol>.interval`, `<protocol>.address`, `<protocol>.port`, `<protocol>.devices`; creates a Netty `Bootstrap` with the pipeline factory.
- `addProtocolHandlers(PipelineBuilder pipeline, Config config)` — abstract; implement with frame decoder + `*ProtocolDecoder`.
- `getDevices()` — array of device unique IDs this client polls for (from `<protocol>.devices` config).
- `start()` — connects and registers a `GenericFutureListener` that reconnects after `interval` seconds on disconnect.
- `stop()` — closes the channel group.

## Key flows

### Reconnect loop (lines 106-118)
On channel close, if `interval > 0`, schedules a reconnect via `GlobalEventExecutor`. The `GenericFutureListener` self-references to form a perpetual reconnect loop.

## Gotchas / non-obvious

- `isDatagram()` is always false — `TrackerClient` is TCP-only.
- `getDevices()` is not used by the base class — it's surfaced for subclass protocol decoders that need to know which device IDs to request data for.
- The Netty `Bootstrap` uses the worker group only (no boss group for outbound).

## Line index

- 56-92 — constructor (config read + Bootstrap init)
- 94 — abstract `addProtocolHandlers`
- 96-98 — `getDevices()`
- 106-119 — `start()` with reconnect listener
- 121-123 — `stop()`
