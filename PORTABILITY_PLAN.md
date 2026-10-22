# Portable SOC Lab Plan

## Goal

Keep the SOC learning environment reproducible between suitable Windows PCs without making the external drive itself a single, fragile runtime. The drive holds project material and recovery copies; each Windows host still requires a supported WSL2 environment and a working Docker Desktop installation.

## Portable project contents

Keep the following lab-specific resources together on the external lab drive:
- Docker Compose definitions and sanitized configuration.
- The Alpine WSL endpoint image and its recovery copy.
- Lab scripts, evidence, screenshots, packet captures, and reports.
- Explicit backups of application state that must survive a host change.

Unrelated personal files on the same drive must remain outside container bind mounts and lab scripts.

## Runtime dependencies

Each new Windows host needs:
- A supported Windows build with virtualization enabled.
- WSL2 installed and updated.
- Docker Desktop installed, started, and configured to use its Linux backend.
- Local, private credentials and certificate material for the environment.
- The endpoint distribution registered and validated on that host.

Host-specific WSL settings are not automatically portable. Validate each host's configuration rather than copying machine-specific settings blindly.

## Recommended operating model

1. Keep Compose definitions and lab scripts as the repeatable deployment recipe.
2. Pin image versions and validate Compose configuration before starting services.
3. Keep Dashboard and Indexer local-only during initial testing.
4. Back up relevant application data using documented procedures and verify the backup before any recovery attempt.
5. Do not manually replace Docker-managed virtual disks; a matching filename does not guarantee that the current runtime can attach the data.
6. Test startup, health, event collection, and recovery on a second suitable host before describing the setup as plug-and-play.

## Current blockers

- End-to-end endpoint ingestion is verified through a temporary foreground Windows UDP relay, but the relay and Alpine syslog destination are not yet persistent across restarts.
- Direct Alpine-to-Docker-published UDP/514 delivery was not observed during a bounded Manager-side capture. A Windows-side relay is the current proven workaround; the exact underlying cause remains unresolved.
- The Alpine WSL endpoint returned expected version and log output after a targeted Windows ownership/access correction.
- The previous Docker data was not attached successfully to the current runtime; a clean deployment from the existing Compose project is running. The preserved prior-data archive should remain untouched unless a supported recovery path is established.

## Safe handling

- Never disconnect removable storage while WSL or Docker is active.
- Do not delete Docker volumes as a cleanup shortcut.
- Keep secrets, private keys, personal identifiers, raw unsanitized captures, and machine-specific storage information out of the public repository.

## Official references

- [Docker Desktop on Windows](https://docs.docker.com/desktop/setup/install/windows-install/)
- [Docker Desktop WSL2 backend](https://docs.docker.com/desktop/features/wsl/)
- [Docker Desktop backup and restore](https://docs.docker.com/desktop/settings-and-maintenance/backup-and-restore/)
- [Microsoft WSL command reference](https://learn.microsoft.com/windows/wsl/reference)


## Portability implication — 22 October 2026

The current relay workaround depends on a Windows user-session process and WSL/Docker gateway addresses. For a portable setup, treat the relay as a host-specific adapter that must discover current addresses, start only after WSL/Docker are available, and be verified after restart. Do not call the lab plug-and-play until these steps have been validated on a second start/session or another host. A supported Linux VM with the Wazuh agent remains a longer-term alternative for richer endpoint telemetry.