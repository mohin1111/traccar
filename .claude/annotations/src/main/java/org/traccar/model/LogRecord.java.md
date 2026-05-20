# LogRecord.java

**Role:** Represents a protocol data log entry pushed to connected WebSocket clients when log streaming is enabled. Not a DB entity (no `@StorageName`, no `BaseModel` inheritance) — a transient value object.
**Fits in:** Created by `StandardLoggingHandler` (or protocol handlers) and dispatched via `ConnectionManager.updateLog()`. WebSocket clients receive `{logs: [...]}` messages containing these objects.
**Read next:** [[Position.java]] (what the log captures), [[Device.java]]

## Public API

- `localAddress InetSocketAddress` / `remoteAddress InetSocketAddress` — Netty channel endpoints; set in constructor.
- `getHost()` (line 44) — returns `remoteAddress.getHostString()` — the device's IP address.
- `getConnectionKey()` (line 39) — returns `ConnectionKey(localAddress, remoteAddress)` — identifies the Netty channel. `@JsonIgnore`.
- `getAddress()` (line 35) — returns `remoteAddress`; `@JsonIgnore`.
- `protocol String` (line 47) — protocol name.
- `uniqueId String` (line 57) — device identifier from the protocol frame.
- `deviceId long` (line 67) — resolved `tc_devices.id`.
- `data String` (line 77) — raw hex or decoded payload string.

## Gotchas / non-obvious

- `localAddress` and `remoteAddress` are cast from `SocketAddress` in constructor (line 30-31). Will throw `ClassCastException` for non-inet addresses (UDP datagrams with unusual address types).
- `getAddress()` and `getConnectionKey()` are `@JsonIgnore` — not sent to WebSocket clients.

## Line index

- 24 — `class LogRecord` (no inheritance, no @StorageName)
- 29-32 — constructor (cast SocketAddress to InetSocketAddress)
- 35-37 — getAddress (@JsonIgnore)
- 39-41 — getConnectionKey (@JsonIgnore)
- 44-46 — getHost (device IP)
- 47-55 — protocol
- 57-65 — uniqueId
- 67-75 — deviceId
- 77-84 — data
