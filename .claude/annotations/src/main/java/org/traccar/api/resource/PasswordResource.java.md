# PasswordResource.java

**Role:** REST resource at `/api/password` for self-service password reset via email token. Two-step: request reset email, then POST new password with the token.
**Fits in:** Extends BaseResource. Both endpoints are @PermitAll (unauthenticated).
**Read next:** [[BaseResource.java]], [[signature/TokenManager.java]] (token verify), `MailManager.java`, `TextTemplateFormatter.java`

## Public API

- `POST /api/password/reset` — `@PermitAll`; `?email=...`; looks up user, sends password-reset email with tokenized link. Always returns 200 (no user enumeration).
- `POST /api/password/update` — `@PermitAll`; form params `token` + `password`; verifies token, updates `hashedPassword`+`salt`.

## Gotchas / non-obvious

- `reset` returns 200 even if email not found (prevents user enumeration).
- `update` verifies the token via `tokenManager.verifyToken` (checks signature + expiration + revocation).
- The email template `passwordReset` lives in `templates/` and uses Velocity.

## Line index

- 56-69 — `POST /password/reset`
- 74-90 — `POST /password/update`
