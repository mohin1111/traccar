# CommandResource.java

**Role:** REST resource at `/api/commands` for device command templates and sending. Extends ExtendedObjectResource for saved command CRUD. Adds send endpoint (immediate or queued), command type enumeration, and group-broadcast.
**Fits in:** Entity: `Command` (tc_commands — saved templates). Extends ExtendedObjectResource.
**Read next:** [[ExtendedObjectResource.java]], `CommandsManager.java`, `CommandSenderManager.java`, `QueuedCommand.java`

## Public API

| Path | Purpose |
|---|---|
| CRUD `/api/commands` | Saved command template CRUD |
| `GET /api/commands/send?deviceId=` | List sendable commands for device |
| `POST /api/commands/send?groupId=` | Send command to device or group |
| `GET /api/commands/types?deviceId=&textChannel=` | List supported command types |

## Key flows

### `GET /api/commands/send` (lines 97-118)
Loads saved commands linked to both the user AND the device. Filters by protocol support (text vs data channel, protocol-declared type set).

### `POST /api/commands/send` (lines 121-158)
- Pre-saved command (id > 0): loads from DB, overrides deviceId.
- Ad-hoc command: checks `limitCommands` restriction.
- Group send: iterates accessible devices in group; `commandsManager.sendCommand` for each; returns accepted QueuedCommands if queued.
- Single device: `commandsManager.sendCommand` → if queued (device offline), returns 202 Accepted with QueuedCommand; if sent immediately, returns 200 OK.

### `GET /api/commands/types` (lines 161-199)
If deviceId given: checks `CommandSenderManager` first (for custom senders); else queries protocol for supported types by channel. No deviceId: returns all `Command.TYPE_*` constants via reflection.

## Gotchas / non-obvious

- 200 vs 202 response: 200 = command sent (device online); 202 = queued (device offline, will send on next connection).
- `textChannel=true` selects SMS-style commands; false selects binary protocol commands.

## Line index

- 63 — extends `ExtendedObjectResource<Command>`
- 86-94 — `getDeviceProtocol` helper
- 97-118 — list sendable commands
- 121-158 — send command (single + group)
- 161-199 — command types
