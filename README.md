# Clima Sanca

Documentation recovered from conversations about the São Carlos weather and Meshtastic project.

[Public weather dashboard](https://clima.falafel.com.br/sanca) · [Upstream telemetry exporter](https://github.com/tcivie/meshtastic-metrics-exporter)

The reported deployment collects environmental telemetry from ESP32/LoRa nodes using BME680 sensors. Meshtastic provides RF networking and Wi-Fi/MQTT publishing. Mosquitto feeds a Python exporter, TimescaleDB stores measurements, and Grafana presents them.

## Documentation

- [Deployment overview](docs/deployment.md): hardware, service placement, and evidence limits.
- [VPS migration and incident history](docs/history/vps-migration.md): disk exhaustion, recovery, database/Grafana migration, and dashboard troubleshooting.
- [Ingestion resilience and API research](docs/research/ingestion-resilience-api.md): deduplication review, proposed local MQTT bridge, PostgREST, and pressure anomaly.
- [MeshCore investigation](docs/research/meshcore-integration.md): deferred integration and architectural alternatives.
- [Sources and remaining gaps](docs/sources.md).

These are historical reconstructions from supplied summaries, not a live infrastructure inventory or a complete installation guide. Reported deployments, suggested commands, and untested proposals are labeled separately. No server changes were made while assembling this documentation.
