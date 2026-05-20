# `api/security/` — Authentication filter chain and permission enforcement

9 classes implementing Traccar's auth model: 5+ authentication methods, a request-scoped permission service, OIDC provider support, and supporting value objects.

## File index

| File | One-liner | Annotation |
|---|---|---|
| `SecurityRequestFilter.java` | ContainerRequestFilter; the auth gate for every JAX-RS request | [SecurityRequestFilter.java.md](SecurityRequestFilter.java.md) |
| `LoginService.java` | Credential verification: Bearer/Basic/LDAP/OIDC upsert/TOTP | [LoginService.java.md](LoginService.java.md) |
| `PermissionsService.java` | Permission-graph enforcement; request-scoped; caches User + Server | [PermissionsService.java.md](PermissionsService.java.md) |
| `OidcSessionManager.java` | Traccar-as-OIDC-provider: code flow, PKCE, ES256 ID tokens, JWKS | [OidcSessionManager.java.md](OidcSessionManager.java.md) |
| `LoginResult.java` | Value object: User + optional token expiration | [LoginResult.java.md](LoginResult.java.md) |
| `CodeRequiredException.java` | Marker exception for TOTP code missing | [CodeRequiredException.java.md](CodeRequiredException.java.md) |
| `ServiceAccountUser.java` | Synthetic admin User for machine-to-machine token auth | [ServiceAccountUser.java.md](ServiceAccountUser.java.md) |
| `UserPrincipal.java` | JAX-RS Principal carrying userId + expiration | [UserPrincipal.java.md](UserPrincipal.java.md) |
| `UserSecurityContext.java` | JAX-RS SecurityContext wrapping UserPrincipal | [UserSecurityContext.java.md](UserSecurityContext.java.md) |

## Auth filter chain (per request)

```
HTTP request
  ↓
SecurityRequestFilter.filter()
  ├── OPTIONS → pass through (CORS preflight)
  ├── Authorization header present?
  │   ├── "Bearer <token>" → LoginService.login(token)
  │   │     ├── serviceAccountToken match → ServiceAccountUser
  │   │     └── TokenManager.verifyToken → DB User lookup
  │   └── "Basic <b64>" → LoginService.login(email, password, null)
  │         ├── LDAP (if configured + user.login set)
  │         └── bcrypt password check
  └── No header → session cookie path
        ├── SessionHelper.isSessionOriginValid (CSRF check)
        ├── Read userId from HttpSession
        ├── Check token expiration
        └── PermissionsService.getUser(userId) → checkDisabled()
  ↓
securityContext set → UserPrincipal(userId, expiration) in UserSecurityContext
  OR
securityContext null → method @PermitAll? → pass | → 401 Unauthorized
```

## The 5 authentication methods

| Method | When active | Config keys |
|---|---|---|
| **Session cookie** | Browser with existing session (POST /api/session) | Always available |
| **Bearer token** | `Authorization: Bearer <token>` header | Always; `web.serviceAccountToken` for machine auth |
| **Basic auth** | `Authorization: Basic <b64>` header | Always |
| **LDAP** | User has `login` field OR `ldap.force=true` | `ldap.url`, `ldap.force` |
| **OIDC (consumer)** | Via OpenIdProvider + callback | `openid.clientId`, `openid.clientSecret`, etc. |
| **TOTP 2FA** | User has `totpKey` set | `web.totpEnable`, `web.totpForce` |

## Permission graph enforcement

`PermissionsService` is `@RequestScoped` — one instance per JAX-RS request:

```
checkPermission(clazz, userId, objectId)
  ├── user.administrator → pass
  ├── clazz == User && userId == objectId → pass (self-access)
  └── storage.getObject(clazz,
          Condition.And(
            Equals("id", objectId),
            Permission(User, userId, clazz OR ManagedUser if User)))
       → null → SecurityException("X access denied")
```

Three effective roles (no RBAC columns):
1. `user.administrator = true` → full access everywhere
2. Manager: `user.userLimit != 0` + rows in `tc_user_user`
3. Regular user: access only via explicit `tc_user_*` rows

## Token lifecycle

```
POST /api/session/token
  → TokenManager.generateToken(userId, expiration)
  → CryptoManager.sign(JSON{i,u,e})
  → Base64url encode → opaque string

Authorization: Bearer <token>
  → Base64url decode
  → CryptoManager.verify (ECDSA check)
  → JSON deserialize → check expiration
  → storage.getObject(RevokedToken, id) → check not revoked
  → LoginResult(User, expiration)

POST /api/session/token/revoke
  → TokenManager.decodeToken (no verify) → extract id
  → storage.addObject(RevokedToken{id})
```

## Dependency graph (intra-package)

```
SecurityRequestFilter
  └── LoginService
        ├── TokenManager    (Bearer)
        ├── LdapProvider    (LDAP)
        └── GoogleAuthenticator (TOTP)
  └── PermissionsService
        └── Storage (DB queries)

OidcSessionManager
  ├── CryptoManager  (signing key)
  └── TokenManager   (access token generation)
```

## How to add a new auth method

1. Add a new case in `LoginService.login(scheme, credentials)` for a new scheme string.
2. Implement credential verification; return `LoginResult(user)` on success, throw `SecurityException` on failure.
3. Ensure `SecurityRequestFilter` passes the Authorization header to `LoginService` (it already does for any `Authorization` header via the `login(scheme, credentials)` dispatch).
