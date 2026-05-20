# LookupContext.java

**Role:** Marker interface with three nested record-like classes — `Global`, `User(userId)`, `Device(deviceId)` — used to indicate which entity's attribute inheritance chain to walk when resolving a config key.
**Fits in:** Passed into the attribute lookup helpers in `handler/` (e.g., `CopyAttributesHandler`, computed attributes). Tells the resolver whether to look in device → group → server hierarchy.
**Read next:** [[Config.java]], [[KeyType.java]]

## Public API

```java
interface LookupContext {
    class Global implements LookupContext {}           // server-level only
    class User implements LookupContext {
        long getUserId();
    }
    class Device implements LookupContext {
        long getDeviceId();
    }
}
```

## Key flows

When a handler needs a config value that can be overridden at the device level, it constructs `new LookupContext.Device(deviceId)` and passes it to `AttributeUtil.lookup(key, context)` (in the `helper/model` subpackage). The lookup chain is: device attribute → device's group attribute → server attribute → global config.

## Gotchas / non-obvious

- `Global` is a plain marker; it holds no data. Used when there is no user or device context (e.g., background tasks).
- These are plain inner classes, not Java records (written pre-records style with manual getters).

## Line index

- 18 — interface declaration
- 20 — `class Global`
- 22-33 — `class User` with `userId` field + getter
- 35-46 — `class Device` with `deviceId` field + getter
