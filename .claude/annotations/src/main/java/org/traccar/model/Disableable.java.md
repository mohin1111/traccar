# Disableable.java

**Role:** Interface for entities that can be administratively disabled or set to expire. Implemented by `User` and `Device`. Provides a `checkDisabled()` default method that throws `SecurityException` when disabled or expired.
**Fits in:** Mixed into `User` and `Device`. Called during authentication (`LoginService`) and protocol decoder session lookup (`BaseProtocolDecoder.getDeviceSession()`).
**Read next:** [[User.java]], [[Device.java]]

## Public API

- `getDisabled() / setDisabled(boolean)` — explicit disable flag.
- `getExpirationTime() / setExpirationTime(Date)` — optional expiry timestamp.
- `checkDisabled()` (line 30) — default method; throws `SecurityException("ClassName is disabled")` or `SecurityException("ClassName has expired")`. Called before allowing login or device connection.

## Gotchas / non-obvious

- `expirationTime == null` means "never expires" — `checkDisabled` only checks if non-null.
- The `SecurityException` thrown here is `java.lang.SecurityException`, not a web-layer exception. Callers must catch and map to HTTP 401/403.

## Line index

- 20 — `public interface Disableable`
- 22-28 — abstract getter/setter declarations
- 30-37 — `checkDisabled()` default method
