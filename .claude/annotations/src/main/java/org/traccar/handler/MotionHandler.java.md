# MotionHandler.java

**Role:** Derives `KEY_MOTION` (boolean) from position speed when the device does not report it natively. Threshold is per-device via `AttributeUtil.lookup`.
**Fits in:** Extends `BasePositionHandler`. Step 12 in the pipeline (after `SpeedLimitHandler`, before `ComputedAttributesHandler.Late`). `MotionEventHandler` (event layer) depends on this value.
**Read next:** [[SpeedLimitHandler.java.md]] (step before), [[events/MotionEventHandler.java.md]] (consumer)

## Public API

### `onPosition(Position, Callback)` (lines 35-43)
- Skips if position already has `KEY_MOTION` (device-reported value wins).
- Reads `EVENT_MOTION_SPEED_THRESHOLD` (knots) per-device via `AttributeUtil.lookup`.
- Sets `KEY_MOTION = (position.speed > threshold)`.

## Gotchas / non-obvious

- **Speed is in knots** throughout Traccar's model. `threshold` from config is also in knots — no conversion needed.
- **Device-reported `KEY_MOTION` always wins** — guard at line 36.
- **`threshold = 0`** means motion is detected if speed > 0 (any movement). A small non-zero value avoids flagging GPS noise.

## Line index

- 36 — guard: skip if device-reported
- 37-39 — threshold lookup + set motion boolean
