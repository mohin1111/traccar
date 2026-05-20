# AcknowledgementHandler.java

**Role:** Outbound Netty handler that holds back protocol ACK responses until all decoded objects from a single network message have been fully processed by the position pipeline. Prevents the device from receiving an ACK before the server has handled its data.
**Fits in:** Netty channel pipeline, outbound direction. Located between the protocol encoder and the wire. Works via the inner event types `EventReceived`, `EventDecoded`, `EventHandled` written to the channel by `ProcessingHandler`.
**Read next:** [[MainEventHandler.java.md]], [[NetworkMessageHandler.java.md]]

## Public API

### Inner event classes (lines 36-60)
- `EventReceived` — signals start of a new inbound message batch. Opens the "queue" window.
- `EventDecoded` — provides the set of decoded objects (positions). Each is added to `waiting`.
- `EventHandled` — removes a single object from `waiting`. When `waiting` is empty and `EventDecoded` has been seen, the queued outbound messages are released.

### `write(ChannelHandlerContext, Object, ChannelPromise)` (lines 84-115)
- If `msg instanceof Event`: updates the state machine.
- If the message is a regular outbound buffer (ACK) and a queue is open: queues the write.
- Once all objects are handled (`waiting.isEmpty()`): flushes the queued writes downstream.

## Key flows

1. Inbound message arrives → `ProcessingHandler` writes `EventReceived` to channel.
2. Protocol decoder produces N positions → `ProcessingHandler` writes `EventDecoded({pos1, pos2, ...})`.
3. Each position completes the pipeline → `ProcessingHandler` writes `EventHandled(pos)`.
4. Protocol encoder writes ACK to channel → `AcknowledgementHandler` holds it in `queue`.
5. When `waiting` is empty (all positions handled): releases queued ACK to the wire.

## Gotchas / non-obvious

- **Synchronized block on `this`** (line 86) — the event state is shared between multiple Netty I/O operations; synchronization prevents race between decoding and handling events.
- **`ChannelOutboundHandlerAdapter`** — only intercepts outbound (write) direction, not reads.
- **Handles multi-position messages** — a single UDP frame can decode to multiple positions; all must be handled before ACK is released.

## Line index

- 36-48 — `EventReceived`, `EventDecoded`, `EventHandled` inner classes
- 62-78 — `Entry` (queued message + promise)
- 84-115 — `write()` state machine
- 89 — `EventReceived`: open queue
- 93-95 — `EventDecoded`: add all to waiting set
- 96-98 — `EventHandled`: remove from waiting
- 100-102 — flush queue when all handled
