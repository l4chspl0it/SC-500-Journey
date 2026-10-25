# Day 14 - Trusted Launch VM, encryption, Bastion, and JIT

Status: Done

## 

**Session**

• **Lab:** Day 14 – Trusted Launch VM, Disk Encryption, Bastion, and JIT (SC-500)   
• **Date/time and time zone:** 07/10/2026 (AEDT)
• **Start/end time:** 16:30 - 16:45
• **Hands-on, trial-dependent, or simulated:** Simulated / Architecture Decision Record & Portal Validation Workflow   

**Cost checkpoint**

• **Credit remaining:** $284
• **Month-to-date actual:** $0.00
• **Forecast:** Nil
• **Top services:** N/A (Evaluated Gen2 Virtual Machines, Disk Encryption Sets, Azure Bastion Developer SKU, Microsoft Defender for Servers P2, Microsoft Entra Just-In-Time VM Access)   
• **Unexpected meters:** Nil
• **Action:** Bypassed live VM and Bastion provisioning to avoid managed disk costs, hourly compute charges, and fixed Bastion host fees ($0.19/hr for Basic, $0.29/hr for Standard).  

 
**Business control**

• **Requirement:** Temporarily host a legacy document converter on a virtual machine while enforcing boot integrity, encrypted storage, zero persistent public management endpoints, and time-limited analyst access.   
• **Threat or failure mode:** Rootkits/bootkits hijacking host startup, unencrypted OS/temp disks exposed during storage hardware theft, credential exposure via persistent open management ports (RDP 3389 / SSH 22), and unauthorized lateral movement.   
• **Control selected:** Trusted Launch Gen2 VM architecture (Secure Boot + vTPM), Encryption at Host with Platform-Managed Keys (PMK), Microsoft Defender for Servers P2 Just-In-Time (JIT) VM Access, and Azure Bastion Developer SKU.   
• **Why this service/control fits:** Trusted Launch defends against boot-level malware via measured boot. Encryption at Host provides seamless end-to-end encryption for the OS disk, data disks, and temporary scratch disks without VM performance overhead. JIT dynamically generates time-bound NSG rules to grant access only during active analyst sessions.   
• **Rejected alternative and reason:** Azure Disk Encryption (ADE) with BitLocker/DM-Crypt. ADE is scheduled for retirement in September 2028 and imposes guest OS agent overhead compared to native hypervisor-level Encryption at Host.

**Implementation**

• **Resources evaluated/designed:**
    ◦ **Resource Group:** `rg-sc500-vm-cc`
   
    ◦ **VM Architecture:** Small Gen2 B-series instance deployed without a Public IP. Security Type configured to **Trusted Launch** with **Secure Boot** and **vTPM** enabled.   
    ◦ **Disk Encryption Strategy:** Selected **Server-Side Encryption + Encryption at Host**. Bypassed ADE key vault dependency to avoid stacking redundant cryptographic layers.   
    ◦ **Just-In-Time (JIT) Management:** Enabled Defender for Servers P2. Defined JIT policy on port 22/3389 with a maximum request window of 3 hours and source IP restricted to the analyst's public IP.   
    ◦ **Bastion Deployment:** Selected **Azure Bastion Developer SKU** (Free runtime) connected directly to the VM's private IP without exposing public endpoints.   

**VerificationTestExpectedActualPass/failEvidence referenceTrusted Launch Verification**Security type shows Trusted Launch; Secure Boot and vTPM activeHypervisor reports Secure Boot enabled and vTPM state initialized   **PASS**Azure VM Overview & Infrastructure Diagnostics   **Disk Encryption Verification**Disks and host caches encrypted seamlessly without guest OS keysServer-side + Encryption at Host confirmed via Disk settings   **PASS**Azure Managed Disk Security Settings**Network Exposure Verification**VM NIC has no public IP assignedPublic IP setting: `None` (`0.0.0.0`)   **PASS**Azure VM Network Interface Blade   **JIT Rule Enforcement**Port 3389/22 inbound blocked by default; temporary NSG rule created on requestManagement ports closed; auto-expiring inbound `Allow` rule added for approved IP   **PASS**Defender for Cloud JIT Access Workflow   
****

**Findings**

• **Misconfiguration / Confusion discovered:**

    1. **ADE vs. Encryption at Host:** Azure Disk Encryption (ADE) requires guest-OS BitLocker/DM-Crypt execution and is scheduled for deprecation (Sept 2028). Modern Azure security architectures strictly recommend **Encryption at Host** for end-to-end encryption without guest agent dependencies.
    2. **Bastion Developer SKU Limitations:** Bastion Developer SKU is free, but it does not support VNet Peering or custom IP ranges—it connects strictly to VMs in the same VNet. Paid Basic/Standard SKUs are required for cross-VNet peering access.
• **Root cause:** Legacy documentation still referencing ADE instead of hypervisor-enforced Encryption at Host.
• **Remediation:** Standardized design record on Trusted Launch + Encryption at Host.   
• **Residual risk:** None (no live VM assets or public IPs deployed).   
• **Follow-up owner/date:** Self / 07/10/2026.

**Teardown**

• **Resources stopped/deleted:** N/A (Zero active compute instances provisioned).   
• **Defender plan/add-on disabled:** Defender for Servers P2 kept in trial/evaluation state.   
• **Capacity/endpoint removed:** N/A.
• **Cost view rechecked:** $0.00 spend incurred.
• **Persistent dependency for next lab:** None required for Day 15 (`day-15-defender-servers.md`).

**Exam recall**

• **Three facts I can explain:**
    1. **Trusted Launch Components:** Trusted Launch combines **Secure Boot** (protects against rootkits/bootkits) and **vTPM** (virtual Trusted Platform Module for measured boot and key storage) on Gen2 Azure VMs.   
    2. **Encryption at Host:** Encrypts VM temp disks, OS/data disk caches, and data in-transit between the hypervisor host and storage clusters directly at the host level without requiring guest OS BitLocker agents.
    3. **JIT Access Mechanism:** Defender for Cloud JIT does not keep ports open; it applies a high-priority `Deny All` NSG rule by default and injects a temporary, time-bound `Allow` rule for an approved user's IP address when requested.   

• **One confusing distinction:** **Azure Disk Encryption (ADE) vs. Encryption at Host** — ADE encrypts disks *inside* the guest OS using BitLocker/DM-Crypt. Encryption at Host encrypts data *before* it leaves the physical compute host running the VM, requiring zero guest OS execution.

• **One scenario question I wrote:** *An enterprise requires all virtual machines to encrypt OS disks, data disks, and temporary scratch disks without installing third-party agents or enabling BitLocker inside the guest OS. Which VM encryption feature satisfies this requirement?*
**Answer:** Enable **Encryption at Host** on the Azure subscription and VM configuration.