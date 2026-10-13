# Salesforce hygiene checks

SOQL behind each tile of `salesforce-hygiene-dashboard.html`. Re-run to refresh. Native build: one Report per check, grouped into a Lightning Dashboard (Ownership / Pipeline / Users / Data quality / Automation).

| Check | SOQL / source | Native report type |
|---|---|---|
| Accounts, inactive owner | `SELECT COUNT(Id) FROM Account WHERE Owner.IsActive = false` | Accounts, filter Owner: Active = False |
| Open opps, inactive owner | `... FROM Opportunity WHERE IsClosed = false AND Owner.IsActive = false` | Opportunities |
| Open opps past close date | `... WHERE IsClosed = false AND CloseDate < TODAY` | Opportunities |
| Open opps, no activity 30d | `... WHERE IsClosed = false AND LastActivityDate < LAST_N_DAYS:30` | Opportunities (note: opps with no activity ever have null, add `OR LastActivityDate = null`) |
| Open opps, no amount | `... WHERE IsClosed = false AND (Amount = null OR Amount = 0)` | Opportunities |
| Users, no login 7d | `... FROM User WHERE IsActive = true AND UserType = 'Standard' AND (LastLoginDate < LAST_N_DAYS:7 OR LastLoginDate = null)` | Users |
| Leads, inactive owner | `... FROM Lead WHERE IsConverted = false AND Owner.IsActive = false` | Leads |
| Cases, inactive owner | `... FROM Case WHERE IsClosed = false AND Owner.IsActive = false` | Cases |
| Contacts, no account | `... FROM Contact WHERE AccountId = null` | Contacts & Accounts |
| Contacts, no email | `... FROM Contact WHERE Email = null` | Contacts |
| Flows, stale | `SELECT ApiName, ProcessType, TriggerType, LastModifiedDate FROM FlowDefinitionView WHERE IsActive = true AND NamespacePrefix = null ORDER BY LastModifiedDate` | none (Tooling API) |
| Paused interviews | `SELECT COUNT(Id) FROM FlowInterview WHERE CreatedDate < LAST_N_DAYS:30` | none |

## Known limits
- Flow last run date is not queryable. Last modified date is a proxy. Use Flow Event Monitoring or an in-flow log for real run history.
- `FlowDefinitionView` does not support `COUNT()`, page the rows.

## Next checks to add
Duplicate accounts by normalized name/domain, opportunities without a contact role, accounts without an owner-region match, validation rules and flows referencing deactivated users, permission sets assigned to inactive users, unused custom fields.
