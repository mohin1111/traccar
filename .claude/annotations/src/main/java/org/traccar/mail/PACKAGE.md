# `mail/` — SMTP email sending

Three files providing email delivery for notifications and log-report attachments.

## File index

| File | One-liner |
|---|---|
| `MailManager.java` | Interface: `sendMessage(User user, String subject, String body)` |
| `SmtpMailManager.java` | Jakarta Mail SMTP implementation; supports per-user SMTP settings + server fallback |
| `LogMailManager.java` | Subclass of `SmtpMailManager` for sending log files as email attachments (used by `StatusScreen` analog server-side) |

## Dependency graph

```
NotificatorMail (notificators/) → MailManager
LogMailManager → SmtpMailManager → Jakarta Mail SMTP

SmtpMailManager reads:
  Keys.MAIL_SMTP_HOST, PORT, SSL, STARTTLS
  Keys.MAIL_SMTP_USERNAME, PASSWORD, FROM, FROM_NAME
  Per-user attribute overrides: mail.smtp.*
```

## Configuration

Server-wide SMTP in `traccar.xml`:
```xml
<entry key="mail.smtp.host">smtp.example.com</entry>
<entry key="mail.smtp.port">587</entry>
<entry key="mail.smtp.starttls.enable">true</entry>
<entry key="mail.smtp.username">noreply@example.com</entry>
<entry key="mail.smtp.password">secret</entry>
<entry key="mail.smtp.from">noreply@example.com</entry>
```

Per-user override: a user with `mail.smtp.host` attribute set in their profile uses their own SMTP settings (used when managers want to send from their own domain).

## Key behavior

- **Per-user SMTP** — if user has any `mail.smtp.*` attributes, those override server config for that notification. Enables fleet managers to use their own email accounts.
- **HTML body** — `NotificatorMail` passes an HTML body (rendered from Velocity template); `sendMessage` sends as `text/html`.
- **`StatisticsManager.registerMail()`** called on each send.
- **Synchronous send** — mail sending blocks the notification thread. Consider async wrapper for high-volume deployments.
