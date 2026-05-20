# `config/` — Configuration loading and typed key catalog

Provides the runtime configuration layer: XML file loading, environment variable override, and the complete catalog of typed config keys. Nothing in this package touches the database or network.

## File index

| File | One-liner | Annotation |
|---|---|---|
| `Config.java` | Singleton XML config loader with env-var override and typed getters | [Config.java.md](Config.java.md) |
| `Keys.java` | Static catalog of every config key (1400+ constants) — the config schema | [Keys.java.md](Keys.java.md) |
| `ConfigKey.java` | Abstract typed key base + five concrete subclasses (String/Boolean/Integer/Long/Double) | [ConfigKey.java.md](ConfigKey.java.md) |
| `ConfigSuffix.java` | Per-protocol key template — generates `ConfigKey` from `withPrefix("protocol")` | [ConfigSuffix.java.md](ConfigSuffix.java.md) |
| `KeyType.java` | Enum: `CONFIG` / `SERVER` / `USER` / `DEVICE` scope flags | [KeyType.java.md](KeyType.java.md) |
| `LookupContext.java` | Marker interface + nested `Global`/`User`/`Device` classes for attribute chain resolution | [LookupContext.java.md](LookupContext.java.md) |
| `PortConfigSuffix.java` | `ConfigSuffix<Integer>` with 265-entry default port map for all protocols | [PortConfigSuffix.java.md](PortConfigSuffix.java.md) |

## Dependency graph

```
Keys.java
 ├── ConfigKey.java  (5 concrete subclasses)
 ├── ConfigSuffix.java  (5 concrete subclasses)
 │    └── PortConfigSuffix.java  (overrides port defaults)
 └── KeyType.java  (enum used as Set<KeyType> in ConfigKey)

Config.java  (reads Keys.* constants)
 └── Log.java  (helper, sets up logger)

LookupContext.java  (used by handler/AttributeUtil, not by Config directly)
```

## Key design: scoped config

Every `ConfigKey` declares which `KeyType` scopes it supports:
- `CONFIG` → read from `traccar.xml` only
- `SERVER` → also overridable via `tc_server.attributes` JSON blob
- `USER` / `DEVICE` → overridable per entity in the DB

The handler layer uses `LookupContext` to walk the hierarchy: device attrs → group attrs → server attrs → global config. This means e.g. `Keys.GEOCODER_ON_REQUEST` can be overridden per-device without changing `traccar.xml`.

## Protocol port conventions

All 265 protocol default ports live in `PortConfigSuffix.PORTS`:
- `osmand=5055` (Traccar client HTTP)
- `its=5179` (AIS-140 / ITS)
- `jt808=5015` (Chinese fleet standard, common in India)
- `teltonika=5027` (global popular hardware)
- `aquila=5089`, `idpl=5110`, `racedynamics=5195` (India-specific)

Ports 5001–5265 are the Traccar port range. Avoid binding other services in this range.

## How to add a new config key

1. Decide the name (dot-notation, e.g., `transport.myFeatureEnabled`).
2. Pick scope(s): `KeyType.CONFIG` for file-only, add `KeyType.DEVICE` if per-device override is needed.
3. Add a `public static final ConfigKey<T> MY_KEY = new BooleanConfigKey("transport.myFeatureEnabled", List.of(KeyType.CONFIG), false);` line in `Keys.java` near related keys.
4. Read it: `config.getBoolean(Keys.MY_KEY)`.
5. Document the key with a Javadoc comment above the constant (all existing keys have this).

## How to add a new protocol port

Add `PORTS.put("myprotocol", 5XXX)` in `PortConfigSuffix` static initializer. Pick the next available port after 5265. The protocol auto-discovers it via `Keys.PROTOCOL_PORT.withPrefix("myprotocol")`.
