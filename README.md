# Wazuh SIEM Lab: Manager Startup Timeout Troubleshooting

![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04.5_LTS-E95420?logo=ubuntu&logoColor=white)
![Wazuh](https://img.shields.io/badge/Wazuh-4.14.8-005571)
![VirtualBox](https://img.shields.io/badge/VirtualBox-Lab-183A61?logo=virtualbox&logoColor=white)

Deployment verification and troubleshooting of a hands-on Wazuh SIEM lab, where the Wazuh Manager failed to start because systemd's startup timeout was exceeded.

## Lab Environment

| Component | Configuration |
|---|---|
| Operating System | Ubuntu 22.04.5 LTS |
| Virtualization | Oracle VirtualBox |
| Wazuh Version | 4.14.8 |
| SIEM Components | Wazuh Manager, Wazuh Indexer, Wazuh Dashboard |

## The Problem

The Wazuh Manager service entered a failed state:

```text
Active: failed
Result: timeout
start operation timed out. Terminating.
```

The systemd startup timeout was 45 seconds. Although the startup operation timed out, several Wazuh processes remained running in the background.

## Investigation

**1. Check the three main services**

```bash
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard --no-pager
```

The Manager had failed, while the Indexer and Dashboard were running.

**2. Review the Manager logs**

```bash
sudo journalctl -u wazuh-manager --no-pager
```

Wazuh components were being started, but the systemd startup operation eventually timed out.

**3. Inspect the systemd configuration**

```bash
systemctl show wazuh-manager -p Type -p TimeoutStartUSec
```

Service type was `forking`, with a startup timeout of `45s`.

**4. Check internal Wazuh processes**

```bash
sudo /var/ossec/bin/wazuh-control status
```

Several components were still running even though systemd had marked the service as failed.

**5. Stop the remaining processes**

`sudo systemctl stop wazuh-manager` did not fully terminate them. The service showed `KillMode=process` and no active `MainPID`, while child processes remained in the service cgroup. The Wazuh control utility stopped them cleanly:

```bash
sudo /var/ossec/bin/wazuh-control stop
sudo /var/ossec/bin/wazuh-control status
```

All components were then reported as not running.

## Solution

Increase the startup timeout from 45 to 180 seconds using a systemd override:

```bash
sudo mkdir -p /etc/systemd/system/wazuh-manager.service.d
printf '[Service]\nTimeoutStartSec=180s\n' | sudo tee /etc/systemd/system/wazuh-manager.service.d/timeout.conf
sudo systemctl daemon-reload
```

Verify the new value:

```bash
systemctl show wazuh-manager -p TimeoutStartUSec
# TimeoutStartUSec=3min
```

Start the Manager and check its status:

```bash
sudo systemctl start wazuh-manager
sudo systemctl status wazuh-manager --no-pager -l
```

The service reached `Active: active (running)` and systemd reported `Started Wazuh manager.`

## Post-Resolution Verification

### Services

```bash
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard --no-pager
```

| Service | Status |
|---|---|
| Wazuh Manager | active (running) |
| Wazuh Indexer | active (running) |
| Wazuh Dashboard | active (running) |

### Internal components

```bash
sudo /var/ossec/bin/wazuh-control status
```

The main components were running, including `wazuh-modulesd`, `wazuh-monitord`, `wazuh-logcollector`, `wazuh-remoted`, `wazuh-syscheckd`, `wazuh-analysisd`, `wazuh-execd`, `wazuh-db`, `wazuh-authd` and `wazuh-apid`.

The optional cluster, mail, agentless, integration and C-Syslog daemons were not running. They were not required for this standalone lab.

### Endpoint connectivity

```bash
curl -k -sS -o /dev/null -w "Wazuh API HTTP: %{http_code}\n" https://127.0.0.1:55000/
# Wazuh API HTTP: 401

curl -k -sS -o /dev/null -w "OpenSearch HTTP: %{http_code}\n" https://127.0.0.1:9200/
# OpenSearch HTTP: 401
```

> **Note:** HTTP 401 confirms that each endpoint responds and requires authentication. It does not verify successful authentication or full application-level integration. `-k` skips TLS certificate verification and is used here for local lab testing.

## Result

- Wazuh Manager started successfully.
- Wazuh Indexer and Dashboard remained operational.
- The Wazuh API and OpenSearch endpoints responded.
- The Wazuh Dashboard was accessible and working.

## Important Observation

Increasing the timeout resolved the observed startup failure in this lab. It does **not** establish why the Manager was slow to initialize. Further performance investigation may be needed to find the root cause.

## Skills Practiced

- Ubuntu Linux administration
- systemd service management and troubleshooting
- Journal log analysis
- Wazuh deployment verification and process inspection
- API and OpenSearch connectivity testing
- Technical documentation

## Next Steps

- [ ] Verify Filebeat forwarding to the Indexer (`systemctl status filebeat`, `filebeat test output`)
- [ ] Investigate the cause of the slow Manager startup
- [ ] Deploy an endpoint agent
- [ ] Collect security events and investigate alerts
- [ ] Test detection rules
- [ ] Deploy the Cowrie honeypot and integrate its logs with Wazuh
- [ ] Collect and analyze threat intelligence
