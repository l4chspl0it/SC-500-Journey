# Day 6 - Hub-spoke network foundation

Status: Done

## Session

- Lab: **Hub-spoke network foundation**
- Date/time and time zone: 06/`0/2026
- Start/end time: 11:18pm
- Hands-on, trial-dependent, or simulated: Hands-on
- Microsoft documentation checked on:
- Region/SKU used: Canada Central and East US 2

## Cost checkpoint

- Credit remaining: $284
- Month-to-date actual: Nil
- Forecast: Nil
- Top services:
- Unexpected meters: N/A
- Action: N/A

## Business control

- Requirement: The public portal, internal application, AKS, and data services must not share one flat network. NCES also needs a central path for inspection services without paying for them all month.
- Threat or failure mode: Flat network allows lateral movement and no central traffic inspection point
- Control selected: Hub and spoke network design via vnets, subnets, private endpoints, AZ firewall and Azure bastion to allow central traffic inspection and routing
- Why this service/control fits: isolates traffic into management boundaries whilst enforcing inspection via the hub. Vnet peering enforces strict boundaries by default.
- Rejected alternative and reason: Single flat network - a single vnet allows internal routing between subnets with no boundaries at the management plane.

## Implementation

- Resources changed:
    - R**esource Group:** `rg-sc500-core-cc` (Australia East).   **Network Address Spaces:**
    • Hub VNet (`vnet-sc500-hub`): `10.20.0.0/16`
       
    • App Spoke VNet (`vnet-sc500-app`): `10.21.0.0/16`
       
    • AKS Spoke VNet (`vnet-sc500-aks`): `10.22.0.0/16`
       **Reserved Subnets:**
    • `AzureFirewallSubnet` (`10.20.1.0/26`) in Hub VNet (no NSG attached).
    • `AzureBastionSubnet` (`10.20.2.0/26`) in Hub VNet.
    • `AppSubnet` (`10.21.1.0/24`) in App Spoke VNet.
    • `AksSubnet` (`10.22.1.0/24`) in AKS Spoke VNet.**Peering Configuration:**
    • `peer-hub-to-app` $\leftrightarrow$ `peer-app-to-hub` (Connected, Gateway Transit disabled).   
    • `peer-hub-to-aks` $\leftrightarrow$ `peer-aks-to-hub` (Connected, Gateway Transit disabled).
- Identity and role/scope:
- Management-plane controls:
- Data-plane controls:
- Network controls:
- Monitoring destination:

![image.png](Day%206%20-%20Hub-spoke%20network%20foundation/image.png)

![image.png](Day%206%20-%20Hub-spoke%20network%20foundation/image%201.png)

## Findings

- Misconfiguration discovered: N/A
- Root cause: N/A
- Remediation: N/A
- Residual risk: N/A
- Follow-up owner/date: N/A

## Teardown

- Resources stopped/deleted: N/A
- Defender plan/add-on disabled: N/A
- Capacity/endpoint removed: N/A
- Cost view rechecked: Yes
- Persistent dependency for next lab: N/A

## Exam recall

- Three facts I can explain:
    - **Non-Transitive VNet Peering:** VNet peering is non-transitive. If VNet A is peered to VNet B, and VNet B is peered to VNet C, VNet A **cannot** communicate with VNet C through VNet B unless a Network Virtual Appliance or Azure Firewall is configured with custom UDRs.
    - **Reserved Subnet Naming:** System-managed subnets like `AzureFirewallSubnet` and `AzureBastionSubnet` require exact case-sensitive names and cannot have NSGs directly assigned to `AzureFirewallSubnet`.
    - **Bi-Directional Peering Requirements:** VNet peering requires two independent peering links (one on each VNet) pointing to each other before the connection status transitions to `Connected`
- One confusing distinction: The difference between **VNet Peering** (connects virtual networks at the ARM layer using Azure's backbone without bandwidth bottlenecks) vs. **VPN Gateways / ExpressRoute** (requires gateway compute instances, introduces throughput caps, and incurs hourly costs).
    - **One Scenario Question I Wrote:** *An architect peers Spoke VNet A and Spoke VNet B to a central Hub VNet. Workloads in Spoke VNet A are unable to reach workloads in Spoke VNet B. What is the root cause?*
    • **Answer:** VNet peering is non-transitive. Traffic cannot pass through the Hub VNet between spokes without a central firewall or NVA and User-Defined Routes (UDRs) specifying the next hop.