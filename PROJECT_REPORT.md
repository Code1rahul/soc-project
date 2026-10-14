# SOC Home Lab — Project Progress Report

**Snapshot:** 14 October 2026  
**Status:** Wazuh 4.14.8 deployed; indexer reported green; dashboard login confirmed; Alpine telemetry ingestion pending.  
**Project type:** Controlled home lab / SOC investigation case study.

## Current host handover — 14 October 2026

The same portable lab drive is now attached to a different Windows computer and is mounted as `E:`. No drive migration is intended. The portable drive reports 422.4 GB free; internal `D:` reports 55.5 GB free and is not the lab storage target.

This host reports an Intel Core i3 4th generation CPU (4 logical processors, stated 3.5 GHz), 16 GB RAM, and an NVIDIA Quadro P400 with 2 GB dedicated VRAM. WSL global configuration is set to 8 GB memory maximum, four processors, 4 GB swap, with the swap path configured on E:.

**Environment discrepancy clarified:** WSL registry inspection shows the currently registered `Alpine` points to `D:\WSL\Alpine\ext4.vhdx` and reports 3.23.3. A separate `E:\CyberLab\WSL\Alpine\ext4.vhdx` exists but is not the currently registered distro path. Previous project sessions reported Alpine 3.24.2. The E: VHDX may be the project instance, but its identity is not yet verified; compare file metadata before changing registration. Do not unregister, overwrite, or re-import either VHDX.

Docker Desktop's executable exists at `E:\CyberLab\Tools\DockerDesktop\Docker Desktop.exe`, and `E:\CyberLab\DockerData` contains `disk` and `main` folders. However, PowerShell did not resolve the `docker` command and WSL listed only Alpine. Therefore the previous Wazuh deployment is not yet confirmed available on this host. Start Docker Desktop and confirm the existing data image and containers before taking any cleanup or reinstall action.

## Executive summary

This lab combines Windows as the analyst workstation, Alpine Linux 3.24.2 on WSL2 as a monitored Linux endpoint, and Wazuh 4.14.8 as the SIEM platform. The foundation now supports SSH connectivity and local syslog collection, a controlled authentication investigation, network-service discovery, and packet-level SSH analysis. Wazuh's Manager, Indexer and Dashboard containers are running. The indexer health API returned green, and the user confirmed interactive dashboard login. The next key milestone is to prove that Alpine events are actually ingested by Wazuh and then build a detection and investigation workflow from observed evidence.

## Why validation comes before installation

The project uses a validation-first method: inspect current state, install only when the tool/component is missing or unsuitable, make one change, and verify the expected result. This avoids duplicate installations and unnecessary configuration changes, narrows troubleshooting, gives a defensible baseline, and creates evidence for the final report. Each step should explain both **what** is being done and **why** it matters.

## Architecture

- **Windows analyst workstation:** PowerShell, Nmap, Wireshark and Docker Desktop.
- **Alpine Linux 3.24.2 on WSL2:** `analyst` user, `doas`, OpenSSH, BusyBox syslogd, and `/var/log/messages`.
- **Wazuh 4.14.8 single-node stack:** Manager, Indexer, and Dashboard via Docker Compose.
- **Storage rule:** lab-specific files and evidence are kept under `E:\CyberLab` wherever supported.

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


## WSL registration clarification

The current `Alpine` registration points to `D:\WSL\Alpine\ext4.vhdx`; this is likely the older local distro mentioned by the user. The candidate `E:\CyberLab\WSL\Alpine\ext4.vhdx` exists separately and is not currently registered as `Alpine`. The next step is a read-only comparison of both file sizes and timestamps; do not make destructive changes until the correct lab instance is identified.


## Alpine VHDX comparison — 14 October 2026

Observed PowerShell metadata:

| Path | Size reported | Created | Last written | Interpretation |
|---|---:|---|---|---|
| `D:\WSL\Alpine\ext4.vhdx` | 11.59 GB | 28 Mar 2026 20:24:57 | 14 Oct 2026 21:08:40 | Strong candidate for the older long-used local Alpine installation. It is the distro currently registered as `Alpine`. |
| `E:\CyberLab\WSL\Alpine\ext4.vhdx` | 0.10 GB | 4 Oct 2026 22:55:25 | 11 Oct 2026 02:15:58 | Strong candidate for the newer portable-drive project import. It is not currently registered under the `Alpine` name. Contents/version must be verified before relying on this inference. |

Recommended next step: preserve both disks and register the E: candidate as a second distro named `AlpineLab` using WSL's documented `wsl --import-in-place` command. Then verify its Alpine version, user, `/etc/wsl.conf`, SSH, and syslog. Do not run `wsl --unregister Alpine`, and do not delete either VHDX.
