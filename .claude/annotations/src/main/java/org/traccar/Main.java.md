# Main.java

**Role:** Entry point and Guice bootstrap. Owns the `injector` singleton, starts/stops all `LifecycleObject` services, handles Windows-service mode, and installs a JVM shutdown hook.
**Fits in:** Called by `java -jar tracker-server.jar <config.xml>`. Creates the injector with `MainModule`, `DatabaseModule`, `WebModule`. Then starts `ScheduleManager`, `ServerManager`, `WebServer`, `BroadcastService` in order.
**Read next:** [[MainModule.java]] (all DI bindings), [[ServerManager.java]] (protocol auto-discovery), [[LifecycleObject.java]] (start/stop contract)

## Public API

- `Main.main(String[] args)` (line 77) — entry; parses Windows flags (`--install`, `--uninstall`, `--service`) or calls `run()` directly.
- `Main.run(String configFile)` (line 114) — creates injector, calls `.start()` on all four services, registers shutdown hook.
- `Main.getInjector()` (line 46) — static accessor used throughout the codebase (notably `TrackerServer`, `TrackerClient`) to obtain DI instances outside the injection graph.

## Key flows

### Boot sequence (lines 114-155)
1. `Guice.createInjector(new MainModule(configFile), new DatabaseModule(), new WebModule())` — DI graph built. Liquibase migrations run eagerly inside `DatabaseModule.provideDataSource()`.
2. Four services started in strict order: `ScheduleManager` → `ServerManager` (opens all device ports) → `WebServer` (REST + WebSocket) → `BroadcastService` (cluster fan-out).
3. Shutdown hook (line 133) stops services in the same list order, then shuts down the shared `ExecutorService`.
4. Any `ProvisionException` is unwrapped to expose the real cause before `System.exit(1)`.

### Windows service mode (lines 90-111)
`--install`/`--uninstall`/`--service` flags delegate to `WindowsService`, which wraps the JNA `Advapi32` SCM API. `run()` is called inside the SCM callback. Only matters for Windows installations; Linux/Mac deployments skip this path.

## Gotchas / non-obvious

- `Main.getInjector()` is a static mutable field — it is set once in `run()` and never changed. Safe in normal operation; watch out in tests where `run()` may be skipped.
- Service start order matters: `ServerManager` must start before `WebServer` or the REST API might answer before device ports are open.
- `Locale.setDefault(Locale.ENGLISH)` at line 78 forces English for all number/date formatting in parsers — critical for protocol decoders that split on `.` decimal points.

## Line index

- 44 — static `injector` field
- 46 — `getInjector()` accessor
- 52 — `logSystemInfo()` (OS / JVM / memory diagnostics)
- 77 — `main()` entry
- 90-111 — Windows-service switch
- 114 — `run()` start
- 116 — Guice injector creation (three modules)
- 122-129 — service start loop (`ScheduleManager`, `ServerManager`, `WebServer`, `BroadcastService`)
- 131 — uncaught exception handler
- 133-144 — JVM shutdown hook
- 145-154 — error handling / System.exit(1)
