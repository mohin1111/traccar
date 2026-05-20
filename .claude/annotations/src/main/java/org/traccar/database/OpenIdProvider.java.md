# OpenIdProvider.java

**Role:** OIDC/OAuth2 SSO implementation using the Nimbus OAuth2 SDK. Builds the authorization redirect URI and handles the callback to exchange the code for tokens, fetch user info, and upsert the Traccar user.
**Fits in:** Constructed in `MainModule` if `Keys.OPENID_CLIENT_ID` is set. Called from `api/resource/OidcResource.java`.
**Read next:** [[LdapProvider.java]] (alternative auth provider), [[LoginService.java]] (creates/finds user on login)

## Public API

- `createAuthUri()` (line 110) — builds the OAuth2 authorization redirect URL with `openid profile email` scope (+ groups scope if admin group configured). Returns a `URI` to redirect the browser to.
- `handleCallback(String queryParameters, HttpServletRequest request)` (line 153) — processes the OIDC callback: validates auth code, exchanges for tokens, fetches user info, evaluates group membership, calls `loginService.login()`, sets session cookie, returns redirect URI (`baseUrl?openid=success`).
- `getForce()` (line 192) — returns whether OIDC login is forced (disables password login).

## Key flows

### Group-based access control (lines 176-182)
If `OPENID_ADMIN_GROUP` is configured: user gets `administrator=true` if member.
If `OPENID_ALLOW_GROUP` is configured: user must be in the allow group OR be admin, else throws `GeneralSecurityException`.
If neither group config is set: all authenticated users are allowed.

### Auto-provision (line 184)
`loginService.login(email, name, administrator)` creates a new Traccar user if one doesn't exist for that email, or returns the existing user with updated name/admin flag.

## Gotchas / non-obvious

- **Issuer URL auto-discovery**: if `OPENID_ISSUER_URL` is set, the provider's metadata (auth/token/userinfo endpoints) is fetched automatically via `OIDCProviderMetadata.resolve()`. Otherwise all three URLs must be explicit.
- **Groups claim name** is configurable (`OPENID_GROUPS_CLAIM_NAME`) — default is likely `"groups"` but varies by provider (Keycloak uses `"groups"`, Okta uses `"groups"`, Google Workspace uses custom).
- **`redirect_uri` override**: the callback accepts a `redirect_uri` request param for flexibility with proxied deployments.
- **Session cookie is set by `SessionHelper.userLogin`** (line 187) — same path as password login, so the session management is unified.
- **Constructor can throw** `IOException`, `URISyntaxException`, `GeneralException` — Guice will propagate these as startup failures.

## Line index

- 79-108 — `@Inject` constructor: reads all OIDC config keys, optionally resolves metadata
- 110-120 — `createAuthUri` (builds authorization redirect)
- 122-135 — `getToken` (exchanges auth code for token)
- 137-151 — `getUserInfo` (fetches user claims from userinfo endpoint)
- 153-190 — `handleCallback` (callback processing + user upsert + session setup)
- 192-194 — `getForce`
