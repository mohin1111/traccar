# StreamWriter.java

**Role:** JAX-RS `MessageBodyWriter<Stream<?>>` — allows resource methods to return `java.util.Stream<T>` directly and have it serialized as a JSON array without materializing the entire collection in memory. Used by all list endpoints in the `ExtendedObjectResource` and `SimpleObjectResource` hierarchy.
**Fits in:** Registered as a Jersey `@Provider`. Jersey's default writers cannot handle `Stream`; this fills the gap.
**Read next:** [[ExtendedObjectResource.java]] (returns `Stream<T>`), [[SimpleObjectResource.java]] (same), `ObjectMapper` (the serializer)

## Public API

- `isWriteable(type, ...)` (line 44) — returns true if the return type is `Stream` or subclass.
- `writeTo(stream, ...)` (line 49) — writes opening `[`, iterates stream writing each element, writes closing `]`, then closes the stream (try-with-resources).

## Key flows

Jackson `createGenerator` writes directly to the response `OutputStream` — no intermediate buffer. The stream is closed inside try-with-resources ensuring DB cursors are released even on early disconnect.

## Gotchas / non-obvious

- **`try (stream; var generator = ...)`** (line 52) — both the `Stream` and the Jackson generator are closed in the same try-with-resources block. Closing the `Stream` closes any underlying DB `ResultSet` cursor. This is the key memory-efficiency feature.
- If the stream throws mid-iteration, the generator may produce malformed JSON. Jersey will abort the connection; the client sees a truncated array.

## Line index

- 34 — `@Provider @Produces(APPLICATION_JSON)` + class declaration
- 44-46 — `isWriteable`
- 49-59 — `writeTo` (the streaming JSON array write loop)
