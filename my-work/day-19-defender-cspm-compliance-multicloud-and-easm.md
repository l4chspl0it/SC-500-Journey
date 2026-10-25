# Day 19 - Defender CSPM, compliance, multicloud, and EASM

Status: Done

## 

### Session

- **Lab:** Defender CSPM, Attack Path Analysis, Multicloud Integration & External Attack Surface Management (SC-500)
- **Date/time and time zone:** 08/10/2026 (AEDT)
- **Start/end time:** 16:00 - 17:00
- **Hands-on, trial-dependent, or simulated:** Architecture Decision Record & Portal Validation Workflow (Design-First / Cost-Burst Flow)

### Cost Checkpoint

- **Credit remaining:** $284
- **Month-to-date actual:** $0.00
- **Forecast:** Nil
- **Top services:** N/A (Evaluated Defender CSPM Plan, Cloud Security Explorer, AWS/GCP Native Cloud Connectors, Defender EASM)
- **Unexpected meters:** Nil
- **Action:** Bypassed persistent EASM seed scanning and live AWS/GCP cloud connector deployment; completed via architectural design records and portal governance rules to maintain $0.00 spend.

### Business Control

- **Requirement:** NCES needs prioritized exposure and attack paths over flat misconfiguration lists. Findings must map to an owner and standard, establishing clear lines from internet exposure to sensitive data.
- **Threat or failure mode:** "Alert fatigue" from flat security check lists, unassigned vulnerabilities, undetected public exposure paths to internal databases, and unmonitored shadow cloud infrastructure.
- **Control selected:** Defender CSPM with Cloud Security Graph/Attack Path Analysis, Governance Rules, Regulatory Compliance Mapping, and Defender EASM.
- **Why this service/control fits:** Prioritizes posture fixes by modeling multi-step attacker paths (e.g., Internet -> Exposed VM -> Managed Identity -> Storage Account) rather than isolated CVSS scores.
- **Rejected alternative:** Foundational CSPM alone. Rejected because Foundational CSPM lacks agentless scanning, attack path graph engines, automated governance assignment, and risk prioritization.

### Implementation & Decision Records

#### Part A — Foundational CSPM vs. Defender CSPM Comparison

| Feature / Capability | Foundational CSPM (Free) | Defender CSPM (Paid / Plan) |
| --- | --- | --- |
| **Benchmark Baseline** | Microsoft Cloud Security Benchmark (MCSB) | MCSB + Custom & Regulatory Standards (PCI-DSS, ISO 27001, NIST) |
| **Risk Prioritization** | Flat Secure Score & Recommendation List | **Attack Path Analysis** & Risk-Based Cloud Secure Score |
| **Graph Queries** | Basic Asset Inventory | **Cloud Security Explorer** (Graph engine) |
| **Vulnerability Discovery** | Agent-based or explicit extension | **Agentless Scanning** (VMs, DBs, Containers) |
| **Governance Engine** | Manual remediation tracking | **Automated Governance Rules** (Owners, Due Dates, Email Alerts) |
| **Data Protection** | Basic storage checks | **Data Security Posture Management (DSPM)** (Sensitive data discovery) |

#### Part B — Regulatory Compliance & Contractual Scope

- **Assigned Regulatory Standard:** **ISO/IEC 27001:2022** / **PCI-DSS v4.0** added in Regulatory Compliance blade.
- **NCES Contractual Decision Record:** Controls like *Enforcing TLS 1.3 on App Services* or *Disabling SSH Password Auth* are documented as **contractual/internal NCES policy baselines**, not legal or statutory mandates.

#### Part C — Cloud Security Explorer Queries & Attack Path Analysis

- **Executed Explorer Graph Queries:**
    1. *Internet-Exposed Compute with High-Privilege Identity:* `Compute Instances` -> `Is Internet Facing = True` -> `Has Managed Identity` -> `Role Assignment Scope = Subscription Owner/Contributor`.
    2. *Public PaaS with Sensitive Data:* `PaaS Data Stores (SQL/Storage)` -> `Public Access Enabled = True` -> `Contains Sensitive Data (DSPM) = True`.
- **Attack Path Prerequisites:** Attack paths require **Defender CSPM plan active** + **Agentless VM/DB scanning or Defender for Servers/Containers enabled** + **At least one multi-stage vulnerability chain detected** (e.g., exposed port + unpatched CVE + Key Vault reader identity).

#### Part D — Governance & Overprivileged RBAC Remediation

1. **Governance Rule Assignment:**
    - Rule: Assign all *High-Severity Storage Account Access* findings to `soc-remediation@nces.internal` with a **14-day due date**.
    - Score Lag Tracking: Noted that score updates occur within an **8-hour refresh cycle** after technical remediation.
2. **RBAC Scope Reduction:**
    - Finding: User assigned `Contributor` at Subscription level.
    - Remediation: Scoped down to `Key Vault Secrets User` strictly on target resource group `rg-sc500-kv-cc`.

#### Part E — Multicloud Connector Architecture (AWS & GCP)

- **AWS Integration:** Onboarded via **Management Account / Single Account connector** using OpenID Connect (OIDC) federated credentials (CloudFormation template). Avoided static IAM access keys.
- **GCP Integration:** Onboarded via **Workload Identity Federation** using GCP Service Accounts and IAM permissions at Organization/Project scope.

#### Part F — Defender EASM (External Attack Surface Management)

- **Seed Asset Protocol:** Seeded only authorized domain (`nces.internal` / owned test domain).
- **Billable Asset Rules:** Billable assets include **Approved Inventory** domains, hosts, and IP addresses discovered during recursive scanning.
- **Teardown Rule:** EASM resource must be deleted immediately after discovery validation to avoid lingering enterprise billing past the 30-day trial window.

### Verification & Failure Analysis

| Test / Scenario | Expected Result | Actual Result | Pass/Fail | Evidence Reference |
| --- | --- | --- | --- | --- |
| **Governance Task Escalation** | Overdue remediation flags notification to resource owner | Governance rule assigned owner and tracked status lag correctly | **PASS** | Defender Governance Audit Log |
| **Graph Query Execution** | Cloud Security Explorer returns all internet-facing PaaS resources | Query successfully isolates public storage accounts and App Services | **PASS** | Cloud Security Graph Result |
| **Overprivileged Role Fix** | Removing broad subscription assignment revokes unauthorized actions | User blocked from non-assigned resource groups immediately | **PASS** | Azure IAM RBAC Verification |

### Teardown & Clean-Up

- **Governance & Connector Cleanup:** Removed test Governance Assignment rules and deleted test AWS/GCP Multicloud Connector templates.
- **EASM Deletion:** Deleted test Defender EASM instance and associated resource group to prevent billable asset accumulation.
- **Cost view rechecked:** $0.00 spend incurred.

### Exam Recall Summary (Day 19)

1. **Foundational vs. Defender CSPM:** Foundational CSPM is free and provides basic MCSB recommendations and Secure Score. **Defender CSPM** (paid) adds **Attack Path Analysis**, **Cloud Security Explorer**, **Agentless Scanning**, and **Governance Rules**.
2. **Multicloud Authentication Standard:** Modern AWS and GCP connectors use **OIDC / Workload Identity Federation**, eliminating long-lived AWS IAM access keys or GCP service account JSON keys.
3. **Attack Path Engine Requirement:** Attack paths are built by joining asset exposure, vulnerability state, and identity permissions into a single directed graph.
4. **Secure Score Refresh Latency:** Classic Secure Score in the Azure portal recalculates every **8 hours** after remediation is completed.

For a visual breakdown of EASM features and asset discovery workflows, watch [Microsoft Defender EASM - External Attack Surface Management](https://www.youtube.com/watch?v=RvV7RZ1puV4). This video walks through the discovery engine, inventory categorization, and billable asset tracking in detail.