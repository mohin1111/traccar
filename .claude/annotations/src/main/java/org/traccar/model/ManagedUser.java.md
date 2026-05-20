# ManagedUser.java

**Role:** Permission proxy class. Identical to `User` (same `@StorageName("tc_users")` table) but used as the **property type** in `Permission` to route to `tc_user_user` instead of `tc_user_device`. This is the "managed sub-user" side of the user-management hierarchy.
**Fits in:** Used exclusively by `Permission.getStorageName()` to distinguish `User→ManagedUser` (→ `tc_user_user`) from `User→Device` (→ `tc_user_device`). `PermissionsService.checkUser()` queries `tc_user_user` to verify a manager can access a sub-user.
**Read next:** [[User.java]] (parent, identical), [[Permission.java]] (table-name derivation), [[LinkedDevice.java]] (parallel pattern for devices)

## Public API

No additional fields or methods. Empty class body.

- `@StorageName("tc_users")` — reads/writes from the same table as `User`.
- Class name `ManagedUser` → `Permission.getStorageName()` strips `"Managed"` prefix → property segment becomes `"user"` → table = `"tc_user_user"`.

## Key flows

### How `tc_user_user` gets its name
```
Permission.getStorageName(User.class, ManagedUser.class)
→ ownerName = "User"           → "user"
→ propertyName = "ManagedUser" → strip "Managed" → "User" → "user"
→ returns "tc_user_user"
```

## Gotchas / non-obvious

- `ManagedUser` carries **no** additional state. It is purely a class identity token. Do not add fields here.
- When `PermissionsResource` receives `{"userId": 1, "managedUserId": 2}`, Jackson maps `"managedUserId"` key → strips `"Id"` → looks up `"managedUser"` in `CLASSES` map → resolves to `ManagedUser.class` → table = `tc_user_user`.

## Line index

- 20 — `@StorageName("tc_users")`
- 21 — `public class ManagedUser extends User {}` (empty body)
