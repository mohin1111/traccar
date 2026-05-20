# OverspeedEventHandler.java

**Role:** Detects sustained overspeeding using a state-machine (`OverspeedState`/`OverspeedProcessor`). Supports per-device, per-geofence, and road-network (`KEY_SPEED_LIMIT`) speed limits. Persists state to `tc_devices`.
**Fits in:** Extends `BaseEventHandler`. Most config-rich event handler. Works closely with `SpeedLimitHandler` (pipeline) and `GeofenceHandler` (pipeline).
**Read next:** [[BaseEventHandler.java.md]], [[handler/SpeedLimitHandler.java.md]], [[handler/GeofenceHandler.java.md]]

## Public API

### `onPosition(Position, Callback)` (lines 59-117)
Speed limit priority order (highest wins):
1. Geofence-specific limit (`Geofence.getDouble(EVENT_OVERSPEED_LIMIT.getKey())`).
2. Road network limit from `KEY_SPEED_LIMIT` (set by `SpeedLimitHandler`).
3. Device/group/server configured limit (`EVENT_OVERSPEED_LIMIT`).

If `preferLowest=true`: among multiple active geofence limits, the smallest is used (for overlapping geofences). Otherwise the largest.

Then delegates to `OverspeedProcessor.updateState(state, position, speedLimit, multiplier, minimalDuration, overspeedGeofenceId)`.

### State persistence (lines 105-112)
If `state.isChanged()`: saves `overspeedState`, `overspeedTime`, `overspeedGeofenceId` to `tc_devices`.

## Gotchas / non-obvious

- **`minimalDuration`** — overspeed must be sustained for N milliseconds before the event fires. This prevents false alarms from momentary GPS spikes.
- **`multiplier`** — speed limit * multiplier = actual detection threshold. E.g., `multiplier=1.1` triggers at 10% over the limit.
- **Geofence speed limit trumps road speed limit trumps device config** — the priority chain is explicit. If no limit is found at all (`speedLimit == 0`), the handler returns silently (line 98-100).
- **`overspeedGeofenceId` in the event** — the event records which geofence triggered the limit (if any), useful for "speeding inside school zone" type alerts.

## Line index

- 53-55 — constructor: minimalDuration, preferLowest, multiplier
- 60-63 — device + isLatest guard
- 70 — device-level speed limit lookup
- 72-75 — road network speed limit override
- 77-93 — geofence speed limit search
- 94-95 — geofence limit overrides road/device
- 98-100 — early exit if no limit
- 102-112 — state machine update + DB persist
- 114-116 — event emission
