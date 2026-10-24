# Ingestion resilience, API, and anomaly investigation

> Historical source: supplied conversation summary S2. Upstream findings below were reported by that conversation, with no commit pinned. They have not been revalidated against current upstream code or the deployed database.

Scope: Results and decisions from this conversation only. Suitable for incorporation into the `clima-sanca` GitHub repository.

This conversation documented an existing Meshtastic-based weather telemetry deployment for São Carlos, SP, investigated its ingestion and deduplication behavior, and explored future improvements involving MQTT bridging and a public API.

No installations, configuration changes, commits, or deployments were performed during this conversation. The existing infrastructure was described as operational, and upstream source code was inspected.

## 1. Confirmed infrastructure

| Component            | Confirmed information                                                                           |
| -------------------- | ----------------------------------------------------------------------------------------------- |
| Weather nodes        | CSC1 and BTL1                                                                                   |
| Hardware platform    | Meshtastic-compatible ESP32/LoRa nodes                                                          |
| Environmental sensor | BME680 used in the weather-node project                                                         |
| MQTT broker          | Mosquitto on an internet-accessible VPS                                                         |
| Telemetry processor  | `tcivie/meshtastic-metrics-exporter`                                                            |
| Database             | PostgreSQL with TimescaleDB                                                                     |
| Visualization        | Private Grafana instance                                                                        |
| Public dashboards    | Two public dashboards exposed from Grafana                                                      |
| Public URL           | [https://clima.falafel.com.br/sanca](https://clima.falafel.com.br/sanca) |
| Historical retention | 30 days in the inspected upstream TimescaleDB initialization configuration                      |

The reported flow is Meshtastic nodes → internet Mosquitto → exporter → TimescaleDB → Grafana → public dashboards.

The deployment was already operational when this conversation began.

The conversation does not establish the exact firmware versions, container versions, service names, host directories, or complete node configurations.

## 2. Upstream project and source files

Upstream repository:

[https://github.com/tcivie/meshtastic-metrics-exporter](https://github.com/tcivie/meshtastic-metrics-exporter)

The following files were inspected:

| File                               | Purpose                                                                |
| ---------------------------------- | ---------------------------------------------------------------------- |
| `main.py`                          | MQTT connection, subscriptions, message handling, packet deduplication |
| `exporter/db_handler.py`           | Database inserts and queries                                           |
| `exporter/processor/processors.py` | Meshtastic payload decoding and telemetry processing                   |
| `docker/timescaledb/init.sql`      | Database schema, hypertables, triggers, retention policies, and views  |

These are paths within the upstream GitHub repository, not confirmed filesystem paths on the deployed VPS.

### Database objects reported by the source review

The inspected initialization SQL defines:

- `messages` — temporary packet deduplication
- `node_details` — node metadata
- `node_configurations` — node configuration and telemetry status
- `environment_metrics` — environmental telemetry
- `device_metrics` — device telemetry
- `air_quality_metrics` — air-quality measurements
- `power_metrics` — power measurements
- `mesh_packet_metrics` — packet metadata
- `node_telemetry` — view combining latest node measurements

The schema also defines other supporting tables and functions.

The environment telemetry table includes fields for temperature, relative humidity, barometric pressure, gas resistance, IAQ, light measurements, wind measurements, and other environmental values.

The upstream initialization SQL applies 30-day retention policies to telemetry hypertables.

Important limitation: These findings describe the inspected upstream source. The deployed database schema and retention settings were not independently verified against the running instance.

## 3. Deduplication behavior reported by the source review

The exporter already performs packet-level deduplication.

In `main.py`, incoming MQTT messages are decoded into Meshtastic service envelopes. The exporter extracts `packet.id` and checks the `messages` table before processing the packet.

The relevant operations are conceptually:

```
SELECT id FROM messages WHERE id = %s;
```

and:

```
INSERT INTO messages (id, received_at)
VALUES (%s, NOW())
ON CONFLICT (id) DO NOTHING;
```

The database initialization defines:

```
CREATE TABLE IF NOT EXISTS messages
(
    id          TEXT PRIMARY KEY,
    received_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

A trigger deletes entries older than one minute.

### Limitations reported by the source review

1. Deduplication uses `packet.id` alone, without including the sender's node identity.
2. The deduplication record expires after approximately one minute.
3. The `environment_metrics` hypertable does not store the originating packet ID.
4. Environmental measurements are inserted without a unique constraint on packet identity.
5. The exporter uses `datetime.now()` when inserting measurements rather than preserving the original observation timestamp.

Consequently, a previously processed packet replayed after the deduplication window can generate another environmental telemetry row.

The actual deduplication window is not a strict sliding 60-second guarantee: expired records are removed by an insert-triggered cleanup mechanism.

### Additional technical consideration

The check-then-insert sequence in `main.py` is not an atomic claim-and-process operation. Two concurrent consumers could potentially observe the same packet as unseen before one inserts its ID.

This is a potential implementation issue, not a concurrency bug demonstrated during this conversation.

## 4. Proposed upstream contribution

Status: Discussed and drafted; not submitted.

We prepared a GitHub feature request proposing stronger idempotency for telemetry storage.

Proposed title:

Improve idempotency of telemetry inserts using Meshtastic packet identity

The motivation is to prevent duplicate historical measurements when messages are replayed after the current one-minute deduplication period.

Possible implementation elements:

- Preserve packet identity alongside telemetry observations.
- Include sender identity in the deduplication key.
- Add database-level uniqueness protection.
- Use conflict-aware inserts.
- Preserve compatibility with TimescaleDB hypertable constraints.

The exact implementation was deliberately left open for discussion with the maintainer.

A significant consideration is that TimescaleDB requires unique indexes on hypertables to include all partitioning dimensions. Therefore, a straightforward unique constraint on `(node_id, packet_id)` would not be sufficient for the existing time-partitioned table.

The issue draft also mentions the project's real-world use at:

[https://clima.falafel.com.br/sanca](https://clima.falafel.com.br/sanca)

No upstream issue number or pull request was created in this conversation.

## 5. Proposed local MQTT bridge

Status: Architectural exploration only. Not implemented.

We discussed introducing a local MQTT broker on the hackerspace LAN.

Proposed flow: CSC1 → local Mosquitto with persistent queue → internet Mosquitto → existing exporter → existing TimescaleDB → Grafana.

The objective is to tolerate temporary internet outages without requiring a second telemetry database.

### Architectural preferences

- Use Mosquitto broker-to-broker bridging.
- Use QoS 1 for forwarded telemetry.
- Enable broker persistence.
- Allow messages to queue locally during internet outages.
- Forward queued messages when connectivity returns.
- Keep the existing internet TimescaleDB as the historical database.

Example Mosquitto configuration discussed:

```
persistence true
persistence_location /var/lib/mosquitto/
```

Example bridge configuration:

```
connection cloud
address <remote-broker-host>:1883

topic sensors/# out 1
```

These are illustrative examples, not deployed configuration. The `sensors/#` topic was not verified as the actual Meshtastic topic.

The following remain untested:

- Queue persistence across broker restarts
- Offline queue limits
- Recovery after extended internet outages
- MQTT QoS compatibility between node, local broker, and remote broker
- Duplicate behavior after delayed forwarding
- Meshtastic firmware compatibility with the proposed local broker topology

A local PostgreSQL database was considered but not selected as the preferred initial approach.

## 6. Proposed telemetry API

Status: Architectural decision in principle; no implementation.

The goal is to make the existing weather measurements available to a separate hackerspace API aggregator.

We selected PostgREST as the likely simplest option for exposing curated TimescaleDB data.

Proposed flow: TimescaleDB → restricted PostgREST provider → hackerspace FastAPI aggregator → public consumers.

### Agreed design principles

Source of truth: The existing TimescaleDB remains authoritative for weather observations.

Provider API: PostgREST exposes a limited, read-only subset of database views or functions.

Access control: The provider API can be restricted to a known IP address or authenticated consumer.

Aggregator: The separate FastAPI service owns the public API contract and may normalize field names, units, timestamps, and metadata.

Avoid permanent duplication: The aggregator should not initially maintain a second database of weather observations.

Caching: Short-lived response caching is acceptable if needed.

Grafana: Remains the visualization layer rather than becoming the data API.

Possible API resources discussed include station metadata, latest measurements, historical time series, and daily aggregates.

No API endpoints, SQL views, PostgREST configuration files, or access-control rules were actually created.

## 7. Pressure anomaly investigation

Status: Anomaly observed; cause unresolved.

A Grafana screenshot showed two abrupt downward excursions in BTL1 atmospheric-pressure measurements.

One highlighted observation was:

| Property            | Value               |
| ------------------- | ------------------- |
| Timestamp displayed | 2026-04-20 20:56:19 |
| BTL1 pressure       | 916.88 hPa          |
| CSC1 pressure       | 922.15 hPa          |
| Difference          | 5.27 hPa            |

BTL1's pressure curve dropped sharply and recovered almost immediately, while CSC1 remained comparatively smooth.

The chart suggested isolated anomalous measurements rather than a sustained atmospheric-pressure change.

### Hypotheses considered

- Transient sensor reading error
- I²C communication problem
- Power instability
- Firmware or measurement-cycle issue
- Data parsing or conversion error
- Delayed message insertion or timestamp mismatch

A sensor/acquisition glitch was considered plausible, but none of these explanations was confirmed.

No raw MQTT payload, sensor log, or database query was used to determine the cause.

### Suggested investigation

Compare the anomalous pressure observations with temperature, humidity, and device telemetry from the same period.

If possible, inspect original MQTT payloads to determine whether the anomaly originated at the sensor or during processing.

We also discussed preserving raw measurements and potentially applying visualization-level anomaly filtering rather than silently discarding observations during ingestion.

No filters were implemented.

## 8. Commands, paths, and service configuration

This conversation did not record completed shell commands or installation procedures.

The following concrete paths and names appeared:

| Item                               | Context                                |
| ---------------------------------- | -------------------------------------- |
| `main.py`                          | Upstream MQTT ingestion entry point    |
| `exporter/db_handler.py`           | Upstream database handler              |
| `exporter/processor/processors.py` | Upstream payload processors            |
| `docker/timescaledb/init.sql`      | Upstream database initialization       |
| `/var/lib/mosquitto/`              | Suggested broker persistence directory |
| `environment_metrics`              | TimescaleDB environmental measurements |
| `messages`                         | Temporary deduplication table          |
| `node_telemetry`                   | Latest telemetry view                  |

The example SQL statements and Mosquitto configuration were inspected or discussed, not executed.

No passwords, tokens, connection strings, or private host addresses are included here.

## 9. Outstanding questions

| Question                                                                             | Status                                  |
| ------------------------------------------------------------------------------------ | --------------------------------------- |
| Should packet-level idempotency be added to telemetry storage?                       | Proposed; not submitted |
| What constitutes a safe long-term Meshtastic packet identity?                        | Requires investigation                  |
| How should uniqueness be implemented with TimescaleDB hypertables?                   | Unresolved                              |
| Should observation time be preserved separately from ingestion time?                 | Unresolved                              |
| Can Mosquitto reliably buffer the expected outage duration?                          | Not tested                              |
| Can CSC1 publish through the proposed local broker without losing existing behavior? | Not tested                              |
| What caused BTL1's isolated pressure drops?                                          | Unresolved                              |
| What exact SQL views should PostgREST expose?                                        | Not designed                            |
| What authentication/network restriction should protect PostgREST?                    | Not selected                            |
| Does the deployed database match the inspected upstream schema?                      | Not verified                            |

## 10. Conclusions and documentation implications

The most important concrete result is that the existing MQTT → exporter → TimescaleDB → Grafana pipeline is operational and already serving public historical weather measurements.

This conversation also established that the upstream exporter performs short-lived packet deduplication but does not enforce long-term idempotency on environmental telemetry rows.

Two possible improvements emerged:

1. Improve the existing ingestion pipeline's resistance to delayed duplicate messages, preferably in collaboration with the upstream maintainer.
2. Add a local MQTT broker with persistent bridging to improve resilience during internet outages without duplicating the historical database.

Separately, PostgREST was identified as a promising lightweight provider API for the future hackerspace FastAPI aggregator.

Repository documentation should present the current deployment as implemented, while keeping MQTT bridging, PostgREST, and telemetry idempotency changes clearly marked as proposals.

The conversation contains enough information to document the architecture and technical rationale, but not enough to reconstruct the installation from scratch. A reproducible deployment guide would require the actual VPS configuration, service definitions, database configuration, and node firmware settings.