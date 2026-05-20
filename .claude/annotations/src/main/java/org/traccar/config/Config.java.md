# Config.java

**Role:** Singleton config loader. Reads the XML properties file passed as the first CLI argument, exposes typed getters, and optionally reads from environment variables.
**Fits in:** Loaded once by `MainModule.java` via Guice `@Named("configFile")` binding. Injected into almost every service in the codebase.
**Read next:** [[Keys.java]] (the typed key catalog), [[MainModule.java]] (where this is bound)

## Public API

### Constructors (lines 39-59)
- `Config()` — no-arg, for testing. Properties remain empty.
- `Config(String file)` — `@Inject` constructor. Loads XML via `Properties.loadFromXML`. Sets up logger. Throws `RuntimeException` if XML is malformed.

### Key lookup (lines 61-68)
- `hasKey(ConfigKey<?>)` — true if key exists in properties or env.

### Typed getters (lines 70-133)
- `getString(ConfigKey<String>)` / `getString(ConfigKey<String>, String defaultValue)` / `getString(String)` — raw string read; env override checked first.
- `getBoolean(ConfigKey<Boolean>)` — parses as `Boolean.parseBoolean`; falls back to key default then `false`.
- `getInteger(ConfigKey<Integer>)` / `getInteger(ConfigKey<Integer>, int)` — parses as int; 0 if missing.
- `getLong(ConfigKey<Long>)` — parses as long; 0L if missing.
- `getDouble(ConfigKey<Double>)` — parses as double; 0.0 if missing.

### Test helper (lines 135-138)
- `setString(ConfigKey<?>, String)` — `@VisibleForTesting`. Directly puts into `Properties`. Used in tests without a file.

## Key flows

### Env variable override (lines 48-49, 80-87)
When `config.useEnvironmentVariables=true` (in file) or `CONFIG_USE_ENVIRONMENT_VARIABLES=true` (env), each lookup checks `System.getenv(getEnvironmentVariableName(key))` first.

### Env variable name transform (line 140-142)
`getEnvironmentVariableName("geocoder.type")` → `GEOCODER_TYPE`. Dots replaced with underscores, then camelCase humps get an underscore prefix and uppercased.

## Gotchas / non-obvious

- **Config file is XML Properties format** (`<properties>` / `<entry key="...">...`) — not YAML, not `.properties` text. Using `loadFromXML`, not `load`.
- **Env override is all-or-nothing at the key level** — if env var is set but empty string, it is treated as not present (line 83).
- **All numeric defaults are 0 / false** when no default is set on the key and the key is missing. This means missing integer keys silently return 0 — always define defaults in [[Keys.java]].
- **No validation** — `getInteger` will throw `NumberFormatException` if the value is not a valid integer.

## Line index

- 42-59 — `@Inject` constructor: XML load + env flag + logger setup
- 61-68 — `hasKey` (checks both env and properties)
- 70-87 — `getString` overloads including env-variable path
- 90-133 — `getBoolean`, `getInteger`, `getLong`, `getDouble`
- 135-138 — `@VisibleForTesting setString`
- 140-142 — `getEnvironmentVariableName` static helper (dot→underscore + camelCase expand)
