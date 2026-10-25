# Day 5 - Policy, compliance, backup security, and IaC

Status: Done

## Session

- Lab: **Policy, compliance, backup security, and IaC**
- Date/time and time zone: 05/10/26
- Start/end time: 12:46pm
- Hands-on, trial-dependent, or simulated: Hands-on
- Microsoft documentation checked on: [https://learn.microsoft.com/en-us/azure/governance/policy/tutorials/create-custom-policy-definition](https://learn.microsoft.com/en-us/azure/governance/policy/tutorials/create-custom-policy-definition)
- Region/SKU used:

## Cost checkpoint

- Credit remaining: $286
- Month-to-date actual: Nil
- Forecast: Nil
- Top services: Nil
- Unexpected meters: N/A
- Action: N/A

## Business control

- **Requirement:** NCES requires strong administrative governance, prevention of unapproved public IP exposures, immutable backup protection against ransomware, and verified IaC deployments prior to resource creation.
- **Threat or Failure Mode:** Unauthorized public endpoint exposure, accidental or malicious deletion of backup recovery points by compromised administrative credentials, and configuration drift during automated deployments.
- **Control Selected:** Azure Policy Initiatives (`NCES POC Security Baseline`), Custom Deny Policy with Tag Exceptions (`Deny-PublicIP-Without-ApprovedException`), Recovery Services Vault Immutability / Soft Delete / Resource Guard (MUA), Resource Locks (`CanNotDelete`), and Bicep `what-if` pre-deployment analysis.
- **Why This Service/Control Fits:** Enforces declarative governance at the ARM management plane, prevents non-compliant resources from being provisioned, secures backup data with multi-user authorization, and provides pre-flight visibility into IaC template changes.
- **Rejected Alternative and Reason:** Manual periodic compliance audits and post-deployment cleanup scripts. Manual audits allow exposure windows where non-compliant resources remain online, and scripts lack real-time ARM prevention capabilities.

## Implementation

- Resources changed:
    - Created a custom Azure Policy Initiaitve

![image.png](Day%205%20-%20Policy,%20compliance,%20backup%20security,%20and%20I/image.png)

**Resources Changed:** `NCES POC Security Baseline` initiative, `Deny-PublicIP-Without-ApprovedException` custom policy, resource group lock (`SecRG-DeleteLock`), and test Public IP resources.   **Identity and Role/Scope:** Policy assigned at subscription and resource group scopes (`sc500labs-8427-resource` and `sc500labs-1258-resource`). Managed Identity provisioned for DINE policy remediation tasks.   **Management-Plane Controls:** Custom policy requiring `ExceptionApproved=true` AND `expiry` tags for Public IP creation; `CanNotDelete` resource lock applied to security resource group.**Data-Plane Controls:** Secure transfer (HTTPS) enforced for Storage Accounts; Key Vault public access restricted; Recovery Services Vault Soft Delete and Vault Lock enabled.**Network Controls:** Public network access auditing enforced across Storage Accounts, Key Vaults, and Azure SQL Databases.**Monitoring Destination:** Azure Policy Compliance Dashboard and Defender for Cloud Regulatory Compliance view (Microsoft Cloud Security Benchmark).

## Verification

| Test | Expected | Actual | Pass/fail | Evidence reference |
| --- | --- | --- | --- | --- |
|  |  |  |  |  |
|  |  |  |  |  |
|  |  |  |  |  |
|  |  |  |  |  |

### Findings & Remediation

- **Misconfiguration Discovered:** Initiative creation initially failed due to unmapped policy parameters (`tagValue` parameter missing on built-in tag definitions).
    
    PNG+ 1
    
- **Root Cause:** Policy definitions requiring multiple parameters (`tagName` and `tagValue`) were included without explicitly passing values or mapping them to initiative-level parameters.
    
    PNG
    
- **Remediation:** Swapped the policy to `Require a tag on resources` (which only evaluates tag key existence) or mapped initiative parameters directly to all required parameter fields.
    
    PNG
    
- **Residual Risk:** An administrator with Owner/Contributor rights could temporarily remove the policy assignment or resource lock to force a non-compliant deployment.
- **Follow-up Owner/Date:** Cloud Governance Team / 2026-10-15

### Environment Teardown

- **Resources Stopped/Deleted:** Removed test Public IP resources (`pip-test-denied` and approved PIP) and deleted `SecRG-DeleteLock`.
- **Capacity/Endpoint Removed:** Cleaned up test policy assignments scoped strictly to `sc500labs-8427-resource`.
    
    PNG
    
- **Cost View Rechecked:** Subscription cost checked; empty Recovery Services Vault generated $0.00 base cost.
- **Persistent Dependency for Next Lab:** Custom initiative `NCES POC Security Baseline` and custom policy `Deny-PublicIP-Without-ApprovedException` retained in subscription definitions.

### Exam Recall (SC-500 Key Takeaways)

- **Three Facts I Can Explain:**
    1. Azure Policy evaluates requests at the **ARM management plane** before resource provisioning occurs; a `Deny` policy prevents non-compliant resources from ever being created.
    2. A **`CanNotDelete`** lock allows authorized users to read and modify resource configurations, but prevents anyone (including Subscription Owners) from deleting the resource until the lock is removed.
    3. **Resource Guard (Multi-User Authorization)** for Azure Backup requires critical operations (like turning off soft delete) to be approved by a secondary role in Entra ID before the change is processed.
- **One Confusing Distinction:** The difference between a policy with an **`Audit`** effect (flags non-compliant resources in dashboards without blocking deployment) vs. **`DeployIfNotExists` / `Modify`** effects (actively remediates or alters non-compliant resources during or after deployment).
    
    PNG
    
- **One Scenario Question I Wrote:** *An organization requires that no Storage Accounts or Key Vaults can be deleted, but cloud engineers must still be able to update firewall rules and access policies on these resources. Which control should be applied?*
    - **Answer:** Apply a `CanNotDelete` (ReadOnly = False) lock at the Resource Group or Subscription scope.