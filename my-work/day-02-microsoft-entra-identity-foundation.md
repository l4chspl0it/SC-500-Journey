# Day 2 - Microsoft Entra identity foundation

Date Completed: 03/10/26
Status: Done

## Cost checkpoint

- Credit remaining: [Subscriptions > Overview, or "n/a" if pay-as-you-go]
- Month-to-date actual: [Cost analysis, likely ~$0 and delayed]
- Forecast: [Cost analysis, likely ~$0]
- Top services: none yet (setup day)
- Unexpected meters: none expected. The Entra P2 trial bills through Microsoft 365, not Azure, so it won't show in Azure Cost Management
- Action: [budget alert created? e.g. $20, actual 50% and 90%, forecast 100%] + reminder set to cancel P2 trial before it ends

## Business control

- Requirement: Northstar needs identity governance (PIM, Identity Protection, access reviews) available in the tenant, and spend kept in check before deploying services
- Threat or failure mode: without P2, no PIM or risk-based controls, so standing admin access; without a budget, forgotten resources run up cost
- Control selected: Entra ID P2 30-day trial licensed to the lab accounts; budget alerts
- Why this service/control fits: P2 includes PIM, Identity Protection and access reviews used in later labs; the trial is free for 30 days
- Rejected alternative and reason:
    - Entra ID Governance / Entra Suite trials: add-on and larger bundle, not needed here
    - Paid P2 licences: unnecessary cost for a lab
    - New-tenant sign-up form: would create a tenant disconnected from my Azure subscription

## Implementation

- Resources changed:
    - Activated the **Entra ID P2** 30-day trial (ordered through the Microsoft 365 admin center, $0 today)
    - Created user `labadmin`
    - Assigned P2 licences
- Identity and role/scope: `labadmin` = Global Administrator (tenant scope); P2 assigned to `sc500.labs@outlook.com` and `labadmin`; usage location set to Australia
- Management-plane controls: licence assignment via M365 admin center; [budget at subscription scope, if created]
- Data-plane controls: n/a
- Network controls: n/a
- Monitoring destination: n/a yet (no logging configured). Budget alerts go to email

## Verification

| Test | Expected | Actual | Pass/fail | Evidence reference |
| --- | --- | --- | --- | --- |
| Allowed path | P2 shows in All products and can be assigned | Assigned to sc500 account; labadmin after fixing usage location | Pass | M365 admin center, Licenses |
| Denied path | Assigning to a user with no usage location is blocked | "Invalid usage location" error for labadmin | Pass | Assign licenses error message |
| Logging/detection | Licence assignment appears in Entra audit logs | [check Entra ID > Audit logs] | [ ] | Entra audit logs |
| Recovery/rollback | Trial can be cancelled before it ends | [check Billing > Your products shows end date and cancel option] | [ ] | M365 admin center, Billing |

## Notes

- Activate first redirected to a new-tenant form. Fixed by activating as a second Global Admin (`labadmin`), then Sign In (not "create new account")
- Trial ends 30 days after 3 Oct. Cancel by 30 Oct

## Findings

- Misconfiguration discovered: N/A
- Root cause: N/A
- Remediation: N/A
- Residual risk: N/A
- Follow-up owner/date: N/A

## Teardown

- Resources stopped/deleted: N/A
- Defender plan/add-on disabled: P2 license added. Cancel by 30th October 2026
- Capacity/endpoint removed: N/A
- Cost view rechecked: Yes
- Persistent dependency for next lab: N/A

## Exam recall

- Three facts I can explain: How to add licenses in Azure and the Admin center, importance of a breakglass/emergency account, difference auth methods including phishing resistant methods
- One confusing distinction: Activating the P2 license as a new user with GA role
- One scenario question I wrote: N/A