# Day 4 - Conditional Access, application identity, and consent

Status: Done

## Session

- Lab: **Conditional Access, application identity, and consent**
- Date/time and time zone: 05/10/2026
- Start/end time: 11:58am
- Hands-on, trial-dependent, or simulated: Hands-on
- Microsoft documentation checked on:
- Region/SKU used: East US

## Cost checkpoint

- Credit remaining: $284
- Month-to-date actual: Nil
- Forecast: Nil
- Top services:
- Unexpected meters:
- Action:

## Business control

- Requirement: NCES wants strong controls for administrators and cannot allow users to grant high-risk OAuth permissions. Pathfinder needs an application identity, but its developers must not create a long-lived client secret.
- Threat or failure mode: Privileged access via legacy auth, single auth and requireM MFA for medium or high risk sign in’s
- Control selected: Conditional Access policies and Admin Consent Workflow, and Federated Credentials / Managed Identity.
- Why this service/control fits: Provides granular access controls and conditional access if the specified conditions are met and can enforce MFA/block access based of UEBA patterns and prevents users from consenting to high-risk OAuth permissions.
- Rejected alternative and reason: Per-user MFA and unrestricted user consent settings. Per-user MFA lacks risk-based real-time evaluation, and unrestricted consent allows non-admin users to grant high-impact permissions to malicious applications.

## Implementation

From the exam, here is a list of ‘Conditions; and also the ‘Grant’ CA actions

Conditions:

![image.png](Day%204%20-%20Conditional%20Access,%20application%20identity,%20/image.png)

Grant actions based off the conditions being met:

![image.png](Day%204%20-%20Conditional%20Access,%20application%20identity,%20/image%201.png)

And lastly session actions

![image.png](Day%204%20-%20Conditional%20Access,%20application%20identity,%20/image%202.png)

Something to check: when setting the session options - which was required to deploy the policy - selecting some of these did not allow me to deploy the policy in report-only mode. The disable resilient defaults option did allow me to deploy in report-only but research why this was the case.

![image.png](Day%204%20-%20Conditional%20Access,%20application%20identity,%20/image%203.png)

Legacy auth client location in CAP

![image.png](Day%204%20-%20Conditional%20Access,%20application%20identity,%20/image%204.png)

- **Resources changed:** Registered App (`Pathfinder-Case-API`), Enterprise Application service principal, and active Conditional Access policies.
- **Identity and role/scope:** Created delegated scope `Case.Read` and app role `Case.Process`; designated `security.operator` as Admin Consent reviewer.
- **Management-plane controls:** Configured CA policies in Report-only mode; blocked user consent for high-impact permissions; configured Admin Consent request workflow.
- **Data-plane controls:** Prohibited client secrets; designated Federated Identity Credentials / Managed Identity for production authentication.
- **Network controls:** Blocked legacy authentication protocol agents (ActiveSync, basic auth) via CA conditions.
- **Monitoring destination:** Entra ID Audit Logs, Provisioning Logs, and Sign-in Logs (Report-only view).

## Verification

| Test | Expected | Actual | Pass/fail | Evidence reference |
| --- | --- | --- | --- | --- |
| Allowed path | Standard user or admin signs in via modern auth; triggers MFA when risk is Medium/High. | Sign-in succeeds after satisfying risk-based MFA prompt. | Pass | Entra Sign-in Logs |
| Denied path | Legacy auth client attempts sign-in, or non-admin attempts to grant tenant-wide consent. | Legacy client blocked by CA-All-BlockLegacyAuth; non-admin user redirected to Admin Consent workflow. | Pass | Conditional Access What-If / Audit Logs |
| Logging/detection | App registration, consent requests, consent grants, and revocations logged in Entra Audit Logs. | All lifecycle actions recorded with initiating principal ID and operation name. | Pass | Entra ID Audit Logs |
| Recovery/rollback | Admin revokes delegated consent grant from Enterprise Applications; permissions immediately cease. | Revoked OAuth grant removed from permissions list; API access rejected. | Pass | Enterprise Apps > Permissions |

**Findings**

- **Misconfiguration discovered:** Default tenant settings permitted end-users to grant consent to unverified third-party OAuth applications without admin intervention.
- **Root cause:** Default Microsoft Entra ID tenant configurations prioritize user self-service over strict application governance.
- **Remediation:** Disabled user consent for unverified/high-impact apps and configured an explicit Admin Consent Workflow reviewed by `security.operator`.
- **Residual risk:** Admin approving an unsafe consent request submitted through the workflow without proper vetting.
- **Follow-up owner/date:** Identity Security Team / 2026-10-12

**Teardown**

- **Resources stopped/deleted:** Revoked test admin consent grant for `Pathfinder-Case-API`.
- **Defender plan/add-on disabled:** N/A
- **Capacity/endpoint removed:** N/A
- **Cost view rechecked:** Verified remaining subscription credits ($286).
- **Persistent dependency for next lab:** Registered app `Pathfinder-Case-API` and CA policies (`CA-All-BlockLegacyAuth`, `CA-Lab-RiskySignIn`) in Report-only mode.

## Exam recall

- **Three facts I can explain:**
    1. Registering an application automatically provisions a Service Principal in Enterprise Applications to act as the local identity instance in the tenant.
    2. Declaring a scope in App Registrations defines an permission offer, but an entry only appears in Enterprise Applications once an actual consent grant is issued.
    3. Legacy authentication protocols cannot enforce MFA prompts, requiring explicit blocking via Conditional Access client app conditions.
- **One confusing distinction:** The difference between the **Application Object** (the global template created in App Registrations) and the **Service Principal** (the local active identity created in Enterprise Applications).
- **One scenario question I wrote:** *An organization needs to allow developers to publish internal APIs, but must prevent users from consenting to unauthorized scopes. Developers must authenticating applications without managing secrets. Which combination of controls should be implemented?*
    - **Answer:** Configure the Admin Consent Workflow with designated reviewers, restrict user consent settings, and mandate Federated Identity Credentials (OIDC) or Managed Identities instead of client secrets.

### 1. Explanation of Concepts

#### Why a Service Principal was Automatically Created in Enterprise Applications

When you perform an **App Registration**, you are creating the **Application Object** (the blueprint/definition of your app across all tenants).

In a single-tenant scenario, Entra ID automatically provisions a corresponding **Service Principal** (the concrete instance/identity of the app within your specific local tenant) in **Enterprise Applications**. The Application Object defines *what* the application is, while the Service Principal defines *how* the application behaves, who can access it, and what permissions it holds inside your specific Entra directory.

#### Why You Couldn't Initially See or Revoke the Permissions

You couldn't see any permissions under **Enterprise Applications > Permissions** because **defining** permissions is different from **granting** them.

1. **Defining Scopes (`Case.Read`):** In App Registrations, you declared what permissions your API *offers*. Declaring a scope does not create an active OAuth consent grant.
2. **Granting Consent:** The **Permissions** blade inside Enterprise Applications only displays active **OAuth2PermissionGrants** (actual tokens/consents issued to users or admins).
3. **The Fix:** Until you clicked **Grant admin consent for Default Directory**, no active permission grant existed in the directory for the system to list or revoke. Once consented, the record was created, populated in the list, and became available to revoke.