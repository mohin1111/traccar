# Server.java

**Role:** Singleton server configuration entity. `@StorageName("tc_servers")` → `tc_servers` (always exactly one row). Returned by `GET /api/server`; provides global defaults for map, theme, coordinate format, and feature flags. Also implements `UserRestrictions` — server-level restrictions cascade to all users.
**Fits in:** Extends `ExtendedModel`. `Server.attributes` carries branding config (`colorPrimary`, `colorSecondary`, `logo`, `title`, `language`, `darkMode`) that the web frontend reads from `state.session.server.attributes`.
**Read next:** [[UserRestrictions.java]] (interface), [[ExtendedModel.java]] (parent), [[User.java]] (user-level overrides of same restrictions)

## Public API

### DB-mapped fields (`tc_servers` columns)
- `registration boolean` (line 27) — allow public user self-registration.
- `readonly boolean` (line 37) — server-wide read-only mode.
- `deviceReadonly boolean` (line 47) — all users cannot edit device config.
- `map String` (line 57) — default map tile provider key.
- `bingKey String` (line 67) — Bing Maps API key.
- `mapUrl String` (line 77) — custom tile URL template.
- `overlayUrl String` (line 87) — map overlay URL.
- `latitude/longitude/zoom` (lines 97-127) — default map center/zoom.
- `forceSettings boolean` (line 129) — server settings override user's saved preferences.
- `coordinateFormat String` (line 139) — `"dd"`, `"ddm"`, `"dms"`.
- `limitCommands boolean` (line 149) — restrict all non-admin users from sending commands.
- `disableReports boolean` (line 159) — disable reports for all users.
- `fixedEmail boolean` (line 169) — users cannot change their email.
- `poiLayer String` (line 179) — global POI layer URL.
- `announcement String` (line 189) — banner text shown on login.
- Inherited `attributes AttributeMap` — branding + extra server config (JSON).

### Not DB-mapped (`@QueryIgnore`)
- `getVersion()` (line 202) — reads JAR manifest implementation version.
- `emailEnabled/textEnabled/geocoderEnabled` (lines 207-242) — runtime-computed; set by `MainModule` after checking config.
- `storageSpace long[]` (line 244) — disk space stats; set at request time.
- `newServer boolean` (line 255) — true when `tc_servers` was empty on startup.
- `openIdEnabled/openIdForce boolean` (lines 267-288) — OIDC config; set at request time.

## Gotchas / non-obvious

- `Server` implements `UserRestrictions` — `PermissionsService` merges server restrictions with user-level restrictions using OR logic (most restrictive wins).
- Branding (`colorPrimary`, `title`, etc.) lives in `attributes` JSON, not in typed columns. Frontend reads these via `server.attributes.colorPrimary`.

## Line index

- 23 — `@StorageName("tc_servers")`
- 25 — `class Server extends ExtendedModel implements UserRestrictions`
- 27-35 — registration
- 202-205 — getVersion (@QueryIgnore)
- 207-242 — runtime flags (emailEnabled, textEnabled, geocoderEnabled)
- 255-264 — newServer
- 267-288 — openIdEnabled, openIdForce
