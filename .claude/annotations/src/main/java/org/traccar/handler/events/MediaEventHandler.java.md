# MediaEventHandler.java

**Role:** Emits `TYPE_MEDIA` events for positions that carry image, video, or audio file references (MDVR camera or dashcam integrations). One event per media attribute type present.
**Fits in:** Extends `BaseEventHandler`. Runs in the event-detection phase.
**Read next:** [[BaseEventHandler.java.md]]

## Public API

### `onPosition(Position, Callback)` (lines 30-40)
- Streams `[KEY_IMAGE, KEY_VIDEO, KEY_AUDIO]`.
- For each present in `position.attributes`, creates a `TYPE_MEDIA` event with `media` = attribute key and `file` = attribute value (file path/URL).

## Gotchas / non-obvious

- **Stateless** — no cache reads, no config. Pure attribute → event mapping.
- **Multiple media per position** — a dashcam position can carry all three types simultaneously; three events fire.
- **File path/URL is stored in the event** — clients retrieve the actual media via the `/api/media` endpoint.

## Line index

- 30 — `onPosition`
- 31-39 — stream-based attribute → event mapping
