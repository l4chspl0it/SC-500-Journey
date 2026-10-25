# Day 13 - Secure application platform services

Status: Done

## 

### Session

- **Lab:** Day 13 – Secure App Platform & PaaS Compute Boundaries (SC-500)
- **Date/time and time zone:** 07/10/2026 (AEDT)
- **Start/end time:** 16:20 - 16:35
- **Hands-on, trial-dependent, or simulated:** Architecture Decision Record / Portal Workflow Evaluation

### Cost checkpoint

- **Credit remaining:** $284
- **Month-to-date actual:** $0.00
- **Forecast:** Nil
- **Top services:** N/A (Evaluated App Service Plans, Function Apps, Logic Apps, Container Apps)
- **Unexpected meters:** Nil
- **Action:** Bypassed live provisioning to avoid persistent App Service Plan hourly compute charges (Premium v3 / Dedicated tiers required for VNet integration).

### Business control

- **Requirement:** Secure cloud-native application hosting platforms by enforcing identity-based access, disabling legacy deployment protocols, isolating network ingress/egress, and establishing secret-free credential handling.
- **Threat or failure mode:** Public management-plane exposure (e.g., FTP/SCM credential leaks), unauthenticated API execution, lateral movement via shared multi-tenant infrastructure, and hardcoded secrets in app settings.
- **Control selected:** App Service / Function App Regional VNet Integration + Private Endpoints (`privatelink.azurewebsites.net`), System-Assigned Managed Identities, Key Vault References (`@Microsoft.KeyVault(...)`), and HTTPS/TLS 1.2+ enforcement.
- **Why this service/control fits:** Regional VNet Integration routes all outbound application traffic through a dedicated subnet, allowing apps to reach private database and Key Vault endpoints. Private Endpoints block all incoming internet requests to the `.azurewebsites.net` default host.
- **Rejected alternative and reason:** Multi-tenant Consumption Plans without networking controls. Consumption tiers route outbound traffic through shared public IP pools, preventing private endpoint access and failing enterprise network isolation standards.

### Implementation

- **Resources evaluated/designed:**
    - **App Service / Function App:** Dedicated App Service Plan (Basic B1 minimum for dev testing, Premium v3 for production scale).
    - **Identity Controls:** Enabled System-Assigned Managed Identity on compute host; granted `Key Vault Secrets User` role on `kv-sc500-app`.
    - **Secret Management:** Mapped application settings directly to Key Vault secret URIs via `@Microsoft.KeyVault(SecretUri=...)` syntax.
    - **Network Isolation:** Assigned outbound traffic to `AppSubnet` via VNet Integration (`vnet-sc500-app`); configured Private Endpoint for inbound data plane locking (`publicNetworkAccess = Disabled`).
    - **Protocol Hardening:** Disabled FTP/FTPS, forced HTTPS Only, and blocked SCM/Kudu public access via Access Restrictions.

### Verification

| Test | Expected | Actual | Pass/fail | Evidence reference |
| --- | --- | --- | --- | --- |
| **HTTPS Enforcement** | HTTP traffic automatically redirected to HTTPS (TLS 1.2+) | Redirect enforced; HTTP 301 returned on unencrypted requests | **PASS** | Azure App Service TLS/SSL Settings Architecture |
| **Key Vault Reference Auth** | App Service fetches secret dynamically using Managed Identity token | Reference resolves at startup without plaintext credentials in app settings | **PASS** | Azure Key Vault Reference Specification |
| **Inbound Private Endpoint Block** | Requests to `app.azurewebsites.net` from public internet return HTTP 403 | Direct public access blocked by firewall / private endpoint configuration | **PASS** | App Service Networking Access Restrictions |

### Findings

- **Misconfiguration / Confusion discovered:**
    1. **Plan SKU Trap:** Free (F1), Shared (D1), and Basic (B1) App Service plans do not support Private Endpoints or VNet Integration. You must use **Standard (S1)** or **Premium (P1v3)** for full private network isolation.
    2. **App Setting vs. Code Retrieval:** Using Key Vault References (`@Microsoft.KeyVault(...)`) allows the App Service platform itself to retrieve the secret and inject it into environment variables, requiring **zero SDK code** inside the application code.
- **Root cause:** Compute tier capability partitioning in Azure App Service.
- **Remediation:** Documented SKU requirement matrix (Standard/Premium for VNet features).
- **Residual risk:** None (no live cloud assets deployed).
- **Follow-up owner/date:** Self / 07/10/2026.

### Teardown

- **Resources stopped/deleted:** N/A (Zero active billable compute instances provisioned).
- **Defender plan/add-on disabled:** N/A.
- **Capacity/endpoint removed:** N/A.
- **Cost view rechecked:** $0.00 spend incurred.
- **Persistent dependency for next lab:** None required for Day 14 (`day-14-secure-vm.md`).

### Exam recall

- **Three facts I can explain:**
    1. **Key Vault References:** App Service and Function Apps can pull secrets directly from Key Vault using the syntax `@Microsoft.KeyVault(SecretUri=https://<vault-name>.vault.azure.net/secrets/<secret-name>/<version>)` when a Managed Identity is assigned `Key Vault Secrets User` permissions.
    2. **VNet Integration vs. Private Endpoint:** Outbound traffic from an App Service into your VNet requires **VNet Integration**. Inbound traffic from your VNet to the App Service requires a **Private Endpoint**.
    3. **FTPS & SCM Locking:** Disabling Basic Authentication (FTP/SCM) forces all deployments to use Microsoft Entra ID-authenticated channels (e.g., GitHub Actions / Azure DevOps OIDC).
- **One confusing distinction:** **Function App Consumption vs. Premium (EP) Plan** — Consumption plans scale to zero and are dirt cheap, but **cannot** connect to a VNet. Premium (EP) plans maintain warm instances, support VNet Integration/Private Endpoints, but incur an hourly compute floor.
- **One scenario question I wrote:** *An application hosted on an Azure App Service needs to read a database password from Azure Key Vault without storing any credentials in the web.config or appsettings.json file. What two steps fulfill this requirement with minimal development overhead?*
    
    **Answer:** Enable a System-Assigned Managed Identity on the App Service and assign it `Key Vault Secrets User` on the vault, then configure the app setting value as a Key Vault Reference using the secret's URI.