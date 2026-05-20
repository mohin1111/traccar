# LdapProvider.java

**Role:** LDAP authentication provider. Looks up users by login name, binds to verify password, and checks group membership for admin elevation.
**Fits in:** Constructed in `MainModule` if `Keys.LDAP_URL` is set. Called from `api/security/LoginService.login()` as the alternate auth path (before password hash check).
**Read next:** [[OpenIdProvider.java]] (OIDC alternative), [[Config.java]] (reads LDAP_* keys)

## Public API

- `login(String username, String password)` (line 171) — looks up the user's DN, then binds with the provided password. Returns `true` on success.
- `getUser(String accountName)` (line 136) — returns a partially populated `User` object (login, name, email from LDAP attributes; administrator from group membership check). Used to auto-provision the user in the Traccar DB on first LDAP login.

## Key flows

### Lookup flow (lines 112-133)
1. `initContext()` binds a service-account connection using `Keys.LDAP_USER` / `LDAP_PASSWORD`.
2. Searches `searchBase` with `searchFilter` (`:login` replaced by the escaped account name).
3. Returns the first `SearchResult` (warns if multiple results matched).

### Admin check (lines 85-106)
`isAdmin(accountName)` runs a second search with `adminFilter`. If `LDAP_ADMIN_GROUP` is configured, the default filter is `(&(uid=:login)(memberOf=<group>))`.

### LDAP injection prevention (lines 184-204)
`encodeForLdap()` escapes special LDAP filter characters (`\`, `*`, `(`, `)`, `&`, `|`, `=`, null byte) to prevent injection.

## Gotchas / non-obvious

- **Service account required** — `LDAP_USER` / `LDAP_PASSWORD` must be a read-only service account with search permissions. Anonymous bind is not supported.
- **Multiple results = auth failure** — `lookupUser` returns `null` and logs a warning if the search returns more than one result. This prevents account confusion in case of duplicate CNs.
- **No `@Singleton`** — `LdapProvider` is constructed without injection and must be instantiated manually in `MainModule`.
- **`encodeForLdap` is public** — tested directly; reuse it for any user-supplied input in LDAP queries.
- **`mail` attribute key is `LDAP_MAIN_ATTRIBUTE`** (line 53) — a likely typo in the original code (`MAIN` vs. `MAIL`). The actual config key is `ldap.mailAttribute`.

## Line index

- 48-71 — constructor: reads all LDAP config keys, builds search/admin filters
- 73-83 — `auth(accountName, password)` — JNDI bind helper
- 85-106 — `isAdmin` — group membership check via second LDAP search
- 108-110 — `initContext` — service account bind (public for testing)
- 112-133 — `lookupUser` — SUBTREE_SCOPE search returning first result
- 136-169 — `getUser` — builds a `User` from LDAP attributes
- 171-182 — `login` — password verification via bind
- 184-204 — `encodeForLdap` — LDAP filter escape
