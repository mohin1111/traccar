# `reports/` — Excel and GPX/KML/CSV report generators

24 files producing trip, stop, event, geofence, route, summary, and device reports as Excel workbooks, GPX tracks, KML, or CSV files. Reports can run synchronously (API download) or asynchronously (scheduled, emailed).

## File index

| File | One-liner |
|---|---|
| `CombinedReportProvider.java` | Merged trips+stops report |
| `RouteReportProvider.java` | Raw position list (the route) |
| `TripsReportProvider.java` | Trip segments with start/end address, distance, duration, max speed |
| `StopsReportProvider.java` | Stop periods with location, duration |
| `EventsReportProvider.java` | Event log (geofence enter/exit, alarms, ignition, etc.) |
| `SummaryReportProvider.java` | Per-device totals: distance, engine hours, max speed, avg speed |
| `GeofenceReportProvider.java` | Geofence crossing events |
| `DevicesReportProvider.java` | Device list with last position |
| `GpxExportProvider.java` | GPX track file export |
| `KmlExportProvider.java` | KML track file export |
| `CsvExportProvider.java` | Raw positions as CSV |
| `common/` | `ReportUtils.java` — shared trip/stop detection logic + Excel workbook helpers |
| `model/` | `TripReportItem.java`, `StopReportItem.java`, `DeviceReportSection.java` — report row models |

## Dependency graph

```
ReportResource (api/resource/) → *ReportProvider
TaskReports (schedule/) → *ReportProvider → MailManager (email delivery)

*ReportProvider → ReportUtils + Storage + PositionUtil (helper/model/)
ReportUtils → Apache POI (Excel) + DistanceCalculator + UnitsConverter
```

## Report generation flow

1. `ReportResource.getX()` called via `GET /api/reports/trips?deviceId=1&from=...&to=...`.
2. If request `Accept: application/vnd.ms-excel` → streams Excel via `OutputStream`.
3. If `Accept: application/json` → returns list of model objects.
4. Provider queries `tc_positions` for the time range, runs trip/stop detection via `ReportUtils`, builds rows.
5. Excel output uses Apache POI (`XSSFWorkbook`) with `WorkbookUtil.createSafeSheetName`.

## Trip/stop detection (`ReportUtils`)

Core algorithm:
1. Walk positions chronologically.
2. Track `motion` state (from `position.getBoolean(Position.KEY_MOTION)` or speed threshold).
3. Trip = consecutive positions where `motion=true`.
4. Stop = consecutive positions where `motion=false` and duration exceeds `Keys.REPORT_MINIMUM_IDLING_PERIOD`.
5. Trip start/end address geocoded from first/last position.

Config keys:
- `report.trip.minimalTripDistance` (meters, default 500)
- `report.trip.minimalTripDuration` (seconds, default 300)
- `report.trip.minimalNoDataDuration` — marks trip gap as separate trips
- `report.idle.minimalIdlingPeriod` (seconds, default 300)

## Scheduled reports (`TaskReports`)

`TaskReports` runs daily (configurable). For each `Report` entity in `tc_reports` with a `calendarId`, it:
1. Computes the time range for the past period.
2. Generates the Excel file.
3. Emails it to all users associated with the report.

## Transport OS relevance

**High relevance** — Trip reports are core to freight billing (distance per trip), driver performance (overspeed incidents), and regulatory compliance (daily driving hours). `TripsReportProvider` + `StopsReportProvider` are the starting point for Transport OS trip analytics. The report model objects (`TripReportItem`) can be directly serialized to the Transport OS API as JSON.

The `CsvExportProvider` / `GpxExportProvider` are useful for exporting raw position data to external analytics or AIS-140 archival systems.

## How to add a new report type

1. Create `MyReportProvider.java` following the pattern of `TripsReportProvider`.
2. Add corresponding model class in `model/` if needed.
3. Add a JAX-RS method in `api/resource/ReportResource.java`.
4. If schedulable, add support in `TaskReports`.
5. No registration — `ReportResource` injects providers by class name.
