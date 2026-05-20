# BaseHttpProtocolDecoder.java

**Role:** Thin subclass of `BaseProtocolDecoder` for HTTP-based protocol decoders. Adds `sendResponse()` helpers to write back HTTP responses to the device.
**Fits in:** Extended by protocols that receive device data via HTTP POST/GET (e.g., OsmAnd, Teltonika HTTP). Pipeline includes Netty `HttpServerCodec` + `HttpObjectAggregator` in `addTransportHandlers`.
**Read next:** [[BaseProtocolDecoder.java]] (parent class), [[BasePipelineFactory.java]] (pipeline context)

## Public API

- `sendResponse(Channel channel, HttpResponseStatus status)` (line 33) — sends empty HTTP response.
- `sendResponse(Channel channel, HttpResponseStatus status, ByteBuf buf)` (line 37) — sends HTTP response with body. Writes `Content-Length` header. Wraps in `NetworkMessage`.

## Key flows

Subclass `decode()` calls `sendResponse` before returning the decoded `Position`. Typical pattern:
```java
sendResponse(channel, HttpResponseStatus.OK);
return position;
```

## Gotchas / non-obvious

- **No content-type header set** — callers add `Content-Type` to `buf` before passing if needed (e.g., for JSON responses).
- Null `buf` is handled: substituted with `Unpooled.buffer(0)`, so `sendResponse(channel, OK)` is safe.
- HTTP protocol decoders must also send a response for login/heartbeat messages, not just position messages.

## Line index

- 33-34 — `sendResponse` (no body)
- 37-46 — `sendResponse(status, buf)` with Content-Length
