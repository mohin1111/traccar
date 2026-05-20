# ConfigKey.java

**Role:** Base class + five concrete typed subclasses for config key descriptors. Each key carries its dot-notation name, which `KeyType` scopes it belongs to, its Java type, and an optional default value.
**Fits in:** Used exclusively in [[Keys.java]] (catalog of all keys) and read by [[Config.java]] getters.
**Read next:** [[Keys.java]], [[KeyType.java]], [[ConfigSuffix.java]]

## Public API

### `ConfigKey<T>` (abstract, lines 22-52)
- `getKey()` — returns the dot-notation string key (e.g., `"geocoder.type"`).
- `hasType(KeyType)` — checks whether this key is in the given scope (CONFIG, SERVER, USER, DEVICE).
- `getValueClass()` — returns `Class<T>` (used for reflection / serialization).
- `getDefaultValue()` — returns `T` or `null` if none.

### Package-private concrete subclasses (lines 54-97)
All follow the same two-constructor pattern (with and without default):
- `StringConfigKey` — wraps `String.class`
- `BooleanConfigKey` — wraps `Boolean.class`
- `IntegerConfigKey` — wraps `Integer.class`
- `LongConfigKey` — wraps `Long.class`
- `DoubleConfigKey` — wraps `Double.class`

## Key flows

Keys are instantiated once as static constants in [[Keys.java]] and referenced by name when calling `Config.getString(Keys.FOO)`. No instances are created at runtime.

## Gotchas / non-obvious

- **Package-private subclasses** — only code in `org.traccar.config` can `new StringConfigKey(...)`. External code only uses the `ConfigKey<T>` base type.
- **Default `null` for String/Boolean** — `null` default on a `ConfigKey<Boolean>` means the getter in `Config.getBoolean` falls back to `false`, not a `NullPointerException`.
- **`types` is a `HashSet`** — a key can belong to multiple scopes simultaneously (e.g., `CONFIG` and `DEVICE`).

## Line index

- 22-52 — abstract `ConfigKey<T>` with constructor + four getters
- 54-60 — `StringConfigKey` (null default / explicit default)
- 63-70 — `BooleanConfigKey`
- 72-79 — `IntegerConfigKey`
- 81-88 — `LongConfigKey`
- 90-97 — `DoubleConfigKey`
