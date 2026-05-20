# GeolocationHandler.java

**Role:** Resolves cell-tower/WiFi network info to GPS coordinates for positions that have no GPS fix (outdated or invalid). Fills in `latitude`, `longitude`, `accuracy`, marks position as `KEY_APPROXIMATE`. Async.
**Fits in:** Extends `BasePositionHandler`. Step 4 in the pipeline (after `TimeHandler`, before `HemisphereHandler`). Only active if `GeolocationProvider` is configured (Google Geolocation API or similar).
**Read next:** [[HemisphereHandler.java.md]] (step after), [[TimeHandler.java.md]] (step before)

## Public API

### `onPosition(Position, Callback)` (lines 50-86)
- Triggers only when: position is outdated OR (`processInvalidPositions=true` AND `!position.valid`), AND `position.network != null`, AND (WiFi required by config → WiFi access points present).
- If `reuse=true` and network info matches the last position's network: copies the last position's coordinates without an API call.
- Otherwise calls `geolocationProvider.getLocation(network, callback)` — async.
- On success: `updatePosition` fills in coords and marks `KEY_APPROXIMATE = true`.
- On failure: logs and continues (position remains without valid coords).

## Gotchas / non-obvious

- **`KEY_APPROXIMATE = true`** is set on geolocation-resolved positions. `FilterHandler.filterApproximate` can drop these if configured — ensure filter config is consistent with geolocation intent.
- **WiFi-only mode** (`requireWifi=true`): cells alone are insufficient; both cells and WiFi must be present.
- **`reuse=true`** avoids repeated API calls when the device is stationary. It compares network info by equality — same cell tower set produces the same coordinate instantly.
- **Not `@Inject`** — injected in `MainModule` only when a geolocation provider is configured.

## Line index

- 34 — `processInvalidPositions` flag
- 35 — `reuse` flag
- 36 — `requireWifi` flag
- 51-53 — trigger condition
- 54-63 — reuse shortcut
- 65-83 — async API call + callbacks
- 88-97 — `updatePosition` (marks approximate, zeros speed/course)
