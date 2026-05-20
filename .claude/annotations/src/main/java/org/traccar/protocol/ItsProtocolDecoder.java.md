# ItsProtocolDecoder.java

**Role:** Decodes AIS-140 / ITS protocol text frames into Traccar `Position` objects. Handles login, heartbeat, and position message types. Parses multi-variant frame formats (NRM, EPB, RLP, C subtypes) via a single composite regex pattern.
**Fits in:** Extends `BaseProtocolDecoder`. Receives UTF-8 strings from `StringDecoder` in the pipeline. This is the highest-value decoder for Transport OS AIS-140 compliance — annotated in depth.
**Read next:** [[ItsFrameDecoder.java]] (upstream framer), [[ItsProtocol.java]] (pipeline setup), [[ItsProtocolEncoder.java]] (reverse: command → string), [[BaseProtocolDecoder.java]] (parent class)

## Public API

- `ItsProtocolDecoder(Protocol protocol)` — constructor; protocol ref passed through.
- `decode(Channel channel, SocketAddress remoteAddress, Object msg)` — main decode method. Receives `String` from pipeline.

## Key flows

### Handshake handling (lines 141-149)
Before regex parsing, three handshake types are detected and ACKed:
- `$,01,...` → responds `"$,1,*"` (registration/login)
- `$,LGN,...` → responds `"$LGN" + timestamp + "*"` (login with time sync); timestamp is `ddMMyyyyHHmmss` UTC
- `$,HBT,...` → responds `"$HBT*"` (heartbeat)

These return null (no `Position` generated). The device remains online via `onMessageEvent`.

### Regex parsing (lines 45-118)
The single `PATTERN` handles multiple ITS firmware variants:
- **NRM** — standard position: status, event code, history flag, IMEI, date/time, lat/lon, speed, course, satellites, altitude, PDOP/HDOP, operator, ignition, charging, power, battery, emergency, cell towers (primary + 4 neighbor), digital I/O, odometer, ADC, G-sensor (X/Y/Z), tilt
- **EPB** — simplified: lat/lon only + altitude + speed (last two groups)
- **RLP** — odometer + checksum variant
- **C** — ADC1, ADC2 (no odometer)
- Format with `responseFormat`/`response` fields (some firmware versions)

The pattern uses nested `groupBegin()`/`or()`/`groupEnd()` to handle the format variants with one pass.

### Alarm decoding (`decodeAlarm`, lines 120-132)
Status codes → `Position.ALARM_*`:
| Code | Alarm |
|---|---|
| `WD`, `EA` | `ALARM_SOS` |
| `BL` | `ALARM_LOW_BATTERY` |
| `HB` | `ALARM_BRAKING` |
| `HA` | `ALARM_ACCELERATION` |
| `RT` | `ALARM_CORNERING` |
| `OS` | `ALARM_OVERSPEED` |
| `TA` | `ALARM_TAMPERING` |
| `BD` | `ALARM_POWER_CUT` |
| `BR` | `ALARM_POWER_RESTORED` |

### Status / ignition (lines 181-194)
- `"IN"` → `KEY_IGNITION = true`
- `"IF"` → `KEY_IGNITION = false`
- Other two-char codes → passed to `decodeAlarm()`

### Cell tower parsing (lines 223-238)
If cell string does not contain `"x"`:
```
cells[0] = signal strength (RSSI)
cells[1] = MCC (e.g., 404 for India)
cells[2] = MNC
cells[3] = LAC (hex)
cells[4] = CID (hex)
```
Parses primary tower + up to 4 neighbor towers (at offset 5+3i for i in 0..3). Creates `Network` with `CellTower.from(mcc, mnc, lac, cid, signal)`.

### Position fields populated (lines 167-282)
| Field | Source |
|---|---|
| `position.valid` | `valid` flag or `[1A]` regex |
| `position.time` | `DMY_HMS` date/time in device timezone |
| `position.latitude` / `longitude` | `DEG_HEM` format |
| `position.speed` | km/h → knots via `knotsFromKph` |
| `position.course` | degrees |
| `KEY_SATELLITES` | satellite count |
| `position.altitude` | meters |
| `KEY_PDOP` / `KEY_HDOP` | dilution of precision |
| `KEY_OPERATOR` | network operator name |
| `KEY_IGNITION` | boolean |
| `KEY_CHARGE` | charging state |
| `KEY_POWER` | external power (V) |
| `KEY_BATTERY` | battery level (V) |
| `"emergency"` | boolean flag |
| `network` | cell towers (primary + neighbors) |
| `KEY_INPUT` | 4-bit bitmask (inputs 0-3) |
| `KEY_OUTPUT` | 2-bit bitmask |
| `KEY_ODOMETER` | km → meters (× 1000) |
| `PREFIX_ADC + 1` | ADC channel 1 (V) |
| `KEY_G_SENSOR` | JSON array `[x, y, z]` |
| `"tiltY"` / `"tiltX"` | tilt angles |
| `KEY_EVENT` | event code (integer) |
| `KEY_ARCHIVE` | true if historical message |

## Gotchas / non-obvious

- **Input bit 3 remap** (lines 243-246): if the last input digit is `'2'`, it's replaced with `'0'` before binary parse. This is a quirk of some ITS firmware that uses `2` to indicate a floating/NC contact. **Critical for correct panic button detection.**
- **Odometer is km in protocol, meters in `Position`** — always multiply by 1000 (lines 252, 261, 271).
- **Multiple odometer parse paths** — depending on the variant matched, odometer may come from group 7 (NRM full), group 8 (RLP), or group 9 (C format). All populate `KEY_ODOMETER`.
- **DATE_FORMAT is UTC (`ZoneOffset.UTC`)** for LGN response only. Position timestamps use `getTimeZone(deviceId)` which may be a local timezone.
- **`parser.hasNext(n)` group counting** — the optional group checks (`hasNext(3)`, `hasNext(8)`, `hasNext(7)`, `hasNext(2)`, `hasNext(5)`) correspond to specific variant branches in the regex. Missing any means the frame matched a shorter variant.
- AIS-140 cell MCC for India: 404 (Airtel/BSNL/etc.) or 405. Test data in `ItsProtocolDecoderTest` uses MCC 404.

## Line index

- 38-39 — `DATE_FORMAT` (ddMMyyyyHHmmss UTC, for LGN ACK)
- 45-118 — `PATTERN` regex (multi-variant composite)
- 120-133 — `decodeAlarm` switch
- 136 — `decode()` entry
- 141-149 — handshake ACK responses
- 152-155 — regex parse attempt + early return
- 157-163 — status / event / history / type parsing
- 162-165 — `getDeviceSession(channel, remoteAddress, parser.next())` — IMEI extraction
- 167-194 — status/ignition/alarm + validity + time + coordinates
- 205-209 — speed / course / satellites
- 211-248 — altitude / PDOP / HDOP / operator / ignition / charge / power / battery / emergency / cell towers / I/O
- 251-258 — NRM full: odometer + ADC + G-sensor + tilt
- 260-262 — RLP: odometer + checksum
- 264-267 — C format: ADC1 + ADC2
- 269-275 — response format variant
- 277-280 — EPB: altitude + speed only
