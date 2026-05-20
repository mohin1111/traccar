# PortConfigSuffix.java

**Role:** Specialization of `ConfigSuffix<Integer>` for the `.port` key. Holds a static map of all 265 protocol names → default port numbers (5001–5265). `withPrefix("its")` returns a key defaulting to port 5179.
**Fits in:** Instantiated as `Keys.PROTOCOL_PORT` in [[Keys.java]]. Used when each `BaseProtocol` constructor calls `Keys.PROTOCOL_PORT.withPrefix(protocolName)` to find its listen port.
**Read next:** [[Keys.java]], [[ConfigSuffix.java]]

## Public API

- `withPrefix(String protocol)` → `IntegerConfigKey` with default = `PORTS.get(protocol)`.

## Key flows

Protocol auto-discovery flow:
1. `ClassScanner` finds all `BaseProtocol` subclasses.
2. Each protocol ctor calls `config.getInteger(Keys.PROTOCOL_PORT.withPrefix(name))`.
3. If no `<entry key="its.port">` in config, returns `PORTS.get("its")` = 5179.
4. `ServerManager` binds a Netty TCP/UDP server on that port.

## Gotchas / non-obvious

- **Port map is the canonical port registry** — if a new protocol is added, its entry here is its "official" default port. The ITS (AIS-140) protocol is at 5179.
- Indian/relevant protocols and their default ports: `its=5179` (AIS-140), `aquila=5089`, `idpl=5110`, `racedynamics=5195`, `jt808=5015` (Chinese standard), `teltonika=5027`.
- **`PORTS.get(protocol)` returns null** for unknown protocol names, which becomes `null` default in the `IntegerConfigKey`, causing `Config.getInteger` to return 0. A port of 0 means Netty will not bind a server.

## Line index

- 24 — `PORTS` static map declaration
- 27-291 — static initializer: 265 `PORTS.put(name, port)` entries
- 293-295 — constructor
- 297-300 — `withPrefix` override
