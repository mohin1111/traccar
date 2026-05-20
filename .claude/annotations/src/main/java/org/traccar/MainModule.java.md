# MainModule.java

**Role:** Central Guice DI module. Every application-level singleton is bound here as a `@Provides @Singleton` factory method. This file is the single place to learn what services are wired and under what conditions they are active.
**Fits in:** Instantiated by `Main.run()` alongside `DatabaseModule` and `WebModule`. All protocol decoders, handlers, and forwarders receive their dependencies via injection from bindings declared here.
**Read next:** [[Main.java]] (creates the injector), [[ServerManager.java]] (consumes `Config`+`Injector`), [[ProcessingHandler.java]] (consumes all the `@Nullable` handler singletons)

## Public API

### `configure()` (lines 125-129)
Only three hard bindings:
- `configFile` string constant (named)
- `Config.class` as eager singleton (reads XML on first instantiation)
- `Timer.class` → `HashedWheelTimer` singleton (used by protocol decoders for timeouts)

### Factory methods (all `@Singleton @Provides`)

| Method | Returns | Active when |
|---|---|---|
| `provideExecutorService` | `ExecutorService` | Always (`newCachedThreadPool`) |
| `provideStorage` | `Storage` | Always; memory if `database.memory=true` |
| `provideObjectMapper` | `ObjectMapper` | Always; BlackbirdModule + JSONPModule |
| `provideClient` | JAX-RS `Client` | Always (HTTP client for geocoders/forwarders) |
| `provideSmsManager` | `SmsManager` | `sms.http.url` OR `sms.aws.region` present |
| `provideMailManager` | `MailManager` | Always; `LogMailManager` if `mail.debug=true` |
| `provideLdapProvider` | `LdapProvider` | `ldap.url` present |
| `provideOpenIDProvider` | `OpenIdProvider` | `openid.clientId` present |
| `provideWebServer` | `WebServer` | `web.port > 0` |
| `provideGeocoder` | `Geocoder` | `geocoder.enable=true`; 20 provider types |
| `provideGeolocationProvider` | `GeolocationProvider` | `geolocation.enable=true` |
| `provideSpeedLimitProvider` | `SpeedLimitProvider` | `speedlimit.enable=true` |
| `provideGeolocationHandler` | `GeolocationHandler` | Only if `GeolocationProvider` non-null |
| `provideGeocoderHandler` | `GeocoderHandler` | Only if `Geocoder` non-null |
| `provideMapMatcher` / `Handler` | `MapMatcher/Handler` | `map.matcher.enable=true` |
| `provideSpeedLimitHandler` | `SpeedLimitHandler` | Only if provider non-null |
| `provideCopyAttributesHandler` | `CopyAttributesHandler` | `processing.copyAttributes.enable=true` |
| `provideFilterHandler` | `FilterHandler` | `filter.enable=true` |
| `provideBroadcastService` | `BroadcastService` | `broadcast.type` key present |
| `provideEventForwarder` | `EventForwarder` | `event.forward.url` present |
| `providePositionForwarder` | `PositionForwarder` | `forward.url` present |
| `provideVelocityEngine` | `VelocityEngine` | Always (templates for notifications) |

## Key flows

### Nullable handler pattern
Handlers that depend on optional external services (geocoder, geolocation, speed limit, map matcher) return `null` when the service is disabled. `ProcessingHandler` filters these out with `.filter(Objects::nonNull)` when building its pipeline list. This is the canonical pattern for conditional pipeline stages.

### Geocoder selection (lines 224-245)
20 geocoder types dispatched via a `switch` statement. `mapmyindia` case (line 237) is directly relevant for Indian deployment. Key+URL are read from config; all share the same `AddressFormat` and `statisticsManager` callback.

### Position forwarder selection (lines 391-401)
7 forwarder types: `url` (default HTTP webhook), `json`, `amqp`, `kafka`, `mqtt`, `redis`, `wialon`. This is the integration point to push positions to external systems (our Transport OS API).

## Gotchas / non-obvious

- Methods returning `null` **must** have corresponding `@Nullable` annotations on injection sites that receive them, otherwise Guice throws a provision error.
- `provideWebServer` is **NOT `@Singleton`** — it is `@Provides` only. The `WebServer` itself is not a singleton by design; `Main.run()` starts it exactly once.
- The `ObjectMapper` disables `WRITE_DATES_AS_TIMESTAMPS` (line 153) — all JSON dates are ISO-8601 strings. Don't rely on numeric epoch in REST responses.
- `VelocityEngine` (lines 406-414) sets the file path for templates and the web URL from config. Notification email/SMS templates live in `templates/`.

## Line index

- 125-129 — `configure()` hard bindings
- 131-135 — `ExecutorService` (cached thread pool)
- 137-145 — `Storage` (memory vs DB)
- 147-154 — `ObjectMapper`
- 163-170 — `SmsManager`
- 173-181 — `MailManager`
- 213-250 — `Geocoder` factory (20 providers)
- 252-267 — `GeolocationProvider` (4 providers)
- 269-281 — `SpeedLimitProvider`
- 356-368 — `BroadcastService` (null/multicast/redis)
- 370-383 — `EventForwarder` (json/amqp/kafka/mqtt)
- 385-402 — `PositionForwarder` (7 types)
- 406-414 — `VelocityEngine`
