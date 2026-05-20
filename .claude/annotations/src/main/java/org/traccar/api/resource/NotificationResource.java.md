# NotificationResource.java

**Role:** REST resource at `/api/notifications`. CRUD for notification rules plus utility endpoints: list event types, list notificators, test notification delivery, send announcement messages.
**Fits in:** Entity: `Notification`. Extends ExtendedObjectResource.
**Read next:** [[ExtendedObjectResource.java]], `NotificatorManager.java`, `Notification.java` model

## Public API

| Path | Purpose |
|---|---|
| `GET /api/notifications` | List rules (base) |
| CRUD `/api/notifications/{id}` | Base class |
| `GET /api/notifications/types` | Enumerate `Event.TYPE_*` constants |
| `GET /api/notifications/notificators?announcement=` | List delivery channels |
| `POST /api/notifications/test` | Send test via all channels |
| `POST /api/notifications/test/{notificator}` | Send test via specific channel |
| `POST /api/notifications/send/{notificator}?userId=` | Send announcement to users |

## Key flows

`send` endpoint (lines 113-145): manager+ only; broadcasts a `NotificationMessage` to managed users (or all if admin). Skips `temporary` users (share links).

Event types enumerated by reflection over `Event.class.getDeclaredFields()` filtering `TYPE_*` static fields.

## Gotchas / non-obvious

- `?announcement=true` on notificators excludes `command` and `web` channels (not suitable for bulk announcements).
- `send` uses `notificatorManager.getNotificator(notificator).send(user, message, null, null)` — first arg null = no session context.

## Line index

- 56 — extends `ExtendedObjectResource<Notification>`
- 67-81 — event types by reflection
- 83-91 — notificators list
- 93-109 — test endpoints
- 113-145 — send announcement
