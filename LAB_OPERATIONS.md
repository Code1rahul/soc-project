# Lab Operations — SOC Home Lab

This guide covers starting and verifying the current Wazuh Compose project while avoiding destructive cleanup actions.

## Current state

- Docker Desktop engine responds successfully.
- Wazuh Manager, Indexer, and Dashboard 4.14.8 are running.
- Dashboard login works.
- Dashboard and Indexer are bound to local access.
- The Alpine WSL endpoint registered as `AlpineLab` currently fails to start with WSL `E_ACCESSDENIED`; do not alter its permissions or unregister it before the cause is understood.
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
