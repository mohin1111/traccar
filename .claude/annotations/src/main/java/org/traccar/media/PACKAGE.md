# `media/` — Media file storage

Two files for managing binary media (dashcam images, video clips) uploaded by devices.

## File index

| File | One-liner |
|---|---|
| `VideoStreamManager.java` | Manages active HLS video stream sessions per device; serves via `api/resource/VideoStreamResource` |
| `VideoStreamWriter.java` | Writes HLS `.ts` segments received from device into the media path |

Note: `MediaManager.java` (the primary media store) lives in `database/` but is closely related — see [[MediaManager.java.md]].

## Dependency graph

```
VideoStreamResource (api/resource/) → VideoStreamManager
VideoStreamManager → VideoStreamWriter → MediaManager (database/)
Protocol decoders (e.g., DualCam) → MediaManager.writeFile() or .createFileStream()
```

## How media storage works

1. Device sends binary payload (image/video chunk) in a protocol frame.
2. Protocol decoder calls `mediaManager.writeFile(uniqueId, byteBuf, "jpg")` or `mediaManager.createFileStream(uniqueId, name, "ts")`.
3. Files are stored at `<MEDIA_PATH>/<deviceUniqueId>/<timestamp>.<ext>`.
4. Client requests media via `GET /api/media/<deviceUniqueId>/<filename>` — served as static files by Jetty.

## Video stream flow

`VideoStreamManager` tracks active stream sessions: when `VideoStreamResource` opens an SSE connection, the manager registers it. As device sends `.ts` segments, `VideoStreamWriter` appends them. The SSE stream notifies the client of new segments for HLS playback.

## Configuration

```xml
<entry key="media.path">/var/traccar/media</entry>
```

Directory must be writable by the Traccar process. Disk usage scales with device count × media frequency.
