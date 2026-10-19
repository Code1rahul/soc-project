# SOC Home Lab — Project Report

## Objective

Build a reproducible SOC practice environment to collect authentication telemetry, verify network exposure, detect suspicious login patterns, and conduct a structured investigation using Wazuh.

## Environment and current state

Wazuh Manager, Indexer, and Dashboard 4.14.8 are running. The dashboard login succeeds. The Compose configuration is valid, and the dashboard and indexer are currently bound to local access only.

The lab endpoint is an Alpine WSL2 distribution registered as `AlpineLab`. Earlier checks verified Alpine 3.24.2, user `analyst`, SSH, and local syslog configuration. An earlier attempt to start it failed with WSL `E_ACCESSDENIED`. The user subsequently confirmed that the expected Alpine version and log output returned after a targeted Windows ownership/access correction. The root cause is not definitively established. Endpoint-to-Wazuh event ingestion remains unproven.

The previous Docker data could not be attached successfully to the current Docker Desktop setup. The existing Compose project was used to deploy a clean Wazuh stack. A prior-data copy remains preserved separately; don't assume its containers, images, or volumes have been recovered.

## Verified lab exercises

### SSH authentication simulation

Three deliberately generated failed SSH logins were followed by a successful authentication. This is a controlled lab scenario. It is not evidence of an external compromise.

### Network baseline

A default Nmap TCP scan found SSH as the only open TCP service in that scan. This is a scoped service baseline, not a full vulnerability assessment.

### Packet inspection

A normal SSH session capture was inspected. Network and TCP metadata were visible, while commands inside the SSH session remained encrypted. Endpoint logs are needed to investigate post-login activity.

## Detection and investigation milestones still pending

1. Restore reliable access to the Alpine WSL endpoint.
2. Configure a valid Wazuh remote syslog receiver and an appropriately scoped sender allowlist.
3. Prove collection by searching for a unique benign test event.
4. Generate controlled failed SSH authentication and confirm a corresponding event/alert.
5. Correlate authentication records, endpoint evidence, and network metadata.
6. Build an incident timeline and document severity, response recommendations, false positives, and limitations.

## Lessons learned

- A published container port does not automatically enable the corresponding application listener.
- Container status alone is not proof of healthy indexing or endpoint telemetry ingestion.
- Replacing a Docker data disk manually can conflict with the current installation's provisioning state; future recovery should follow a validated, reversible procedure.
- Evidence must distinguish observed facts from assumptions, and simulations must not be described as real compromises.

## Publication hygiene

Only publish sanitized configuration excerpts and redacted screenshots. Do not publish credentials, private keys, raw packet captures, personal identifiers, or machine-specific storage information.


## End-of-day status — 15 October 2026

The deployment was brought up from the validated Compose project after the old Docker state failed to attach to the current runtime. The three Wazuh 4.14.8 services are running, dashboard login succeeds, and the UI/Indexer host ports are loopback-only.

The AlpineLab WSL endpoint subsequently started after a targeted Windows file-owner correction. Its version and local logs were accessible, but endpoint event ingestion into Wazuh has not yet been proven. The next session should resume with health validation, explicit remote syslog configuration and a harmless uniquely marked event. Detection development follows only after the test event is searchable.

The session shutdown sequence is: stop Compose services, quit Docker Desktop, run `wsl --shutdown`, and check that WSL distributions are stopped. Do not use volume-deleting cleanup commands.


## End-of-day checkpoint — 19 October 2026

### Confirmed
- Wazuh Manager, Indexer, and Dashboard 4.14.8 run; dashboard login succeeded.
- Indexer cluster health returned green with a single node, 29 active primary shards, and zero unassigned shards.
- AlpineLab started and reported local authentication/syslog output; SSH and BusyBox syslog were running.
- Wazuh Manager accepted a remote syslog receiver configuration and reported listening on UDP/514.
- Dashboard and Indexer remain bound to local access.

### Not yet confirmed
A unique synthetic event was written to the Alpine local log, but no matching record was found in Wazuh alert/archive files. Raw JSON archive logging was disabled during that test, so no archive match alone does not prove transport failure. The prepared temporary archive-ingestion test has not yet been confirmed executed.

### Next evidence milestone
Temporarily enable Wazuh JSON archive logging for one controlled test; send a newly marked synthetic syslog event; search for it in the Manager archive; restore the original archive setting and verify restoration. If received, verify indexing and rule behavior. Then generate controlled failed SSH attempts, confirm a detection, and create a timestamped incident timeline. Clearly distinguish synthetic test events from genuine authentication failures.
