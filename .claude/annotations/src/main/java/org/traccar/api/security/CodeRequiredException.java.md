# CodeRequiredException.java

**Role:** Marker exception thrown by `LoginService.checkUserCode()` when a user has TOTP enabled but no `code` was provided. Caught specifically by `SessionResource.add()` to return a `401 WWW-Authenticate: TOTP` response (instead of a generic failure).
**Fits in:** Thrown in [[LoginService.java]] line 163; caught in `SessionResource.java` lines 119-126.
**Read next:** [[LoginService.java]] (`checkUserCode`), `SessionResource.java` (catches and converts to HTTP 401+TOTP header)

## Gotchas / non-obvious

- Extends `SecurityException` — if not caught specifically, falls through to the generic security handler and becomes a warning log + 400/401.
- The `WWW-Authenticate: TOTP` header is a non-standard Traccar convention; the web UI checks for it and shows the TOTP input field.
