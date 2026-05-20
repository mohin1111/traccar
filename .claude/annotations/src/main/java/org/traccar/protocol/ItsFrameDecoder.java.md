# ItsFrameDecoder.java

**Role:** AIS-140 / ITS frame boundary detector. Reads a raw TCP `ByteBuf`, skips leading non-`$` bytes, and extracts one complete ITS frame using either `\r\n` or `*` as the frame terminator.
**Fits in:** First handler in the ITS pipeline. Extends `BaseFrameDecoder`. Sits between TCP stream and `StringDecoder`. Returns a `ByteBuf` slice containing one frame.
**Read next:** [[BaseFrameDecoder.java]] (parent), [[ItsProtocolDecoder.java]] (downstream consumer), [[ItsProtocol.java]] (pipeline setup)

## Public API

- `decode(ChannelHandlerContext ctx, Channel channel, ByteBuf buf)` — returns one frame or null (wait for more data).
- `MINIMUM_LENGTH = 20` — frames shorter than 20 bytes are not yet considered complete.

## Key flows

### Frame extraction algorithm (lines 40-65)
1. **Skip junk**: advance reader index until `$` is found (lines 43-45). ITS frames always start with `$`.
2. **Try `\r\n` terminator**: search for `\r\n` via `BufferUtil.indexOf`. If found at index > 20, call `readFrame(buf, delimiterIndex, skip=2)`.
3. **Fall back to `*` terminator**: search for `*` byte. If found at index > 20:
   - If `buf[delimiterIndex+1] == '*'`: advance `delimiterIndex` by 1 (double `**` terminator variant).
   - If `buf[delimiterIndex-2] == ','`: return `readFrame(buf, delimiterIndex-1, skip=2)` — skips binary checksum after `,`.
   - Otherwise: return `readFrame(buf, delimiterIndex, skip=1)`.
4. Return null if no valid terminator found yet.

### `readFrame(buf, delimiterIndex, skip)` (lines 28-36)
Checks if there's another `$` between current reader index+1 and `delimiterIndex`. If yes (embedded next frame), returns a slice up to the inner `$` — handles back-to-back frames with no delimiter. Otherwise returns a slice up to `delimiterIndex` and skips `skip` bytes (the delimiter itself).

## Gotchas / non-obvious

- **Double `**` terminator** (line 53-55): some ITS firmware variants use `**` as the end-of-frame marker instead of single `*`. The decoder handles this by advancing one byte.
- **Binary checksum skip** (line 57): when a `','` precedes the `*`, the two bytes after `*` are a binary checksum (not ASCII). These are skipped with `skip=2`.
- **Embedded next frame detection** in `readFrame` (line 29-31): if the buffer contains `$FRAME1$FRAME2`, where `$FRAME1` has no delimiter yet, the inner `$` triggers a slice at the boundary — preventing frame merging on reconnect.
- Not re-entrant by design — `ByteToMessageDecoder` (parent of `BaseFrameDecoder`) guarantees sequential calls per channel.

## Line index

- 26 — `MINIMUM_LENGTH = 20`
- 28-36 — `readFrame` helper
- 40-65 — `decode` main logic
- 43-45 — skip non-`$` bytes
- 47-49 — `\r\n` path
- 51-63 — `*` path with double-star and binary-checksum variants
