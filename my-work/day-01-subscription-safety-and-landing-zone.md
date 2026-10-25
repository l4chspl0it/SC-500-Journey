# Day 1 - Subscription safety and landing zone

Date Completed: 02/10/26
Status: Done

## Session

- Lab: Subscription Safety
- Date/time and time zone: 02/10/2026 UTC +10
- Start/end time: Around 2pm
- Hands-on, trial-dependent, or simulated: Hands-on
- Microsoft documentation checked on: Nil
- Region/SKU used: N/A

## Cost checkpoint

- Credit remaining: $200
- Month-to-date actual: N/A
- Forecast: N/A
- Top services: Cost alerts (Billing) and Resource Groups and Tags
- Unexpected meters: N/A
- Action: N/A

## Business control

- Requirement: Ensures budgets are not exceeded. Tag all resource groups with Delete value
- Threat or failure mode: N/A
- Control selected: Cost alerts
- Why this service/control fits: N/A
- Rejected alternative and reason: N/A

## Implementation

- Resources changed: Ran [bootstrap.sh](http://bootstrap.sh) file to auto create resource groups
- Identity and role/scope: Azure Subscription 1 scope renamed to sc-500
- Management-plane controls:
    - Added Cost alerts for x3 budgets and added a daily cost email
    - Registered resource providers
        - Also learn a quicker way to do this via AZ CLI using:
        
        ```jsx
        az provider register --namespace Microsoft.Security
        ```
        
    - Set up Microsoft Foundry and project
    - Confirm sub is linked to MDC already
    
    ![image.png](Day%201%20-%20Subscription%20safety%20and%20landing%20zone/image.png)
    
    - Added DeleteAfter resource tags to all resources
- Data-plane controls:
- Network controls:
- Monitoring destination:

## Verification

- Used the following to confirm everything exists
    
    *az account show --query "{name:name,state:state,tenant:tenantId}" --output table
    az provider list --query "[?registrationState=='Registered'].namespace" --output tsv
    az group list --tag project=sc500-30d --query "[].{name:name,location:location,tags:tags}" --output table*
    

| Test | Expected | Actual | Pass/fail | Evidence reference |
| --- | --- | --- | --- | --- |
| Allowed path | Budget "$20 lab" shows in Cost Management, then Budgets, with 3 alerts | Budget listed, thresholds 50% / 90% / forecast 100% | Pass | Portal, Budgets page |
| Denied path | n/a (budgets notify, they don't block spending) | n/a | n/a | n/a |
| Logging/detection | Alert reaches my email | Test email arrived (or "waiting, no spend yet") | Pass | Inbox |
| Recovery/rollback | I can edit or delete the budget | Changed threshold, then restored it | Pass | Budgets page |

## Findings

- Misconfiguration discovered: N/A
- Root cause: N/A
- Remediation: N/A
- Residual risk: N/A
- Follow-up owner/date: N/A

## Teardown

- Resources stopped/deleted: N/A - RG’s and cost alerts incur no cost
- Defender plan/add-on disabled: Nil at this stage
- Capacity/endpoint removed: N/A
- Cost view rechecked: Yes
- Persistent dependency for next lab: N/A

## Exam recall

- Three facts I can explain:
- One confusing distinction:
- One scenario question I wrote:

## Explain before moving on

- Why is a budget not a hard cap? - Because it is a monitoring tool not an enforcement action
- What protection is lost when moving from Free Trial to pay-as-you-go? This removes the spending limit and any usage will incur a billing cost right away.
- Why is East US 2 acceptable only for synthetic AI data in this scenario? Because any other AI data raises data sovereignty / regulatory compliance issues. As this is a test lab, this issue is not applicable.

- [ ]  Read all MS learn links in the lab walkthrough before moving on