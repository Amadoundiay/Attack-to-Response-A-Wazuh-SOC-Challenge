<h1 align="center">Wazuh SOC Lab</h1>

<p align="center">
  <em>Detecting and responding to attacks on Windows and Linux endpoints with Wazuh, Sysmon, and Tines SOAR</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/SIEM-Wazuh-00A4E4?style=for-the-badge" alt="Wazuh">
  <img src="https://img.shields.io/badge/Telemetry-Sysmon-0078D4?style=for-the-badge" alt="Sysmon">
  <img src="https://img.shields.io/badge/SOAR-Tines-6C47FF?style=for-the-badge" alt="Tines">
  <img src="https://img.shields.io/badge/Platform-VMware-607078?style=for-the-badge" alt="VMware">
</p>

---

## Overview

Wazuh SOC Lab is a hands-on security operations project. Simulated attacks run against a Windows and a Linux endpoint. Wazuh collects their telemetry, detects suspicious activity with a custom rule, and triggers an automated response. The response is a Tines SOAR workflow that enriches the alert with threat intelligence and asks an analyst in Slack to approve the block before the attacker is blocked through the Wazuh API.

## Architecture

The lab runs on three virtual machines on the `192.168.160.0/24` network.

| Machine | Role | IP Address | Operating System |
|---------|------|------------|------------------|
| Wazuh Server | SIEM manager, indexer, and dashboard | 192.168.160.137 | Ubuntu |
| Windows Target | Victim endpoint with the Wazuh agent and Sysmon | 192.168.160.136 | Windows 10 |
| Ubuntu Target | Victim endpoint with the Wazuh agent and Sysmon for Linux | 192.168.160.138 | Ubuntu 24.04 |
| Kali Linux | Attacker machine | 192.168.160.139 | Kali Linux |

**Technologies:** Wazuh (SIEM, rules, active response), Sysmon for Windows and Linux, Filebeat, Tines (SOAR), Slack (analyst notifications), AbuseIPDB and VirusTotal (threat intelligence), and ngrok (exposes the on-premises Wazuh API to Tines).

## Wazuh Dashboard and Agents

Both endpoints register as active agents with the Wazuh manager, which receives their telemetry in real time.

<p align="center">
  <img src="images/wazuh-dashboard.png" alt="Wazuh dashboard overview" width="800">
</p>

<p align="center">
  <img src="images/wazuh-agents.png" alt="Connected Wazuh agents" width="800">
</p>

## Project Phases

Each phase has its own folder with the commands, logs, and captures for that step.

| # | Phase | Description | Folder |
|---|-------|-------------|--------|
| 1 | Generating Telemetry | Simulated account activity on Windows, SSH brute force and login on Ubuntu, and the events Wazuh recorded | [`01-generating-telemetry/`](01-generating-telemetry/) |
| 2 | Dashboard Creation | Building the Wazuh dashboards that visualize the collected telemetry | [`02-dashboard-creation/`](02-dashboard-creation/) |
| 3 | FIM and Detection | File integrity monitoring on sensitive folders and a custom rule (ID 100101) with active response | [`03-fim-and-detection/`](03-fim-and-detection/) |
| 4 | C2 Setup | Command and control setup and detection | [`04-c2-setup/`](04-c2-setup/) |
| 5 | SOAR | Tines workflow with threat intelligence, analyst approval, and Wazuh API blocking | [`05-soar/`](05-soar/) |

### Phase 1: Generating Telemetry
Windows discovery and account manipulation generated Security events (4720, 4732, 4726). Repeated failed SSH logins were followed by a successful login from the same source IP.

### Phase 2: Dashboard Creation
Dashboards were built to show authentication activity, account changes, and FIM alerts.

### Phase 3: FIM and Detection
FIM monitors `C:\CompanyData` on Windows and `/opt/company-data` on Linux. A custom rule flags three failed authentications within 120 seconds. When it fires, Wazuh runs `firewall-drop` on the endpoint that generated the alert.

### Phase 4: C2 Setup
Command and control setup and detection.

### Phase 5: SOAR with Tines
Wazuh sends rule 100101 alerts to Tines through a custom integration. Tines enriches the source IP with AbuseIPDB and VirusTotal, gets a block recommendation from an AI agent, and posts the evidence to Slack. The Wazuh API blocks the attacker only after the analyst approves.

## Repository Structure

```
wazuh-soc-lab/
├── README.md
├── images/                     # Architecture, dashboard, and agent captures
├── 01-generating-telemetry/    # Commands, logs, and captures
├── 02-dashboard-creation/      # Dashboard screenshots and build notes
├── 03-fim-and-detection/       # FIM configuration, custom rule, and active response
├── 04-c2-setup/                # C2 setup and detection
└── 05-soar/                    # Tines workflow, Wazuh integration, and Slack setup
```

## Security Notice

This lab runs on isolated virtual machines for learning. No credentials, API keys, webhook URLs, or tunnel addresses are stored in this repository. Use placeholders such as `<TINES_WEBHOOK_URL>` and keep real secrets in a password manager.

---

<p align="center">
  <sub>Built for hands-on learning in security operations.</sub>
</p>
