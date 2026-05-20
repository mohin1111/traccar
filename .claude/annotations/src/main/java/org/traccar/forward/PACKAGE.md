# `forward/` — Position and event forwarders

Integrations that push processed positions and events to external systems in real time. Each forwarder is a separate class implementing a common interface. Transport OS can use these to feed a data lake, stream processor, or custom API.

## File index

| File | One-liner |
|---|---|
| `PositionForwarder.java` | Interface: `forward(PositionData positionData, ResultHandler resultHandler)` |
| `EventForwarder.java` | Interface: `forward(EventData eventData, ResultHandler resultHandler)` |
| `PositionData.java` | Data envelope: `Position` + `Device` |
| `EventData.java` | Data envelope: `Event` + `Position` + `Device` + optional `Geofence`/`Maintenance` |
| `ResultHandler.java` | Callback interface: `onResult(boolean success, Throwable throwable)` |
| `PositionForwarderUrl.java` | HTTP GET with URL template substitution (`{latitude}`, `{uniqueId}`, `{gprmc}`, etc.) |
| `PositionForwarderJson.java` | HTTP POST with full position JSON body |
| `PositionForwarderKafka.java` | Apache Kafka producer (position JSON as message value, deviceId as key) |
| `PositionForwarderMqtt.java` | MQTT publish via HiveMQ client |
| `PositionForwarderAmqp.java` | RabbitMQ AMQP publish |
| `PositionForwarderRedis.java` | Redis pub/sub publish via Lettuce |
| `PositionForwarderWialon.java` | Wialon IPS UDP datagram (legacy fleet protocol) |
| `EventForwarderAmqp.java` | AMQP event forwarding |
| `EventForwarderKafka.java` | Kafka event forwarding |
| `EventForwarderMqtt.java` | MQTT event forwarding |
| `EventForwarderJson.java` | HTTP POST event forwarding |
| `AmqpClient.java` | Shared RabbitMQ AMQP connection wrapper |
| `MqttClient.java` | Shared HiveMQ MQTT connection wrapper |
| `NetworkForwarder.java` | Raw TCP/UDP byte mirroring (forward raw device traffic to another host) |

## Dependency graph

```
PositionForwardingHandler (handler/) → PositionForwarder
NotificationManager (database/) → EventForwarder

Bound in MainModule based on config:
  forward.url → PositionForwarderUrl or PositionForwarderJson
  forward.kafka.* → PositionForwarderKafka + EventForwarderKafka
  forward.mqtt.* → PositionForwarderMqtt + EventForwarderMqtt (MqttClient)
  forward.amqp.* → PositionForwarderAmqp + EventForwarderAmqp (AmqpClient)
  forward.redis.* → PositionForwarderRedis
  forward.wialon.* → PositionForwarderWialon
```

## URL template tokens (`PositionForwarderUrl`)

`{name}`, `{uniqueId}`, `{status}`, `{deviceId}`, `{protocol}`, `{deviceTime}`, `{fixTime}`, `{valid}`, `{latitude}`, `{longitude}`, `{altitude}`, `{speed}`, `{course}`, `{accuracy}`, `{address}`, `{attributes}` (JSON), `{gprmc}` (NMEA sentence), `{statusCode}` (OpenGTS).

## Configuration

```xml
<!-- HTTP URL template (GET) -->
<entry key="forward.url">https://my-api.example.com/track?lat={latitude}&amp;lon={longitude}&amp;id={uniqueId}</entry>

<!-- Kafka -->
<entry key="forward.kafka.bootstrapServers">localhost:9092</entry>
<entry key="forward.kafka.topic">traccar-positions</entry>

<!-- MQTT -->
<entry key="forward.mqtt.url">tcp://localhost:1883</entry>
<entry key="forward.mqtt.topic">traccar/positions</entry>

<!-- AMQP (RabbitMQ) -->
<entry key="forward.amqp.url">amqp://guest:guest@localhost</entry>
<entry key="forward.amqp.exchange">traccar</entry>
```

## Transport OS integration points

**Primary integration:** Use `PositionForwarderKafka` to stream all positions into a Kafka topic consumed by the Transport OS backend. Topic partitioned by `deviceId` for ordering.

**Event forwarding:** `EventForwarderKafka` / `EventForwarderJson` to push geofence enter/exit, overspeed, alarms, etc. to the Transport OS event service.

**AIS-140 compliance:** Forward panic/SOS events to the control-room endpoint via `EventForwarderJson` with a webhook filter on `event.type = "alarm"` and `event.attributes.alarm = "sos"`.

## How to add a new forwarder

1. Implement `PositionForwarder` (and/or `EventForwarder`).
2. Add config keys to [[Keys.java]] (`forward.mytype.*`).
3. In `MainModule`, add conditional binding: `if (config.hasKey(Keys.FORWARD_MY_URL)) { bind(PositionForwarder.class).to(PositionForwarderMy.class); }`.
4. No changes to `PositionForwardingHandler` needed — it uses the bound `PositionForwarder` instance.
