# RevokedToken.java

**Role:** Token revocation blacklist entry. `@StorageName("tc_revoked_tokens")` → `tc_revoked_tokens`. Stores the `id` (which IS the token identifier) of a revoked bearer token. `SecurityRequestFilter` checks this table on every authenticated request.
**Fits in:** Extends `BaseModel` only (just `id`). `TokenManager` writes a row here when a user logs out or an admin revokes a token.
**Read next:** [[BaseModel.java]] (parent)

## Public API

- Only `id long` (inherited from `BaseModel`) — the token's numeric identifier.

## Line index

- 20 — `@StorageName("tc_revoked_tokens")`
- 21 — `public class RevokedToken extends BaseModel {}` (empty body)
