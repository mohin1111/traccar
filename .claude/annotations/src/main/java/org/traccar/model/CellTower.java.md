# CellTower.java

**Role:** Value object for a single cell tower observation. Embedded in `Network`. Used by `GeolocationHandler` to call cell-based geolocation APIs (Google Geolocation, etc.) when device has no GPS fix.
**Fits in:** Created by protocol decoders; collected in `Network.cellTowers`. `@JsonInclude(NON_NULL)` — null fields omitted. Implements `equals`/`hashCode` for set membership in `Network`.
**Read next:** [[Network.java]] (container), [[WifiAccessPoint.java]] (sibling)

## Public API

### Static factories
- `from(mcc, mnc, lac, cid)` (line 27) — build with 4 key fields.
- `from(mcc, mnc, lac, cid, rssi)` (line 36) — adds signal strength.
- `fromLacCid(Config, lac, cid)` (line 42) — reads MCC/MNC from server config `Keys.GEOLOCATION_MCC/MNC`; used when device only reports LAC+CID.
- `fromCidLac(Config, cid, lac)` (line 46) — same, argument order swapped.

### Fields
- `radioType String` (line 50) — `"gsm"`, `"cdma"`, `"lte"`, `"wcdma"`.
- `cellId Long` (line 60) — Cell ID (CID).
- `locationAreaCode Integer` (line 70) — LAC.
- `mobileCountryCode Integer` (line 80) — MCC; India = 404/405.
- `mobileNetworkCode Integer` (line 90) — MNC.
- `signalStrength Integer` (line 100) — RSSI in dBm; **always stored as negative** (line 107: `signalStrength > 0 ? -signalStrength : signalStrength`).

### `setOperator(long operator)` (line 110)
Splits a combined operator long (e.g., `40420`) into MCC (first 3 digits) and MNC (remaining).

## Gotchas / non-obvious

- `signalStrength` is always negative (dBm). The setter normalizes positive inputs by negating them.
- `fromLacCid` relies on `Keys.GEOLOCATION_MCC/MNC` config; if not configured (defaults to 0), the geolocation request will fail silently.

## Line index

- 24 — `@JsonInclude(NON_NULL)`
- 27-33 — from(mcc, mnc, lac, cid)
- 36-39 — from(... rssi)
- 42-44 — fromLacCid (reads config for MCC/MNC)
- 100-108 — signalStrength (normalizes to negative)
- 110-114 — setOperator (splits combined MCC+MNC long)
