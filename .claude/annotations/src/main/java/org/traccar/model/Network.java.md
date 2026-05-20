# Network.java

**Role:** Container for cell tower and WiFi access point data embedded in a `Position`. Not a persisted entity on its own — serialized to JSON inside the `Position` record. Passed to `GeolocationHandler` for cell/WiFi → GPS coordinate lookup when the device has no GPS fix.
**Fits in:** `Position.network` field. Created by protocol decoders; consumed by `GeolocationHandler`. `@JsonInclude(NON_NULL)` ensures null fields are omitted from JSON.
**Read next:** [[CellTower.java]], [[WifiAccessPoint.java]], [[Position.java]]

## Public API

- `homeMobileCountryCode Integer` (line 41) — home network MCC.
- `homeMobileNetworkCode Integer` (line 51) — home network MNC.
- `radioType String` (line 61) — default `"gsm"`; also `"cdma"`, `"lte"`, `"wcdma"`.
- `carrier String` (line 71) — network operator name string.
- `considerIp Boolean` (line 81) — hint to geolocation provider to include IP-based lookup; default `false`.
- `cellTowers Set<CellTower>` (line 91) — set of visible cell towers.
- `wifiAccessPoints Set<WifiAccessPoint>` (line 108) — set of visible WiFi APs.

### Constructors
- `Network(CellTower... cellTowers)` (line 29) — varargs for quick inline construction.
- `Network(WifiAccessPoint... accessPoints)` (line 35) — varargs.

### Mutation methods
- `addCellTower(CellTower)` (line 101) — lazy-init `cellTowers` set and add.
- `addWifiAccessPoint(WifiAccessPoint)` (line 118) — lazy-init `wifiAccessPoints` set and add.

## Line index

- 24 — `@JsonInclude(NON_NULL)`
- 25 — `class Network` (no @StorageName — not a DB entity)
- 29-38 — constructors (varargs CellTower, WifiAccessPoint)
- 41-88 — scalar fields
- 91-123 — cellTowers + wifiAccessPoints (lazy-init sets)
- 126-140 — equals/hashCode
