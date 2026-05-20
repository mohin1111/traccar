# KeyType.java

**Role:** Four-value enum that classifies which configuration scope a `ConfigKey` belongs to.
**Fits in:** Stored in each [[ConfigKey.java]] via a `Set<KeyType>`. Checked during server attribute resolution to know which keys are user/device overridable.
**Read next:** [[ConfigKey.java]], [[Keys.java]]

## Public API

```java
public enum KeyType {
    CONFIG,  // traccar.xml file only
    SERVER,  // server-level attribute (tc_server.attributes)
    USER,    // user-level attribute (tc_users.attributes)
    DEVICE,  // device-level attribute (tc_devices.attributes)
}
```

## Key flows

The `handler/` package uses `KeyType` to walk the lookup hierarchy: device attributes → user attributes → server attributes → config file. A key with `List.of(KeyType.CONFIG, KeyType.DEVICE)` can be overridden per-device.

## Gotchas / non-obvious

- `CONFIG` keys are **read-only at runtime** — only changeable in `traccar.xml`. `SERVER`/`USER`/`DEVICE` keys are stored in the JSON `attributes` blob of the matching entity and are editable via API.
- A single key can have **multiple types** — e.g., a boolean that can be set globally in the config file or overridden per-device.

## Line index

- 18-23 — the entire file
