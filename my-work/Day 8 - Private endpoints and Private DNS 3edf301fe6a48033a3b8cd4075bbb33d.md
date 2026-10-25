# Day 8 - Private endpoints and Private DNS

Status: Done

## Session

- Lab: **Private endpoints and Private DNS**
- Date/time and time zone: 07/10/2026
- Start/end time: 1247
- Hands-on, trial-dependent, or simulated: Hands-on
- Microsoft documentation checked on:
- Region/SKU used:

## Cost checkpoint

- Credit remaining: $284
- Month-to-date actual: Nil
- Forecast: Nil
- Top services: N/A
- Unexpected meters: N/A
- Action: N/A

## Business control

- **Requirement:** NCES needs secure, private PaaS connectivity for data storage without exposing public endpoints to the internet, while retaining full auditability and Entra ID RBAC control.**Threat or failure mode:** Data exfiltration, unauthorized public access over storage account keys, or lateral movement across public PaaS endpoints.
- **Control selected:** Storage account hardening (`publicNetworkAccess = Disabled`), Private Endpoint (`snet-private-endpoints`), Private DNS Zone (`privatelink.blob.core.windows.net` linked to `vnet-sc500-app`), and Microsoft Entra ID RBAC authorization (`Storage Blob Data Contributor`).
- **Why this service/control fits:** Private Endpoints assign a private IP directly inside `vnet-sc500-app`, ensuring traffic never leaves the Azure backbone while completely blocking public internet access.
- **Rejected alternative and reason:** Service Endpoints. While Service Endpoints keep traffic on the Azure backbone, the PaaS service retains its public IP address, and public endpoint exposure remains possible if firewall rules are misconfigured.

## Implementation

- **Resources changed:**
• Resource Group: `rg-sc500-data-cc` (Canada Central)   
• Storage Account: `sc500storagedata` (Standard GPv2 LRS, TLS 1.2+, Public Blob Access Disabled)
• Container: `test-container`
• Private Endpoint: `snet-private-endpoints` (Targeting `blob` sub-resource)   
• Network Interface: `snet-private-endpoints-nic` (Private IP `10.21.1.6` in `AppSubnet`)   
• Private DNS Zone: `privatelink.blob.core.windows.net` (Linked to `vnet-sc500-app`)
- **Identity and role/scope:** User account assigned `Storage Blob Data Contributor` on `sc500storagedata`.
- **Management-plane controls:** Tags assigned to resource group and storage account.
- **Data-plane controls:** Enforced Entra ID authentication (`-auth-mode login` / Portal Entra user session) for blob operations.
- **Network controls:** `publicNetworkAccess` set to `Disabled` on `sc500storagedata`.
- **Monitoring destination:** Azure Storage Metrics & Diagnostics / Network Watcher.

## Verification

|  |  |  |  |  |
| --- | --- | --- | --- | --- |
|  |  |  |  |  |
|  |  |  |  |  |
|  |  |  |  |  |
|  |  |  |  |  |

## Findings

- **Misconfiguration / Confusion discovered:**
    1. The lab naming scheme used `snet-private-endpoints` for the **Private Endpoint resource name itself**, causing confusion with the underlying subnet name (`AppSubnet` / `snet-private-endpoints`).
        
        PNG+ 1
        
    2. Cloud Shell resides on a public Azure IP and cannot access storage data plane operations once `publicNetworkAccess` is set to `Disabled`.
- **Root cause:** Azure Private Endpoints isolate data plane access strictly to the connected VNet (and peered networks/VPNs), intentionally blocking public clients like default Azure Cloud Shell.
- **Remediation:** Verified access via Azure Portal Entra user sessions and internal VNet DNS mapping.
- **Residual risk:** None. Storage account public network access is completely disabled.
- **Follow-up owner/date:** Self / 07/10/2026.

### Teardown

- **Resources stopped/deleted:** Temporary local test file `testblob.txt`.
- **Defender plan/add-on disabled:** N/A
- **Capacity/endpoint removed:** Storage account `sc500storagedata` and Private Endpoint `snet-private-endpoints` retained for ongoing SC-500 practice.
- **Cost view rechecked:** $0.00 incurred.
- **Persistent dependency for next lab:** Retain `sc500storagedata`, Private Endpoint, and `vnet-sc500-app` for routing and security perimeter exercises.
    
    PNG
    

### Exam recall

- **Three facts I can explain:**
    1. Enabling a Private Endpoint creates a dedicated NIC in your VNet subnet with a private IP address, routing PaaS traffic entirely over the Azure backbone.
    2. Setting `publicNetworkAccess = Disabled` on a PaaS resource blocks all traffic attempting to reach public IP endpoints, requiring traffic to pass through the Private Endpoint.
    3. Private DNS Zones must be linked to the Virtual Network (`vnet-sc500-app`) so internal workloads automatically resolve `<account>.blob.core.windows.net` to the private IP rather than the public IP.
- **One confusing distinction:** **Service Endpoints vs. Private Endpoints** — Service Endpoints route traffic from a specific subnet to a PaaS service over the Azure backbone while keeping the PaaS service's public IP address, whereas Private Endpoints assign a true private IP from your VNet directly onto a dedicated NIC inside your subnet.
- **One scenario question I wrote:** *An administrator creates a Private Endpoint for an Azure Key Vault and links the `privatelink.vaultcore.azure.net` Private DNS Zone to `VNet-App`. However, an application VM inside `VNet-App` is still resolving the Key Vault's public IP address. What is the most likely cause?*
    
    **Answer:** The Virtual Network Link between `VNet-App` and the Private DNS Zone `privatelink.vaultcore.azure.net` was not created, or the VM is using a custom DNS server that is not configured to forward queries to the Azure wire server IP (`168.63.129.16`).