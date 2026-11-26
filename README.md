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
  <img src="06-images/1.png" alt="Wazuh dashboard overview" width="800">
</p>

<p align="center">
  <img src="06-images/2.png" alt="Connected Wazuh agents" width="800">
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

# Phase 1: Generating Telemetry

This phase simulates common activities an attacker might perform after gaining initial access to an environment. Before moving further, an attacker needs to understand the compromised system, identify available accounts, discover network information, and determine which users have administrative privileges.

The objective of this phase is to generate realistic security telemetry on both Windows and Linux endpoints and observe how Wazuh captures these activities.

> **Screenshots:** All captures for this phase are stored in `01-generating-telemetry/`, named `1.png` through `9.png`, plus `6.1.png`.

## 1. Host and Environment Discovery

After compromising a machine, an attacker rarely knows the internal environment. They need to gather information about the host, its network configuration, and available user accounts before deciding what to do next.

On the Windows target, I used the following commands to simulate this discovery activity:

| Command | Attacker's objective |
|---|---|
| `ipconfig /all` | Identify network interfaces, IP addresses, DNS servers, and network configuration. |
| `net user` | Enumerate local user accounts and identify potential targets. |
| `net localgroup administrators` | Identify accounts with administrative privileges. |

**Why this matters:** This information helps an attacker understand the machine's network position, identify privileged accounts, and assess potential opportunities for privilege escalation or lateral movement.

![Windows host and network discovery](01-generating-telemetry/1.png)

## 2. Account Manipulation

After discovering the available accounts and privileges, I simulated suspicious account manipulation on the Windows endpoint.

The actions included:
- Enabling the built-in Guest account.
- Setting a password for the Guest account.
- Creating a new account named `student1`.
- Adding `student1` to the local Administrators group.
- Deleting the account afterward.

**Attacker's objective:** Creating or modifying accounts and assigning administrative privileges can provide an attacker with an additional way to access and maintain control of a compromised system.

![Windows account manipulation](01-generating-telemetry/2.png)

## 3. Windows Event Detection in Wazuh

Wazuh collected the Windows Security events generated by the account manipulation activities.

I examined three important Windows Security Event IDs:

**Event ID 4720 – A user account was created**

Records the creation of the `student1` account. The SID ending in `1002` identifies the newly created account.

![Event 4720 - Account created](01-generating-telemetry/3.png)

**Event ID 4732 – A member was added to a security-enabled local group**

Records the addition of `student1` to the local Administrators group. The Member SID matches the SID observed in Event 4720, linking the two activities to the same account.

![Event 4732 - Account added to Administrators](01-generating-telemetry/4.png)

**Event ID 4726 – A user account was deleted**

Records the deletion of the `student1` account, completing the simulated account manipulation sequence.

![Event 4726 - Account deleted](01-generating-telemetry/5.png)

## 4. SSH Authentication Attempts – Linux Endpoint

I then moved to the Ubuntu target to simulate suspicious SSH authentication activity.

First, I attempted to authenticate using `fakeuser1`, a username that does not exist on the system. These attempts were rejected.

I subsequently authenticated using `ahmadou`, a valid local account, and successfully established an SSH session.

![SSH authentication attempts](01-generating-telemetry/6.1.png)

**Failed authentication attempts:**

Wazuh recorded the unsuccessful SSH authentication attempts, providing visibility into potential brute-force or username-enumeration activity.

![SSH failed authentication events](01-generating-telemetry/6.png)

![Additional SSH failed authentication events](01-generating-telemetry/7.png)

**Successful authentication:**

The subsequent successful login demonstrates why monitoring failed authentication attempts alone is insufficient. A successful authentication following multiple failures can indicate that an attacker eventually obtained valid credentials.

![SSH successful authentication event](01-generating-telemetry/8.png)

## 5. Security Significance

This phase demonstrates how seemingly ordinary system administration commands and authentication activities can become valuable indicators of compromise when observed in the right context.

The generated telemetry provides the foundation for the next phase: building dashboards to visualize authentication activity, account changes, and suspicious behavior.

More importantly, the failed SSH authentication attempts followed by a successful login establish the scenario used to develop and test a custom detection rule in Phase 3.

## Phase 2: Security Monitoring Dashboard

After generating telemetry on both endpoints, the next step was to build a centralized dashboard in Wazuh to provide visibility into authentication activity and account-related changes.

The objective was to transform raw security events into visual information that an analyst can quickly review to identify suspicious patterns and investigate potentially malicious activity.

### 1. Dashboard Overview

The dashboard contains three main visualizations covering Windows authentication, Windows account management, and Linux SSH authentication activity.

![Wazuh Security Monitoring Dashboard](02-dashboard-creation/1.png)

### 2. Windows Failed Logons

**Total failed logons: 11**

This visualization tracks unsuccessful Windows authentication attempts.

Monitoring failed logons helps identify potential password-guessing activity, brute-force attempts, and suspicious authentication patterns.

An analyst can use this information as a starting point to investigate:
- Which accounts are being targeted?
- Are multiple failed attempts coming from the same source?
- Are failures followed by successful authentication?
- Is the activity concentrated within a short period?

### 3. Windows Account Changes Over Time

This visualization tracks account-related security events over time, using a three-hour aggregation interval.

The dashboard monitors the following Windows Security Event IDs:

| Event ID | Description |
|---|---|
| 4738 | A user account was changed |
| 4722 | A user account was enabled |
| 4724 | An attempt was made to reset an account's password |
| 4732 | A member was added to a security-enabled local group |
| 4733 | A member was removed from a security-enabled local group |

Monitoring these events provides visibility into changes that could affect account security and access privileges.

For example, an unexpected account being added to the local Administrators group (Event ID 4732) could indicate an attempt to obtain elevated privileges or establish persistence.

### 4. Linux Failed SSH Authentication Activity

This table provides visibility into unsuccessful SSH authentication attempts collected from the Linux endpoint.

The dashboard displays:
- **Agent:** The endpoint generating the event.
- **Timestamp:** When the authentication attempt occurred.
- **Source user:** The username used during authentication.
- **Source IP:** The IP address associated with the attempt.
- **Destination user:** The destination account, when available.
- **Count:** Number of events grouped together.

In the simulated scenario, multiple failed authentication attempts were generated using the non-existent username `fakeuser1`.

This visualization helps an analyst identify targeted usernames, investigate repeated authentication failures, and identify potential brute-force activity.

### 5. Security Value

The dashboard brings together authentication and account-management telemetry from Windows and Linux endpoints into a single monitoring interface.

Rather than manually reviewing individual events, an analyst can quickly identify unusual activity and use the underlying security events for further investigation.

This phase establishes the monitoring and visualization layer of the lab. The next phase focuses on File Integrity Monitoring (FIM) and custom detection rules to move beyond visibility toward automated detection and response.

## Phase 3: FIM and Detection

After generating telemetry and building the monitoring dashboard, I moved on to implementing detection capabilities and automated response.

In this phase, I configured File Integrity Monitoring (FIM) on both Windows and Linux endpoints, created a custom Wazuh rule to detect repeated SSH authentication failures, and configured Active Response to automatically block the source IP responsible for the suspicious activity.

> **Screenshots:** All captures for this phase are stored in `03-fim-and-detection/`, named `1.png` through `9.png`.

### 1. File Integrity Monitoring (FIM) – Windows Endpoint

I started by configuring FIM on the Windows endpoint to monitor a directory containing potentially sensitive business data.

I created the directory `C:\CompanyData` and added a file named `payroll.txt`.

The objective was to simulate a situation where an attacker modifies or deletes sensitive files after gaining access to a system.

![Windows FIM monitored directory](03-fim-and-detection/1.png)

**Testing File Integrity Monitoring**

After creating the file, I modified its contents and then deleted it to generate file integrity events.

Wazuh successfully detected both activities and generated the following alerts:

- **Rule ID 550 (Level 7):** Integrity checksum changed – triggered when the file was modified.
- **Rule ID 553 (Level 7):** File deleted – triggered when the file was removed.

These events demonstrate how FIM can help detect unauthorized modifications or deletion of important files.

![Wazuh Windows FIM alerts](03-fim-and-detection/2.png)

### 2. File Integrity Monitoring (FIM) – Linux Endpoint

I then implemented the same monitoring approach on the Ubuntu endpoint.

I created the directory `/opt/company-data` and a file named `test.txt`.

To configure Wazuh to monitor this directory, I edited the agent configuration file:

```bash
sudo vi /var/ossec/etc/ossec.conf
```

After updating the configuration, I restarted the Wazuh agent:

```bash
sudo systemctl restart wazuh-agent.service
```

To test the configuration, I modified the file by appending new content:

```bash
echo "123" >> /opt/company-data/test.txt
```

Wazuh successfully detected the modification and subsequent deletion of the file, generating the expected FIM alerts:

- **Rule ID 550:** Integrity checksum changed.
- **Rule ID 553:** File deleted.

![Linux FIM modification alert](03-fim-and-detection/3.png)

![Linux FIM deletion alert](03-fim-and-detection/4.png)

### 3. Creating a Custom SSH Brute-Force Detection Rule

After validating FIM on both endpoints, I moved on to developing a custom detection rule for the SSH authentication activity simulated in Phase 1.

The objective was to detect repeated failed SSH authentication attempts originating from the same source IP address within a short period.

I created a custom rule in Wazuh's `local_rules.xml` configuration file.

**Detection logic:**

| Parameter | Value |
|---|---|
| Rule ID | `100101` |
| Severity | Level 10 – High |
| Parent rule | `5760` |
| Frequency | 3 failed authentication events |
| Timeframe | 120 seconds |
| Source correlation | Same source IP |
| MITRE ATT&CK | T1110 – Brute Force |

**How the rule works:**

- Wazuh first identifies SSH authentication failures matching the built-in rule ID `5760`.
- The correlation condition checks whether the events originate from the same source IP.
- After three matching events within 120 seconds, the custom rule triggers.
- Wazuh generates a Level 10 alert to highlight the suspicious authentication activity.

This allows me to correlate individual failed logins into a higher-level detection rather than treating each authentication failure as an isolated event.

![Custom Wazuh detection rule](03-fim-and-detection/7.png)

### 4. Simulating SSH Brute-Force Activity, Validating Detection and Response capability

With the custom rule configured, I returned to the Windows machine to simulate repeated SSH authentication failures against the Ubuntu endpoint (`192.168.160.138`).

Using Windows PowerShell, I attempted multiple SSH connections to the Linux machine using the valid username `ahmadou` but incorrect passwords.

As expected, the SSH server rejected the authentication attempts and returned multiple `Permission denied` messages.

![SSH brute-force simulation from Windows](03-fim-and-detection/9.png)

**Detection in Wazuh**

I then checked the Wazuh dashboard to verify whether the custom rule had detected the activity.

The rule successfully triggered:

- **Rule ID:** `100101`
- **Severity:** Level 10
- **Description:** Multiple SSH login failures observed from the same source IP.

Shortly afterward, Wazuh's Active Response module executed the `firewall-drop` script and generated another alert:

**Host Blocked by firewall-drop Active Response**

This confirmed that the suspicious authentication activity was not only detected but also triggered an automated response.

![Wazuh SSH brute-force detection and Active Response](03-fim-and-detection/8.png)

![iptables rules showing the blocked source IP](03-fim-and-detection/10.png)

### 5. Verifying the Automated IP Block

To make sure the response was actually applied, I accessed the Ubuntu endpoint and inspected its firewall rules:

```bash
sudo iptables -L -n --line-numbers
```

The output showed that Wazuh had automatically inserted DROP rules into the `INPUT` and `FORWARD` chains.

The source IP address of the Windows machine (`192.168.160.136`) appeared in the blocking rules.

This confirmed that the `firewall-drop` script had successfully modified the endpoint's firewall configuration to block traffic from the offending IP.

![iptables rules showing the blocked source IP](03-fim-and-detection/12.png)

**Cleaning Up the Environment**

After verifying the automated response, I manually removed the temporary firewall rules to restore connectivity:

```bash
sudo iptables -D INPUT 1
sudo iptables -D FORWARD 1
```

### Why It Matters

This phase allowed me to move beyond collecting and visualizing security events and start implementing actual detection and response capabilities.

Through this implementation, I demonstrated three important SOC capabilities:

1. **File Integrity Monitoring:** Detecting unauthorized modifications and deletions of files on Windows and Linux endpoints.
2. **Threat Detection:** Correlating multiple SSH authentication failures from the same source IP using a custom Wazuh rule.
3. **Automated Response:** Using Wazuh Active Response to automatically block the source IP through the endpoint firewall.

The result is a basic detection and response workflow in which suspicious authentication activity can be identified, correlated, and mitigated automatically without requiring an analyst to manually intervene at every step.

In the next phase, I explore Command and Control (C2) activity and how endpoint telemetry can help identify suspicious attacker behavior beyond authentication events.

## Phase 4: Command and Control (C2) Setup and Detection

After implementing FIM, custom detection rules, and automated response, I moved on to simulating a more advanced attack scenario involving Command and Control (C2) activity.

The objective was to simulate how an attacker can establish a remote connection to a compromised endpoint, execute commands remotely, perform post-exploitation activities, and attempt to maintain access to the system.

For this phase, I used Metasploit to establish reverse TCP Meterpreter sessions against the Windows and Linux environments and used Wazuh with Sysmon telemetry to observe the activities performed on the Windows endpoint.

> **Screenshots:** All captures for this phase are stored in `04-c2-setup/`, named `1.png` through `5.png`.

### 1. Generating Reverse TCP Payloads

I started by generating reverse TCP Meterpreter payloads on the Kali Linux attacker machine.

The objective was to prepare executable payloads for the Windows and Linux endpoints. When executed on a target, each payload attempts to establish an outbound connection back to the attacker's machine.

This simulates the establishment of an initial Command and Control channel, allowing the attacker to interact remotely with a compromised endpoint.

![Reverse TCP payload generation](04-c2-setup/1.png)

### 2. Establishing a C2 Connection

After generating the payloads, I configured Metasploit to listen for incoming connections using the `multi/handler` module.

I configured the handler with the corresponding payload and the attacker's IP address, then started the listener.

Once the Windows payload was executed on the target machine, a Meterpreter session was successfully established.

This provided a remote interactive shell through which I could execute commands and interact with the compromised system.

I then performed initial reconnaissance to identify the compromised user context and inspect running processes.

![Metasploit handler and established Meterpreter session](04-c2-setup/2.png)

### 3. Post-Exploitation: Privilege Escalation and Persistence

With the Meterpreter session established, I moved on to simulating common post-exploitation activities on the Windows endpoint.

The objective was to demonstrate how an attacker with remote access might attempt to obtain elevated privileges and maintain access to a compromised machine.

I used a PowerShell session to perform the following actions:

- **System reconnaissance:** Inspected the network configuration to understand the compromised machine's environment.
- **Account creation:** Created a new local account named `attacker`.
- **Privilege escalation:** Added the newly created account to the local Administrators group.
- **Registry-based persistence:** Created a registry Run key under `HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run`.

For the persistence test, I configured the registry entry to launch `calc.exe` at user logon. This was used as a benign executable to demonstrate the persistence mechanism without deploying another malicious program.

These activities simulate techniques an attacker could use after obtaining an initial foothold, particularly account manipulation, privilege escalation, and registry-based persistence.

![Windows post-exploitation activities](04-c2-setup/3.png)

### 4. Investigating Process Creation Events in Wazuh

After performing the post-exploitation activities, I moved to the Wazuh dashboard to investigate the telemetry generated on the Windows endpoint.

Using Sysmon process creation events, I examined the execution of the commands performed during the previous step.

The objective was to understand not only which commands were executed, but also how they were launched and which processes were responsible for initiating them.

By examining the process execution chain, I could trace the activity from the original payload to the commands executed during the post-exploitation phase.

This provides useful context for identifying suspicious process relationships and understanding how an attacker interacts with a compromised endpoint.

![Wazuh process creation telemetry](04-c2-setup/4.png)

### 5. Investigating Command-Line Activity and Registry Modifications

I then examined the detailed event fields collected by Wazuh to investigate the command-line arguments and registry modifications associated with the attack simulation.

Command-line telemetry is particularly valuable during incident investigations because it provides additional context about what a process was instructed to do.

I focused on two important activities:

- **Account manipulation:** Investigated the command-line events associated with creating the local account and adding it to the Administrators group.
- **Registry modification:** Examined the registry command used to create the Run key responsible for launching the executable at user logon.

By correlating process creation events with their command-line arguments and process metadata, I could reconstruct important parts of the attack sequence and identify the actions performed after the initial compromise.

![Wazuh command-line and registry modification telemetry](04-c2-setup/5.png)

### Why It Matters

This phase allowed me to simulate and investigate activities that occur after an attacker gains initial access to an endpoint.

It demonstrated several important aspects of endpoint investigation:

1. **Command and Control:** Establishing a remote interactive session between an attacker machine and a compromised endpoint.
2. **Post-Exploitation:** Performing reconnaissance and manipulating local accounts to obtain administrative privileges.
3. **Persistence:** Modifying the Windows Registry to maintain execution across user logons.
4. **Endpoint Visibility:** Using Wazuh and Sysmon telemetry to investigate process creation, command-line execution, and registry modifications.
5. **Attack Reconstruction:** Following process relationships and event details to understand the sequence of activities performed on the compromised machine.

Most importantly, this phase demonstrates why collecting endpoint telemetry is essential for incident response. Detecting an initial suspicious connection is useful, but understanding what happened afterward is equally important when investigating a compromised system.

The next phase focuses on integrating Wazuh with Tines to build a SOAR workflow that enriches security alerts, notifies an analyst, and automates response actions with human approval.

## Phase 5: SOAR Automation with Tines

In this phase, I integrated Wazuh with **Tines** to automate the investigation and response process for suspicious SSH authentication activity.

The workflow is triggered by my custom Wazuh rule **100101 – Multiple SSH login failures observed from the same source IP**. Once Wazuh generates the alert, it sends the event to Tines through a webhook. An AI Agent analyzes the alert, prepares an investigation report, and recommends whether the source IP should be blocked.

The workflow then sends the report to Slack and presents an approval form to the analyst. The analyst remains responsible for the final decision. If blocking is approved, Tines sends a request to the Wazuh API to execute the `firewall-drop` Active Response.

### 1. Tines automation workflow

![Tines SOAR workflow](05-soar/1.png)

I designed the workflow to connect alert ingestion, AI-assisted investigation, notification, analyst approval, and automated response.

The main steps are:

1. **Wazuh-Alert webhook:** receives the alert generated by Wazuh.
2. **AI Agent:** analyzes the alert and uses connected tools to investigate available indicators.
3. **Event Transform:** extracts the AI Agent's decision and the relevant IOC information.
4. **Conditional branching:** checks whether the AI recommends blocking the source IP.
5. **Slack notification and approval form:** sends the report to the SOC analyst and provides a blocking decision.
6. **Wazuh Active Response:** executes the blocking action if the analyst approves it.

A separate branch handles the decision not to block, allowing the analyst to retain control over containment.

### 2. Initial connectivity and SSH authentication tests

![Initial connectivity and SSH tests](05-soar/2.png)

Before testing the response workflow, I established the initial connectivity between the Windows machine (`192.168.160.136`) and the Ubuntu target (`192.168.160.138`).

I used continuous ping to verify that the target was reachable. I then attempted to authenticate over SSH using the account `ahmadou` and deliberately entered incorrect passwords.

The SSH attempts resulted in authentication failures, while the target remained reachable through ICMP. This provided the initial conditions for testing detection and containment.

### 3. Wazuh alert generation

![Wazuh alert details](05-soar/3.png)

The repeated SSH authentication failures triggered my custom detection rule:

- **Rule ID:** 100101
- **Description:** Multiple SSH login failures observed from the same source IP
- **Source IP:** `192.168.160.136`
- **Target user:** `ahmadou`
- **Target host:** `TEST-Linux`
- **SSH service:** `sshd`

The alert contains the authentication failure details, including the source IP and source port. Wazuh forwards this event to Tines, initiating the automated investigation workflow.

### 4. AI-assisted investigation and IOC extraction

![Tines AI Agent analysis](05-soar/4.png)

After receiving the alert, the AI Agent analyzes the available event information and produces a structured investigation result.

For this event, the agent recommends blocking the source IP and returns the following values:

- `recommend_block`: `true`
- `ioc_type`: `srcip`
- `ioc_value`: `192.168.160.136`

The Event Transform component extracts these values so that the workflow can evaluate the recommendation and follow the appropriate branch.

The AI Agent also has access to threat-intelligence tools, including AbuseIPDB and VirusTotal, to investigate supported indicators. Since the source IP in this lab is a private RFC1918 address, public reputation services cannot establish its public reputation. The recommendation is therefore based primarily on the observed activity in the lab, rather than a claim that external intelligence has confirmed the IP as malicious.

### 5. Slack notification and investigation report

![Slack incident notification](05-soar/5.png)

Once the investigation is completed, Tines sends a notification to the `#wazuh` Slack channel.

The report provides the analyst with the main incident details:

- **Detection:** Multiple SSH login failures from the same source IP
- **Rule ID:** 100101
- **Severity:** Level 10
- **Source IP:** `192.168.160.136`
- **Target:** `ahmadou` on `TEST-Linux` (`192.168.160.138`)
- **MITRE ATT&CK:** T1110 – Brute Force
- **Observed activity:** Three failed SSH password attempts within approximately 10 seconds
- **Recommendation:** Block the source IP and review the affected account and SSH security configuration.

The notification also includes the AI Agent's recommendation and a link to the analyst approval page.

This allows the analyst to receive the investigation context without having to manually inspect every event before deciding on the next action.

### 6. Analyst approval: block or don't block

![Analyst IP blocking approval form](05-soar/6.png)

The approval page presents the analyst with a simple decision:

**Do you want to block the IP?**

The analyst can select either **Yes** or **No**, then submit the decision.

This introduces a human-in-the-loop step into the workflow. The AI Agent provides an investigation and recommendation, but the analyst retains responsibility for authorizing the containment action.

For this test, I selected **Yes** to block `192.168.160.136`.

### 7. Approval submission confirmation

![Approval submission confirmation](05-soar/7.png)

After submitting the decision, the approval page displays a confirmation message:

**Thank you for your submission.**

This confirms that the analyst's decision was submitted to the workflow, allowing Tines to continue with the corresponding response branch.

### 8. Connectivity verification after containment

![Connectivity test after IP blocking](05-soar/8.png)

After approving the blocking action, I returned to the Windows machine and checked connectivity to the Ubuntu target again.

The continuous ping initially shows successful replies, followed by repeated **Request timed out** messages after the containment action.

This demonstrates the change in network connectivity after the response was triggered. The SSH window also shows authentication failures, although these alone do not prove that the firewall action succeeded.

### 9. Firewall verification

![Wazuh Active Response firewall rules](05-soar/9.png)

To verify the containment action directly, I inspected the firewall rules on the Ubuntu target using:

```bash
sudo iptables -L -n --line-numbers
```

The output shows DROP rules for source IP `192.168.160.136` in both the **INPUT** and **FORWARD** chains.

This confirms that the source IP was added to the firewall blocking rules following the approved Active Response.

### Why this matters

This phase demonstrates how I integrated detection, investigation, collaboration, analyst approval, and automated containment into a single SOC workflow.

The complete process is:

**Wazuh Detection → Tines Webhook → AI Investigation → Slack Notification → Analyst Approval → Wazuh Active Response → Firewall Verification**

By combining Wazuh, Tines, AI-assisted analysis, Slack, and Wazuh Active Response, I built a human-in-the-loop SOAR workflow that reduces manual investigation steps while keeping the final containment decision under analyst control.


<p align="center">
  <sub>Built for hands-on learning in security operations.</sub>
</p>
