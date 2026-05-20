# `org/traccar/` — Core boot, pipeline, and protocol infrastructure

The root package is the engine room. It contains exactly two responsibilities: (1) bootstrapping the application via Guice DI and starting services, (2) defining the abstract Netty pipeline and protocol decoder/encoder contracts that all 265 protocol implementations follow.

## File index

| File | One-liner | Annotation |
|---|---|---|
| `Main.java` | Entry point; Guice bootstrap; service lifecycle; Windows service mode | [Main.java.md](Main.java.md) |
| `MainModule.java` | All Guice `@Singleton @Provides` bindings — geocoders, forwarders, handlers, SMS, mail | [MainModule.java.md](MainModule.java.md) |
| `ServerManager.java` | Auto-discovers `*Protocol` classes via `ClassScanner`; starts/stops all connectors | [ServerManager.java.md](ServerManager.java.md) |
| `BasePipelineFactory.java` | Abstract `ChannelInitializer`; wires fixed infrastructure layers + delegates to subclass | [BasePipelineFactory.java.md](BasePipelineFactory.java.md) |
| `ProcessingHandler.java` | `@Singleton @Sharable` handler; per-position enrichment chain + event handlers + DB persist | [ProcessingHandler.java.md](ProcessingHandler.java.md) |
| `BaseProtocol.java` | Abstract base for all 265 protocols; name derivation, connector registration, command dispatch | [BaseProtocol.java.md](BaseProtocol.java.md) |
| `BaseProtocolDecoder.java` | Abstract decoder; `getDeviceSession()`, speed/TZ helpers, device-online callback | [BaseProtocolDecoder.java.md](BaseProtocolDecoder.java.md) |
| `BaseProtocolEncoder.java` | Abstract encoder; intercepts `Command` in outbound pipeline, delegates to `encodeCommand()` | [BaseProtocolEncoder.java.md](BaseProtocolEncoder.java.md) |
| `BaseFrameDecoder.java` | Abstract `ByteToMessageDecoder` + `TooLongFrameException` guard | [BaseFrameDecoder.java.md](BaseFrameDecoder.java.md) |
| `BaseHttpProtocolDecoder.java` | HTTP decoder base; adds `sendResponse()` helpers | [BaseHttpProtocolDecoder.java.md](BaseHttpProtocolDecoder.java.md) |
| `BaseMqttProtocolDecoder.java` | MQTT decoder base; handles CONNECT/SUBSCRIBE/PUBLISH internally | [BaseMqttProtocolDecoder.java.md](BaseMqttProtocolDecoder.java.md) |
| `BaseProtocolPoller.java` | Polling-style handler; schedules periodic `sendRequest()` on channel activation | [BaseProtocolPoller.java.md](BaseProtocolPoller.java.md) |
| `CharacterDelimiterFrameDecoder.java` | Netty `DelimiterBasedFrameDecoder` wrapper with char/String/String[] constructors | [CharacterDelimiterFrameDecoder.java.md](CharacterDelimiterFrameDecoder.java.md) |
| `EventLoopGroupFactory.java` | Singleton; creates and holds Netty boss + worker `EventLoopGroup` | [EventLoopGroupFactory.java.md](EventLoopGroupFactory.java.md) |
| `ExtendedObjectDecoder.java` | Direct parent of `BaseProtocolDecoder`; implements `channelRead`, calls `decode()`, fires downstream | [ExtendedObjectDecoder.java.md](ExtendedObjectDecoder.java.md) |
| `LifecycleObject.java` | Interface: `start()` + `stop()`. Implemented by the four top-level services | [LifecycleObject.java.md](LifecycleObject.java.md) |
| `NetworkMessage.java` | Value object: `(Object message, SocketAddress remoteAddress)` — universal pipeline envelope | [NetworkMessage.java.md](NetworkMessage.java.md) |
| `PipelineBuilder.java` | Functional interface: `addLast(ChannelHandler)` — lambda target for pipeline assembly | [PipelineBuilder.java.md](PipelineBuilder.java.md) |
| `Protocol.java` | Interface: name, connectors, supported commands, `sendDataCommand`, `sendTextCommand` | [Protocol.java.md](Protocol.java.md) |
| `StringProtocolEncoder.java` | Encoder base for text protocols; `formatCommand(Command, format, keys...)` helper | [StringProtocolEncoder.java.md](StringProtocolEncoder.java.md) |
| `TrackerClient.java` | Outbound TCP connector; reconnect loop; abstract `addProtocolHandlers` | [TrackerClient.java.md](TrackerClient.java.md) |
| `TrackerConnector.java` | Interface: `isDatagram()`, `isSecure()`, `getChannelGroup()` + `LifecycleObject` | [TrackerConnector.java.md](TrackerConnector.java.md) |
| `TrackerServer.java` | Inbound TCP/UDP listener; `ServerBootstrap`/`Bootstrap`; abstract `addProtocolHandlers` | [TrackerServer.java.md](TrackerServer.java.md) |
| `WindowsService.java` | JNA-based Windows SCM integration; install/uninstall/run as Windows Service | [WindowsService.java.md](WindowsService.java.md) |
| `WrapperContext.java` | `ChannelHandlerContext` decorator; wraps raw messages in `NetworkMessage` on fire/write | [WrapperContext.java.md](WrapperContext.java.md) |
| `WrapperInboundHandler.java` | Unwraps `NetworkMessage` before delegating to wrapped handler; exposes `getWrappedHandler()` | [WrapperInboundHandler.java.md](WrapperInboundHandler.java.md) |
| `WrapperOutboundHandler.java` | Outbound counterpart to `WrapperInboundHandler`; same unwrap/rewrap for writes | [WrapperOutboundHandler.java.md](WrapperOutboundHandler.java.md) |

## Boot sequence

```
java -jar tracker-server.jar config.xml
  │
  ├── Main.main()
  │     └── Main.run(configFile)
  │           ├── Guice.createInjector(MainModule, DatabaseModule, WebModule)
  │           │     ├── Config.class ← config.xml (eager singleton)
  │           │     ├── Liquibase migrations (inside DatabaseModule.provideDataSource)
  │           │     └── All @Singleton providers in MainModule resolved lazily
  │           │
  │           ├── ScheduleManager.start()    (background task scheduler)
  │           ├── ServerManager.start()      (opens all device ports ← THE gateway)
  │           │     └── ClassScanner.findSubclasses(BaseProtocol.class, "org.traccar.protocol")
  │           │           └── for each *Protocol with its.port > 0:
  │           │                 injector.getInstance(ItsProtocol.class)   ← calls constructor
  │           │                   └── addServer(new TrackerServer(config, "its", false))
  │           │                         └── creates Netty ServerBootstrap
  │           │                               └── BasePipelineFactory.initChannel() per connection
  │           │
  │           ├── WebServer.start()           (REST + WebSocket on web.port)
  │           └── BroadcastService.start()    (multi-node fan-out)
```

## Netty channel pipeline (per accepted TCP connection)

```
[Transport Layer]
  SslHandler                           ← if <protocol>.ssl=true
  <protocol-specific transport>        ← addTransportHandlers (HTTP codec, MQTT codec, etc.)

[Infrastructure Layer — fixed by BasePipelineFactory]
  IdleStateHandler(timeout, 0, 0)      ← close idle TCP channels
  OpenChannelHandler                   ← register with channelGroup
  NetworkForwarderHandler              ← mirror raw bytes (optional, server.forward)
  NetworkMessageHandler                ← wrap DatagramPacket → NetworkMessage
  StandardLoggingHandler               ← hex dump (debug)
  AcknowledgementHandler               ← delay TCP ACK (optional, server.delayAck)

[Protocol Layer — addProtocolHandlers]
  WrapperInboundHandler[ItsFrameDecoder]   ← ByteBuf stream → ByteBuf frames
  WrapperInboundHandler[StringDecoder]     ← ByteBuf frame → String
  WrapperOutboundHandler[StringEncoder]    ← String → ByteBuf (outbound only)
  ItsProtocolDecoder   ← String → Position (Guice-injected, receives NetworkMessage)

[Processing Layer — fixed by BasePipelineFactory]
  RemoteAddressHandler                 ← stamp remote IP on Position
  ProcessingHandler                    ← enrichment chain + event handlers + DB
  MainEventHandler                     ← write protocol ACK to device
```

The `WrapperInboundHandler`/`WrapperOutboundHandler` pattern is what lets non-decoder handlers (frame decoders, string codecs) see raw `ByteBuf`/`String` while the rest of the pipeline uses `NetworkMessage`.

## Guice DI model

- **Framework:** Google Guice (not Spring). `@Singleton @Provides` in `MainModule`.
- **Three modules:** `MainModule` (app-level), `DatabaseModule` (HikariCP + Liquibase), `WebModule` (Jersey + Jetty).
- **Protocol injection:** `ServerManager` calls `injector.getInstance(protocolClass)` for each discovered protocol. Protocol constructors receive `Config` via `@Inject`. Protocol decoders receive `CacheManager`, `ConnectionManager`, etc. via `@Inject` setter methods.
- **Nullable pattern:** Optional services (geocoder, geolocation handler, filter handler, etc.) are bound as `null` when disabled. `ProcessingHandler` filters them out with `Stream.filter(Objects::nonNull)`.
- **Static injector access:** `Main.getInjector()` is a static field used by `TrackerServer`/`TrackerClient` constructors (called during `ServerManager` init, before DI graph is fully resolved for those classes).

## Protocol auto-discovery (the "no registration" guarantee)

```
ServerManager constructor:
  ClassScanner.findSubclasses(BaseProtocol.class, "org.traccar.protocol")
    → returns all classes in the JAR whose superclass chain includes BaseProtocol
    → for each: nameFromClass(clazz) → lowercase protocol name
    → check: config.getInteger("<name>.port") > 0
    → if yes: injector.getInstance(clazz) → protocol constructor runs
              → addServer/addClient called → TrackerServer/TrackerClient created
              → connectorList populated
```

**Adding a new protocol requires zero changes to `ServerManager`.** Just:
1. Create `FooProtocol extends BaseProtocol` in `org.traccar.protocol`.
2. Create `FooProtocolDecoder extends BaseProtocolDecoder`.
3. Set `foo.port = <N>` in config.
4. Done — `ClassScanner` finds it on next startup.

## `ProcessingHandler` — the per-position chain

18 position handlers (in order), 12 event handlers (in parallel after position chain). See [[ProcessingHandler.java.md]] for the full ordered list. Key invariant: per-device queue ensures positions for the same device are processed sequentially even though Netty channels are concurrent. The async callback pattern (`BasePositionHandler.Callback`) allows geocoding/map-matching to be non-blocking without blocking Netty threads.

## Multi-file flows

### Position lifecycle (full path)
```
TCP bytes → ItsFrameDecoder → StringDecoder → ItsProtocolDecoder.decode()
  → getDeviceSession(channel, remoteAddress, imei)  ← resolves IMEI → deviceId
  → Position populated with coords/speed/ignition/cells/odometer/...
  → ExtendedObjectDecoder fires Position → RemoteAddressHandler stamps IP
  → ProcessingHandler:
      ComputedAttributesHandler.Early → OutdatedHandler → TimeHandler
      → GeolocationHandler → HemisphereHandler → MapMatcherHandler
      → DistanceHandler → FilterHandler → GeofenceHandler → GeocoderHandler
      → SpeedLimitHandler → MotionHandler → ComputedAttributesHandler.Late
      → DriverHandler → CopyAttributesHandler → EngineHoursHandler
      → PositionForwardingHandler → DatabaseHandler
      → [event handlers: Overspeed, Ignition, Alarm, ...]
      → postProcessHandler → AcknowledgementHandler.EventHandled
  → MainEventHandler writes TCP ACK to device
```

### Command sending (full path)
```
REST POST /api/commands → CommandResource → CommandsManager.sendCommand()
  → ServerManager.getProtocol("its") → ItsProtocol.sendDataCommand(channel, remoteAddress, command)
  → BaseProtocol.sendDataCommand:
      if ENGINE_STOP: StringEncoder in pipeline detected
        → writes "@SET#RLP,OP1," as NetworkMessage to channel
  → ItsProtocolDecoder.sendQueuedCommands called on next device message
    (commands queued if device offline, flushed on reconnect)
```

## Conventions

- **All protocol-level constants in `BaseProtocol`:** `MAX_FRAME_LENGTH=1024`, `MAX_FRAME_LENGTH_LARGE=32768`, `MAX_HTTP_LENGTH=65536`.
- **Speed always in knots** in `Position`. Decoders must call `UnitsConverter.knotsFromKph()` (or `convertSpeed()`). Never store raw km/h.
- **Timezone default is UTC** for all position timestamps unless `decoder.timezone` device attribute is set.
- **`Locale.ENGLISH` is forced** at JVM startup — number parsing in decoders is locale-independent.
- **`database.saveOriginal=true`** stores hex-dumped raw frame in `Position.KEY_ORIGINAL` for audit.

## How to add a new protocol

1. Create `src/main/java/org/traccar/protocol/FooProtocol.java extends BaseProtocol`:
   ```java
   @Inject
   public FooProtocol(Config config) {
       setSupportedDataCommands(Command.TYPE_ENGINE_STOP);
       addServer(new TrackerServer(config, getName(), false) {
           @Override
           protected void addProtocolHandlers(PipelineBuilder pipeline, Config config) {
               pipeline.addLast(new FooFrameDecoder());
               pipeline.addLast(new StringDecoder());
               pipeline.addLast(new FooProtocolDecoder(FooProtocol.this));
           }
       });
   }
   ```
2. Create `FooProtocolDecoder extends BaseProtocolDecoder`:
   ```java
   @Override
   protected Object decode(Channel channel, SocketAddress remoteAddress, Object msg) throws Exception {
       String sentence = (String) msg;
       // ... parse IMEI ...
       DeviceSession deviceSession = getDeviceSession(channel, remoteAddress, imei);
       if (deviceSession == null) return null;
       Position position = new Position(getProtocolName());
       position.setDeviceId(deviceSession.getDeviceId());
       // ... populate position fields ...
       return position;
   }
   ```
3. Add `foo.port = <N>` to `traccar.xml` / `debug.xml`.
4. Add `FooProtocolDecoderTest extends ProtocolTest`.
5. No other changes needed.

## How to add a new position handler to the processing chain

1. Create `FooHandler extends BasePositionHandler` in `src/main/java/org/traccar/handler/`.
2. Implement `handlePosition(Position position, Callback callback)`. Call `callback.processed(false)` to continue, `callback.processed(true)` to filter the position.
3. Add `@Singleton @Provides` in `MainModule.java` returning null if feature is disabled.
4. Add the class to the `positionHandlers` list in `ProcessingHandler` constructor at the correct position.
