# BaseProtocolEncoder.java

**Role:** Abstract base for all protocol command encoders. Intercepts outbound `NetworkMessage` objects containing a `Command`, delegates to `encodeCommand()` (subclass), logs the result, and writes back a new `NetworkMessage` with the encoded payload.
**Fits in:** `ChannelOutboundHandlerAdapter` placed in the protocol handler section of the pipeline (in `addProtocolHandlers`). Receives Guice injection for `CacheManager`. Subclasses override `encodeCommand(Command)` or `encodeCommand(Channel, Command)`.
**Read next:** [[BaseProtocol.java]] (calls `sendDataCommand` which writes the `Command` as `NetworkMessage`), [[StringProtocolEncoder.java]] (text-command helper subclass), [[BaseProtocolDecoder.java]] (inbound counterpart)

## Public API

- `encodeCommand(Command command)` (line 108) — override in subclass; return encoded string/ByteBuf or null if not supported. Default returns null.
- `encodeCommand(Channel channel, Command command)` (line 104) — channel-aware variant; calls `encodeCommand(command)` by default. Override when encoding depends on channel state.
- `getUniqueId(long deviceId)` (line 61) — looks up the device's unique identifier (IMEI) from cache. Used in command formatting.
- `initDevicePassword(Command command, String defaultPassword)` (line 65) — populates `KEY_DEVICE_PASSWORD` from device attributes if not already set.
- `getDeviceModel(long deviceId)` (line 77) — returns model string with optional override.

## Key flows

### `write` interception (lines 83-102)
```
write(ctx, msg, promise):
  if msg instanceof NetworkMessage && message instanceof Command:
    encodedCommand = encodeCommand(channel, command)
    log id + command type + sent/not-sent
    ctx.write(new NetworkMessage(encodedCommand, remoteAddress), promise)
    return
  super.write(...)   // pass through non-command messages
```
If `encodedCommand` is null, the message is still written (as null) — the downstream codec handles null gracefully, effectively a no-op.

## Gotchas / non-obvious

- **Two overload variants** — most protocols only override `encodeCommand(Command)`. Override the `Channel`-aware variant only if you need to inspect channel state (e.g., TLS vs plain, current session context).
- **`CacheManager` access** — injected via `@Inject setCacheManager`. `getUniqueId` / `getDeviceModel` need it; don't call them before injection completes.
- **Null encoded command** is not an error — it means "don't send." The log shows "not sent". The `CommandsManager` may retry or the command is marked as failed depending on caller.

## Line index

- 44 — constructor (stores protocol ref)
- 53-55 — `@Inject setCacheManager`
- 61-63 — `getUniqueId`
- 65-71 — `initDevicePassword`
- 77-80 — `getDeviceModel`
- 83-102 — `write` (intercept + log + re-wrap)
- 104-106 — `encodeCommand(Channel, Command)` — default delegates
- 108-110 — `encodeCommand(Command)` — override point, default null
