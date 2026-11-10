# SOC Home Lab — Wazuh, Alpine WSL2, SSH Investigation

A hands-on SOC learning project focused on authentication telemetry, network exposure validation, security monitoring, alert triage, investigation timelines, and detection engineering.

> **Scope:** This is a self-owned learning environment for authorized testing. It is not a production deployment.

## Current status

- Wazuh Manager, Indexer, and Dashboard version 4.14.8 are running.
- Dashboard login and browser API requests have been verified.
- Dashboard and Indexer are restricted to local access for the current validation phase.
- The Alpine WSL endpoint (`AlpineLab`) forwards its normal BusyBox syslog stream to a Windows UDP relay on the WSL gateway (UDP/15144), which forwards to the Manager's published UDP/514 endpoint.
- The Windows 10 user-logon scheduled task and relay supervisor have been registered and started successfully; supervisor logs confirm relay readiness and Alpine service startup.
- End-to-end ingestion is **verified**: a fresh marker generated through Alpine's normal `logger` path appeared in Wazuh `alerts.json` under SSH rule 5710, level 5, and was visible in Wazuh Discover. The direct Alpine-to-Docker-published UDP/514 path remains unresolved; the Windows relay is the proven workaround. Full sign-out/reboot recovery is still pending.
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

1. Conduct actual SSH authentication attempts against AlpineLab and inspect the resulting Wazuh alerts.
2. Validate repeated-failure/brute-force correlation and document its threshold, source-IP correlation, and false-positive limits.
3. Build a timestamped incident timeline and SOC Analyst L1 triage report.
4. Verify full WSL/Windows logon recovery for the scheduled relay supervisor and recheck Indexer health.
5. Publish a sanitized technical walkthrough and learning summary after the operational recovery check.

## Security and privacy

Keep the dashboard and Indexer local-only during development. Open only the minimum necessary ingestion path when endpoint networking is validated. Never publish credentials, private keys, personal identifiers, raw captures, or machine-specific storage details.


## Verified ingestion checkpoint — 22 October 2026

A controlled synthetic SSH-failure event was sent through Alpine's normal local logging path. The foreground Windows relay logged a forwarded datagram, and the exact marker `SOC_LAB_NORMAL_SYSLOG_RELAY_20261022_210527` was found in Wazuh `alerts.json` with rule 5710, level 5. The original Manager configuration was preserved; `logall_json=no` remains the intended setting. The relay and Alpine syslog destination are currently temporary runtime settings. Do not describe the setup as restart-persistent until startup behavior is validated.

## Verified relay automation and Dashboard checkpoint — 10 November 2026

The Windows 10-specific installer updated Alpine's startup script to use the dynamic WSL gateway and UDP/15144, registered the `SOC Lab Syslog Relay` user-logon scheduled task, and created the supervisor. The task was started successfully during the session. Supervisor and relay logs confirm the listener, Alpine services, and forwarded datagrams.

Marker `SOC_LAB_PERSISTENCE_20261110_214100` was generated through Alpine's normal `logger` path, raised SSH rule 5710 (level 5), and was visible in Wazuh Discover. Routine JSON archiving remains disabled. A full Windows sign-out/reboot recovery test has not yet been performed.