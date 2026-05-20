# ExtendedModel.java

**Role:** Second tier of the model hierarchy. Adds a JSON `attributes` bag (`AttributeMap`) and a full suite of typed getter/setter/remove helpers for reading from it. All configuration-carrying entities extend this.
**Fits in:** Sits between `BaseModel` and most concrete entities. `DatabaseStorage`/`QueryBuilder` JSON-serializes `attributes` into a single `attributes` column in each entity's table.
**Read next:** [[BaseModel.java]], [[AttributeMap.java]] (the underlying map), [[GroupedModel.java]] (next tier), [[Position.java]] (heavy user of `set(key, value)`)

## Public API

### Fields
- `attributes AttributeMap` (line 25) — serialized as JSON in every entity's `attributes` column.

### Key methods
- `hasAttribute(String key)` (line 27) — containsKey check.
- `getAttributes()` / `setAttributes(AttributeMap)` (lines 31-37) — raw map access; setter null-guards.
- `set(String, Boolean|Byte|Short|Integer|Long|Float|Double|String)` (lines 39-85) — typed setters; all are null-safe (skip put if value is null). `Float` is upcast to `double`; `Byte`/`Short` are upcast to `int`.
- `add(Map.Entry<String, Object>)` (line 87) — null-safe entry put.
- `getString/getDouble/getBoolean/getInteger/getLong(key)` (lines 93-123) — typed getters with defaults.
- `removeAttribute/removeString/removeDouble/removeBoolean/removeInteger/removeLong(key)` (lines 125-146) — remove-and-return helpers.

## Key flows

### Protocol decoder → Position attributes
Protocol decoders call `position.set(KEY_ALARM, "sos")` etc. These all funnel through overloaded `set()` into the `AttributeMap`. `QueryBuilder` later JSON-serializes the map into `tc_positions.attributes`.

## Gotchas / non-obvious

- `set(String, String value)` (line 81) skips put when value is **empty string** as well as null — distinct from the numeric overloads which only skip on null.
- `Float` values are stored as `double` (line 70); reading back via `getDouble()` is fine, but `getInteger()` will truncate.
- `setAttributes(null)` replaces with a **new empty** `AttributeMap`, never stores null (line 36).

## Line index

- 25 — `AttributeMap attributes` field
- 27-29 — `hasAttribute`
- 31-37 — get/setAttributes
- 39-85 — overloaded `set()` suite
- 87-91 — `add(entry)`
- 93-127 — typed getters
- 125-146 — remove helpers
- 149-193 — private parseAs* helpers
