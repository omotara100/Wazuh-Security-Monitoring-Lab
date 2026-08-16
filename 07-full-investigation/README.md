# Full Security Investigation and Evidence Analysis

## Overview

In this final stage, I conducted a retrospective security investigation using telemetry collected throughout my Wazuh security monitoring lab.

Rather than focusing on configuration, the objective was to use the evidence already collected to reconstruct security-relevant activity, correlate related events, determine what the telemetry could prove, identify what remained unknown, and develop appropriate security recommendations.

The investigation focused on answering:

- What happened?
- When did it happen?
- Which hosts were involved?
- Which accounts were changed?
- Was a file deleted?
- Was SSH abuse observed?
- What should be recommended?

The events analyzed were intentionally generated across multiple controlled lab sessions between August 11 and August 16, 2026. They are therefore **not presented as one continuous security incident**.

The purpose of this investigation is to demonstrate how I use centralized telemetry to reconstruct activity and distinguish confirmed evidence from analytical inference.

---

# Investigation Scope

The investigation involved the following systems:

| System | IP Address | Role |
|---|---|---|
| Wazuh Server | `192.168.71.128` | Centralized monitoring, detection, investigation, and response |
| Ubuntu Endpoint | `192.168.71.129` | Linux endpoint and SSH target |
| Windows Endpoint | `192.168.71.130` | Windows endpoint and controlled SSH source |

Telemetry sources included:

- Windows Security events
- Linux authentication logs
- Sysmon telemetry
- Wazuh File Integrity Monitoring
- Wazuh custom detection rules
- Wazuh Active Response events
- Network connectivity observations
- Linux firewall state

---

# Findings

My retrospective investigation identified the following confirmed activity:

1. A local Windows account named `Student1` was created.

2. Approximately three seconds later, `Student1` was added to the local Administrators group.

3. The `Student1` account was subsequently deleted approximately four minutes after its creation.

4. Failed SSH authentication involving the intentionally invalid `fakeuser` account was observed on the Linux endpoint.

5. Successful SSH authentication involving the valid `myDFIR` account was subsequently observed.

6. Wazuh File Integrity Monitoring recorded file deletion activity within monitored Windows and Linux directories.

7. Windows Event ID `4722` recorded enablement of the built-in `Guest` account.

8. Custom Wazuh Rule `100101` successfully correlated repeated SSH authentication failures from the same source during controlled testing.

9. During a separate Active Response test, Wazuh recorded a `firewall-drop` response, connectivity from the controlled source was interrupted, and firewall DROP entries were observed on the Ubuntu endpoint.

Because these activities were intentionally generated across different lab sessions, they do not represent a single compromise.

The investigation demonstrates how I used centralized telemetry to reconstruct activity, correlate related events, distinguish evidence from inference, and determine what additional evidence would be required before assigning malicious intent.

---

# Investigation Timeline

I reconstructed the following timeline from security-relevant telemetry collected across multiple controlled lab sessions.

All timestamps are displayed in **Eastern Daylight Time (EDT)**, consistent with the timestamps shown in my Wazuh environment.

| Time (EDT) | Host | Activity | Evidence |
|---|---|---|---|
| August 11, 2026 23:20:24.833 | Windows | `Student1` local account created | Event ID `4720` |
| August 11, 2026 23:20:27.960 | Windows | `Student1` added to local Administrators group | Event ID `4732` |
| August 11, 2026 23:24:21.348 | Windows | `Student1` local account deleted | Event ID `4726` |
| August 13, 2026 13:28:31.675 | Linux | Failed SSH authentication involving `fakeuser` | SSH authentication telemetry |
| August 13, 2026 13:29:46.169 | Linux | Successful SSH authentication involving `myDFIR` | SSH authentication telemetry |
| August 15, 2026 23:22:11.765 | Windows | Monitored file deleted from `C:\CompanyData` | Wazuh FIM |
| August 15, 2026 23:45:20.822 | Linux | Monitored file deleted from `/opt/company-data` | Wazuh FIM |
| August 15, 2026 23:55:15.483 | Windows | Built-in `Guest` account enabled | Event ID `4722` |
| August 16, 2026 14:20:29.563 | Linux | Repeated SSH authentication failures correlated during rule validation | Rule `100101` |
| August 16, 2026 14:50:22.301 | Linux | Source blocked during separate Active Response test | Wazuh Active Response |

> **Investigation Note:** The Rule `100101` event at `14:20:29.563 EDT` and Active Response event at `14:50:22.301 EDT` were generated during separate tests. They should not be interpreted as a single detection taking approximately 30 minutes to produce a response.

---

# Investigation Summary

The investigation identified security-relevant activity affecting both monitored endpoints.

On the Windows endpoint, account-management telemetry showed the creation of a local account, addition of that account to the local Administrators group, subsequent deletion of the account, and enablement of the built-in Guest account.

File Integrity Monitoring also identified deletion activity within the monitored Windows directory.

On the Linux endpoint, authentication telemetry showed both failed and successful SSH authentication activity. File Integrity Monitoring recorded deletion activity within the monitored Linux directory.

During later testing, repeated failed SSH authentication events were correlated using a custom Wazuh detection rule.

The custom rule:

```text
Rule ID: 100101
```

identified repeated SSH authentication failures from the same source within the configured time window.

The detection was subsequently connected to Wazuh Active Response using:

```text
firewall-drop
```

During controlled Active Response testing, Wazuh recorded the response, connectivity from the controlled source was interrupted, and firewall DROP entries were observed on the Ubuntu endpoint.

The investigation therefore examined evidence spanning:

```text
Account Activity
       ↓
Authentication Activity
       ↓
File Activity
       ↓
Event Correlation
       ↓
Custom Detection
       ↓
Automated Response
       ↓
Response Validation
```

---

# Who, What, When, Where, Why, How

## Who

The investigation identified several accounts in the collected telemetry.

### Windows Accounts

```text
Bob
Student1
Guest
```

`Student1` was a temporary local account created during controlled Windows account-management testing.

The built-in `Guest` account was enabled during controlled testing.

`Bob` was the Windows account associated with activity generated from the Windows endpoint during earlier stages of the lab.

### Linux Accounts

```text
myDFIR
fakeuser
```

`myDFIR` was used for valid SSH authentication.

`fakeuser` was intentionally used to generate invalid SSH authentication telemetry.

The controlled SSH activity originated from the Windows endpoint:

```text
192.168.71.130
```

and targeted the Ubuntu endpoint:

```text
192.168.71.129
```

---

# What Happened?

The investigation identified four major categories of activity:

1. Windows account-management activity
2. File modification and deletion activity
3. Linux SSH authentication activity
4. Detection and automated response activity

Each category was analyzed independently before related events were correlated.

---

# Windows Account-Management Activity

## Student1 Account Sequence

The earliest correlated sequence involved the `Student1` local Windows account.

At:

```text
August 11, 2026 23:20:24.833 EDT
```

Windows Event ID `4720` recorded creation of the `Student1` account.

Approximately three seconds later, at:

```text
August 11, 2026 23:20:27.960 EDT
```

Event ID `4732` recorded `Student1` being added to the local Administrators group.

At:

```text
August 11, 2026 23:24:21.348 EDT
```

Event ID `4726` recorded deletion of the account.

The sequence was:

```text
23:20:24.833 EDT
Student1 Created
Event ID 4720
        ↓
~3 seconds
        ↓
23:20:27.960 EDT
Student1 Added to Administrators
Event ID 4732
        ↓
~3 minutes 53 seconds
        ↓
23:24:21.348 EDT
Student1 Deleted
Event ID 4726
```

The telemetry establishes that the account was created, granted local administrative membership, and subsequently deleted.

It does **not** independently establish malicious intent.

This sequence demonstrates why related account-management events are more useful when correlated rather than evaluated independently.

---

# Guest Account Activity

At:

```text
August 15, 2026 23:55:15.483 EDT
```

Windows Event ID `4722` recorded enablement of the built-in `Guest` account.

The relevant telemetry could be isolated using:

```text
data.win.eventdata.targetUserName: Guest AND data.win.system.eventID: 4722
```

This behavior was also used as the basis for my custom Guest-account detection:

```xml
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
```

The evidence establishes that the Guest account was enabled.

It does not independently establish why the account was enabled or whether the activity was authorized.

---

# SSH Authentication Activity

On August 13, I identified both unsuccessful and successful SSH authentication activity against the Linux endpoint.

At:

```text
August 13, 2026 13:28:31.675 EDT
```

Linux authentication telemetry recorded failed SSH authentication involving:

```text
fakeuser
```

At:

```text
August 13, 2026 13:29:46.169 EDT
```

approximately 75 seconds later, Linux telemetry recorded successful SSH authentication involving:

```text
myDFIR
```

The sequence was:

```text
13:28:31.675 EDT
Failed SSH Authentication
User: fakeuser
        ↓
~75 seconds
        ↓
13:29:46.169 EDT
Successful SSH Authentication
User: myDFIR
```

These events involve **different usernames**.

Therefore, I would not conclude that a failed authentication attempt against one account subsequently succeeded.

The evidence supports only that failed SSH authentication involving `fakeuser` was followed by successful SSH authentication involving `myDFIR`.

Additional correlation using source IP addresses, SSH session identifiers, process identifiers, and surrounding events would be required before drawing a stronger relationship between the two events.

---

# File Integrity Activity

Wazuh File Integrity Monitoring provided evidence of file-system activity on both endpoints.

## Windows

The monitored directory was:

```text
C:\CompanyData
```

At:

```text
August 15, 2026 23:22:11.765 EDT
```

Wazuh FIM recorded deletion activity within the monitored Windows directory.

## Linux

The monitored directory was:

```text
/opt/company-data
```

At:

```text
August 15, 2026 23:45:20.822 EDT
```

Wazuh FIM recorded deletion activity within the monitored Linux directory.

The FIM telemetry establishes that monitored files were deleted.

It does not independently establish:

```text
Who deleted the file
Why the file was deleted
Whether the deletion was authorized
Whether the deletion was malicious
```

Attribution would require correlation with authentication, process, user, and endpoint telemetry around the corresponding timestamps.

---

# Repeated SSH Authentication Detection

During the August 16 testing session, I extended the SSH investigation from individual authentication events into event correlation.

I created the following custom rule:

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

The rule used:

```text
Rule ID: 100101
Level: 10
Frequency: 3
Timeframe: 120 seconds
Correlation: Same source IP
MITRE ATT&CK: T1110 - Brute Force
```

At:

```text
August 16, 2026 14:20:29.563 EDT
```

Wazuh generated:

```text
Multiple SSH login failures observed from the same source IP
```

This confirmed that the custom correlation rule was working during the controlled rule-validation test.

The alert demonstrates that the configured correlation conditions were satisfied.

It does not independently prove that a real adversary was conducting a brute-force attack.

---

# Active Response Investigation

After validating Rule `100101`, I configured Wazuh Active Response:

```xml
<active-response>
  <disabled>no</disabled>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>100101</rules_id>
</active-response>
```

This connected:

```text
Rule 100101
      ↓
firewall-drop
```

During a **separate Active Response test**, I generated another controlled set of failed SSH authentication attempts.

Before the test, connectivity from the Windows endpoint to Ubuntu was confirmed using:

```powershell
ping 192.168.71.129
```

I then started continuous connectivity monitoring:

```powershell
ping 192.168.71.129 -t
```

and generated repeated failed SSH authentication attempts.

During the test, the ping began returning:

```text
Request timed out
```

and the SSH connection was interrupted.

At:

```text
August 16, 2026 14:50:22.301 EDT
```

Wazuh recorded the firewall-drop Active Response event.

---

# Firewall Validation

I did not rely solely on the Wazuh alert to determine whether the response worked.

On the Ubuntu endpoint, I inspected the firewall using:

```bash
sudo iptables -L -n --line-numbers
```

The firewall contained DROP entries associated with the controlled source:

```text
192.168.71.130
```

This gave me three independent validation points:

```text
Wazuh Active Response Event
            +
Network Connectivity Interrupted
            +
iptables DROP Entry
```

Together, these observations demonstrated that the automated response resulted in an actual firewall change on the Ubuntu endpoint.

---

# Recovery

Because this was a controlled lab test, I manually restored connectivity after validating the response.

I removed the relevant firewall entries using:

```bash
sudo iptables -D INPUT 1
```

and:

```bash
sudo iptables -D FORWARD 1
```

I then inspected the firewall again:

```bash
sudo iptables -L -n --line-numbers
```

The DROP entries observed during testing were no longer present.

This completed the response lifecycle:

```text
Detect
  ↓
Correlate
  ↓
Respond
  ↓
Validate
  ↓
Recover
```

---

# When Did It Happen?

The analyzed activity occurred across controlled lab sessions between:

```text
August 11, 2026
and
August 16, 2026
```

All timestamps in this report are recorded in:

```text
Eastern Daylight Time (EDT)
```

Because the activities occurred across different sessions, I do not treat the entire timeline as one continuous security incident.

---

# Where Did It Happen?

The activity occurred within my isolated Wazuh lab environment.

### Windows Endpoint

```text
IP: 192.168.71.130
```

Observed activity included:

- Local account creation
- Administrative group membership changes
- Account deletion
- Guest account enablement
- File Integrity Monitoring activity
- Controlled SSH connections toward Ubuntu

### Ubuntu Endpoint

```text
IP: 192.168.71.129
```

Observed activity included:

- Failed SSH authentication
- Successful SSH authentication
- Linux file-integrity events
- Repeated SSH authentication detection
- Active Response execution
- Firewall changes

### Wazuh Server

```text
IP: 192.168.71.128
```

The Wazuh server centralized the resulting telemetry and provided the investigation, detection, correlation, and response capabilities used throughout the analysis.

---

# Why Did It Happen?

The activity was intentionally generated as part of my controlled security monitoring lab.

Therefore, I know the cause and authorization of the activity in this environment.

However, during the investigation I separated that knowledge from what the telemetry itself could establish.

For example:

```text
Account Created
       +
Added to Administrators
       +
Account Deleted
```

would be security-relevant in a production environment, but those events alone would not prove malicious account manipulation.

Similarly:

```text
Repeated SSH Failures
```

could result from:

- User error
- Incorrect stored credentials
- Misconfigured automation
- Security testing
- Unauthorized password guessing
- Brute-force activity

Additional context would be required before assigning intent.

---

# How Did It Happen?

The activity was generated through controlled administrative, authentication, and file-system actions on the monitored endpoints.

Windows account-management actions generated Windows Security events.

SSH authentication attempts generated Linux authentication telemetry.

Changes to monitored files generated Wazuh FIM events.

Wazuh agents collected endpoint telemetry and forwarded it for centralized processing.

Custom rules were then used to identify specific behaviors and correlate repeated events.

The overall investigation workflow was:

```text
Endpoint Activity
       ↓
Telemetry Generated
       ↓
Wazuh Collection
       ↓
Search and Filtering
       ↓
Field Analysis
       ↓
Event Correlation
       ↓
Timeline Reconstruction
       ↓
Custom Detection
       ↓
Automated Response
       ↓
Response Validation
       ↓
Recovery
```

---

# Was a File Deleted?

**Yes.**

Wazuh File Integrity Monitoring identified file deletion activity on both monitored endpoints.

Confirmed deletion telemetry included:

```text
Windows:
August 15, 2026 23:22:11.765 EDT
C:\CompanyData

Linux:
August 15, 2026 23:45:20.822 EDT
/opt/company-data
```

The FIM events establish that file deletion occurred.

They do not independently establish who performed the deletion or whether it was malicious.

---

# Was SSH Abuse Observed?

Repeated failed SSH authentication activity was observed.

The evidence included:

- Failed password telemetry
- Invalid user activity
- Multiple failures from the same source
- Custom Rule `100101`

The custom rule correlated repeated SSH authentication failures within:

```text
3 matching events
within
120 seconds
from
the same source IP
```

Because I intentionally generated the authentication failures during controlled testing, I describe this evidence as:

> **Controlled repeated SSH authentication failures matching the configured detection criteria.**

I would not describe the lab activity itself as a confirmed real-world brute-force attack.

In a production environment, however, the pattern would warrant investigation for potential SSH password guessing or brute-force activity.

---

# Findings Summary

| Finding | Evidence | Assessment |
|---|---|---|
| `Student1` created | Event ID `4720` | Confirmed |
| `Student1` added to Administrators | Event ID `4732` | Confirmed |
| `Student1` deleted | Event ID `4726` | Confirmed |
| Guest account enabled | Event ID `4722` | Confirmed |
| Windows monitored file deleted | Wazuh FIM | Confirmed |
| Linux monitored file deleted | Wazuh FIM | Confirmed |
| Failed SSH authentication | Linux authentication telemetry | Confirmed |
| Successful SSH authentication | Linux authentication telemetry | Confirmed |
| Repeated SSH failures correlated | Rule `100101` | Confirmed |
| Automated firewall response | Wazuh Active Response | Confirmed |
| Network interruption following response | Continuous ping test | Observed |
| Firewall DROP entry | `iptables` inspection | Confirmed |
| Malicious intent | Available evidence | **Not established** |

---

# Recommendations

## 1. Investigate Unexpected Account Changes

Unexpected account creation, enablement, deletion, or group membership changes should be investigated.

Relevant events should be correlated with:

- Administrator identity
- Change-management records
- Authentication activity
- Endpoint activity
- Business justification

A sequence such as:

```text
New Account
     ↓
Added to Administrators
     ↓
Authentication
     ↓
Subsequent Activity
```

should receive greater scrutiny than an isolated account-management event.

---

## 2. Monitor Privileged Group Membership

Changes to privileged groups such as local Administrators should be monitored because they can materially change an account's access.

Alerting should focus on unexpected additions and correlate them with account creation and authentication events.

---

## 3. Keep Unnecessary Accounts Disabled

If the built-in Windows Guest account is not required, it should remain disabled.

Unexpected Guest-account enablement should trigger investigation into:

- Who enabled it
- Whether the action was authorized
- Whether the account subsequently authenticated
- What activity followed enablement

---

## 4. Protect Sensitive Files with FIM

File Integrity Monitoring should be applied to directories containing sensitive or operationally important information.

Monitoring should prioritize:

- Unexpected modifications
- File creation
- File deletion
- Permission changes
- Critical configuration changes

Monitoring scope should also be tuned to reduce unnecessary noise.

---

## 5. Investigate Repeated SSH Authentication Failures

Repeated SSH authentication failures should be investigated using:

- Source IP
- Target username
- Authentication result
- Frequency
- Time window
- Subsequent successful authentication
- Related endpoint activity

A particularly important sequence to investigate is:

```text
Repeated Failures
       ↓
Successful Authentication
```

especially when the same account or source is involved.

---

## 6. Harden SSH Access

Where appropriate, reduce SSH exposure by:

- Restricting access to authorized sources
- Using strong authentication controls
- Limiting unnecessary accounts
- Monitoring repeated authentication failures
- Reviewing successful logons following repeated failures
- Applying network restrictions appropriate to the environment

---

## 7. Use Automated Response Carefully

Automated containment can reduce response time, but it can also interrupt legitimate activity.

Before enabling automatic blocking broadly, define:

- Detection thresholds
- Time windows
- Allowlists
- Block duration
- Recovery procedures
- Critical-source exclusions
- Escalation procedures
- False-positive handling

A poorly scoped automated response could block legitimate administrators, shared infrastructure, or trusted services.

---

# Key Investigation Lessons

## Evidence Before Conclusion

I used the telemetry to establish what happened before attempting to determine why it happened.

For example:

```text
Evidence:
Student1 was created.

Evidence:
Student1 was added to Administrators.

Evidence:
Student1 was deleted.

Inference requiring additional context:
The activity was malicious.
```

This distinction prevents security conclusions from exceeding what the evidence supports.

---

## Correlation Changes the Meaning of Events

An individual event may have limited security significance.

A sequence can provide substantially more context.

For example:

```text
Account Created
      ↓
Administrator Membership
      ↓
Account Deleted
```

is more informative than reviewing Events `4720`, `4732`, and `4726` independently.

---

## Detection Does Not Equal Compromise

Rule `100101` proves that the configured correlation conditions were satisfied.

It does not prove that an adversary conducted a brute-force attack.

Likewise, Guest-account enablement proves the account was enabled but does not establish malicious intent.

---

## Response Must Be Independently Validated

I did not treat the Wazuh Active Response message alone as proof that containment succeeded.

I also observed:

```text
Ping timeout
+
iptables DROP entry
```

This provided independent evidence that the response affected connectivity.

---

# Final Assessment

The investigation confirmed multiple categories of security-relevant activity across the Windows and Linux endpoints, including:

- Account creation
- Privileged group membership changes
- Account deletion
- Guest-account enablement
- File deletion
- Failed SSH authentication
- Successful SSH authentication
- Repeated SSH authentication failures
- Custom detection
- Automated firewall response

The available telemetry allowed me to reconstruct significant portions of the activity by correlating:

```text
Windows Security Events
Linux Authentication Logs
Sysmon Telemetry
File Integrity Monitoring
Custom Wazuh Rules
Active Response Events
Network Behavior
Firewall State
```

Because the activities were intentionally generated inside a controlled lab, their cause and authorization were known.

The more important investigation objective was determining what the telemetry itself could prove independently of that knowledge.

The evidence supported conclusions about **what actions occurred, when they occurred, which systems and accounts were involved, and how Wazuh detected and responded to the activity**.

The evidence did not independently establish malicious intent.

That distinction between **observed evidence, analytical inference, and confirmed intent** was central to the investigation.

---

# Project Progression

This investigation completes the initial Wazuh lab progression:

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
07 - Full Security Investigation
```

Across these stages, I progressed from deploying the monitoring infrastructure to collecting telemetry, investigating endpoint activity, building visualizations, creating custom detection logic, configuring automated response, and reconstructing security activity from collected evidence.
