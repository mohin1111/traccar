# BaseMqttProtocolDecoder.java

**Role:** Specialized `BaseProtocolDecoder` for MQTT-over-TCP device protocols. Handles CONNECT, SUBSCRIBE, and PUBLISH message types internally; subclass implements only `decode(DeviceSession, MqttPublishMessage)`.
**Fits in:** Extended by protocols that use MQTT as their transport (e.g., devices running MqttClient talking to Traccar as broker). Pipeline includes Netty `MqttDecoder`/`MqttEncoder` in transport handlers.
**Read next:** [[BaseProtocolDecoder.java]] (parent), [[BasePipelineFactory.java]] (pipeline context)

## Public API

- `decode(DeviceSession deviceSession, MqttPublishMessage message)` — abstract; implement topic-based position parsing here. `deviceSession` is already resolved from the CONNECT clientId.

## Key flows

### MQTT handshake handling (lines 41-85)
- `MqttConnectMessage`: calls `getDeviceSession(channel, remoteAddress, clientIdentifier)`. Responds with `CONNACK` — `CONNECTION_ACCEPTED` or `CONNECTION_REFUSED_IDENTIFIER_REJECTED`.
- `MqttSubscribeMessage`: responds with `SUBACK`. No data processing.
- `MqttPublishMessage`: retrieves the session resolved at CONNECT time; calls subclass `decode(deviceSession, message)`; responds with `PUBACK`.

## Gotchas / non-obvious

- The client identifier (IMEI) is set at CONNECT time; PUBLISH messages use `getDeviceSession(channel, remoteAddress)` with no new identifiers — session is cached per channel.
- `decode` is `final` — subclasses cannot override the base dispatch logic. Only `decode(DeviceSession, MqttPublishMessage)` is the extension point.
- QoS 0 PUBLISH doesn't expect PUBACK, but Traccar sends one anyway — most devices tolerate this.

## Line index

- 35 — abstract `decode(DeviceSession, MqttPublishMessage)`
- 38-88 — final `decode` override
- 41-52 — CONNECT handling (session resolution + CONNACK)
- 55-63 — SUBSCRIBE handling (SUBACK)
- 65-84 — PUBLISH handling (session lookup + subclass dispatch + PUBACK)
