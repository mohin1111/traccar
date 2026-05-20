# VideoStreamResource.java

**Role:** REST resource at `/api/stream` for HLS video streaming from devices with camera support. Returns M3U8 playlists and MPEG-TS segments.
**Fits in:** Extends BaseResource. Uses VideoStreamManager.
**Read next:** [[BaseResource.java]], `VideoStreamManager.java`, `media/` package

## Public API

- `GET /api/stream/{deviceId}/{channel}/live.m3u8` — returns HLS playlist for device + channel; permission checked.
- `GET /api/stream/{deviceId}/{channel}/{index}.ts` — returns MPEG-TS segment; permission checked.

## Gotchas / non-obvious

- Permission checked against `Device.class` for both endpoints — viewer must have access to the device.
- Segment data is a Netty `ByteBuf`; written via `StreamingOutput` that calls `data.getBytes(...)` directly into the response stream.

## Line index

- 31 — class extends `BaseResource`
- 37-44 — playlist endpoint
- 46-58 — segment endpoint
