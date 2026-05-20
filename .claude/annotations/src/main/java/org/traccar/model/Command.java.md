# Command.java

**Role:** Represents a saved command template or an API-initiated command dispatch request. `@StorageName("tc_commands")` → `tc_commands`. Contains the command type, optional parameters in `attributes`, and a `description` label.
**Fits in:** Extends `BaseCommand` → `Message` → `ExtendedModel`. `CommandResource` persists/retrieves saved commands. `CommandManager` translates a `Command` to protocol-specific bytes via the appropriate `*ProtocolEncoder`. For queued delivery, `QueuedCommand.fromCommand()` clones it into `tc_commands_queue`.
**Read next:** [[BaseCommand.java]] (parent), [[QueuedCommand.java]] (queue storage), [[Typed.java]] (used in type filtering)

## Public API

### DB-mapped fields (`tc_commands` columns)
- Inherited `type String` (from `Message`) — one of the ~40 `TYPE_*` constants below. This is the command type, not an event type.
- Inherited `deviceId long` — target device. Note: overridden here with `@QueryIgnore` (lines 90-99) meaning the `deviceId` from `Message` is NOT stored in `tc_commands` (it's a saved template, not bound to a device). The `deviceId` is only set at dispatch time.
- Inherited `attributes AttributeMap` — command parameters. Keys: `KEY_FREQUENCY`, `KEY_TIMEZONE`, `KEY_MESSAGE`, `KEY_ENABLE`, `KEY_PHONE`, etc.
- Inherited `textChannel boolean` — SMS vs data channel.
- `description String` (line 102) — label for UI display.

### Type constants (lines 27-73)
~40 `TYPE_*` constants. Key ones for Transport OS / AIS-140:
- `TYPE_ENGINE_STOP` / `TYPE_ENGINE_RESUME` — immobilizer control.
- `TYPE_POSITION_SINGLE` / `TYPE_POSITION_PERIODIC` / `TYPE_POSITION_STOP` — polling control.
- `TYPE_ALARM_SOS` — SOS alarm configuration.
- `TYPE_CUSTOM` — raw arbitrary command text.

### Parameter key constants (lines 75-88)
`KEY_UNIQUE_ID`, `KEY_FREQUENCY`, `KEY_LANGUAGE`, `KEY_TIMEZONE`, `KEY_DEVICE_PASSWORD`, `KEY_RADIUS`, `KEY_MESSAGE`, `KEY_ENABLE`, `KEY_DATA`, `KEY_INDEX`, `KEY_PHONE`, `KEY_SERVER`, `KEY_PORT`, `KEY_NO_QUEUE`.

## Gotchas / non-obvious

- `deviceId` is `@QueryIgnore` in `Command` (lines 90-99) — saved command templates are not device-specific. The device target is supplied when dispatching via `CommandResource.send()`.
- `@JsonIgnoreProperties(ignoreUnknown = true)` (line 24) — safe to add new server-side fields without breaking older API clients.

## Line index

- 23 — `@StorageName("tc_commands")`
- 25 — `class Command extends BaseCommand`
- 27-73 — TYPE_* constants (~40)
- 75-88 — KEY_* parameter constants
- 90-99 — deviceId override (@QueryIgnore)
- 102-109 — description
