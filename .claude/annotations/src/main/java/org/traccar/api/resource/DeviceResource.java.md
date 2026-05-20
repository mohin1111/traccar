# DeviceResource.java

**Role:** REST resource for `tc_devices` at `/api/devices`. Extends [[BaseObjectResource.java]] for CRUD. Adds custom list endpoint with `?uniqueId`, `?id`, `?deviceId` (linked devices), and `?keyword` filtering. Also exposes accumulator reset and device image upload.
**Fits in:** Entity: `Device`. Most-queried resource in practice (called on every page load and WS reconnect).
**Read next:** [[BaseObjectResource.java]] (CRUD), [[ExtendedObjectResource.java]] (contrast — not used here, custom `GET /`), `Device.java` model, `MediaManager.java` (image storage)

## Public API (endpoints)

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/devices` | List devices; multiple filter modes |
| `GET` | `/api/devices/{id}` | Single device (from base class) |
| `POST` | `/api/devices` | Create device (from base class) |
| `PUT` | `/api/devices/{id}` | Update device (from base class) |
| `DELETE` | `/api/devices/{id}` | Delete device (from base class) |
| `PUT` | `/api/devices/{id}/accumulators` | Reset odometer/hours |
| `POST` | `/api/devices/{id}/image` | Upload device avatar image |

## Key flows

### `GET /api/devices` — two code paths (lines 84-146)
**Path 1** — `?uniqueId=` or `?id=` list (lines 95-113): each ID/uniqueId fetches individually; permission checked per item via `Condition.Permission`. Returns list.
**Path 2** — full filter mode (lines 115-145): conditions built from `?all`, `?userId`, `?deviceId` (linked-from-device via `LinkedDevice` proxy), `?keyword` (searches name/uniqueId/phone/model/contact). Sorted by `name` with pagination.

### `PUT /api/devices/{id}/accumulators` (lines 149-186)
1. Permission check on device.
2. Load latest position.
3. Mutate `totalDistance` and/or `hours` attributes on the position.
4. Insert as a NEW position row (preserves history).
5. Update `device.positionId` to point to the new position.
6. Propagate through CacheManager + ConnectionManager (live push to WebSocket clients).

### `POST /api/devices/{id}/image` (lines 199-232)
- Limit: 500 KB (`IMAGE_SIZE_LIMIT`).
- Accepted types: JPEG, PNG, GIF, WebP.
- Stored via `MediaManager.createFileStream(uniqueId, "device", extension)`.
- Returns just the filename `"device.jpg"` (no full URL).

## Gotchas / non-obvious

- **`?keyword`** searches 5 columns (name, uniqueId, phone, model, contact) — useful for admin device search.
- **`?deviceId`** (linked-from-device, line 131) — uses `LinkedDevice.class` which is a proxy for `tc_device_device` link table (device-to-device relationships). Not the same as `?id`.
- **Accumulators update creates a NEW position** (line 163) — this is intentional so the history tracks when the reset happened.
- **Image 500 KB limit** enforced by streaming read, not buffered. Throws `IllegalArgumentException` mid-stream if exceeded; caller gets 400.

## Line index

- 59 — class + extends `BaseObjectResource<Device>`
- 84-146 — custom `GET /api/devices`
- 95-113 — `?uniqueId`/`?id` path
- 115-145 — filter/pagination path
- 149-186 — accumulator reset
- 199-232 — image upload
