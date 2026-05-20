# StatisticsManager.java

**Role:** In-memory per-day counters for server telemetry. At day rollover, persists a `Statistics` row to the DB and optionally POSTs aggregated stats to the Traccar upstream URL.
**Fits in:** Injected as singleton into REST filters, geocoder, geolocation provider, and mail/SMS managers which call `register*()` methods. Also used by `api/security/SecurityRequestFilter` to count requests.
**Read next:** [[Config.java]] (`Keys.SERVER_STATISTICS`), `model/Statistics.java`

## Public API

All methods are `synchronized`. Each one calls `checkSplit()` first to handle day rollover.

- `registerRequest(long userId)` — increments REST request count; adds userId to active set.
- `registerMessageReceived()` — increments raw messages received counter.
- `registerMessageStored(long deviceId, String protocol)` — increments stored messages; tracks per-device protocol and per-device message count.
- `registerMail()` — mail notification sent.
- `registerSms()` — SMS sent.
- `registerGeocoderRequest()` — geocoder API call made.
- `registerGeolocationRequest()` — cell/WiFi geolocation call made.
- `messageStoredCount()` / `messageStoredCount(long deviceId)` — current stored message count for reporting.

## Key flows

### Day rollover — `checkSplit()` (lines 79-155)
Compares `Calendar.DAY_OF_MONTH` with last recorded day (atomic int). On change:
1. Snapshots all counters into a `Statistics` object.
2. Resets all counters to 0.
3. Persists `statistics` row to DB.
4. If `Keys.SERVER_STATISTICS` URL is set, POSTs a form to that URL with version, user/device counts, message counts, protocol breakdown (as JSON), and attributes.

### Protocol breakdown
`deviceProtocols` is a `Map<Long deviceId, String protocol>`. On snapshot, it is grouped into `Map<String protocol, Integer count>` and stored in `Statistics.protocols`.

## Gotchas / non-obvious

- **Service account user excluded** from active user count (line 160) — `ServiceAccountUser.ID` is a sentinel.
- **All methods `synchronized`** — contention expected; this is by design for simplicity, not performance.
- **The upstream POST** (lines 122-153) is `async()` — non-blocking fire-and-forget.
- **`checkSplit()` is called on every register invocation** — day rollover is detected lazily, not on a schedule. If no traffic occurs overnight, stats won't be flushed until next request.
- **`Keys.SERVER_STATISTICS` URL leaks usage data** to traccar.org by default in official builds — disable or override for privacy in production forks.

## Line index

- 57-77 — fields: counters, sets, maps
- 72-77 — `@Inject` constructor
- 79-155 — `checkSplit` (rollover detection + persist + optional upstream POST)
- 157-204 — `register*` public methods (all call `checkSplit` then increment)
