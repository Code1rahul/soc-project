# Master Tracker — SOC Home Lab

**Updated:** 14 October 2026  
**Repository:** `Code1rahul/soc-project`  
**Working lab path:** `E:\CyberLab`  
**Current stage:** Portable-drive handover to new Windows PC; Docker Desktop CLI/engine availability is pending on this host.

## Current host / portable-drive handover — 14 October 2026

This lab is now being used on a different Windows PC with the same portable drive. This is **not a migration**: keep the drive mounted as `E:` and continue using the existing files.

- Portable lab drive: `E:\CyberLab`; reported free space 422.4 GB.
- Internal `D:` drive is separate and had 55.5 GB free at the check. Do not redirect lab data to it.
- Current PC: Intel Core i3 4th generation, 4 logical processors (reported 3.5 GHz), 16 GB RAM, NVIDIA Quadro P400 with 2 GB dedicated VRAM. GPU acceleration is not needed for Wazuh.
- `.wslconfig` is host-specific and remains under `%USERPROFILE%\.wslconfig`; current configured values are `memory=8GB`, `processors=4`, `swap=4GB`, `swapFile=E:\\CyberLab\\WSL\\wsl-swap.vhdx`, and `kernelCommandLine="sysctl.vm.max_map_count=262144"`.
- WSL registry inspection confirmed the currently registered `Alpine` distribution has `BasePath: D:\WSL\Alpine` and VHDX `D:\WSL\Alpine\ext4.vhdx`. This appears to be the older/local installation the user had installed previously; it reports Alpine `3.23.3` and `vm.max_map_count=262144`. A separate candidate VHDX exists at `E:\CyberLab\WSL\Alpine\ext4.vhdx`, but it is not the currently registered `Alpine` base path. Previous project setup recorded Alpine `3.24.2`. The VHDX sizes/timestamps have now been compared; see the dated comparison section. Do not unregister, overwrite, re-import, or upgrade either instance yet.
- `E:\\CyberLab\\Tools\\DockerDesktop\\Docker Desktop.exe` exists, and `E:\\CyberLab\\DockerData` contains `disk` and `main` directories.
- On this PC, `Get-Command docker` returned no result and `docker` was not recognized. `wsl --list --verbose` listed Alpine only; Docker Desktop has not yet been confirmed running/registered on this host.
- Do not assume the presence of Docker data folders proves the old images and volumes are usable. Start the existing Docker Desktop application and verify the effective disk-image location and containers before pulling images, changing data location, or initializing another backend.
- Always close Docker Desktop, run `wsl --shutdown`, and wait for it to stop before physically disconnecting the portable drive. Never unplug E: while its WSL VHDX, swap VHDX, or Docker disk is in use.

Next host-resume sequence:
1. Ensure the portable drive is mounted as `E:`.
2. Launch `E:\CyberLab\Tools\DockerDesktop\Docker Desktop.exe`.
3. Wait for Docker Desktop to finish starting, then check `Get-Command docker`, `wsl --list --verbose`, and `docker info`.
4. In Docker Desktop Settings, verify the WSL 2 backend and inspect the disk image location. Do not change the data location until the existing data is identified.
5. Inspect metadata for both `D:\WSL\Alpine\ext4.vhdx` and `E:\CyberLab\WSL\Alpine\ext4.vhdx`; preserve both until the intended distro is determined.
5. From `E:\CyberLab\Projects\wazuh-docker\single-node`, inspect `docker compose ps -a`; do not run `down -v`.
6. Recheck indexer health and endpoint integration; the earlier machine's green cluster result does not automatically prove current-host health.

## Working rules

- **Validation-first:** inspect existing state before installing or modifying anything.
- **Explain the why:** every step must state the action, purpose, expected result, and evidence.
- **Change one variable at a time** and verify it before proceeding.
- Do not mark a checkpoint complete without observed output or user-confirmed evidence.
- Do not skip or compress a technically relevant phase only to conserve model usage.
- Normal ChatGPT is the default; reserve Thinking mode for major design choices or difficult analysis if the user chooses to switch.
- Use current official documentation for version-dependent commands.
- Keep all lab artifacts on `E:\CyberLab` where practical; record exceptions for Windows-managed paths.
- This public repository must not contain passwords, tokens, private keys, raw unsanitized PCAPs, or sensitive system details.

## Architecture

```text
Windows analyst workstation
  ├─ PowerShell, Nmap, Wireshark
  ├─ Docker Desktop / WSL2 backend
  └─ E:\CyberLab\PCAP, Evidence, Projects, Reports
         │
         ├─ Alpine Linux 3.24.2 on WSL2
         │    ├─ analyst user + doas
         │    ├─ OpenSSH
         │    ├─ BusyBox syslogd
         │    └─ /var/log/messages
         │
         └─ Wazuh 4.14.8 single-node stack
              ├─ Manager
              ├─ Indexer
              └─ Dashboard
```

## Progress checklist

### Phase 1 — Lab foundation
- [x] WSL2 installed/updated and default version configured.
- [x] Alpine 3.24.2 x86_64 downloaded, checksum verified, and imported to D:.
- [x] Alpine networking, route, DNS, package repository access, and outbound connectivity verified.
- [x] `analyst` user created and membership in `wheel` verified.
- [x] `doas` configured and privilege elevation successfully tested.
- [x] Imported WSL startup adapted to WSL boot command because OpenRC service control failed in this model.

### Phase 2 — SSH, logs, and network evidence
- [x] OpenSSH and BusyBox syslog configured.
- [x] `/var/log/messages` records test events.
- [x] Windows-to-Alpine SSH TCP connectivity verified.
- [x] Normal SSH login as `analyst` verified and related session logs observed.
- [x] Three intentional failed SSH authentications followed by successful authentication recorded in the private lab.
- [x] Nmap default TCP scan completed; SSH was the only open TCP service found in that scan.
- [x] Phase 2F packet-capture/inspection reported complete by the user.
- [x] Documented the key PCAP limitation: encrypted SSH payload does not normally reveal interactive commands.
- [ ] Attach sanitized PCAP/screenshot to the local evidence folder and independently validate it before final portfolio release, if not already done.
- [ ] Phase 2G: test what endpoint command/process telemetry is supported in Alpine-on-WSL; choose a compatible method and validate a benign event.

### Phase 3 — Wazuh SIEM
- [x] Docker Desktop and Docker Compose installed and engine verified.
- [x] Wazuh 4.14.8 indexer, manager, and dashboard images pulled.
- [x] Resolved Windows excluded-port conflict by mapping host ports 2514/2515 to manager container ports 1514/1515.
- [x] All three Wazuh containers started.
- [x] Indexer health API returned green, one node, 22 active primary shards, zero unassigned shards, and zero pending tasks.
- [x] Dashboard HTTPS endpoint responded with a login redirect.
- [x] User confirmed dashboard loaded and interactive login worked.
- [ ] Recheck recent indexer and manager logs for persistent errors/GC pressure.
- [ ] Review and restrict Docker host port bindings; do not expose the indexer to a network unnecessarily.
- [ ] Confirm what the dashboard's initial alert counts represent; do not attribute them to Alpine without evidence.
- [ ] Choose and validate a supported Alpine-to-Wazuh event ingestion path.
- [ ] Prove ingestion using a harmless unique test event and search for it in Wazuh.
- [ ] Implement a detection for repeated failed SSH logins, then test it against controlled events.
- [ ] Document thresholds, false positives, alert evidence, and limitations.

### Phase 4 — Investigation
- [ ] Build a timestamped incident timeline.
- [ ] Correlate SSH auth events, endpoint activity telemetry, and packet metadata.
- [ ] Extract relevant indicators and map the simulated behavior to MITRE ATT&CK.
- [ ] Assess severity and explain why it is a simulation, not a confirmed real compromise.
- [ ] Document response/containment recommendations.

### Phase 5 — Publish
- [x] Create public sanitized Markdown project documentation.
- [ ] Add redacted screenshots and selected sanitized evidence.
- [ ] Final review for credentials, private keys, tokens, sensitive IPs/hostnames, and unrelated traffic.
- [ ] Publish Medium walkthrough and LinkedIn summary after the evidence is reproducible.

## Verified observations

### SSH authentication
- Three intentionally generated failed password attempts were followed by one successful authentication for the same lab account.
- The source address seen in Alpine was the WSL NAT-side host/gateway address. Do not interpret it as an external attacker address.
- This is a controlled simulation, not evidence of a real compromise.
- Potential ATT&CK reference: Password Guessing, `T1110.001`; validate mapping during final analysis.

### Nmap
- Nmap 7.99.1 observed SSH on TCP/22; the other 999 TCP ports in the default scan were reported closed/reset.
- This is a service baseline, not a comprehensive vulnerability assessment.

### Packet evidence
- User reports that a normal SSH capture was inspected.
- TCP/session metadata was visible; interactive commands were not readable due to SSH encryption.
- Actual PCAP/screenshot were not attached for independent inspection during the chat.

### Wazuh
- Images: `wazuh/wazuh-indexer:4.14.8`, `wazuh/wazuh-manager:4.14.8`, `wazuh/wazuh-dashboard:4.14.8`.
- Current compose folder: `E:\CyberLab\Projects\wazuh-docker\single-node`.
- Indexer cluster returned `green` at the last health check.
- Dashboard login was confirmed by the user.
- Initial indexer logs showed cluster event timeouts and slow cluster-state writes; follow-up showed a GC-overhead warning. Keep monitoring.
- No agents were registered in the last observed dashboard screenshot; actual Alpine event ingestion remains unproven.
- Docker host published services on all interfaces at the last `docker compose ps` check. Review before using on shared networks.

## Issue register

| ID | Status | Issue / observation | Learning or resolution |
|---|---|---|---|
| ISS-001 | Resolved | `adduser -D` did not prompt for password | Password was configured separately from a privileged shell. |
| ISS-002 | Resolved | Password update from analyst shell reported privilege/SUID requirement | Initial password setup was performed in the correct privileged context. |
| ISS-003 | Resolved | OpenRC failed under imported WSL startup | A controlled WSL boot command starts SSH and syslog. |
| ISS-004 | Resolved | First doas attempt failed authentication | Re-entered credentials and verified `doas id` succeeded. |
| ISS-005 | Resolved | Windows excluded TCP range blocked default Wazuh manager host ports | Host ports changed to 2514/2515 while container ports remained 1514/1515. |
| OBS-001 | Informational | First SSH connection prompted for host fingerprint | Expected first-contact behavior; host key was accepted for the lab. |
| OBS-002 | Complete | Failed SSH logins followed by success | Controlled simulated detection scenario; not a real compromise. |
| OBS-003 | Complete | Nmap found only SSH open in default TCP scan | Establishes intended service baseline for that scan. |
| OBS-004 | Complete by user report | PCAP did not expose interactive SSH commands | Expected encryption; endpoint telemetry is needed for post-login activity. |
| OBS-005 | Open | Wazuh first start had indexer timeouts and slow cluster-state writes | Indexer later reported green; monitor and validate integration. |
| OBS-006 | Open | Wazuh dashboard initially showed no agents | Dashboard counts are not evidence of Alpine log ingestion; integration is next. |

## Next session handover

1. Start Docker Desktop; enter the Wazuh Compose directory.
2. Run `docker compose ps`, `docker stats --no-stream`, and recent service logs.
3. Check indexer cluster health using configured credentials (never copy the password/token into chat or docs).
4. Review host port bindings; restrict the indexer/dashboard to appropriate host access while keeping required telemetry path available.
5. Validate the dashboard's manager/indexer views.
6. Select a compatible path for Alpine SSH/syslog events.
7. Send a uniquely identifiable benign test event and prove it is searchable before building detections.

See [Lab Operations](LAB_OPERATIONS.md) for start, verification, and stop commands.


## Alpine VHDX comparison — 14 October 2026

Observed PowerShell metadata:

| Path | Size reported | Created | Last written | Interpretation |
|---|---:|---|---|---|
| `D:\WSL\Alpine\ext4.vhdx` | 11.59 GB | 28 Mar 2026 20:24:57 | 14 Oct 2026 21:08:40 | Strong candidate for the older long-used local Alpine installation. It is the distro currently registered as `Alpine`. |
| `E:\CyberLab\WSL\Alpine\ext4.vhdx` | 0.10 GB | 4 Oct 2026 22:55:25 | 11 Oct 2026 02:15:58 | Strong candidate for the newer portable-drive project import. It is not currently registered under the `Alpine` name. Contents/version must be verified before relying on this inference. |

Recommended next step: preserve both disks and register the E: candidate as a second distro named `AlpineLab` using WSL's documented `wsl --import-in-place` command. Then verify its Alpine version, user, `/etc/wsl.conf`, SSH, and syslog. Do not run `wsl --unregister Alpine`, and do not delete either VHDX.


## Portability design and current state — 14 October 2026

**This section supersedes earlier lines above that described the E: Alpine VHDX as not yet registered.** The user successfully ran `wsl --import-in-place AlpineLab "E:\CyberLab\WSL\Alpine\ext4.vhdx"`. The result is registered as `AlpineLab` (WSL 2), reports Alpine `3.24.2`, runs as `analyst` (UID 1000, member of `wheel`), and has the expected `/etc/wsl.conf` with default user and boot script.

The desired future state is a **portable lab kit**, not one all-in-one container and not the entire external drive used as Docker storage. Keep Compose files, the Alpine VHDX, sanitized evidence, scripts, and backups under `E:\CyberLab`. Unrelated personal files on E: are out of scope.

- [ ] Launch Docker Desktop on this host and prove `docker info` works before changing Docker data or Compose state.
- [ ] Inventory `E:\CyberLab\DockerData\disk` and `main`; do not delete/reinitialize them.
- [ ] Inspect existing Wazuh containers and volumes before bringing anything down or recreating it.
- [ ] Create tested Compose-based start/stop/preflight scripts; never use `docker compose down -v` for routine shutdown.
- [ ] Add a backup/restore plan for Wazuh data that needs to survive host changes.
- [ ] Evaluate a dedicated SSH-test endpoint container later, after the existing telemetry flow is working.

Docker Desktop still needs to be installed/configured on each target Windows host. An external Docker data root is an advanced option, not a guarantee of plug-and-play operation. See [Portable SOC Lab Plan](PORTABILITY_PLAN.md).
