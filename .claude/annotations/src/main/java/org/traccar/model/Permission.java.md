# Permission.java

**Role:** Generic model for permission link-table rows. A `Permission` is a pair `{ownerClass → ownerId, propertyClass → propertyId}`. It does **not** extend `BaseModel`. The DB table name is **derived dynamically** from the two class names at runtime.
**Fits in:** `PermissionsResource` accepts JSON `{"userId": 1, "deviceId": 5}` → constructs a `Permission` → calls `DatabaseStorage.addPermission()` which uses `getStorageName()` to target the right join table.
**Read next:** [[ManagedUser.java]], [[LinkedDevice.java]] (permission proxy classes that change the derived table name), [[BaseModel.java]] (used as type bound), [[User.java]], [[Device.java]]

## Public API

### Fields (all `@QueryIgnore @JsonIgnore` except the data map)
- `data LinkedHashMap<String, Long>` (line 47) — insertion-ordered; entry 0 = owner, entry 1 = property.
- `ownerClass / ownerId` (lines 49-50) — parsed from `data` entry 0.
- `propertyClass / propertyId` (lines 51-52) — parsed from `data` entry 1.

### Key static methods
- `getKeyClass(String key)` (line 77) — strips `"Id"` suffix from JSON key, looks up in static `CLASSES` map. E.g., `"userId"` → `User.class`.
- `getKey(Class<?>)` (line 81) — `decapitalize(simpleName) + "Id"`. E.g., `Device.class` → `"deviceId"`.
- `getStorageName(Class<?> ownerClass, Class<?> propertyClass)` (line 85) — **THE critical method**. Builds `"tc_" + decapitalize(ownerSimpleName) + "_" + decapitalize(propertySimpleName)`. Strips `"Managed"` or `"Linked"` prefix from property class name before decapitalization.

### Instance methods
- `getStorageName()` (line 99) — instance version; delegates to static.
- `get()` (`@JsonAnyGetter`) (line 104) — exposes `data` map as flat JSON properties.
- `set(key, value)` (`@JsonAnySetter`) (line 110) — receives arbitrary JSON key-value pairs; builds `data`.

## Key flows

### Table name derivation (line 85-94)
```
ownerClass = User.class        → "user"
propertyClass = Device.class   → "device"
table = "tc_user_device"

ownerClass = User.class        → "user"
propertyClass = ManagedUser    → "Managed" prefix stripped → "user"
table = "tc_user_user"

ownerClass = User.class        → "user"
propertyClass = LinkedDevice   → "Linked" prefix stripped → "device"
table = "tc_user_device"        ← same table, different proxy class
```

### JSON round-trip
Request body `{"userId": 1, "deviceId": 5}` → `@JsonAnySetter set("userId", 1)` then `set("deviceId", 5)` → `Permission(data)` constructor deduces `ownerClass=User`, `propertyClass=Device` → `getStorageName() = "tc_user_device"`.

### Static CLASSES map (lines 35-45)
Built once at class load via `ClassScanner.findSubclasses(BaseModel.class)` — maps every `BaseModel` subclass simple name to its `Class<?>`. Case-insensitive lookup. Used by `getKeyClass`.

## Gotchas / non-obvious

- `Permission` is NOT a `BaseModel` — has no `id`. Link tables have composite PKs.
- The order in `data` (LinkedHashMap insertion order) determines which entry is "owner" and which is "property". `PermissionsResource` always puts user-side first.
- `ManagedUser` and `LinkedDevice` exist **solely** to route to the correct table name: they share the same underlying DB table (`tc_users` and `tc_devices`), but `getStorageName()` produces different join-table names (`tc_user_user` vs `tc_user_device`).

## Line index

- 35-45 — static CLASSES map (class discovery on load)
- 47-52 — data + parsed owner/property fields
- 54-62 — constructor from LinkedHashMap (parse owner/property)
- 65-75 — constructor from explicit classes
- 77-79 — getKeyClass (strip "Id" suffix)
- 81-83 — getKey (add "Id" suffix)
- 85-95 — getStorageName static (table name derivation, strips Managed/Linked prefix)
- 99-101 — getStorageName instance
- 104-112 — JsonAnyGetter/JsonAnySetter for flat JSON
