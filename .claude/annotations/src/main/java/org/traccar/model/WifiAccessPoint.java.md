# WifiAccessPoint.java

**Role:** Value object for a WiFi access point observation. Embedded in `Network`. Used alongside `CellTower` for geolocation when no GPS fix is available.
**Fits in:** Collected in `Network.wifiAccessPoints`. Created by protocol decoders. `@JsonInclude(NON_NULL)`.
**Read next:** [[Network.java]] (container), [[CellTower.java]] (sibling)

## Public API

### Static factories
- `from(macAddress, signalStrength)` (line 25) — basic factory.
- `from(macAddress, signalStrength, channel)` (line 32) — with WiFi channel.

### Fields
- `macAddress String` (line 38) — BSSID in MAC format (e.g., `"00:11:22:33:44:55"`).
- `signalStrength Integer` (line 48) — RSSI dBm; **always stored as negative** (line 55).
- `channel Integer` (line 58) — WiFi channel number; optional.

## Gotchas / non-obvious

- `signalStrength` is normalized to negative, same convention as `CellTower`.

## Line index

- 22 — `@JsonInclude(NON_NULL)`
- 25-30 — from(macAddress, signalStrength)
- 32-36 — from(..., channel)
- 48-56 — signalStrength (normalizes to negative)
