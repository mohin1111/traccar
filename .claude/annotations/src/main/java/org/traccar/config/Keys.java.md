# Keys.java

**Role:** Exhaustive static catalog of every typed config key in Traccar. The single source of truth for what can be configured, at what scope, and with what default. Extremely large file (~1400+ lines of key constants).
**Fits in:** Referenced everywhere — every service that reads config does `config.getX(Keys.SOME_KEY)`. Keys.java is the "schema" of `traccar.xml`.
**Read next:** [[Config.java]] (consumes keys), [[ConfigKey.java]] (key type hierarchy), [[PortConfigSuffix.java]] (port defaults)

## Public API

All members are `public static final`. No instances. Key groups:

### Protocol-level suffix constants (lines ~27-290)
`ConfigSuffix<T>` instances — combined with `withPrefix("protocolName")` to produce full keys.
- `PROTOCOL_ADDRESS` — bind address per protocol (`.address`)
- `PROTOCOL_PORT` — listen port per protocol (`.port`) — see [[PortConfigSuffix.java]] for defaults
- `PROTOCOL_TIMEOUT` — idle TCP connection timeout (`.timeout`)
- `PROTOCOL_SSL` — enable SSL on the protocol server
- `PROTOCOL_DEVICES`, `PROTOCOL_INTERVAL` — polling protocol config
- `PROTOCOL_EXTENDED`, `PROTOCOL_CAN`, `PROTOCOL_ACK`, `PROTOCOL_UTF8` — per-protocol feature flags
- `PROTOCOL_DEVICE_PASSWORD`, `PROTOCOL_MASK`, `PROTOCOL_CONFIG`, `PROTOCOL_FORM`, etc.

### Database keys (lines ~300-400)
- `DATABASE_URL`, `DATABASE_DRIVER`, `DATABASE_USER`, `DATABASE_PASSWORD`
- `DATABASE_CHANGELOG` — Liquibase master file path
- `DATABASE_MAX_IDLE_CONNECTIONS`, `DATABASE_MAX_POOL_SIZE`
- `DATABASE_POSITION_BATCH_SIZE`, `DATABASE_POSITION_BATCH_INTERVAL` — batch writer tuning
- `DATABASE_THROTTLE_UNKNOWN` — throttle unknown device lookups

### Web / server keys
- `WEB_PORT` (default 8082), `WEB_ADDRESS`, `WEB_PATH`, `WEB_ORIGIN`
- `WEB_LOCALIZATION_PATH` — path to l10n JSON files
- `SERVER_STATISTICS` — URL to POST usage stats

### Auth / SSO keys
- `LDAP_URL`, `LDAP_BASE`, `LDAP_ID_ATTRIBUTE`, `LDAP_USER`, `LDAP_PASSWORD`, `LDAP_ADMIN_FILTER`, `LDAP_ADMIN_GROUP`, `LDAP_SEARCH_FILTER`
- `OPENID_CLIENT_ID`, `OPENID_CLIENT_SECRET`, `OPENID_ISSUER_URL`, `OPENID_AUTH_URL`, `OPENID_TOKEN_URL`, `OPENID_USERINFO_URL`, `OPENID_FORCE`, `OPENID_ADMIN_GROUP`, `OPENID_ALLOW_GROUP`, `OPENID_GROUPS_CLAIM_NAME`

### Geocoder keys
- `GEOCODER_ENABLE`, `GEOCODER_TYPE` (google/nominatim/mapbox/mapmyindia/etc.), `GEOCODER_URL`, `GEOCODER_KEY`, `GEOCODER_CACHE_SIZE`, `GEOCODER_ON_REQUEST`, `GEOCODER_ADDRESS_FORMAT`

### Geolocation keys
- `GEOLOCATION_ENABLE`, `GEOLOCATION_TYPE` (google/opencellid/unwired/universal), `GEOLOCATION_URL`, `GEOLOCATION_KEY`

### Position forwarding keys
- `FORWARD_ENABLE`, `FORWARD_URL`, `FORWARD_HEADER`, `FORWARD_EVENTS`
- `FORWARD_KAFKA_BOOTSTRAP_SERVERS`, `FORWARD_KAFKA_TOPIC`
- `FORWARD_MQTT_URL`, `FORWARD_MQTT_TOPIC`, `FORWARD_MQTT_CLIENT_ID`
- `FORWARD_AMQP_URL`, `FORWARD_AMQP_EXCHANGE`, `FORWARD_AMQP_ROUTING_KEY`
- `FORWARD_REDIS_URL`, `FORWARD_REDIS_CHANNEL`

### Notification keys
- `NOTIFICATION_BLOCK_USERS` — comma-separated user IDs to suppress all notifications
- `NOTIFICATOR_TIME_THRESHOLD` — skip notifications for events older than X ms
- `NOTIFICATOR_TELEGRAM_KEY`, `NOTIFICATOR_TELEGRAM_CHAT_ID`, `NOTIFICATOR_TELEGRAM_SEND_LOCATION`
- `NOTIFICATOR_FIREBASE_SERVICE_ACCOUNT` — JSON service account blob
- `NOTIFICATOR_TRACCAR_KEY` — Traccar push service API key
- `NOTIFICATOR_PUSHOVER_TOKEN`, `NOTIFICATOR_PUSHOVER_USER`

### SMS keys
- `SMS_HTTP_URL`, `SMS_HTTP_AUTHORIZATION`, `SMS_HTTP_PARAMETER_FROM`, `SMS_HTTP_PARAMETER_TO`, `SMS_HTTP_PARAMETER_MESSAGE`
- `SMS_AWS_REGION`, `SMS_AWS_ACCESS` (SNS SMS)

### Broadcast keys
- `BROADCAST_TYPE` (null/redis/multicast), `BROADCAST_ADDRESS`, `BROADCAST_PORT`, `BROADCAST_SECONDARY`
- `BROADCAST_REDIS_URL`

### Media / speed / map keys
- `MEDIA_PATH` — filesystem path for device-uploaded media files
- `SPEED_LIMIT_ENABLE`, `SPEED_LIMIT_URL`, `SPEED_LIMIT_ACCURACY`
- `COMMAND_SENDER` (per-device), `COMMAND_CLIENT_SERVICE_ACCOUNT`
- `SERVER_BUFFERING_THRESHOLD` — position re-ordering buffer in ms

## Gotchas / non-obvious

- **Config vs. server/user/device scope** — every key lists its `KeyType` scope. Only `CONFIG` keys live in `traccar.xml`; `SERVER`/`USER`/`DEVICE` keys are entity attributes stored in the DB.
- **All suffix keys need `.withPrefix(protocolName)`** — never read a suffix directly from `Config`.
- **Numeric defaults are exactly 0** when missing unless explicitly set — e.g., `GEOCODER_CACHE_SIZE` defaults to 0, which disables caching.
- **`GEOCODER_TYPE = "mapmyindia"`** is the key to activate the MapmyIndia geocoder for Transport OS Indian-address reverse geocoding.
- **ITS protocol port** is defined in [[PortConfigSuffix.java]] as 5179, not here.

## Line index

- 27-37 — `PROTOCOL_ADDRESS` and `PROTOCOL_PORT` (suffix constants)
- 43-83 — other protocol suffix constants (timeout, ssl, devices, etc.)
- 74 — `DEVICE_PASSWORD` (device-scoped, not a suffix)
- ~250-350 — database keys
- ~350-500 — web, server, filter, geocoder keys
- ~500-700 — geolocation, map matcher, notificator, SMS keys
- ~700-900 — forward, broadcast, media, speed limit, command keys
- ~900+ — schedule, report, session keys
