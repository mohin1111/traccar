# StandardLoggingHandler.java

**Role:** Logs raw inbound/outbound bytes for each protocol connection. Also notifies `ConnectionManager` of log records (used by the real-time log WebSocket stream). One instance per protocol channel.
**Fits in:** Netty pipeline, bidirectional (step 6). `ChannelDuplexHandler` — sees both inbound data and outbound writes.
**Read next:** [[AcknowledgementHandler.java.md]] (next in pipeline), [[NetworkMessageHandler.java.md]] (previous)

## Public API

### Constructor (lines 49-55)
Takes `protocol` name. Sets a `logLimit` of 50 for `jt1078` (video streaming protocol that floods logs) and 0 (unlimited) for all others.

### `channelRead` (lines 67-73)
- Creates a `LogRecord` from the inbound `NetworkMessage`.
- Logs via SLF4J.
- Fires message downstream.
- Sends `LogRecord` to `ConnectionManager.updateLog` (for live log streaming to WebSocket clients).

### `write` (lines 77-80)
Same but for outbound: logs the write, then calls `super.write`.

### `createLogRecord` (lines 82-97)
- Only processes `NetworkMessage` with `ByteBuf` content.
- If `decodeTextData=true` and buffer is printable ASCII: logs as text (with `\r`/`\n` escapes).
- Otherwise: hex dump via `ByteBufUtil.hexDump`.

## Gotchas / non-obvious

- **`logLimit` for JT1078** — video streaming fills logs with binary frames. After 50 messages, logging is suppressed for that channel.
- **`decodeTextData`** (config `LOGGER_TEXT_PROTOCOL`) makes text-based protocols (NMEA, ITS, etc.) readable in logs. Disabled by default.
- **Not `@Sharable`** — maintains per-channel `logCount`. Each protocol channel gets its own instance.
- **`@Inject` methods** for `Config` and `ConnectionManager` — field injection after construction (Guice method injection). Handlers are constructed by protocol factories before Guice runs the injections.

## Line index

- 49-55 — constructor: protocol name + logLimit
- 57-58 — `@Inject setConfig` (decodeTextData)
- 60-62 — `@Inject setConnectionManager`
- 67-73 — `channelRead`
- 77-80 — `write`
- 82-97 — `createLogRecord` (hex or text)
- 100-114 — `log()` with throttle check
