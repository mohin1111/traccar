# Protocol.java

**Role:** Narrow interface defining the contract every protocol must fulfill. Exposes name, connector list, supported command sets, and command dispatch methods.
**Fits in:** `BaseProtocol` implements this. `BaseProtocolDecoder` holds a `Protocol` reference (not `BaseProtocol`) for loose coupling. `CommandManager` routes commands via `ServerManager.getProtocol(name)`.
**Read next:** [[BaseProtocol.java]] (implementation), [[ServerManager.java]] (stores protocols by name)

## Public API

```java
String getName();
Collection<TrackerConnector> getConnectorList();
Collection<String> getSupportedDataCommands();
void sendDataCommand(Channel channel, SocketAddress remoteAddress, Command command);
Collection<String> getSupportedTextCommands();
void sendTextCommand(String destAddress, Command command) throws Exception;
```

`sendTextCommand` declares `throws Exception` for SMS library checked exceptions. `sendDataCommand` does not — errors are wrapped in `RuntimeException` by `BaseProtocol`.
