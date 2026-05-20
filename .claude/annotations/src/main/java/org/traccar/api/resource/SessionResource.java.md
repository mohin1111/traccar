# SessionResource.java

**Role:** The login/logout/session management endpoint at `GET/POST/DELETE /api/session`. Handles form-based login, token-based session setup, OIDC redirect, Bearer token generation/revocation. The most-called resource in any browser session.
**Fits in:** Extends [[BaseResource.java]] directly (not the CRUD hierarchy). Entity: `User` (returns the authenticated user on login).
**Read next:** [[security/LoginService.java]] (all credential checks), [[signature/TokenManager.java]] (token generation/revocation), `OpenIdProvider.java` (OIDC consumer-side redirect), `SessionHelper.java`

## Public API (endpoints)

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `GET` | `/api/session` | `@PermitAll` | Restore session from cookie or `?token=`; returns `User` |
| `GET` | `/api/session/{id}` | Required | Impersonate / switch to managed user |
| `POST` | `/api/session` | `@PermitAll` | Login with `email`+`password`+optional `code` (TOTP); returns `User` |
| `DELETE` | `/api/session` | Required | Logout (remove session attribute) |
| `POST` | `/api/session/token` | Required | Generate a Bearer token (with optional `expiration` cap) |
| `POST` | `/api/session/token/revoke` | Required | Add token to `tc_revoked_tokens` |
| `GET` | `/api/session/openid/auth` | `@PermitAll` | Redirect to OIDC provider |
| `GET` | `/api/session/openid/callback` | `@PermitAll` | Handle OIDC callback; exchange code for session |

## Key flows

### Form login (`POST /api/session`, lines 112-136)
1. `loginService.login(email, password, code)` — catches `CodeRequiredException` → 401 + `WWW-Authenticate: TOTP`.
2. Null result (wrong credentials) → log failed attempt → 401.
3. Success → `SessionHelper.userLogin(...)` sets `userId` in HTTP session + audit.
4. Returns `User` JSON.

### Token login (`GET /api/session?token=...`, lines 79-99)
1. `loginService.login(token)` → calls `SessionHelper.userLogin` → sets session.
2. Falls back to checking existing session cookie.
3. 404 if neither path succeeds (triggers re-login on the client).

### Token generation (`POST /api/session/token`, lines 147-157)
- Caps expiration: if session has its own expiration (from the session token) and the requested expiration extends beyond it, uses the session expiration. Prevents token escalation.

### Token revocation (`POST /api/session/token/revoke`, lines 159-168)
- Decodes (does not verify!) the token to extract `id`, inserts into `tc_revoked_tokens`. Works even for expired tokens.

## Gotchas / non-obvious

- **`@PermitAll` on `POST /api/session`** — login must be reachable without authentication.
- **`GET /api/session/{id}`** — managers can switch to a managed user's perspective by hitting this endpoint. It creates a new session as the target user.
- **`@Consumes(APPLICATION_FORM_URLENCODED)`** on the class — login is form-encoded, not JSON. Consistent with standard web login forms.
- `failedLogin` audit log entry (line 133) — important for security monitoring.

## Line index

- 79-99 — `GET /api/session` (token + cookie restore)
- 103-110 — `GET /api/session/{id}` (impersonate)
- 112-136 — `POST /api/session` (form login)
- 139-143 — `DELETE /api/session` (logout)
- 147-157 — `POST /api/session/token` (generate Bearer)
- 159-168 — `POST /api/session/token/revoke`
- 170-178 — OIDC redirect
- 181-188 — OIDC callback
