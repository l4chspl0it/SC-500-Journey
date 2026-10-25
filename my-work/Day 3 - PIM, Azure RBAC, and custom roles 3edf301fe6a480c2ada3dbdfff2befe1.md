# Day 3 - PIM, Azure RBAC, and custom roles

Status: Done

Session
Lab: PIM, Azure RBAC, and custom roles
Date/time and time zone: 03/10/26 01:07pm AEST
Start/end time: 1:00pm - 1:47pm AEST
Hands-on, trial-dependent, or simulated: Hands-on
Microsoft documentation checked on: N/A
Region/SKU used: West US 3 (Foundry/Hub) / Australia East

Cost checkpoint

Credit remaining: $286

Month-to-date actual: N/A

Forecast: N/A

Top services: Microsoft Foundry, Microsoft Entra ID Governance

Unexpected meters: N/A

Action: N/A

Business control

Requirement: The platform team needs to operate resources without standing subscription Owner. Security staff need time-bound approval for role assignments, while application developers only require deployment access in the app resource group.

Threat or failure mode: Over-privileged access, permanent standing administrative privileges, and unauthorized privilege escalation.

Control selected: PIM roles (Eligible assignments with activation controls and time-bound approvals)

Why this service/control fits: Ensures least privilege and Just-In-Time (JIT) access with mandatory justification, MFA, and approval workflows.

Rejected alternative and reason: Permanent active roles and standing assignments; these leave high-privilege credentials continuously exposed to lateral movement or compromise.

Implementation
Resources changed: Resource Group (`sc500labs-8427-resource`) updated with `DeleteAfter` tag; PIM role settings for `Contributor` modified.
Identity and role/scope:

- `platform.operator` assigned **Eligible Contributor** role on Resource Group `sc500labs-8427-resource`.
- `security.operator` designated as the **Approver** for Contributor activation requests.
    
    Management-plane controls: PIM activation policy set to max 1-hour duration, requiring MFA, business justification, and explicit approval by `security.operator`.
    
    Data-plane controls: N/A
    
    Network controls: N/A
    
    Monitoring destination: Entra Audit Logs and PIM Alert History.
    

Verification

| **Test** | **Expected** | **Actual** | **Pass/fail** | **Evidence reference** |
| --- | --- | --- | --- | --- |
| **Allowed path** | `platform.operator` requests `Contributor` activation on Resource Group with justification; `security.operator` approves; role activates for 1 hr max. | Role assignment became active for 1 hour after `security.operator` approved request. | Pass | `day1-evidence.txt` / PIM Activation History |
| **Denied path** | `platform.operator` attempts direct resource deployment without activating PIM role, or attempts activation past the 1-hour limit without approval. | Access denied via ARM authorization checks due to missing active RBAC permissions. | Pass | Azure Activity Log / ARM Authorization |
| **Logging/detection** | PIM activation requests, approvals, and expirations generate log entries in Entra ID Audit Logs. | Activation events logged with principal ID, approver ID, and justification string. | Pass | Entra ID Audit Logs |
| **Recovery/rollback** | Removing eligible assignment or letting 1-hour window elapse automatically revokes Contributor permissions. | Role automatically de-allocated after 1 hour without manual intervention. | Pass | PIM Assignment Overview |

Findings

Misconfiguration discovered: Initial default PIM policy allowed 8-hour maximum activation without requiring explicit approver signatures.

Root cause: Default Azure PIM role settings prioritize basic security defaults over strict governance requirements.

Remediation: Updated Contributor role settings in PIM for the resource group scope to mandate 1-hour max duration, MFA, justification, and approval by `security.operator`.

Residual risk: Approvers (`security.operator`) approving requests without verifying actual business change-tickets.

Follow-up owner/date: Security Admin / 2026-10-10

Teardown

Resources stopped/deleted: Temporary deployments/test agents cleared.

Defender plan/add-on disabled: N/A

Capacity/endpoint removed: N/A

Cost view rechecked: Confirmed remaining credit at $286.

Persistent dependency for next lab: Resource Group `sc500labs-8427-resource` and assigned tags (`DeleteAfter`).

Exam recall

Three facts I can explain:

1. Azure RBAC custom roles CAN be cloned/created from existing roles, whereas Entra ID custom roles CANNOT be cloned directly from built-in roles in the same manner.
2. **Entra Privileged Role Administrator** manages Identity & Directory Roles (Entra ID), whereas **Azure User Access Administrator** manages Resource Access Roles (Azure RBAC).
3. Entra ID and Azure RBAC operate on two separate control planes; identity roles do not grant ARM resource permissions directly, and ARM administrators cannot grant directory permissions.
    
    One confusing distinction: The difference between **Entra PIM for Directory Roles** (managed via Entra ID Governance) versus **Entra PIM for Azure Resources** (managed via ARM resource scope settings).
    
    One scenario question I wrote: *An organization wants to allow system engineers to manage virtual machines in a specific Resource Group without giving them permanent Contributor permissions. They also require a senior security analyst to sign off on every elevation request. Which configuration satisfies these requirements with least privilege?*
    
    **Answer:** Assign the engineers an **Eligible** Contributor role on the Resource Group in PIM, configure the role policy with a 1-hour max activation duration, and require approval from the security analyst's account.