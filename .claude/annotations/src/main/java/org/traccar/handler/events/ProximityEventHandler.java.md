# ProximityEventHandler.java

**Role:** Detects proximity events between devices — enter/exit proximity zones (`TYPE_PROXIMITY_ENTER`, `TYPE_PROXIMITY_EXIT`) and unaccompanied motion (`TYPE_UNACCOMPANIED_MOTION`) when a device starts moving without any companion device nearby.
**Fits in:** Extends `BaseEventHandler`. Newest event handler (2026). Uses `CacheManager.getDeviceObjects(deviceId, Device.class)` to get linked devices.
**Read next:** [[BaseEventHandler.java.md]]

## Public API

### `onPosition(Position, Callback)` (lines 40-100)
- Guards: latest position, linked devices exist, meaningful configuration.
- For each linked device: computes distance between current and linked device's last position.
- Enter event: previous distance > `enterDistance` AND current <= `enterDistance`.
- Exit event: previous distance <= `exitDistance` AND current > `exitDistance`.
- Unaccompanied motion: device transitions from not-moving to moving, and no linked device is within `unaccompaniedDistance`.

### `boundedDistance` (lines 102-109)
Bounding-box prefilter before Haversine: returns `POSITIVE_INFINITY` if lat or lon delta exceeds the max distance delta. Avoids calling Haversine for devices far apart.

## Gotchas / non-obvious

- **Three independent distance thresholds** — `enterDistance`, `exitDistance`, `unaccompaniedDistance` — all per-device via `AttributeUtil.lookup`. Any combination can be zero (disabled).
- **`unaccompaniedMotion`** fires only on the start-of-motion transition (`!lastMotion && currentMotion`). It does not re-fire on subsequent positions.
- **Linked devices** = devices linked via `tc_device_device` (device-to-device permission rows), not just devices in the same group.
- **`DistanceCalculator.getLatitudeDelta/getLongitudeDelta`** converts meters to degree deltas for the bounding-box prefilter.

## Line index

- 52-54 — load three distance thresholds
- 61-63 — unaccompanied check condition (motion start)
- 64 — early exit if nothing to check
- 68-70 — bounding-box deltas for prefilter
- 73-94 — linked device loop: enter/exit events
- 92-94 — track if any device is nearby (for unaccompanied check)
- 97-99 — unaccompanied motion event
- 102-109 — `boundedDistance` prefilter
