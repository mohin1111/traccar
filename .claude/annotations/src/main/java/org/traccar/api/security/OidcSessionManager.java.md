# OidcSessionManager.java

**Role:** Implements Traccar as an OAuth2/OIDC **provider** (not just a consumer). Issues authorization codes, validates code exchanges (PKCE), generates signed ID tokens (ES256 JWT). Used by [[resource/OidcResource.java]] to allow third-party apps to authenticate against Traccar.
**Fits in:** Singleton. Called exclusively by `OidcResource.java`. Uses [[signature/CryptoManager.java]] for the EC key pair.
**Read next:** [[resource/OidcResource.java]] (the HTTP endpoints), [[signature/CryptoManager.java]] (EC key pair), [[signature/TokenManager.java]] (the access token issued alongside the ID token)

## Public API

- `AuthorizationCode` record (line 61) — carries all data for the authorization code grant.
- `issueCode(...)` (line 86) — generates a random 32-byte code, stores in `ConcurrentHashMap` with 5-minute TTL.
- `consumeCode(code, clientId, redirectUri, codeVerifier)` (line 104) — removes and validates the code; verifies PKCE challenge if present.
- `generateIdToken(authCode, clientId, tokenData, scopes, user)` (line 126) — builds JWT claims (iss, sub, aud, exp, iat, nonce, email, name), signs with ES256.
- `parseScopes(scope)` (line 162) — splits scope string.
- `getJwks()` (line 168) — returns the public JWK for clients to verify ID tokens.

## Key flows

### Authorization code flow
1. Client → `GET /api/oidc/authorize?client_id=...&redirect_uri=...&scope=...&code_challenge=...` (user must be authenticated).
2. `issueCode` → generates code, stores `AuthorizationCode` record in-memory map.
3. Redirect to `redirect_uri?code=<code>`.
4. Client → `POST /api/oidc/token` with code + `code_verifier`.
5. `consumeCode` validates, removes. `generateIdToken` creates JWT. `TokenManager.generateToken` creates access token.
6. Returns `{access_token, token_type, expires_in, id_token, scope}`.

### PKCE support (lines 173-193)
Both `plain` and `S256` methods supported. If `codeChallenge` present, `codeVerifier` is required on token exchange.

### EC signing key (lines 195-214)
Lazy-loaded from `CryptoManager.getKeyPair()` (secp256r1). Converted to a `nimbus-jose-jwt` `ECKey` with `keyIDFromThumbprint()`. Singleton within the JVM (volatile double-checked locking).

## Gotchas / non-obvious

- **In-memory code store** — authorization codes live only in the JVM heap (`ConcurrentHashMap`). Multi-node deployments will fail if the `authorize` and `token` calls hit different nodes.
- **Clients configured in `Keys.OPENID_CLIENTS`** — format `"clientId:secret:redirect1|redirect2,..."`. No dynamic client registration.
- **Codes expire in 5 minutes** (`DEFAULT_LIFETIME`). Expired codes are checked on consume but not actively evicted — memory leak for high-volume flows.

## Line index

- 61-69 — `AuthorizationCode` record
- 78-79 — in-memory code map
- 86-102 — `issueCode`
- 104-124 — `consumeCode` (validation + PKCE)
- 126-160 — `generateIdToken` (JWT construction + signing)
- 162-165 — `parseScopes`
- 168-171 — `getJwks`
- 173-193 — `verifyCodeChallenge` (PKCE plain + S256)
- 195-214 — `getSigningKey` lazy init
