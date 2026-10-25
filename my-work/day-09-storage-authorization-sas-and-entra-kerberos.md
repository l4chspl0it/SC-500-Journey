# Day 9 - Storage authorization, SAS, and Entra Kerberos

Status: Done

## **Session**

• **Lab:** PaaS Security Hardening, Access Policy Limits & Azure Files Entra Kerberos
• **Date/time and time zone:** 07/10/2026 (AEDT)
• **Start/end time:** 12:30 - 13:58
• **Hands-on, trial-dependent, or simulated:** Hands-on
**Cost checkpoint**
• **Credit remaining:** $284
• **Month-to-date actual:** Nil
• **Forecast:** Nil
• **Top services:** Storage Account (Standard GPv2 LRS & FileStorage Premium SSD), Private Endpoint
• **Unexpected meters:** Nil
• **Action:** Delete the short-lived Premium `FileStorage` instance immediately after testing to prevent ongoing hourly billing.   

**Business control**
• **Requirement:** NCES requires secure, private PaaS storage with disabled account keys, enforced Entra ID RBAC, and cloud-only Entra Kerberos SMB share access for file storage.   
• **Threat or failure mode:** Public endpoint exposure, bypass of Conditional Access via unexpiring Account Keys, token replay attacks, or broad MFA policies blocking silent Kerberos SMB authentication.   
• **Control selected:** Storage Account Hardening (`publicNetworkAccess = Disabled`, `allowSharedKeyAccess = false`), Private Endpoint (`snet-private-endpoints`), User Delegation SAS, and Azure Files Microsoft Entra Kerberos authentication.   
• **Why this service/control fits:** Disabling Shared Key Access forces 100% of data plane traffic through Entra ID, ensuring auditability and Conditional Access enforcement, while Entra Kerberos provides seamless SMB access without needing domain controllers.   
• **Rejected alternative and reason:** Service Endpoints & Microsoft Entra Domain Services (AD DS). Service Endpoints retain public IPs, and Entra Domain Services requires costly managed domain controllers ($100+/mo) that are redundant for cloud-only identities.   

**Implementation**
• **Resources changed:**
    ◦ Resource Group: `rg-sc500-data-cc` (Canada Central)   
    ◦ Storage Account (Blob): `sc500storagedata` (Standard GPv2 LRS, `allowSharedKeyAccess = false`, `publicNetworkAccess = Disabled`)
    ◦ Storage Account (Files): Premium `FileStorage` account with `sc500fileshare` SMB share   
    ◦ Private Endpoint: `snet-private-endpoints` (Targeting `blob` sub-resource)   
    ◦ Network Interface: `snet-private-endpoints-nic` (Private IP `10.21.1.6` in `AppSubnet`)   
    ◦ Identity / Auth: Microsoft Entra Kerberos enabled on Azure Files with Admin Consent granted to the enterprise app object   
• **Identity and role/scope:** User account assigned `Storage Blob Data Contributor` and `Storage File Data SMB Share Contributor`.   
• **Management-plane controls:** Admin consent granted for the Microsoft Entra Kerberos service principal.   
• **Data-plane controls:** Forced Entra ID user authentication (`--auth-mode login` / User Delegation SAS).
• **Network controls:** Enforced SMB 3.1.1 encryption, Private Endpoint isolation, and public access lockdown.   
• **Monitoring destination:** Azure Storage Metrics & Diagnostics / Entra ID Audit Logs.   

**VerificationTestExpectedActualPass/failEvidence referencePrivate Endpoint Routing**`sc500storagedata.blob.core.windows.net` resolves to `10.21.1.6` inside VNetCNAME resolved to `privatelink.blob.core.windows.net` and private IP `10.21.1.6`**PASS**Private Endpoint NIC IP Configuration (`snet-private-endpoints-nic`)   **Denied Public Path**Public browser requests blocked with HTTP 403 when public access disabledError returned: `403 Forbidden / This request is not authorized` with client public IP listed**PASS**Storage Firewall Error Banner**User Delegation SAS**SAS creation requires both clear data-plane network path and `Storage Blob Data Delegator/Contributor` roleGenerated successfully when client IP was temporarily whitelisted and RBAC validated**PASS**Container Shared Access Tokens Blade**Shared Key Hardening**Setting `allowSharedKeyAccess = false` blocks master key auth while preserving Entra ID / User Delegation SASShared account key authentication blocked; Entra user delegation remains operational**PASS**Storage Account Configuration Settings**Entra Kerberos Setup**Cloud-only identity SMB share authentication enabled via Azure FilesEntra Kerberos enabled, Enterprise App consented, and share-level RBAC assigned**PASS**Azure Files File Share Settings & Entra ID Enterprise Apps   

**Findings**

• **Misconfiguration / Confusion discovered:**
    1. The lab naming scheme used `snet-private-endpoints` for the **Private Endpoint resource name itself**, causing confusion with the underlying subnet (`AppSubnet`).   
    2. Generating a User Delegation Key requires an active **data-plane network connection** (whitelisted IP or VNet access) in addition to the `Storage Blob Data Contributor` RBAC role.
    3. Stored Access Policies on a Blob Container are capped at **5 policies maximum**. If 1 exists, only 4 additional policies can be added.
• **Root cause:** Public network isolation blocks browser-based data-plane API calls (`comp=userdelegationkey`) unless the client IP is explicitly permitted.
• **Remediation:** Temporarily added client IP to firewall rules to issue SAS tokens, then re-disabled public network access.
• **Residual risk:** None. Shared account key auth disabled and public access locked down.
• **Follow-up owner/date:** Self / 07/10/2026.

**Teardown**
• **Resources stopped/deleted:** Deleted the temporary Premium `FileStorage` account immediately after testing.   
• **Defender plan/add-on disabled:** N/A
• **Capacity/endpoint removed:** Premium SMB share and associated Kerberos enterprise application object removed.   
• **Cost view rechecked:** $0.00 incurred.
• **Persistent dependency for next lab:** Retain `sc500storagedata`, Private Endpoint, and `vnet-sc500-app` for upcoming networking and security perimeter exercises.   

**Exam recall**

• **Three facts I can explain:**
    1. **User Delegation SAS vs. Account Key SAS:** User Delegation SAS uses temporary keys derived from Microsoft Entra ID credentials (requiring `Storage Blob Data Delegator`), whereas Account Key SAS uses master keys that bypass Entra ID entirely.
    2. **Stored Access Policy Limits:** A blob container supports a maximum of **5 Stored Access Policies** at a time.
    3. **Azure Files Entra Kerberos:** Enables cloud-only Entra joined devices to mount SMB shares using Kerberos without requiring on-premises AD domain controllers or Entra Domain Services.   

• **One confusing distinction:** **User Delegation SAS vs. User-Bound SAS** — User Delegation SAS is signed by an Entra identity but can be used by anyone holding the URL, whereas a **User-Bound SAS** restricts usage so that only the user matching the specific Entra Object ID (OID) can execute requests with the token.

• **One scenario question I wrote:** *An administrator attempts to disable Shared Key access (`allowSharedKeyAccess = false`) on a storage account. Which authentication method will CONTINUE to work for accessing blob containers?*
**Answer:** Microsoft Entra ID OAuth authentication and User Delegation SAS tokens.

For the exam, the max shared access policies that can be created in a storage container is 5. The max immutable blob storage policies that can be created is 1

![image.png](Day%209%20-%20Storage%20authorization,%20SAS,%20and%20Entra%20Kerb/image.png)

- 

##