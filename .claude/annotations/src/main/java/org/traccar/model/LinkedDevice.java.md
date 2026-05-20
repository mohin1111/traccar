# LinkedDevice.java

**Role:** Permission proxy class. Identical to `Device` (same `@StorageName("tc_devices")`) but used as the **property type** in `Permission` to route to `tc_user_device`. Parallel to `ManagedUser` for the user→device link table.
**Fits in:** Used exclusively by `Permission.getStorageName()`. `PermissionsResource` creates `Permission(User, userId, LinkedDevice, deviceId)` rows to grant a user access to a device.
**Read next:** [[Device.java]] (parent, identical), [[Permission.java]] (table-name derivation), [[ManagedUser.java]] (parallel pattern for users)

## Public API

No additional fields or methods. Empty class body.

- `@StorageName("tc_devices")` — same table as `Device`.
- Class name `LinkedDevice` → `Permission.getStorageName()` strips `"Linked"` prefix → property segment becomes `"device"` → table = `"tc_user_device"`.

## Key flows

### How `tc_user_device` gets its name
```
Permission.getStorageName(User.class, LinkedDevice.class)
→ ownerName = "User"           → "user"
→ propertyName = "LinkedDevice" → strip "Linked" → "Device" → "device"
→ returns "tc_user_device"
```

## Gotchas / non-obvious

- This class was added in copyright year 2026 — it is newer than most model classes. Earlier code may have handled user-device links differently.
- Do not confuse with `Device`: when you get a `LinkedDevice` back from a query it IS a `Device` with all device fields populated; the subclass distinction is purely for permission routing.

## Line index

- 20 — `@StorageName("tc_devices")`
- 21 — `public class LinkedDevice extends Device {}` (empty body)
