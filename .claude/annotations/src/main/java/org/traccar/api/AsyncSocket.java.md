# AsyncSocket.java

**Role:** The server-side WebSocket endpoint. Represents one authenticated browser/client connection. Implements `ConnectionManager.UpdateListener` so it receives push notifications for device/position/event changes and writes them as JSON to the WebSocket session.
**Fits in:** Instantiated by [[AsyncSocketServlet.java]] in the WebSocket creator. Registered with `ConnectionManager` on open; deregistered on close.
**Read next:** [[AsyncSocketServlet.java]] (creates instances), `ConnectionManager.java` (the hub that calls `onUpdateX` callbacks), `PositionUtil.java` (initial position snapshot on connect)

## Public API

- `AsyncSocket(ObjectMapper, ConnectionManager, Storage, userId)` (line 57) — constructor; one instance per connection.
- `onWebSocketOpen(Session)` (line 65) — registers with `ConnectionManager`; sends initial positions snapshot.
- `onWebSocketClose(int, String, Callback)` (line 78) — unregisters from `ConnectionManager`.
- `onWebSocketText(String)` (line 85) — client-to-server messaging; currently only handles `{"logs": true/false}` toggle.
- `onKeepalive()` (line 104) — sends empty `{}` ping to keep connection alive.
- `onUpdateDevice(Device)` (line 109) — pushes `{"devices": [device]}`.
- `onUpdatePosition(Position)` (line 114) — pushes `{"positions": [position]}`.
- `onUpdateEvent(Event)` (line 119) — pushes `{"events": [event]}`.
- `onUpdateLog(LogRecord)` (line 124) — pushes `{"logs": [record]}` only if client opted in.

## Key flows

### Connection lifecycle
1. Client upgrades HTTP to WS at `/api/socket`.
2. `AsyncSocketServlet` verifies auth (token param or session cookie) → creates `AsyncSocket(userId)`.
3. `onWebSocketOpen`: loads all latest positions for this userId via `PositionUtil.getLatestPositions`, sends them as the initial `{"positions": [...]}` burst. Then calls `connectionManager.addListener(userId, this)`.
4. From that point every position/device/event update arriving via `ConnectionManager` is pushed immediately.

### Wire format
All messages are JSON objects with one key from `{devices, positions, events, logs}`, each mapping to an array. Keepalive = `{}`. Client-to-server = `{"logs": true}` to enable log streaming.

### Log opt-in (line 54, 124-128)
`includeLogs` flag defaults false. Client sends `{"logs": true}` to start receiving `LogRecord` pushes. Useful for admin live-tail feature in the web UI.

## Gotchas / non-obvious

- **`Session.Listener.AutoDemanding`** (line 40) — Jetty 12 interface; auto-demands the next WebSocket frame after each callback. Without this the connection would pause reading after the first message.
- **`ClosedChannelException` is suppressed** (line 98-100) — normal during connection teardown; only unexpected errors are logged.
- **Initial snapshot is positions-only**, not devices. Devices arrive via `ConnectionManager` callbacks as they update (or the client fetches them separately via REST).
- **`userId` is captured at connect time** — no re-auth mid-connection. If the user is deleted or disabled, events still flow until the WS is closed.

## Line index

- 40 — class declaration + interfaces
- 44-47 — message key constants
- 57 — constructor
- 65-75 — `onWebSocketOpen` (snapshot + register)
- 78-82 — `onWebSocketClose` (unregister)
- 85-94 — `onWebSocketText` (log toggle)
- 104-106 — keepalive (empty ping)
- 109-128 — `onUpdateX` callback implementations
- 130-138 — `sendData` helper (null/open guard + JSON serialize)
