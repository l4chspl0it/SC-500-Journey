# Day 12 - Enterprise network controls

Status: Done

## 

### Session

- **Lab:** Day 12 – Enterprise Network Controls (SC-500)
- **Date/time and time zone:** 07/10/2026 (AEDT)
- **Start/end time:** 16:00 - 16:20
- **Hands-on, trial-dependent, or simulated:** Simulated / Architecture Decision Record

### Cost checkpoint

- **Credit remaining:** $284
- **Month-to-date actual:** $0.00
- **Forecast:** Nil
- **Top services:** N/A (Evaluated Azure Firewall, AVNM, VPN Gateway, Virtual WAN, Private Link Service)
- **Unexpected meters:** Nil
- **Action:** Bypassed live provisioning to avoid deployment hourly floors ($0.395–$1.25/hr for Azure Firewall, $73/mo subscription fee for AVNM); completed lab via architecture decision review and portal validation steps.

### Business control

- **Requirement:** Evaluate centralized network governance, hub-and-spoke inspection paths, and secure cross-boundary access for partner and remote user connectivity.
    
    PNG
    
- **Threat or failure mode:** Uninspected egress/ingress traffic, unauthorized management access (RDP/SSH) exposed to internet, and overly permissive VNet-to-VNet lateral movement.
    
    PNG
    
- **Control selected:** Security Admin rules via Azure Virtual Network Manager (AVNM) for baseline rules, Azure Firewall (Basic/Standard) for hub traffic inspection, and Microsoft Entra Private Access for user-to-app zero trust connectivity.
    
    PNG
    
- **Why this service/control fits:** AVNM allows security teams to enforce non-overrideable baseline rules across all spoke VNets before local NSGs evaluate. Azure Firewall centralizes network/app filtering and threat intelligence.
    
    PNG+ 1
    
- **Rejected alternative and reason:** Virtual WAN with Secured Hub. Rejected for POC/small-scale footprint due to high base cost and unnecessary complexity compared to a standard hub-and-spoke VNet peering architecture.
    
    PNG
    

### Implementation

- **Resources evaluated/designed:**
    - **Part A (AVNM):** Network Groups scoped to lab subscription. Security Admin rules evaluated *before* local NSG evaluation to enforce mandatory internet inbound blocks on ports 22 and 3389.
        
        PNG+ 1
        
    - **Part B (Azure Firewall):** Hub firewall placed in `AzureFirewallSubnet` with user-defined routes (UDRs `0.0.0.0/0` -> Firewall Private IP) forcing spoke egress through central inspection.
        
        PNG
        
    - **Part C (Architecture Stations):**
        - *VPN Gateway:* P2S with Microsoft Entra authentication for admin access.
            
            PNG
            
        - *Virtual WAN:* Secured Hub design mapped for multi-office enterprise scaling.
            
            PNG
            
        - *Entra Private Access:* App segments and connector groups leveraging Global Secure Access for identity-centric micro-segmentation without network exposure.
            
            PNG
            
        - *Private Link Service:* Standard Load Balancer frontend mapped to consumer private endpoints across tenant boundaries.
            
            PNG
            

### Verification

| Test | Expected | Actual | Pass/fail | Evidence reference |
| --- | --- | --- | --- | --- |
| **AVNM Rule Precedence** | Security Admin rules evaluate before NSG rules | Evaluated at network fabric layer; blocks traffic regardless of NSG `Allow` rules   PNG | **PASS** | Azure Networking Architecture Documentation |
| **Firewall Route Hijack** | Spoke route table (`0.0.0.0/0`) forces all outbound internet traffic to Firewall private IP | UDR pattern validated; prevents direct internet breakout   PNG | **PASS** | Azure Route Table Configuration Design   PNG |
| **Entra Private Access vs. VPN** | Private Access enforces app-level access via Conditional Access without routing whole subnet | Access granted per-application segment using Global Secure Access client   PNG | **PASS** | Microsoft Entra Private Access Architecture   PNG |

### Findings

- **Misconfiguration / Confusion discovered:**
    1. Security Admin Rules in AVNM do not replace NSGs; they act as a higher-level policy fence. If AVNM explicitly denies a port, an NSG `Allow` rule on the NIC/Subnet cannot override it.
        
        PNG+ 1
        
    2. Azure Firewall incurs a fixed hourly cost immediately upon deployment regardless of traffic throughput ($0.395/hr for Basic, $1.25/hr for Standard).
- **Root cause:** Native cloud architecture separation between policy governance (AVNM) and localized packet filtering (NSGs).
    
    PNG
    
- **Remediation:** Documented design decision record using NSGs for POC testing and AVNM for enterprise scaling.
    
    PNG
    
- **Residual risk:** None (no live cloud assets deployed).
- **Follow-up owner/date:** Self / 07/10/2026.

### Teardown

- **Resources stopped/deleted:** N/A (Zero active billable resources provisioned).
- **Defender plan/add-on disabled:** N/A.
- **Capacity/endpoint removed:** N/A.
- **Cost view rechecked:** $0.00 spend incurred.
- **Persistent dependency for next lab:** None required for Day 13 (`day-13-app-platform.md`).

### Exam recall

- **Three facts I can explain:**
    1. **AVNM Rule Evaluation Order:** Security Admin Rules created in Azure Virtual Network Manager are evaluated **before** NSG rules. If AVNM blocks a packet, local NSG rules are never processed.
        
        PNG+ 1
        
    2. **Entra Private Access vs. VNet Peering:** Entra Private Access secures access at Layer 7 to specific IP/port app segments using identity and Conditional Access, whereas VNet Peering connects entire network address spaces at Layer 3.
        
        PNG
        
    3. **Private Link Service (PLS):** Allows a service provider to expose a service behind an internal Standard Load Balancer privately to clients in different Entra tenants/VNets via Private Endpoints, eliminating NAT/IP overlap conflicts.
        
        PNG
        
- **One confusing distinction:** **Azure Firewall Basic vs. Standard SKU Pricing** — Basic has a lower hourly floor ($0.395/hr) but a higher per-GB data processing fee ($0.065/GB) compared to Standard ($1.25/hr floor, $0.016/GB processing). Standard becomes cheaper for high-throughput workloads.
- **One scenario question I wrote:** *An enterprise requires all spoke VNets to block inbound SSH (port 22) from the internet, but application teams frequently add high-priority NSG allow rules that override central policy. Which service prevents this without deleting the application teams' NSGs?*
    
    **Answer:** Create an Azure Virtual Network Manager (AVNM) Security Admin Configuration with a `Deny` rule for port 22 scoped to the network group. Because Security Admin rules take precedence over NSGs, internet SSH will be blocked regardless of local NSG rules.
    
    PNG+ 1