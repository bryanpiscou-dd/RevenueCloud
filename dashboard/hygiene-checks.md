# Salesforce Admin Hygiene Dashboard: checks

SOQL behind each tile of `salesforce-hygiene-dashboard.html`. Re-run to refresh. Native build: one Report per check, grouped into a Lightning Dashboard (Ownership / Pipeline / Users / Data quality / Automation).

| Check | SOQL / source | Native report type |
|---|---|---|
| Accounts, inactive owner | `SELECT COUNT(Id) FROM Account WHERE Owner.IsActive = false` | Accounts, filter Owner: Active = False |
| Open opps, inactive owner | `... FROM Opportunity WHERE IsClosed = false AND Owner.IsActive = false AND Type != 'Partner Opportunity'` | Opportunities |
| Open opps past close date | `... WHERE IsClosed = false AND CloseDate < TODAY` | Opportunities |
| Open opps, no activity 30d | `... WHERE IsClosed = false AND LastActivityDate < LAST_N_DAYS:30` | Opportunities (note: opps with no activity ever have null, add `OR LastActivityDate = null`) |
| Open opps, no amount | `... WHERE IsClosed = false AND (Amount = null OR Amount = 0)` | Opportunities |
| Users, no login 7d | `... FROM User WHERE IsActive = true AND UserType = 'Standard' AND (LastLoginDate < LAST_N_DAYS:7 OR LastLoginDate = null)` | Users |
| Contacts, no account | `... FROM Contact WHERE AccountId = null` | Contacts & Accounts |
| Contacts, no email | `... FROM Contact WHERE Email = null` | Contacts |
| Accounts, Assigned AE inactive | `SELECT Assigned_AE__r.Name, COUNT(Id) FROM Account WHERE Assigned_AE__c != null AND Assigned_AE__r.IsActive = false GROUP BY Assigned_AE__r.Name` (do not use `Assigned_AE__r.IsActive = false` alone, it also matches null) | Accounts |
| Accounts, no Country | `... FROM Account WHERE Country__c = null` | Accounts |
| Mismatch: US country, non-US territory | `... WHERE Country__c = 'United States' AND New_US_Territory__c IN ('EMEA','Latin America')` | Accounts |
| Mismatch: US or Canada, US State and State Region both empty | `... WHERE Country__c IN ('United States','Canada') AND US_State__c = null AND State_Region__c = null` | Accounts |
| Mismatch: US or Canada, US State empty, State Region filled (country likely wrong) | `... AND US_State__c = null AND State_Region__c != null` | Accounts |
| Mismatch: US or Canada, US State empty (state field also holds provinces) | `... WHERE Country__c IN ('United States','Canada') AND US_State__c = null` | Accounts |
| Mismatch: US State on non-US/CA country | `... WHERE Country__c NOT IN ('United States','Canada') AND Country__c != null AND US_State__c != null` | Accounts |
| Mismatch: non-US/CA country in US sub-territory | `... WHERE Country__c NOT IN ('United States','Canada') AND Country__c != null AND New_US_Territory__c IN ('Midwest','Northeast','Northern California','Pacific','New York','Southeast')` | Accounts |
| Mismatch: LatAm country elsewhere | `... WHERE Country__c IN ('Brazil','Mexico','Argentina','Colombia') AND New_US_Territory__c != 'Latin America'` | Accounts |
| Mismatch: Europe country, other Region | `... WHERE Country__c IN (<European list>) AND Region__c IN ('North America','Latin America','Asia Pacific','Oceania')` | Accounts |
| Territory set, Country blank | `... WHERE Country__c = null AND New_US_Territory__c IN (<US sub-territories>)` | Accounts |
| Duplicate contacts by email | `SELECT Email, COUNT(Id) FROM Contact WHERE Email != null GROUP BY Email HAVING COUNT(Id) > 1` | Contacts (summary report grouped by Email, record count > 1) |
| Duplicate accounts by website | `SELECT Website, COUNT(Id) FROM Account WHERE Website != null GROUP BY Website HAVING COUNT(Id) > 1` | Accounts (summary by Website) |
| Duplicate accounts by name | `SELECT Name, COUNT(Id) FROM Account GROUP BY Name HAVING COUNT(Id) > 1 LIMIT 2000` (capped, real total is higher) | Accounts (summary by Name) |
| Duplicate sets from rules | `SELECT COUNT(Id) FROM DuplicateRecordSet` | none |
| US zips missing leading zero | `... FROM Account WHERE Country__c = 'United States' AND (BillingPostalCode LIKE '____' OR BillingPostalCode LIKE '___')` | Accounts |
| US accounts with no zip | `... WHERE Country__c = 'United States' AND BillingPostalCode = null` | Accounts |
| Flows, stale | `SELECT ApiName, ProcessType, TriggerType, LastModifiedDate FROM FlowDefinitionView WHERE IsActive = true AND NamespacePrefix = null AND LastModifiedDate < <today - 365d>` (split by date to stay under 2,000 rows) | none (Tooling API) |
| Paused interviews | `SELECT COUNT(Id) FROM FlowInterview WHERE CreatedDate < LAST_N_DAYS:30` | none |

Each tile in the dashboard also has a Generate SOQL button (records list and count).

## Live refresh and enrichment
- The dashboard page calls the viewer's Salesforce connector (`soqlQuery`) on open and every 24h while open. Viewers without the connector see the last published snapshot (13 Oct 2026).
- Commentary text on each tile is a snapshot. Counts, percentages and severity update live.
- "Enrich with Clay" runs the Clay Company Address function (one credit per account) on up to 50 accounts from a check and proposes Country__c, State_Region__c, City__c and BillingPostalCode in one CSV, with an option to fill empty fields only (US_State__c optional, off by default because it is derived from State_Region__c). It never writes to Salesforce: load the copied CSV with Data Loader after review.

## Known limits
- Flow last run date is not queryable. Last modified date is a proxy. Use Flow Event Monitoring or an in-flow log for real run history.
- `FlowDefinitionView` does not support `COUNT()`, page the rows.
- `New_US_Territory__c` cannot be grouped in SOQL, so territory rules filter on explicit value lists. EMEA is used as the rest-of-world default (Japan, India, China, Australia), so it is not flagged for those countries.

## Next checks to add
Duplicate accounts by normalized name/domain, opportunities without a contact role, accounts without an owner-region match, validation rules and flows referencing deactivated users, permission sets assigned to inactive users, unused custom fields.
