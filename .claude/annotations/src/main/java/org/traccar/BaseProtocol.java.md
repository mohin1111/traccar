# BaseProtocol.java

**Role:** Abstract base for all 265 protocol implementations. Holds the protocol name, the list of `TrackerConnector` instances (servers/clients), supported command types, and command dispatch logic (data vs. SMS).
**Fits in:** Every `*Protocol.java` in `org.traccar.protocol` extends this. `ServerManager` instantiates subclasses via Guice and calls `getConnectorList()`. The `Protocol` interface is the narrow contract; this class provides the full implementation.
**Read next:** [[Protocol.java]] (interface), [[ServerManager.java]] (discovery + lifecycle), [[TrackerServer.java]] (connector added via `addServer`), [[BaseProtocolDecoder.java]] (decoder holds back-ref to Protocol)

## Public API

- `nameFromClass(Class<?> clazz)` (line 51) — strips "Protocol" suffix and lowercases. `ItsProtocol` → `"its"`. Used by `ServerManager` to match config keys.
- `getName()` (line 65) — returns the lowercased protocol name.
- `addServer(TrackerServer server)` (line 70) — subclass calls in constructor to register a TCP or UDP server.
- `addClient(TrackerClient client)` (line 74) — for outbound-polling protocols.
- `getConnectorList()` (line 79) — returns all registered connectors; `ServerManager` aggregates these.
- `setSupportedDataCommands(String... commands)` (line 83) — declare which `Command.TYPE_*` constants this protocol handles in-band.
- `setSupportedTextCommands(String... commands)` (line 87) — declare SMS-encodable commands.
- `sendDataCommand(Channel channel, SocketAddress remoteAddress, Command command)` (line 106) — dispatches command to device: encodes known types via protocol encoder, or sends raw hex for `TYPE_CUSTOM`.
- `sendTextCommand(String destAddress, Command command)` (line 132) — sends SMS via `SmsManager` + `StringProtocolEncoder`.
- `setTextCommandEncoder(StringProtocolEncoder)` (line 127) — link the SMS encoder.

## Key flows

### Command dispatch (lines 106-125)
```
sendDataCommand:
  if supportedDataCommands.contains(command.getType()):
    message = command          // encoder in pipeline handles it
  elif TYPE_CUSTOM:
    if StringEncoder in pipeline:
      message = data as String
    else:
      message = data as hex ByteBuf
  else:
    throw RuntimeException

channel.writeAndFlush(new NetworkMessage(message, remoteAddress)).sync()
```
The `.sync()` blocks until the write is acknowledged by the OS — commands are synchronous.

### SMS text command (lines 132-150)
Requires `smsManager != null` (injected, may be null if SMS not configured). Delegates encoding to `textCommandEncoder.encodeCommand(command)`. Returns null → encode failed → exception.

## Gotchas / non-obvious

- `Command.TYPE_CUSTOM` is always added to `getSupportedDataCommands()` (line 93) — every protocol supports raw hex/string commands without custom encoder logic.
- `setSupportedDataCommands` should be called in the subclass constructor **before** `addServer()` — ordering doesn't technically matter since the list is read only at command dispatch time, but keep it consistent.
- `MAX_FRAME_LENGTH = 1024`, `MAX_FRAME_LENGTH_LARGE = 32 * 1024` — constants used by frame decoders to prevent memory exhaustion.
- `@Inject setSmsManager(@Nullable SmsManager)` — injected via method injection; null if SMS not configured. Checked at `sendTextCommand` call time.

## Line index

- 38-39 — `MAX_FRAME_LENGTH` / `MAX_FRAME_LENGTH_LARGE` constants
- 51-54 — `nameFromClass` (strips "Protocol", lowercase)
- 56-58 — no-arg constructor (sets name via reflection)
- 60-63 — `@Inject setSmsManager`
- 70-78 — `addServer` / `addClient` / `getConnectorList`
- 83-103 — `setSupportedDataCommands`, `setSupportedTextCommands`, getters (always include `TYPE_CUSTOM`)
- 106-125 — `sendDataCommand` dispatch
- 127-129 — `setTextCommandEncoder`
- 132-150 — `sendTextCommand` (SMS)
