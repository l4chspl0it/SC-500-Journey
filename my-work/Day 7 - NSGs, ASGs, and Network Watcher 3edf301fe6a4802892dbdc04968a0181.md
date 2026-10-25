# Day 7 - NSGs, ASGs, and Network Watcher

Status: Done

## **Session**

• **Lab:** NSGs, ASGs, and Network Watcher
• **Date/time and time zone:** 06/10/2026 (AEDT)
• **Start/end time:** 13:52 - 14:42
• **Hands-on, trial-dependent, or simulated:** Hands-on
**Cost checkpoint**
• **Credit remaining:** $284
• **Month-to-date actual:** Nil
• **Forecast:** Nil
• **Top services:** N/A (Standalone test NICs generate zero compute/idle charges)
• **Unexpected meters:** Nil
• **Action:** Nil

**Business control**
• **Requirement:** NCES needs application-role rules rather than IP-address spreadsheets and must be able to prove why traffic is allowed or denied.   
• **Threat or failure mode:** Lateral movement across application tiers and unverified firewall rule sets leading to undetected broad access.
• **Control selected:** Application Security Groups (`asg-web`, `asg-api`) combined with Network Security Group (`nsg-app`) explicit rules and Network Watcher diagnostic tools (`test-ip-flow`, `show-next-hop`).   
• **Why this service/control fits:** ASGs allow grouping workloads logically by application role rather than static IP addresses. Network Watcher provides definitive packet-path proof for security audits.   
• **Rejected alternative and reason:** Hardcoded IP address ranges in NSG rules (unmaintainable at scale and prone to dynamic IP reassignment drift).
**Implementation**

• **Resources changed:**
    ◦ Resource Group: `rg-sc500-core-cc` (Canada Central)   
    ◦ ASGs: `asg-web`, `asg-api`
   
    ◦ NSG: `nsg-app` attached to `AppSubnet` on `vnet-sc500-app`
   
    ◦ Diagnostic NICs: `nic-web-test` (`10.21.1.4`), `nic-api-test` (`10.21.1.5`)   
• **Identity and role/scope:** Subscription level contributor / Network Contributor on `rg-sc500-core-cc`.
• **Management-plane controls:** Tags assigned to resource groups and network objects.
• **Data-plane controls:** N/A
• **Network controls:**
    ◦ Priority 100: `Allow-Web-To-API-8443` (Source: `asg-web` $\rightarrow$ Dest: `asg-api`, TCP 8443, Allow)   
    ◦ Priority 110: `Allow-AzureLoadBalancer-Inbound` (Source: `AzureLoadBalancer`, Allow)   
    ◦ Priority 200: `Deny-Lateral-App-Traffic` (Source: `VirtualNetwork` $\rightarrow$ Dest: `VirtualNetwork`, Any, Deny)   
• **Monitoring destination:** Network Watcher in Canada Central.   
**VerificationTestExpectedActualPass/failEvidence referenceAllowed path**`10.21.1.4` $\rightarrow$ `10.21.1.5:8443` allowed`Access: Allowed` via rule `Allow-Web-To-API-8443`**PASS**`az network watcher test-ip-flow` output   **Denied path**`10.21.1.4` $\rightarrow$ `10.21.1.5:8080` denied`Access: Denied` via rule `Deny-Lateral-App-Traffic`**PASS**`az network watcher test-ip-flow` output   **Logging/detection**Defender/review flags broad rule; Network Watcher traces next hopOverly broad rule flagged; Next hop returned `System route / Internet` for `8.8.8.8`**PASS**`az network watcher show-next-hop` output   **Recovery/rollback**Overly broad test rule deleted cleanlyRule `Overly-Broad-Test-Rule` deleted without impacting priority 100/200 rules**PASS**`az network nsg rule delete` execution   
****

**Findings**

• **Misconfiguration discovered:**
    1. Standalone NICs cannot have ASGs bound via the Azure Portal IP Configuration menu.   
    2. Network Watcher NSG Diagnostics and IP Flow Verify enforce a mandatory VM association in the Azure Portal GUI.   
• **Root cause:** Azure Portal UI validation requires a compute context (`Microsoft.Compute/virtualMachines`) for diagnostic blades and NIC-level ASG binding.   
• **Remediation:** Executed ASG assignment via CLI (`az network nic ip-config update --application-security-groups`) and diagnostic checks via CLI (`az network watcher test-ip-flow`).   
• **Residual risk:** None. Subnet-level NSG remains in place with strict deny-all lateral rules.   
• **Follow-up owner/date:** Self / 06/10/2026.

**Teardown**
• **Resources stopped/deleted:** Deleted temporary rule `Overly-Broad-Test-Rule`.   
• **Defender plan/add-on disabled:** N/A
• **Capacity/endpoint removed:** Standalone test NICs can remain or be removed (cost is $0.00).
• **Cost view rechecked:** $0.00 incurred.
• **Persistent dependency for next lab:** Keep `nsg-app`, `asg-web`, `asg-api`, and `vnet-sc500-app` for future routing and firewall labs.   

**Exam recall**

• **Three facts I can explain:**
    1. Application Security Groups (ASGs) allow network rules to be defined using workload identity/tags instead of static IP addresses or subnets.   
    2. Subnet NSGs and NIC NSGs process independently; traffic must be permitted by *both* layers to pass.
    3. Network Watcher IP Flow Verify checks 5-tuple rules to evaluate explicit flow allow/deny without generating synthetic test traffic.   

• **One confusing distinction:** **NSG Diagnostics vs. IP Flow Verify** — NSG Diagnostics evaluates the overall rule configuration context on a VM/NIC, whereas IP Flow Verify checks a specific 5-tuple direction (Source/Dest IP, Source/Dest Port, Protocol) to pinpoint the exact winning rule.   
• **One scenario question I wrote:** *An engineer deploys an application with a Web tier ASG and an API tier ASG in the same subnet. A rule with Priority 100 allows TCP 8443 from Web to API. A rule with Priority 200 denies VirtualNetwork to VirtualNetwork traffic. If an attacker attempts to connect from the Web tier to the API tier on port 22, which rule processes the traffic and what is the outcome?*
**Answer:** Priority 200 (`Deny-Lateral-App-Traffic`) processes the request and **Denies** access, because port 22 does not match the Priority 100 allow rule, falling through to the next lowest priority explicit deny rule before default VNet rules are evaluated.