# Pair.java

**Role:** Generic immutable two-element tuple. Java record. Used for returning dual values from helper methods without creating dedicated result classes.
**Fits in:** Utility used across the codebase wherever a method needs to return two related values together.

## Public API

- `record Pair<K, V>(K first, V second)` (line 18) — generic; `getFirst()`/`getSecond()` implicit; immutable; `equals`/`hashCode`/`toString` auto-generated.

## Line index

- 18 — `public record Pair<K, V>(K first, V second) {}`
