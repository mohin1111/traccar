# MediaManager.java

**Role:** Filesystem storage for device-uploaded binary media files (photos, video clips). Organizes files under `<mediaPath>/<deviceUniqueId>/<timestamp>.<ext>`.
**Fits in:** `@Singleton` injected into protocol decoders that handle media (e.g., video/dashcam protocols) and into `VideoStreamManager`. Exposed indirectly via `api/resource/MediaResource`.
**Read next:** `VideoStreamManager.java` (caller), [[Config.java]] (`Keys.MEDIA_PATH`)

## Public API

- `createFileStream(String uniqueId, String name, String extension)` (line 67) — creates the directory path and returns an open `FileOutputStream` for streaming writes. Caller is responsible for closing.
- `writeFile(String uniqueId, ByteBuf buf, String extension)` (line 71) — writes a Netty `ByteBuf` to a timestamped file. Returns the generated filename (e.g., `20240518143022.jpg`), or `null` if `MEDIA_PATH` is not configured.

## Key flows

### Path construction (`createFile`, lines 55-64)
`path.resolve(uniqueId).resolve(name).normalize()` then validates the result starts with `path` to prevent path traversal. Creates directories with `Files.createDirectories`.

### Timestamped filename (line 77)
`DATE_FORMAT = "yyyyMMddHHmmss"`. Filename is `<timestamp>.<extension>`.

### ByteBuf write (lines 79-88)
Uses `FileChannel.write(ByteBuffer)` in a loop until all bytes are written, then `fileChannel.force(false)` to flush to OS (not necessarily disk).

## Gotchas / non-obvious

- **`path` is null when `MEDIA_PATH` not configured** — `writeFile` returns `null` silently. Protocol decoders should handle null return gracefully.
- **`createFile` throws `IOException`** on path traversal attempts — the `!filePath.startsWith(path)` check.
- **No size limits** — device can fill the disk with media. Consider adding quotas for production.
- **`createFileStream` leaves the stream open** — caller must close it. Used for streaming protocols where data arrives in chunks.

## Line index

- 43-44 — `DATE_FORMAT` constant (`yyyyMMddHHmmss`, system timezone)
- 50-52 — `@Inject` constructor: resolves and normalizes media path
- 55-64 — `createFile` (path construction + traversal check + mkdir)
- 67-69 — `createFileStream` (streaming write setup)
- 71-89 — `writeFile` (ByteBuf → timestamped file via FileChannel)
