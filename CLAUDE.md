# Traccar Server — Module Guide

**Repo:** fork of `traccar/traccar` at `mohin1111/traccar`. Current version v6.13.3.
**Language:** Java 21 source / JDK 17 bytecode (enforced). Gradle. Apache-2.0.
**Role in Transport OS:** device-gateway service — keep as-is, wrap with our own multi-tenant API layer above it. Read [`../CLAUDE.md`](../CLAUDE.md) for the wider strategy.

---

## 1. Top-level layout

```
build.gradle           Gradle build; Java 21 src/target; checkstyle; protobuf; deps
settings.gradle        Single-module "tracker-server"
gradlew, gradlew.bat   Use these, not system gradle
gradle/                Wrapper JAR + checkstyle.xml
src/                   All Java sources and tests
schema/                Liquibase XML changelogs (per-version files + master)
docker/                Dockerfile.alpine/.debian/.ubuntu + compose/{mysql,timescaledb}.yaml
openapi.yaml           REST spec, 3403 lines, v6.13.3
templates/             Velocity email/notification templates
tools/                 Misc dev tools
traccar-web/           Git submodule pointer (frontend lives separately)
debug.xml              Sample debug config
```

Build outputs land in `target/` (not committed). Ignore `gradle/wrapper/` internals.

## 2. Build & run

| Task | Command |
|---|---|
| Build thin JAR + copy deps | `./gradlew assemble` → `target/tracker-server.jar` + `target/lib/` |
| Unit tests | `./gradlew test` |
| Checkstyle | `./gradlew checkstyleMain` (checkstyleTest disabled) |
| Run locally | `java -jar target/tracker-server.jar debug.xml` |
| Docker (MySQL) | `docker compose -f docker/compose/traccar-mysql.yaml up` |
| Docker (TimescaleDB) | `docker compose -f docker/compose/traccar-timescaledb.yaml up` |

**Java requirement:** JDK 21 to compile, JDK 17+ to run.

**Config file:** XML passed as first CLI arg. Key keys (typed catalog in `src/main/java/org/traccar/config/Keys.java`):
- `database.url`, `database.driver`, `database.user`, `database.password`
- `web.port` (default 8082)
- `<protocolname>.port` (e.g., `its.port=5050`)
- `database.changelog` (path to Liquibase master)

**Schema migration:** Liquibase runs on every startup inside `DatabaseModule.provideDataSource()`. Blocking, distributed-locked. All schema changes go through `schema/changelog-*.xml`.

## 3. Source tree (`src/main/java/org/traccar/`)

| Package | Files | Purpose | Key files |
|---|---|---|---|
| `api/` | 47 total | Jersey JAX-RS REST + WebSocket | `BaseObjectResource.java`, `AsyncSocket.java`, `AsyncSocketServlet.java`, `BaseResource.java`, `ExtendedObjectResource.java` |
| `api/resource/` | 24 | One per domain entity | `SessionResource`, `DeviceResource`, `PositionResource`, `UserResource`, `ReportResource`, `CommandResource`, `PermissionsResource`, `OidcResource`, `VideoStreamResource`, `HealthResource`, ... |
| `api/security/` | 9 | Auth filter + login + permissions | `SecurityRequestFilter.java`, `LoginService.java`, `PermissionsService.java` |
| `broadcast/` | 7 | Multi-node fan-out (cluster) | `BroadcastService.java`, `RedisBroadcastService.java`, `MulticastBroadcastService.java` |
| `command/` | 5 | Device command queue/dispatch | `CommandManager.java` |
| `config/` | 7 | Config loading + typed keys | `Config.java`, **`Keys.java`** |
| `database/` | 10 | Higher-level DB managers (not ORM) | `NotificationManager.java`, `CommandsManager.java`, `OpenIdProvider.java`, `LdapProvider.java`, `PositionBatchWriter.java` |
| `forward/` | 19 | Position/event forwarders | `PositionForwarderUrl/Kafka/Mqtt/Amqp/Redis/Wialon.java` |
| `geocoder/` | 24 | Reverse-geocoders | Google, Nominatim, **MapmyIndia**, Bing, HERE, TomTom, Mapbox, LocationIQ, Geoapify, etc. |
| `geofence/` | 4 | Geofence geometry (JTS) | `GeofenceCircle.java`, `GeofencePolygon.java` |
| `geolocation/` | 6 | Cell/WiFi geolocation | `GoogleGeolocationProvider.java` |
| `handler/` | 40 | Position pipeline + event detectors | `FilterHandler`, `ComputedAttributesHandler`, `GeofenceHandler`, `DatabaseHandler`, ... |
| `helper/` | 31 | Parsing/units/logging utilities | `Parser.java`, `PatternBuilder.java`, `UnitsConverter.java` |
| `mail/` | 3 | SMTP email | `MailManager.java` |
| `mapmatcher/` | 2 | Road snapping | `MapMatcherProvider.java` |
| `media/` | 2 | Media file storage | `MediaManager.java` |
| `model/` | 39 | Plain Java domain objects | `Position`, `Device`, `User`, `Permission`, ... (see §6) |
| `notification/` | 6 | Notification rule evaluation | `NotificationFormatter.java`, `NotificationMessage.java` |
| `notificators/` | 10 | Delivery channels | Mail, SMS, Firebase, Telegram, Pushover, Web, Traccar, Whatsapp, Command |
| `protocol/` | 671 | 265 protocol families (decoders+encoders+framers) | see §5 |
| `reports/` | 24 | Excel/HTML report generators | `TripsReportProvider`, `StopsReportProvider`, `ReportUtils` |
| `schedule/` | 11 | Scheduled background tasks | `ScheduleManager`, `TaskExpirations`, `TaskSessionTimeout` |
| `session/` | 14 | Device session lifecycle + cache | `ConnectionManager`, `DeviceSession`, `cache/CacheManager` |
| `sms/` | 3 | SMS sending (HTTP + AWS SNS) | `SmsManager`, `SnsSmsClient`, `HttpSmsClient` |
| `speedlimit/` | 3 | Overpass API lookups | `OverpassSpeedLimitProvider` |
| `storage/` | 12 | Custom JDBC "ORM" | **`DatabaseStorage.java`**, `QueryBuilder.java`, `StorageName.java` |
| `web/` | 11 | Jetty + static serving | `WebServer.java` |

Root-level critical files: `Main.java`, **`MainModule.java`** (Guice DI), **`BasePipelineFactory.java`**, **`ProcessingHandler.java`**, `ServerManager.java`, `BaseProtocol.java`, `BaseProtocolDecoder.java`.

## 4. The position pipeline (THE critical path)

**Files:** `BasePipelineFactory.java` (Netty), `ProcessingHandler.java` (per-position chain).

**Netty channel pipeline** (built per protocol in `BasePipelineFactory.initChannel`):

1. Transport handlers (protocol-specific framing) — `addTransportHandlers` (subclass)
2. `IdleStateHandler` — close idle TCP connections
3. `OpenChannelHandler` — register/deregister with `TrackerConnector`
4. `NetworkForwarderHandler` — mirror raw bytes to another host (optional)
5. `NetworkMessageHandler` — wrap `DatagramPacket` → `SocketAddress`-aware message
6. `StandardLoggingHandler` — hex-dump for debug
7. `AcknowledgementHandler` — delay TCP ACK until processed
8. Protocol handlers — `addProtocolHandlers` (subclass): frame decoder, string codecs, `*ProtocolDecoder`
9. `RemoteAddressHandler` — stamp remote IP
10. **`ProcessingHandler`** — main per-position chain (below)
11. `MainEventHandler` — write protocol ACK

**Inside `ProcessingHandler`** (sequential per-device chain):

1. `ComputedAttributesHandler.Early` — early JEXL evaluation
2. `OutdatedHandler` — drop stale positions
3. `TimeHandler` — fix/adjust timestamps
4. `GeolocationHandler` — cell/WiFi → coords if no GPS
5. `HemisphereHandler` — hemisphere-sign fix
6. `MapMatcherHandler` — snap to road
7. `DistanceHandler` — odometer delta
8. `FilterHandler` — drop invalid/duplicate
9. `GeofenceHandler` — geofence membership
10. `GeocoderHandler` — reverse-geocode to address
11. `SpeedLimitHandler` — attach road speed limit
12. `MotionHandler` — motion state
13. `ComputedAttributesHandler.Late` — late JEXL evaluation
14. `DriverHandler` — RFID/iButton → driver
15. `CopyAttributesHandler` — copy defaults from group/device
16. `EngineHoursHandler` — accumulate engine hours
17. `PositionForwardingHandler` — push to external forwarders
18. `DatabaseHandler` — persist to `tc_positions`

Then event handlers fire in parallel: overspeed, geofence enter/exit, ignition, motion, alarm, fuel, driver, maintenance, proximity, media, command-result.

## 5. Protocol layer

**Count:** exactly **265** `*Protocol.java` files in `src/main/java/org/traccar/protocol/`.

**AIS-140 (Indian VLT) = `Its` protocol:**
- `ItsProtocol.java` (46 lines) — registers TCP `TrackerServer` with `StringDecoder`/`StringEncoder` + `ItsProtocolDecoder`; supports `ENGINE_STOP` (immobilizer) via `ItsProtocolEncoder`.
- `ItsProtocolDecoder.java` (285 lines) — decodes `$,EVENT,VENDOR,FW,STATUS,IMEI,…` frames with IMEI, lat/lon, speed, course, ignition, battery, cell towers (MCC/MNC/LAC/CID), odometer, ADC, accelerometer axes. Handles NRM/EPB/RLP/C subtypes for different ITS firmware variants.
- `ItsProtocolDecoderTest.java` — tests login null responses, message subtypes, odometer attribute extraction using Indian MCC 404 cell data.

**Indian / regionally relevant protocols:**
- `AtrackProtocol` — Atrack AX series (binary + text)
- `AquilaProtocol` — Aquila VT series (Indian OEM)
- `IdplProtocol` — IDPL (Indian OEM)
- `RaceDynamicsProtocol` — RaceDynamics (India)
- `Jt808Protocol` — JT/T 808 Chinese standard (widely used via Chinese hardware in India)
- `TeltonikaProtocol` — Teltonika FMxxx (global, common in India)
- `GalileoProtocol`, `TopflytechProtocol`, `TeraTrackProtocol`

**Adding a new protocol:**
1. Create `FooProtocol extends BaseProtocol` — call `addServer(new TrackerServer(...))` in ctor; wire frame decoder + `FooProtocolDecoder` in `addProtocolHandlers()`.
2. Create `FooProtocolDecoder extends BaseProtocolDecoder` — implement `decode(Channel, SocketAddress, Object)`; call `getDeviceSession(channel, remoteAddress, identifier)`; populate `Position`; return it.
3. **No registration code needed** — `ServerManager` uses `ClassScanner.findSubclasses(BaseProtocol.class, "org.traccar.protocol")` for auto-discovery. Restrict via `protocols.enable` config if desired.
4. Add `FooProtocolDecoderTest extends ProtocolTest` — `verifyPosition(decoder, binary(...))` / `verifyNull` / `verifyAttribute`.

## 6. Data model & storage

**Model classes** (`src/main/java/org/traccar/model/`, 39 total):
- Hierarchy: `BaseModel` → `ExtendedModel` (adds JSON `attributes`) → `GroupedModel` (adds `groupId`)
- Core entities: `Device` (`tc_devices`), `Position` (`tc_positions`), `User` (`tc_users`), `Group`, `Geofence`, `Driver`, `Calendar`, `Event`, `Notification`, `Maintenance`, `Command`, `Attribute` (computed/JEXL), `Order`, `Server`, `Statistics`, `LogRecord`, `Report`, `QueuedCommand`, `RevokedToken`, `Message`, `DeviceAccumulators`, `CellTower`, `Network`, `WifiAccessPoint`
- Permission proxies: `Permission` (generic), `ManagedUser` (User→User), `LinkedDevice` (User→Device)

**Storage layer (custom JDBC, no Hibernate/JPA):**
- `@StorageName("tc_xxx")` annotation maps a model class to a DB table
- `@QueryIgnore` excludes getters from column mapping
- `Storage` (abstract) defines: `getObjects`, `getObject`, `addObject`, `updateObject`, `removeObject`, `getPermissions`, `addPermission`, `removePermission`
- `DatabaseStorage` — implements `Storage` with HikariCP + `QueryBuilder`; detects DB type from JDBC metadata; builds SQL from `Request(Columns, Condition, Order)`
- `QueryBuilder` — reflection-based: iterates JavaBean getters/setters, JSON-serializes `attributes`, supports `Condition.And/Or/Equals/Permission`
- `MemoryStorage` — in-memory impl used in tests
- **Supported DBs:** H2, MySQL, MariaDB, PostgreSQL, MSSQL (drivers bundled in `build.gradle`)

**Liquibase:** master at `schema/changelog-master.xml`; current version `changelog-6.13.0.xml`. Applied on startup with distributed lock.

**`tc_users` key columns:** `id, name, email (unique), hashedpassword, salt, administrator (bool), disabled, expirationtime, devicelimit (-1=unlimited), userlimit (0=normal user), token, login (LDAP field), readonly, devicereadonly, limitcommands, attributes (JSON)`

**`tc_devices` key columns:** `id, name, uniqueid (unique — IMEI/identifier), groupid (FK), lastupdate, positionid, attributes, phone, model, contact, category, disabled`

## 7. REST API + WebSocket

**Resources** (`src/main/java/org/traccar/api/resource/`):
`SessionResource` (login/logout/OIDC), `DeviceResource`, `UserResource`, `GroupResource`, `PositionResource`, `GeofenceResource`, `AttributeResource` (computed), `NotificationResource`, `CommandResource`, `EventResource`, `DriverResource`, `CalendarResource`, `MaintenanceResource`, `ServerResource`, **`PermissionsResource`** (add/remove link rows), `ReportResource`, `StatisticsResource`, `OrderResource`, `ShareResource` (temp shareable links), `PasswordResource`, `OidcResource`, `AuditResource`, `VideoStreamResource`, `HealthResource` (`/api/server/health`).

**WebSocket:**
- `AsyncSocket.java` implements `ConnectionManager.UpdateListener`
- Clients connect to `/api/socket`; `AsyncSocketServlet` handles upgrade
- On connect, socket registers with `ConnectionManager`; receives `updateDevice`/`updatePosition`/`updateEvent` callbacks, JSON-serialized and pushed
- Client implicitly subscribes to all devices it has permission to see (filtered by userId)

**Authentication** (`api/security/SecurityRequestFilter.java`):
- **Session cookie** — HTTP session set on POST `/api/session` (email+password). CSRF-validated.
- **Bearer token** — `Authorization: Bearer <token>` via `TokenManager` (JWT-like); revocation in `tc_revoked_tokens`
- **Basic auth** — `Authorization: Basic <base64>` decoded to email:password
- **LDAP** — `LdapProvider` if `ldap.url` configured
- **OIDC** — `OpenIdProvider` + `OidcResource` for OAuth2/OIDC SSO
- **2FA (TOTP)** — `GoogleAuthenticator` library; `LoginService.login(email, password, code)`

**OpenAPI spec:** `openapi.yaml` — 3403 lines, v6.13.3.

## 8. Permissions & tenancy

**Model:** `model/Permission.java`. Holds a 2-entry `LinkedHashMap`: `{ownerTypeId → ownerId, propertyTypeId → propertyId}`. Storage table name derived as `tc_<decapitalizedOwner>_<decapitalizedProperty>` (e.g., `tc_user_device`, `tc_user_group`, `tc_group_geofence`).

**Enforcement:** `api/security/PermissionsService.java`:
- `checkAdmin(userId)` — throws if `user.administrator == false`
- `checkUser(userId, managedUserId)` — admin OR row exists in `tc_user_user` (via `ManagedUser` proxy)
- `checkPermission(userId, clazz, objectId)` — verifies link in appropriate join table

**Tenancy:** NO `tenant_id` column anywhere in the codebase. Confirmed absent. Tenancy is graph-based — an "org admin" is a regular user with rows in `tc_user_user` linking to sub-users, and those sub-users' devices in `tc_user_device`.

**Role model — three effective roles:**
1. `administrator = true` → full access
2. Manager → has `tc_user_user` rows
3. Regular user → only sees devices linked via `tc_user_device`

**For Transport OS:** wrap, don't extend. Build our own tenant-aware API gateway in front; treat Traccar as a per-tenant-isolated device gateway (one Traccar instance per tenant, or single instance with strict permission enforcement).

## 9. Tests

- 422 Java files under `src/test/`
- Protocol decoder test pattern: `*ProtocolDecoderTest extends ProtocolTest`; use `inject(new XxxProtocolDecoder(null))`; call `verifyPosition`/`verifyNull`/`verifyAttribute` with raw frames
- `ItsProtocolDecoderTest.java` — exists, covers login responses + NRM/EPB/RLP/C + odometer extraction with Indian MCC 404 data

## 10. Extension points

**Computed Attributes (JEXL):**
- Defined in `tc_attributes`: `expression` (JEXL string), `attribute` (output key), `type` (cast)
- Executed in `handler/ComputedAttributesProvider.java` with `JexlEngine` + `JexlSandbox` + `MapContext`
- Two phases: `Early` (before filter) and `Late` (after geocode/geofence)

**Position forwarders** (`forward/`):
- HTTP webhook (`PositionForwarderUrl`)
- Kafka (`PositionForwarderKafka`)
- MQTT (`PositionForwarderMqtt`, HiveMQ client)
- RabbitMQ AMQP (`PositionForwarderAmqp`)
- Redis pub/sub (`PositionForwarderRedis`)
- Wialon UDP (`PositionForwarderWialon`)
- Event-forwarder variants for Kafka/MQTT/AMQP/JSON

**Broadcast bus** (`broadcast/`):
- Purpose: fan-out device/position/event updates across multiple Traccar nodes
- Implementations: `NullBroadcastService` (single-node), `MulticastBroadcastService` (UDP multicast), `RedisBroadcastService` (Redis pub/sub)

**Notificators** (`notificators/`): Mail, SMS, Firebase (push), Telegram, Pushover, Web (in-browser via WebSocket), Traccar app, WhatsApp, Command (send device command as action).

**Geocoders supported (22+):** Google, Nominatim (OSM), Bing, HERE, TomTom, Mapbox, MapQuest, OpenCage, LocationIQ, Geoapify, **MapmyIndia**, GeocodeFarm, GeocodeXyz, BAN (France), Gisgraphy, FactualGeocoder, PositionStack, PlusCodes, GeocodeJson, MapTiler.

## 11. Things future-Claude must know

**Read first, in this order:**
1. `ProcessingHandler.java` — the position lifecycle, the single most important file
2. `BasePipelineFactory.java` — how Netty channels are wired
3. `BaseProtocolDecoder.java` — the `getDeviceSession()` / `Position` population contract all 265 decoders follow
4. `DatabaseStorage.java` + `StorageName.java` together — custom reflection-based ORM (no JPA)
5. `MainModule.java` — Guice DI bindings; this is where everything wires up
6. `Permission.java` + `api/security/PermissionsService.java` — the tenancy model

**Key invariants / gotchas:**
- DI framework is **Google Guice**, not Spring. All singletons declared in `MainModule.java` and protocol-specific modules.
- Protocol auto-discovery via `ClassScanner.findSubclasses()` — no manual registration.
- Schema changes ONLY via Liquibase changesets in `schema/`. Never run DDL directly.
- Build produces **thin** JAR at `target/tracker-server.jar` with deps in `target/lib/` — always use `./gradlew assemble`, never `jar` alone.
- Tests use `MemoryStorage` (no real DB). Use it for unit work; integration testing needs a real DB.
- For multi-tenancy, do not try to retrofit `tenant_id` — wrap Traccar with a per-tenant gateway above it.

**License notice:** Apache-2.0. We can fork, modify, redistribute (closed or open), and sell SaaS. Just preserve NOTICE.
