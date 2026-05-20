# ConnectionKey.java

**Role:** Immutable record identifying a unique network endpoint — the (localPort, remoteAddress) pair. Used as the map key in `ConnectionManager` to group device sessions by connection.
**Fits in:** Created by [[DeviceSession.java]] and used as keys in [[ConnectionManager.java]]'s `sessionsByEndpoint` and `unknownByEndpoint` maps. Also used by `CacheManager` to track which devices are connected.
**Read next:** [[DeviceSession.java]], [[ConnectionManager.java]]

## Public API

```java
public record ConnectionKey(SocketAddress localAddress, SocketAddress remoteAddress) {
    public ConnectionKey(Channel channel, SocketAddress remoteAddress) {
        this(channel.localAddress(), remoteAddress);
    }
}
```

Two fields: `localAddress` (server-side port, identifies the protocol) + `remoteAddress` (device IP:port). Record equality uses both fields.

## Key flows

### Why both local and remote address?
Traccar runs multiple protocol servers on different ports simultaneously (e.g., port 5050 for ITS, port 5055 for OsmAnd). A device at `192.168.1.100:34567` connecting to port 5050 is a different key than one connecting to port 5055. The `localAddress` disambiguates protocol.

### Convenience constructor
`new ConnectionKey(channel, remoteAddress)` extracts `channel.localAddress()` automatically — used in `ConnectionManager.getDeviceSession` and `deviceDisconnected`.

## Gotchas / non-obvious

- **Record equality** — Java records use all components for `equals`/`hashCode`. Both local and remote address must match for lookup to succeed.
- For UDP protocols, the `remoteAddress` is the device's UDP endpoint, not a persistent connection — a device sending from different ports creates different `ConnectionKey` instances.
