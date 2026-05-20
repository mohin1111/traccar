# PostProcessHandler.java

**Role:** Updates the device's `positionId` pointer to the newly persisted position, refreshes the in-memory `CacheManager`, and notifies WebSocket clients via `ConnectionManager`. The final step of the full pipeline.
**Fits in:** Extends `BasePositionHandler`. Runs last in the chain (after `DatabaseHandler`). Depends on `position.getId()` being set by `DatabaseHandler`.
**Read next:** [[DatabaseHandler.java.md]] (must run before to set position ID), [[BasePositionHandler.java.md]]

## Public API

### `onPosition(Position, Callback)` (lines 48-66)
- Checks `PositionUtil.isLatest(cacheManager, position)` — only the most recent position for a device should update the device pointer. Out-of-order positions (replay) are skipped.
- Updates `tc_devices.positionId` = `position.getId()` via `storage.updateObject`.
- Calls `cacheManager.updatePosition(position)` — updates the in-memory last-position cache.
- Calls `connectionManager.updatePosition(true, position)` — pushes the position to connected WebSocket clients.

## Gotchas / non-obvious

- **`PositionUtil.isLatest`** check prevents older replayed positions from overwriting the device's current position pointer. Only forward-time positions advance the device state.
- **`StorageException` is soft** — logs a warning but does not re-throw. The callback always fires `processed(false)`.
- **WebSocket push happens here** — this is where live tracking clients receive position updates in near-real-time.
- **The `true` flag in `updatePosition(true, position)`** indicates this is a live update (not a historical replay), triggering immediate WebSocket dispatch.

## Line index

- 48 — `onPosition`
- 50 — `PositionUtil.isLatest` guard
- 51-56 — update `tc_devices.positionId`
- 58 — `cacheManager.updatePosition`
- 59 — `connectionManager.updatePosition` (WebSocket push)
