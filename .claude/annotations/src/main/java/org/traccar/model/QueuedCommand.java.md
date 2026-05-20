# QueuedCommand.java

**Role:** Persistent queue entry for commands that could not be delivered immediately (device offline). `@StorageName("tc_commands_queue")` → `tc_commands_queue`. Structurally identical to `BaseCommand` content; provides `fromCommand`/`toCommand` conversion helpers.
**Fits in:** `CommandsManager` creates `QueuedCommand` rows when a device is offline; `ConnectionManager` dequeues and dispatches them when the device reconnects.
**Read next:** [[Command.java]], [[BaseCommand.java]] (parent)

## Public API

- `fromCommand(Command)` static (line 25) — copies deviceId, type, textChannel, attributes from a `Command` into a new `QueuedCommand` for persistence.
- `toCommand()` (line 34) — reconstructs a `Command` from the queued row; sets `description = ""`.

## Gotchas / non-obvious

- `@JsonIgnoreProperties(ignoreUnknown = true)` — same as `Command`.
- The `attributes` copy in `fromCommand` (line 30) uses `new AttributeMap(command.getAttributes())` — a shallow copy of the map contents. Sufficient because attribute values are immutable types.

## Line index

- 21 — `@StorageName("tc_commands_queue")`
- 22 — `class QueuedCommand extends BaseCommand`
- 25-32 — fromCommand static factory
- 34-43 — toCommand converter
