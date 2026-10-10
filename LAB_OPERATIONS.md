# Lab Operations — Start, Verify, and Stop

This guide is for the local SOC lab. It intentionally uses manual, transparent commands first; tested start/stop `.bat` wrappers can be added later. **Do not use `docker compose down -v`** for routine shutdown because it removes volumes and can delete lab data.

## Layout

- Wazuh Compose directory: `D:\CyberLab\Projects\wazuh-docker\single-node`
- Lab evidence root: `D:\CyberLab\Evidence`
- Packet captures: `D:\CyberLab\PCAP`
- Alpine WSL distribution: `D:\CyberLab\WSL\Alpine`
- Global WSL config file: `%USERPROFILE%\.wslconfig` (Windows-managed location; do not relocate this file)
- WSL swap file is configured on D: in the user's current WSL config.

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
Set-Location 'D:\CyberLab\Projects\wazuh-docker\single-node'
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
