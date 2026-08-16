# Wazuh Security Monitoring Lab

## Overview

This repository documents my hands-on implementation and investigation of an on-premises security monitoring environment using Wazuh.

I built a centralized Wazuh deployment with Windows and Linux endpoints to collect and analyze security telemetry, investigate endpoint and authentication activity, develop custom detection logic, monitor file integrity, build security visualizations, and test automated response.

The project progressed from infrastructure deployment through telemetry collection, analysis, detection, response, and investigation.

My focus was not only on configuring Wazuh, but on understanding what the collected evidence showed, correlating related activity, distinguishing observations from assumptions, and making security conclusions supported by the available telemetry.

---

## Lab Environment

I built the environment using three virtual machines in VMware Workstation Pro:

| System | IP Address | Purpose |
| --- | --- | --- |
| Wazuh Server | `192.168.71.128` | Centralized security monitoring, detection, analysis, and management |
| Windows Endpoint | `192.168.71.130` | Windows Security and Sysmon telemetry |
| Ubuntu Endpoint | `192.168.71.129` | Linux, SSH, system, and Sysmon telemetry |

---

## Architecture

```text
                    ┌─────────────────────┐
                    │    Wazuh Server     │
                    │   192.168.71.128    │
                    │                     │
                    │ Manager             │
                    │ Indexer             │
                    │ Dashboard           │
                    └──────────┬──────────┘
                               │
                    Security Telemetry
                     ┌─────────┴─────────┐
                     │                   │
                     ▼                   ▼
          ┌──────────────────┐   ┌──────────────────┐
          │ Windows Endpoint │   │ Ubuntu Endpoint  │
          │ 192.168.71.130   │   │ 192.168.71.129  │
          │                  │   │                  │
          │ Wazuh Agent      │   │ Wazuh Agent     │
          │ Sysmon           │   │ Sysmon for Linux│
          │ Windows Events   │   │ SSH / Syslog    │
          └──────────────────┘   └──────────────────┘
```

---

## What I Implemented

Throughout the project, I:

- Deployed and configured an on-premises Wazuh server
- Enabled archive logging for expanded telemetry visibility
- Configured Filebeat to process Wazuh archive data
- Created the `wazuh-archives` index pattern for event analysis
- Deployed Windows and Ubuntu virtual machines
- Enrolled Windows and Linux endpoints using Wazuh agents
- Installed and configured Sysmon telemetry
- Validated Windows and Linux telemetry ingestion
- Generated controlled Windows account-management activity
- Generated failed and successful SSH authentication activity
- Investigated Windows and Linux security events
- Correlated account, authentication, session, and endpoint telemetry
- Built a three-panel security activity dashboard
- Configured File Integrity Monitoring on Windows and Linux
- Created a custom detection for Windows Guest-account enablement
- Created a correlation rule for repeated SSH authentication failures
- Configured Wazuh Active Response using `firewall-drop`
- Validated automated containment using network behavior and Linux firewall state
- Restored connectivity after response validation
- Conducted a retrospective security investigation
- Reconstructed an evidence-based timeline
- Developed security findings and recommendations

---

## Security Monitoring Dashboard

I built a three-panel dashboard to surface selected authentication and account-management activity across the monitored environment.

### Failed Windows Logons

The first panel monitors failed Windows authentication using Event ID:

```text
4625
```

I used the specific Windows event field:

```text
data.win.system.eventID
```

rather than searching for the value `4625` anywhere in the event.

### Windows Account Changes Over Time

The second panel monitors selected Windows account-management events:

```text
data.win.system.eventID: ("4720" OR "4722" OR "4723" OR "4724" OR "4725" OR "4726" OR "4732" OR "4733" OR "4738")
```

This provides visibility into activities such as account creation, enablement, deletion, modification, and local group membership changes.

### Linux Failed SSH Authentication Activity

The third panel provides visibility into failed SSH authentication associated with the Ubuntu endpoint.

Together, the three panels provide a concise operational view of selected authentication and account-management activity across the lab.

---

## Detection and Investigation Highlights

### Windows Account Activity

I generated and investigated Windows account-management telemetry including:

| Event ID | Activity |
| --- | --- |
| `4624` | Successful logon |
| `4625` | Failed logon |
| `4720` | User account created |
| `4722` | User account enabled |
| `4726` | User account deleted |
| `4732` | Member added to a security-enabled local group |

One sequence showed:

```text
Student1 Created
Event ID 4720
        ↓
~3 seconds
        ↓
Student1 Added to Administrators
Event ID 4732
        ↓
~3 minutes 53 seconds
        ↓
Student1 Deleted
Event ID 4726
```

Rather than treating these as isolated alerts, I correlated the events to reconstruct the account lifecycle and assess the security significance of the sequence.

The telemetry established what happened and when. Determining whether equivalent activity in a production environment was authorized or malicious would require additional context.

---

## SSH Authentication Analysis

I generated and investigated Linux SSH activity, including:

- Invalid-user authentication attempts
- Failed password events
- Successful authentication
- SSH session identifiers
- SSH process identifiers
- Source information
- Surrounding session events

I used Wazuh event fields and raw log information to understand the authentication activity and correlate related events.

This reinforced an important investigative principle: an individual authentication event rarely provides enough evidence to determine the complete security context.

For example, a failed authentication followed by a successful authentication should not automatically be treated as compromise without confirming whether the account, source, session, and surrounding activity actually correlate.

---

## File Integrity Monitoring

I configured Wazuh File Integrity Monitoring on both Windows and Linux.

### Windows

I monitored:

```text
C:\CompanyData
```

using:

```xml
<directories realtime="yes">C:\CompanyData</directories>
```

I generated file modification and deletion activity and confirmed that Wazuh recorded the changes.

### Linux

I monitored:

```text
/opt/company-data
```

I modified and deleted a test file within the monitored directory and validated the resulting Wazuh FIM events.

### Investigation Lesson

FIM can establish that a monitored file was created, modified, or deleted.

However, a FIM alert alone may not establish:

- Who performed the action
- Which process performed it
- Whether the activity was authorized
- Why the change occurred

Those conclusions may require correlation with authentication, process, endpoint, and other security telemetry.

---

## Custom Detection - Windows Guest Account Enablement

I created a custom Wazuh detection for enablement of the built-in Windows Guest account.

I first isolated the activity using:

```text
data.win.eventdata.targetUserName: Guest AND data.win.system.eventID: 4722
```

I then created a custom rule using the underlying Windows event fields:

```xml
<group name="windows,windows_security,account_changed,adduser">

  <rule id="100200" level="12">
    <if_sid>60103</if_sid>
    <field name="win.system.eventID">^4722$</field>
    <field name="win.eventdata.targetUserName">^Guest$</field>

    <description>MYDFIR-Omotara Windows Guest account was enabled.</description>

    <mitre>
      <id>T1078</id>
    </mitre>

    <group>
      windows,
      windows_account_management,
      account_enabled,
      guest_account,
    </group>
  </rule>

</group>
```

I generated controlled Guest-account enablement activity and confirmed that the expected event was visible in Wazuh.

---

## Custom Detection - Repeated SSH Authentication Failures

I also created a correlation rule to detect repeated SSH authentication failures from the same source.

```xml
<group name="local,syslog,sshd,authentication_failed,">
  <rule id="100101" level="10" frequency="3" timeframe="120">
    <if_matched_sid>5760</if_matched_sid>
    <same_source_ip />
    <description>Multiple SSH login failures observed from the same source IP</description>
    <mitre>
      <id>T1110</id>
    </mitre>
    <group>authentication_failed,ssh_bruteforce,credential_access,</group>
  </rule>
</group>
```

The detection logic evaluates:

```text
3 SSH Authentication Failures
             +
     Within 120 Seconds
             +
       Same Source IP
             ↓
       Custom Alert
```

I generated controlled authentication failures and confirmed that Wazuh generated the custom alert when the configured conditions were satisfied.

The alert confirmed that the detection logic worked as configured. It did not, by itself, establish that a real adversary was performing a brute-force attack.

---

## Active Response

After validating the repeated SSH authentication detection, I connected the rule to Wazuh Active Response.

```xml
<active-response>
  <disabled>no</disabled>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>100101</rules_id>
</active-response>
```

The configuration instructs Wazuh to execute the `firewall-drop` response locally on the endpoint associated with the alert generated by Rule `100101`.

### Response Validation

I did not rely only on the Wazuh alert to conclude that the response worked.

During controlled testing, I:

1. Confirmed connectivity between the Windows and Ubuntu endpoints
2. Generated repeated failed SSH authentication
3. Observed the custom detection
4. Observed network connectivity interruption
5. Confirmed the Wazuh firewall-drop event
6. Inspected the Ubuntu firewall state

I used:

```bash
sudo iptables -L -n --line-numbers
```

to confirm the resulting firewall entries.

The validation sequence was:

```text
Repeated SSH Failures
          ↓
Custom Detection
          ↓
Active Response
          ↓
Firewall Action
          ↓
Connectivity Interrupted
          ↓
iptables Validation
```

After testing, I removed the relevant firewall entries and restored connectivity.

---

## Full Investigation

I concluded the project by conducting a retrospective investigation using telemetry generated throughout the lab.

The investigation addressed:

- What happened?
- When did it happen?
- Which systems were involved?
- Which accounts were changed?
- Was file deletion observed?
- Was repeated SSH authentication activity observed?
- What evidence supported each finding?
- What remained unknown?
- What security recommendations should be made?

### Investigation Timeline

All timestamps are Eastern Daylight Time (EDT).

| Time | Host | Activity | Evidence |
| --- | --- | --- | --- |
| Aug 11, 2026 23:20:24.833 | Windows | `Student1` created | Event ID `4720` |
| Aug 11, 2026 23:20:27.960 | Windows | `Student1` added to Administrators | Event ID `4732` |
| Aug 11, 2026 23:24:21.348 | Windows | `Student1` deleted | Event ID `4726` |
| Aug 13, 2026 13:28:31.675 | Linux | Failed SSH authentication involving `fakeuser` | SSH telemetry |
| Aug 13, 2026 13:29:46.169 | Linux | Successful SSH authentication involving `myDFIR` | SSH telemetry |
| Aug 15, 2026 23:22:11.765 | Windows | Monitored file deleted | Wazuh FIM |
| Aug 15, 2026 23:45:20.822 | Linux | Monitored file deleted | Wazuh FIM |
| Aug 15, 2026 23:55:15.483 | Windows | Guest account enabled | Event ID `4722` |
| Aug 16, 2026 14:20:29.563 | Linux | Repeated SSH failures detected | Rule `100101` |
| Aug 16, 2026 14:50:22.301 | Linux | Source blocked during separate response test | Active Response |

> **Correlation Note:** The Rule `100101` alert at 14:20 EDT and the firewall-drop event at 14:50 EDT were generated during separate controlled tests. They should not be interpreted as a single detection taking approximately 30 minutes to trigger a response.

The final investigation focused on distinguishing what the telemetry directly established from conclusions that would require additional evidence.

---

## Key Lessons

### 1. Centralized telemetry creates investigative context

Windows Security events, Linux authentication logs, Sysmon, and Wazuh telemetry became significantly more useful when I could analyze them centrally.

### 2. Individual events rarely tell the complete story

A single account-creation event tells me an account was created.

Correlating account creation with privileged group membership and later deletion provides substantially more investigative context.

### 3. Detection is not confirmation

A detection rule firing confirms that the configured detection conditions were satisfied.

It does not automatically establish malicious intent.

### 4. File monitoring requires additional context

FIM can establish that a monitored file changed or was deleted, but attribution may require authentication, process, and endpoint telemetry.

### 5. Automated response must be validated

A response alert alone was not enough for me to conclude that containment worked.

I validated the response through network behavior and the resulting firewall state on the Linux endpoint.

### 6. Evidence should drive the conclusion

The investigation reinforced the importance of separating:

```text
Observed Evidence
       ↓
Analytical Interpretation
       ↓
Security Conclusion
```

A security conclusion should not exceed what the available evidence can support.

---

## Project Documentation

The project is organized into seven stages, progressing from infrastructure deployment through investigation and automated response validation.

| Stage | Documentation | Focus |
| --- | --- | --- |
| 01 | [Lab Architecture and Server Deployment](./01-lab-architecture-and-server-deployment/) | Wazuh server deployment, archive logging, Filebeat configuration, and telemetry validation |
| 02 | [Endpoint Integration and Sysmon](./02-endpoint-integration-and-sysmon/) | Windows and Linux agent enrollment, Sysmon integration, and telemetry validation |
| 03 | [Telemetry Generation and Analysis](./03-telemetry-generation-and-analysis/) | Windows account activity, SSH authentication, event analysis, and telemetry correlation |
| 04 | [SOC Dashboard](./04-soc-dashboard/) | Three-panel dashboard for Windows logons, account changes, and Linux SSH authentication |
| 05 | [File Integrity Monitoring and Custom Detection](./05-file-integrity-monitoring-and-custom-detection/) | Windows/Linux FIM and custom detection for Guest-account enablement |
| 06 | [Active Response](./06-active-response/) | Repeated SSH failure detection, automated firewall response, validation, and recovery |
| 07 | [Full Investigation](./07-full-investigation/) | Evidence correlation, timeline reconstruction, findings, assessment, and recommendations |

---

## Project Progression

```text
01 - Lab Architecture & Server Deployment
                    ↓
02 - Endpoint Integration & Sysmon
                    ↓
03 - Telemetry Generation & Analysis
                    ↓
04 - SOC Dashboard
                    ↓
05 - File Integrity Monitoring & Custom Detection
                    ↓
06 - Active Response
                    ↓
07 - Full Investigation
```

The project progressed from building the monitoring infrastructure to collecting telemetry, analyzing endpoint activity, developing visualizations and custom detection logic, configuring automated response, and conducting an evidence-based security investigation.

---

## Documentation Approach

I documented each stage based on work I actually performed and validated.

My documentation focuses on:

- What I built
- What I configured
- How I validated the configuration
- What telemetry was collected
- What the evidence showed
- How related events were correlated
- What problems I encountered and how I resolved them
- What security conclusions the evidence supported
- What the evidence did not prove
- What I would investigate or improve further

---

## Technologies and Concepts

**Security Monitoring:** Wazuh, SIEM, XDR, centralized logging

**Endpoint Telemetry:** Sysmon, Sysmon for Linux, Windows Event Logs

**Operating Systems:** Windows 10, Ubuntu Server 24.04

**Virtualization:** VMware Workstation Pro

**Detection:** Wazuh custom rules, event correlation, MITRE ATT&CK

**Monitoring:** File Integrity Monitoring, authentication monitoring

**Response:** Wazuh Active Response, `firewall-drop`, `iptables`

**Investigation:** Event correlation, timeline reconstruction, evidence analysis

**Administration:** PowerShell, Linux CLI, SSH, RDP

---

## Conclusion

This project gave me hands-on experience across an end-to-end security monitoring workflow:

```text
Deploy
  ↓
Collect
  ↓
Analyze
  ↓
Visualize
  ↓
Detect
  ↓
Investigate
  ↓
Respond
  ↓
Validate
```

The most valuable part of the project was moving beyond simply generating alerts.

I used the collected telemetry to understand what happened, correlate related activity, reconstruct timelines, determine what the available evidence could support, identify what remained unknown, and validate whether the configured response actually affected the endpoint.

The project reinforced that effective security monitoring is not just about seeing an alert. It is about using evidence to determine what happened, understanding the security significance of the activity, and making a defensible decision about what should happen next.

---

> **Note:** This repository represents a controlled cybersecurity lab environment. All security-relevant activity documented in this project was intentionally generated for testing, detection, investigation, and response validation.
