# ServerManager.java

**Role:** Protocol auto-discovery and lifecycle orchestration. Scans `org.traccar.protocol` for all `BaseProtocol` subclasses, instantiates those with a configured port, collects their `TrackerConnector` lists, and starts/stops them.
**Fits in:** Started by `Main.run()` as the second `LifecycleObject`. Depends on `Config` and the Guice `Injector`. Zero protocol-specific code here — all logic is in the individual `*Protocol` classes.
**Read next:** [[BaseProtocol.java]] (what each protocol registers), [[TrackerServer.java]] / [[TrackerClient.java]] (the connectors), [[BasePipelineFactory.java]] (channel pipeline wired per connector)

## Public API

- Constructor `ServerManager(Injector injector, Config config)` (line 48) — calls `ClassScanner.findSubclasses(BaseProtocol.class, "org.traccar.protocol")`, instantiates matching protocols via Guice, populates `connectorList` and `protocolList`.
- `getProtocol(String name)` (line 66) — lookup by protocol name string (e.g., `"its"`); used by `CommandManager` to route commands to the right protocol.
- `start()` / `stop()` (lines 71-88) — iterate `connectorList` and call `connector.start()` / `connector.stop()`. `BindException` and `ConnectException` are caught and logged as warnings (port conflict does not crash the server).

## Key flows

### Auto-discovery (lines 50-63)
```
ClassScanner.findSubclasses(BaseProtocol.class, "org.traccar.protocol")
  → for each protocolClass:
      name = BaseProtocol.nameFromClass(protocolClass)   // "ItsProtocol" → "its"
      if enabledProtocols == null OR name in enabledProtocols:
          if config.getInteger("its.port") > 0:
              protocol = injector.getInstance(protocolClass)
              connectorList.addAll(protocol.getConnectorList())
              protocolList.put(name, protocol)
```

**Filter priority:**
1. Optional whitelist: `protocols.enable = its,teltonika,...` — if absent, all protocols with a configured port are started.
2. Port config: `its.port = 5050` — protocols with no port configured are silently skipped.

### Port conflict handling (line 75)
`BindException` during `connector.start()` logs a warning and continues — the server does not exit. This means a misconfigured or conflicting port just disables that protocol silently. Check logs on startup.

## Gotchas / non-obvious

- **No manual registration** — adding a new `*Protocol.java` class to `org.traccar.protocol` with a configured port is sufficient. `ClassScanner.findSubclasses` discovers it via classpath scanning.
- `protocolList` is a `ConcurrentHashMap` — thread-safe for `getProtocol()` reads from any thread.
- `enabledProtocols` split regex is `[, ]` (comma or space) — both separators are valid in config.
- `ClassScanner` uses the classpath JAR manifest for discovery in production; in tests it scans the compiled class directory.

## Line index

- 44-45 — `connectorList` + `protocolList` fields
- 48-64 — constructor (auto-discovery loop)
- 50-53 — optional `protocols.enable` whitelist
- 54 — `ClassScanner.findSubclasses(BaseProtocol.class, "org.traccar.protocol")` — the key call
- 55 — `BaseProtocol.nameFromClass` (strips "Protocol" suffix, lowercases)
- 57 — port check (`its.port > 0`)
- 58-62 — Guice instantiation + connector collection
- 66-68 — `getProtocol(name)` lookup
- 71-81 — `start()` with BindException/ConnectException catch
- 83-88 — `stop()`
