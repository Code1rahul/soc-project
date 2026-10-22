# Master Tracker — SOC Home Lab

**Updated:** 22 October 2026  
**Project:** Wazuh + Alpine WSL2 + SSH event investigation

## Verified status

- [x] Docker Desktop engine responds successfully.
- [x] Wazuh Manager, Indexer, and Dashboard version 4.14.8 containers are running.
- [x] Dashboard login works and browser API requests return successful responses.
- [x] Dashboard and Indexer are loopback-bound for the current local validation phase.
- [x] Compose configuration validates.
- [x] A copy of the previous Docker data was preserved before recovery attempts.
- [x] Current Docker service state was restored after an unsuccessful attempt to reuse the prior data disk.
- [x] Alpine 3.24.2 and the `analyst` account were verified after the lab endpoint was registered under the name `AlpineLab`.
- [x] `AlpineLab` returned the expected version and log output after a targeted Windows ownership/access correction; no further permission changes are planned unless the error recurs.
- [ ] Confirm current Indexer cluster health after the latest service restarts.
- [x] Configure the Wazuh Manager remote syslog receiver on UDP/514 with a narrow allowlist.
- [x] Prove normal Alpine syslog ingestion through the temporary Windows relay: marker `SOC_LAB_NORMAL_SYSLOG_RELAY_20261022_210527` generated Wazuh rule 5710, level 5.
- [ ] Make the Windows relay persistent and restart-safe; it is currently a foreground PowerShell process.
- [ ] Update Alpine's startup configuration to forward to the relay on UDP/15144; the running syslogd process was changed at runtime, but persistence is not yet confirmed.
- [ ] Verify the alert is searchable in the Indexer/Dashboard.
- [ ] Build and validate a detection for repeated failed SSH authentication.
- [ ] Create a timestamped incident timeline and sanitized evidence package.

## Architecture

- Windows host provides PowerShell, Docker Desktop, and WSL2.
- Alpine WSL2 endpoint provides SSH and local authentication logs.
- Wazuh Manager receives and analyses endpoint events.
- Wazuh Indexer stores searchable data.
- Wazuh Dashboard supports monitoring and investigation.

## Important technical findings

### Docker and Wazuh

The Wazuh Compose project defines three services, all pinned to version 4.14.8. The configuration validates and all three containers were observed running. A fresh deployment is in use; the prior Docker containers, images, and volumes were not recovered into the current engine.

Published access is intentionally scoped:
- Dashboard: host 127.0.0.1:443 to container 5601/TCP.
- Indexer: host 127.0.0.1:9200 to container 9200/TCP.
- Manager secure-agent ports: host 127.0.0.1:2514 and 127.0.0.1:2515 to the corresponding container TCP ports.
- Manager API: host 127.0.0.1:55000 to container TCP/55000.
- Manager syslog: UDP/514 is published on the current WSL gateway address only, not on all host interfaces.

The Manager has a Wazuh remote syslog receiver on UDP/514. Normal Alpine syslog events were successfully relayed through a foreground Windows PowerShell process listening on the WSL gateway at UDP/15144 and forwarding to the published Manager UDP/514 endpoint. The direct Alpine-to-Docker-published UDP/514 path remains unreliable/unproven, while the relay path generated a confirmed rule 5710 alert. The relay and Alpine syslog target are currently runtime/manual settings, not yet made persistent. Keep Dashboard and Indexer local-only; do not widen the Manager listener beyond the WSL gateway.

### Alpine endpoint

The project endpoint distribution is `AlpineLab`; earlier verification reported Alpine 3.24.2 and user `analyst`. A past WSL `E_ACCESSDENIED` startup problem stopped occurring after a targeted Windows ownership/access correction. During the 22 October session the endpoint started, logged locally, and generated a Wazuh alert through the relay path. Do not repeat permission changes or filesystem recovery unless the startup error recurs.

### Existing evidence

- Three intentionally generated failed SSH password attempts were followed by one successful login in the lab. This was a controlled simulation, not a real compromise.
- The source address observed by Alpine was the WSL-side host/gateway address; it should not be described as an external attacker.
- A default Nmap TCP scan observed SSH as the only open TCP service found in that scan. This is not a comprehensive vulnerability assessment.
- A normal SSH packet capture was inspected; TCP metadata was visible, while the encrypted payload did not expose interactive commands.
- Directly replacing the active Docker data disk with the previous disk failed during Docker Desktop provisioning. The old data copy remains archived; the attempt is not considered a successful recovery. Do not repeat manual data-disk replacement.

## Execution plan — current fixed order (22 October 2026)

1. **Persist Alpine forwarding:** locate the Alpine startup configuration and change the syslog target from direct UDP/514 to the WSL-gateway relay listener on UDP/15144. Preserve dynamic gateway discovery.
2. **Make the relay restart-safe:** define a user-scoped Windows startup task that launches only after WSL and Docker are available; retain gateway-only binding, dynamic Alpine IP discovery, and operational logs.
3. **Validate persistence:** after the configuration changes are made, perform one planned restart-recovery check and confirm a normal syslog marker becomes an alert with `logall_json=no`.
4. **Verify search/UI:** confirm the alert is searchable in the Indexer/Dashboard and recheck Indexer cluster health after the recent container restarts.
5. **Build SSH detection:** simulate repeated failed SSH authentication, validate rule/correlation behavior and false-positive limitations, and distinguish synthetic events from actual login attempts.
6. **Investigate and report:** build a timestamped incident timeline, sanitized evidence package, and short detection rationale.
7. **Document portability limits:** record that the direct Alpine-to-published-UDP path remains unresolved and that the current relay is a workaround, not proof of a Docker/WSL root cause.

## Working and safety rules

- No more Docker storage changes during endpoint troubleshooting.
- Do not use `docker compose down -v` for routine stopping.
- Do not use Factory Reset as a general troubleshooting step.
- Keep private keys, passwords, tokens, unsanitized PCAPs, personal identifiers, and machine-specific storage information out of public documentation.
- Update this tracker after a milestone is verified, not after every individual command.

## Official references

- [Wazuh: Docker deployment](https://documentation.wazuh.com/current/deployment-options/docker/index.html)
- [Wazuh: Syslog collection](https://documentation.wazuh.com/current/user-manual/capabilities/log-data-collection/syslog.html)
- [Microsoft: WSL command reference](https://learn.microsoft.com/windows/wsl/reference)


## Endpoint resumed; syslog integration next — 15 October 2026

User confirmed the Alpine endpoint command now returns the expected results after the targeted ownership fix, so the prior WSL access-denied blocker appears resolved. The next phase is event transport and proof of ingestion, not more Docker recovery. The Wazuh Manager currently has a secure agent listener on TCP/1514; a remote syslog receiver has not yet been verified/configured. Published ports are loopback-only, so endpoint reachability must be designed deliberately before sending events.

Next single phase:
1. Read the current Alpine route/interface and check the matching Windows WSL virtual IP.
2. Configure Wazuh Manager's documented syslog receiver with a narrow sender allowlist.
3. Make only the required ingestion port reachable from Alpine; keep Dashboard/Indexer local-only.
4. Send a harmless unique test marker and search for it in Wazuh.
5. Only then work on a controlled SSH-failure detection.

No local disk paths/sizes are recorded here. Do not make further Docker storage changes.


## End-of-day handover — 15 October 2026

### Verified today
- Docker Engine is healthy and all three Wazuh 4.14.8 services were reported Up.
- Dashboard login succeeded.
- Dashboard and Indexer are published on loopback only.
- AlpineLab starts after a targeted Windows file-ownership correction and reports Alpine 3.24.2; SSH and BusyBox syslog processes are present.
- Alpine interface and default route were inspected. Local logs are readable.
- The Wazuh Manager configuration still uses its secure Wazuh-agent listener; a distinct generic syslog receiver has not yet been configured.
- Alpine-to-Wazuh ingestion remains unproven.

### Shutdown for today
User was instructed to stop the Compose services with `docker compose stop`, quit Docker Desktop through the system tray, then run `wsl --shutdown` and verify distributions are stopped. Shutdown confirmation is pending.

### Next session — fixed sequence
1. Start Docker Desktop and verify `docker info`.
2. Start the existing Wazuh stack and check container status and Indexer health.
3. Confirm AlpineLab starts and recheck its route (do not make further ownership changes unless the error returns).
4. Make one planned configuration change: add a Wazuh-supported UDP syslog receiver with an explicit narrow sender allowlist, and configure only the needed ingestion path from Alpine. Preserve Dashboard/Indexer loopback access.
5. Send one unique benign marker and search for it in Wazuh.
6. After ingestion is proven, generate controlled failed SSH attempts and build/validate the alert and incident timeline.

No more storage changes are planned. Do not use `docker compose down -v` or repeat the previous Docker disk replacement attempt.


## Syslog receiver active; event proof pending — 19 October 2026

Latest user-provided command output confirms:
- Alpine BusyBox syslogd is running with remote forwarding enabled (`-R` and `-L`).
- The Wazuh Manager configuration inside the container has a `connection=syslog` block on UDP/514.
- Wazuh logs report allowed senders for the endpoint and Docker network gateway, and `wazuh-remoted` listening on 514/UDP.
- A uniquely marked test was confirmed in Alpine's local log, but wasn't found in Wazuh alert/archive files at that time.
- The earlier no-match check is inconclusive for transport because Wazuh archive logging may be disabled; events that don't trigger an alert won't appear in alert files.

Next: inspect `logall_json` in the mounted Manager configuration. If disabled, enable it temporarily to prove receipt in `archives.json`, send a fresh unique marker, verify arrival, and turn archive logging off again to avoid long-term disk growth. Do not declare end-to-end ingestion complete until the marker is visible at the Manager. Then check rule match / alert pipeline and verify searchable indexing separately.


## End-of-session handover — 19 October 2026

### Verified today
- Docker Engine is responding; all three Wazuh 4.14.8 services were observed running.
- Indexer cluster health returned `green`, with one node, 29 active primary shards, zero unassigned shards, and no pending tasks.
- Dashboard login succeeded earlier in the session.
- The project endpoint is registered as `AlpineLab`; Alpine returned its expected version and local logs. SSH and BusyBox syslog are running.
- Wazuh Manager now accepts the explicit remote syslog configuration and its logs report listening on UDP/514. The dashboard and indexer remain local-only.
- A unique synthetic test message was confirmed in Alpine's local log, but the marker was not found in Wazuh alert/archive files.
- The Manager config shows `logall=no` and `logall_json=no`. Therefore, the lack of an archive match is inconclusive; archive logging was disabled when the test was performed.
- The first receiver-config attempt had an invalid combined allowlist value; it was corrected to separate XML entries. Current `wazuh-remoted` remains running and there are no recent configuration errors in the last check.
- The temporary ingestion test script was prepared for the next step; it has not yet been confirmed run.

### Shutdown sequence
1. From the Compose project, run `docker compose stop` and verify container status.
2. Quit Docker Desktop from the system tray.
3. Run `wsl --shutdown` and `wsl --list --verbose` to confirm distributions are stopped.
4. Leave the portable lab media connected until Docker and WSL are fully stopped.
5. Do not use `docker compose down -v`, Factory Reset, or any storage cleanup.

### Next session — fixed plan
1. Start Docker Desktop; check `docker info`, start the existing Compose stack, and validate Indexer health.
2. Start/check `AlpineLab`; verify SSH and syslog processes if needed.
3. Run the prepared temporary archive-ingestion test once. It should back up the Manager config, enable JSON archive logging only for the test, send a new unique marker, search for it, and restore `logall_json=no`. Confirm the original setting was restored after the test even if it fails.
4. If the marker appears in the Manager's `archives.json`, receipt at the Manager is proven; then verify whether the event triggers a rule and whether it becomes searchable in the Dashboard.
5. If there is no match, do not change several variables at once. Check UDP transport and receiver logs with a bounded diagnostic.
6. Only after ingestion works, run controlled real failed SSH attempts and validate the detection/incident timeline.

No more Docker storage changes are planned. No machine-specific storage paths or virtual-disk sizes are recorded in this handover.


## Current execution checkpoint — 22 October 2026

### Verified in the practical session
- Wazuh Manager, Indexer, and Dashboard 4.14.8 were running after startup; Indexer health has not yet been rechecked after the most recent restarts.
- The Manager's original syslog configuration was restored after temporary tests: `logall_json=no`; the original allowlist contains `172.25.95.140` and `172.18.0.1`. Temporary test sender entries were removed.
- A temporary internal Docker-network test sender produced an exact match in `archives.json` while JSON archiving was temporarily enabled; configuration hash restoration was verified.
- A Windows UDP listener received a test datagram from AlpineLab at `172.25.95.140`.
- A Windows-generated UDP message to the published Manager endpoint was observed in the Manager network namespace as `172.18.0.1 -> 172.18.0.4:514`.
- A foreground Windows relay listened on the WSL gateway at UDP/15144, accepted a datagram from the current AlpineLab IP, and forwarded it to the published UDP/514 endpoint.
- Alpine's running BusyBox syslogd process was changed at runtime to forward to `<current WSL gateway>:15144`. A message sent via Alpine `logger` appeared in its local log, the relay reported forwarding the datagram, and the exact marker `SOC_LAB_NORMAL_SYSLOG_RELAY_20261022_210527` appeared in `alerts.json` under rule 5710, level 5.
- The verified workaround is the relay path. The precise cause of the direct Alpine-to-Docker-published-port failure is not yet identified.

### Pending — fixed order
1. Update the Alpine startup configuration so syslogd uses the relay destination port UDP/15144 after a distro restart.
2. Define relay startup order and create a user-scoped scheduled task only after verifying the script's behavior across WSL/network address changes and Docker startup.
3. Verify the same alert is searchable in the Indexer/Dashboard while `logall_json=no`.
4. Recheck Indexer health and validate relay recovery after a controlled WSL restart.
5. Develop repeated-failed-SSH detection, an investigation timeline, and a sanitized evidence package.
6. Update the project report/evidence after each milestone is actually verified.

No Docker storage changes, firewall rule changes, broad port bindings, or GitHub file changes were made by the diagnostic tests. Do not publish machine-specific storage paths or sizes.