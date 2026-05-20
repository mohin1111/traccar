# BehaviorEventHandler.java

**Role:** Detects harsh acceleration and harsh braking by computing m/s² from the speed delta between consecutive positions. Fires `ALARM_ACCELERATION` or `ALARM_BRAKING` events.
**Fits in:** Extends `BaseEventHandler`. Runs in the event-detection phase.
**Read next:** [[BaseEventHandler.java.md]], [[AlarmEventHandler.java.md]]

## Public API

### `onPosition(Position, Callback)` (lines 41-57)
- Requires a prior position (`lastPosition != null`) with a different `fixTime`.
- Computes: `acceleration = Δspeed_mps / Δtime_seconds` (m/s²).
- If `acceleration >= accelerationThreshold` → fire `ALARM_ACCELERATION`.
- If `acceleration <= -brakingThreshold` → fire `ALARM_BRAKING`.

## Gotchas / non-obvious

- **Acceleration threshold in m/s²** — config keys `EVENT_BEHAVIOR_ACCELERATION_THRESHOLD` and `EVENT_BEHAVIOR_BRAKING_THRESHOLD`. Typical passenger-car harsh threshold: ~3–4 m/s².
- **Speed must be in knots** (Traccar model) — `UnitsConverter.mpsFromKnots` converts before computing the delta.
- **Same-timestamp guard** (`!position.getFixTime().equals(lastPosition.getFixTime())`) prevents division by zero.
- **Threshold = 0 disables the check** — explicit guard at lines 47 and 50.

## Line index

- 35-36 — threshold fields
- 41 — `onPosition`
- 44 — acceleration = Δspeed_mps / Δtime_s
- 47-53 — acceleration/braking event emission
