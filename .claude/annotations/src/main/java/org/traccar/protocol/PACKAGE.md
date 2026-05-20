# `org/traccar/protocol/` — The 265-protocol decoder layer

This package contains every protocol implementation: frame decoders, protocol decoders, and protocol encoders. It is the largest package in the codebase (671 files, 265 protocol families). Every file follows the same `BaseProtocol` / `BaseProtocolDecoder` pattern. **Only the ITS (AIS-140 / Indian VLT) files are individually annotated here** — they are the highest-value protocols for Transport OS.

## Package layout

```
protocol/
├── ItsFrameDecoder.java       ← annotated (AIS-140)
├── ItsProtocol.java           ← annotated (AIS-140)
├── ItsProtocolDecoder.java    ← annotated (AIS-140) ← read this for AIS-140 data model
├── ItsProtocolEncoder.java    ← annotated (AIS-140)
├── ItsProtocolDecoderTest.java   (test, in src/test/)
├── <264 other *Protocol.java>
├── <264 other *ProtocolDecoder.java>
└── <N *ProtocolEncoder.java, *FrameDecoder.java, ...>
```

## ITS (AIS-140) file index

| File | One-liner | Annotation |
|---|---|---|
| `ItsProtocol.java` | Pipeline setup: `ItsFrameDecoder` → `StringDecoder` → `ItsProtocolDecoder`; ENGINE_STOP/RESUME | [ItsProtocol.java.md](ItsProtocol.java.md) |
| `ItsFrameDecoder.java` | Frame boundary detector: handles `\r\n` and `*` terminators, skips binary checksums | [ItsFrameDecoder.java.md](ItsFrameDecoder.java.md) |
| `ItsProtocolDecoder.java` | Full AIS-140 position/event/alarm/cell-tower decoder (285 lines, 5 frame variants) | [ItsProtocolDecoder.java.md](ItsProtocolDecoder.java.md) |
| `ItsProtocolEncoder.java` | `ENGINE_STOP` → `@SET#RLP,OP1,` / `ENGINE_RESUME` → `@CLR#RLP,OP1,` | [ItsProtocolEncoder.java.md](ItsProtocolEncoder.java.md) |

## The `BaseProtocol` / `BaseProtocolDecoder` pattern

Every protocol family follows this exact structure:

```
FooProtocol extends BaseProtocol
  └── constructor(@Inject Config config):
        setSupportedDataCommands(...)      // optional
        setSupportedTextCommands(...)      // optional
        addServer(new TrackerServer(config, getName(), isDatagram) {
            @Override
            protected void addProtocolHandlers(PipelineBuilder pipeline, Config config) {
                pipeline.addLast(new FooFrameDecoder());   // optional
                pipeline.addLast(new StringEncoder());      // text protocols
                pipeline.addLast(new StringDecoder());      // text protocols
                pipeline.addLast(new FooProtocolDecoder(FooProtocol.this));
            }
        });

FooProtocolDecoder extends BaseProtocolDecoder
  └── decode(Channel channel, SocketAddress remoteAddress, Object msg):
        // 1. Parse raw bytes/string
        // 2. Extract IMEI/identifier
        DeviceSession session = getDeviceSession(channel, remoteAddress, imei);
        if (session == null) return null;
        // 3. Create and populate Position
        Position position = new Position(getProtocolName());
        position.setDeviceId(session.getDeviceId());
        // 4. Return position (or Collection<Position>)
```

## How `ServerManager` auto-discovers all 265 protocols

```java
// ServerManager constructor:
for (Class<?> protocolClass : ClassScanner.findSubclasses(BaseProtocol.class, "org.traccar.protocol")) {
    String name = BaseProtocol.nameFromClass(protocolClass);   // "ItsProtocol" → "its"
    if (config.getInteger(name + ".port") > 0) {
        BaseProtocol protocol = injector.getInstance(protocolClass);
        connectorList.addAll(protocol.getConnectorList());
    }
}
```

**No manual registration.** Adding a class → sets config port → protocol starts automatically.

## Key protocols relevant to Transport OS / India

| Protocol name | Class | Notes |
|---|---|---|
| `its` | `ItsProtocol` | **AIS-140 VLT** — primary Indian VLT standard |
| `teltonika` | `TeltonikaProtocol` | Teltonika FMxxx — widely deployed globally + India |
| `jt808` | `Jt808Protocol` | JT/T 808 Chinese standard — Chinese-OEM hardware common in India |
| `atrack` | `AtractProtocol` | Atrack AX series (binary + text) |
| `aquila` | `AquilaProtocol` | Aquila VT series — Indian OEM |
| `idpl` | `IdplProtocol` | IDPL — Indian OEM |
| `racedynamics` | `RaceDynamicsProtocol` | RaceDynamics — India |
| `totem` | `TotemProtocol` | Totem — popular with Indian fleet operators |
| `osmand` | `OsmAndProtocol` | HTTP GET (port 5055) — Traccar client + mobile apps |
| `gps103` | `Gps103Protocol` | TK102/TK103 clones — cheap hardware common in India |

## AIS-140 data fields decoded by `ItsProtocolDecoder`

From a fully-populated NRM frame, `ItsProtocolDecoder` extracts:

**Core positioning:**
- `latitude`, `longitude` (decimal degrees, signed)
- `speed` (knots, converted from km/h)
- `course` (degrees 0-359)
- `altitude` (meters)
- `time` (device timestamp, device timezone)
- `valid` (GPS fix validity)

**AIS-140 mandatory fields:**
- `KEY_IGNITION` — ignition state (boolean)
- `KEY_POWER` — external power voltage (V)
- `KEY_BATTERY` — backup battery voltage (V)
- `KEY_CHARGE` — charging state (boolean)
- `"emergency"` — panic/SOS flag (boolean)
- `KEY_ODOMETER` — odometer (meters)
- `KEY_INPUT` — digital inputs bitmask (4 bits)
- `KEY_OUTPUT` — digital outputs bitmask (2 bits)
- `network` — cell towers (MCC/MNC/LAC/CID + up to 4 neighbors)

**Extended fields:**
- `KEY_SATELLITES` — satellite count
- `KEY_PDOP` / `KEY_HDOP` — precision
- `KEY_OPERATOR` — GSM network operator name
- `PREFIX_ADC + 1` — analog input (ADC1)
- `KEY_G_SENSOR` — 3-axis accelerometer `[x, y, z]`
- `"tiltY"` / `"tiltX"` — tilt angles
- `KEY_EVENT` — event code
- `KEY_ARCHIVE` — `true` if historical (buffered) message
- `addAlarm(ALARM_SOS)` — when status=WD/EA or type=EMR

**Alarm types decoded (from status field):**
`WD/EA→SOS`, `BL→low_battery`, `HB→braking`, `HA→acceleration`, `RT→cornering`, `OS→overspeed`, `TA→tampering`, `BD→power_cut`, `BR→power_restored`

## Frame format summary

```
$,<event>,<vendor>,<fw>,<status>[,<eventCode>],<history>,<IMEI>,<date>,<time>,
<lat>,<N/S>,<lon>,<E/W>,<speed>,<course>,<satellites>,<altitude>,
<PDOP>,<HDOP>,<operator>,<ignition>,<charging>,<power>,<battery>,<emergency>,
<cells>,<inputs>,<outputs>[,<index>,<odometer>,<adc1>,<ax>,<ay>,<az>,<tiltY>,<tiltX>]
```

Multiple variants handled: NRM (full), EPB (minimal lat/lon/alt/speed), RLP (odometer + checksum), C (ADC1+ADC2). Framed by `\r\n` or `*`.

Handshake frames (not position data):
- `$,01,...` — registration → ACK `"$,1,*"`
- `$,LGN,...` — login → ACK `"$LGN" + ddMMyyyyHHmmss + "*"`
- `$,HBT,...` — heartbeat → ACK `"$HBT*"`

## How to add a new protocol (step-by-step)

1. **Create `FooProtocol.java`** in `org.traccar.protocol`:
   ```java
   public class FooProtocol extends BaseProtocol {
       @Inject
       public FooProtocol(Config config) {
           addServer(new TrackerServer(config, getName(), false) {
               @Override
               protected void addProtocolHandlers(PipelineBuilder pipeline, Config config) {
                   pipeline.addLast(new CharacterDelimiterFrameDecoder(1024, '\n'));
                   pipeline.addLast(new StringDecoder());
                   pipeline.addLast(new FooProtocolDecoder(FooProtocol.this));
               }
           });
       }
   }
   ```

2. **Create `FooProtocolDecoder.java`** in `org.traccar.protocol`:
   ```java
   public class FooProtocolDecoder extends BaseProtocolDecoder {
       public FooProtocolDecoder(Protocol protocol) { super(protocol); }

       @Override
       protected Object decode(Channel channel, SocketAddress remoteAddress, Object msg) throws Exception {
           String sentence = (String) msg;
           String imei = /* extract from sentence */;
           DeviceSession session = getDeviceSession(channel, remoteAddress, imei);
           if (session == null) return null;

           Position position = new Position(getProtocolName());
           position.setDeviceId(session.getDeviceId());
           position.setTime(/* parse timestamp */);
           position.setLatitude(/* lat */);
           position.setLongitude(/* lon */);
           position.setSpeed(UnitsConverter.knotsFromKph(/* speed in km/h */));
           position.setValid(true);
           return position;
       }
   }
   ```

3. **Set config:** `foo.port = <portNumber>` in `traccar.xml`.

4. **Add test:** `FooProtocolDecoderTest extends ProtocolTest` using `verifyPosition(decoder, text("..."))` or `verifyPosition(decoder, binary("..."))`.

5. **No other changes** — `ServerManager` discovers `FooProtocol` automatically via `ClassScanner`.

## How to add a new command type to an existing protocol

1. Call `setSupportedDataCommands(Command.TYPE_NEW_CMD)` in the protocol constructor.
2. Add a `case Command.TYPE_NEW_CMD:` in the encoder's `encodeCommand()`.
3. If the command returns text: ensure `StringEncoder` is in the pipeline (for automatic string-to-bytes encoding).
4. If binary: return a `ByteBuf` from `encodeCommand()`.
5. Restart server; the command type is now available via REST `POST /api/commands`.

## Conventions

- Protocol name = class name minus "Protocol" suffix, lowercased. `ItsProtocol` → `"its"`.
- Config prefix: `<protocolname>.<key>` (e.g., `its.port=5050`, `its.timeout=600`).
- All speeds stored in knots in `Position`. Convert with `UnitsConverter.knotsFromKph()`.
- All distances in meters in `Position`. Convert from km with `× 1000`.
- Timestamps: call `position.setTime(parser.nextDateTime(...))` or `new Date()` — not `System.currentTimeMillis()`.
- Binary protocols: use `buf.readUnsignedByte()`, `buf.readUnsignedShort()` (unsigned!), not signed variants.
- Text protocols: use `helper/Parser.java` + `helper/PatternBuilder.java` for regex-based parsing.
