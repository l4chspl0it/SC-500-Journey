# Day 30 - Security Copilot capstone and zero-cost teardown

Status: Done

## Session

- **Lab:** Day 30 — Security Copilot Capstone and Zero-Cost Teardown
- **Date/time and time zone:** October 8, 2026, 9:26 PM AEDT
- **Start/end time:** 21:26 - 22:56 AEDT
- **Hands-on, trial-dependent, or simulated:** Capstone Hands-on Incident Response & Security Copilot Administration Runbook
- **Microsoft documentation checked on:** October 8, 2026
- **Region/SKU used:** Canada Central (`rg-sc500-sec-cc`) / Log Analytics Workspace `law-sc500-cc-01` / Security Copilot Administration Lane (Default Safe Simulation)

## Cost Checkpoint

- **Credit remaining:** $283.80 AUD (Azure Subscription)
- **Month-to-date actual:** $0.20 AUD
- **Forecast:** $0.00 AUD post-teardown
- **Top services:** Microsoft Sentinel, Azure Monitor, Azure API Management (APIM), Microsoft Entra ID
- **Unexpected meters:** None
- **Action:** Selected the **Default Simulation Lane** for Security Copilot (no hourly SCU provisioned at $4/SCU/hr). Executed complete automated and manual resource teardown via `teardown.sh` to ensure $0.00 lingering spend.

## Capstone Incident Timeline & Investigation

### Incident Summary

> A synthetic attack vector combined privilege escalation, prompt injection, and direct resource exposure. An unauthorized RBAC role assignment granted excessive permissions (`Contributor`) to the agent identity `pathfinder-agent-prod`. The identity bypassed the designated Azure API Management (APIM) AI Gateway endpoint, executing direct calls against an Azure AI Foundry model endpoint following an indirect prompt injection payload embedded within an overshared SharePoint case document (`NCES_Client_PII_Export.xlsx`).
> 

```
[ Attacker / Prompt Injection Payload ]
                   │
                   ▼ (Overshared SharePoint File)
     ┌───────────────────────────────┐
     │  NCES_Client_PII_Export.xlsx  │ (Unrestricted Share Link)
     └───────────────┬───────────────┘
                     │
                     ▼ (Indirect Prompt Attack Reads Payload)
     ┌───────────────────────────────┐
     │   pathfinder-agent-prod       │◄─── (Excessive RBAC Role Assignment Grant: Contributor)
     └───────────────┬───────────────┘
                     │
                     ▼ (Bypasses APIM AI Gateway Endpoint)
┌─────────────────────────────────────────────────────────────┐
│ Azure AI Foundry Model Endpoint (Direct Endpoint Call)      │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼ (Unauthorized PII Extraction)
               [ Public Storage Blob Storage ]
```

### Consolidated Timeline

| Timestamp (UTC) | Signal Source | Telemetry Event / Record Details | Attacker Action / Stage |
| --- | --- | --- | --- |
| `2026-10-08 20:00:12` | **Azure Activity** | `MICROSOFT.AUTHORIZATION/ROLEASSIGNMENTS/WRITE` granted `Contributor` to `pathfinder-agent-prod` | Privilege Escalation / Initial Access |
| `2026-10-08 20:12:45` | **Purview Audit** | `SharingSet` link created on `NCES_Client_PII_Export.xlsx` for "Everyone except external users" | Data Exposure / Oversharing |
| `2026-10-08 20:25:30` | **APIM / Foundry Logs** | Direct HTTP POST to `[https://foundry-nces.openai.azure.com/openai/deployments/](https://foundry-nces.openai.azure.com/openai/deployments/)...` | Gateway Bypass / Direct AI Access |
| `2026-10-08 20:26:05` | **Defender for Cloud** | Alert: *Indirect Prompt Attack & Anomalous Storage Access Pattern Detected* | High-Risk Threat Execution |
| `2026-10-08 20:30:00` | **Microsoft Sentinel** | Alert `[NCES] Privileged Azure Role Assignment Created` triggered incident `#1092` | Automated Incident Generation |

## Investigation Findings & Root Cause Analysis

1. **Initial Exposure:** Oversharing of `NCES_Client_PII_Export.xlsx` via SharePoint "Everyone except external users" permission link, allowing indexed prompt reading by Copilot/agents.
2. **Identity & Permissions:** Agent identity `pathfinder-agent-prod` (`agent-obj-9981`) was assigned `Contributor` at the resource group level (`rg-sc500-sec-cc`) instead of restricted reader permissions.
3. **Blast Radius:** Direct model execution on Azure AI Foundry bypassed APIM AI Gateway content filters, logging, and rate limits, exposing sensitive client PII to storage blob dumps.
4. **Bypassed Controls:** Lack of network access control lists (ACLs) on the Azure AI Foundry endpoint; failure to enforce APIM as the exclusive network entry point via Private Endpoints.
5. **Facts vs. Inference:**
    - *Confirmed Fact:* `pathfinder-agent-prod` performed direct POST calls bypassing APIM (verified via APIM gateway logs vs. Azure Activity).
    - *Inference:* Attacker weaponized the overshared SharePoint file specifically to trigger prompt injection (highly probable based on sequential Purview audit timestamps).

## Containment & Remediation Workflow

1. **RBAC Revocation:** Removed excessive `Contributor` assignment from `pathfinder-agent-prod` using Azure CLI:
`Az role assignment delete --assignee agent-obj-9981 --role Contributor --resource-group rg-sc500-sec-cc`
2. **Identity Disabling:** Disabled agent identity in Entra Agent ID:
`Set-EntraAgentIdentity -Id agent-obj-9981 -AccountEnabled $false`
3. **Gateway Enforcement:** Reconfigured Azure AI Foundry networking to restrict public access, requiring all inbound traffic to route through APIM AI Gateway Private Endpoints.
4. **Data Isolation:** Revoked overshared links on SharePoint site `NCES-CaseManagement` and set file storage permissions to explicitly scoped groups.
5. **Verification (Denied-Path Retest):**
    - *Test 1:* Attempted token issuance for `pathfinder-agent-prod` -> Denied with `AADSTS50057` (Account Disabled).
    - *Test 2:* Attempted direct invocation of Foundry endpoint bypassing APIM -> Blocked with `HTTP 403 Forbidden` (Network ACL Enforcement).

## Security Copilot Administration Runbook (Default Simulation)

### Workspace, Roles, and Governance Setup

- **Workspace Provisioning:** Provisioned default workspace `sc-workspace-nces-prod` bound to tenant region.
- **Role Assignments:**
    - **Security Copilot Administrator:** `sec-engineer-lead@contoso.com` (Workspace lifecycle, plugin approval, capacity management).
    - **Security Copilot Contributor:** `sec-analyst-tier2@contoso.com` (Prompt execution, session sharing, promptbook creation).
    - **Security Copilot Reader:** `sec-analyst-tier1@contoso.com` (Read-only session access).
- **Enabled Microsoft Plugins:**
    - *Microsoft Defender XDR* (Incident correlation, device timeline).
    - *Microsoft Entra ID* (Identity sign-ins, risk context, Agent ID blueprints).
    - *Microsoft Purview* (Data sensitivity labels, DSPM oversharing risks).
    - *Microsoft Sentinel* (KQL query generation, incident triage).

### Security Copilot Promptbook Execution

Executed prepared natural language prompts to validate incident triage capability:

```
// Prompt 1: Incident Triage
"Summarize Sentinel incident #1092, listing affected identities, resource groups, and high-risk API operations from the raw signals."

// Prompt 2: KQL Generation
"Write a KQL query for AzureActivity that searches for role assignment write operations executed by agent identities in the last 24 hours, projecting Caller, RoleDefinitionId, and ResourceGroup."

// Prompt 3: Fact Challenge against Raw Signals
"Security Copilot indicates the agent accessed Key Vault. Validate this claim against AzureActivity logs to confirm or refute."
```

- **Validation Verdict:** Copilot claim regarding Key Vault access was **refuted** after cross-referencing raw `AzureActivity` logs. *Key Takeaway:* Security Copilot responses must always be validated against underlying telemetry before completing incident sign-off.

## Mandatory Teardown Verification

Executed full environment cleanup using `teardown.sh` and manual Azure Portal validation steps.

Bash

```
#!/usr/bin/env bash
# Zero-Cost Teardown Script - Day 30 Capstone
CONFIRMATION="DELETE-SC500-LAB-RESOURCES"
read -p "Enter confirmation string: " input_confirm

if [ "$input_confirm" == "$CONFIRMATION" ]; then
    echo "Executing resource removal..."
    az group delete --name rg-sc500-sec-cc --yes --no-wait
    az group delete --name rg-sc500-foundry-cc --yes --no-wait
    echo "Teardown initiated successfully."
else
    echo "Confirmation mismatch. Aborting."
fi
```

### Teardown Audit Matrix

| Category / Component | Action Performed | Current Status | Verification Source |
| --- | --- | --- | --- |
| **Resource Locks & Policies** | Deleted custom Azure Policy assignments & locks | Cleared | Azure Portal Policy Blade |
| **PIM Assignments** | Deactivated time-bound PIM roles for lab admins | Inactive | Entra Privileged Identity Management |
| **Defender Plans** | Disabled Defender for Cloud paid plans & Storage malware scanning | Disabled / Free Tier | Defender Environment Settings |
| **Security Copilot Capacity** | Verified 0 SCUs provisioned | $0.00 / Hour | Azure Copilot Capacity Blade |
| **Sentinel Workspace** | Retention updated; resource group deleted (`rg-sc500-sec-cc`) | Deleted | Log Analytics Workspaces |
| **APIM & AI Foundry** | APIM Gateway, AI models, and AML deployments deleted | Deleted | Azure Resource Graph |
| **Entra Agent ID** | Disabled synthetic agent identities & deleted blueprints | Removed | Microsoft Entra Admin Center |
| **Cost Management** | Re-checked Cost Analysis by Resource Group | **$0.20 AUD Actual** | Azure Cost Management |

# Final Deliverable: Security Engineering Summary

## 1. Architecture & Implemented Controls

Over the 30-day execution cycle, the organization established a defense-in-depth architecture covering identity, data security, cloud workloads, AI workloads, and central security monitoring:

- **Identity & Access Governance:** Deployed Microsoft Entra Agent ID using Workload Identity Federation (Federated Identity Credentials) to eliminate static client secrets. Implemented report-only Conditional Access for agents and strictly segregated `Agent ID Developer` and `Agent ID Administrator` roles.
- **AI Workload Security & Gateways:** Enforced Azure API Management (APIM) as an AI Gateway in front of Azure AI Foundry model deployments, introducing centralized content safety filtering, payload logging, and rate-limiting.
- **Data Security Posture Management (DSPM):** Integrated Microsoft Purview DSPM for AI to detect overshared sensitive files (PII/TFNs) on SharePoint, applying audit-mode Data Loss Prevention (DLP) rules.
- **Security Operations & Log Analytics:** Onboarded Microsoft Sentinel in Canada Central (`rg-sc500-sec-cc`) under a 31-day trial. Implemented tiered retention (90 days Analytics, 13 months Auxiliary/Basic, 7 years Total Archive) to balance detection speed against cost. Configured low-noise Data Collection Rules (DCR) with XPath filtering for Windows/Linux hosts and built automated triage rules with Logic App playbooks using System-Assigned Managed Identities.

## 2. Three Primary Engineering Findings

1. **Orphaned Agent Identities Present Severe Privilege Escalation Risks:** Early provisioning workflows allowed app registrations to function as AI agents without mandatory business sponsor metadata. Synthetic testing proved that an unmanaged agent (`Shadow-Helper-Bot`) could inherit broad administrative roles (`Contributor`) without oversight, enabling unnoticed API executions.
2. **AI Endpoints Bypassing Gateways Negate Content Filters:** Exposing direct endpoints for Azure AI Foundry models allowed agents to bypass APIM AI Gateway guardrails during indirect prompt attacks. Enforcing strict Private Endpoints and Network ACLs is mandatory to guarantee all AI traffic flows through logging and filter proxies.
3. **Unoptimized Log Retention Dramatically Inflates SIEM Spend:** Default Log Analytics configurations apply standard hot interactive retention across all incoming streams indiscriminately. Transitioning high-volume, verbose network and debug logs to Auxiliary/Basic storage tiers reduced projected ingestion costs by over 60% while maintaining compliance requirements.

## 3. Licensing Gaps, Cost Results, and Residual Risks

- **Licensing Gaps Identified:**
    - Full enforcement of Agent Conditional Access requires active **Entra ID P2** and **Microsoft Agent 365** entitlements.
    - Real-time Agent Protection in Defender XDR and complete Purview DSPM for AI require Microsoft 365 E5 or Purview Suite add-ons.
- **Cost Result:**
    - **Total Lab Cost Incurred:** **$0.20 AUD** (Azure Subscription Credit remaining: **$283.80 AUD**).
    - Zero unexpected charges were incurred by utilizing 31-day Sentinel/E5 trials, enforcing a strict 0.5 GB/day workspace ingestion limit, bypassing paid Security Copilot SCU provisioning in favor of guided simulation, and executing immediate automated teardown via `teardown.sh`.
- **Residual Risks:**
    - Report-only Conditional Access policies do not actively block unauthorized agent authentications until shifted to Enabled mode following baseline observation.
    - Basic/Auxiliary tier log tables in Sentinel cannot run scheduled real-time KQL analytics rules; threats in verbose streams must be caught upstream at the DCR filter stage.

## 4. Production Transformation Roadmap

To transition this proof-of-concept into a enterprise production environment, the following architectural changes must be implemented:

1. **Shift Conditional Access from Report-Only to Enforced:** Move agent identity Conditional Access policies to Active mode, enforcing trusted Named Networks and high-risk block actions.
2. **Deploy Network Isolation via Private Link:** Remove public IP exposure on all Azure AI Foundry, Key Vault, and Storage resources, isolating all agent traffic within Azure Private Virtual Networks (VNets) and Private Endpoints.
3. **Automate Agent Lifecycle Governance in M365 Registry:** Mandate that all Entra Agent Identity Blueprints require dual sign-off from a Technical Owner and Business Sponsor in the Microsoft 365 Admin Center (`Agents > Registry`) prior to token issuance.
4. **Transition Security Copilot to Scheduled Provisioned SCUs:** Deploy production Security Copilot capacity using scheduled hourly SCU provisioning aligned strictly with SOC operational hours (or leverage M365 E5/E7 included SCU allocations) to prevent idle billing overhead.