# Sources and documentation gaps

Compiled on 2026-10-23 from three summaries supplied by the project owner. Original full conversations were not retrieved. The summaries are secondary evidence, and no live server inspection or new upstream code audit was performed.

| ID | Supplied source | Coverage | Limitations |
| --- | --- | --- | --- |
| S1 | Attachment: Texto colado.txt | April 2026 VPS migration, disk incident, Grafana | Ends mid-sentence in section 8.3; missing ending not reconstructed |
| S2 | Attachment: Texto colado (2).txt | Exporter source review, resilience, API, pressure anomaly | No upstream commit pinned; deployed schema not checked |
| S3 | MeshCore Integration Investigation, pasted in this conversation | Alternatives and decision to defer MeshCore | Research only; candidate tools and capabilities unverified |

## Original conversation links

Four unique links were supplied. Their contents were inaccessible during the previous attempt; no one-to-one mapping to S1–S3 has been established.

- https://chatgpt.com/share/6adc06c1-aa28-83e9-ad83-1afe8db04344
- https://chatgpt.com/share/6adc06e2-9c04-83e8-800b-5b1b24409d48
- https://chatgpt.com/share/6adc06f4-dab8-83e8-ac0d-8b7bd57adb77
- https://chatgpt.com/share/6adc0654-f550-83e8-8268-8458957445c5

## Reading status labels

“Reported operational” means the source summary records a working result; it does not imply a check today. “Source review” describes findings attributed to upstream code inspected in the original chat. “Proposed” and “unverified” are not deployments. Commands in historical documents include both recorded actions and advice, with surrounding notes distinguishing them.

The one-VPS architecture described generally by S2/S3 is supplemented by S1's explicit post-migration split: MQTT on the old server, exporter/database/Grafana on the new server. This is the most specific supplied deployment account, not proof of present topology.

## Remaining collection work

- Obtain the initial node deployment summary, including RF-only reachability tests, exact node settings, wiring, and firmware versions.
- Recover S1's missing ending, if available.
- Capture sanitized Compose, broker, Caddy, and firewall configuration and pinned software versions.
- Verify effective retention, Docker logging, journald limits, and any scheduled cleanup.
- Verify deployed schema and exporter revision before designing replay/idempotency changes.
- Record backups and a successful restore verification; the migration summary lacks a complete final restore transcript.
- Investigate BTL1 pressure outliers using raw telemetry and timestamps.

This repository documents Clima Sanca's use of [tcivie/meshtastic-metrics-exporter](https://github.com/tcivie/meshtastic-metrics-exporter). It does not claim that upstream software was created for this project. No upstream issue or PR submission is established by these sources.
