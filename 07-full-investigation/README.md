# 07 - Full Security Investigation and Evidence Analysis

**Environment:** Wazuh Security Monitoring Lab
**Investigation Period:** August 11-16, 2026
**Timezone:** Eastern Daylight Time (EDT)
**Report Type:** Retrospective investigation of controlled security activity

This stage completes the Wazuh lab progression. Rather than focusing on configuration, the objective was to use telemetry already collected across earlier stages to reconstruct security-relevant activity, correlate related events, and separate confirmed evidence from analytical inference.

Because the events were intentionally generated across multiple controlled lab sessions, they are **not presented as one continuous incident**.

---

## Environment

| System | IP Address | Role |
|---|---|---|
| Wazuh Server | `192.168.71.128` | Centralized monitoring, detection, investigation, and response |
| Ubuntu Endpoint | `192.168.71.129` | Linux endpoint and SSH target |
| Windows Endpoint | `192.168.71.130` | Windows endpoint and controlled SSH source |

Evidence sources reviewed: Windows Security events, Linux authentication logs, Sysmon telemetry, Wazuh File Integrity Monitoring, Wazuh detection rules, Wazuh Active Response events, network connectivity observations, and Linux firewall state.

---

## Key Findings

- Windows Event ID `4720` recorded creation of the `Student1` local account.
- Approximately three seconds later, Event ID `4732` recorded `Student1` being added to the local Administrators group.
- Event ID `4726` recorded deletion of `Student1` approximately four minutes after creation.
- Linux telemetry recorded failed SSH authentication involving `fakeuser`, followed by successful SSH authentication involving `myDFIR`.
- Wazuh FIM recorded deletion activity in monitored Windows and Linux directories.
- Windows Event ID `4722` recorded enablement of the built-in `Guest` account.
- A Wazuh correlation rule (`100101`), configured and adapted from course reference material, correlated repeated SSH authentication failures from the same source during controlled testing.
- During a separate Active Response test, Wazuh recorded a `firewall-drop` response; connectivity was interrupted and DROP entries were observed in `iptables`.
- The evidence confirms the actions above but **does not independently establish malicious intent**.

---

## Investigation Timeline

| Time (EDT) | Host | Activity | Evidence |
|---|---|---|---|
| Aug 11, 2026 23:20:24.833 | Windows | `Student1` created | Event ID `4720` |
| Aug 11, 2026 23:20:27.960 | Windows | `Student1` added to Administrators | Event ID `4732` |
| Aug 11, 2026 23:24:21.348 | Windows | `Student1` deleted | Event ID `4726` |
| Aug 13, 2026 13:28:31.675 | Linux | Failed SSH authentication (`fakeuser`) | SSH telemetry |
| Aug 13, 2026 13:29:46.169 | Linux | Successful SSH authentication (`myDFIR`) | SSH telemetry |
| Aug 15, 2026 23:22:11.765 | Windows | File deleted from `C:\CompanyData` | Wazuh FIM |
| Aug 15, 2026 23:45:20.822 | Linux | File deleted from `/opt/company-data` | Wazuh FIM |
| Aug 15, 2026 23:55:15.483 | Windows | Built-in `Guest` account enabled | Event ID `4722` |
| Aug 16, 2026 14:20:29.563 | Linux | Repeated SSH failures correlated during rule validation | Rule `100101` |
| Aug 16, 2026 14:50:22.301 | Linux | Source blocked during separate Active Response test | Active Response |

> **Note:** The Rule `100101` alert and the Active Response event came from **separate tests**. They are not a single detection taking ~30 minutes to trigger a response.

---

## Detailed Investigation

### Windows Account-Management Activity

`Student1` was created (`4720`), added to local Administrators three seconds later (`4732`), and deleted about four minutes after creation (`4726`).

```
23:20:24.833 - Student1 created (4720)
23:20:27.960 - Student1 added to Administrators (4732)
23:24:21.348 - Student1 deleted (4726)
```

**Assessment:** the sequence is security-relevant because a newly created account quickly received administrative membership and was later removed. The telemetry proves the sequence occurred; it does not, by itself, prove malicious account manipulation.

### Guest Account Enablement

Event ID `4722` recorded enablement of the built-in `Guest` account, isolated using:

```
data.win.eventdata.targetUserName: Guest AND data.win.system.eventID: 4722
```

**Assessment:** unexpected Guest-account enablement would warrant investigation in a production environment — who performed the change, whether it was authorized, and whether the account subsequently authenticated.

### File Integrity Monitoring

Wazuh FIM recorded deletion activity in monitored directories on both endpoints (`C:\CompanyData` on Windows, `/opt/company-data` on Linux).

**Assessment:** FIM establishes that the monitored files were deleted. It does not, on its own, establish the responsible user, process, authorization, or intent — that requires correlation with authentication and endpoint telemetry.

### SSH Authentication Activity

Failed SSH authentication involving `fakeuser` was followed ~75 seconds later by successful SSH authentication involving `myDFIR`.

**Assessment:** these events involve different usernames. I do not characterize them as a failed attempt followed by success for the same account. A stronger relationship would require correlating source IP, SSH session/process identifiers, and surrounding events.

### Configured Detection - Repeated SSH Failures

I adapted a correlation rule from course reference material to flag three matching SSH authentication failures within 120 seconds from the same source IP:

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

**Assessment:** the rule confirms the configured detection criteria were met during a controlled test. It is evidence that the correlation logic works, not proof of a real adversary conducting brute-force activity.

### Active Response and Validation

The rule was connected to Wazuh Active Response:

```xml
<active-response>
  <disabled>no</disabled>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>100101</rules_id>
</active-response>
```

I validated the response independently rather than relying only on the Wazuh alert:

- Continuous ping from the Windows source began timing out
- `sudo iptables -L -n --line-numbers` showed DROP entries for the controlled source IP `192.168.71.130`

Recovery was performed manually (`iptables -D`) and confirmed via a follow-up `iptables -L`.

```
Detect → Correlate → Respond → Validate → Recover
```

---

## Who / What / When / Where / Why / How

- **Who:** Windows accounts involved were `Student1` and `Guest`. Linux authentication involved `myDFIR` and `fakeuser`. The controlled SSH source was the Windows endpoint (`192.168.71.130`).
- **What:** Account creation, privileged group membership change, account deletion, Guest enablement, file deletion, SSH authentication activity, repeated-failure correlation, and automated firewall response.
- **When:** Controlled sessions between August 11-16, 2026. All timestamps EDT.
- **Where:** Isolated Wazuh lab — Windows (`.130`), Ubuntu (`.129`), Wazuh server (`.128`).
- **Why:** Actions were intentionally generated to validate monitoring, detection, investigation, and response capabilities. In a production investigation, intent would require additional context beyond what's captured here.
- **How:** Endpoint activity generated Windows/Linux telemetry, Wazuh agents forwarded it centrally, field analysis reconstructed the activity, a configured rule correlated selected behavior, and Active Response executed containment logic.

---

## Recommendations

1. **Investigate unexpected account changes** — correlate creation, enablement, deletion, and privileged group membership with administrator identity, change records, and business justification.
2. **Monitor privileged group membership** — alert on unexpected additions to local Administrators; correlate with account creation and subsequent logons.
3. **Keep unnecessary accounts disabled** — the built-in Guest account should stay disabled unless explicitly required.
4. **Protect sensitive files with FIM** — monitor important directories for unexpected creation, modification, deletion, and permission changes, tuning scope to control noise.
5. **Investigate repeated SSH failures** — review source IP, target account, frequency, time window, and any subsequent successful authentication.
6. **Harden SSH access** — restrict to authorized sources, use strong authentication, limit unnecessary accounts.
7. **Use Active Response carefully** — define thresholds, allowlists, block duration, recovery procedures, and false-positive handling before broad deployment.

---

## Final Assessment

This stage demonstrated an end-to-end workflow: telemetry collection, search and filtering, evidence review, event correlation, configured detection, automated response, independent validation, and recovery.

The evidence was sufficient to establish **what** occurred, **when**, and **which systems and accounts** were involved. It was not sufficient, on its own, to establish malicious intent.

The central distinction throughout this investigation is between **observed evidence** and **inference**. Account changes, file deletion, repeated authentication failures, and response events are confirmed observations. Whether equivalent activity in a production environment is authorized, accidental, or malicious would require additional corroborating evidence.

---

## Project Progression

```
01 - Lab Architecture & Server Deployment
02 - Endpoint Integration & Sysmon
03 - Telemetry Generation & Analysis
04 - SOC Dashboard
05 - File Integrity Monitoring & Detection Configuration
06 - Active Response
07 - Full Security Investigation (this stage)
```

