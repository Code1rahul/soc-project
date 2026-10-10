# SOC Home Lab — Wazuh, Alpine WSL2, SSH Investigation

A hands-on home lab for learning SOC workflows: collecting authentication telemetry, validating network exposure, investigating suspicious login patterns, and building detections with Wazuh.

> **Scope:** This is a self-owned lab for authorized testing only. It is a learning environment, not a production deployment.

## Current status

- Alpine Linux 3.24.2 runs as an imported WSL2 distribution.
- OpenSSH and BusyBox syslog are configured on Alpine.
- Windows-to-Alpine SSH connectivity was verified.
- A controlled sequence of failed SSH authentications followed by a successful login was recorded in Alpine logs.
- Nmap confirmed SSH as the only open TCP service in the default scan performed.
- A normal SSH packet capture was inspected. It showed network/session metadata, while SSH's encrypted payload did not reveal interactive commands.
- Wazuh 4.14.8 single-node Docker stack is running.
- Wazuh Indexer cluster health returned `green`; the dashboard was opened successfully.
- **Alpine/Wazuh event ingestion is not yet proven.** The dashboard previously showed no registered agents. Visible alert counts must not be interpreted as evidence that Alpine telemetry has arrived.

## Project principles

- Validate current state before installing or changing components.
- Every meaningful step explains both **what we do** and **why**.
- Keep useful technical phases; do not skip them solely to conserve model usage.
- Preserve evidence and record both successes and failures.
- Keep lab data under `D:\CyberLab` where supported.
- Never commit credentials, API tokens, private keys, unsanitized PCAP files, or sensitive host logs.

## Documentation

- [Master Tracker](MASTER_TRACKER.md) — current state, checklist, issues, and handover.
- [Project Report](PROJECT_REPORT.md) — outcomes, evidence, and lessons learned.
- [Lab Operations](LAB_OPERATIONS.md) — how to start, verify, and stop the lab safely.

## Planned milestones

1. Validate stable Wazuh operation and review exposed host ports.
2. Select a supported telemetry path from Alpine to Wazuh.
3. Prove ingestion with a harmless test event.
4. Build a detection for repeated failed SSH authentication.
5. Correlate network and endpoint evidence into an investigation timeline.
6. Publish a sanitized technical walkthrough.

## Security and privacy

The Wazuh dashboard and indexer should remain private to the lab host. Review Docker port bindings before using the lab on shared networks. Use non-default credentials and never publish secrets. Do not expose SSH or vulnerable test services to the public internet.

## Versioning

This repository contains the public, sanitized project documentation. The live environment and evidence files remain on the lab machine unless deliberately sanitized and added for publication.
