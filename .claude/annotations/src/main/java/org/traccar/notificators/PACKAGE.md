# `notificators/` — Notification delivery channels

10 `Notificator` implementations sending event alerts through different channels. All extend `Notificator` (abstract) which calls `NotificationFormatter.formatMessage()` before delegating to the channel-specific `send()`.

## File index

| File | Channel | Key config |
|---|---|---|
| `Notificator.java` | Abstract base; defines `send(Notification, User, Event, Position)` contract | — |
| `NotificatorMail.java` | SMTP email via `MailManager` | `mail.smtp.*` |
| `NotificatorSms.java` | SMS via `SmsManager` (HTTP or AWS SNS) | `sms.*` |
| `NotificatorFirebase.java` | Firebase Cloud Messaging push (Android + iOS via `notificationTokens` user attribute) | `notificator.firebase.serviceAccount` |
| `NotificatorTelegram.java` | Telegram Bot API text + optional location message | `notificator.telegram.key`, `chatId` |
| `NotificatorPushover.java` | Pushover push notification API | `notificator.pushover.token`, `user` |
| `NotificatorWeb.java` | In-browser WebSocket push via `ConnectionManager` (the events drawer) | — (no external API) |
| `NotificatorTraccar.java` | Traccar cloud push service (alternative to Firebase for Traccar mobile app) | `notificator.traccar.key` |
| `NotificatorWhatsapp.java` | WhatsApp Cloud API (Meta) | `notificator.whatsapp.*` |
| `NotificatorCommand.java` | Sends a device command as a notification action (e.g., immobilize on overspeed) | per-notification `command` attribute |

## Dependency graph

```
NotificatorManager (notification/) → Map<String, Notificator>
Each Notificator:
  → NotificationFormatter.formatMessage() → NotificationMessage (subject + digest + full + priority)
  → channel-specific delivery (HTTP, SMTP, WebSocket, FCM, etc.)
```

## Firebase notificator detail

`NotificatorFirebase`:
1. Reads `notificationTokens` from user attributes (comma-separated FCM registration tokens).
2. Builds `MulticastMessage` with title=`message.subject()`, body=`message.digest()`.
3. Sets `HIGH` priority for priority notifications.
4. On token failure (`INVALID_ARGUMENT` or `UNREGISTERED`), removes bad tokens from user attributes + invalidates cache.
5. Requires `notificator.firebase.serviceAccount` = full JSON service account.

## Telegram notificator detail

`NotificatorTelegram`:
- Uses per-user `telegramChatId` attribute if set, else global `notificator.telegram.chatId`.
- Optionally sends a location message after text (when `notificator.telegram.sendLocation=true`).
- Supports HTTP proxy via `notificator.telegram.proxyUrl` for regions where Telegram is blocked.

## WebSocket notificator

`NotificatorWeb` pushes directly to all active WebSocket connections for the user via `ConnectionManager.updateEvent`. No config needed; always available. This is what populates the events bell/drawer in the web UI.

## How to add a new notificator channel

1. Extend `Notificator` (abstract) or implement the notification contract directly.
2. Override `send(User user, NotificationMessage message, Event event, Position position)`.
3. Add the type string to `NotificatorManager`'s type map.
4. Add any needed config keys to [[Keys.java]].
5. Users configure notifications in the web UI by selecting notificator types for their notification rules.

## Transport OS relevance

For AIS-140 SOS alert delivery:
- `NotificatorWeb` for control-room real-time alerts (already works via WebSocket).
- `NotificatorFirebase` for mobile control-room app push.
- `NotificatorSms` for the 5-number panic SMS (AIS-140 requirement) — configure `sms.http.url` to an Indian SMS gateway (MSG91, Exotel, etc.).
- Consider a custom `NotificatorCommand` → `ENGINE_STOP` for automated immobilization on geofence violation.
