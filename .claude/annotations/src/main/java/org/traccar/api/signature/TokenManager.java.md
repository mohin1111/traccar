# TokenManager.java

**Role:** Generates and verifies Bearer tokens. A token is a base64url-encoded, ECDSA-signed JSON blob `{"i": id, "u": userId, "e": expiration}`. Revocation checked against `tc_revoked_tokens`. Default lifetime: 7 days.
**Fits in:** Used by [[security/LoginService.java]] (Bearer auth), `SessionResource.java` (POST /api/session/token), `ShareResource.java` (share links), [[security/OidcSessionManager.java]] (OIDC access token).
**Read next:** [[CryptoManager.java]] (signing), [[security/LoginService.java]] (verifies tokens on auth), `SessionResource.java` (generates tokens on request), `RevokedToken.java` model

## Public API

- `generateToken(userId)` (line 75) — 7-day default expiration.
- `generateToken(userId, expiration)` (line 79) — explicit expiration. Builds `TokenData`, signs, base64url-encodes.
- `verifyToken(token)` (line 93) — decodes → checks expiration → checks `tc_revoked_tokens` → returns `TokenData`.
- `decodeToken(token)` (line 106) — like `verifyToken` but skips expiration + revocation checks. Used to extract token ID for revocation.

### `TokenData` inner class (lines 47-66)
- `i` — random token ID (used as revocation key in `tc_revoked_tokens`)
- `u` — userId
- `e` — expiration timestamp

## Key flows

### Token generation (lines 79-91)
1. Build `TokenData` with random `id` (`SecureRandom.nextLong() & Long.MAX_VALUE` — ensures positive).
2. Jackson serialize to bytes.
3. `CryptoManager.sign(bytes)` — ECDSA-sign; prepend signature.
4. Base64url encode → opaque string returned to client.

### Token verification (lines 93-104)
1. Base64url decode.
2. `CryptoManager.verify` — check signature, extract original bytes.
3. Jackson deserialize to `TokenData`.
4. Check `expiration.before(now)` → SecurityException.
5. Query `tc_revoked_tokens` by `id` → SecurityException if found.

## Gotchas / non-obvious

- **Not a JWT** — token format is custom (ECDSA-signed JSON blob). Simpler than JWT but not interoperable with standard JWT libraries.
- **`tc_revoked_tokens` grows unboundedly** unless `TaskExpirations` (schedule/) periodically prunes expired revocations. Check that scheduled task.
- **Thread-safe `SecureRandom`** — `SecureRandom` is instantiated once as a field (line 45) and used for `nextLong()` which is thread-safe.
- Expiration in `generateToken(userId, null)` falls back to 7 days from now (line 86).

## Line index

- 39 — `DEFAULT_EXPIRATION_DAYS = 7`
- 47-66 — `TokenData` inner class
- 75-91 — `generateToken` overloads
- 93-104 — `verifyToken` (expiration + revocation checks)
- 106-109 — `decodeToken` (no checks)
