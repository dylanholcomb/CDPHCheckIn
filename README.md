# CDPH Check In App — Web Based

Power Apps canvas app for the CDPH Accounts Payable division to manage daily employee check-in, check-out, work plans, and manager oversight.

**Solution version:** 1.0.0.14  
**Last published:** 2026-03-02  
**Platform:** Power Apps (Canvas, Tablet layout)  
**Publisher contact:** austin.friello@cdph.ca.gov

---

## Overview

This app replaces a paper/manual process with a structured digital check-in workflow. Employees log their planned tasks and hours at check-in, then submit actual hours and completion notes at check-out. Managers can view their team's status in real time.

## Architecture

| Layer | Technology |
|---|---|
| App | Power Apps Canvas (web-based, tablet layout) |
| Data — Daily Log | SharePoint Online list (`Daily Log`) |
| Data — Members | SharePoint Online list (`Employee Members`) |
| Environment config | Power Platform Environment Variables |
| Auth | Azure AD (user context via `User()` function) |

## SharePoint Lists

### Employee Members
Columns: `Name`, `Title`, `UPN`, `ManagerUPN`, `Department/Unit`, `Active`, `Travel Team Member`

### Daily Log
Columns: `Entry`, `EmployeeUPN`, `Date`, `Status`, `CheckInTime`, `BreakTimeOut`, `BreakTimeIn`, `CheckOutTime`, `Plan 1–6`, `Hours 1–6`, `CheckoutText 1–6`, `CheckOutHours 1–6`, `Roadblocks`, `Sentiment`, `Manager Comments`, `OvertimeNotes`, `OvertimeHours`, `CheckoutType`, `BreakReason`, `BreakNote`, `SegmentNumber`

## Environment Variables

| Schema Name | Type | Purpose | Current Value |
|---|---|---|---|
| `cdph_spo_List` | Connection (dataset) | SharePoint site URL | `https://cdph.sharepoint.com/sites/APandTravel` |
| `cdph_spo_LogList` | Connection (table) | Daily Log list GUID | `7f32f3ad-327e-4780-a9ac-fd3f5909e7cd` |
| `cdph_spo_MemberList` | Connection (table) | Employee Members list GUID | `84610c34-3506-48e6-aa52-f390d0e1d184` |

## App Screens

| Screen | Purpose |
|---|---|
| `scrHome` | Landing screen — check-in / check-out / overtime / break buttons + status |
| `scrCheckIn` | Task planning form (up to 6 tasks + hours at check-in) |
| `scrCheckOut` | Actual hours form + roadblocks at check-out |
| `scrOvertime` | Overtime notes and hours entry |
| `scrManager` | Manager view — team status gallery |
| `scrManagerDetail` | Manager detail view — employee's full day record (read-only form) |

## Deployment

See [OPERATIONS.md](OPERATIONS.md) for full M&O runbook.

Quick steps:
1. Import `CDPHCheckInWebBased_1_0_0_14.zip` into the target Power Platform environment
2. Set all three environment variables to the correct SharePoint site and list GUIDs
3. Share the app with the AP division security group
4. Verify the SharePoint connection is authenticated under a service account or shared connection

## PII Notice

`Daily Log (7).csv` and `Employee Members (2).csv` contain real employee UPNs, names, and work activity. These are committed as schema/data references only. Do not push to a public repository without scrubbing PII.

## Version History

| Version | Date | Notes |
|---|---|---|
| 1.0.0.14 | 2026-03-02 | Added Travel Team Overtime access |
| 1.0.0.4 | — | Added `cdph_spo_MemberList` environment variable |
| 1.0.0.2 | — | Added `cdph_spo_List` and `cdph_spo_LogList` environment variables |
| 1.0 | — | Initial release |
