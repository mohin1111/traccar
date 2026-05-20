# Attribute.java

**Role:** Represents a computed attribute rule (JEXL expression). `@StorageName("tc_attributes")` → `tc_attributes`. Each row defines a JEXL expression that is evaluated against each `Position`'s attributes to produce a derived value. Used for fuel conversion, threshold normalisation, custom calculations.
**Fits in:** Does NOT extend `ExtendedModel` — extends `BaseModel` directly (no attributes bag of its own). Loaded by `ComputedAttributesHandler` which evaluates `expression` against a `MapContext` containing the position's attributes. Linked to devices/groups/users via `tc_device_attribute`, `tc_group_attribute`, `tc_user_attribute`.
**Read next:** [[BaseModel.java]] (parent), [[Position.java]] (the context evaluated against), [[ExtendedModel.java]] (result written back into position.attributes)

## Public API

### DB-mapped fields (`tc_attributes` columns)
- `description String` (line 24) — human-readable label shown in UI.
- `attribute String` (line 34) — the output attribute key written back into `position.attributes` after evaluation (e.g., `"fuelLevel"`, `"customTemp"`).
- `expression String` (line 44) — JEXL expression string. Context variables are position attribute keys. Example: `"fuel / 100.0 * 60"`.
- `type String` (line 54) — cast target after evaluation: `"String"`, `"Double"`, `"Boolean"`, `"Integer"`, `"Long"`. Used by `ComputedAttributesProvider` to cast the JEXL result.
- `priority int` (line 64) — evaluation order; lower = earlier. Used to allow one computed attribute to depend on another.

## Key flows

### JEXL evaluation in `ComputedAttributesHandler`
1. Load all `Attribute` rows linked to device (via group chain).
2. Sort by `priority`.
3. For each `Attribute`, build a `JexlContext` from `position.getAttributes()`.
4. Evaluate `expression` with `JexlEngine` (sandboxed).
5. Cast result to `type`.
6. Write back: `position.set(attribute, result)`.

## Gotchas / non-obvious

- `Attribute` is the entity for **computed/JEXL attributes** — not to be confused with the `attributes` JSON column in `ExtendedModel`. The name "attribute" is overloaded in this codebase.
- `priority` is evaluated lowest-first; use it to chain computations (e.g., first compute `normalizedFuel`, then reference it in `fuelLevel`).
- There is no validation of `expression` at persist time; a bad JEXL expression will throw at runtime and the position will be stored without that attribute.

## Line index

- 21 — `@StorageName("tc_attributes")`
- 22 — `class Attribute extends BaseModel`
- 24-31 — description
- 34-41 — attribute (output key)
- 44-51 — expression (JEXL string)
- 54-61 — type (cast target)
- 64-71 — priority (evaluation order)
