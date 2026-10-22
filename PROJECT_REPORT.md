# SOC Home Lab — Project Report

## Objective

Build a reproducible SOC practice environment to collect authentication telemetry, verify network exposure, detect suspicious login patterns, and conduct a structured investigation using Wazuh.

## Environment and current state

Wazuh Manager, Indexer, and Dashboard 4.14.8 are running. The dashboard login succeeds. The Compose configuration is valid, and the dashboard and indexer are currently bound to local access only.

The lab endpoint is an Alpine WSL2 distribution registered as `AlpineLab`. Earlier checks verified Alpine 3.24.2, user `analyst`, SSH, and local syslog configuration. An earlier WSL `E_ACCESSDENIED` issue stopped occurring after a targeted Windows ownership/access correction; its underlying cause was not definitively established.

On 22 October 2026, endpoint-to-Wazuh ingestion was proven through a temporary foreground Windows UDP relay. Alpine's normal `logger`/BusyBox `syslogd` path sent a unique marker to the relay on the WSL gateway at UDP/15144; the relay forwarded it to the Manager's published UDP/514 endpoint; the marker appeared in `alerts.json` as rule 5710, level 5. The direct Alpine-to-published-UDP/514 path remains unresolved. The relay and Alpine forwarding destination are not yet persistent across restart.

The previous Docker data could not be attached successfully to the current Docker Desktop setup. The existing Compose project was used to deploy a clean Wazuh stack. A prior-data copy remains preserved separately; don't assume its containers, images, or volumes have been recovered.

## Verified lab exercises

### SSH authentication simulation

Three deliberately generated failed SSH logins were followed by a successful authentication. This is a controlled lab scenario. It is not evidence of an external compromise.

### Network baseline

A default Nmap TCP scan found SSH as the only open TCP service in that scan. This is a scoped service baseline, not a full vulnerability assessment.

### Packet inspection

A normal SSH session capture was inspected. Network and TCP metadata were visible, while commands inside the SSH session remained encrypted. Endpoint logs are needed to investigate post-login activity.

## Remaining detection and investigation milestones

1. Make Alpine's relay destination persistent in its startup configuration.
2. Configure reliable Windows relay startup after WSL and Docker are ready; ensure the gateway and Alpine source address are rediscovered safely.
3. Verify a normal alert is searchable in the Indexer/Dashboard with `logall_json=no` and recheck Indexer health.
4. Run a planned restart-recovery check and confirm ingestion resumes.
5. Simulate repeated failed SSH authentication and validate detection/correlation behavior.
6. Correlate authentication evidence, source context, the alert, and timeline; document response recommendations, false positives, and limitations.

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


## Verified syslog relay checkpoint — 22 October 2026

### Evidence
- A synthetic marker sent from the Indexer container appeared in `archives.json` while `logall_json=yes` was enabled temporarily. The exact original Manager configuration was restored and its hash verified; the active setting is `logall_json=no`.
- Windows UDP on the WSL gateway received a test datagram from AlpineLab.
- A direct Windows-generated datagram to the published Manager endpoint was seen in the Manager network namespace.
- Direct Alpine-to-published Manager UDP/514 traffic was not observed in the bounded Manager-side capture.
- A one-message Windows relay test successfully generated a rule 5710, level 5 alert.
- Alpine's normal syslog forwarding was then reconfigured at runtime to send to the relay on UDP/15144. Marker `SOC_LAB_NORMAL_SYSLOG_RELAY_20261022_210527` was present in the local Alpine log, the relay logged forwarding, and the matching alert was present in `alerts.json` with rule 5710, level 5.

### Interpretation
The relay path is a verified workaround; the underlying cause of the direct Alpine-to-Docker-published-port behavior is not conclusively identified. Do not claim the direct path is fixed. The current relay and Alpine syslog process changes are runtime-only.

### Remaining work
1. Make Alpine syslogd's UDP/15144 relay destination persistent in its startup configuration.
2. Configure reliable relay startup after WSL/Docker networking is available, with narrow binding, current-address discovery, and operational logs.
3. Confirm the alert is searchable in the Indexer/Dashboard while `logall_json=no`.
4. Recheck Indexer health and validate relay recovery after a controlled WSL restart.
5. Build repeated-failed-SSH detection, a timestamped incident timeline, and a sanitized evidence package.

Keep Dashboard/Indexer local-only. Do not broaden firewall rules or bind UDP/514 to all host interfaces. No storage changes were made.