# LoginService.java

**Role:** Central credential-verification service. Handles all five authentication methods: Bearer token, Basic (email:password), LDAP, OIDC user lookup, and TOTP 2FA validation. Returns a `LoginResult` (User + optional token expiration) on success.
**Fits in:** Called by [[SecurityRequestFilter.java]] (Authorization header), [[AsyncSocketServlet.java]] (WS token upgrade), and `SessionResource.java` (POST /api/session login). Singleton.
**Read next:** [[SecurityRequestFilter.java]] (caller for header auth), `SessionResource.java` (caller for form-based login), [[signature/TokenManager.java]] (Bearer token verification), `LdapProvider.java` (LDAP backend), [[LoginResult.java]] (return type)

## Public API

- `login(scheme, credentials)` (line 65) — dispatches `"bearer"` → `login(token)`, `"basic"` → decodes and calls `login(email, password, null)`.
- `login(token)` (line 79) — service-account bypass; else `TokenManager.verifyToken` + DB user lookup.
- `login(email, password, code)` (line 92) — main email+password path with LDAP fallback and TOTP check.
- `login(email, name, administrator)` (line 121) — OIDC-specific: upsert a user by email (creates if new, updates name/admin flag if changed).

## Key flows

### Bearer token login (lines 79-90)
1. If token equals `serviceAccountToken` config value → return `ServiceAccountUser` (synthetic admin, no DB lookup).
2. `TokenManager.verifyToken(token)` → verify ECDSA signature + expiration + revocation check.
3. Load `User` from DB by `tokenData.getUserId()`.
4. `checkUserEnabled(user)` — throws if null or disabled.

### Email+password login (lines 92-119)
1. Return null immediately if `forceOpenId=true` (OIDC-only mode).
2. Normalize email to lowercase; query DB with `LOWER(email)` OR `LOWER(login)`.
3. If user found: try LDAP auth (if `ldapProvider` configured and user has `login` field) OR check `user.isPasswordValid(password)` (bcrypt).
4. If user found and password valid: `checkUserCode(user, code)` — validates TOTP if user has `totpKey`.
5. If user NOT found but LDAP configured: try `ldapProvider.login(email, password)`; if succeeds, auto-provision new User via `ldapProvider.getUser(email)`.

### TOTP check (lines 161-172)
Throws `CodeRequiredException` (a `SecurityException` subclass) if user has a TOTP key but no code was provided. `SessionResource` catches this and returns 401 with `WWW-Authenticate: TOTP`.

### OIDC upsert (lines 121-153)
Creates user on first OIDC login; updates `name` and `administrator` if changed on subsequent logins. `fixedEmail = true` prevents the user from changing their email (since it's from the IdP).

## Gotchas / non-obvious

- **`forceLdap`** — when true, native password check is skipped entirely (line 105). LDAP becomes the only path.
- **`forceOpenId`** — `login(email, password, code)` returns null (not error) so callers redirect to OIDC. This is how the "OIDC-only" mode works.
- **Case-insensitive email query** — uses `LOWER(email)` and `LOWER(login)` SQL functions for DB-level case folding. The normalization at line 97 is for the Java-side match.
- **`ServiceAccountUser.ID = 9000000000000000000L`** — a sentinel ID that skips owner-link creation in `BaseObjectResource.add()`.

## Line index

- 65-77 — `login(scheme, credentials)` dispatch
- 79-90 — Bearer token path
- 92-119 — email+password+LDAP+TOTP path
- 121-153 — OIDC upsert path
- 154-158 — `checkUserEnabled`
- 161-172 — `checkUserCode` (TOTP)
