# Event.java

**Role:** Represents a detected fleet event (geofence crossing, alarm, ignition change, driver change, etc.). `@StorageName("tc_events")` → `tc_events`. Created by event handler classes in `handler/` and pushed to WebSocket clients in real time.
**Fits in:** Extends `Message` (gets `deviceId`, `type`, `attributes`). Created by `GeofenceHandler`, `OverspeedEventHandler`, `IgnitionEventHandler`, etc. Delivered via `NotificationManager` → notificator channels.
**Read next:** [[Message.java]] (parent), [[Notification.java]] (notification rule that fires on event type), [[Position.java]] (event references a positionId)

## Public API

### DB-mapped fields (`tc_events` columns)
- `type String` (from `Message`) — event type string; one of the `TYPE_*` constants below.
- `deviceId long` (from `Message`) — FK to `tc_devices.id`.
- `attributes AttributeMap` (from `ExtendedModel`) — event-specific payload (e.g., alarm type in `attributes.alarm`).
- `eventTime Date` (line 73) — when the event occurred; set from `position.getDeviceTime()` in the Position constructor or `new Date()` in the device-only constructor.
- `positionId long` (line 83) — FK to `tc_positions.id`; 0 when event is not tied to a position.
- `geofenceId long` (line 93) — FK to `tc_geofences.id`; 0 when not a geofence event.
- `maintenanceId long` (line 103) — FK to `tc_maintenances.id`; 0 when not a maintenance event.

### Event type constants (lines 41-71)
- Device lifecycle: `TYPE_DEVICE_ONLINE`, `TYPE_DEVICE_OFFLINE`, `TYPE_DEVICE_UNKNOWN`, `TYPE_DEVICE_INACTIVE`.
- Motion: `TYPE_DEVICE_MOVING`, `TYPE_DEVICE_STOPPED`.
- Speed: `TYPE_DEVICE_OVERSPEED`.
- Fuel: `TYPE_DEVICE_FUEL_DROP`, `TYPE_DEVICE_FUEL_INCREASE`.
- Geofence: `TYPE_GEOFENCE_ENTER`, `TYPE_GEOFENCE_EXIT`.
- Proximity: `TYPE_PROXIMITY_ENTER`, `TYPE_PROXIMITY_EXIT`, `TYPE_UNACCOMPANIED_MOTION`.
- Alarm: `TYPE_ALARM` (generic; actual alarm type in `attributes.alarm`).
- Ignition: `TYPE_IGNITION_ON`, `TYPE_IGNITION_OFF`.
- Other: `TYPE_MAINTENANCE`, `TYPE_DRIVER_CHANGED`, `TYPE_MEDIA`, `TYPE_COMMAND_RESULT`, `TYPE_QUEUED_COMMAND_SENT`.
- `ALL_EVENTS = "allEvents"` — wildcard used by `Notification` subscriptions.

### Constructors
- `Event(String type, Position position)` (line 25) — links to position; sets `deviceId` from position.
- `Event(String type, long deviceId)` (line 32) — device-only event; `eventTime = new Date()`.
- `Event()` (line 38) — no-arg for Jackson/ORM deserialization.

## Key flows

### Event creation in handlers
```java
// GeofenceHandler example
Event event = new Event(Event.TYPE_GEOFENCE_ENTER, position);
event.setGeofenceId(geofenceId);
// → stored in tc_events, pushed via ConnectionManager.updateEvent()
```

## Gotchas / non-obvious

- `geofenceId` and `maintenanceId` default to `0` (lines 93, 103) — check `> 0` before treating as a valid FK.
- Alarm events: `type = TYPE_ALARM`, and the alarm kind is in `attributes.get("alarm")` (a `Position.ALARM_*` constant). Notification rules match on `type` first, then optionally on the alarm attribute.

## Line index

- 22 — `@StorageName("tc_events")`
- 23 — `class Event extends Message`
- 25-37 — constructors (position-linked, device-only)
- 40-71 — TYPE_* + ALL_EVENTS constants
- 73-81 — eventTime
- 83-91 — positionId
- 93-101 — geofenceId
- 103-111 — maintenanceId
