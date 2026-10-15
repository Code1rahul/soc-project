# SOC Home Lab — Wazuh, Alpine WSL2, SSH Investigation

A hands-on SOC learning project focused on authentication telemetry, network exposure validation, security monitoring, alert triage, investigation timelines, and detection engineering.

> **Scope:** This is a self-owned learning environment for authorized testing. It is not a production deployment.

## Current status

- Wazuh Manager, Indexer, and Dashboard version 4.14.8 are running.
- Dashboard login and browser API requests have been verified.
- Dashboard and Indexer are restricted to local access for the current validation phase.
- The Alpine WSL endpoint (`AlpineLab`) has returned the expected version and log output after a targeted Windows ownership/access correction. End-to-end event ingestion is still pending.
- The endpoint's log ingestion into Wazuh is **not yet proven**.
- A controlled sequence of failed SSH authentications followed by a successful login was previously observed in the lab logs.
- Nmap established a service baseline, and a normal SSH packet capture was inspected. Encrypted SSH payload did not expose interactive commands.

## Project principles

- Validate current state before changing components.
- Prefer one planned change at a time and verify its result.
- Preserve recovery options; do not reset a working component to fix an unrelated issue.
- Record verified outcomes and explicitly mark unverified assumptions.
- Keep configuration secrets, private keys, credentials, unsanitized PCAPs, personal details, and machine-specific storage information out of public project documentation.

## Documentation

- [Master Tracker](MASTER_TRACKER.md) — verified status, milestones, and blockers.
- [Project Report](PROJECT_REPORT.md) — evidence, findings, and lessons learned.
- [Lab Operations](LAB_OPERATIONS.md) — start, verify, and stop the services.
- [Portable Lab Plan](PORTABILITY_PLAN.md) — portable workflow and recovery principles.

## Next milestones

1. Resolve the Alpine WSL startup failure without destructive changes.
2. Configure the Wazuh Manager to receive syslog from the approved lab endpoint.
3. Prove ingestion with a harmless uniquely marked event.
4. Generate controlled failed SSH authentications and validate detection.
5. Build a timestamped investigation timeline with evidence and limitations.
6. Publish a sanitized technical walkthrough and learning summary.

## Security and privacy

Keep the dashboard and Indexer local-only during development. Open only the minimum necessary ingestion path when endpoint networking is validated. Never publish credentials, private keys, personal identifiers, raw captures, or machine-specific storage details.
