# OverspeedState.java

**Role:** Mutable state object holding overspeed detection state for one device. Persisted to the `Device` row and loaded back on reconnect. Mirrors the pattern of [[MotionState.java]].
**Fits in:** Created by `OverspeedHandler` via `fromDevice`; mutated by [[OverspeedProcessor.java]]; written back via `toDevice`.
**Read next:** [[OverspeedProcessor.java]], [[MotionState.java]] (structural parallel)

## Public API

### Factory / persistence bridge (lines 25-37)
- `fromDevice(Device)` → `OverspeedState` — loads three fields from Device
- `toDevice(Device)` — writes three fields back

### State fields (lines 39-86)
All setters set `changed = true`:
- `overspeedState: boolean` — whether device is currently flagged as speeding
- `overspeedTime: Date | null` — time at which speeding began (null = not currently speeding)
- `overspeedGeofenceId: long` — geofence that triggered the speed limit (0 = device-level limit)
- `event: Event | null` — set by [[OverspeedProcessor.java]] when an event should fire

## Gotchas / non-obvious

- Same `changed` auto-set pattern as `MotionState` — set on any state-field setter; `event` setter does NOT set changed.
- `overspeedGeofenceId = 0` means the speed limit came from the device or server config, not a geofence polygon.
