# Day 28 - Sentinel onboarding, connectors, and data tiers

Status: Done

## Session

- **Lab:** Day 28 — Sentinel Onboarding, Connectors, and Data Tiers
- **Date/time and time zone:** October 8, 2026, 9:17 PM AEDT
- **Start/end time:** 21:17 - 22:50 AEDT
- **Hands-on, trial-dependent, or simulated:** Combined Hands-on & Safe-Lane Architectural Simulation
- **Microsoft documentation checked on:** October 8, 2026
- **Region/SKU used:** Canada Central (`rg-sc500-sec-cc`) / Log Analytics Workspace / Microsoft Sentinel (31-day trial)

## Cost Checkpoint

- **Credit remaining:** $283.80 AUD (Azure Subscription)
- **Month-to-date actual:** $0.20 AUD
- **Forecast:** ~$0.20 AUD (31-day trial active; daily ingestion capped to limit overage)
- **Top services:** Azure Monitor Log Analytics, Microsoft Sentinel, Azure Monitor Data Collection Rules (DCR)
- **Unexpected meters:** None
- **Action:** Enabled 31-day Microsoft Sentinel free trial on new workspace `law-sc500-cc-01` and configured a strict **0.5 GB/day** workspace ingestion cap.

## Business Control

- **Requirement:** Establish central security logging with 90 days of hot Analytics tier access for real-time detections, 13 months total retention for verbose investigation logs, and 7 years of low-cost compliance archive for selected audit evidence without incurring hot-tier storage overhead across all tables.
- **Threat or failure mode:** Runaway log ingestion costs, unmonitored custom application security logs, excessive identity permissions on security monitoring infrastructure, and permanent lock-in to Sentinel Data Lake architecture.
- **Control Selected:** Microsoft Sentinel unified onboarding in Defender portal + Azure RBAC fine-grained role scopes + Data Collection Rules (DCR) / Custom Tables (`AppSecEvents_CL`) + Tiered Retention Policy (Analytics vs. Auxiliary/Basic vs. Long-Term Archive).
- **Why this service/control fits:** Direct alignment with NCES cost-efficiency goals, balancing high-performance KQL detections on critical security tables with low-cost retention tiers for compliance.
- **Rejected alternative and reason:** Immediate native Sentinel Data Lake onboarding. **Decision Gate:** Selected the **Default Safe Lane**. Self-offboarding is unsupported without a Defender support request, same-region workspace attachments are enforced, and CMK-protected data is unsupported. Data lake pipeline operations are documented as architectural simulations.

## Implementation

### 1. Workspace Onboarding & Scoped Azure RBAC

- **Resource Group & Workspace:** Provisioned Log Analytics workspace `law-sc500-cc-01` in `rg-sc500-sec-cc` (Canada Central region).
- **Sentinel Enablement:** Onboarded Microsoft Sentinel via the unified Microsoft Defender portal (`security.microsoft.com`). Activated 31-day trial.
- **Ingestion Cap:** Configured workspace daily volume limit to **0.5 GB/day** (`DailyQuotaGb: 0.5`).
- **RBAC Role Matrix:**

| Identity / Principal | Role Assigned | Scope | Justification |
| --- | --- | --- | --- |
| `sec-analyst-tier1` | **Microsoft Sentinel Reader** | Workspace `law-sc500-cc-01` | Read-only access for SOC triage and workbook viewing. |
| `sec-analyst-tier2` | **Microsoft Sentinel Responder** | Workspace `law-sc500-cc-01` | Incident management, status changes, and alert handling. |
| `sec-engineer-lead` | **Microsoft Sentinel Contributor** | Resource Group `rg-sc500-sec-cc` | Full solution installation, analytics rule tuning, and connector setup. |
| `sec-automation-spn` | **Microsoft Sentinel Playbook Operator** | Resource Group `rg-sc500-sec-cc` | Automation rule execution and Logic App playbook triggers. |

### 2. Content Hub Solutions & Microsoft Connectors

Installed the following Content Hub solutions and initialized data connectors:

- **Azure Activity Connector:** Connected at subscription scope to stream control-plane administrative operations.
- **Defender for Cloud Connector:** Enabled tenant-wide security alert ingestion.
- **Microsoft Defender XDR Connector:** Configured incident and alert bi-directional sync (using synthetic lab telemetry).
- **Inventory Summary:**
    - *Data Connectors:* 3
    - *Analytics Rules:* 14 enabled (scheduled & NRT)
    - *Workbooks:* 4 installed
    - *Hunting Queries:* 22 imported

### 3. Custom Table Ingestion (Logs Ingestion API / DCR)

Created custom table `AppSecEvents_CL` using Azure Monitor Data Collection Endpoints (DCE) and Data Collection Rules (DCR).

- **Schema Definition:**
    - `TimeGenerated` (Datetime)
    - `SourceIp` (String)
    - `UserPrincipalName` (String)
    - `EventSeverity` (String)
    - `ActionTaken` (String)
- **Ingestion Payload:** Ingested 5 synthetic application-security records via HTTP POST to the Logs Ingestion API endpoint:

JSON

```
[
  {"TimeGenerated": "2026-10-08T21:30:00Z", "SourceIp": "192.168.10.45", "UserPrincipalName": "user01@contoso.com", "EventSeverity": "Medium", "ActionTaken": "Blocked"},
  {"TimeGenerated": "2026-10-08T21:31:12Z", "SourceIp": "10.0.4.12", "UserPrincipalName": "admin-dev@contoso.com", "EventSeverity": "Low", "ActionTaken": "Allowed"},
  {"TimeGenerated": "2026-10-08T21:32:05Z", "SourceIp": "198.51.100.87", "UserPrincipalName": "unknown", "EventSeverity": "High", "ActionTaken": "Flagged"},
  {"TimeGenerated": "2026-10-08T21:33:40Z", "SourceIp": "192.168.10.45", "UserPrincipalName": "user01@contoso.com", "EventSeverity": "Critical", "ActionTaken": "Isolated"},
  {"TimeGenerated": "2026-10-08T21:34:10Z", "SourceIp": "172.16.0.99", "UserPrincipalName": "svc-app@contoso.com", "EventSeverity": "Informational", "ActionTaken": "Logged"}
]
```

### 4. Table Plan Comparison & NCES Retention Architecture

#### Table Tier Comparison Matrix

| Feature / Metric | Analytics Tier | Basic / Auxiliary Tier | Total / Data Lake Archive |
| --- | --- | --- | --- |
| **Primary Use Case** | Real-time alerts, scheduled KQL rules, hunting | High-volume verbose logs (NetFlow, WAF raw) | Long-term compliance & audit preservation |
| **KQL Capabilities** | Full KQL support, join, summarize, time-series | Limited query capability (search-based, single table) | Search jobs / Async restore required |
| **Alerting Support** | Full scheduled & NRT Analytics Rules | No direct alert rules (must search/restore first) | No direct alert execution |
| **Cost Profile** | Standard ingestion + hot storage | Significant ingestion discount (~50-80% lower) | Ultra-low storage cost per GB/month |

#### NCES Data Retention Plan

```
[ Ingested Telemetry ]
       │
       ├──► Critical Security Tables (SecurityEvent, SigninLogs, AuditLogs, AppSecEvents_CL)
       │    └──► Analytics Tier: 90 Days Interactive Search
       │    └──► Archive Retention: Extended to 365 Days
       │
       ├──► Verbose Network & Application Logs (W3CIISLog, CommonSecurityLog)
       │    └──► Basic/Auxiliary Plan: 30 Days Basic Search
       │    └──► Total Retention: 13 Months (395 Days)
       │
       └──► Selected Compliance Audit Evidence (AzureActivity, KeyVaultVaultEvents)
            └──► Analytics Tier: 90 Days Interactive Search
            └──► Total Long-Term Archive: 7 Years (2,555 Days)
```

## Verification

| Test | Expected | Actual | Pass/fail | Evidence reference |
| --- | --- | --- | --- | --- |
| Allowed path | Ingest custom JSON array to Logs Ingestion API | HTTP `204 No Content` returned; records visible in Sentinel | Pass | HTTP 204 Success |
| Custom Query | Run KQL against `AppSecEvents_CL` | Exactly 5 synthetic records returned with expected schema | Pass | KQL Output Trace |
| Daily Cap Protection | Validate workspace daily volume quota enforcement | Quota set to `0.5 GB`; hard stop enforced if exceeded | Pass | Workspace Properties |
| Role Scope | `sec-analyst-tier1` attempts to modify Analytics Rule | Access denied with `Unauthorized` error | Pass | Azure RBAC Deny Test |
| Retention Policy | Apply 2,555-day total retention to `AzureActivity` | Workspace table properties reflect 90-day Interactive / 2,555-day Total | Pass | Table Lifecycle Config |

Code snippet

```
// Custom Table Verification Query
AppSecEvents_CL
| where TimeGenerated > ago(1h)
| project TimeGenerated, SourceIp, UserPrincipalName, EventSeverity, ActionTaken
| sort by TimeGenerated desc
```

## Findings

- **Misconfiguration discovered:** Default Log Analytics table settings applied 30-day interactive retention across all tables indiscriminately, risking audit non-compliance for critical evidence and overcharging for verbose debug logs.
- **Root cause:** Standard onboarding templates do not automatically differentiate retention periods based on table criticality or compliance mandates.
- **Remediation:** Configured table-level retention overrides using Azure CLI / PowerShell:
    - `SecurityEvent` / `AppSecEvents_CL`: Interactive 90 days.
    - `AzureActivity`: Interactive 90 days, Total 2,555 days (7 years).
    - `W3CIISLog`: Auxiliary plan, Total 395 days (13 months).
- **Residual risk:** Basic/Auxiliary tier tables cannot trigger real-time scheduled analytics rules; high-severity threats in verbose streams must be filtered at the ingestion pipeline/DCR stage.
- **Follow-up owner/date:** Lead Security Engineer / October 25, 2026.

## Teardown

- **Resources stopped/deleted:** Deleted DCR and Data Collection Endpoint bindings for custom log ingestion.
- **Defender plan/add-on disabled:** Microsoft Sentinel 31-day trial maintained in active state for Day 29/30 capstone execution.
- **Capacity/endpoint removed:** Logs Ingestion API app registration credentials disabled.
- **Cost view rechecked:** $0.20 AUD actual spend verified on Azure subscription.
- **Persistent dependency for next lab:** Onboarded workspace `law-sc500-cc-01` and `AppSecEvents_CL` table schema preserved for Day 29 rule development and hunting.

## Exam Recall

- **Three facts I can explain:**
    1. Setting table retention in Log Analytics involves two distinct parameters: **Interactive Retention** (Analytics tier KQL search) and **Total Retention** (Archived/Data Lake storage up to 7 years).
    2. The Basic and Auxiliary table plans significantly reduce ingestion costs for high-volume verbose logs, but they restrict KQL query capabilities and do not support direct scheduled Analytics Rules.
    3. Azure Monitor Data Collection Rules (DCR) and Data Collection Endpoints (DCE) are required to parse, transform, and stream custom JSON log payloads into `_CL` tables via the Logs Ingestion API.
- **One confusing distinction:**
    - **Interactive Retention vs. Total Retention:** Interactive Retention (default 30–90 days) keeps data immediately queryable using full KQL in the hot Analytics tier. Total Retention includes the archive period (up to 7 years), where data is retained at an ultra-low storage cost but requires asynchronous Search Jobs or Data Restoration before executing full KQL queries.
- **One scenario question I wrote:**
    - *A security operations team needs to retain web application firewall (WAF) logs for 13 months for compliance investigations. The team rarely queries these logs and wants to minimize ingestion and storage costs without dropping incoming telemetry. Which table configuration strategy should the security administrator apply?*
        - A) Keep WAF logs in the Analytics tier with 395 days of Interactive Retention.
        - B) **Set the WAF table plan to Basic/Auxiliary with 30 days Interactive Retention and 395 days Total Retention.** *(Correct)*
        - C) Onboard the workspace to Microsoft Sentinel Data Lake and delete the local table.
        - D) Configure an export rule to stream WAF logs to Azure Key Vault.