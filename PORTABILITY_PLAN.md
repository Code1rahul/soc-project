# Portable SOC Lab Plan

**Updated:** 14 October 2026  
**Working root:** `E:\CyberLab`  
**Goal:** Carry the SOC lab between compatible Windows PCs without touching unrelated personal files on the same external drive.

## Recommendation

Build a **portable lab kit**, not one giant container. Keep one Compose project for the SIEM and add lab services as separate containers only when their telemetry path is designed and tested. Docker Compose is the repeatable recipe; the external drive stores that recipe, the Alpine WSL virtual disk, evidence, and backups.

This is not literal plug-and-play on an arbitrary PC. Each host still needs Windows/WSL 2 and a correctly installed and working Docker runtime. Docker Desktop is not a self-contained portable app merely because an executable is stored on E:.

## What travels with the drive

- `Projects\wazuh-docker\single-node\`: Compose file and stack configuration.
- `WSL\Alpine\ext4.vhdx`: the validated Alpine 3.24.2 lab endpoint. Register it on each new Windows host with WSL's import-in-place command; do not register the same VHDX on two hosts at the same time.
- `Evidence\`, `PCAP\`, and `Reports\`: sanitized artifacts and investigation outputs.
- A dedicated `Backups\Docker\` directory: image exports, configuration backups, and volume-backup archives when required.
- Start/stop scripts and a host preflight checklist, to be added after the current Docker engine is verified.

The external drive may contain unrelated personal files. Leave those files and folders untouched. All lab-specific paths and scripts must refer to `E:\CyberLab`, not to the root of E:.

## What must be ready on each Windows host

- WSL 2, hardware virtualization, and a supported Windows build.
- Docker Desktop installed and started, with the Linux-container engine responding to `docker info`.
- The external drive mounted as `E:`.
- `AlpineLab` registered from `E:\CyberLab\WSL\Alpine\ext4.vhdx` (one-time registration per host).
- Compose configuration and any required environment files present, with secrets stored locally and excluded from Git.

A per-user `.wslconfig` file remains under that Windows user's profile; it is not automatically carried by the external drive. Host-specific memory and swap settings must be checked on each PC.

## Docker storage options

| Option | Benefits | Trade-offs | Recommendation |
|---|---|---|---|
| **Portable recipe + backups** | Easier recovery; external drive holds the project and important evidence; engine data can be recreated | Images may need pulling again; selected Docker volumes must be backed up/restored to carry SIEM state | **Use this first** |
| Docker's WSL data root on the external drive | Can keep Docker-managed image/container storage on E: | Still requires Docker Desktop installed/configured on each host; USB disconnection or drive-letter changes can break the runtime; must be configured through supported installation/settings, not by manually moving files | Evaluate only after auditing existing data |
| Bind-mount every database/index path directly onto the Windows external drive | Visible host files | Linux database/indexer storage on a Windows-mounted drive may have permissions and performance drawbacks; not a blanket solution | Do not adopt without testing |

Docker documents a `--wsl-default-data-root=<path>` installer option for selecting the default location of the WSL distribution disk. That is an installation/configuration choice, not proof that Docker Desktop is portable. Do not apply it or change the existing `E:\CyberLab\DockerData` folder until its current contents and any active Docker data have been identified.

## Proposed drive layout

```text
E:\CyberLab\
  WSL\Alpine\ext4.vhdx
  WSL\wsl-swap.vhdx
  Projects\wazuh-docker\single-node\
  DockerData\          # existing contents: audit before changing
  Evidence\
  PCAP\
  Reports\
  Backups\Docker\
  Scripts\             # preflight/start/stop/backup scripts to add
```

This is a proposed lab layout, not permission to move or delete current files. Personal files elsewhere on the portable drive are out of scope.

## Implementation order

1. **Preserve and identify:** inventory `E:\CyberLab\DockerData` and start the existing Docker Desktop executable; confirm whether the Docker CLI and engine actually work. Do not delete/reinitialize its data.
2. **Verify the endpoint:** `AlpineLab` is now registered from the E: VHDX and reports Alpine 3.24.2, user `analyst`, and the expected `/etc/wsl.conf`. Recheck its SSH/syslog startup on this PC.
3. **Recover the existing Wazuh stack:** validate the Compose file, list existing containers and volumes, and check indexer health before recreating anything.
4. **Make it reproducible:** keep Compose/configuration on E:, document required settings, and create a tested backup/restore procedure for the state that matters.
5. **Add one-click operations:** write preflight, start, stop, and backup scripts with clear errors if Docker/WSL/E: is unavailable. Scripts must never run `docker compose down -v` or stop/delete volumes as a cleanup shortcut.
6. **Improve portability:** after log ingestion is working, assess whether a dedicated SSH-test endpoint container should replace or complement the WSL endpoint. Keep services separate rather than packaging the whole stack into one container.
7. **Test recovery on a second host:** verify the checklist using the drive, a clean Docker host setup, and sanitized evidence before claiming portability.

## Safe removal rules

- Quit Docker Desktop and stop containers before shutdown.
- Exit WSL shells, then run `wsl --shutdown` and wait until all VHDX files are idle before disconnecting the drive.
- Never unplug E: while WSL, the configured swap VHDX, or Docker storage is active.
- Do not manually copy/move an active Docker/WSL disk image.
- Do not delete `DockerData\disk`, `DockerData\main`, either Alpine VHDX, or Docker volumes during discovery.
- Never store passwords, API tokens, private keys, or unsanitized captures in the public repository.

## Official documentation

- [Install Docker Desktop on Windows](https://docs.docker.com/desktop/setup/install/windows-install/) — host requirements and the `--wsl-default-data-root` installer option.
- [Docker Desktop WSL 2 backend](https://docs.docker.com/desktop/features/wsl/) — WSL backend requirements.
- [Docker Desktop backup and restore](https://docs.docker.com/desktop/settings-and-maintenance/backup-and-restore/) — preserve Docker data safely.
- [Docker Compose volumes](https://docs.docker.com/reference/compose-file/volumes/) — persistence and volume configuration.
- [WSL command reference](https://learn.microsoft.com/windows/wsl/reference) — export/import and `--import-in-place`.
