# Day 15 - Defender for Servers, MDVM, Arc, and Machine Configuration

Status: Done

## 

### Session

- **Lab:** Microsoft Defender for Servers & Azure Arc Integration (SC-500)
    
    PNG
    
- **Date/time and time zone:** 07/10/2026 (AEDT)
- **Start/end time:** 20:15 - 20:45
- **Hands-on, trial-dependent, or simulated:** Architecture Decision Record & Portal Validation Workflow (Design-First / Simulated Generation Flow)
    
    PNG
    

### Cost Checkpoint

- **Credit remaining:** $284
- **Month-to-date actual:** $0.00
- **Forecast:** Nil
- **Top services:** N/A (Evaluated Defender for Servers P1/P2, Azure Arc, Azure Machine Configuration)
    
    PNG
    
- **Unexpected meters:** Nil
- **Action:** Avoided activating persistent Defender for Servers P2 paid billing ($15/server/month) and bypassed live Arc agent script execution on host machines; completed via portal validation, generation workflow analysis, and decision records.
    
    PNG
    

### Business Control

- **Requirement:** NCES operates Azure and on-premises servers and requires a single security posture/threat management view without assuming every server is cloud-native.
    
    PNG
    
- **Threat or failure mode:** Siloed security visibility across hybrid/on-prem infrastructure, unpatched host vulnerabilities, zero-day endpoint threats, and unmonitored critical OS file modifications.
    
    PNG
    
- **Control selected:** Defender for Servers Plan 2 combined with Azure Arc-enabled servers and Azure Machine Configuration.
    
    PNG
    
- **Why this service/control fits:** Azure Arc projects non-Azure machines into Azure Resource Manager, allowing Defender for Cloud to apply identical security baselines, agentless scanning, EDR (MDE), FIM, and JIT rules across all environments.
    
    PNG
    
- **Rejected alternative:** Native standalone Defender for Endpoint deployment without Azure Arc. Rejected because it lacks centralized Azure Policy enforcement, unified vulnerability management in Defender for Cloud, and Azure native governance.
    
    PNG
    

### Implementation & Decision Records

#### Part A — Defender for Servers Configuration Inspection

- **Plan Status Check:** Validated Defender for Servers Plan 2 trial status and resource coverage in Defender for Cloud > Environment Settings.
    
    PNG
    
- **Day 14 VM Inspection Details:**
    - *Defender for Endpoint (MDE):* Verified auto-provisioning status (`MDE.Linux` / `MDE.Windows` extension healthy).
        
        PNG
        
    - *Vulnerability Management:* Reviewed Microsoft Defender Vulnerability Management (MDVM) integration findings.
        
        PNG
        
    - *Agentless Scanning:* Validated snapshot-based agentless VM scanning status for zero-impact vulnerability/secret discovery.
        
        PNG
        
    - *Advanced Security:* Inspected Just-In-Time (JIT) VM access and File Integrity Monitoring (FIM) log analytics change tracking settings.
        
        PNG
        
- **Alert Validation:** Documented execution of Microsoft’s supported sample alert validation via the Defender for Cloud "Sample alerts" generator (avoiding actual malware execution).
    
    PNG
    

#### Part B — Plan 1 vs. Plan 2 Decision Record

| Feature / Capability | Defender for Servers Plan 1 | Defender for Servers Plan 2 |
| --- | --- | --- |
| **Endpoint Detection & Response (EDR)** | Included (Microsoft Defender for Endpoint P2)   PNG | Included (Microsoft Defender for Endpoint P2)   PNG |
| **Vulnerability Assessment** | MDVM core features   PNG | MDVM full features + Agentless scanning   PNG |
| **Just-In-Time (JIT) VM Access** | Not Included   PNG | **Included**
   PNG |
| **File Integrity Monitoring (FIM)** | Not Included   PNG | **Included**
   PNG |
| **Network Layer Threat Detection** | Not Included   PNG | **Included** (Agentless + Network inspection)   PNG |
| **Data Ingestion Inclusion** | None | **50 MB/day free data ingestion** per server   PNG |
| **Decision Verdict** | Rejected (Fails JIT & FIM requirements)   PNG | **Selected** (Required for compliance & advanced protection)   PNG |

#### Part C — Hybrid Lane (Azure Arc Onboarding Workflow)

- **Arc Onboarding Generation:** Walked through Azure Arc > **Servers** > **Add/Register**.
    
    PNG
    
- **Service Principal Setup:** Configured scoped Service Principal with `Azure Connected Machine Onboarding` role.
    
    PNG
    
- **Script Inspection:** Generated onboarding script (`OnboardingScript.ps1` / `.sh`) and audited outbound network requirements:
    
    PNG
    
    - Target Endpoints: `.his.arc.azure.com`, `.guestconfiguration.azure.com`, `.servicebus.windows.net` over TCP 443.
        
        PNG
        
- **Machine Configuration Assignment:** Evaluated assignment of Azure Machine Configuration baselines (e.g., *Insecure OS configurations baseline for Windows/Linux*).
    
    PNG
    

### Verification & Failure Analysis

| Test / Scenario | Expected Result | Actual Result | Pass/Fail | Evidence Reference |
| --- | --- | --- | --- | --- |
| **Missing Update Detection** | Defender highlights missing OS/software security updates | Recommendation flagged in Defender for Cloud Recommendations blade   PNG | **PASS** | Defender Recommendations Diagnostics   PNG |
| **Telemetry Trace Flow** | Vulnerability -> Affected Resource -> Remediation Owner -> Telemetry | Actionable path mapped from recommendation to Azure Policy / Arc agent telemetry   PNG | **PASS** | Governance & Policy Workflow   PNG |
| **Agentless vs. Agent EDR** | Agentless provides disk snapshot analysis; Agent provides real-time behavior blocking | Agentless scans offline disks for secrets/CVEs; MDE agent intercepts runtime execution | **PASS** | Defender Architecture Specs   PNG |

### Teardown & Clean-Up

- **Resources stopped/deleted:** Removed test onboarding Service Principal credentials and cleaned up any trial server assignments.
    
    PNG
    
- **VM Clean-up:** Verified `rg-sc500-vm-cc` (from Day 14) along with associated NICs and disks are deleted.
    
    PNG
    
- **Cost view rechecked:** $0.00 spend incurred.
    
    PNG
    

### Exam Recall Summary (Day 15)

1. **Plan 1 vs. Plan 2 Boundaries:** Plan 1 only provides MDE integration and basic vulnerability management. Advanced features like **JIT**, **FIM**, **Agentless scanning**, and **50 MB/day free ingestion** strictly require **Plan 2**.
    
    PNG+ 1
    
2. **Arc Onboarding Role:** To onboard servers to Azure Arc at scale using a Service Principal, assign the minimum required role: **`Azure Connected Machine Onboarding`**.
    
    PNG
    
3. **Agentless Scanning Engine:** Uses out-of-band cloud snapshots of VM disks to discover vulnerabilities and software inventories without consuming guest OS compute resources or requiring agent deployment.
    
    PNG
    
4. **Agent-Based EDR (MDE):** Required for real-time memory monitoring, process isolation, behavioral threat response, and active attack blocking that agentless snapshot scanning cannot perform in real time.
    
    PNG