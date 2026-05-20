# OpenChannelHandler.java

**Role:** Maintains the `TrackerConnector`'s channel group — registers newly active channels and deregisters closed ones. Enables broadcasting commands to all connected device channels.
**Fits in:** Early Netty pipeline stage (step 3). One `OpenChannelHandler` per `TrackerConnector` (i.e., per protocol server port).
**Read next:** [[MainEventHandler.java.md]] (last in pipeline), [[NetworkMessageHandler.java.md]]

## Public API

### `channelActive` (lines 31-34)
Adds the channel to `connector.getChannelGroup()` after calling `super.channelActive` (important order — Netty requires parent to run first).

### `channelInactive` (lines 37-40)
Removes the channel from the group.

## Gotchas / non-obvious

- **`ChannelGroup`** is Netty's thread-safe broadcast collection — used by `CommandManager` to push commands to connected devices.
- Not `@Sharable` — each connector instance gets its own handler. One handler manages one connector's channel group.
- The handler is constructed with the `TrackerConnector` reference directly, not injected.

## Line index

- 23 — `connector` field
- 31-34 — `channelActive` + group add
- 37-40 — `channelInactive` + group remove
