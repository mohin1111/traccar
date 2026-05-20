# `model/` — Domain entity catalog

Plain Java domain objects for the Traccar server. No JPA/Hibernate — all persistence is via the custom `@StorageName`-annotated ORM in `storage/`.

## Class hierarchy

```
BaseModel                     (id only)
 ├── ExtendedModel             (+ attributes JSON bag)
 │    ├── GroupedModel         (+ groupId FK)
 │    │    ├── Device          → tc_devices
 │    │    │    └── LinkedDevice → tc_devices (proxy for tc_user_device)
 │    │    └── Group           → tc_groups
 │    ├── Message              (+ deviceId, type)
 │    │    ├── Position        → tc_positions (THE telemetry entity)
 │    │    └── BaseCommand     (+ textChannel)
 │    │         ├── Command    → tc_commands
 │    │         └── QueuedCommand → tc_commands_queue
 │    ├── User                 → tc_users
 │    │    └── ManagedUser     → tc_users (proxy for tc_user_user)
 │    ├── Event                → tc_events
 │    ├── Geofence             → tc_geofences
 │    ├── Driver               → tc_drivers
 │    ├── Calendar             → tc_calendars
 │    ├── Maintenance          → tc_maintenances
 │    ├── Notification         → tc_notifications
 │    ├── Server               → tc_servers (singleton row)
 │    ├── Statistics           → tc_statistics
 │    ├── Order                → tc_orders
 │    ├── Action               → tc_actions
 │    └── Report               → tc_reports
 └── Attribute                 → tc_attributes (JEXL rules; no attributes bag)
     RevokedToken              → tc_revoked_tokens (id only)

Non-BaseModel:
  Permission                  (link-table row; table name derived dynamically)
  DeviceAccumulators          (DTO, no @StorageName)
  LogRecord                   (transient WS push object)
  Network                     (embedded in Position, not its own table)
  CellTower                   (embedded in Network)
  WifiAccessPoint             (embedded in Network)
  Typed                       (record: single type string)
  Pair<K,V>                   (generic 2-tuple record)
  ObjectOperation             (enum: ADD / UPDATE / DELETE)

Interfaces:
  Disableable    — disabled + expirationTime + checkDisabled(); impl: User, Device
  Schedulable    — calendarId; impl: Device, Geofence, Notification, Report
  UserRestrictions — 5 boolean caps; impl: User, Server
```

## Entity → DB table map

| Class | DB table | Key columns |
|---|---|---|
| `Device` | `tc_devices` | id, name, uniqueid (IMEI), groupid, positionid, lastupdate, disabled, expirationtime, category, attributes |
| `LinkedDevice` | `tc_devices` | (same; proxy for permission routing only) |
| `Group` | `tc_groups` | id, name, groupid (parent), attributes |
| `Position` | `tc_positions` | id, deviceid, servertime, devicetime, fixtime, valid, latitude, longitude, altitude, speed (knots), course, address, accuracy, attributes, network |
| `User` | `tc_users` | id, name, email (unique), login, hashedpassword, salt, administrator, disabled, expirationtime, devicelimit, userlimit, readonly, devicereadonly, limitcommands, disablereports, fixedemail, totpkey, attributes |
| `ManagedUser` | `tc_users` | (same; proxy for permission routing only) |
| `Event` | `tc_events` | id, type, deviceid, positionid, geofenceid, maintenanceid, eventtime, attributes |
| `Geofence` | `tc_geofences` | id, name, description, area (WKT), calendarid, attributes |
| `Driver` | `tc_drivers` | id, name, uniqueid (RFID token), attributes |
| `Calendar` | `tc_calendars` | id, name, data (iCal bytes), attributes |
| `Maintenance` | `tc_maintenances` | id, name, type (accumulator key), start, period, attributes |
| `Notification` | `tc_notifications` | id, type, description, calendarid, always, notificators (CSV), commandid, attributes |
| `Attribute` | `tc_attributes` | id, description, attribute (output key), expression (JEXL), type (cast), priority |
| `Command` | `tc_commands` | id, description, type, textchannel, attributes |
| `QueuedCommand` | `tc_commands_queue` | id, deviceid, type, textchannel, attributes |
| `Server` | `tc_servers` | id, registration, readonly, map, bingkey, mapurl, latitude, longitude, zoom, limitcommands, disablereports, fixedemail, announcement, attributes |
| `Statistics` | `tc_statistics` | id, capturetime, activeusers, activedevices, requests, messagesreceived, messagesstored, mailsent, smssent, geocoderrequests, geolocationrequests, protocols (JSON), attributes |
| `Order` | `tc_orders` | id, uniqueid, description, fromaddress, toaddress, attributes |
| `Action` | `tc_actions` | id, actiontime, address, userid, actiontype, objecttype, objectid, attributes |
| `Report` | `tc_reports` | id, calendarid, type, description, attributes |
| `RevokedToken` | `tc_revoked_tokens` | id (the token id) |

## `@StorageName` and `@QueryIgnore` conventions

- `@StorageName("tc_xxx")` on a class → `DatabaseStorage` maps the class to that table. The ORM reads all JavaBean getters that are NOT annotated `@QueryIgnore` to build SELECT/INSERT/UPDATE column lists.
- `@QueryIgnore` on a getter → that field is excluded from all ORM operations. Use for: runtime-computed values, transient state, fields managed by separate queries (e.g., `Device.status`, `Device.positionId`), and for suppressing inherited getters (e.g., `Position.getType()` suppresses `Message.getType`).
- `@JsonIgnore` → excluded from API JSON serialization (independent of `@QueryIgnore`). Often used together for internal fields (e.g., `hashedPassword`, `salt`, motion state fields in `Device`).

## Permission-proxy pattern

Two "empty" subclasses (`ManagedUser`, `LinkedDevice`) exist purely to produce different link-table names in `Permission.getStorageName()`:

```
User       → ManagedUser  ─→  tc_user_user     (manager ↔ sub-user)
User       → LinkedDevice ─→  tc_user_device   (user ↔ device)
User       → Group        ─→  tc_user_group
User       → Geofence     ─→  tc_user_geofence
User       → Driver       ─→  tc_user_driver
User       → Attribute    ─→  tc_user_attribute
User       → Notification ─→  tc_user_notification
User       → Calendar     ─→  tc_user_calendar
Group      → Device       ─→  tc_group_device
Group      → Geofence     ─→  tc_group_geofence
Group      → Driver       ─→  tc_group_driver
Group      → Attribute    ─→  tc_group_attribute
Group      → Notification ─→  tc_group_notification
Group      → Calendar     ─→  tc_group_calendar
Device     → Geofence     ─→  tc_device_geofence
Device     → Driver       ─→  tc_device_driver
Device     → Attribute    ─→  tc_device_attribute
Device     → Notification ─→  tc_device_notification
Device     → Command      ─→  tc_device_command
Device     → Order        ─→  tc_device_order
```

`Permission.getStorageName(ownerClass, propertyClass)` derives the table name as:
`"tc_" + decapitalize(ownerSimpleName) + "_" + decapitalize(propertySimpleName)` — after stripping a `"Managed"` or `"Linked"` prefix from the property class name.

## Tenancy — there is NO `tenant_id` anywhere

This is the single most important architectural fact for Transport OS. Tenancy is **graph-based via permission link tables only**:

- An "org admin" in Traccar is a regular `User` with `userLimit != 0` and rows in `tc_user_user` linking to sub-users.
- Those sub-users' devices are linked via `tc_user_device`.
- There is NO `tenant_id` column in any table. Cross-tenant isolation is enforced by `PermissionsService` checking link tables on every API call.
- **For Transport OS multi-tenancy:** do not attempt to retrofit `tenant_id`. Wrap Traccar with a per-tenant gateway. Each tenant can be a top-level manager user, or a separate Traccar instance.

## Entity annotation index

| File | Annotation |
|---|---|
| `Action.java` | [[Action.java.md]] |
| `Attribute.java` | [[Attribute.java.md]] |
| `AttributeMap.java` | [[AttributeMap.java.md]] |
| `BaseCommand.java` | [[BaseCommand.java.md]] |
| `BaseModel.java` | [[BaseModel.java.md]] |
| `Calendar.java` | [[Calendar.java.md]] |
| `CellTower.java` | [[CellTower.java.md]] |
| `Command.java` | [[Command.java.md]] |
| `Device.java` | [[Device.java.md]] |
| `DeviceAccumulators.java` | [[DeviceAccumulators.java.md]] |
| `Disableable.java` | [[Disableable.java.md]] |
| `Driver.java` | [[Driver.java.md]] |
| `Event.java` | [[Event.java.md]] |
| `ExtendedModel.java` | [[ExtendedModel.java.md]] |
| `Geofence.java` | [[Geofence.java.md]] |
| `Group.java` | [[Group.java.md]] |
| `GroupedModel.java` | [[GroupedModel.java.md]] |
| `LinkedDevice.java` | [[LinkedDevice.java.md]] |
| `LogRecord.java` | [[LogRecord.java.md]] |
| `Maintenance.java` | [[Maintenance.java.md]] |
| `ManagedUser.java` | [[ManagedUser.java.md]] |
| `Message.java` | [[Message.java.md]] |
| `Network.java` | [[Network.java.md]] |
| `Notification.java` | [[Notification.java.md]] |
| `ObjectOperation.java` | [[ObjectOperation.java.md]] |
| `Order.java` | [[Order.java.md]] |
| `Pair.java` | [[Pair.java.md]] |
| `Permission.java` | [[Permission.java.md]] |
| `Position.java` | [[Position.java.md]] |
| `QueuedCommand.java` | [[QueuedCommand.java.md]] |
| `Report.java` | [[Report.java.md]] |
| `RevokedToken.java` | [[RevokedToken.java.md]] |
| `Schedulable.java` | [[Schedulable.java.md]] |
| `Server.java` | [[Server.java.md]] |
| `Statistics.java` | [[Statistics.java.md]] |
| `Typed.java` | [[Typed.java.md]] |
| `User.java` | [[User.java.md]] |
| `UserRestrictions.java` | [[UserRestrictions.java.md]] |
| `WifiAccessPoint.java` | [[WifiAccessPoint.java.md]] |
