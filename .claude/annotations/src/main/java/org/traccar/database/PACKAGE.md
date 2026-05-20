# `database/` — Higher-level DB managers and auth providers

Domain-level services that sit above the raw `storage/` JDBC layer. Not a traditional ORM layer — each class has a specific responsibility. The "database" name is historical; several files here are more about auth or I/O than DB access.

## File index

| File | One-liner | Annotation |
|---|---|---|
| `NotificationManager.java` | Persists events, forwards to external systems, fans out notifications to users | [NotificationManager.java.md](NotificationManager.java.md) |
| `CommandsManager.java` | Dispatches outbound device commands — live TCP, SMS, push, or queued | [CommandsManager.java.md](CommandsManager.java.md) |
| `PositionBatchWriter.java` | Optional async bulk inserter for `tc_positions` to reduce per-row DB overhead | [PositionBatchWriter.java.md](PositionBatchWriter.java.md) |
| `StatisticsManager.java` | Per-day in-memory counters; rolls over to DB row + optional upstream POST daily | [StatisticsManager.java.md](StatisticsManager.java.md) |
| `BufferingManager.java` | Re-orders out-of-sequence positions using timed per-device sorted queues | [BufferingManager.java.md](BufferingManager.java.md) |
| `DeviceLookupService.java` | Device-by-uniqueId lookup with exponential backoff throttle for unknown IDs | [DeviceLookupService.java.md](DeviceLookupService.java.md) |
| `LocaleManager.java` | Loads + caches l10n JSON bundles for notification template translation | [LocaleManager.java.md](LocaleManager.java.md) |
| `MediaManager.java` | Filesystem store for device-uploaded binary media (photos, video clips) | [MediaManager.java.md](MediaManager.java.md) |
| `OpenIdProvider.java` | OIDC/OAuth2 SSO via Nimbus SDK — auth redirect + callback + user upsert | [OpenIdProvider.java.md](OpenIdProvider.java.md) |
| `LdapProvider.java` | LDAP authentication — bind, user lookup, group-based admin check | [LdapProvider.java.md](LdapProvider.java.md) |

## Dependency graph

```
NotificationManager
 ├── Storage (persist events)
 ├── CacheManager (resolve notifications + users)
 ├── EventForwarder? (external fan-out)
 ├── NotificatorManager (delivery channels)
 └── Geocoder? (on-request address)

CommandsManager  implements BroadcastInterface
 ├── Storage (queue/dequeue)
 ├── ServerManager (protocol-level text commands)
 ├── SmsManager? (SMS channel)
 ├── ConnectionManager (live session commands)
 ├── BroadcastService (cross-node queue signals)
 ├── NotificationManager (queued-command-sent events)
 ├── CacheManager (device token updates)
 └── CommandSenderManager (Firebase / FindHub / Traccar push)

PositionBatchWriter → Storage (bulk addObjects)
StatisticsManager  → Storage + JAX-RS Client (HTTP POST)
BufferingManager   → (Netty timer, no DB)
DeviceLookupService → Storage + Netty timer
LocaleManager      → filesystem (JSON l10n files)
MediaManager       → filesystem (binary media)
OpenIdProvider     → LoginService (user upsert) + Nimbus SDK
LdapProvider       → JNDI LDAP
```

## Critical flows (multi-file)

### Event delivery (end of position pipeline)
1. `ProcessingHandler` event handlers detect events → call `notificationManager.updateEvents(Map<Event,Position>)`.
2. `NotificationManager.updateEvent` persists → forwards → fans out via `notificatorManager`.
3. `CacheManager.getDeviceNotifications` returns pre-resolved notification rules (cached).
4. Per user, per notificator type: `notificatorManager.getNotificator(type).send(...)`.

### Command dispatch
1. `POST /api/commands/send` → `CommandResource` → `commandsManager.sendCommand(command)`.
2. If device offline: command persisted to `tc_queued_commands`.
3. On device reconnect: `BaseProtocolDecoder.getDeviceSession` → `commandsManager.readQueuedCommands(deviceId)`.
4. Cross-node: `broadcastService.updateCommand(true, deviceId)` → Redis/multicast → other nodes flush.

### Position write path
`DatabaseHandler.processPosition(position)` → `positionBatchWriter.submit(position)`.future → `storage.addObject/addObjects`.

## How to add a new auth provider (e.g., SAML)

1. Create `SamlProvider.java` in this package.
2. Read config keys from [[Keys.java]] (add new ones if needed).
3. Call `loginService.login(email, name, administrator)` after successful assertion.
4. Register conditional binding in `MainModule`: `if (config.hasKey(Keys.SAML_ENABLED)) { bind(SamlProvider.class); }`.
5. Add `SamlResource` in `api/resource/` for the callback endpoint.

## How to enable position batching

In `traccar.xml`:
```xml
<entry key="database.positionBatchSize">100</entry>
<entry key="database.positionBatchInterval">500</entry>  <!-- ms -->
```
`PositionBatchWriter` will insert up to 100 positions every 500 ms. Reduce DB connections needed at high throughput.
