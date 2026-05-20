# ComputedAttributesProvider.java

**Role:** Singleton JEXL engine that compiles and executes computed attribute expressions. The central extension point for server-side position enrichment logic.
**Fits in:** `@Singleton` injected into `ComputedAttributesHandler` (both Early and Late). Holds the shared `JexlEngine` and sandbox configuration.
**Read next:** [[ComputedAttributesHandler.java.md]] (consumer), [[BasePositionHandler.java.md]]

## Public API

### Constructor (lines 57-79)
Builds the `JexlEngine` with:
- **Sandbox:** only `Math`, primitive wrappers, `String`, `Date`, `HashMap`, `LinkedHashMap`, and array types are allowed. Classes not on the allowlist cannot be instantiated from expressions.
- **Features:** `localVar`, `loops`, `newInstance` — each controlled by config keys `PROCESSING_COMPUTED_ATTRIBUTES_LOCAL_VARIABLES`, `PROCESSING_COMPUTED_ATTRIBUTES_LOOPS`, `PROCESSING_COMPUTED_ATTRIBUTES_NEW_INSTANCE_CREATION`. All default to off.
- **Namespace:** `math:` prefix maps to `java.lang.Math` — expressions can call `math:abs(x)`, `math:sqrt(x)`, etc.
- **`strict(true)`** — undefined variables throw instead of returning null silently.

### `compute(Attribute, Position)` (lines 81-85)
- Creates a JEXL script from `attribute.getExpression()`, executes it against a `MapContext` built from the position, returns the result (or `null`).
- Throws `JexlException` on expression error (caller catches it).

## Key flows

### `prepareContext(Position)` (lines 87-123)
The JEXL context variables available to expressions:
1. **Device attributes** (if `includeDeviceAttributes` is true): each `device.attributes` entry is set as a top-level variable.
2. **Position fields** via `ReflectionCache.getProperties(Position.class, "get")`: every getter on `Position` becomes a variable (e.g., `speed`, `latitude`, `valid`, `fixTime`). Map-typed getters (like `getAttributes()`) are flattened — each key in the map becomes its own variable.
3. **Last position fields** (if `includeLastAttributes` is true): same as above but prefixed with `last` (e.g., `lastSpeed`, `lastLatitude`).

## Gotchas / non-obvious

- **Sandbox is deny-by-default** — `new JexlSandbox(false)` means everything is denied unless explicitly allowed. Custom Java classes are unreachable from expressions unless added to the sandbox.
- **`strict(true)` + `localVar(false)` by default** means expressions must reference existing position fields; typos in variable names throw at runtime.
- **`ReflectionCache`** caches `MethodHandle`s — reflection cost is paid once, not per position.
- **`lastXxx` variables** expose the previous stored position — useful for delta computations (e.g., `speed - lastSpeed`). Only available if `PROCESSING_COMPUTED_ATTRIBUTES_LAST_ATTRIBUTES = true`.
- **`com.safe.Functions`** is explicitly allowed in the sandbox — this is a placeholder for a custom utility class (not present in open-source build; add your own at that package path).

## Line index

- 57-79 — constructor: sandbox + features + engine construction
- 59 — `JexlSandbox(false)` deny-by-default
- 60 — allow `com.safe.Functions`
- 67-71 — feature flags from config
- 72-76 — engine with strict mode + math namespace
- 77-78 — `includeDeviceAttributes`, `includeLastAttributes` flags
- 81-85 — `compute()` public entry point
- 87-123 — `prepareContext()` builds MapContext
- 125-127 — `prefixAttribute()` helper (camel-cases `last` prefix)
