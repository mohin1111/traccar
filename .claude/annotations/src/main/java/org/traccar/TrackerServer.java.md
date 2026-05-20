# TrackerServer.java

**Role:** `TrackerConnector` for inbound listening sockets (TCP `ServerBootstrap` or UDP `Bootstrap`). Binds to a port, accepts connections, and runs `BasePipelineFactory` on each channel.
**Fits in:** `BaseProtocol.addServer(new TrackerServer(config, getName(), false) {...})` — each protocol creates one or more `TrackerServer` instances in its constructor.
**Read next:** [[TrackerConnector.java]], [[BasePipelineFactory.java]], [[BaseProtocol.java]]

## Public API

- `TrackerServer(Config config, String protocol, boolean datagram)` — reads port/address/SSL from config. If `datagram=true`, uses `NioDatagramChannel` + single-group `Bootstrap`; if false, uses `NioServerSocketChannel` + `ServerBootstrap` with boss+worker groups.
- `addProtocolHandlers(PipelineBuilder pipeline, Config config)` — abstract; override in anonymous subclass in `BaseProtocol` constructor.
- `start()` — binds the bootstrap; blocks until bound; adds channel to `channelGroup`.
- `stop()` — closes all channels in the group.
- `getPort()` / `getAddress()` — read-only accessors.

## Key flows

### TCP bootstrap (lines 88-95)
```
ServerBootstrap
  .group(bossGroup, workerGroup)
  .channel(NioServerSocketChannel.class)
  .childHandler(pipelineFactory)
```
Boss accepts connections; worker runs I/O. `pipelineFactory.initChannel()` is called per accepted channel.

### UDP bootstrap (lines 84-88)
```
Bootstrap
  .group(workerGroup)
  .channel(NioDatagramChannel.class)
  .handler(pipelineFactory)
```
Single channel, no boss group. `NetworkMessageHandler` converts `DatagramPacket` to `NetworkMessage`.

## Gotchas / non-obvious

- `datagram=false` for ITS (AIS-140 is TCP). `datagram=true` for OsmAnd and other UDP protocols.
- SSL is supported for TCP only — `SslHandler` is added in `addTransportHandlers`. No DTLS for UDP.
- If `address` is null (not set in config), binds on all interfaces (`new InetSocketAddress(port)`).

## Line index

- 58-95 — constructor (reads config + creates bootstrap)
- 63-80 — anonymous `BasePipelineFactory` with `addTransportHandlers` (SSL) + `addProtocolHandlers` delegation
- 84-94 — UDP vs TCP bootstrap branch
- 97 — abstract `addProtocolHandlers`
- 113-125 — `start()` (bind + channelGroup add)
- 127-129 — `stop()`
