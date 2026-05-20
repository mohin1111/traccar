# DriverHandler.java

**Role:** Injects `KEY_DRIVER_UNIQUE_ID` into a position from a linked Driver entity when the device does not report one via RFID/iButton. Falls back to the first driver linked to the device in the permission graph.
**Fits in:** Extends `BasePositionHandler`. Step 14 in the pipeline (after `ComputedAttributesHandler.Late`, before `CopyAttributesHandler`). Controlled by config key `PROCESSING_USE_LINKED_DRIVER`.
**Read next:** [[CopyAttributesHandler.java.md]], [[events/DriverEventHandler.java.md]] (fires driver-changed events)

## Public API

### `onPosition(Position, Callback)` (lines 37-44)
- Only acts if `useLinkedDriver = true` AND the position does not already carry `KEY_DRIVER_UNIQUE_ID`.
- Loads drivers linked to the device via `CacheManager.getDeviceObjects(deviceId, Driver.class)`.
- Takes the first driver (arbitrary iteration order from the cache set).
- Sets `KEY_DRIVER_UNIQUE_ID` on the position.

## Gotchas / non-obvious

- **Device-reported ID wins** — guard `!position.hasAttribute(KEY_DRIVER_UNIQUE_ID)` means hardware RFID/iButton always takes priority.
- **First driver is not deterministic** — `getDeviceObjects` returns a `Set`; if multiple drivers are linked, the one used depends on iteration order. Production usage typically links one driver at a time.
- **`useLinkedDriver = false` by default** — feature is opt-in via config.

## Line index

- 33 — `useLinkedDriver` from `PROCESSING_USE_LINKED_DRIVER`
- 38-43 — guard + lookup + set driver ID
