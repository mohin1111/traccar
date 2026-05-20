# Typed.java

**Role:** Minimal Java record wrapping a single `type` string. Used as a request/response body when an API endpoint needs only a type string (e.g., listing supported command types for a device's protocol).
**Fits in:** Used by `CommandResource.getTypes()` which returns `List<Typed>` — the set of command types the device's protocol encoder supports. Also used in computed attribute type selection.
**Read next:** [[Command.java]] (TYPE_* constants), [[Attribute.java]]

## Public API

- `record Typed(String type)` (line 19) — canonical record; `getType()` implicit; immutable; `equals`/`hashCode`/`toString` auto-generated.

## Line index

- 19 — `public record Typed(String type) {}`
