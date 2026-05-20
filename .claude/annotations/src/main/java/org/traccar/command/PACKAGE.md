# `command/` — Device command dispatch strategies

Provides pluggable strategies for sending out-of-band commands to devices that may not be reachable via their current TCP/UDP connection.

## File index

| File | One-liner |
|---|---|
| `CommandSender.java` | Interface: `sendCommand(Device device, Command command)` |
| `CommandSenderManager.java` | Picks the right `CommandSender` implementation for a given device |
| `FirebaseCommandSender.java` | Sends command via Firebase Cloud Messaging (FCM) as a data push to the device |
| `TraccarCommandSender.java` | Sends command via the Traccar push notification cloud service (uses `Keys.NOTIFICATOR_TRACCAR_KEY`) |
| `FindHubCommandSender.java` | Sends command via the FindHub device management cloud API |

## Dependency graph

```
CommandSenderManager
 ├── FirebaseCommandSender   (when device has notificationTokens + FCM configured)
 ├── TraccarCommandSender    (when device has notificationTokens + Traccar key configured)
 └── FindHubCommandSender    (when device has sender="findHub" attribute)

CommandsManager (database/) → CommandSenderManager.getSender(device)
```

## Sender selection logic (`CommandSenderManager.getSender`)

1. If device has `commandSender` attribute set explicitly: use the named sender (`firebase` / `traccar` / `findHub`).
2. Else if device has `notificationTokens` attribute:
   - If `Keys.COMMAND_CLIENT_SERVICE_ACCOUNT` configured → `FirebaseCommandSender`
   - Else if `Keys.NOTIFICATOR_TRACCAR_KEY` configured → `TraccarCommandSender`
3. Else → `null` (fall through to live TCP session or queue in `CommandsManager`).

## How to add a new command sender

1. Implement `CommandSender` interface.
2. Add a constant to `CommandSenderManager.SENDERS_ALL` map (key = attribute value string).
3. Configure the sender's credentials in [[Keys.java]] and `traccar.xml`.
4. Set `commandSender=mykey` in a device's attributes to route commands to it.

## Key config

- `Keys.COMMAND_CLIENT_SERVICE_ACCOUNT` — Firebase service account JSON for FCM
- `Keys.NOTIFICATOR_TRACCAR_KEY` — Traccar cloud push API key
- Per-device: `commandSender` attribute (`"firebase"` / `"traccar"` / `"findHub"`)
- Per-device: `notificationTokens` attribute — comma-separated FCM registration tokens

## Transport OS note

For AIS-140 immobilization (ENGINE_STOP command), use the live TCP path via `ItsProtocolEncoder` — no `CommandSender` needed. `CommandSenderManager` is only needed when the device is not connected via TCP (mobile app push or third-party cloud API).
