# Day 29 - Sentinel collection, detection, and automation

Status: Done

## Session

- **Lab:** Day 29 — Sentinel Collection, Detection, and Automation
- **Date/time and time zone:** October 8, 2026, 9:22 PM AEDT
- **Start/end time:** 21:22 - 22:55 AEDT
- **Hands-on, trial-dependent, or simulated:** Combined Hands-on Live Execution & Architectural Diagramming
- **Microsoft documentation checked on:** October 8, 2026
- **Region/SKU used:** Canada Central (`rg-sc500-sec-cc`) / Log Analytics Workspace `law-sc500-cc-01` / Microsoft Sentinel

## Cost Checkpoint

- **Credit remaining:** $283.80 AUD (Azure Subscription)
- **Month-to-date actual:** $0.20 AUD
- **Forecast:** ~$0.20 AUD (All operations executed under active Sentinel 31-day trial with low-volume telemetry)
- **Top services:** Microsoft Sentinel, Azure Monitor Data Collection Rules (DCR), Logic Apps, Azure Activity
- **Unexpected meters:** None
- **Action:** Kept data ingestion well below the 0.5 GB/day daily cap by using targeted DCR XPath filters and synthetic role-assignment test triggers.

## Business Control

- **Requirement:** Establish low-noise detections for high-privilege Azure RBAC modifications, automate triage workflows (incident tagging, severity scoping, owner assignment), and design efficient Linux/Windows hybrid log collection architectures without generating excessive log volume.
- **Threat or failure mode:** Persistence via unauthorized RBAC role grants, alert fatigue from noisy Syslog/Windows log collection, unassigned or untagged security incidents, and slow manual triage workflows.
- **Control Selected:** Scheduled Sentinel Analytics Rule + Azure Activity Log trigger + Native Sentinel Automation Rule (Tag, Assign, Scope) + Logic App Playbook (System-Assigned Managed Identity) + AMA DCR filtering.
- **Why this service/control fits:** Ensures immediate, deterministic detection of privileged RBAC changes while leveraging native, serverless automation to structure incident queues without paying high compute or storage overhead.
- **Rejected alternative and reason:** Broad Syslog/Windows "All Events" log ingestion. Rejected due to cost overruns and noise; replaced with narrow facility filters and custom XPath DCRs.

## Implementation

### Station 1 — Data Collection Architectures & AMA DCR Setup

#### Linux Syslog & CEF via AMA Architecture

To prevent disk saturation and log noise, Syslog collection is restricted to specific facilities (`auth`, `authpriv`, `daemon`) at severity `Warning` and above. CEF messages are forwarded via local rsyslog/syslog-ng daemon to the Azure Monitor Agent (AMA) running on a dedicated Linux Log Forwarder.

```
[ Security Appliance / Device ]
           │
           │ (CEF / Syslog over UDP/TCP Port 514)
           ▼
┌──────────────────────────────────────────────────────────┐
│ Linux Log Forwarder Machine                              │
│                                                          │
│  ┌────────────────────┐     (Port 28330 or Socket)       │
│  │ rsyslog / syslog-ng├───────────────────────────────┐  │
│  └────────────────────┘                               │  │
│                                                       ▼  │
│                                            ┌──────────┴─┐│
│                                            │ Azure      ││
│                                            │ Monitor    ││
│                                            │ Agent (AMA)││
│                                            └──────┬─────┘│
└───────────────────────────────────────────────────┼──────┘
                                                    │
                                                    │ (HTTPs / Logs Ingestion API)
                                                    ▼
                                    ┌──────────────────────────────┐
                                    │ Data Collection Rule (DCR)   │
                                    │ Stream: Custom-CommonSec*    │
                                    └───────────────┬──────────────┘
                                                    │
                                                    ▼
                                    ┌──────────────────────────────┐
                                    │ Sentinel Log Analytics       │
                                    │ - CommonSecurityLog Table    │
                                    │ - Syslog Table               │
                                    └──────────────────────────────┘
```

#### Windows Event Forwarding (WEF) Architecture

For Windows environments, source-initiated WEF pushes Security Event logs from domain endpoints to a central WEF Collector Server. The AMA agent is installed solely on the Collector Server, reading the forwarded event log and applying DCR XPath filters before streaming to Sentinel.

```
┌────────────────────────┐      ┌────────────────────────┐
│ Domain Endpoint A      │      │ Domain Endpoint B      │
└───────────┬────────────┘      └───────────┬────────────┘
            │                               │
            │ (Source-Initiated WEF / HTTP WinRM 5985)
            └───────────────┬───────────────┘
                            ▼
┌──────────────────────────────────────────────────────────┐
│ Central WEF Collector Server                             │
│ (ForwardedEvents Event Log Channel)                      │
│                                                          │
│ ┌──────────────────────────────────────────────────────┐ │
│ │ Azure Monitor Agent (AMA)                            │ │
│ └─────────────────────────┬────────────────────────────┘ │
└───────────────────────────┼──────────────────────────────┘
                            │
                            │ (DCR XPath Query Filtered)
                            ▼
            ┌────────────────────────────────┐
            │ Data Collection Rule (DCR)     │
            │ Event Set: Custom XPath Stream │
            └───────────────┬────────────────┘
                            │
                            ▼
            ┌────────────────────────────────┐
            │ Microsoft Sentinel             │
            │ SecurityEvent Table            │
            └────────────────────────────────┘
```

#### Custom XPath Filter applied in Windows Security Events DCR:

XML

```
<Select Path="Security">
  *[System[(EventID=4624 or EventID=4625 or EventID=4648 or EventID=4672 or EventID=4720 or EventID=4728 or EventID=4732)]]
</Select>
```

### Station 2 — Detection: Scheduled Analytics Rule

Created Scheduled Analytics Rule: **`[NCES] Privileged Azure Role Assignment Created`**.

- **Rule Query (KQL):**

Code snippet

```
AzureActivity
| where OperationNameValue =~ "MICROSOFT.AUTHORIZATION/ROLEASSIGNMENTS/WRITE"
| where ActivityStatusValue in~ ("Success", "Succeeded", "Start", "Started")
| project TimeGenerated, Caller, CallerIpAddress, ResourceGroup, ActivityStatusValue, CorrelationId, Properties
```

- **Entity Mapping:**
    - **Account:** `Caller` -> `Name`
    - **IP Address:** `CallerIpAddress` -> `Address`
    - **Azure Resource:** `ResourceGroup` -> `ResourceGroup`
- **Rule Logic & Scheduling:**
    - **Run Query Every:** 5 minutes
    - **Lookup Data From Last:** 5 minutes
    - **Alert Threshold:** Greater than 0
    - **Event Grouping:** Trigger an alert for each event
    - **Incident Settings:** Enabled (Create incidents from alerts generated by this analytics rule)
    - **Suppression:** Disabled

#### Benign Trigger Test

Executed test by creating and immediately deleting a temporary `Reader` role assignment for a lab identity:

1. Created assignment: `New-AzRoleAssignment -SignInName "sec-analyst-tier1@contoso.com" -RoleDefinitionName "Reader" -ResourceGroupName "rg-sc500-sec-cc"`
2. Verified Activity Log entry.
3. Removed assignment: `Remove-AzRoleAssignment -SignInName "sec-analyst-tier1@contoso.com" -RoleDefinitionName "Reader" -ResourceGroupName "rg-sc500-sec-cc"`

### Station 3 — Incident Automation & Playbook

#### Native Automation Rule

Configured Automation Rule: **`[Auto-Triage] Handle Privileged Role Assignment`**

- **Trigger:** When incident is created from rule `[NCES] Privileged Azure Role Assignment Created`
- **Conditions:** All conditions match
- **Actions:**
    1. **Add Tag:** `Identity-Privilege`, `RoleAssignment`, `NCES-Triage`
    2. **Assign Owner:** `sec-analyst-tier2@contoso.com`
    3. **Set Severity:** `Medium` (Escalated to `High` if target scope is Subscription root)
    4. **Add Analyst Task:**
        - "Verify requester UPN against approved Change Management ticket."
        - "Confirm whether role assigned is Owner/Contributor or custom identity role."
        - "Validate if assignment was auto-remediated or temporary."

#### Logic App Playbook Integration

Provisioned Logic App `playbook-nces-notify-secops` with **System-Assigned Managed Identity**.

- **RBAC Scope:** Assigned **Microsoft Sentinel Automation Contributor** to `playbook-nces-notify-secops` managed identity on Resource Group `rg-sc500-sec-cc`.
- **Playbook Action Sequence:**
    1. *Microsoft Sentinel Incident Trigger* -> 2. *Get Incident Entities* -> 3. *Post Message to Teams/Email Channel* -> 4. *Add Incident Comment* ("Notification dispatched to SecOps On-Call").

### Station 4 — Purview Audit Correlation (Day 27 Cross-Reference)

Queried Microsoft Purview Unified Audit logs to correlate the agent identity modified in Day 27 (`pathfinder-agent-prod`) with Azure Activity administrative actions.

Code snippet

```
// Correlation Query: Cross-referencing Agent ID operations with Azure Activity RBAC changes
let AgentActivity =
    PurviewAudit
    | where TimeGenerated > ago(24h)
    | where Operation in ("AgentIdentityModified", "AgentRegistryUpdated", "SharingSet")
    | project TimeGenerated, UserId, Operation, TargetResource = AuditData.TargetResource, ClientIP;
AzureActivity
| where TimeGenerated > ago(24h)
| where OperationNameValue =~ "MICROSOFT.AUTHORIZATION/ROLEASSIGNMENTS/WRITE"
| join kind=inner (AgentActivity) on $left.Caller == $right.UserId
| project TimeGenerated, Caller, OperationNameValue, Operation, TargetResource, ResourceGroup, CallerIpAddress
```

## Verification

| Test | Expected | Actual | Pass/fail | Evidence reference |
| --- | --- | --- | --- | --- |
| Benign Role Trigger | Grant `Reader` role in `rg-sc500-sec-cc` | Action logged in `AzureActivity` table within 2 mins | Pass | Azure Activity Query |
| Alert Generation | Analytics rule executes on 5-min schedule | Alert `[NCES] Privileged Azure Role Assignment Created` generated | Pass | Sentinel Alerts Queue |
| Automation Rule | Trigger native auto-triage rule | Incident tagged (`NCES-Triage`), assigned to Tier 2, tasks appended | Pass | Incident Timeline |
| Entity Mapping | Validate Incident Entities | `Caller` mapped to Account, `CallerIpAddress` mapped to IP | Pass | Incident Graph View |
| Playbook Access | Execute `playbook-nces-notify-secops` | Managed Identity authenticated; status 200 comment added | Pass | Logic App Run History |
| Incident Closure | Close test incident in Sentinel | Closed as **Closed - True Positive (Benign Test Activity)** | Pass | Incident Audit Log |

Code snippet

```
// Verification Query: Confirming Ingested Benign Role Assignment Alert & Entities
SecurityIncident
| where Title startswith "[NCES]"
| project TimeGenerated, IncidentNumber, Title, Severity, Status, Owner, Tags, RelatedAnalystTasks
| sort by TimeGenerated desc
```

## Findings

- **Misconfiguration discovered:** Default analytics rule frequency was set to 1 hour with a 1-hour lookback, causing up to a 60-minute triage delay for privilege escalation events.
- **Root cause:** Pre-packaged Content Hub rule templates default to relaxed scheduling to minimize workspace query unit consumption.
- **Remediation:** Reconfigured analytics rule execution schedule to **5 minutes** frequency with a **5 minutes** lookback window to achieve near-real-time detection.
- **Residual risk:** If Azure Activity log ingestion experiences Azure platform delays (>5 minutes), events may fall outside the tight lookback window.
- **Follow-up owner/date:** SOC Engineering Team / October 22, 2026.

## Teardown

- **Resources stopped/deleted:** Deleted temporary role assignment for `sec-analyst-tier1@contoso.com`.
- **Defender plan/add-on disabled:** Sentinel workspace `law-sc500-cc-01` retained in active trial mode.
- **Capacity/endpoint removed:** Logic App playbook trigger disabled to prevent unintended execution costs.
- **Cost view rechecked:** $0.20 AUD actual spend verified on Azure subscription.
- **Persistent dependency for next lab:** Analytics rule, automation rule, and workspace `law-sc500-cc-01` preserved for Day 30 capstone incident response simulation.

## Exam Recall

- **Three facts I can explain:**
    1. The **Azure Monitor Agent (AMA)** uses **Data Collection Rules (DCRs)** to apply precise XPath filters on Windows Event logs and Syslog facilities, significantly reducing unwanted log volume before ingestion.
    2. For **CEF/Syslog collection via AMA**, logs are received by a local daemon (`rsyslog`/`syslog-ng`) on port 514 and forwarded to the AMA agent (listening on port 28330/socket) for DCR processing.
    3. **Sentinel Automation Rules** execute serverless incident actions (tagging, assignment, severity change, tasks) natively without consuming Logic App workflow runs or requiring external API connections.
- **One confusing distinction:**
    - **Analytics Rules vs. Automation Rules:** Analytics Rules evaluate KQL queries against Log Analytics tables to generate alerts and create incidents. Automation Rules act *after* an incident is created, executing automated triage, tagging, owner assignment, or triggering Logic App playbooks.
- **One scenario question I wrote:**
    - *A security engineer needs to configure Microsoft Sentinel to collect only high-priority Windows Security events (such as account lockouts and user creations) from 500 domain-joined servers via Windows Event Forwarding (WEF). To minimize network overhead, agent management, and ingestion costs, which architecture should be deployed?*
        - A) Install the Azure Monitor Agent on all 500 domain servers with an "All Events" DCR.
        - B) **Configure source-initiated WEF to forward events to a single Collector Server, install AMA on the Collector, and apply a custom XPath DCR filter.** *(Correct)*
        - C) Install the Log Analytics Legacy Agent on all servers and configure Syslog UDP 514 forwarding.
        - D) Deploy a Logic App playbook to query each server's Event Viewer via WinRM every 5 minutes.