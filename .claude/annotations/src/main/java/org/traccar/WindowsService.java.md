# WindowsService.java

**Role:** JNA-based Windows Service Control Manager (SCM) integration. Enables Traccar to install, uninstall, start, and stop itself as a Windows service. Abstract `run()` is overridden in `Main` to call `Main.run(configFile)`.
**Fits in:** Used only when `Main.main()` receives `--install`, `--uninstall`, or `--service` flags. Irrelevant for Linux/Mac/Docker deployments.
**Read next:** [[Main.java]] (caller)

## Public API

- `install(displayName, description, dependencies, account, password, config)` — registers the JVM + JAR as a service entry in Windows SCM with `SERVICE_AUTO_START`.
- `uninstall()` — removes the service entry.
- `start()` / `stop()` (static helpers) — start/stop the service via SCM (not the current process).
- `init()` — enters `StartServiceCtrlDispatcher`; blocks until SCM signals stop.
- `run()` — abstract; called when the service starts.

## Gotchas / non-obvious

- Requires JNA (`com.sun.jna`) and `jnr-posix` on the classpath — both are in `build.gradle`.
- `System.exit(0)` in `ServiceMain.callback` (line 200) is intentional — avoids a Windows crash bug when ServiceMain returns.
- Not relevant for Transport OS containerized deployment; safe to ignore on Linux.

## Line index

- 49-88 — `install()`
- 91-103 — `uninstall()`
- 105-121 — `start()` (SCM)
- 123-141 — `stop()` (SCM)
- 142-153 — `init()` (StartServiceCtrlDispatcher)
- 170 — abstract `run()`
- 173-204 — `ServiceMain` inner class
- 206-218 — `ServiceControl` inner class (stop handler)
