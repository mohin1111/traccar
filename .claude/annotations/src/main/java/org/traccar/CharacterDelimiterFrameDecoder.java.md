# CharacterDelimiterFrameDecoder.java

**Role:** Convenience wrapper around Netty's `DelimiterBasedFrameDecoder`. Accepts `char`, `String`, or `String[]` delimiter specifications and converts them to `ByteBuf` delimiters.
**Fits in:** Used in `addProtocolHandlers` for protocols with simple line-based or character-terminated frames (e.g., `\r\n`, `;`, `*`).
**Read next:** [[BaseFrameDecoder.java]] (alternative for custom framing logic), [[BasePipelineFactory.java]] (pipeline context)

## Public API

Five constructors: `(maxFrameLength, char)`, `(maxFrameLength, String)`, `(maxFrameLength, stripDelimiter, String)`, `(maxFrameLength, String...)`, `(maxFrameLength, stripDelimiter, String...)`.

## Key flows

Static helpers `createDelimiter(char)`, `createDelimiter(String)`, `convertDelimiters(String[])` build `ByteBuf` wrappers; all delegate to `DelimiterBasedFrameDecoder`.

## Gotchas / non-obvious

- Strips the delimiter from the frame by default (Netty default behavior). Use `stripDelimiter=false` constructors if the protocol decoder needs to see the delimiter.
- Only ASCII characters work correctly — byte cast of `charAt(i)`. Non-ASCII delimiters will silently truncate.

## Line index

- 24-35 — `createDelimiter` helpers
- 37-43 — `convertDelimiters`
- 45-63 — five constructors
