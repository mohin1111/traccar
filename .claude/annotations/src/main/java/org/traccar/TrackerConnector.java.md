# TrackerConnector.java

**Role:** Interface combining `LifecycleObject` with connector metadata (`isDatagram`, `isSecure`, `getChannelGroup`). Extended by both `TrackerServer` and `TrackerClient`.
**Fits in:** `ServerManager` holds a `List<TrackerConnector>` and starts/stops all. `OpenChannelHandler` uses `connector.getChannelGroup()` to register new channels.
**Read next:** [[TrackerServer.java]], [[TrackerClient.java]], [[LifecycleObject.java]]

## Public API

```java
boolean isDatagram();   // UDP = true, TCP = false
boolean isSecure();     // TLS = true
ChannelGroup getChannelGroup();   // all active channels for this connector
```

Inherits `start()` and `stop()` from `LifecycleObject`.
