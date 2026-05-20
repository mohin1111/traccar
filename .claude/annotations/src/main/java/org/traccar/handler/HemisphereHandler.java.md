# HemisphereHandler.java

**Role:** Corrects coordinates from devices that report unsigned latitude/longitude. Forces the correct hemisphere sign based on global config (`LOCATION_LATITUDE_HEMISPHERE`, `LOCATION_LONGITUDE_HEMISPHERE`).
**Fits in:** Extends `BasePositionHandler`. Step 5 in the pipeline (after `GeolocationHandler`, before `MapMatcherHandler`). A one-time global fix for low-quality device protocols.
**Read next:** [[MapMatcherHandler.java.md]] (step after), [[GeolocationHandler.java.md]] (step before)

## Public API

### `onPosition(Position, Callback)` (lines 49-57)
- If `latitudeFactor` is set (non-zero): replaces latitude with `|lat| * factor`.
- If `longitudeFactor` is set: same for longitude.
- `N` → factor +1, `S` → -1, `E` → +1, `W` → -1.

## Gotchas / non-obvious

- **Global config only** — there is no per-device override. If you have devices from both hemispheres, this cannot be used.
- **No-op if config is absent** — factors remain 0 (initialized by Java default), and the `if (factor != 0)` guards skip execution.
- **Designed for old OEM devices** that emit absolute lat/lon without direction indicators, common in early Indian VLT firmware.

## Line index

- 29-46 — constructor reads N/S/E/W config and sets factors
- 49 — `onPosition`
- 50-51 — latitude correction
- 52-54 — longitude correction
