# Day 25 - AI Gateway with Azure API Management

Status: Done

Session

Lab: Day 25 — AI Gateway with Azure API Management

Date/time and time zone: October 8, 2026, 8:58 PM AEDT

Start/end time: 19:00 - 20:45 AEDT

Hands-on, trial-dependent, or simulated: Hands-on Live Execution (Azure AI Foundry Portal `ai.azure.com` & Azure APIM Management Plane)

Microsoft documentation checked on: October 8, 2026

Region/SKU used: East US 2 / Basic v2 SKU (Foundry AI Gateway integration) / `gpt-4o-mini`

Cost checkpoint

Credit remaining: $283.90 AUD

Month-to-date actual: $0.10 AUD

Forecast: ~$0.20 AUD

Top services: Azure API Management (Basic v2 - Foundry Free Tier promotion), Microsoft AI Foundry, Azure Log Analytics

Unexpected meters: None

Action: Initialized APIM Basic v2 via Foundry **Manage > AI Gateway** wizard; stayed within the eligible 100,000 free request threshold to prevent SKU base compute billing.

Business control

Requirement: Applications must not call model endpoints directly. Centralize Microsoft Entra ID caller authentication, token rate limits (TPM), total usage quotas, Content Safety policies, and request logging at the edge.

Threat or failure mode: Unrestricted direct model access leading to credential leakage, runaway token costs, prompt injection bypass, and denial of service via API flooding.

Control selected: Azure API Management AI Gateway (Basic v2 via Foundry) with System-Assigned Managed Identity, Cognitive Services OpenAI User RBAC, `<azure-openai-token-limit>`, and Azure AI Content Safety policy integration.

Why this service/control fits: Provides native LLM traffic management and Entra ID JWT validation at the gateway, completely isolating backend AI Foundry resources from direct client access.

Rejected alternative and reason: APIM Consumption tier (lacks native LLM token management XML policy schemas) and standalone APIM AI Gateway Public Preview SKU (limited regional availability and separate pending pricing model).

Implementation

Resources changed: Created APIM instance (`apim-nces-gateway-dev`) via Foundry portal, attached system-assigned identity, imported model API endpoint, applied XML policies, and linked to project `case-agent-purple`.

Identity and role/scope: Assigned `Cognitive Services OpenAI User` role to `apim-nces-gateway-dev` system-assigned managed identity scoped to the AI Foundry hub resource.

Management-plane controls: Added Gateway in Foundry under **Manage > AI Gateway**. Set up diagnostic settings to stream APIM logs to Log Analytics Workspace (`law-nces-security`).

Data-plane controls: Configured inbound XML policies enforcing Entra JWT validation (`validate-azure-ad-token`), token rate limits (`azure-openai-token-limit` set to 500 TPM limit), total token quota (1,000 tokens), request size restrictions, and Content Safety checks.

Network controls: Direct application endpoint URLs barricaded; traffic strictly routed through `[https://apim-nces-gateway-dev.azure-api.net/](https://apim-nces-gateway-dev.azure-api.net/)`.

Monitoring destination: `ApiManagementGatewayLogs` table in Azure Log Analytics.

Verification

| **Test** | **Expected** | **Actual** | **Pass/fail** | **Evidence reference** |
| --- | --- | --- | --- | --- |
| Allowed path | Authorized Entra JWT token request below token ceiling | `200 OK` response with completion output and returned token header metrics | Pass | Test Console / HTTP 200 |
| Rate Limit Breach | High-volume multi-turn prompts exceeding 500 TPM | Request blocked with `429 Too Many Requests` | Pass | HTTP 429 Rate Exceeded |
| Quota Exceeded | Cumulative prompt executions exceeding 1,000 total token limit | Gateway drops execution with `403 Forbidden` | Pass | HTTP 403 Quota Limit |
| Logging/detection | Log Analytics query captures operational metrics | KQL query returns timestamped log entries with status codes 200, 429, and 403 | Pass | KQL Log Trace |

Code snippet

```
ApiManagementGatewayLogs
| where TimeGenerated > ago(1h)
| project TimeGenerated, OperationName, ResponseCode, TotalTime
```

Findings

Misconfiguration discovered: Applications could initially bypass the gateway and invoke the Azure AI Foundry target directly using key-based authentication.

Root cause: Access keys remained enabled on the underlying AI Services resource alongside loose network access rules.

Remediation: Disabled local key authentication on the target AI service (`disableLocalAuth: true`), strictly requiring Azure AD / Managed Identity tokens, and modified network ACLs to allow requests strictly from APIM's public IP / Subnet.

Residual risk: Preview features in APIM v2 Basic integrated via Foundry may undergo policy schema updates before General Availability (GA).

Follow-up owner/date: Security Architect / October 15, 2026.

Teardown

Resources stopped/deleted: APIM instance left intact for integration suite; rate limit policies retained.

Defender plan/add-on disabled: None.

Capacity/endpoint removed: None.

Cost view rechecked: $0.10 AUD actual spend verified.

Persistent dependency for next lab: APIM Gateway endpoint URL and managed identity bindings preserved for Day 30 cleanup.

Exam recall

Three facts I can explain:

1. Creating an AI Gateway directly through the Microsoft AI Foundry workflow provisions an eligible APIM Basic v2 instance with free-tier request limits (up to 100,000 calls).
2. When APIM handles AI Foundry model backends, a system-assigned managed identity must be granted the `Cognitive Services OpenAI User` RBAC role on the target resource to allow keyless data-plane inference.
3. Exceeding a configured `<azure-openai-token-limit>` TPM threshold in APIM triggers an HTTP `429 Too Many Requests`, whereas breaching total token quotas triggers an HTTP `403 Forbidden`.

One confusing distinction:

- **APIM via Foundry (Basic v2) vs. Standalone APIM AI Gateway Tier:** APIM created via the Foundry workflow uses standard v2 architecture with built-in LLM policies under an eligible promotional tier, whereas the standalone APIM "AI Gateway Tier" is a separate public preview offering with distinct SKU boundaries and regional limitations.

One scenario question I wrote:

- *An organization provisions Azure API Management (APIM) through Microsoft AI Foundry to enforce security and rate limits on Azure OpenAI endpoints. Which response code should a client application expect if it exceeds the configured Tokens-Per-Minute (TPM) limit set on the APIM gateway policy?*
    - A) `401 Unauthorized`
    - B) `403 Forbidden`
    - C) **`429 Too Many Requests`** *(Correct)*
    - D) `503 Service Unavailable`