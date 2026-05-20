# BaseCommand.java

**Role:** Abstract base for command objects. Extends `Message` (gets `deviceId`, `type`, `attributes`), adds `textChannel` flag distinguishing SMS commands from data-channel commands.
**Fits in:** Parent of both `Command` (saved command templates in `tc_commands`) and `QueuedCommand` (pending delivery in `tc_commands_queue`).
**Read next:** [[Message.java]] (parent), [[Command.java]], [[QueuedCommand.java]]

## Public API

- `textChannel boolean` (line 21) — `true` = send via SMS; `false` = send via primary data channel. Determines which `*ProtocolEncoder` path is taken by `CommandsManager`.

## Line index

- 18 — `class BaseCommand extends Message`
- 21-28 — textChannel
