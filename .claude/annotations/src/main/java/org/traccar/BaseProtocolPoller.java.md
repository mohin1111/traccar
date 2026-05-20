# BaseProtocolPoller.java

**Role:** `ChannelDuplexHandler` that fires periodic outbound requests on a fixed schedule. Used by protocols that require the server to poll the device (rather than device-initiated push).
**Fits in:** Added to the pipeline in `addProtocolHandlers` for polling-style protocols. Typically combined with a `TrackerClient` (outbound connector).
**Read next:** [[TrackerClient.java]] (outbound TCP connector), [[BasePipelineFactory.java]] (pipeline context)

## Public API

- `BaseProtocolPoller(long interval)` — interval in seconds. 0 = disabled.
- `sendRequest(Channel channel, SocketAddress remoteAddress)` — abstract; implement to send the polling message.

## Key flows

- `channelActive`: schedules `sendRequest` at `0, interval, TimeUnit.SECONDS` on the channel executor.
- `channelInactive`: cancels the scheduled task.

## Gotchas / non-obvious

- Runs on the Netty event loop — `sendRequest` must not block. Use `channel.writeAndFlush(new NetworkMessage(...))`.
- Interval is in seconds (not milliseconds). Pass `config.getLong(Keys.PROTOCOL_INTERVAL.withPrefix(protocol))`.

## Line index

- 31-32 — constructor
- 35 — abstract `sendRequest`
- 38-43 — `channelActive` (schedule)
- 46-53 — `channelInactive` (cancel)
