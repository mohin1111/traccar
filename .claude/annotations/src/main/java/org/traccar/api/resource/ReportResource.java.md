# ReportResource.java

**Role:** REST resource at `/api/reports` for analytical reports: route, events, geofences, summary, trips, stops, combined, and devices. Each report has JSON and Excel variants. Extends SimpleObjectResource for saved report config CRUD.
**Fits in:** Entity: `Report` (saved report config). Data from provider classes. `disableReports` restriction enforced on all data endpoints.
**Read next:** [[SimpleObjectResource.java]], `TripsReportProvider.java`, `StopsReportProvider.java`, `ReportMailer.java`

## Public API

Reports at `/api/reports/{type}`: combined, route, events, geofences, summary, trips, stops. Excel at `/api/reports/{type}/xlsx`, email at `/api/reports/{type}/mail`. Saved config CRUD via base class at `/api/reports/{id}`.

## Key flows

`executeReport(userId, mail, executor)` (lines 105-120): if mail=true, runs async via `reportMailer.sendAsync`; else wraps in `StreamingOutput` returning xlsx with `Content-Disposition: attachment`.

Regex `@Path("route/{type:xlsx|mail}")` routes both xlsx and mail suffix to one method, dispatching on `type.equals("mail")`.

All data endpoints: checkRestriction(disableReports) + audit log + provider call.

## Gotchas / non-obvious

- `disableReports` enforced on data endpoints but not on saved report CRUD.
- `?daily=true` on summary splits results by day.
- Events: `?type=` and `?alarm=` list params narrow results.

## Line index

- 64 — class + extends `SimpleObjectResource<Report>`
- 101 — constructor sortField=`"description"`
- 105-120 — `executeReport` helper
- 122-365 — all report endpoints
