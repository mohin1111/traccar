# Driver.java

**Role:** Represents a driver identified by RFID/iButton token or manual assignment. `@StorageName("tc_drivers")` → `tc_drivers`. `DriverHandler` in the position pipeline matches `position.attributes.driverUniqueId` against `Driver.uniqueId` to link a driver to a position.
**Fits in:** Extends `ExtendedModel` (has attributes). Linked to users/groups/devices via permission link tables. Keyed by `uniqueId` not `id` in the Redux web store (`drivers.js` uses `uniqueId` as map key).
**Read next:** [[ExtendedModel.java]] (parent), [[Position.java]] (KEY_DRIVER_UNIQUE_ID attribute)

## Public API

### DB-mapped fields (`tc_drivers` columns)
- Inherited `attributes AttributeMap` — custom driver attributes (license number, contact, etc.).
- `name String` (line 23) — driver display name.
- `uniqueId String` (line 33) — RFID/iButton token or other identifier; trimmed on set; matched against position's `driverUniqueId` attribute.

## Gotchas / non-obvious

- `uniqueId` here is the RFID token, NOT a device IMEI — same field name, different semantic from `Device.uniqueId`.
- `DriverHandler` does a case-sensitive match on `uniqueId`. Token encoding varies by device (some hex, some decimal).

## Line index

- 21 — `@StorageName("tc_drivers")`
- 22 — `class Driver extends ExtendedModel`
- 23-31 — name
- 33-41 — uniqueId (RFID token; trimmed)
