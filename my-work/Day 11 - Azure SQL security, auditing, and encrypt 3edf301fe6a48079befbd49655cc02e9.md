# Day 11 - Azure SQL security, auditing, and encryption

Status: Done

## 

### Session

- **Lab:** Azure SQL PaaS Security Hardening & Data Protection (SC-500)
- **Date/time and time zone:** 07/10/2026 (AEDT)
- **Start/end time:** 15:15 - 15:55
- **Hands-on, trial-dependent, or simulated:** Hands-on

### Cost checkpoint

- **Credit remaining:** $284
- **Month-to-date actual:** < $1.00
- **Forecast:** Nil
- **Top services:** Azure SQL Logical Server, Azure SQL Database (Serverless / Basic tier)
- **Unexpected meters:** Nil
- **Action:** Retain SQL Server and Database for upcoming network lockdown (Private Endpoint) and auditing modules.

### Business control

- **Requirement:** Enforce strict identity-based database access, eliminate local SQL passwords, implement least-privilege database roles, and prepare for network-level isolation.
- **Threat or failure mode:** Credential spraying/brute-forcing of local SQL admin accounts, unauthorized data exfiltration via over-privileged database users, and unencrypted transport/storage exposure.
- **Control selected:** Microsoft Entra-only authentication, contained database users (`FROM EXTERNAL PROVIDER`), and default Transparent Data Encryption (TDE) with TLS 1.2+ enforcement.
- **Why this service/control fits:** Entra-only authentication disables local SQL passwords at the logical server control plane, eliminating SQL credential management and centralizing identity/access management (IAM) under Entra ID conditional access and RBAC.
- **Rejected alternative and reason:** Mixed-mode (SQL + Entra) authentication. Maintaining local SQL logins introduces persistent credential management overhead, risks credential exposure, and bypasses Entra ID MFA/Conditional Access policies.

### Implementation

- **Resources changed:**
    - Logical Server: `sql-sc500-cca` (Location: Canada Central, Entra-only auth enforced)
    - Database: `sqldb-sc500-test` (Serverless / low-cost tier)
    - Database Schema/Table: `dbo.SecurityIncidentCases` (Populated with 10 synthetic case rows)
- **Identity and role/scope:**
    - Entra Admin: User account assigned as Microsoft Entra admin at Logical Server scope.
    - Contained Users: `SecOps-Data-Readers` and `logic-sc500-test` mapped via `CREATE USER ... FROM EXTERNAL PROVIDER`.
    - Database Roles: Granted `db_datareader` (least privilege) on `sqldb-sc500-test`.
- **Management-plane controls:** Enforced Entra-only authentication policy on `sql-sc500-cca`.
- **Data-plane controls:** Contained database users isolated to `sqldb-sc500-test` with zero server-level admin privileges.
- **Network controls:** Default public endpoint temporarily active for Query Editor setup; scheduled for Private Endpoint (`privatelink.database.windows.net`) isolation in Step 7.
- **Monitoring destination:** SQL Auditing to Log Analytics queued for Step 6 configuration.

### Verification

| Test | Expected | Actual | Pass/fail | Evidence reference |
| --- | --- | --- | --- | --- |
| **Entra-Only Auth Enforcement** | Local SQL password login attempts fail at control plane | SQL authentication disabled; connection rejected for local logins | **PASS** | Azure SQL Logical Server Entra ID Blade |
| **Contained User Provisioning** | T-SQL creates contained users mapped to Entra principals | `CREATE USER ... FROM EXTERNAL PROVIDER` executed successfully | **PASS** | T-SQL Query Execution in Query Editor |
| **Least-Privilege Role Assignment** | Users granted `db_datareader` can query but not modify tables | Database role `db_datareader` assigned to contained users | **PASS** | T-SQL `ALTER ROLE db_datareader ADD MEMBER` |
| **Synthetic Data Ingestion** | Table created and 10 non-PII synthetic rows inserted | `SELECT COUNT(*)` returns exactly 10 rows | **PASS** | `dbo.SecurityIncidentCases` Query Output |

### Findings

- **Misconfiguration / Confusion discovered:**
    1. Azure SQL deployment tiers can be confusing in the portal—Azure SQL Logical Server is a free administrative container, whereas compute costs stem solely from the hosted database tier (Serverless/Provisioned).
    2. Local SQL logins are disabled tenant-wide at the server scope once Entra-only authentication is toggled on, preventing fallback SQL admin logins.
- **Root cause:** Native design of Azure SQL control plane enforcing single-identity authentication paradigms.
- **Remediation:** Validated Entra ID passwordless / token-based authentication via Query Editor.
- **Residual risk:** SQL Logical Server public endpoint remains accessible until Private Endpoint and firewall deny rules are applied in Step 7.
- **Follow-up owner/date:** Self / 07/10/2026.

### Teardown

- **Resources stopped/deleted:** None required; serverless database will auto-pause after 1 hour of inactivity.
- **Defender plan/add-on disabled:** Defender for SQL kept in trial/staged status for Day 20 per plan.
- **Capacity/endpoint removed:** None.
- **Cost view rechecked:** $0.00 additional run costs incurred.
- **Persistent dependency for next lab:** Retain `sql-sc500-cca` and `sqldb-sc500-test` for TLS/TDE verification (Step 5), SQL Auditing (Step 6), and Private Endpoint deployment (Step 7).

### Exam recall

- **Three facts I can explain:**
    1. **Entra-Only Authentication:** Toggling Entra-only authentication on an Azure SQL Logical Server disables all local SQL authentication logins (including `sa`), requiring all database users and applications to authenticate via Microsoft Entra ID.
    2. **Contained Database Users:** Creating users with `CREATE USER [principal] FROM EXTERNAL PROVIDER` creates a contained database user in the target database without creating a server-level login, supporting least-privilege security boundaries.
    3. **Auto-Pause in Serverless:** Azure SQL Serverless compute automatically pauses the database when inactive for the configured delay period, pausing compute charges entirely while retaining storage.
- **One confusing distinction:** **Server-level Logins vs. Contained Database Users** — Server-level logins exist at the logical server master database level and can potentially access multiple databases. Contained database users exist strictly within a single user database, reducing cross-database lateral movement risks.
- **One scenario question I wrote:** *An organization needs to grant a third-party audit team read-only access to an Azure SQL Database while ensuring they cannot access any other database on the logical server and without creating local SQL credentials. What is the most secure implementation?*
    
    **Answer:** Provision a Microsoft Entra group for the audit team, connect to the database as the Entra Admin, create a contained user mapped to the Entra group using `CREATE USER [Group-Name] FROM EXTERNAL PROVIDER`, and add the user to the `db_datareader` database role.