# `api/` — REST + WebSocket layer

Jersey JAX-RS REST API and Jetty WebSocket server. All client interaction (browser, mobile, integrations) goes through this layer. Serves at `/api/**`.

## File index

| File | One-liner | Annotation |
|---|---|---|
| `AsyncSocket.java` | Per-connection WebSocket; pushes device/position/event updates | [AsyncSocket.java.md](AsyncSocket.java.md) |
| `AsyncSocketServlet.java` | WS upgrade handler; authenticates + creates AsyncSocket instances | [AsyncSocketServlet.java.md](AsyncSocketServlet.java.md) |
| `BaseResource.java` | Root base for all resources; provides Storage + PermissionsService + getUserId() | [BaseResource.java.md](BaseResource.java.md) |
| `BaseObjectResource.java` | Generic CRUD (GET/{id}, POST, PUT/{id}, DELETE/{id}) with permission checks + cache invalidation | [BaseObjectResource.java.md](BaseObjectResource.java.md) |
| `ExtendedObjectResource.java` | Adds list GET with all/userId/groupId/deviceId/keyword filtering | [ExtendedObjectResource.java.md](ExtendedObjectResource.java.md) |
| `SimpleObjectResource.java` | Adds list GET with user-only filtering | [SimpleObjectResource.java.md](SimpleObjectResource.java.md) |
| `CorsResponseFilter.java` | CORS headers on every response | [CorsResponseFilter.java.md](CorsResponseFilter.java.md) |
| `DateParameterConverterProvider.java` | ISO 8601 Date query param converter | [DateParameterConverterProvider.java.md](DateParameterConverterProvider.java.md) |
| `MediaFilter.java` | Servlet filter guarding /api/media/ (session auth + device permission) | [MediaFilter.java.md](MediaFilter.java.md) |
| `ResourceErrorHandler.java` | Global exception mapper; renders stack trace as response body | [ResourceErrorHandler.java.md](ResourceErrorHandler.java.md) |
| `StreamWriter.java` | MessageBodyWriter for Java Stream<T> → JSON array (memory-efficient) | [StreamWriter.java.md](StreamWriter.java.md) |

Sub-packages: `resource/` (24 resource classes), `security/` (auth + permissions), `signature/` (crypto + tokens).

## Resource base-class hierarchy

```
BaseResource
├── SessionResource          (login/logout/OIDC)
├── PositionResource         (position queries + export)
├── EventResource            (single event read)
├── PermissionsResource      (link-table management)
├── ServerResource           (server config + utilities)
├── StatisticsResource       (usage stats)
├── AuditResource            (audit log)
├── VideoStreamResource      (HLS streaming)
├── ShareResource            (share token generation)
├── PasswordResource         (password reset flow)
└── OidcResource             (Traccar as OIDC provider)

BaseObjectResource<T>        (GET/{id}, POST, PUT/{id}, DELETE/{id})
├── ExtendedObjectResource<T>  (+ GET / with groupId/deviceId filter)
│   ├── AttributeResource       /api/attributes/computed
│   ├── CommandResource         /api/commands
│   ├── DriverResource          /api/drivers
│   ├── GeofenceResource        /api/geofences
│   ├── MaintenanceResource     /api/maintenance
│   └── NotificationResource    /api/notifications
├── SimpleObjectResource<T>    (+ GET / with user-only filter)
│   ├── CalendarResource        /api/calendars
│   ├── GroupResource           /api/groups
│   ├── OrderResource           /api/orders
│   └── ReportResource          /api/reports
└── DeviceResource              /api/devices  (custom GET /)
    UserResource                /api/users    (custom GET /, overrides POST)
    HealthResource              /api/health
```

## Authentication model (5 methods)

Processed by `security/SecurityRequestFilter.java` before every request:

| Method | Trigger | Handler |
|---|---|---|
| **Session cookie** | No `Authorization` header + valid `Origin` | HTTP session attribute `userId` |
| **Bearer token** | `Authorization: Bearer <token>` | `TokenManager.verifyToken` + revocation check |
| **Basic auth** | `Authorization: Basic <base64>` | `LoginService.login(email, password, null)` |
| **LDAP** | (embedded in Basic/form; `ldap.url` configured) | `LdapProvider.login` |
| **OIDC** (consumer) | `GET /api/session/openid/auth` redirect | `OpenIdProvider` + callback |
| **TOTP 2FA** | Extra `code` param on `POST /api/session` | `GoogleAuthenticator.authorize` |

`@PermitAll` methods bypass the 401 rejection (login, registration, health, OIDC endpoints). OPTIONS requests always pass through.

## WebSocket protocol

Connect: `GET /api/socket` (upgrade). Auth via `?token=` or session cookie.

Client→Server messages (JSON):
- `{"logs": true}` — opt in to log record streaming
- `{"logs": false}` — opt out

Server→Client messages (JSON, one key per message):
- `{"positions": [...]}` — position update(s); initial burst on connect = all latest positions
- `{"devices": [...]}` — device state change
- `{"events": [...]}` — event notification
- `{"logs": [...]}` — log record (if opted in)
- `{}` — keepalive ping

## Permission graph enforcement

`PermissionsService.checkPermission(clazz, userId, objectId)`:
1. Admin → pass.
2. User accessing own user record → pass.
3. Otherwise → query `storage.getObject(clazz, Condition.And(Equals(id, objectId), Permission(User, userId, clazz)))`. Null → 403.

Join table naming: `tc_<decapOwner>_<decapProperty>` e.g. `tc_user_device`, `tc_group_geofence`.

## OpenAPI spec

`openapi.yaml` at the server module root — 3403 lines, v6.13.3. Documents every endpoint with request/response schemas.

## How to add a new REST endpoint

1. Create `FooResource extends ExtendedObjectResource<FooModel>` (or `SimpleObjectResource`, or `BaseResource`).
2. Annotate with `@Path("foos")` + `@Produces(APPLICATION_JSON)` + `@Consumes(APPLICATION_JSON)`.
3. Constructor calls `super(FooModel.class, "sortField", List.of("searchCol"))`.
4. Register in `MainModule.provideApplication()` (Jersey `ResourceConfig.register(FooResource.class)`).
5. Add to `openapi.yaml`.
