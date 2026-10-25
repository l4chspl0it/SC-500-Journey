# Day 18 - Defender for Containers

Status: Done

## 

### Session

- **Lab:** Defender for Containers & AKS Runtime Threat Protection (SC-500)
- **Date/time and time zone:** 08/10/2026 (AEDT)
- **Start/end time:** 14:15 - 15:00
- **Hands-on, trial-dependent, or simulated:** Defender Trial Enablement & Architecture Decision Record (Design-First / Cost-Burst Flow)

### Cost Checkpoint

- **Credit remaining:** $284
- **Month-to-date actual:** $0.00
- **Forecast:** Nil
- **Top services:** N/A (Evaluated Defender for Containers Plan, Defender Profile extension on AKS, agentless vulnerability assessment for ACR)
- **Unexpected meters:** Nil
- **Action:** Enabled Defender for Containers plan in trial mode ($0.00 during initial 30 days). Maintained AKS cluster in stopped state (`az aks stop`) during portal/policy validations to avoid compute charges.

### Business Control

- **Requirement:** NCES requires registry-to-runtime security visibility and contextual risk analysis, rather than relying solely on static CI/CD image scanning.
- **Threat or failure mode:** Zero-day runtime pod exploits, unauthorized registry image tampering, misconfigured cluster RBAC/network policies, and undetected container drift/escapes.
- **Control selected:** Defender for Containers plan providing agentless discovery + Defender sensor daemonset on AKS, MDVM (Microsoft Defender Vulnerability Management) agentless scanning for ACR, and Defender CSPM attack path analysis.
- **Why this service/control fits:** Fuses static registry image vulnerability scanning with real-time Kubernetes control-plane audit logs and in-cluster eBPF runtime telemetry.
- **Rejected alternative:** Standalone third-party container scanners (e.g., Trivy or Aqua) without Azure Defender integration. Rejected because standalone scanners lack native integration with Azure Cloud Security Explorer, unified XDR alert correlation, and agentless Azure resource dependency mapping.

### Implementation & Decision Records

#### Part A — Defender Plan & Extension Architecture

1. **Subscription Plan Status:**
    - Enabled **Defender for Containers** in Defender for Cloud > Environment Settings. Recorded trial status (30-day trial active, standard rate ~$7/vCPU/month or per-core billing thereafter).
2. **AKS Defender Sensor Verification:**
    - Enabled cluster Defender profile: `az aks update --enable-defender`.
    - Verified extension pods in `kube-system` / `gatekeeper-system` namespace:
        - `microsoft-defender-collector-ds-*` (Collects runtime inventory & telemetry).
        - `microsoft-defender-publisher-ds-*` (Publishes signals securely to Defender backend).
3. **Agentless Kubernetes Posture & ACR Assessment:**
    - Configured agentless discovery for Kubernetes (inspects control-plane audit logs and API server state).
    - Configured MDVM agentless image scanning on ACR. Confirmed scan triggers: **On-push**, **continuous assessment** (scans images running in the cluster within 24 hours), and **recent import**.

#### Part B — Alert Simulation & Attack Path Analysis

1. **Simulated Container Alert Execution:**
    - Generated test alert using Microsoft’s official mock command:Bash
        
        ```
        kubectl get pods --namespace=asc-alerttest-662jfi039n
        ```
        
    - *Alternative execution:* Deployed test shell in container and executed `cp /bin/echo ./asc_alerttest_662jfi039n` to trigger process modification/runtime alert.
2. **Security Posture & Attack Path Mapping:**
    - Inspected **Cloud Security Explorer** graph relationships: `Container Image` -> `Vulnerability (MDVM)` -> `Running Pod` -> `Internet Exposed Service (LoadBalancer)` -> `Kubernetes Cluster`.
    - Documented remediation of a misconfiguration (e.g., applying `readOnlyRootFilesystem` or blocking root context) and noted latency (typically **15 to 60 minutes** for posture recommendations to refresh, up to 24 hours for full vulnerability rescan updates).

### Comparison: Security Control Layers

| **Capability** | **Scope / Engine** | **Execution Point** | **Primary Threat Addressed** |
| --- | --- | --- | --- |
| **Admission Prevention (PSA / Gatekeeper)** | API Server Admission | **Pre-deployment** (In-line block) | Prevents privileged pods, host path mounts, or root execution from being scheduled. |
| **Posture Assessment (Defender CSPM)** | Control Plane / Audit Logs | **Continuous / Agentless** | Identifies missing network policies, exposed dashboards, and loose K8s RBAC rules. |
| **Registry Vulnerability Assessment (MDVM)** | ACR Storage Engine | **On-Push / Continuous** | Detects CVEs, outdated packages, and plain-text secrets embedded in Docker layers. |
| **Runtime Threat Detection (Defender Sensor)** | Host eBPF / DaemonSet | **Real-time Runtime** | Intercepts crypto-miners, reverse shells, privilege escalation attempts, and binary drift. |

### Verification & Failure Analysis

| **Test / Scenario** | **Expected Result** | **Actual Result** | **Pass/Fail** | **Evidence Reference** |
| --- | --- | --- | --- | --- |
| **Defender Sensor Health** | `microsoft-defender-collector` pods reach `Running` state on all nodes | Verified daemonset pods running without `CrashLoopBackOff` | **PASS** | `kubectl get pods -n kube-system` Audit |
| **Mock Alert Trigger** | `asc-alerttest` command generates a high-severity test alert in Defender for Cloud | Alert flagged: *"Microsoft Defender for Containers test alert"* in portal | **PASS** | Defender Security Alerts Blade |
| **Posture Latency Check** | Pod security posture remediation updates in security recommendations | Recommendation marked as `healthy` after policy re-evaluation window | **PASS** | Posture Assessment Audit Log |

### Teardown & Clean-Up

- **Complete Teardown Executed:**
    - Deleted AKS & ACR Resource Group (`az group delete --name rg-sc500-aks-cc --yes --no-wait`).
    - Confirmed associated Node Resource Group (`MC_rg-sc500-aks-cc_*`), Managed Disks, Load Balancers, Public IPs, Managed Identities, and Role Assignments were removed.
- **Defender Plan State:** Retained Defender for Containers plan active on subscription for Day 19 (Defender CSPM) and Day 20 review, avoiding re-enablement overhead.
- **Cost view rechecked:** $0.00 spend incurred.

### Exam Recall Summary (Day 18)

1. **Sensor vs. Agentless Dual Architecture:** Defender for Containers uses **agentless scanning** for control-plane configuration and registry images, combined with a lightweight **in-cluster sensor (DaemonSet)** for eBPF runtime process inspection.
2. **ACR Scan Triggers:** Images in Azure Container Registry are scanned automatically **on push**, during **continuous assessment** (if running on an active AKS cluster), and when **imported**.
3. **Runtime Alerting Mechanism:** Runtime threat detection catches live attacks (like shell execution inside a web container) that static image scanning and admission policies cannot detect prior to execution.
4. **Clean-up Boundary:** Deleting an AKS cluster deletes the node infrastructure, but **does not disable** the Defender for Containers plan at the subscription level. Subscription plan settings must be modified separately under Defender for Cloud pricing settings.