# ItsProtocol.java

**Role:** Bootstrap class for the AIS-140 / Indian VLT ITS protocol. Registers `ENGINE_STOP` and `ENGINE_RESUME` command support, creates a single TCP `TrackerServer`, and wires the pipeline: `ItsFrameDecoder` → `StringEncoder` → `StringDecoder` → `ItsProtocolDecoder`.
**Fits in:** Extends `BaseProtocol`. Auto-discovered by `ServerManager` via `ClassScanner`. Active when `its.port` is set in config. The highest-value protocol for Indian VLT / Transport OS AIS-140 compliance.
**Read next:** [[ItsFrameDecoder.java]] (byte → text frame), [[ItsProtocolDecoder.java]] (text frame → `Position`), [[ItsProtocolEncoder.java]] (command → text string), [[BaseProtocol.java]] (parent)

## Public API

- `ItsProtocol(Config config)` — `@Inject` constructor. Sets two supported data commands; creates one TCP server.
- Protocol name: `"its"` (derived by `BaseProtocol.nameFromClass`).
- Config keys: `its.port` (required), `its.address` (optional bind address), `its.ssl` (optional), `its.timeout`.

## Key flows

### Pipeline (lines 37-43)
```
ItsFrameDecoder      — ByteBuf → ByteBuf (complete frame, handles \r\n and * delimiters)
StringEncoder        — outbound String → ByteBuf
StringDecoder        — ByteBuf → String (UTF-8)
ItsProtocolDecoder   — String → Position (or null for handshakes)
```

The encoder `StringEncoder` (from Netty) is listed before `StringDecoder` in the pipeline. For inbound: only `ItsFrameDecoder` and `StringDecoder` are active. For outbound (command sending): `StringEncoder` is active. `ItsProtocolEncoder` is NOT added to this pipeline — command encoding is via `BaseProtocol.sendDataCommand` which detects `StringEncoder` and passes the string directly.

## Gotchas / non-obvious

- `ItsProtocolEncoder` is a standalone class but NOT added to the pipeline in `ItsProtocol`. The protocol uses `setSupportedDataCommands(ENGINE_STOP, ENGINE_RESUME)` + detects `StringEncoder` in pipeline (via `BasePipelineFactory.getHandler`) — the encoder string is what gets written. The `ItsProtocolEncoder.encodeCommand()` method is only used if wired via `setTextCommandEncoder()` for SMS delivery. Study `BaseProtocol.sendDataCommand` lines 106-125 carefully.
- `ENGINE_RESUME` (immobilizer relay off) is the paired reverse of `ENGINE_STOP`.
- TCP only (`datagram=false`). AIS-140 mandates TCP for reliable delivery.

## Line index

- 32-34 — `setSupportedDataCommands(ENGINE_STOP, ENGINE_RESUME)`
- 35-43 — `addServer(new TrackerServer(...))` with pipeline wiring
- 38 — `ItsFrameDecoder()`
- 39 — `StringEncoder`
- 40 — `StringDecoder`
- 41 — `ItsProtocolDecoder(ItsProtocol.this)`
