# SOC Home Lab — Project Progress Report

**Snapshot:** 11 October 2026  
**Status:** Wazuh 4.14.8 deployed; indexer reported green; dashboard login confirmed; Alpine telemetry ingestion pending.  
**Project type:** Controlled home lab / SOC investigation case study.

## Executive summary

This lab combines Windows as the analyst workstation, Alpine Linux 3.24.2 on WSL2 as a monitored Linux endpoint, and Wazuh 4.14.8 as the SIEM platform. The foundation now supports SSH connectivity and local syslog collection, a controlled authentication investigation, network-service discovery, and packet-level SSH analysis. Wazuh's Manager, Indexer and Dashboard containers are running. The indexer health API returned green, and the user confirmed interactive dashboard login. The next key milestone is to prove that Alpine events are actually ingested by Wazuh and then build a detection and investigation workflow from observed evidence.

## Why validation comes before installation

The project uses a validation-first method: inspect current state, install only when the tool/component is missing or unsuitable, make one change, and verify the expected result. This avoids duplicate installations and unnecessary configuration changes, narrows troubleshooting, gives a defensible baseline, and creates evidence for the final report. Each step should explain both **what** is being done and **why** it matters.

## Architecture

- **Windows analyst workstation:** PowerShell, Nmap, Wireshark and Docker Desktop.
- **Alpine Linux 3.24.2 on WSL2:** `analyst` user, `doas`, OpenSSH, BusyBox syslogd, and `/var/log/messages`.
- **Wazuh 4.14.8 single-node stack:** Manager, Indexer, and Dashboard via Docker Compose.
- **Storage rule:** lab-specific files and evidence are kept under `D:\CyberLab` wherever supported.

## Completed work

### 1. Lab foundation
- WSL2 updated and configured; Alpine Mini Root Filesystem checksum verified before import.
- Alpine imported to the D: lab directory.
- Network interface, route, DNS, package repositories, and outbound connectivity verified.
- Created the non-root `analyst` account, added the user to `wheel`, and verified privilege escalation with `doas`.
- OpenRC service commands failed because OpenRC had not booted in the imported WSL environment. The lab was adapted to use WSL's boot-command mechanism to start `sshd` and `syslogd`.

### 2. SSH and authentication logging
- OpenSSH listens on TCP/22.
- BusyBox syslog writes to `/var/log/messages`.
- A harmless manual logger event appeared in the log.
- Windows successfully reached Alpine's TCP/22 listener and logged in as `analyst`.
- Three intentionally incorrect SSH password attempts followed by one successful authentication were observed in the server logs.
- This simulated pattern provides a useful SOC triage scenario. It is not proof of a real account compromise.

### 3. Network service baseline
- Nmap 7.99.1 found TCP/22 open with OpenSSH 10.3 reported by service detection.
- The remaining 999 ports in the default TCP scan were reported closed/reset.
- This describes the ports covered by that scan only; it is not a full vulnerability assessment.

### 4. SSH packet investigation
- The user reports inspecting a packet capture of a normal SSH session.
- Packet data provided network/session metadata (addresses, ports, TCP flags, timing, sizes, and SSH protocol activity).
- The interactive commands typed after login were not visible. This is expected because SSH encrypts the session payload.
- The result demonstrates why a SOC needs network evidence **and** endpoint telemetry: PCAP alone cannot establish which commands were executed over a normal encrypted SSH session.
- The actual PCAP and Wireshark screenshot were not attached to this conversation for independent inspection.

### 5. Wazuh deployment
- Pulled Wazuh indexer, manager, and dashboard images version 4.14.8.
- The first deployment attempt failed on host port 1515 because Windows had an excluded TCP port range covering the default manager ports.
- The manager host ports were changed to 2514 and 2515 while the internal container ports remained 1514 and 1515. The stack then started.
- The initial startup showed temporary HTTP 503 responses, cluster-operation timeouts, slow cluster-state writes, and dashboard backend errors.
- A later authenticated indexer health query reported `green`, one node, 22 active primary shards, no unassigned shards, and no pending tasks.
- The dashboard redirected to its login route, and the user subsequently confirmed successful interactive login.
- At the last observed dashboard screenshot, no agents were registered. Visible alert counts should not be treated as evidence of Alpine telemetry arrival.

## Key findings

### Finding A — repeated SSH authentication failures followed by success
**Scenario:** controlled, self-generated lab event.

**Observed sequence:** three failed password attempts for the same lab account from the same WSL-side source address, followed by a successful authentication.

**Interpretation:** a reasonable simulated password-guessing/account-access pattern for detection engineering. ATT&CK reference to validate in the final case: Password Guessing (`T1110.001`). Do not describe the simulation as a real compromise.

### Finding B — SSH encryption limits packet inspection
A packet capture can establish transport/session behavior but normally cannot reveal the encrypted interactive commands. This is expected. To determine what occurred after login, collect endpoint-side execution telemetry and correlate it with SSH authentication/session timestamps.

### Finding C — successful deployment is not the same as verified ingestion
The Wazuh indexer health was green and the dashboard login worked, but no endpoint was registered in the last screenshot. A successful event-ingestion test is still required before claiming the SIEM is monitoring Alpine.

## Current limitations and risks

- Alpine-on-WSL command/process auditing support has not been established; the initial checks found `auditctl` and the Alpine `audit` package absent. This does not prove the kernel lacks audit support.
- No Wazuh agent was registered in the last observed dashboard view.
- Docker published the dashboard/indexer/manager ports on all host interfaces in the last observed Compose output. Review and restrict exposure before using the stack on a shared network; do not blindly restrict agent ports until the chosen telemetry path is known.
- The host has 7.9 GB physical RAM and Docker reports approximately 4.806 GiB available to containers. First startup included indexer timeout and GC-overhead warnings. Monitor performance and keep the stack off when not in use.
- A user-reported PCAP was not attached for independent validation; sanitize PCAP/screenshots before public publication.

## Next steps

1. Verify the dashboard and manager/indexer integration views under normal load.
2. Recheck recent logs and memory/CPU usage.
3. Review published Docker ports and restrict unnecessary external access.
4. Determine a compatible method for Alpine to send SSH/syslog events to Wazuh.
5. Ingest one harmless, unique test event and prove it is searchable.
6. Build a repeated-failed-login detection, test it with controlled attempts, and document false-positive considerations.
7. Test supported endpoint execution telemetry, or record platform limitations and use a compatible monitored endpoint if needed.
8. Build a timeline, correlate evidence, map the simulated behavior to ATT&CK, and prepare response recommendations.
9. Publish sanitized evidence, a GitHub walkthrough, a Medium article, and a LinkedIn summary.

## Evidence inventory

- Windows-to-Alpine TCP/22 connectivity result.
- Successful SSH login and `whoami` output.
- Alpine `/var/log/messages` entries showing failed and accepted authentication.
- Nmap screenshot showing SSH as the only open service in the default TCP scan.
- User-reported SSH PCAP/Wireshark inspection; file itself not attached for this review.
- Docker Compose status, container memory snapshot, indexer green health response, and dashboard login screenshot.

## Publication hygiene

Never publish credentials, tokens, private keys, default passwords, raw unsanitized PCAPs, or logs containing sensitive data. Redact hostnames/IPs where appropriate. The project is a self-owned simulation and should be described transparently as such.
