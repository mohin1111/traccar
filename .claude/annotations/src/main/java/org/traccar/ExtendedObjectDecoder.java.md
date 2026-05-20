# ExtendedObjectDecoder.java

**Role:** Netty `ChannelInboundHandlerAdapter` that is the direct parent of `BaseProtocolDecoder`. Implements `channelRead`: unwraps `NetworkMessage`, calls abstract `decode()`, fires results downstream, and optionally saves the original raw bytes to the `Position`.
**Fits in:** Sits between the frame decoder output and `ProcessingHandler`. All 265 protocol decoders ultimately inherit `channelRead` from here.
**Read next:** [[BaseProtocolDecoder.java]] (extends this), [[NetworkMessage.java]] (the wrapper type consumed here), [[ProcessingHandler.java]] (downstream recipient)

## Public API

- `decode(Channel channel, SocketAddress remoteAddress, Object msg)` — abstract; the method all protocol decoders implement. Return a `Position`, a `Collection<Position>`, or null.
- `getConfig()` — returns the injected `Config` instance. Available after Guice injection.
- `onMessageEvent(Channel, SocketAddress, Object originalMessage, Object decodedMessage)` — hook called after decode; overridden by `BaseProtocolDecoder` to update device status.
- `handleEmptyMessage(Channel, SocketAddress, Object)` — called when `decode()` returns null; override to synthesize a position for heartbeats.

## Key flows

### `channelRead` (lines 67-95)
1. Unwraps `NetworkMessage` → `originalMessage` + `remoteAddress`.
2. Fires `AcknowledgementHandler.EventReceived()` immediately.
3. Calls `decode()`.
4. Calls `onMessageEvent()`.
5. If null: calls `handleEmptyMessage()`.
6. If collection: fires each position downstream; fires `EventDecoded(collection)`.
7. If single: fires single position; fires `EventDecoded(list-of-one)`.
8. Always: releases `originalMessage` via `ReferenceCountUtil.release()`.

### Original message saving (lines 55-63)
If `database.saveOriginal=true`, hex-dumps the raw `ByteBuf` or ASCII-encodes the raw `String` into `Position.KEY_ORIGINAL`. Useful for audit/debugging.

## Gotchas / non-obvious

- `AcknowledgementHandler.EventReceived()` is written before `decode()` — even if decode throws, the ACK handler knows a message arrived. Exception propagates to Netty's `exceptionCaught`.
- `ReferenceCountUtil.release(originalMessage)` in `finally` — critical for `ByteBuf` ref-counting. Never release `originalMessage` inside `decode()`.
- `Config` is injected via `@Inject setConfig(Config)` which also calls `init()` — init() runs AFTER construction, so constructor-time use of `getConfig()` would NPE.

## Line index

- 38-43 — `Config` injection + `init()` call
- 53 — `init()` hook (empty, override if needed)
- 55-63 — `saveOriginal` helper
- 67-95 — `channelRead` (the full decode loop)
- 71 — `AcknowledgementHandler.EventReceived()` fired first
- 72 — `decode()` call
- 73 — `onMessageEvent()` hook
- 78-90 — collection vs single dispatch
- 94 — `ReferenceCountUtil.release`
- 97-103 — `onMessageEvent` / `handleEmptyMessage` default (empty)
- 104 — abstract `decode` signature
