# Master Tracker — SOC Home Lab

**Updated:** 15 October 2026  
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
- [ ] Confirm current Indexer cluster health after this deployment.
- [ ] Configure a genuine Wazuh syslog receiver; publishing a UDP port alone does not configure a listener.
- [ ] Prove endpoint event ingestion using a unique harmless test marker.
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

All currently published host ports are bound to loopback:
- Dashboard: host 443 to container 5601/TCP.
- Indexer: host 9200 to container 9200/TCP.
- Manager: host 2514 to container 1514/TCP; host 2515 to container 1515/TCP; UDP 514 and TCP 55000 are also currently loopback-bound.

The Manager configuration currently shows its standard secure agent listener on TCP/1514. It does not yet have a verified remote syslog listener. Before endpoint ingestion, configure the Wazuh-supported syslog receiver with a narrow sender allowlist, and then make only the required ingestion path reachable from the endpoint.

### Alpine endpoint

The WSL distribution name used for the project endpoint is `AlpineLab`. Earlier verification reported Alpine 3.24.2 and user `analyst`. The latest startup attempt failed with `Wsl/Service/CreateInstance/MountDisk/HCS/E_ACCESSDENIED`. This error alone does not establish that the filesystem is corrupt. Do not unregister the distro, alter ACLs, or replace its filesystem image before the cause is investigated.

### Existing evidence

- Three intentionally generated failed SSH password attempts were followed by one successful login in the lab. This was a controlled simulation, not a real compromise.
- The source address observed by Alpine was the WSL-side host/gateway address; it should not be described as an external attacker.
- A default Nmap TCP scan observed SSH as the only open TCP service found in that scan. This is not a comprehensive vulnerability assessment.
- A normal SSH packet capture was inspected; TCP metadata was visible, while the encrypted payload did not expose interactive commands.
- Directly replacing the active Docker data disk with the previous disk failed during Docker Desktop provisioning. The old data copy remains archived; the attempt is not considered a successful recovery. Do not repeat manual data-disk replacement.

## Execution plan — fixed order

1. **Resolve Alpine startup:** inspect WSL/Windows security or attachment causes using read-only checks. Change only a demonstrated cause.
2. **Validate Wazuh health:** inspect service status, Manager logs, and Indexer cluster health.
3. **Configure syslog receiver:** use Wazuh's documented `connection=syslog` configuration, protocol/port, and a narrow `allowed-ips` value.
4. **Network minimally:** keep Dashboard and Indexer local-only; make only the chosen ingestion port reachable from the lab endpoint.
5. **Prove event ingestion:** send a uniquely marked benign event; find it in Wazuh and capture evidence.
6. **Build SSH detection:** run controlled failures, verify alert criteria and false-positive limitations.
7. **Publish:** add only redacted evidence and sanitized documentation.

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
