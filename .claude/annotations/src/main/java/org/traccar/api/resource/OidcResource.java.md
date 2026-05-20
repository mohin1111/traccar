# OidcResource.java

**Role:** REST resource at `/api/oidc` implementing Traccar as an OIDC identity provider. Provides the authorization endpoint, token endpoint, userinfo, and JWKS for third-party apps to authenticate users against Traccar.
**Fits in:** Extends BaseResource. Uses OidcSessionManager for code flow and TokenManager for access tokens.
**Read next:** [[BaseResource.java]], [[security/OidcSessionManager.java]], [[signature/TokenManager.java]]

## Public API

| Path | Auth | Purpose |
|---|---|---|
| `GET /api/oidc/authorize` | `@PermitAll` (but user must be authenticated via session) | Start authorization code flow |
| `POST /api/oidc/token` | `@PermitAll` | Exchange code for tokens |
| `GET /api/oidc/userinfo` | Required | Standard OIDC userinfo |
| `GET /api/oidc/jwks` | `@PermitAll` | Public JWK set |

## Key flows

`authorize`: validates client_id against `Keys.OPENID_CLIENTS` config; validates redirect_uri; calls `sessionManager.issueCode`; redirects.

`token`: parses Basic auth header for client credentials; calls `sessionManager.consumeCode` (PKCE validation); issues access token via `TokenManager.generateToken`; generates ID token (ES256 JWT) via `sessionManager.generateIdToken`; returns standard OAuth2 token response.

Clients configured in `Keys.OPENID_CLIENTS` as `"clientId:secret:redirect1|redirect2,..."`.

## Gotchas / non-obvious

- `authorize` is @PermitAll but requires an active session (getUserId() = 0 if unauthenticated → code issued with userId=0, which would fail the token exchange).
- Token response includes both `access_token` (Traccar Bearer token) and `id_token` (ES256 JWT).

## Line index

- 57 — class extends `BaseResource`
- 72-102 — `authorize` endpoint
- 104-149 — `token` endpoint
- 152-160 — `userinfo`
- 162-165 — `jwks`
- 169-182 — `getClients()` parser
