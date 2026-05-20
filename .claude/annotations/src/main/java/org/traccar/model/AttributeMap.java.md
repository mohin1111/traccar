# AttributeMap.java

**Role:** Memory-efficient `Map<String, Object>` implementation used as the `attributes` bag inside every `ExtendedModel`. Stores key-value pairs in a flat interleaved `Object[]` array instead of a linked structure, avoiding `HashMap` overhead for the typical small attribute count per entity.
**Fits in:** Instantiated by `ExtendedModel` (line 25 of that file). Jackson deserializes into it via `@JsonDeserialize(as = AttributeMap.class)`.
**Read next:** [[ExtendedModel.java]] (owner/user)

## Public API

- Extends `AbstractMap<String, Object>` — full `Map` contract.
- `AttributeMap()` — empty map, starts with `EMPTY` (zero-length) backing array (line 32).
- `AttributeMap(Map<? extends String, ?> source)` — copy constructor (line 36).
- All standard `Map` methods: `put`, `get`, `remove`, `containsKey`, `size`, `isEmpty`, `clear`, `entrySet`.

### Storage layout
Flat `Object[] data` stores `[key0, val0, key1, val1, ...]`. `size` is number of entries (not array length). Initial allocation is 8 slots (4 entries) on first `put` (line 124).

## Key flows

### Lookup (`indexOf`, line 109)
Linear scan by identity (`==`) then `.equals()`. O(n) — acceptable because attribute counts are typically < 20 per entity.

### Resize (`ensureCapacity`, line 121)
Doubles array length each time capacity is exceeded. Starts at `INITIAL_CAPACITY = 8` (4 entries).

## Gotchas / non-obvious

- `remove(key)` compacts by copying tail entries over the removed slot (line 87) — preserves insertion order for remaining entries.
- This is **not thread-safe** — no synchronization. Positions are built on a single decoder thread so this is fine.
- Jackson's `@JsonDeserialize(as = AttributeMap.class)` means the JSON `{"key": value}` attributes column is deserialized directly into this class. Values come back from Jackson as `Integer`/`Long`/`Double`/`Boolean`/`String`.

## Line index

- 28-31 — constants (EMPTY, INITIAL_CAPACITY)
- 32 — `data` + `size` fields
- 54 — `containsKey`
- 58 — `get`
- 64 — `put`
- 79 — `remove`
- 109 — `indexOf` (linear scan)
- 121 — `ensureCapacity`
- 129-214 — inner classes: EntrySet, EntryIterator, ArrayEntry
