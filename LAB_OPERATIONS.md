# Lab Operations — SOC Home Lab

This guide covers starting and verifying the current Wazuh Compose project while avoiding destructive cleanup actions.

## Current state

- Docker Desktop engine responds successfully.
- Wazuh Manager, Indexer, and Dashboard 4.14.8 are running.
- Dashboard login works.
- Dashboard and Indexer are bound to local access.
- The Alpine WSL endpoint registered as `AlpineLab` subsequently returned expected version and log output after a targeted Windows ownership/access correction. Do not make further permissions changes unless the failure recurs.
- Endpoint event ingestion is not yet verified. The Manager's TCP/1514 secure-agent listener is not the same thing as a remote syslog receiver.

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

Then send one uniquely marked, harmless test message and prove it is searchable in Wazuh before claiming collection works.

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
