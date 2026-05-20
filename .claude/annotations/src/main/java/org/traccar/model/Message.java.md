# Message.java

**Role:** Intermediate abstract base for device-associated messages. Adds `deviceId` and `type` to `ExtendedModel`. Parent of `Position`, `Event`, and (via `BaseCommand`) `Command` and `QueuedCommand`.
**Fits in:** Pure structural node in the hierarchy. Not persisted directly — concrete subclasses have `@StorageName` annotations.
**Read next:** [[ExtendedModel.java]] (parent), [[Position.java]], [[Event.java]], [[BaseCommand.java]] (children)

## Public API

- `deviceId long` (line 20) — FK to `tc_devices.id`. Present in all subclass tables.
- `type String` (line 30) — meaning varies by subclass: event type string in `Event`, command type in `Command`, suppressed in `Position` via `@QueryIgnore @JsonIgnore` override.

## Line index

- 18 — `class Message extends ExtendedModel`
- 20-28 — deviceId
- 30-38 — type
