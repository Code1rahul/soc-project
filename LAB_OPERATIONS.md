# Lab Operations — Start, Verify, and Stop

This guide is for the local SOC lab. It intentionally uses manual, transparent commands first; tested start/stop `.bat` wrappers can be added later. **Do not use `docker compose down -v`** for routine shutdown because it removes volumes and can delete lab data.

## Layout

- Wazuh Compose directory: `E:\CyberLab\Projects\wazuh-docker\single-node`
- Lab evidence root: `E:\CyberLab\Evidence`
- Packet captures: `E:\CyberLab\PCAP`
- Currently registered Alpine WSL distribution: `D:\WSL\Alpine` (reports version 3.23.3; likely the older/local install). Candidate portable-drive VHDX: `E:\CyberLab\WSL\Alpine\ext4.vhdx` (exists but is not the current registered path).
- Global WSL config file: `%USERPROFILE%\.wslconfig` (Windows-managed location; do not relocate this file)
- WSL swap file is configured at `E:\CyberLab\WSL\wsl-swap.vhdx` in the current host's `.wslconfig`.

## Portable drive / new Windows host notes

This lab drive is portable and is currently mounted as `E:`. The internal `D:` drive is not the lab volume. Keep all project paths under `E:\CyberLab` where supported.

Before launch:
- Make sure the portable drive is connected and mounted as `E:`.
- Do not disconnect it while WSL or Docker is running.
- The global `.wslconfig` is stored in the current Windows user profile, not on the portable drive.
- The current PC's `Get-Command docker` returned nothing even though the Docker Desktop executable exists on E:. Launch the existing executable first; if Docker CLI remains unavailable, verify the installation/integration on this Windows host rather than recloning the project or deleting DockerData.
- Verify Docker Desktop Settings > Resources/Advanced shows the intended disk-image location before starting or changing containers.
- WSL registry inspection confirmed the current `Alpine` distro uses `D:\WSL\Alpine\ext4.vhdx` and reports 3.23.3. A separate VHDX exists at `E:\CyberLab\WSL\Alpine\ext4.vhdx`, while previous lab sessions used 3.24.2. The size/timestamp comparison shows D: is 11.59 GB (created 28 Mar 2026), while E: is 0.10 GB (created 4 Oct 2026). D: is likely the older local distro; E: is likely the recent portable lab import, but verify its contents. Preserve both.

```powershell
Start-Process 'E:\CyberLab\Tools\DockerDesktop\Docker Desktop.exe'
```

After Docker Desktop finishes starting, verify:

```powershell
Get-Command docker -ErrorAction SilentlyContinue
wsl --list --verbose
docker info
```

If the CLI is still not recognized, do not assume the existing DockerData was imported or delete it. Inspect the actual installation and Docker Desktop settings first.

## Compare both Alpine VHDX files before choosing an instance

The registered distribution currently points to `D:\WSL\Alpine\ext4.vhdx`. A second VHDX exists at `E:\CyberLab\WSL\Alpine\ext4.vhdx`. Use read-only metadata checks first:

```powershell
@('D:\WSL\Alpine\ext4.vhdx','E:\CyberLab\WSL\Alpine\ext4.vhdx') | ForEach-Object {
    if (Test-Path $_) {
        Get-Item $_ | Select-Object FullName, Length, CreationTime, LastWriteTime
    } else {
        Write-Host "Missing: $_"
    }
}
```

Do not unregister, overwrite, or re-import either one until the intended instance is identified.

## Register and inspect the portable Alpine candidate (safe additive check)

Microsoft documents `wsl --import-in-place <Name> <FileName>` for registering an existing ext4-formatted WSL VHDX as another distribution. This does not require deleting the older `Alpine` registration. Use a new name so both installations remain available.

```powershell
wsl --shutdown
wsl --import-in-place AlpineLab 'E:\CyberLab\WSL\Alpine\ext4.vhdx'
wsl --list --verbose
wsl -d AlpineLab -- sh -lc 'cat /etc/alpine-release; id; cat /etc/wsl.conf 2>/dev/null; ls -l /var/log/messages 2>/dev/null'
```

Stop if WSL reports that the file is invalid or already registered under another name; do not force it. Verify the version, `analyst` account, SSH, and syslog before designating this as the project endpoint. Never unregister `Alpine` or delete either VHDX while still identifying the correct instance.

## Start the lab
### 1. Start Docker Desktop
Open Docker Desktop and wait for the engine to become available.

**Why:** Compose commands can exist on Windows even when the Linux engine is not ready.

Verify:

```powershell
docker info
docker compose version
```

Check that `docker info` includes a Server section.

### 2. Go to the Compose directory

```powershell
Set-Location 'E:\CyberLab\Projects\wazuh-docker\single-node'
```

### 3. Validate the Compose configuration before starting

```powershell
docker compose config --quiet
```

**Why:** this catches syntax or interpolation errors before starting or changing containers.

### 4. Start containers

```powershell
docker compose up -d
docker compose ps
```

Expected services: `wazuh.indexer`, `wazuh.manager`, and `wazuh.dashboard` show as running.

### 5. Check resource use and recent logs

```powershell
docker stats --no-stream
docker compose logs --since 5m wazuh.indexer wazuh.manager wazuh.dashboard
```

Look for persistent errors, repeated cluster-event timeouts, manager-to-indexer failures, or sustained JVM garbage-collection pressure. A container being `Up` does not alone prove that the service is healthy.

### 6. Verify indexer health

The following command prompts for the configured indexer password. Don't paste the password or authentication header into notes or chat.

```powershell
curl.exe -sk -u admin 'https://localhost:9200/_cluster/health?pretty'
```

A healthy single-node cluster previously returned `"status" : "green"`, 22 active primary shards, zero unassigned shards, and zero pending tasks. Re-check the current response rather than assuming it remains healthy.

### 7. Open the dashboard

Open `https://localhost` in a browser and sign in with the configured dashboard credentials. The browser may warn about the lab's self-signed certificate.

**Important:** the last verified dashboard showed no agents registered. Do not assume Alpine logs are being ingested until a controlled test event appears in Wazuh.

### 8. Start Alpine if needed

In a separate PowerShell window:

```powershell
wsl -d Alpine
```

Inside Alpine, verify:

```sh
whoami
ps | grep '[s]shd'
ps | grep '[s]yslogd'
tail -n 10 /var/log/messages
```

The current WSL boot script is intended to launch SSH and syslog. Refresh Alpine's IP when needed; WSL NAT addresses can change after restarts.

## Stop the lab safely

### 1. Stop Wazuh containers without deleting data

From the Compose directory:

```powershell
docker compose stop
docker compose ps
```

This stops containers while retaining the containers and named volumes for the next run.

### 2. Optional: fully stop WSL

First exit Alpine shells and quit Docker Desktop from the system tray. Then run:

```powershell
wsl --shutdown
```

This stops all WSL distributions, including the Docker Desktop WSL environment. Do not run it while doing other WSL work.

### 3. Start next time

Open Docker Desktop, wait for the engine, then run the Start section above.

## Do not do these for routine shutdown

- Do not run `docker compose down -v`; the `-v` option removes named volumes.
- Do not delete Docker volumes or the WSL VHDX files to "free RAM".
- Do not expose indexer/dashboard ports to the public internet.
- Do not paste passwords, API tokens, generated authorization headers, or private keys into chat or repository files.

## Security checks still pending

- Review current published host ports. At the last recorded `docker compose ps`, the dashboard, indexer, and manager ports were published on all host interfaces. Restrict unnecessary access before using on shared networks; preserve the connection needed for the intended telemetry route.
- Change any default credentials if still in use.
- Validate manager/indexer logs after restart.
- Prove an Alpine test event appears in Wazuh before claiming the endpoint is monitored.
- Confirm where Docker Desktop stores its disk image in its settings; Docker's in-container path `/var/lib/docker` alone does not prove the backing image is located on D:.

## Troubleshooting quick reference

| Symptom | First check |
|---|---|
| `docker info` has no Server section | Start Docker Desktop and wait for the engine. |
| A container is `Up` but web views fail | Inspect recent manager/indexer/dashboard logs and query indexer health. |
| Indexer returns 401 | Use the actual configured credentials; do not guess repeatedly. |
| Indexer health is not green or timeouts recur | Save logs and resource stats before changing anything. |
| Dashboard loads but no agents are shown | This is not proof of Alpine log ingestion; choose and validate the intended collection path. |
| TCP bind fails | Inspect the Compose port mapping and Windows excluded port ranges before changing ports. |


## Portability target — read before modifying Docker storage

The target is a **portable lab kit**, not a Docker container that makes the whole drive bootable. Compose files, the Alpine VHDX, sanitized evidence, and backups travel on E:. Each Windows host still needs WSL 2 and a working Docker Desktop installation/engine.

Current successful check: `AlpineLab` is registered from `E:\CyberLab\WSL\Alpine\ext4.vhdx`; it reports Alpine `3.24.2`, user `analyst`, and the expected `/etc/wsl.conf`. The earlier `Alpine` distro on D: is separate and must remain untouched while validating the project endpoint.

Before touching Docker storage on this host:
1. Start the existing Docker Desktop executable and prove `docker info` has a Server section.
2. Inspect the current Docker Desktop backend and disk/data location in its supported settings.
3. Inspect `E:\CyberLab\DockerData\disk` and `main`; do not delete, rename, or point an installation at these folders until their purpose is known.
4. From the existing Wazuh Compose directory, inspect containers and volumes before changing them.
5. Keep personal files elsewhere on E: out of all Compose bind mounts and scripts.

Recommended first design: keep the lab configuration and evidence portable, keep Docker-managed database/indexer data in its supported Linux storage, and create tested backups for state that must travel. Only evaluate locating Docker's WSL data root on E: after this audit. Docker documents a `--wsl-default-data-root=<path>` installer option, but this does not remove the per-host runtime requirements or the risk of an unsafe USB disconnect.

Never unplug E: while Docker/WSL is running. Stop the Compose services without deleting volumes, quit Docker Desktop, close WSL sessions, run `wsl --shutdown`, and wait for VHDX files to become idle before disconnecting.

See [Portable SOC Lab Plan](PORTABILITY_PLAN.md) for the options, proposed layout, and staged implementation.


## Verified Docker installation on this PC — 14 October 2026

Docker Desktop is installed in per-user mode at `%LOCALAPPDATA%\Programs\DockerDesktop`. After temporarily adding `%LOCALAPPDATA%\Programs\DockerDesktop\resources\bin` to this PowerShell process's `PATH`, Docker CLI `29.8.2`, Compose `v5.5.1`, and `docker info` all responded successfully. The active context is `desktop-linux`; server reports four CPUs and 7.758 GiB memory.

The active backend currently lists **zero containers and zero images**. This is not the Wazuh stack previously documented on the old host. Before rebuilding, inspect the Compose files and the existing `E:\CyberLab\DockerData\disk` and `main` directories. Do not delete or manually move these directories.

The PATH addition is temporary. To persist it for the current Windows user, use the guarded command below, then close and reopen PowerShell:

```powershell
$dockerBin = Join-Path $env:LOCALAPPDATA 'Programs\DockerDesktop\resources\bin'
$userPath = [Environment]::GetEnvironmentVariable('Path', 'User')
$entries = @($userPath -split ';' | Where-Object { $_ })

if (-not ($entries | Where-Object { $_.TrimEnd('\') -ieq $dockerBin.TrimEnd('\') })) {
    [Environment]::SetEnvironmentVariable('Path', (($entries + $dockerBin) -join ';'), 'User')
}
```

Then verify `docker --version`, `docker compose version`, and `docker info` in a newly opened PowerShell window. This modifies only the current user's PATH; it does not relocate Docker storage. Docker Desktop's default WSL data location is normally under `%LOCALAPPDATA%\Docker\wsl`, but verify **Settings → Resources → Advanced** before inferring where this host's backing disk actually resides. See [Docker's official WSL backend and data location documentation](https://docs.docker.com/desktop/features/wsl/).


## Docker data-disk inventory before recovery — 14 October 2026

The Compose configuration validated successfully with `docker compose config --quiet`. Current engine reports zero containers, zero images, and zero named volumes.

Observed Docker WSL VHDX file lengths:
- Portable `E:\CyberLab\DockerData\disk\docker_data.vhdx`: 45.15 GB, last written 11 Oct 2026.
- Portable `E:\CyberLab\DockerData\main\ext4.vhdx`: 0.09 GB, last written 11 Oct 2026.
- Current host `%LOCALAPPDATA%\Docker\wsl\disk\docker_data.vhdx`: 1.56 GB, last written 14 Oct 2026.
- Current host `%LOCALAPPDATA%\Docker\wsl\main\ext4.vhdx`: 0.09 GB, last written 14 Oct 2026.

The E: disk is likely the previous Docker data store, but contents are not independently validated. File length is not necessarily equal to allocated space for a dynamic VHDX. Before recovery, inspect Docker Desktop → Settings → Resources → Advanced → Disk image location and report the current path. Never move the VHDX in File Explorer, and don't choose an option that replaces the E: disk. Don't run `docker compose up` until we decide whether to use the preserved data disk or recreate images/containers from the validated Compose file.


## Backup completed; restore not started — 14 October 2026

Robocopy copied both files from `E:\CyberLab\DockerData` to `E:\CyberLab\Backups\DockerData-pre-recovery-20261014`, reporting zero failures. Reported lengths match: `docker_data.vhdx` 45.15 GB and `main\ext4.vhdx` 0.09 GB. This is size-based verification; SHA-256 comparison has not been performed. Do not delete either copy.

Docker Desktop currently uses `C:\Users\R123\AppData\Local\Docker\wsl`; C: had 18.79 GB free. The E: disk is likely the previous environment but its contents have not yet been validated. Do not set the UI location to the existing `E:\CyberLab\DockerData` folder because that could collide with the old disk. If relocating the current empty Docker backend to E:, use a separate **new empty destination** such as `E:\CyberLab\DockerDesktop-Active`, verify the move, and only then design the restore step for the old data disk. No restore or replacement has been attempted yet.

Official guidance: [Docker Desktop backup and restore](https://docs.docker.com/desktop/settings-and-maintenance/backup-and-restore/). Fully stop Docker Desktop before direct disk backup/restore.


## Active Docker storage now on E: — 15 October 2026

The user selected `E:\CyberLab\DockerDesktop-Active` in Docker Desktop. Docker remains responsive, and file inspection found:
- `E:\CyberLab\DockerDesktop-Active\DockerDesktopWSL\disk\docker_data.vhdx` — 1.53 GB by reported length.
- `E:\CyberLab\DockerDesktop-Active\DockerDesktopWSL\main\ext4.vhdx` — 0.09 GB by reported length.

The active engine still shows zero images/containers, expected for the clean backend. The old data disk at `E:\CyberLab\DockerData\disk\docker_data.vhdx` remains 45.15 GB and has a verified size-matched copy under `E:\CyberLab\Backups\DockerData-pre-recovery-20261014`; no hash test was done.

To recover previous image/container state, do not repoint the UI at `E:\CyberLab\DockerData` and do not overwrite the only old disk. Follow the Docker backup/restore guidance. Before restoring, stop Docker Desktop, run `wsl --shutdown`, and create a rollback copy of the new active data disk. Replace only the active `DockerDesktopWSL\disk\docker_data.vhdx` from the **backup copy** of the previous `docker_data.vhdx`, leave the current active `main\ext4.vhdx` in place for the initial attempt, then start Docker Desktop and check whether the old containers/images are visible. If it fails, stop and restore the active rollback disk; retain both original copies.


## Restored disk placed; startup validation pending — 15 October 2026

Observed file layout after user's operation:
- Active `DockerDesktopWSL\disk\docker_data.vhdx`: 45.15 GB.
- Preserved clean active disk `docker_data.clean-before-restore-20261015.vhdx`: 1.53 GB.
- Clean active disk rollback copy: 1.53 GB.
- Original old source and separate old-disk backup: 45.15 GB each.

The command invocation of `E:\CyberLab\Scripts\restore-old-docker-disk.ps1` then reported the file did not exist. The displayed sizes nevertheless show that the file replacement had already been performed; do not rerun the restore block.

Now start Docker Desktop normally and check whether it successfully starts and the old images/containers/volumes return. If it fails, stop and report the exact startup error before doing anything else. Do not use Factory Reset, do not delete the old 45 GB disk, and do not overwrite the preserved 1.53 GB disk.


## Critical recovery note — direct replacement failed (15 October 2026)

Replacing the active `docker_data.vhdx` with the previous 45.15 GB VHDX caused Docker Desktop's WSL bootstrap to fail. Error: no block device matched data-disk ID `32d25913-dc40-454e-8d44-ef700957fa14`, and `/opt/docker-desktop/componentsVersion.json` was missing. This suggests the current Docker Desktop provisioning metadata expects a disk identity that the old VHDX does not have. Do **not** attempt the direct replacement a second time.

To restore a working Docker installation:
1. Quit Docker Desktop and run `wsl --shutdown`; verify Docker processes have exited.
2. Rename the current 45.15 GB active file to `docker_data.failed-restore-20261015.vhdx`.
3. Rename `docker_data.clean-before-restore-20261015.vhdx` back to `docker_data.vhdx`.
4. Keep the pre-restore copy and every original/backup of the old VHDX intact.
5. Launch Docker Desktop and verify the engine.

If Docker starts, recreate Wazuh from the validated Compose project and pulled version-pinned images after inspecting the Compose file and security-sensitive config. Prior old images/containers/volumes are not confirmed recovered. Never use Factory Reset or delete the 45.15 GB disk.


## Docker rollback confirmed — 15 October 2026

The user reports that `docker info` is healthy on Docker Desktop 29.8.2 after restoring the clean data disk. `docker ps -a` and `docker image ls` both show empty output. Docker's active data path stays at `E:\CyberLab\DockerDesktop-Active`.

The direct restore of the old 45.15 GB Docker data disk failed due to Docker Desktop's WSL disk-ID mismatch. Do not repeat the manual VHDX replacement. Original and backup disks remain untouched. The Wazuh Compose project still exists under `E:\CyberLab\Projects\wazuh-docker\single-node`.

Next: validate the Compose file/config/cert files; inspect `docker compose config --images`; pull image versions pinned by the Compose file and launch with `docker compose up -d` only after validating the configuration. This creates a clean Wazuh runtime, not recovered historical Docker state. Do not use `docker compose down -v`.


## Clean Wazuh deployment preflight — 15 October 2026

User's current Compose directory passes `docker compose config --quiet`. Services are indexer, manager, dashboard; all image tags are pinned to `4.14.8`. The local `config` tree contains service configuration and TLS private keys.

Before `docker compose pull` / `up -d`, inspect effective host published ports. Do not print or paste full `docker compose config` because it may contain interpolated sensitive values; pipe its JSON output to PowerShell and display only image names and host/target port bindings. Prefer loopback host IP `127.0.0.1` for dashboard/indexer if local-only access is the intent. Keep certificates/private keys out of Git and screenshots.
