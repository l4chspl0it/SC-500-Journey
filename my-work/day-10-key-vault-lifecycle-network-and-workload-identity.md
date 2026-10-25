# Day 10 - Key Vault lifecycle, network, and workload identity

Status: Done

## 

**Session**

• **Lab:** Azure Key Vault RBAC, Lifecycle Testing, Private Endpoint Isolation & Defender Staging
• **Date/time and time zone:** 07/10/2026 (AEDT)
• **Start/end time:** 14:15 - 15:00
• **Hands-on, trial-dependent, or simulated:** Hands-on
**Cost checkpoint**
• **Credit remaining:** $284
• **Month-to-date actual:** Nil
• **Forecast:** Nil
• **Top services:** Azure Key Vault (Standard tier), Private Endpoint, Managed Identity (Logic App Consumption)
• **Unexpected meters:** Nil
• **Action:** Retain Key Vault and Private Endpoint for upcoming security architecture labs; Logic App idle execution cost is $0.00.   

**Business control**

• **Requirement:** Centralize credential and certificate management using Azure RBAC, enforce strict identity-based access controls via Managed Identities, isolate network access via Private Endpoints, and disable public data plane access.   

• **Threat or failure mode:** Public exposure of Key Vault secrets, credential leakage via code repositories, unauthorized access using Vault Access Policies, or broad network traversal.   
• **Control selected:** Azure RBAC Authorization (`Key Vault Secrets User`, `Key Vault Secrets Officer`), System-Assigned Managed Identity, Private Endpoint (`privatelink.vaultcore.azure.net`), and Public Network Access lockdown (`publicNetworkAccess = Disabled`).   
• **Why this service/control fits:** Azure RBAC provides granular, scope-bound access controls that align with Entra ID privileged access models, while Private Endpoints route all secrets traffic over internal private IPs (`10.21.1.x`), preventing public internet exposure.   
• **Rejected alternative and reason:** Key Vault Access Policies. Access Policies operate at the vault level rather than object-specific scope and do not integrate directly with Azure RBAC management-plane roles, increasing management complexity.   

**Implementation**

• **Resources changed:**
    ◦ Key Vault: `kv-sc500-app` (Standard SKU, Soft Delete & Purge Protection enabled, Azure RBAC authorization model)   
    ◦ Synthetic Secret: `synthetic-db-password` (versioned secret configured with explicit expiration date)   
    ◦ Workload Identity: `logic-sc500-test` (Logic App Consumption with System-Assigned Managed Identity enabled)   
    ◦ Private Endpoint: `pe-kv-sc500` connected to `kv-sc500-app` targeting sub-resource `vault`
   
    ◦ DNS Zone Link: `privatelink.vaultcore.azure.net` linked to `vnet-sc500-app`
   
• **Identity and role/scope:** Workload assigned least-privilege `Key Vault Secrets User` role at the vault scope.   
• **Management-plane controls:** Azure RBAC enabled at vault creation; Vault Access Policies disabled.   
• **Data-plane controls:** Public network access disabled (`publicNetworkAccess = Disabled`); bypass allowed only for trusted Microsoft services.   
• **Network controls:** Isolated network traversal via Private Endpoint on `AppSubnet` in `vnet-sc500-app`.   
• **Monitoring destination:** Diagnostics `AuditEvent` logging staged for Log Analytics integration.   

**VerificationTestExpectedActualPass/failEvidence referenceRBAC Secret Retrieval**Managed Identity with `Key Vault Secrets User` retrieves secret metadata URI without exposing plaintextSuccessful HTTP 200 retrieval via Logic App Managed Identity**PASS**Logic App Key Vault Designer Execution / Run History   **Lifecycle Versioning**Rotating secret creates new Version ID GUID while preserving version historySecret updated; old version disabled and access failure validated**PASS**Key Vault Secrets Versions Blade   **Private DNS Resolution**VNet DNS queries for `kv-sc500-app.vault.azure.net` resolve to `privatelink.vaultcore.azure.net` private IPVirtual Network link verified (`vnet-sc500-app` attached to Private DNS zone)**PASS**Private DNS Zone Virtual Network Links Blade   **Public Network Access**Direct internet requests to Key Vault data plane blocked with HTTP 403 ForbiddenFirewall disabled public access; unauthorized public queries rejected**PASS**Key Vault Networking Firewall Rules   

**Findings**

• **Misconfiguration / Confusion discovered:**
    1. Standard Azure Cloud Shell executes outside user VNets; running `nslookup` inside standard Cloud Shell queries public DNS rather than VNet-linked Private DNS Zones (`privatelink.vaultcore.azure.net`).   
    2. The Azure Portal error *"The specified virtual network is already linked to this Private DNS zone"* indicates active VNet DNS integration, requiring in-VNet execution (e.g., test VM) to confirm private IP translation.   
• **Root cause:** Cloud Shell container networking routes queries through Microsoft's public management plane unless deployed in VNet-isolated mode.
• **Remediation:** Validated Private DNS Zone VNet Link status directly in Azure Portal.   
• **Residual risk:** Microsoft Defender for Key Vault remains in staged/off status until Day 20 per lab plan schedule.   
• **Follow-up owner/date:** Self / 07/10/2026.

**Teardown**

• **Resources stopped/deleted:** None required; retained lightweight testing workload and Key Vault infrastructure.
• **Defender plan/add-on disabled:** Defender for Key Vault validated as disabled/staged.   
• **Capacity/endpoint removed:** None.
• **Cost view rechecked:** $0.00 additional run costs.
• **Persistent dependency for next lab:** Retain `kv-sc500-app`, `pe-kv-sc500`, and `privatelink.vaultcore.azure.net` for upcoming Defender and Monitoring labs.   

**Exam recall**

• **Three facts I can explain:**

    1. **RBAC vs. Access Policies:** Azure Key Vault supports two permission models: Vault Access Policies (vault-wide) and Azure RBAC (supports fine-grained role assignments like `Key Vault Secrets User` at management or resource scope).   

    2. **Synthetic Secrets:** Synthetic secrets are generated or mock credentials used in automation workflows to prevent human exposure and facilitate rotation testing.   

    3. **Private Endpoint DNS Zone:** Key Vault Private Endpoints require integration with `privatelink.vaultcore.azure.net` to redirect public vault traffic to the private network interface.   

• **One confusing distinction:** **`Key Vault Reader` vs. `Key Vault Secrets User`** — `Key Vault Reader` grants permission to view Key Vault metadata (e.g., resource properties, tags) but CANNOT read secret contents. `Key Vault Secrets User` grants permission to read the actual data-plane secret values.   

• **One scenario question I wrote:** *An application running on an Azure Virtual Machine needs to retrieve a connection string from Azure Key Vault without storing any credentials in code. What is the recommended configuration?*
**Answer:** Enable a System-Assigned Managed Identity on the Virtual Machine, assign the `Key Vault Secrets User` RBAC role to the identity at the Key Vault scope, and configure the application to authenticate via `DefaultAzureCredential`.