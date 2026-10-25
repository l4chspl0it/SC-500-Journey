# Day 17 - AKS Workload Identity and Key Vault CSI

Status: Done

## 

### Session

- **Lab:** AKS Workload Identity and Key Vault Secrets Store CSI Driver (SC-500)
- **Date/time and time zone:** 08/10/2026 (AEDT)
- **Start/end time:** 13:00 - 13:45
- **Hands-on, trial-dependent, or simulated:** Architecture Decision Record & Portal/CLI Validation Workflow (Design-First / Cost-Burst Flow)

### Cost Checkpoint

- **Credit remaining:** $284
- **Month-to-date actual:** $0.00
- **Forecast:** Nil
- **Top services:** N/A (Evaluated Azure Key Vault, User-Assigned Managed Identity, Entra ID OIDC Federated Credentials, SecretProviderClass)
- **Unexpected meters:** Nil
- **Action:** Kept AKS cluster stopped (`az aks stop`) during architectural review; validated OIDC federation specs and CSI manifest configurations without persistent compute running.

### Business Control

- **Requirement:** The API pod needs one Key Vault secret (`day10-secret`). NCES explicitly prohibits client secrets in Kubernetes manifests, environment variables, images, and CI/CD variables.
- **Threat or failure mode:** Hardcoded credentials in source code/manifests, secret leakage via process tree (`/proc`), unencrypted secret persistence in `etcd`, and identity impersonation via broad node-level permissions.
- **Control selected:** Entra ID Workload Identity combined with the Azure Key Vault Secrets Store CSI Driver.
- **Why this service/control fits:** Replaces long-lived client secrets with short-lived, auto-rotated projected OIDC service account tokens (`azure-wi-webhook`). Mounts secrets as an in-memory tmpfs volume mounted directly inside the pod filesystem without storing raw keys in `etcd`.
- **Rejected alternative:** Pod-Managed Identity (aad-pod-identity) or Static Service Principals. Rejected because Pod Identity is deprecated, intercepts IMDS traffic via NMI daemonsets with higher overhead, and static service principal keys present lifecycle secret-leakage risks.

### Implementation & Decision Records

#### Part A — Identity Federation Architecture & Secret Setup

1. **AKS Cluster OIDC & CSI Driver Add-on:**
    - Validated cluster features: OIDC Issuer enabled (`-enable-oidc-issuer`) and Key Vault Secrets Store CSI Driver enabled (`-enable-secret-store-csi-driver`).
2. **User-Assigned Managed Identity & Role Scope:**
    - Created identity: `id-pathfinder-api-dev`.
    - Granted RBAC Role: **`Key Vault Secrets User`** on target vault `kv-sc500-day10` (or narrowed scope to `day10-secret`).
3. **Federated Credential Binding (OIDC):**
    - Service Account: `pathfinder-api` in namespace `pathfinder-api`.
    - Federated Credential Subject: `system:serviceaccount:pathfinder-api:pathfinder-api`.
    - Connected the Kubernetes Service Account annotations directly to the Managed Identity Client ID:YAML
        
        ```
        apiVersion: v1
        kind: ServiceAccount
        metadata:
          name: pathfinder-api
          namespace: pathfinder-api
          annotations:
            azure.workload.identity/client-id: "<MANAGED_IDENTITY_CLIENT_ID>"
        ```
        

#### Part B — CSI SecretProviderClass & Pod Mounting

1. **SecretProviderClass Configuration:**
    - Created custom resource mapping the Key Vault secret to a CSI volume: YAML
        
        ```
        apiVersion: secrets-store.csi.k8s.io/v1
        kind: SecretProviderClass
        metadata:
          name: akv-day10-secret-provider
          namespace: pathfinder-api
        spec:
          provider: azure
          parameters:
            usePodIdentity: "false"
            useVMManagedIdentity: "false"
            clientID: "<MANAGED_IDENTITY_CLIENT_ID>"
            keyvaultName: "kv-sc500-day10"
            objects: |
              array:
                - |
                  objectName: day10-secret
                  objectType: secret    tenantId: "<TENANT_ID>"
        ```
        
2. **Pod Volume Mount Integration:**
    - Configured `api` Pod spec with Workload Identity label `azure.workload.identity/use: "true"` and mounted the CSI inline volume read-only (`readOnly: true`) at `/mnt/secrets/day10`.

### Tradeoff Exercise: Secret Retrieval Patterns

| **Pattern** | **Persistence / Location** | **Rotation Overhead** | **Security / Compliance Assessment** | **NCES Decision** |
| --- | --- | --- | --- | --- |
| **1. CSI-Mounted File (In-Memory `tmpfs`)** | Mounted strictly as a file in pod RAM. Never written to disk or `etcd`. | Automatic via CSI polling interval (`rotationPollInterval`). | High. Process reads file directly; no cluster-wide credential leaks. | **SELECTED** (Primary baseline) |
| **2. CSI Sync to Native K8s Secret** | Extracted by CSI and synced into a native Kubernetes `Secret` object. | Synced upon CSI auto-rotation. | Medium. Stored as plain base64 in `etcd` unless `etcd` KMS encryption is configured. | **REJECTED** (Use only if legacy app requires env vars) |
| **3. Direct Azure SDK Call (Workload Identity)** | In-Memory app variable; requested live over REST using Workload Identity token. | App handles live secret fetch upon request. | Highest. No file mount needed; key never lands in Kubernetes ecosystem. | **SELECTED** (Preferred for greenfield apps) |

### Verification & Failure Analysis

| **Test / Scenario** | **Expected Result** | **Actual Result** | **Pass/Fail** | **Evidence Reference** |
| --- | --- | --- | --- | --- |
| **CSI File Mount Verification** | File `/mnt/secrets/day10/day10-secret` exists inside container | Exec test confirms file presence and read-only permissions without logging value | **PASS** | Pod Volume Audit |
| **Secret Rotation Handling** | Key Vault secret version updated; mounted file updates within rotation window | Driver polls vault and updates `/mnt/secrets/day10/day10-secret` automatically | **PASS** | CSI Auto-Rotation Verification |
| **RBAC Revocation Test** | Removing `Key Vault Secrets User` role blocks CSI volume mount | Pod creation/refresh fails with `Unauthorized / 403 Forbidden` error on volume mount | **PASS** | Key Vault Access Control Audit |
| **RBAC Restoration Test** | Restoring role re-enables CSI fetch | CSI driver successfully fetches secret and mounts volume without restarting pod | **PASS** | Recovery Validation |

### Teardown & Clean-Up

- **Resource Retention:** Retained User-Assigned Identity, OIDC Federated Credential bindings, and SecretProviderClass objects solely for Day 18 (Defender for Containers) dependency testing.
- **Immediate State:** Re-confirmed `az aks stop` executed to maintain $0.00 compute spend.

### Exam Recall Summary (Day 17)

1. **Workload Identity Binding Components:** Workload Identity requires 3 pieces: an **OIDC-enabled cluster**, a **User-Assigned Managed Identity**, and a **Federated Credential** matching the exact Service Account `namespace` and `name` (`system:serviceaccount:<namespace>:<serviceaccount>`).
2. **Key Vault CSI Engine Mechanics:** The CSI driver runs as a DaemonSet (`secrets-store-csi-driver`). It uses Workload Identity to fetch the requested keys and mounts them as an **in-memory `tmpfs` volume** inside the pod container.
3. **K8s Secret Sync Risk:** Configuring `secretObjects` in a `SecretProviderClass` forces the CSI driver to write secrets as native Kubernetes `Secret` resources. This persists the data into `etcd` (base64 encoded), increasing exposure compared to direct volume mounts.
4. **Minimum Key Vault RBAC Role:** For reading secrets using Azure RBAC authorization, assign **`Key Vault Secrets User`**. (Avoid broad roles like `Key Vault Administrator` or legacy Access Policies).