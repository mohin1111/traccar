# NotificationManager.java

**Role:** Persists events to the DB, forwards them to external systems, and fans out notification delivery to all matching users via all their configured notificator channels.
**Fits in:** Called from event handler classes in `handler/event/` at the end of position processing. Central hub connecting events → storage → forwarder → notificators.
**Read next:** [[NotificatorManager.java]] (channel dispatch), [[CacheManager.java]] (notification + user resolution), [[EventForwarder.java]] (external forwarding)

## Public API

- `updateEvents(Map<Event, Position> events)` (line 177) — public entry point. Called per-position with the set of events that fired. Iterates, persists each, then delivers notifications.

## Key flows

### `updateEvent(Event, Position)` (lines 84-155) — the core loop
1. **Persist** event: `storage.addObject(event, ...)` → sets `event.id` (line 86).
2. **Forward** to external system: `forwardEvent()` → `eventForwarder.forward(eventData, callback)` (line 92).
3. **Age gate**: skip notification delivery if `event.eventTime` is older than `timeThreshold` (line 93).
4. **Resolve matching notifications** from `cacheManager.getDeviceNotifications(deviceId)`:
   - Filter by `notification.type == event.type`
   - For alarm events: also filter by specific alarm types in `notification.attributes.alarms`
   - Filter by specific geofence IDs if `notification.attributes.geofenceIds` set
   - Filter by calendar schedule if `notification.calendarId != 0`
5. **On-demand geocode**: if position has no address and `geocodeOnRequest=true`, geocode synchronously before delivering.
6. **Fan-out per user per notificator**: for each notification → for each linked user → for each notificator type → call `notificatorManager.getNotificator(type).send(...)`.
7. **Blocked users**: skip if `userId` in `blockedUsers` set (configured via `Keys.NOTIFICATION_BLOCK_USERS`).

### `forwardEvent` (lines 157-175)
Builds an `EventData` envelope (event + position + device + optional geofence/maintenance) and calls `eventForwarder.forward()` asynchronously.

## Gotchas / non-obvious

- **`updateEvents` wraps each event in `cacheManager.addDevice / removeDevice`** (lines 182-190) — this ref-counts the device in the cache to prevent eviction while the event is being processed. Always released in `finally`.
- **`eventForwarder` is `@Nullable`** — only present if forwarding is configured. Guards checked with `if (eventForwarder != null)`.
- **Geocoder is also `@Nullable`** and only called if `geocodeOnRequest=true` AND position has no address. Synchronous call on the processing thread — can add latency.
- **`timeThreshold`** defaults to a large value; set `Keys.NOTIFICATOR_TIME_THRESHOLD` to skip old events during catch-up replays.
- Notification delivery failures (`MessageException`) are only **logged as WARN**, not re-thrown — one broken notificator does not prevent others from running.

## Line index

- 51-82 — `@Inject` constructor: wires all deps + reads config keys
- 84-155 — `updateEvent` (single event: persist → forward → age check → match notifications → deliver)
- 98-124 — notification filtering chain (type, alarm, geofence, calendar)
- 134-154 — geocode on request + per-user per-notificator fan-out
- 157-175 — `forwardEvent` helper
- 177-191 — `updateEvents` public entry point
