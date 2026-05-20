# PositionResource.java

**Role:** REST resource for `tc_positions` at `/api/positions`. Read-only access to historical positions plus export (KML/KMZ, CSV, GPX). Extends [[BaseResource.java]] directly (no CRUD hierarchy — positions are not user-created via API).
**Fits in:** Entity: `Position`. Returns positions with device-level permission checks.
**Read next:** [[BaseResource.java]], `PositionUtil.java` (helper for latest/range queries), export providers (`KmlExportProvider`, `CsvExportProvider`, `GpxExportProvider`)

## Public API (endpoints)

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/positions` | Fetch positions (latest, range, or by ID list) |
| `DELETE` | `/api/positions/{id}` | Delete single position |
| `DELETE` | `/api/positions?deviceId=&from=&to=` | Bulk delete range |
| `GET` | `/api/positions/{kml\|kmz}` | Export to KML/KMZ |
| `GET` | `/api/positions/csv` | Export to CSV |
| `GET` | `/api/positions/gpx` | Export to GPX |

## Key flows

### `GET /api/positions` — three modes (lines 69-99)
1. **`?id=` list**: fetch specific position IDs; permission checked by device ID on each position.
2. **`?deviceId=N&from=&to=`**: time-range query; checks `disableReports` restriction; optional `?geofenceId=` to filter positions inside a geofence.
3. **`?deviceId=N` (no time)**: latest position(s) for device.
4. **No params**: all latest positions for the authenticated user (via `PositionUtil.getLatestPositions`).

### Export endpoints (lines 133-197)
All check `checkPermission(Device.class, userId, deviceId)` first. Returns `StreamingOutput` with `Content-Disposition: attachment` header. KMZ wraps KML in a ZIP entry named `doc.kml`.

### Delete (lines 101-131)
Both single and range delete check `readonly` restriction. Range delete uses `Condition.Between("fixTime", from, to)`.

## Gotchas / non-obvious

- **`disableReports` restriction** (line 85) — time-range queries are subject to this restriction (same as report endpoints). Latest-position queries are not.
- **Geofence filter** (lines 87-92) — applied in Java, not SQL. `geofence.containsPosition(position)` uses JTS geometry. For large time ranges this may be slow.
- **`Condition.LatestPositions(deviceId)`** — custom condition that joins to `tc_devices.positionId` to get only the latest position row. Not a time-based query.

## Line index

- 57 — class extends `BaseResource`
- 69-99 — `GET /api/positions` (three modes)
- 101-116 — `DELETE /api/positions/{id}`
- 118-131 — `DELETE /api/positions` (range)
- 133-160 — KML/KMZ export
- 162-178 — CSV export
- 181-197 — GPX export
