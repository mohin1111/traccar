# `notification/` — Notification message formatting

Six files that transform events into formatted notification messages using Velocity templates, and expose the notificator registry.

## File index

| File | One-liner |
|---|---|
| `NotificationFormatter.java` | Builds `VelocityContext` (device, event, position, user, geofence) and calls `TextTemplateFormatter.formatMessage()` |
| `TextTemplateFormatter.java` | Loads Velocity template by event type + language, renders subject + body |
| `NotificationMessage.java` | Record: `subject`, `digest` (short body), `full` (HTML body), `priority` flag |
| `NotificatorManager.java` | Registry of notificator implementations; `getNotificator(type)` returns singleton |
| `PropertiesProvider.java` | Resolves Velocity template properties for server/user context |
| `MessageException.java` | Checked exception thrown by notificator `send()` on delivery failure |

## Dependency graph

```
NotificationManager (database/) → NotificatorManager + NotificationFormatter
NotificationFormatter → TextTemplateFormatter → Velocity + LocaleManager (database/)
NotificatorManager → Map<String, Notificator> (built from notificators/ package)
```

## Velocity template location

Templates live in `templates/<eventType>/` relative to the server install dir:
- `templates/overspeed/full.vm` — HTML email body
- `templates/overspeed/short.vm` — SMS / Telegram body
- `templates/overspeed/subject.vm` — email subject

The `LocaleManager.getTemplateFile(root, path, language, fileName)` resolves language-specific templates with English fallback.

## Velocity context variables

Available in all templates:
- `$device` — `Device` model object
- `$event` — `Event` model object (type, attributes)
- `$position` — `Position` model object (lat, lon, speed, etc.)
- `$user` — `User` model object (email, name, language)
- `$server` — `Server` model object (attributes)
- `$notification` — `Notification` rule
- `$translations` — `Map<String, String>` l10n bundle for user's language
- `$speedUnit`, `$distanceUnit`, `$volumeUnit` — unit strings
- `$geofence`, `$maintenance`, `$driver` — optional, when relevant

## How to add a new notification template (new event type)

1. Create `templates/<eventType>/subject.vm`, `short.vm`, `full.vm`.
2. The event type string must match what's used when creating the `Event` object in the event handler.
3. No Java code changes needed — `TextTemplateFormatter` resolves by event type dynamically.

## `NotificatorManager`

`getNotificator(type)` maps type strings (`"mail"`, `"sms"`, `"firebase"`, `"telegram"`, `"pushover"`, `"web"`, `"traccar"`, `"whatsapp"`, `"command"`) to singleton implementations in `notificators/`. Unknown types throw (logged as WARN by the caller).
