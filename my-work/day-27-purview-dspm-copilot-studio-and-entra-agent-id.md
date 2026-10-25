# Day 27 - Purview DSPM, Copilot Studio, and Entra Agent ID

Status: Done

## Session

- **Lab:** Day 27 — Purview DSPM, Copilot Studio, and Entra Agent ID
- **Date/time and time zone:** October 8, 2026, 9:03 PM AEDT
- **Start/end time:** 21:03 - 22:45 AEDT
- **Hands-on, trial-dependent, or simulated:** Combined Hands-on & Guided Architectural Simulation
- **Microsoft documentation checked on:** October 8, 2026
- **Region/SKU used:** East US 2 / Microsoft 365 E5 Trial / Entra P2 / Microsoft Agent 365 Preview / Azure AI Foundry `case-agent-purple`

## Cost Checkpoint

- **Credit remaining:** $283.80 AUD (Azure Subscription)
- **Month-to-date actual:** $0.20 AUD
- **Forecast:** ~$0.20 AUD (All Station 1–3 operations performed under active Microsoft 365 E5 / Entra P2 / Agent 365 trial entitlements)
- **Top services:** Microsoft Purview, Microsoft Entra Agent ID, Microsoft 365 Admin Center (Agent Registry), Microsoft Defender XDR
- **Unexpected meters:** None
- **Action:** Verified zero billable impact on Azure subscriptions by relying strictly on tenant trial entitlements.

## Business Control

- **Requirement:** Establish data security posture management (DSPM) for AI interactions, secure autonomous and delegated AI agents using Microsoft Entra Agent ID, control the agent blast radius via Conditional Access, and enforce central governance and real-time agent protection in Defender XDR.
- **Threat or failure mode:** Sensitive case files overshared via SharePoint exposed to Copilot/agents, unmanaged or ownerless "shadow" agents operating with excessive permissions, credential theft via static client secrets on agent identities, and malicious tool invocations during agent execution.
- **Control Selected:** Purview DSPM for AI + Microsoft Entra Agent ID Blueprints (Federated Credentials) + Agent 365 Registry Governance + Defender XDR Real-Time Protection (`AgentsInfo` / `BehaviorInfo`).
- **Why this service/control fits:** Addresses data security, identity boundary, and real-time execution threats across both human-to-AI and agent-to-resource communication pathways.
- **Rejected alternative and reason:** Legacy Entra Agent Registry/Collections blades (retired in 2026). Rejected because central agent registry and lifecycle operations have migrated exclusively to Microsoft 365 Admin Center (`Agents > Registry`), while Entra focuses on Agent Identities and Conditional Access.

## Entitlement & Licensing Checkpoint

| Feature / Component | Active Licensing / Trial Status | Execution Mode |
| --- | --- | --- |
| **Azure Subscription Credit** | Active ($283.80 AUD remaining) | Hands-on Live |
| **Microsoft 365 E5 / Purview Suite** | Active 30-Day Enterprise Trial | Hands-on Live |
| **Microsoft Entra ID P2** | Active 30-Day Developer Trial | Hands-on Live |
| **Microsoft Entra Agent ID** | Active Preview Entitlement | Hands-on Live |
| **Microsoft Agent 365 / Real-time Protection** | Active Preview Entitlement | Hands-on Live |
| **Copilot Studio Standalone** | Simulated (Tenant license gap documented) | Architectural Simulation |

## Implementation

### Station 1 — SharePoint and Purview DSPM

- **SharePoint Setup:** Created lab site `[https://contoso.sharepoint.com/sites/NCES-CaseManagement](https://contoso.sharepoint.com/sites/NCES-CaseManagement)` containing synthetic file `Case_Summary_Alpha.docx` and intentionally overshared file `NCES_Client_PII_Export.xlsx` (shared with "Everyone except external users").
- **Purview Audit:** Verified unified audit logging (`Set-AdminAuditLogConfig -UnifiedAuditLogIngestionEnabled $true`).
- **DSPM for AI Review:** Navigated to **Microsoft Purview portal > Solutions > DSPM**. Reviewed AI Observability metrics, identifying `NCES_Client_PII_Export.xlsx` as a high-risk overshared sensitive file accessible to Microsoft 365 Copilot and agent indexing.
- **DLP Policy Creation:** Configured an audit-mode Data Loss Prevention (DLP) policy targeting fictional Tax File Numbers (TFN) and Client IDs when interacted with by generative AI applications and agents.
- **Oversharing Remediation:** Revoked the "Everyone except external users" sharing link on `NCES_Client_PII_Export.xlsx`, restricting access to the `NCES Case Managers` group. Observed data posture risk score reduction within the DSPM dashboard.

### Station 2 — Agent Identity Governance & Blast Radius (Entra Agent ID)

- **Role Separation:**
    - Assigned `Agent ID Developer` to `admin-dev@contoso.com`.
    - Assigned `Agent ID Administrator` to `admin-sec@contoso.com`.
- **Blueprint Configuration:** Created Agent Identity Blueprint `Pathfinder Case Summary` (`bp-pathfinder-001`):
    - *Technical Owner:* `admin-dev@contoso.com`
    - *Business Sponsor:* `sponsor-biz@contoso.com`
    - *Audience / Description:* Internal case processing agent for NCES field summary automation.
- **Identity Provisioning & Comparison:** Provisioned two agent identities from the blueprint: `pathfinder-agent-prod` and `pathfinder-agent-test`.
    - *Blueprint vs Identity:* The blueprint defines required Graph permissions (`Sites.Read.All`, `User.Read`) and federated trust configurations; the agent identity receives the tenant-local object ID (`agent-obj-9981`) and token issuance capability.
    - *Authentication:* Configured Workload Identity Federation (Federated Identity Credentials - FIC) linked to the agent's deployment subject; client secrets were strictly prohibited.
    - *Lifecycle Action:* Disabled `pathfinder-agent-test` via `Set-EntraAgentIdentity -Id <id> -AccountEnabled $false`.
- **Conditional Access for Agents:** Created a **report-only** Conditional Access policy targeting Agent Identities:
    - *Assignments:* Target = All Agent Identities (`bp-pathfinder-001`).
    - *Target Resources:* Cloud Apps -> Microsoft Graph API.
    - *Access Controls:* Block access if Sign-in Risk is High OR if request originates outside trusted Named Networks.
- **Defender XDR Blast Radius Analysis:** Queried `AgentsInfo` and `BehaviorInfo` in Advanced Hunting to trace identity permissions and resource interactions.

Code snippet

```
// Blast Radius Investigation: Identify Agent Identity Permissions and Resource Scope
AgentsInfo
| where AgentName startswith "pathfinder"
| project TimeGenerated, AgentId, AgentName, BlueprintId, SponsorUserPrincipalName, PermissionsGranted, AccountStatus
| join kind=leftouter (
    BehaviorInfo
    | where ActionType startswith "Agent"
    | project TimeGenerated, AgentId, ActionType, ResourceTitle, DestinationUrl, RiskLevel
) on AgentId
```

### Station 3 — Agent Administration & Real-Time Protection

- **Agent Registry Governance:** Navigated to **Microsoft 365 admin center > Agents > Registry**.
    - Exported CSV inventory of tenant agents.
    - Identified ownerless synthetic agent `Shadow-Helper-Bot` possessing unmanaged Graph permissions.
    - Assigned business sponsor (`sponsor-biz@contoso.com`) and blocked execution of `Shadow-Helper-Bot` via the M365 Agent Registry governance blade.
- **Real-Time Agent Protection:** In **Defender XDR > Security for AI > Agent Protection**, configured an audit-first policy for the Copilot Studio agent `NCES-Assist`.
    - *Policy Trigger:* Detect unauthorized tool calls (e.g., attempt to invoke external WebHook).
    - *Staging:* Set action to **Audit** to record telemetry in `BehaviorInfo` before transitioning policy to **Block**.
- **Dashboard Comparison:**
    - *Tenant-wide Security Dashboard for AI:* Executive leadership view providing cross-product AI risk summary (Copilot, Copilot Studio, Foundry apps, shadow AI discovery, board-ready compliance metrics).
    - *Defender for Cloud Data and AI Security Dashboard:* Engineering/SOC view focusing on Azure infrastructure resources, cloud security posture (AI SPM), agentless vulnerability checks, and cloud attack path graphs.

## Verification

| Test | Expected | Actual | Pass/fail | Evidence reference |
| --- | --- | --- | --- | --- |
| Allowed path | Authorized agent identity (`pathfinder-agent-prod`) requests Graph token via Federated Credentials | Token issued successfully; event logged in Entra Sign-ins | Pass | Entra Agent ID Log |
| Denied path | Disabled identity (`pathfinder-agent-test`) attempts token acquisition | Token request denied with `AADSTS50057` (Account Disabled) | Pass | AADSTS50057 Error |
| Oversharing Remediation | Overshared file link revoked on SharePoint site | DSPM risk alert cleared; file removed from Copilot indexing scope | Pass | Purview DSPM Observability |
| Real-time Audit | Simulated unmanaged tool call by `NCES-Assist` agent | Behavior recorded in `BehaviorInfo` table with `Audit` action flag | Pass | Advanced Hunting KQL |
| Shadow Agent Block | Unmanaged `Shadow-Helper-Bot` execution attempt | Execution blocked at M365 Registry gateway | Pass | M365 Agent Registry |

## Findings

- **Misconfiguration discovered:** `Shadow-Helper-Bot` was created without a registered human sponsor or owner, granting it orphaned access to sensitive SharePoint libraries.
- **Root cause:** Pre-2026 provisioning workflows allowed app registrations to act as agents without enforcing blueprint sponsor metadata during creation.
- **Remediation:** Enforced M365 Agent Registry approval policies requiring a assigned Business Sponsor and Technical Owner before an agent identity can receive tokens.
- **Residual risk:** Report-only Conditional Access policies for agents do not actively enforce blocks until shifted to Enabled mode following baseline observation.
- **Follow-up owner/date:** Identity Governance Lead / October 20, 2026.

## Teardown

- **Resources stopped/deleted:** Disabled synthetic test identity `pathfinder-agent-test`. Re-secured overshared SharePoint library.
- **Defender plan/add-on disabled:** Maintained Agent 365 real-time protection in audit mode.
- **Capacity/endpoint removed:** Removed test DLP policy bindings.
- **Cost view rechecked:** $0.20 AUD actual spend verified on Azure subscription dashboard.
- **Persistent dependency for next lab:** Entra Agent ID Blueprint `bp-pathfinder-001` and M365 Agent Registry configurations retained for capstone consolidation.

## Exam Recall

- **Three facts I can explain:**
    1. Entra Agent Identities do not hold native static credentials; they authenticate strictly using Federated Identity Credentials (FIC) inherited from an Agent Identity Blueprint.
    2. Legacy Entra Agent Registry/Collections blades were retired in 2026; central agent lifecycle management is now handled in Microsoft 365 Admin Center (`Agents > Registry`), while Entra manages Agent Identities and Conditional Access.
    3. Advanced Hunting in Defender XDR records real-time agent protection events and block actions in the `BehaviorInfo` table, which correlates with `AgentsInfo` for blast radius analysis.
- **One confusing distinction:**
    - **Tenant-wide Security Dashboard for AI vs. Defender for Cloud Data and AI Dashboard:** The Security Dashboard for AI provides executive/cross-product governance across Microsoft 365 Copilot, Agent 365, and SaaS AI applications, whereas the Defender for Cloud Data and AI Dashboard focuses on technical cloud infrastructure (Azure AI Foundry, AML workspaces, storage accounts, and AI SPM attack path graphs).
- **One scenario question I wrote:**
    - *An organization needs to enforce risk-based access policies on autonomous AI agents calling Microsoft Graph APIs, ensuring agents cannot authenticate from unapproved IP ranges. Which combination of features and tables should the administrator use to configure access control and audit the resulting block events in Defender XDR?*
        - A) Entra App Registrations, Client Secrets, and `AIAgentsInfo` table
        - B) **Entra Agent ID Conditional Access Policy and `BehaviorInfo` table** *(Correct)*
        - C) Azure Key Vault Access Policies and `ApiManagementGatewayLogs` table
        - D) Purview Information Protection Labels and `CloudAppEvents` table