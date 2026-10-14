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
