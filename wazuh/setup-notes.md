# Wazuh EDR — Setup Notes

## Overview

Wazuh is deployed as the EDR (Endpoint Detection and Response) layer of the home SOC lab, complementing Suricata (network IDS) and Splunk (SIEM). While Suricata monitors network traffic, Wazuh monitors the host itself — file changes, running processes, system configuration, and vulnerabilities.

---

## Architecture

```
Ubuntu Server simu-s
├── Wazuh Single-Node Stack (Docker Compose)
│   ├── wazuh.manager
│   ├── wazuh.indexer
│   └── wazuh.dashboard  → https://<tailscale-ip>:8443
└── Wazuh Agent v4.11.2 (native) ← monitors simu-s itself
        └── connects to Manager via 127.0.0.1:1514/tcp
```

The Wazuh Agent runs natively on the same Ubuntu Server as the Manager.
It connects to the Manager via localhost (`127.0.0.1:1514/tcp`) since both are on the same host.

This mirrors the existing Splunk architecture:
- Wazuh Agent = Splunk Universal Forwarder
- Wazuh Manager = Splunk Enterprise
- Wazuh Dashboard = Splunk Dashboard

The Wazuh Dashboard is accessed remotely from another machine via Tailscale VPN — no ports are exposed to the internet.

---

## Deployment — Single-Node Stack (Ubuntu Server simu-s)

Clone the official Wazuh Docker repository and generate SSL certificates:

```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.11.2 --depth 1
cd wazuh-docker/single-node
docker compose -f generate-indexer-certs.yml run --rm generator
```

Edit `docker-compose.yml` to change the dashboard port from 443 to 8443
(port 443 was already in use by Tailscale):

```yaml
ports:
  - "8443:5601"
```

Deploy the stack:

```bash
sudo docker compose up -d
```

Three containers start automatically:
- `wazuh.manager`
- `wazuh.indexer`
- `wazuh.dashboard`

Access the dashboard at: `https://<tailscale-ip>:8443`
Default credentials: `admin / SecretPassword`

---

## Deployment — Agent (Ubuntu Server simu-s)

The Agent is installed natively on the same machine as the Manager to monitor the host itself.

### Version mismatch issue

The default apt repository installs Wazuh Agent v4.14.4, which is newer than the Manager (v4.11.2). Wazuh requires the agent version to be equal to or lower than the manager.

Fix — install the matching version explicitly:

```bash
sudo apt remove wazuh-agent -y
sudo apt install wazuh-agent=4.11.2-1 -y
```

### Configure agent to point to Manager

Edit `/var/ossec/etc/ossec.conf` and set the manager address to localhost:

```xml
<server>
  <address>127.0.0.1</address>
  <port>1514</port>
  <protocol>tcp</protocol>
</server>
```

`127.0.0.1` is used because the Manager is running as a Docker container on the same host.

### Start the agent

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
sudo systemctl status wazuh-agent
```

Confirm connection in logs:

```bash
sudo grep "Connected to the server" /var/ossec/logs/ossec.log
```

Expected output:
```
wazuh-agentd: INFO: Connected to the server ([127.0.0.1]:1514/tcp).
```

---

## Security Configuration Assessment (SCA)

### Problem

Wazuh v4.11.2 ships with only a CIS Ubuntu 22.04 policy. The server runs Ubuntu 24.04, so the scan was skipped with:

```
Skipping policy 'cis_ubuntu22-04.yml': 'Check Ubuntu version.'
```

### Fix

Download the Ubuntu 24.04 CIS benchmark policy from the Wazuh main branch:

```bash
sudo curl -o /var/ossec/ruleset/sca/cis_ubuntu24-04.yml \
  https://raw.githubusercontent.com/wazuh/wazuh/main/ruleset/sca/ubuntu/cis_ubuntu24-04.yml
```

Restart the agent to trigger the scan:

```bash
sudo systemctl restart wazuh-agent
```

### Results

| Policy | Passed | Failed | Not Applicable | Score |
|--------|--------|--------|----------------|-------|
| CIS Ubuntu Linux 24.04 LTS Benchmark v1.0.0 | 108 | 137 | 34 | 44% |

A score of 44% is typical for a freshly configured server with no dedicated hardening applied.
The 137 failed controls represent a clear remediation roadmap for future hardening work.

---

## Active Capabilities

| Module | Status | Details |
| :--- | :--- | :--- |
| **Agent Monitoring** | ` RUNNING ` | Reporting since Apr 3, 2026 |
| **File Integrity (FIM)** | ` ACTIVE ` | Monitoring: `/etc`, `/usr/bin`, `/usr/sbin` |
| **Vulnerability Detection** | ` ALERT ` | 47 Critical, 956 High, 1464 Medium CVEs |
| **Config Assessment (SCA)** | ` WARNING ` | CIS Ubuntu 24.04 — **44%** (108 passed / 137 failed) |
| **Rootcheck** | ` ACTIVE ` | System integrity verification operational |
---

## Useful Commands

```bash
# Check agent status
sudo systemctl status wazuh-agent

# View agent logs
sudo tail -f /var/ossec/logs/ossec.log

# Stop agent (free up RAM when not in use)
sudo systemctl stop wazuh-agent

# Start agent
sudo systemctl start wazuh-agent

# Check Manager containers
sudo docker ps | grep wazuh

# Restart full Wazuh stack
cd ~/wazuh-docker/single-node
sudo docker compose restart
```
