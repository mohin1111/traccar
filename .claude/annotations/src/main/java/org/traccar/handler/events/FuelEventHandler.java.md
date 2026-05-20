# FuelEventHandler.java

**Role:** Detects significant fuel level changes — refueling (`TYPE_DEVICE_FUEL_INCREASE`) and fuel theft/drain (`TYPE_DEVICE_FUEL_DROP`) — by comparing `KEY_FUEL` between consecutive positions.
**Fits in:** Extends `BaseEventHandler`. Runs in the event-detection phase.
**Read next:** [[BaseEventHandler.java.md]]

## Public API

### `onPosition(Position, Callback)` (lines 37-75)
- Guards: device must exist, position must be latest, both current and last must have `KEY_FUEL`.
- Computes `change = after - before`.
- Positive change >= `EVENT_FUEL_INCREASE_THRESHOLD` → fire `TYPE_DEVICE_FUEL_INCREASE`.
- Negative change (absolute) >= `EVENT_FUEL_DROP_THRESHOLD` → fire `TYPE_DEVICE_FUEL_DROP`.
- Events include `before` and `after` fuel values as attributes.

## Gotchas / non-obvious

- **Threshold = 0 suppresses the event** — both thresholds default to 0. You must explicitly configure per-device thresholds to enable fuel events.
- **`KEY_FUEL` units are protocol-dependent** — some devices report liters, some percentages. Threshold must be calibrated to match.
- **Noise sensitivity** — rapid sensor fluctuations can trigger false events. The threshold is the only filter; no smoothing is applied.

## Line index

- 47 — `KEY_FUEL` presence check
- 48-49 — read before/after values
- 50 — compute change
- 54-62 — increase branch + threshold check
- 63-72 — drop branch + threshold check
