# `api/resource/` — REST resource classes

24 resource classes, each mapping one Traccar domain entity to a set of REST endpoints. All registered with Jersey and served under `/api/`.

## File index

| File | Path | Entity | Base class | Annotation |
|---|---|---|---|---|
| `SessionResource.java` | `/api/session` | `User` (auth) | BaseResource | [SessionResource.java.md](SessionResource.java.md) |
| `DeviceResource.java` | `/api/devices` | `Device` | BaseObjectResource | [DeviceResource.java.md](DeviceResource.java.md) |
| `UserResource.java` | `/api/users` | `User` | BaseObjectResource | [UserResource.java.md](UserResource.java.md) |
| `GroupResource.java` | `/api/groups` | `Group` | SimpleObjectResource | [GroupResource.java.md](GroupResource.java.md) |
| `PositionResource.java` | `/api/positions` | `Position` | BaseResource | [PositionResource.java.md](PositionResource.java.md) |
| `GeofenceResource.java` | `/api/geofences` | `Geofence` | ExtendedObjectResource | [GeofenceResource.java.md](GeofenceResource.java.md) |
| `AttributeResource.java` | `/api/attributes/computed` | `Attribute` | ExtendedObjectResource | [AttributeResource.java.md](AttributeResource.java.md) |
| `NotificationResource.java` | `/api/notifications` | `Notification` | ExtendedObjectResource | [NotificationResource.java.md](NotificationResource.java.md) |
| `CommandResource.java` | `/api/commands` | `Command` | ExtendedObjectResource | [CommandResource.java.md](CommandResource.java.md) |
| `EventResource.java` | `/api/events` | `Event` | BaseResource | [EventResource.java.md](EventResource.java.md) |
| `DriverResource.java` | `/api/drivers` | `Driver` | ExtendedObjectResource | [DriverResource.java.md](DriverResource.java.md) |
| `CalendarResource.java` | `/api/calendars` | `Calendar` | SimpleObjectResource | [CalendarResource.java.md](CalendarResource.java.md) |
| `MaintenanceResource.java` | `/api/maintenance` | `Maintenance` | ExtendedObjectResource | [MaintenanceResource.java.md](MaintenanceResource.java.md) |
| `ReportResource.java` | `/api/reports` | `Report` | SimpleObjectResource | [ReportResource.java.md](ReportResource.java.md) |
| `ServerResource.java` | `/api/server` | `Server` | BaseResource | [ServerResource.java.md](ServerResource.java.md) |
| `PermissionsResource.java` | `/api/permissions` | `Permission` (link rows) | BaseResource | [PermissionsResource.java.md](PermissionsResource.java.md) |
| `StatisticsResource.java` | `/api/statistics` | `Statistics` | BaseResource | [StatisticsResource.java.md](StatisticsResource.java.md) |
| `AuditResource.java` | `/api/audit` | `Action` | BaseResource | [AuditResource.java.md](AuditResource.java.md) |
| `OrderResource.java` | `/api/orders` | `Order` | SimpleObjectResource | [OrderResource.java.md](OrderResource.java.md) |
| `ShareResource.java` | `/api/share` | synthetic `User` | BaseResource | [ShareResource.java.md](ShareResource.java.md) |
| `PasswordResource.java` | `/api/password` | `User` (password only) | BaseResource | [PasswordResource.java.md](PasswordResource.java.md) |
| `OidcResource.java` | `/api/oidc` | — (OIDC provider) | BaseResource | [OidcResource.java.md](OidcResource.java.md) |
| `HealthResource.java` | `/api/health` | — (health check) | BaseResource | [HealthResource.java.md](HealthResource.java.md) |
| `VideoStreamResource.java` | `/api/stream` | `Device` (video) | BaseResource | [VideoStreamResource.java.md](VideoStreamResource.java.md) |

## CRUD pattern

Resources extending `BaseObjectResource` (directly or via `Extended/Simple`) get these for free:
- `GET /{id}` — permission check + fetch; 404 if missing
- `POST /` — checkEdit → insert → add owner Permission row → cache/socket invalidation → audit
- `PUT /{id}` — checkPermission → checkEdit → update → cache invalidation → audit
- `DELETE /{id}` — checkPermission → checkEdit → remove → cache invalidation → audit

The `GET /` list endpoint is added by:
- `ExtendedObjectResource`: supports `?all`, `?userId`, `?groupId`, `?deviceId`, `?keyword`, `?limit`, `?offset`
- `SimpleObjectResource`: supports `?all`, `?userId`, `?keyword`, `?limit`, `?offset`
- `DeviceResource`, `UserResource`: fully custom `GET /` implementations

## Entity → join table mapping

| Permission scenario | Join table |
|---|---|
| User owns Device | `tc_user_device` |
| User owns Group | `tc_user_group` |
| Group owns Device | `tc_group_device` |
| User owns Geofence | `tc_user_geofence` |
| Group owns Geofence | `tc_group_geofence` |
| Device owns Geofence | `tc_device_geofence` |
| User manages User | `tc_user_user` (via `ManagedUser` proxy) |
| User owns Notification | `tc_user_notification` |
| Device owns Driver | `tc_device_driver` |
| (+ all others) | `tc_<decapOwner>_<decapProperty>` |

## Multi-file flows

### New device registration
1. Client `POST /api/devices` → `DeviceResource` → `BaseObjectResource.add`.
2. Device row inserted; owner Permission row `(User, userId, Device, deviceId)` added.
3. Cache + ConnectionManager notified → WebSocket clients see new device.

### Sharing a device
1. Client `POST /api/share/device` with `deviceId` + `expiration`.
2. `ShareResource.share` creates a temporary read-only user + `tc_user_device` row.
3. Returns Bearer token; recipient uses it as `?token=` on `GET /api/socket`.

### Report generation
1. Client `GET /api/reports/trips?deviceId=5&from=&to=` → `ReportResource`.
2. `checkRestriction(disableReports)` → `TripsReportProvider.getObjects(userId, [5], [], from, to)`.
3. Provider checks device permissions internally; returns `Collection<TripReportItem>`.

## Conventions

- All resources annotated `@Produces(APPLICATION_JSON)` + `@Consumes(APPLICATION_JSON)` unless they also accept form or binary types.
- Date params use `DateParameterConverterProvider` for ISO 8601 parsing.
- `@PermitAll` on `POST /api/session`, `POST /api/users`, `GET /api/server`, `GET /api/health`, and OIDC/password reset endpoints.
- Stream-returning list methods rely on `StreamWriter` for memory-efficient JSON array output.
- Every write (POST/PUT/DELETE) logs to `LogAction`/`tc_actions` for audit.
