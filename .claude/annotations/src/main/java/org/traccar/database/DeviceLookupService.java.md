# DeviceLookupService.java

**Role:** Throttled device lookup by `uniqueId`. Wraps `Storage.getObject(Device, Condition.Equals("uniqueId", ...))` with exponential backoff throttling for unknown identifiers to prevent DB hammering by unregistered devices.
**Fits in:** Called from `BaseProtocolDecoder.getDeviceSession()` for every incoming message from an unknown device. Critical for DOS protection.
**Read next:** `BaseProtocolDecoder.java` (caller), [[Config.java]] (`Keys.DATABASE_THROTTLE_UNKNOWN`)

## Public API

- `lookup(String[] uniqueIds)` (line 118) — tries each ID in order. Returns first matching `Device`, or `null` if none found. Respects throttling for unknown IDs.

## Key flows

### Throttle on miss (lines 101-116)
1. First miss for an ID: sets `delay = THROTTLE_MIN_MS` (1 min), records `lastQuery`.
2. Subsequent misses: `delay = min(delay * 2, THROTTLE_MAX_MS)` (exponential backoff, cap 30 min).
3. While throttled: `isThrottled` returns true, `lookup` skips the DB call and returns null.
4. After `INFO_TIMEOUT_MS` (60 min) of no attempts, the entry is evicted from `identifierMap`.

### Throttle on hit (lines 92-99)
Successful lookup removes the entry and cancels its timer (device is now known).

## Gotchas / non-obvious

- **Multiple `uniqueIds`** — some protocols send multiple IDs (primary + secondary). The first match wins; throttle is per-ID, not per-set.
- **`throttlingEnabled`** — off by default; enabled via `Keys.DATABASE_THROTTLE_UNKNOWN=true`. In production with many unregistered devices (e.g., stolen/cloned IMEIs), enabling this protects the DB.
- **All throttle state is in-memory** — lost on restart. Acceptable trade-off; the backoff resets on restart.
- **`identifierMap` is `ConcurrentHashMap`** but throttle logic is `synchronized` on `this` — coarse-grained locking is intentional for correctness.

## Line index

- 43-45 — timeout constants (INFO=60min, MIN=1min, MAX=30min)
- 52-57 — `IdentifierInfo` inner class (lastQuery, delay, timeout)
- 58-72 — `IdentifierTask` timer task (evicts entry on expiry)
- 77-81 — `@Inject` constructor
- 83-89 — `isThrottled`
- 92-99 — `lookupSucceeded` (reset throttle)
- 101-116 — `lookupFailed` (exponential backoff)
- 118-139 — `lookup` (iterate ids, check throttle, DB query)
