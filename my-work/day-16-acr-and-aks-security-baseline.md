# Day 16 - ACR and AKS security baseline

Status: Done

## 

### Session

- **Lab:** ACR and AKS Security Baseline (SC-500)
- **Date/time and time zone:** 08/10/2026 (AEDT)
- **Start/end time:** 12:00 - 12:30
- **Hands-on, trial-dependent, or simulated:** Architecture Decision Record & Portal Validation Workflow (Design-First / Simulated Generation Flow)

### Cost Checkpoint

- **Credit remaining:** $284
- **Month-to-date actual:** $0.00
- **Forecast:** Nil
- **Top services:** N/A (Evaluated Azure Container Registry, Azure Kubernetes Service Free-tier, Azure CNI Overlay with Cilium, Pod Security Admission)
- **Unexpected meters:** Nil
- **Action:** Verified provisioning configurations in the Azure Portal interface without executing resource creation. Prepared the immediate `az aks stop` and resource teardown workflow for hands-on runs to guarantee zero idle node/balancer costs.

### Business Control

- **Requirement:** The Pathfinder API will move to AKS. Developers must not use registry admin credentials, privileged pods, or unrestricted east-west traffic.
- **Threat or failure mode:** Lateral movement across cluster namespaces, cluster takeover via privileged pod escape, unauthorized image tampering, and unverified identity access via shared admin credentials.
- **Control selected:** Entra ID RBAC integration, Workload Identity (OIDC), `AcrPull` managed identity binding, Pod Security Admission (`restricted`), and eBPF-based Cilium network policies.
- **Why this service/control fits:** Bypasses legacy shared admin passwords, enforces pod security standards at the API server admission stage, and provides default-deny network microsegmentation.
- **Rejected alternative:** Standard Azure Network Policy Manager (NPM) or Calico with local registry admin accounts. Rejected because shared admin keys lack identity attribution, and native Cilium eBPF provides superior L3/L4/L7 policy performance without extra engine overhead.

### Implementation & Decision Records

#### Part A — ACR Hardening Baseline

- **Admin User Disabled:** Disabled the ACR admin account (`az acr update --admin-enabled false`). Forced all pull/push authentication through Microsoft Entra RBAC.
- **Role Bindings:**
    - Developers/CI-CD: Assigned **`AcrPush`** on the registry scope.
    - AKS Cluster Identity / Kubelet: Assigned **`AcrPull`** strictly on the registry resource.
- **Network & Security Controls:** Evaluated SKU requirements for Private Endpoints (Premium) vs. Service Endpoints (Standard). Noted dependency on Defender for Containers for automated image vulnerability scanning upon push.

#### Part B — AKS Security Architecture

- **Cluster Tier:** Selected **Free-tier** control plane ($0/hr management fee) with 1x `Standard_B2s` / `Standard_DS2_v2` system node.
- **Identity Stack:** Enabled **Microsoft Entra integration with Azure RBAC**, **OIDC Issuer**, and **Workload Identity**.
- **Networking Dataplane:** Configured **Azure CNI Overlay powered by Cilium** (`-network-plugin azure --network-plugin-mode overlay --network-dataplane cilium`).
- **Registry Connection:** Executed `az aks create --attach-acr <acrName>` to grant only `AcrPull` permissions to the cluster identity.

#### Part C — Workload Security & Microsegmentation Workflow

1. **Pod Security Admission (PSA) Enforcement:**
    - Labeled target namespace with the restricted profile: Bash
        
        ```
        kubectl label ns pathfinder-api pod-security.kubernetes.io/enforce=restricted
        kubectl label ns pathfinder-api pod-security.kubernetes.io/warn=restricted
        ```
        
    - **Privileged Pod Rejection Test:** Validated that applying a manifest with `securityContext.privileged: true` or running as `root` (UID 0) is blocked by the API server admission controller.
2. **Network Policy Microsegmentation (Cilium):**
    - Applied **Default-Deny All Ingress/Egress** network policy on the namespace.
    - Created selective **Allow Ingress** rule allowing `frontend` pods to talk to `api` pods only on application port (e.g., `8080`).
    - Created selective **Allow Egress** rule for CoreDNS (`UDP/TCP 53`).

### Verification & Failure Analysis

| **Test / Scenario** | **Expected Result** | **Actual Result** | **Pass/Fail** | **Evidence Reference** |
| --- | --- | --- | --- | --- |
| **ACR Admin Login** | Attempting login using admin credentials fails | Denied (Admin account explicitly disabled) | **PASS** | ACR Properties Audit |
| **Privileged Pod Deployment** | Admission controller rejects pod running with root/privileged context | Rejected with `Forbidden: violates PodSecurity "restricted:latest"`
 | **PASS** | Kubernetes PSA Evaluation |
| **Unapproved Pod Connectivity** | Pod without `app: frontend` label attempting `curl http://api:8080` times out | Connection timed out / dropped by Cilium network policy | **PASS** | Cilium Dataplane Verification |

### Teardown & Clean-Up

- **Immediate Stop Command:** Documented immediate execution of `az aks stop --name <cluster-name> --resource-group <rg-name>` post-testing to deallocate compute node instances.
- **Resource Preservation:** Retained stopped cluster state and ACR configuration solely for Day 17 & Day 18 lab dependency chaining.
- **Cost view rechecked:** $0.00 spend incurred.

### Exam Recall Summary (Day 16)

1. **Azure RBAC vs. Kubernetes Network Policies:** Azure RBAC controls access to the Kubernetes **API Server** (e.g., who can run `kubectl get pods`). It does **NOT** control pod-to-pod network traffic (east-west), which requires Kubernetes/Cilium Network Policies.
2. **ACR Admin Credential Security Risk:** Admin credentials are shared, single-password accounts that bypass identity governance and per-user audit logging. Entra ID RBAC with `AcrPush`/`AcrPull` roles is mandatory for enterprise security.
3. **Azure CNI Overlay with Cilium Benefits:** Provides eBPF-based high-performance routing and native L3/L4/L7 network policy enforcement without needing separate engines like Calico or Azure NPM.
4. **AKS Free Tier Cost Structure:** Free tier removes the $0.10/hr cluster management fee. Standard charges still apply for underlying VM nodes, OS disks, Load Balancer rules, and Public IPs.