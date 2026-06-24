# CDPH Check In App — Maintenance & Operations Guide

**Solution:** CDPHCheckInWebBased  
**Version:** 1.0.0.14  
**Last Updated:** June 2026  
**Primary Contact:** dylan.holcomb@cdph.ca.gov  
**App Owner / Publisher:** austin.friello@cdph.ca.gov

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Architecture](#2-architecture)
3. [Data Model — SharePoint Lists](#3-data-model--sharepoint-lists)
4. [Environment Variables](#4-environment-variables)
5. [App Screens & User Workflow](#5-app-screens--user-workflow)
6. [Deployment & Import Procedure](#6-deployment--import-procedure)
7. [Access Management](#7-access-management)
8. [Day-to-Day Operations](#8-day-to-day-operations)
9. [Known Issues & Technical Debt](#9-known-issues--technical-debt)
10. [Break/Fix Runbook](#10-breakfix-runbook)
11. [Version History](#11-version-history)

---

## 1. System Overview

The CDPH Check In App is a Power Apps canvas application used by the California Department of Public Health Accounts Payable (AP) division to manage daily employee attendance, work planning, and manager oversight.

**Business purpose:**
- Employees check in at the start of the day, logging their planned tasks and expected hours (up to 6 task segments).
- At the end of the day, employees check out and record actual hours completed per task, roadblocks, and sentiment.
- Break/return tracking captures mid-day breaks with optional reason codes.
- Overtime is logged separately with notes and hours.
- Managers access a dedicated screen to view real-time team status, check-in/check-out times, and drill into any employee's full day record.

**Scope:** AP Unit 1, AP Unit 2, AP Unit 3, AP Management Team, and Travel Team (~35 employees). Travel Team members have additional overtime access.

---

## 2. Architecture

```
User (Browser)
      │
      ▼
Power Apps Canvas App
(Web-based, Tablet 1366×768)
      │
      ├─── SharePoint Online Connection (shared_sharepointonline)
      │         Site: https://cdph.sharepoint.com/sites/APandTravel
      │         │
      │         ├─── List: Daily Log          ← time/task records
      │         └─── List: Employee Members   ← employee roster
      │
      └─── Environment Variables (Power Platform)
                cdph_spo_List       → site URL
                cdph_spo_LogList    → Daily Log list GUID
                cdph_spo_MemberList → Employee Members list GUID
```

**Platform:** Power Platform (Power Apps)  
**Solution type:** Unmanaged (see Known Issues — recommend converting to Managed for production)  
**Authentication:** Azure AD — users authenticate with their cdph.ca.gov accounts via SharePoint connector  
**No Power Automate flows** — all logic is contained in the canvas app (PowerFX)

---

## 3. Data Model — SharePoint Lists

### 3.1 Employee Members

Serves as the employee roster. The app reads this list to validate users and populate manager views.

| Column | Type | Notes |
|---|---|---|
| Name | Single line text | Display name |
| Title | Single line text | Job classification |
| UPN | Single line text | Azure AD UPN (email); used as primary key |
| ManagerUPN | Single line text | Supervisor's UPN |
| Department/Unit | Single line text | e.g., `AP - Unit 1`, `Travel` |
| Active | Yes/No | Filter for active employees only |
| Travel Team Member | Yes/No | Controls access to overtime screen |

**Notes:**
- Two `VACANT` placeholder rows exist (with `"VACANT"` as the UPN). These should be filtered in the app and should not appear in manager views.
- Case sensitivity: the app normalizes UPNs with `Lower()` for comparison — ensure data is consistent.

### 3.2 Daily Log

One row per employee per day per segment (break/return can create additional segments via `SegmentNumber`).

| Column | Type | Notes |
|---|---|---|
| Entry | Single line text | Auto-set to EmployeeUPN at creation |
| EmployeeUPN | Single line text | Employee's UPN — lookup key |
| Date | Date | Work date |
| Status | Choice | `Checked In`, `Checked Out`, `On Break` |
| CheckInTime | Date/Time | Timestamp of check-in |
| BreakTimeOut | Date/Time | Timestamp of break start |
| BreakTimeIn | Date/Time | Timestamp of break return |
| CheckOutTime | Date/Time | Timestamp of check-out |
| Plan 1–6 | Single line text | Planned task descriptions |
| Hours 1–6 | Number | Planned hours per task |
| CheckoutText 1–6 | Single line text | Actual task descriptions at checkout |
| CheckOutHours 1–6 | Number | Actual hours at checkout |
| Roadblocks | Multiple lines | Free-text blocker notes |
| Sentiment | Choice | Employee self-reported mood/sentiment |
| Manager Comments | Multiple lines | Manager notes on the record |
| OvertimeNotes | Multiple lines | Overtime description |
| OvertimeHours | Number | Total overtime hours |
| CheckoutType | Single line text | Type of checkout |
| BreakReason | Single line text | Reason code for break |
| BreakNote | Multiple lines | Free-text break notes |
| SegmentNumber | Number | Segment counter for break/return cycles |

**Capacity limit:** 6 task segments per check-in. If users routinely hit 6, the schema and app screens will need expanding.

---

## 4. Environment Variables

All three variables must be set correctly in every environment the solution is imported into.

| Schema Name | Display Name | Type | Current Value | Purpose |
|---|---|---|---|---|
| `cdph_spo_List` | spo_Site | Connection dataset | `https://cdph.sharepoint.com/sites/APandTravel` | SharePoint site URL |
| `cdph_spo_LogList` | spo_LogList | Connection table | `7f32f3ad-327e-4780-a9ac-fd3f5909e7cd` | Daily Log list GUID |
| `cdph_spo_MemberList` | spo_MemberList | Connection table | `84610c34-3506-48e6-aa52-f390d0e1d184` | Employee Members list GUID |

**How to find a SharePoint list GUID:**  
Go to the list in SharePoint → Settings → List Settings → the GUID is in the URL as `List=%7B...%7D` (decode the URL encoding to get the plain GUID).

**How to update environment variables after import:**  
Power Platform Admin Center → Environments → [environment] → Solutions → CDPHCheckInWebBased → Environment Variables → edit each value.

---

## 5. App Screens & User Workflow

### Screens

| Screen | Internal Name | Who Uses It | Purpose |
|---|---|---|---|
| Home | `scrHome` | All users | Landing page — status display, action buttons (Check In, Check Out, Break, Return, Add Overtime) |
| Check In | `scrCheckIn` | Employees | Enter up to 6 planned tasks + planned hours. Submits to Daily Log. |
| Check Out | `scrCheckOut` | Employees | Enter actual hours per task, roadblocks. Updates Daily Log record. |
| Overtime | `scrOvertime` | Travel Team | Log overtime notes and hours. |
| Manager | `scrManager` | Managers | Gallery of team — name, status, check-in/out times for today. |
| Manager Detail | `scrManagerDetail` | Managers | Full read-only view of a selected employee's daily record. |

### User Flow

```
App Open
   │
   ├── [Manager?] → scrManager → scrManagerDetail (read only)
   │
   └── [Employee]
         │
         ├─ Not checked in → scrCheckIn → fill tasks → submit → scrHome (Status: Checked In)
         │
         ├─ Checked in → scrHome → Break Leave → scrHome (Status: On Break)
         │                      → Break Return → scrHome (Status: Checked In)
         │                      → Add Overtime → scrOvertime → scrHome
         │
         └─ Ready to leave → scrCheckOut → fill actual hours + roadblocks → submit → scrHome (Status: Checked Out)
```

### OnStart Logic (PowerFX pattern)

The app sets context variables on launch: current user (`varUser`), today's date (`varToday`), and loads or creates the user's daily log record into a local collection (`colToday`). If no record exists for today, a blank draft shell is created locally (not yet saved to SharePoint) to populate the UI.

---

## 6. Deployment & Import Procedure

### Prerequisites

- Power Platform environment with SharePoint Online connector available
- SharePoint site `APandTravel` provisioned with `Daily Log` and `Employee Members` lists
- Azure AD security group or individual user list for sharing the app

### Steps

**Step 1 — Import the solution**

1. Go to [make.powerapps.com](https://make.powerapps.com) and select the target environment.
2. Navigate to **Solutions** → **Import solution**.
3. Upload `CDPHCheckInWebBased_1_0_0_14.zip`.
4. On the connection step, assign the SharePoint Online connector to an authenticated connection (service account recommended).
5. Complete the import.

**Step 2 — Set environment variables**

After import, set the three environment variables (see Section 4) to match the target environment's SharePoint site and list GUIDs.

**Step 3 — Share the app**

1. Open the solution → Canvas Apps → **CDPH Check In App Web Based**.
2. Share with the AP division security group or individual users.
3. Ensure the SharePoint connection is shared or each user consents to the connector.

**Step 4 — Verify**

1. Open the app as a test user — confirm the home screen loads and shows the correct status.
2. Submit a test check-in and verify the row appears in the Daily Log SharePoint list.
3. Open the Manager screen as a supervisor — confirm the gallery shows team members.

### Promoting to a New Environment

When promoting from dev → test → production:
1. Export the solution as **managed** (not unmanaged) for the production environment.
2. Update environment variable values for the new environment's SharePoint lists.
3. Reassign the SharePoint connection to a production service account or shared connection.

---

## 7. Access Management

### App Access

| Role | Access Level | How Granted |
|---|---|---|
| Employee (AP) | App user | Share app with user or security group |
| Travel Team | App user + Overtime screen | `Travel Team Member = True` in Employee Members list |
| Manager/Supervisor | Manager screen access | Determined by `ManagerUPN` field — user's UPN matches another employee's ManagerUPN |
| IT Admin / M&O | Solution import / edit | Power Platform Environment Admin role |

### SharePoint Permissions

| List | Minimum Permission | Notes |
|---|---|---|
| Daily Log | Contribute | Employees need Create + Edit on their own rows |
| Employee Members | Read | Employees only need read access |

Managers need Read access on Daily Log to view other employees' records.

### Adding New Employees

1. Add a row to the **Employee Members** SharePoint list with their UPN, manager's UPN, department, and `Active = True`.
2. Share the Power App with the new user (or ensure they're in the security group).
3. No app changes required.

### Removing Employees

1. Set `Active = False` on their Employee Members row (do not delete — preserves historical data integrity).
2. Remove them from the app sharing if needed.

---

## 8. Day-to-Day Operations

### Monitoring

- **No automated alerts** are built into the current solution. Check-in status is visible in the Daily Log SharePoint list or via the Manager screen in the app.
- For volume reporting, export the Daily Log list to Excel or connect Power BI to the SharePoint list.

### Data Retention

No automated retention or archival is configured. Daily Log rows accumulate indefinitely. Consider a Power Automate flow to archive or delete records older than a policy-defined threshold.

### Backing Up the Solution

Export the solution from Power Platform periodically:
- make.powerapps.com → Solutions → CDPHCheckInWebBased → Export → as zip
- Store in version control (this repository) and SharePoint.

---

## 9. Known Issues & Technical Debt

### Issue 1 — Unmanaged Solution in Production (HIGH)

The solution is currently exported as **unmanaged** (`<Managed>0</Managed>`). Deploying an unmanaged solution to production allows anyone with Environment Maker access to accidentally modify it. **Recommendation:** before moving to M&O, export as a **managed** solution for production and keep the unmanaged version in dev only.

### Issue 2 — Employee Members Not Fully Parameterized (MEDIUM)

In `customizations.xml`, the `Employee Members` data source has a **hardcoded** dataset URL (`https://cdph.sharepoint.com/sites/APandTravel`) rather than being driven by the `cdph_spo_MemberList` environment variable. The `cdph_spo_MemberList` env var is defined but only the `spo_LogList` uses the full env var override pattern in `ConnectionReferences`. If the SharePoint site URL changes, the Employee Members connection will break while the Daily Log connection will self-heal. **Recommendation:** open the app in Power Apps Studio and re-bind the Employee Members data source via its environment variable.

### Issue 3 — PowerFX OnStart Uses Non-Matching Variable Names (MEDIUM)

The `PowerFX 1.txt` file references `EnvironmentVariableValue("SPO_SiteUrl")` and `EnvironmentVariableValue("LIST_DAILYLOG")`, but the actual environment variable schema names are `cdph_spo_List` and `cdph_spo_LogList`. If this PowerFX was copied from a draft and used as-is inside the app, the env var lookups will return blank. **Recommendation:** confirm the actual OnStart formula in Power Apps Studio uses the correct schema names.

### Issue 4 — VACANT Placeholder Rows in Employee Members (LOW)

Two rows with `UPN = "VACANT"` exist in the Employee Members list. These may appear in manager gallery views if not filtered. **Recommendation:** add a filter on `Active = True AND UPN <> "VACANT"` wherever the Member list is queried.

### Issue 5 — Fixed 6-Task Limit (LOW)

The schema supports a maximum of 6 tasks per day (Plan 1–6, Hours 1–6, CheckoutText 1–6, CheckOutHours 1–6). If the AP team needs more, both the SharePoint list schema and app screens require changes. **Recommendation:** monitor whether users hit this limit; if so, consider a child-list approach (one row per task) for more flexibility.

### Issue 6 — No Data Retention or Archival Policy (LOW)

Daily Log rows grow indefinitely. Large lists can degrade SharePoint connector performance in Power Apps (known threshold: ~5,000 rows for non-delegable queries; 100,000 rows before list view threshold errors). **Recommendation:** implement a Power Automate flow to archive records older than 1 year to a separate archive list or SharePoint archive site.

---

## 10. Break/Fix Runbook

### App Won't Load / Blank Screen

1. Check if the SharePoint connection is still authenticated. Open Power Apps Studio → Data → verify the SharePoint connection shows no error icons.
2. Verify the environment variables are set (see Section 4). Missing values cause the app to fail silently.
3. Try opening the app in a private browser window to rule out cache issues.

### Employee Missing from Manager View

1. Confirm the employee's row exists in the Employee Members SharePoint list.
2. Confirm `Active = True` and `ManagerUPN` matches the manager's UPN exactly (case-insensitive in the app, but should be consistent).
3. Confirm the employee has a check-in record for today in the Daily Log.

### Employee Can't Check In (Button Disabled or Error)

1. Confirm the employee already has no check-in record for today (double check-in is blocked by design).
2. Confirm the SharePoint connection has Contribute permission on the Daily Log list.
3. Check for SharePoint throttling — if many users check in simultaneously, some writes may fail. Retry usually resolves this.

### Data Not Saving (No Error Shown)

1. Verify SharePoint list permissions (Contribute for employees).
2. Check if column names in SharePoint match exactly what the app expects. Any rename in SharePoint without updating the app's data source will silently fail.
3. Test with a known-good admin account to isolate user permission issues.

### Solution Import Fails

1. Ensure the target environment has the SharePoint Online connector available (check connector policies).
2. If import fails on environment variables, manually create them after import using the values in Section 4.
3. Check if there is a version conflict — the import UI will flag if a newer version already exists.

### Re-Importing After an Update

1. Increment the version number in `solution.xml` before exporting a new version.
2. Import as an **upgrade** (not a new import) to preserve existing data and connections.

---

## 11. Version History

| Version | Date | Author | Notes |
|---|---|---|---|
| 1.0.0.14 | 2026-03-02 | Austin Friello | Added Travel Team Overtime access; current production version |
| 1.0.0.4 | — | Austin Friello | Added `cdph_spo_MemberList` environment variable |
| 1.0.0.2 | — | Austin Friello | Added `cdph_spo_List` and `cdph_spo_LogList` environment variables |
| 1.0.0.1 | — | Austin Friello | Initial web-based version (reformatted from mobile) |

---

*For questions about this document, contact dylan.holcomb@cdph.ca.gov.*
