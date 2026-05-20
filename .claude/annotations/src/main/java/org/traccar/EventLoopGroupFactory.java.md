# EventLoopGroupFactory.java

**Role:** `@Singleton` that creates and holds the two Netty `EventLoopGroup` instances (boss and worker). Both `TrackerServer` and `TrackerClient` obtain these via `Main.getInjector().getInstance(EventLoopGroupFactory.class)`.
**Fits in:** Instantiated by Guice on demand. Used in `TrackerServer` constructor (boss + worker groups) and `TrackerClient` constructor (worker group only).
**Read next:** [[TrackerServer.java]], [[TrackerClient.java]]

## Public API

- `getBossGroup()` — `NioIoHandler`-backed group for accepting connections. Configured via `server.netty.bossThreads` (default 0 = `2 * cores`).
- `getWorkerGroup()` — group for I/O processing. Configured via `server.netty.workerThreads` (default 0 = `2 * cores`).

## Gotchas / non-obvious

- Both groups use `MultiThreadIoEventLoopGroup` with `NioIoHandler` (Netty 5 API replacing `NioEventLoopGroup`). If you upgrade Netty, this is the migration point.
- Not stopped on shutdown — Netty event loops are daemon threads; JVM exit terminates them. For graceful shutdown, consider adding `shutdownGracefully()` calls in `Main` shutdown hook.

## Line index

- 33-39 — constructor (creates boss + worker groups)
- 41-43 — `getBossGroup`
- 45-47 — `getWorkerGroup`
