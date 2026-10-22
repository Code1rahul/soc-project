# Lab Operations — SOC Home Lab

This guide covers starting and verifying the current Wazuh Compose project while avoiding destructive cleanup actions.

## Current state

- Docker Desktop engine responds successfully.
- Wazuh Manager, Indexer, and Dashboard 4.14.8 are running.
- Dashboard login works.
- Dashboard and Indexer are bound to local access.
- The Alpine WSL endpoint registered as `AlpineLab` subsequently returned expected version and log output after a targeted Windows ownership/access correction. Do not make further permissions changes unless the failure recurs.
- Alpine-to-Wazuh ingestion is verified through a temporary foreground Windows relay: Alpine syslogd sends to the WSL gateway on UDP/15144; the relay forwards to the Manager's published UDP/514 endpoint. The normal syslog marker generated rule 5710, level 5. The relay and Alpine syslog destination are still runtime-only and have not yet been made persistent.
- The Manager's TCP/1514 secure-agent listener is distinct from the remote syslog receiver on UDP/514.

## Start Wazuh

Open PowerShell in the existing Compose project directory.

Validate the Compose project:

```powershell
docker compose config --quiet
```

Start the services:

```powershell
docker compose up -d
docker compose ps
```

Expected services:
- `wazuh.indexer`
- `wazuh.manager`
- `wazuh.dashboard`

Review recent logs:

```powershell
docker compose logs --since 5m --tail 50 wazuh.indexer wazuh.manager wazuh.dashboard
```

A container being `Up` does not alone confirm Indexer health. Check Indexer cluster health with the configured credentials without placing passwords or authorization headers in a command saved to public notes.

Open the dashboard locally at `https://localhost`. A self-signed certificate warning may appear in this lab environment.

## Alpine endpoint status

The intended endpoint distribution is `AlpineLab`. Its prior known configuration used OpenSSH and BusyBox syslog, with messages in `/var/log/messages`. However, its most recent startup attempt failed at WSL disk attachment with `E_ACCESSDENIED`. Troubleshoot that host-level error with read-only checks before changing permissions or attempting any filesystem recovery.

## Configure event collection only after Alpine starts

A Docker port mapping alone does not configure a Wazuh syslog receiver. Wazuh Manager must be configured explicitly for remote syslog with the correct protocol, port, and a narrow sender allowlist. Keep the Dashboard and Indexer local-only. After the intended sender address and network route are verified, expose only the ingestion port needed for the lab test.

The current validated workaround uses a foreground Windows relay. Start the relay script in a dedicated PowerShell session, confirm it binds only to the WSL gateway on UDP/15144, and confirm Alpine syslogd targets that listener. Do not close the relay window during a practical session. The current runtime setup has not yet been made restart-persistent. Do not enable raw JSON archiving for routine operation; the intended setting is `logall_json=no`.

The normal syslog marker was confirmed in the Manager's `alerts.json` under rule 5710, level 5. The next work is to make Alpine's relay destination persistent, define reliable relay startup after WSL/Docker networking is ready, verify the same alert in the Dashboard/Indexer, and test recovery after a controlled WSL restart.

## Stop safely

Stop containers without removing their data:

```powershell
docker compose stop
docker compose ps
```

Avoid `docker compose down -v`; it removes named volumes. If the external lab drive is removable, quit Docker Desktop and shut down WSL before disconnecting it.

## Do not

- Do not use Factory Reset to troubleshoot a single service.
- Do not manually replace Docker-managed data files.
- Do not delete persistent volumes as a routine cleanup step.
- Do not expose Dashboard or Indexer to the public internet.
- Do not publish passwords, tokens, private keys, unsanitized PCAPs, personal details, or machine-specific storage information.

## Official references

- [Docker Compose CLI](https://docs.docker.com/reference/cli/docker/compose/)
- [Wazuh Docker deployment](https://documentation.wazuh.com/current/deployment-options/docker/index.html)
- [Wazuh Syslog collection](https://documentation.wazuh.com/current/user-manual/capabilities/log-data-collection/syslog.html)


## Current next phase: prove Alpine-to-Wazuh event delivery

The Alpine endpoint now starts and returns its expected version and logs after the ownership fix. Stop troubleshooting Docker storage; it is out of scope unless a new container health failure appears.

Before changing the syslog route, obtain the current Alpine gateway/interface address and identify the matching Windows WSL virtual-interface IP. Configure the Manager explicitly for remote syslog with the documented protocol and a narrow sender allowlist. The Manager's existing `connection=secure` listener on TCP/1514 is for Wazuh agents, not a generic syslog listener. All published ports currently bind to loopback, so only the specific required ingestion path may be changed; keep Dashboard and Indexer loopback-only.

The test is complete only when a uniquely marked harmless event generated from Alpine can be found in Wazuh. Do not claim endpoint monitoring before that result exists.


## End-of-day operational handover — 15 October 2026

Wazuh 4.14.8 Manager, Indexer, and Dashboard were reported Up; dashboard login works. Dashboard and Indexer are loopback-bound. AlpineLab starts and its authentication logs are readable after a targeted Windows file-ownership correction.

**Still not complete:** Alpine-to-Wazuh event ingestion. The Manager has the secure agent listener on TCP/1514, but remote syslog has not yet been configured explicitly.

### Close for the day
1. In the Compose project directory, run `docker compose stop` and confirm the services stop with `docker compose ps`.
2. Quit Docker Desktop from the system tray.
3. Run `wsl --shutdown` and confirm the distributions are stopped with `wsl --list --verbose`.
4. Do not disconnect the external drive until Docker/WSL are stopped.

### Resume tomorrow
1. Start Docker Desktop; confirm `docker info` responds.
2. Start Wazuh via `docker compose up -d`; verify service states, Indexer health, and dashboard access.
3. Start/check AlpineLab and its route.
4. Configure the Wazuh documented remote syslog receiver and narrow sender allowlist. Make only UDP syslog ingestion reachable from the endpoint; keep Dashboard and Indexer local-only.
5. Send a uniquely marked harmless test event and prove it searchable.
6. Move to controlled SSH failure detection only after event ingestion is confirmed.

Do not revisit storage relocation or attempt direct Docker data-disk replacement. Do not run `docker compose down -v`.


## Syslog verification checkpoint — 19 October 2026

The latest logs confirm `wazuh-remoted` accepts syslog configuration, has allowed sender entries for the lab endpoint/network gateway, and reports listening on UDP/514. Alpine BusyBox `syslogd` runs with remote forwarding enabled. A unique event was written to Alpine's local log but not found in Wazuh alert/archive files during the test.

This no-match is not yet conclusive: archive logging may be disabled and alert files contain only rule-triggered events. Read the current `<logall_json>` setting; if disabled, temporarily enable archive JSON logging, send a fresh marker, confirm receipt in `archives.json`, and disable it again to avoid unbounded raw-log storage. Keep the event labelled as synthetic. Don't state endpoint ingestion is working until there is a clear receiver-side match.


## End-of-day handover — 19 October 2026

Docker Engine and all three Wazuh 4.14.8 services were observed running. The Indexer cluster returned green with 29 active primary shards and zero unassigned shards. Dashboard login worked. The Alpine endpoint is currently startable; SSH and BusyBox syslog processes were seen running.

The Manager configuration now includes a remote syslog receiver on UDP/514. The Manager logs report the receiver started and accepted the configured sender entries. Dashboard and Indexer remain loopback-only.

**End-to-end event ingestion is not yet proven.** A unique synthetic message appeared in Alpine's local log but did not appear in Wazuh alert/archive files. Both `logall` and `logall_json` were set to `no`, so archive absence is inconclusive. The temporary test script was generated but has not yet been confirmed run.

### Close the lab
```powershell
# From the existing Wazuh Compose project directory
docker compose stop
docker compose ps
```
Then quit Docker Desktop from the system tray and run:
```powershell
wsl --shutdown
wsl --list --verbose
```
Do not disconnect the portable lab media until Docker and WSL have stopped. Do not use `docker compose down -v`.

### Resume next session
1. Start Docker Desktop, verify `docker info`, start the existing Compose stack, and recheck Indexer health.
2. Confirm `AlpineLab` starts.
3. Run the prepared bounded ingestion test once: temporarily enable JSON archive logging, send one unique synthetic marker, check `archives.json`, and restore `logall_json=no`.
4. Explicitly verify configuration restoration, even if the marker is not received.
5. If receipt is proven, test the dashboard/indexing path and then move to controlled failed SSH detection. If not, isolate UDP transport before making other changes.
6. Update the project report only when each milestone has supporting evidence.

Do not make further storage changes or publish machine-specific storage paths and sizes.


## Syslog relay checkpoint — 22 October 2026

- Verified path: Alpine `logger` → BusyBox `syslogd` forwarding to the WSL gateway on UDP/15144 → foreground Windows relay → published Manager UDP/514 → rule 5710 alert.
- The relay only binds to the discovered WSL gateway and accepts traffic from the current AlpineLab address; it records forwarding/error status without logging every event payload.
- Relay window must remain open until automatic startup is implemented and validated.
- The running Alpine syslogd process was redirected at runtime. The Alpine startup configuration has not yet been updated, so a distro restart may revert its target.
- Manager configuration is restored with `logall_json=no`; allowlist contains Alpine's current address and the Docker gateway. Confirm current network addresses before any future network changes.

Do not change storage settings, broaden the UDP bind to all interfaces, or expose Dashboard/Indexer beyond loopback.