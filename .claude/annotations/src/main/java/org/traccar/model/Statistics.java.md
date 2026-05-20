# Statistics.java

**Role:** Periodic usage snapshot. `@StorageName("tc_statistics")` → `tc_statistics`. Written by `TaskStatistics` (a scheduled job) with counts of active users, devices, API requests, messages, notifications, geocoder/geolocation calls, and a per-protocol message breakdown.
**Fits in:** Extends `ExtendedModel`. Read by `GET /api/statistics` (admin only). Not part of any real-time pipeline.
**Read next:** [[ExtendedModel.java]] (parent)

## Public API

### DB-mapped fields
- `captureTime Date` (line 27) — snapshot timestamp.
- `activeUsers int` (line 37) — users with sessions active in the period.
- `activeDevices int` (line 47) — devices that sent at least one message.
- `requests int` (line 57) — total HTTP API requests.
- `messagesReceived int` (line 67) — protocol messages received.
- `messagesStored int` (line 77) — positions persisted to DB.
- `mailSent int` (line 87) — email notifications sent.
- `smsSent int` (line 97) — SMS notifications sent.
- `geocoderRequests int` (line 107) — reverse geocoder calls.
- `geolocationRequests int` (line 117) — cell/WiFi geolocation calls.
- `protocols Map<String, Integer>` (line 127) — per-protocol message counts; JSON column.

## Line index

- 23 — `@StorageName("tc_statistics")`
- 24 — `class Statistics extends ExtendedModel`
- 27-35 — captureTime
- 127-134 — protocols (Map, JSON)
