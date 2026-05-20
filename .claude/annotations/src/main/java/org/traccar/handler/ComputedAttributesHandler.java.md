# ComputedAttributesHandler.java

**Role:** Evaluates per-device JEXL expressions (computed attributes) and writes results back to the `Position`. Appears **twice** in the pipeline as two inner-class variants: `Early` (step 1, before filtering) and `Late` (step 13, after geocode/geofence).
**Fits in:** Extends `BasePositionHandler`. Delegates expression evaluation to `ComputedAttributesProvider` (the singleton JEXL engine). Reads `Attribute` objects from `CacheManager`.
**Read next:** [[ComputedAttributesProvider.java.md]] (the JEXL engine), [[BasePositionHandler.java.md]]

## Public API

### Inner class `Early` (lines 37-42)
- Guice-injected. Evaluates attributes with `priority < 0`. Runs as step 1 of the pipeline, before `FilterHandler`.

### Inner class `Late` (lines 44-49)
- Guice-injected. Evaluates attributes with `priority >= 0`. Runs as step 13, after geocode/geofence/speed-limit.

### `onPosition(Position, Callback)` (lines 59-105)
- Loads all `Attribute` objects for the device from cache.
- Filters by `early` flag: `priority < 0` for Early, `>= 0` for Late.
- Sorts by priority descending (highest priority evaluated first).
- For each attribute: calls `provider.compute(attribute, position)`, then writes result into `position`.

## Key flows

### Attribute write-back (lines 69-93)
Known field names (`valid`, `latitude`, `longitude`, `altitude`, `speed`, `course`, `address`, `accuracy`) are set via typed setters. All other attribute names are written into the position's attribute map using the attribute's declared `type` (`number`, `boolean`, or string). If result is `null`, the attribute is removed.

## Gotchas / non-obvious

- **Priority sign determines phase** — negative priority = Early, non-negative = Late. Counterintuitive but correct.
- **Sorted descending** — higher priority number evaluated first within each phase.
- **`JexlException` and `ClassCastException` are swallowed with a WARN log** — a bad expression silently skips that attribute; it does not halt the pipeline.
- **`callback.processed(false)` always** — computed attributes never filter positions. The `false` is unconditional.

## Line index

- 37-42 — `Early` inner class
- 44-49 — `Late` inner class
- 59 — `onPosition` entry
- 60-63 — load + filter + sort attributes
- 64-103 — evaluation loop
- 69-93 — field/attribute write-back switch
- 97-101 — exception handling
- 104 — `callback.processed(false)`
