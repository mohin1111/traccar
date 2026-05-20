# ConfigSuffix.java

**Role:** Abstract template for per-protocol config keys. A `ConfigSuffix` holds a key suffix (e.g., `".port"`) and generates a full `ConfigKey` when given a protocol prefix (e.g., `"its"` → `"its.port"`).
**Fits in:** Used in [[Keys.java]] for the ~20 `PROTOCOL_*` suffix constants. Called as `Keys.PROTOCOL_PORT.withPrefix("its")` by protocol modules.
**Read next:** [[Keys.java]], [[PortConfigSuffix.java]], [[ConfigKey.java]]

## Public API

### `ConfigSuffix<T>` (abstract, lines 20-34)
- `withPrefix(String prefix)` → `ConfigKey<T>` — abstract factory method. Concrete subclasses concatenate `prefix + keySuffix`.

### Package-private concrete subclasses (lines 36-99)
- `StringConfigSuffix`, `BooleanConfigSuffix`, `IntegerConfigSuffix`, `LongConfigSuffix`, `DoubleConfigSuffix` — each delegates to the matching `*ConfigKey`.

## Key flows

`Keys.PROTOCOL_PORT.withPrefix("its")` → `new IntegerConfigKey("its.port", ..., PORTS.get("its"))` (port 5179 for the ITS protocol).

## Gotchas / non-obvious

- Each call to `withPrefix()` creates a **new `ConfigKey` object** — there is no caching. Cache the result if calling inside a hot path.
- [[PortConfigSuffix.java]] overrides the default value from a hardcoded port map — the only subclass with non-null defaults.

## Line index

- 20-34 — abstract `ConfigSuffix<T>` (three fields + abstract method)
- 36-47 — `StringConfigSuffix`
- 49-60 — `BooleanConfigSuffix`
- 62-73 — `IntegerConfigSuffix`
- 75-86 — `LongConfigSuffix`
- 88-99 — `DoubleConfigSuffix`
