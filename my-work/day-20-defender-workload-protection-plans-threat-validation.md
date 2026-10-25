# Day 20 - Defender Workload Protection Plans & Threat Validation

Status: Done

### Session

- **Lab:** Defender Workload Protection Plans & Storage Malware Threat Testing (SC-500)
- **Date/time and time zone:** 08/10/2026 (AEDT)
- **Start/end time:** 17:15 - 18:15
- **Hands-on, trial-dependent, or simulated:** Resource-Scoped Storage Malware Test & Portal Sample Alert Validation

### Cost Checkpoint

- **Credit remaining:** $284
- **Month-to-date actual:** $0.00
- **Forecast:** $0.01 (Metered Storage Malware scanning executed on small test file)
- **Top services:** Defender for Storage (On-Upload Malware Scanning add-on)
- **Unexpected meters:** None.
- **Action:** Enabled Defender for Storage with On-Upload Malware Scanning strictly at **resource scope** on the lab storage account. Set a minimal monthly cap (e.g., 1 GB). Disabled the malware scanning extension immediately after EICAR test verification.

### Business Control

- **Requirement:** NCES needs a clear workload-protection matrix. Subscription-wide toggles must not be enabled blindly without understanding covered resources, add-on costs, evidence logging, and alert pathways.
- **Threat or failure mode:** Unmonitored malicious file uploads via web applications into cloud storage, unauthorized credential/secret exfiltration from Key Vault, SQL injection attacks, and blind spots across DNS/ARM control planes.
- **Control selected:** Selective enablement of Defender Workload Protection plans at resource or subscription scope combined with On-Upload Malware Scanning for Storage.
- **Why this service/control fits:** Intercepts runtime threats, file payloads, and control-plane anomalies in near-real-time and feeds normalized alerts directly to Microsoft Sentinel.
- **Rejected alternative:** Relying solely on basic Secure Score or network firewalls. Rejected because network firewalls cannot inspect encrypted blob payloads or distinguish valid REST API calls from compromised credentials inside Key Vault/ARM.

### Plan Inventory & Mapping Matrix

| Workload Plan | Primary Protected Resource | Key Add-ons / Components | Billing & Metering Model | NCES Status / Teardown Rule |
| --- | --- | --- | --- | --- |
| **Defender for Servers** | Azure & Arc VMs | P1 (MDE P2) / P2 (JIT, FIM, Agentless, 50MB/day log) | Per VM / hour | Retain P2 trial for Day 28 Sentinel logging integration. |
| **Defender for Storage** | Blob, ADLS Gen2, Files | On-upload Malware Scanning, Sensitive Data Discovery | Base plan per storage account + **metered per-GB scanning** | **Disable malware add-on immediately** after lab. |
| **Defender for Containers** | AKS, ACR, Container instances | Agentless posture + eBPF runtime sensor + MDVM scanning | Per vCPU core / hour | Active for Day 18-20; cleanup node resources. |
| **Defender for Key Vault** | Key Vault Secrets, Keys, Certs | Anomaly detection (Tor access, unexpected secret extraction) | Per 10k transactions | Active via trial; zero transaction cost on lab scale. |
| **Defender for SQL / Databases** | Azure SQL DB, Managed Instances, VMs | Vulnerability Assessment & Advanced Threat Protection | Per instance / server / month | Enabled at subscription or server level. |
| **Defender for Resource Manager** | ARM API layer (`management.azure.com`) | Detection of suspicious administrative actions / OS dumps | Per subscription / month | Active via trial. |
| **Defender for DNS** | VNet DNS Resolvers | Malicious domain resolution & DNS tunneling detection | Per million queries | Active via trial. |
| **Defender for App Service** | Web Apps, Function Apps | Web app vulnerability scanning & EICAR test endpoints | Per App Service Plan core / hour | Deferred unless active web apps exist. |
| **Defender for APIs** | API Management (APIM) | OWASP API Top 10, data exfiltration, token abuse | Per APIM instance / million calls | Enable in APIM setup. |
| **Defender for AI Services** | Azure OpenAI / Foundry endpoints | Prompt Injection, data exfiltration, jailbreak monitoring | Per 1k text/image transactions | **Enable on Days 21–26** once Foundry instance exists. |

### Implementation & Testing

#### Part A — Defender for Storage Real Malware Test

1. **Scope & Configuration:**
    - Enabled Defender for Storage at the **resource level** on storage account `stsc500day20cc`.
    - Enabled **On-Upload Malware Scanning** and set the monthly cap to `1 GB` to prevent runaway charges.
2. **EICAR Test Execution:**
    - Created a local text file containing the standard, harmless anti-malware test string:
    `X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*`
    - Uploaded `eicar.txt` into container `malware-test`.
3. **Observation & Results:**
    - **Blob Index Tag:** Verified within seconds that Azure applied the Blob Index Tag: `Malware scanning scan result` = `Malicious`.
    - **Alert Details:** Checked **Security Alerts** in Defender for Cloud. Received high-severity alert: *"Malicious file uploaded to storage account"*.
    - **Remediation:** Deleted the `eicar.txt` blob.
    - **Teardown:** Immediately navigated back to the storage account's Defender settings and **disabled On-Upload Malware Scanning** to stop additional meter charges.

#### Part B — Multi-Workload Sample Alert Validation

Generated sample alerts via Defender for Cloud > **Security Alerts** > **Sample Alerts**:

1. **Key Vault:** Alert generated: *"Access from a TOR exit node to a Key Vault"*.
    - *Evidence:* Maps to caller IP address, Tor exit node classification, targeted Key Vault, and specific secret accessed.
2. **Resource Manager:** Alert generated: *"Suspicious administrative operation detected"*.
    - *Evidence:* Maps to caller Service Principal / User Object ID, ARM API method (`Microsoft.Resources/deployments/write`), and source IP.
3. **Defender for SQL:** Alert generated: *"Potential SQL Injection attack"*.
    - *Evidence:* Maps to client app, database name, and parameterized query signature.

### Core Security Architectural Requirement

> **The Three-Layer Capability Model:**
> 
> 1. **Foundational CSPM:** Free posture hygiene. Identifies missing controls against the Microsoft Cloud Security Benchmark (MCSB). It **does not** perform threat detection or intercept attacks.
> 2. **Defender CSPM:** Advanced contextual posture. Provides agentless scanning, Cloud Security Explorer graph queries, and Attack Path Analysis. It prioritizes *how* misconfigurations can be exploited but **does not** stop active runtime attacks.
> 3. **Workload Protection (Defender Plans):** Real-time runtime threat protection. Uses agents, eBPF sensors, and API stream analyzers to generate **Security Alerts** when malicious activity occurs.
> 
> **Exam & Architectural Axiom:** Having a 100% Secure Score via CSPM does **not** protect against zero-day runtime malware uploads or stolen credentials. Conversely, turning on a Defender Workload plan does **not** automatically fix a publicly exposed storage account or an open NSG port.
> 

### Verification & Failure Analysis

| Test / Scenario | Expected Result | Actual Result | Pass/Fail | Evidence Reference |
| --- | --- | --- | --- | --- |
| **EICAR Scan Result** | Storage blob tagged as `Malicious` within seconds | Blob index tag set to `Malicious` | **PASS** | Storage Container Properties |
| **Alert Log Trace** | High-severity alert generated in Defender portal | Alert *"Malicious file uploaded..."* logged with full hash details | **PASS** | Defender Security Alerts Blade |
| **Sample Alert Generation** | Key Vault/ARM sample alerts populate in portal | Alerts created successfully and linked to Sentinel path | **PASS** | Sample Alert Verification |

### Teardown & Clean-Up

- **Storage Add-on Disabled:** Disabled Malware Scanning extension on `stsc500day20cc`.
- **Test Storage Account Deleted:** Deleted `stsc500day20cc` and its container `malware-test`.
- **Plan Review:** Retained core subscription plan trials for Days 21–29 Sentinel log ingestion validation.
- **Cost Check:** $0.00 compute spend; ~$0.01 micro-charge for EICAR blob scan.

### Exam Recall Summary (Day 20)

1. **Defender for Storage Billing Separation:** Defender for Storage base pricing covers transaction monitoring. **On-Upload Malware Scanning** and **Sensitive Data Discovery** are separate metered add-ons that bill based on GB scanned from day one, even during free trial windows.
2. **Malware Scanning Cap:** To prevent unexpected billing spikes from large file uploads, configure `CapGBPerMonthPerStorageAccount` in Defender for Storage settings.
3. **Blob Index Tagging:** When On-Upload Malware Scanning detects infected files, it writes a native Blob Index Tag (`Malware scanning scan result: Malicious`) directly to the object. Applications can read this tag to block user access or trigger automated deletion.
4. **CSPM vs. Workload Protection:** CSPM identifies misconfigurations (posture); Workload Protection detects active malicious behavior (runtime). Both are needed for comprehensive defense-in-depth.