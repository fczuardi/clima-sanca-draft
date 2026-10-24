# MeshCore integration investigation

Source: S3 in [sources](../sources.md). **Research only; no firmware, service, database, or dashboard changes were made.**

## Decision

Continue with the operational Meshtastic telemetry pipeline and defer MeshCore support. A compatible Meshtastic board already combines sensor acquisition, LoRa networking, and direct Wi-Fi/MQTT publishing. Additional MeshCore components were not justified by an immediate operational requirement.

This was a postponement, not a permanent rejection.

## Alternatives discussed

| Alternative | Intended behavior | Status |
| --- | --- | --- |
| Separate ingestion | MeshCore MQTT decoder writes to the shared TimescaleDB; Grafana combines data | Proposed; not installed |
| Protocol translator | Decode MeshCore events and generate synthetic Meshtastic protobuf envelopes for the existing exporter | Concept only; no matching implementation identified in the discussion |
| Standalone MeshCore Wi-Fi/MQTT firmware | One ESP32 reads sensors, uses LoRa, and publishes directly to Mosquitto | Capability not established for official firmware; not tested |

Possible tools mentioned were `meshcore-mqtt`, `meshcore-hub`, Telegraf, Node-RED, or a custom Python consumer. These are historical research leads, not verified recommendations.

The third-party [Meshcore-Repeater-MQTT-Gateway](https://github.com/jmead/Meshcore-Repeater-MQTT-Gateway) was identified for further investigation. Its maintenance, board compatibility, sensor support, TLS support, and maturity were not verified.

A translator would need stable synthetic node identities, sensor conversion, and valid Meshtastic service envelopes. Candidate message types included `TELEMETRY_APP`, `NODEINFO_APP`, and `POSITION_APP`. None were implemented or tested.

## Proposed separation

If revisited, reuse Mosquitto with distinct topic namespaces and independent decoders, then combine measurements in TimescaleDB/Grafana. A `meshcore/...` namespace was suggested. Do not treat example topic paths as deployed configuration or insert foreign payloads into Meshtastic topics without a compatibility layer.

## Internet bridging

Connecting distant RF islands through internet gateways was explored conceptually, using possible MQTT, TCP, or WebSocket transport and selective forwarding. This is not evidence that a specific MeshCore firmware implements those capabilities. No gateway-to-gateway bridge was configured.

## Open questions

- Which maintained firmware supports sensors, LoRa, Wi-Fi, and MQTT together on the intended Heltec board?
- Which bridges support a private broker and TLS, and what telemetry formats do they emit?
- Could Telegraf ingest those payloads without custom processing?
- What shared schema would preserve protocol and station identity?
- Are maintained bidirectional RF-island bridges available?
- Would translation provide a benefit over independent ingestion?

No answers were established because integration was deferred. Claims about current MeshCore capabilities require a new source review before implementation.
