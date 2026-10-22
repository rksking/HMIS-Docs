# Employee Delete, Inactivate, Reactivate & Re-join — Plan

Branch: `employee-bulk-soft-delete` · Created 2026-10-22 · Status: **Steps 0–6 done 2026-10-22 · Step 7 next** · Access 2026-10-22: Super Admin + Developer always, other roles by permission (§1a) · Management artifact: https://claude.ai/artifact/G41UTHjMa4fSMycD22sKmJ

## 1. What we are building

HR asked for four actions on the Employee List:

| # | HR request | Action in HRMS | What happens |
|---|---|---|---|
| A | "We have inactive staff in HRMS; need to inactivate" | **Inactivate** | Employee becomes Inactive from a date, with a reason. Login is turned off. The employee drops out of attendance, roster and payroll from that date. Still visible under the Inactive filter |
| B | "Delete if anyone joined wrongly" | **Delete** | Employee is hidden from everyone. The row stays in the database (`IsDeleted = 1`), because data is never deleted. For HR this works like a permanent delete. Only Developer can see it and restore it |
| C | "Separation was wrong" | **Reactivate** | The same employee record and code come back to Active. The separation or inactivation is marked Reversed, not removed |
| D | "Re-joins after separation" | **Re-join** | A new employee record is created for the new employment, pre-filled from the old one and linked to it. The old record stays Separated with its history |

### Who can do what

| Permission (Role Permissions › Employee List) | Gives |
|---|---|
| `employees.inactivate` | Inactivate icon + bulk **Inactivate** button |
| `employees.reactivate` | Reactivate icon on Inactive / Separated rows; **Re-join** also needs `employees.create` |
| `employees.deleted` | Bin icon + bulk **Delete** button |
| Developer role only (checked in code with `IsDeveloper`, not a permission) | **Deleted** tab on Employee List, **Restore** one or many |

None of the three permissions is given to any role by default. An admin ticks them per role on Role Permissions.
Without a permission, its icons and buttons are not shown and the API returns 403.

> **Back in use (Step 4, 2026-10-22).** §1a first replaced this table with "Super Admin + Developer only". The user then asked
> for it to be permission based as well: Super Admin and Developer always have the actions, and any other role gets them when
> an admin ticks the permission on Role Permissions. See §1a.

### 1a. Access and screens (user decision 2026-10-22)

The user asked for these as **separate tabs shown only to Super Admin and Developer**:

| Tab on Employee List | HR request | Lists | Action |
|---|---|---|---|
| **Inactive Staff** | "We have inactive staff in HRMS; need to inactivate" | Active / Probation / Confirmed staff to pick from, plus the ones already Inactive | **Inactivate** (one or many) |
| **Delete Staff** | "Delete permanently if anyone joined wrongly" | Staff of any status to pick from | **Delete** (one or many) |
| **Reactivate / Re-joined Staff** | "Separation was wrong, or they re-join after separation" | Inactive and Separated staff | **Reactivate** (same record) or **Re-join** (new record + new code) |
| **Deleted** (unchanged, O2) | Developer clean-up | Deleted staff | **Restore** |

- **Who (changed in Step 4, user 2026-10-22):** **Super Admin and Developer always**; **any other role** when Role Permissions
  (Employee List) grants it the tab's permission:

  | Tab | Permission for other roles |
  |---|---|
  | Inactive Staff | `employees.inactivate` |
  | Delete Staff | `employees.deleted` |
  | Reactivate / Re-joined Staff | `employees.reactivate` (Re-join also needs `employees.create`) |

  Rule in code: `IsSuperAdmin || IsDeveloper || HasPermission(code)` (`EmployeeStatusActionService.EnsureCanAct`). No role has the
  permissions by default. Everyone else sees the Employee List as today, with no checkboxes, bulk bar or status icons.
- **API:** every action API checks the same rule in the service and returns 403 otherwise (no `[Authorize(Policy)]`, because a
  policy would refuse a Super Admin whose role is not god-mode). Step 4 switched `bulk-delete` to it; Steps 5–6 use it for
  Reactivate and Re-join.
- **Deleted tab and Restore** stay **Developer only** (O2).

## 2. Agreed rules

1. **Delete columns on `Employees`:** `IsDeleted bit NOT NULL DEFAULT 0`, `DeletedOn datetime2 NULL`, `DeletedBy nvarchar NULL`.
2. **Nothing is removed from the database**, ever. All four actions and Restore write to the audit log, and the log is never cleaned (follow.md 20a).
3. **Employee code stays reserved** after delete, so Restore never clashes.
4. **No static values:** reasons come from lookup masters (`INACTIVE_REASON`, `REACTIVATE_REASON`), and statuses come from the `EmploymentStatuses` constants.

## 3. New data (besides the 3 delete columns)

| Item | Why |
|---|---|
| `EmploymentStatuses.Inactive = "INACTIVE"` | So Inactive is different from Separated (resigned) and Terminated |
| Table `employee_status_history` — EmployeeId, Action (`INACTIVATE`/`REACTIVATE`/`REJOIN`/`DELETE`/`RESTORE`), EffectiveDate, ReasonCode, Remarks, PreviousStatus, NewStatus, PreviousIsActive, ActionBy, ActionAt | One place for the reason and dates of every action. Reactivate reads the previous status from here. Shown as a timeline in the Employee Dossier |
| `Employees.RejoinOfEmployeeId nvarchar NULL` | Links a re-joined record to the old one |
| Unique indexes on National ID and work email become filtered (`isActive = 1 AND isDeleted = 0`) | Re-join and re-adding a wrongly joined person are blocked today (§8 e) |
| Lookups `INACTIVE_REASON`, `REACTIVATE_REASON` | Reason dropdowns, managed on Masters |

## 4. Rules per action (defaults, change before Step 1 if needed)

**Shared pre-check (Inactivate and Delete).** It blocks yourself, anyone who still has active direct reports, and anyone with open tasks assigned to them on /task.
Pending requests the employee raised themselves only give a warning. All actions respect company + location access scope
(docs/access.md). Bulk is limited to 200 rows per action, and each row gets its own result (Done / Skipped + reason).

| Action | Rules |
|---|---|
| **Inactivate** | Effective date is **today or earlier** (cleaning up old data needs past dates; a future exit goes through the Separation workflow). Reason + remarks are required. Sets `IsActive = 0`, EmploymentStatus `INACTIVE` and `AccessBlockedAt`, and turns login off unless another active employee record shares that login (same rule as Separation release). Warns if a payroll period after the effective date is already finalised. Only Active / Probation / Confirmed employees |
| **Delete** | Reason is required. Sets the 3 delete columns **and `IsActive = 0`** (the earlier `IsActive` is saved in history; see §5 for why), turns login off and hides the employee everywhere. Works on any status |
| **Reactivate** | Only Inactive or Separated rows. Reason is required. Restores `IsActive = 1` and the status from before (from history), clears `AccessBlockedAt` and `SeparationDate`, turns login back on, and marks the separation request **Reversed**. Separation release does **not** move direct reports (they stay with this person), so nothing needs to move back. Leave that release cancelled is **not** brought back; the drawer lists it. Warns if final settlement was already paid. If `employment_details` has no row for the employee, it is skipped with a message |
| **Re-join** | Only Separated / Inactive rows. Opens Add Employee pre-filled (personal, ID, bank, education, documents linked) with a **new employee code** and a new joining date. Needs the unique-index change in Step 1 (National ID and work email are unique today). The app check allows a match only with the old Separated / Inactive record. Leave and service start fresh. Sets `RejoinOfEmployeeId`. The new code gives a new username; the person's **old login moves** to the new record (Step 6, because `users.email` is unique) |
| **Restore (Developer)** | Clears the 3 delete columns and puts back the saved `IsActive`. Login comes back only if the employee is Active. Blocked if an active record now uses the same National ID / work email |

## 5. How "deleted" is hidden

Step 0 found that a database filter alone is **not enough**. The code calls `IgnoreQueryFilters()` without a filter
name 1,216 times (in 123 files), and each call switches off every filter, including a new one. So deleted employees are hidden in three layers:

1. **Delete also sets `IsActive = 0`.** About 47 services and 20 database procedures already leave out inactive
   employees, so a deleted employee drops out of attendance, roster, payroll, leave, approvals and dropdowns at once,
   the same way a separated employee does.
2. **EF Core named filter `SoftDelete`** (`!e.IsDeleted`) on `Employee`, next to `CompanyFilter`
   ([HrmsDbContext.cs:942](../backend/Infrastructure/Persistence/HrmsDbContext.cs#L942)). It covers normal LINQ reads.
   The Developer list uses `IgnoreQueryFilters([SoftDeleteFilter])`, so the company filter still applies.
3. **Explicit `!IsDeleted` / `isDeleted = 0`** in places that **show inactive people on purpose**: Employee List
   (Inactive/Separated filter), employee reports, by-id reads (Dossier, Assignment, Profile), lookups that include inactive
   people, and the 109 `Employees.IgnoreQueryFilters()` reads that have no `IsActive` check nearby (Step 3 goes through them).

- **Login guard:** [AuthService.cs:204](../backend/Infrastructure/Services/AuthService.cs#L204) already only looks at active
  employees. Step 3 adds `!IsDeleted` there as a second guard.
- **Inactive** needs no new hiding. Code that already uses `IsActive` treats Inactive like Separated.

## 6. Steps

Each step runs in a fresh chat. After each step update `docs/policies.md`, the management artifact (incl. the How-to-operate
guide), the browser test checklist, and `task.xlsx`.

| Step | What | Output |
|---|---|---|
| **0** | Read-only audit: (a) every raw SQL / procedure / Dapper / report / job that reads `Employees`; (b) every `.Include(x => x.Employee)` that could break when hidden; (c) where `IsActive` / `EmploymentStatus` drive attendance, roster, payroll and leave; (d) everything Separation release changes; (e) Add Employee duplicate checks; (f) whether `JobHistory` can hold status events instead of a new table | §8 filled. No code |
| **1** | DB + entity: delete columns + index, `RejoinOfEmployeeId`, `employee_status_history` (+ `PreviousIsActive`), `INACTIVE` status, 2 reason lookups, 3 permission rows under Employee List (no role grant), `SoftDelete` named filter. Change unique indexes `UQ_Employee_NationalId` and `UQ_Employee_Company_WorkEmail` to filtered (`WHERE isActive = 1 AND isDeleted = 0`) so Re-join and re-adding a wrongly joined person work. **Keep** `UQ_Employee_Company_EmpNum` (codes stay reserved). SQL in `docs/db_changes.sql`, run on HRMSCore_Local2. Filter tests | Schema live |
| **2** | Delete API `POST /api/employees/bulk-delete` + Developer APIs `GET /api/employees/deleted`, `POST /api/employees/bulk-restore` | API + tests |
| **3** | Hide deleted employees everywhere: layer 3 of §5 using the §8 lists (procedures without an `IsActive` check, `sp_ReportEmployees`, the 1 raw SQL in AttendanceService, the 109 unchecked `Employees.IgnoreQueryFilters()` reads, by-id reads) + login guard | Deleted vanish app-wide |
| **4** ✅ | Inactivate API `POST /api/employees/bulk-inactivate` `{ ids[], effectiveDate, reasonCode, remarks }`. Gate = Super Admin, Developer or the action's permission (§1a); also switch `bulk-delete` to this gate | API + tests (§8d) |
| **5** ✅ | Reactivate API `POST /api/employees/{id}/reactivate` (single, because each case needs a check) + separation request marked Reversed. Gate = `EnsureCanAct(PermReactivate)` | API + tests (§8e) |
| **6** ✅ | Re-join API: `GET /api/employees/{id}/rejoin-draft` (pre-fill) + create with `RejoinOfEmployeeId`, duplicate-check exception | API + tests (§8f) |
| **7** | Employee List UI (ui.md): three tabs **Inactive Staff**, **Delete Staff**, **Reactivate / Re-joined Staff**, shown to Super Admin + Developer and to roles with the tab's permission (§1a). Each tab: row checkboxes, bulk button, Actions icons only, drawers in New Request design with pre-check list and per-row results. Re-join on the third tab → Add Employee pre-filled | Super Admin / Developer can use A–D |
| **8** | Developer **Deleted** tab on Employee List (code, name, company, location, deleted on/by, reason; restore icon + bulk Restore) + status history timeline in the Employee Dossier | Developer restore + history |
| **9** | E2E on a throwaway DB clone only: each action → effect on lists, dropdowns, attendance, payroll, login → reverse it → check it is all back. Final docs + artifact + task.xlsx | Ready for browser check |

## 7. Decisions (agreed 2026-10-22, all defaults accepted)

- **O1** — No approval; all actions take effect immediately.
- **O2** — Only Developer sees the Deleted tab (not Superadmin).
- **O3** — Anyone with `employees.inactivate` can set a past date (today or earlier).
- **O4** — Re-join gets a **new employee code** on a new record.
- **O5** — Inactive clean-up is done by selecting rows on Employee List. No Excel upload.
- **O6 (2026-10-22, user)** — Inactivate, Delete and Reactivate / Re-join are separate tabs for **Super Admin and Developer** (§1a).
- **O7 (2026-10-22, user, Step 4)** — Also permission based: another role can do an action when it has `employees.inactivate`, `employees.deleted` or `employees.reactivate`.

## 8. Step 0 audit result (2026-10-22, read-only)

**Current data (HRMSCore_Local2).** There are 3,879 employees: 3,878 active and 1 inactive. In `employment_details` there are 3,594 Confirmed, 281 Probation and
1 Separated/Resigned. **3 employees have no `employment_details` row**, so Inactivate / Reactivate must skip them with a message.
None of the new columns exist yet. Existing permissions under Employee List are `employees.view`, `view_all`, `create`, `edit`,
`export`, `import`, `transfer`, `transfer.view` and `transfer.export`.

**(a) Database code that reads `Employees`.** There are 32 objects. 20 of them already check `IsActive`, so they hide deleted employees once Delete sets
`IsActive = 0`. Step 3 still checks the reporting ones (`sp_ReportEmployees` can show inactive people on purpose).

| Group | Objects with **no** `IsActive` check (Step 3 looks at each) |
|---|---|
| Attendance | `sp_FlagUnauthorisedAbsence`, `sp_GetAttendanceAdjustments`, `sp_GetAttendanceExceptions`, `sp_GetAttendanceExceptionsPaged`, `sp_GetBiometricPunchesPaged`, `sp_GetLateArrivalWarnings`, `sp_GetTeamAttendanceSummary` |
| Other | `sp_GetTeamConfirmationStatus`, `sp_RunExitEncashment`, `fn_LeaveDayBreakdown`, `fn_LeavePayDays` |
| Leave as they are | `fn_EmployeeTimeZoneSql`, `fn_LocationTimeZoneSql` (one employee by id); `sp_AllocateEmployeeCodes` (must still see deleted codes so they are never given again) |

Raw SQL in C#: only one query reads `Employees` (in `AttendanceService.cs`).

**(b) EF includes and filter bypasses.**
- 90 `.Include(x => x.Employee / Manager / …)` calls across about 25 services. The busiest files are ApprovalsService 16, Letters 6, Leave 6,
  ComplianceMaster 6, BellRules 6 and AuthorizedSignatories 6. `Employee` already has a `CompanyFilter`, so EF already drops related rows when the
  employee is filtered out. `SoftDelete` behaves the same way, which means a deleted employee's history is hidden with them. Nothing new breaks.
- **Main finding:** `IgnoreQueryFilters()` is called without a filter name **1,216 times in 123 files**. Each call switches off all
  filters, including a new `SoftDelete` filter. 126 of these calls are on `Employees`, and **109** of those have no `IsActive` check nearby.
  This is why §5 now hides deleted employees in three layers instead of relying on one filter.

**(c) Where `IsActive` drives things.** About 47 services use the employee's `IsActive`, including Attendance, Roster, Shift, Payroll,
PayrollArrear, Leave, LeaveAccrual, Overtime, NightAllowance, CompOff, Approvals, ApprovalEngine, Task, Dashboard, Reports,
Notification, Auth and EmployeeMasterLookups. So an employee who is Inactive (or Deleted, because Delete sets `IsActive = 0`) drops out of all of them, the same way a separated employee does.
Status is held in two places, `employment_details.EmploymentStatus` (`CONFIRMED`/`PROBATION`/`NOTICE`/`SEPARATED`/
`TERMINATED`) and `employment_details.Status` (enum `ACTIVE`/`PROBATION`/`SUSPENDED`/`TERMINATED`/`RESIGNED`/`ON_LEAVE`).
Step 1 adds `INACTIVE` to both.

**(d) What Separation release changes** ([SeparationReleaseService.cs](../backend/Infrastructure/Services/SeparationReleaseService.cs)).
It cancels the employee's pending leave, marks the separation case Released, completes the letter task, sets EmploymentStatus `SEPARATED` and Status `RESIGNED`,
sets `Employees.IsActive = 0`, and turns the user login off (unless another active employee record shares the login or the user has a god-mode role).
It writes an audit row listing direct reports and stage mappings but **does not move them**. `AccessBlockedAt` is set in
SeparationExitRules, SeparationApprovalRules (2 places), AccessGateService and SeparationConfigService. Reactivate must undo the
status fields, `IsActive`, `AccessBlockedAt` and the login. It cannot bring back cancelled leave.

**(e) Add Employee duplicate checks.** *(Corrected in Step 6: there is an app check — `EmployeeCreateValidator` refuses a National ID, personal email, company work email or login email already on file, any status; see §8f.)* The service has no duplicate check in code; the database enforces it with unique indexes:
`UQ_Employee_NationalId` (National ID across all companies), `UQ_Employee_Company_WorkEmail`, `UQ_Employee_Company_EmpNum`
and `UX_Employees_UserId`; on users, `email` and `username`. **Re-join, and re-adding a wrongly joined person after Delete,
are blocked today by the National ID and work email indexes.** Step 1 turns them into filtered indexes.

**(f) Job history.** `employee_job_history` stores one changed field per row (FieldKey, From/To, EffectiveDate, ChangedByName, BatchId).
It has no action, reason code, remarks or saved `IsActive`, so the new `employee_status_history` table is still needed. Each action **also** writes
one `employee_job_history` row (FieldKey `EMPLOYMENT_STATUS`), so the change shows in the existing Job History on My Profile.

**Employee List UI today** ([EmployeeDirectoryView.tsx](../frontend/src/modules/employee-directory/EmployeeDirectoryView.tsx), 602 lines).
It has a status filter (`EMPLOYEE_STATUS_OPTIONS`), sorting and permission checks for `employees.export` and `employees.create`. There are **no row
checkboxes and no Actions column** yet, so Step 7 adds both.

## 8a. Step 1 result (2026-10-22) — schema live on HRMSCore_Local2

| Item | Done |
|---|---|
| `Employees` | `isDeleted bit NOT NULL DEFAULT 0`, `deletedOn datetime2`, `deletedBy nvarchar(450)` (user id), `rejoinOfEmployeeId nvarchar(36)`. Filtered indexes `IX_Employees_Deleted` (`isDeleted = 1`) and `IX_Employees_RejoinOf` |
| Unique indexes | `UQ_Employee_NationalId` (was a UNIQUE **constraint** on Local2, so the script drops a constraint or an index, whichever exists) and `UQ_Employee_Company_WorkEmail` are now filtered `isActive = 1 AND isDeleted = 0` (email also keeps `workEmail <> ''`). `UQ_Employee_Company_EmpNum` and `UX_Employees_UserId` unchanged |
| `employee_status_history` | As §3, plus a full audit trail per row: `batchId` (one bulk action), `reasonName` (as it was then), previous/new EmploymentStatus, Status, IsActive and IsDeleted, previous SeparationDate and AccessBlockedAt, `loginChange` (DISABLED / ENABLED / KEPT_SHARED / NO_LOGIN), `separationRequestId`, `rejoinEmployeeId`, `reversesHistoryId` (the row a Reactivate / Restore undoes), `warningsJson` (warnings the user accepted), `actionByUserId`, `actionByName`, `actionAt`, `ipAddress`. Check on `action`, FK to Employees |
| Append-only | Trigger `TR_esh_append_only` (INSTEAD OF UPDATE, DELETE) refuses every change or removal, so the history cannot be edited even by SQL. It does not depend on the Audit Settings switch. EF knows the trigger (`HasTrigger`) |
| Audit area | `EmployeeStatusHistory` added to the **EMPLOYEES** row of `audit_settings`. Steps 2–8 write both a history row and an `audit_log` row (EntityType `Employee`) plus a `employee_job_history` row (FieldKey `EMPLOYMENT_STATUS`) per employee |
| Status | `EmploymentStatuses.Inactive = "INACTIVE"`; `EmployeeStatus.INACTIVE` already existed |
| Reasons | Lookups `INACTIVE_REASON` (5), `REACTIVATE_REASON` (4) and **`DELETE_REASON` (4, added because Delete needs a reason and rule 4 forbids static values)**. `CK_lv_category` widened. New tabs on Masters → Employee Lookups |
| Permissions | `employees.inactivate`, `employees.reactivate`, `employees.deleted` under `menu-emp-list` (Workforce). Granted to **no** role (checked: 0 role rows) |
| EF | `Employee.IsDeleted / DeletedOn / DeletedBy / RejoinOfEmployeeId`, entity `EmployeeStatusHistory` + `EmployeeStatusActions`, `StatusLoginChanges`, `LookupCategories.InactiveReason / ReactivateReason / DeleteReason`, named filter `HrmsDbContext.SoftDeleteFilter` (`!e.IsDeleted`) |
| Tests | `EmployeeSoftDeleteFilterTests` (6): normal read hides deleted; `IgnoreQueryFilters([CompanyFilter])` still hides them; `IgnoreQueryFilters([SoftDeleteFilter])` shows them but keeps the company filter; Developer sees all companies' deleted; history row saves. Full suite 600/600; frontend tsc clean. On Local2, in a rolled-back transaction: history UPDATE and DELETE refused; an **active** duplicate National ID / email still refused; a copy with `isActive = 0` allowed; no test rows left |
| ~~Interim gap (until Step 6)~~ | **Corrected in Step 6:** there was no gap. Add Employee's own validator (`EmployeeCreateValidator` + `EmployeeMasterLookups`) already refused a National ID / email held by any record, of any status. Step 6 narrows it: deleted records no longer count, and Re-join may match only its old record (§8f) |
| SQL | `docs/db_changes.sql` block "Step 1"; ran twice on Local2 (second run changes nothing) |

## 8b. Step 2 result (2026-10-22) — Delete + Developer Restore API

| Item | Done |
|---|---|
| Files | `EmployeeStatusActionService` (Infrastructure), `IEmployeeStatusActionService`, `EmployeeStatusActionDtos`, `EmployeeStatusActionsController` (route `api/employees`), DI in Program.cs |
| `POST /api/employees/bulk-delete` | Policy `employees.deleted`. Body `{ ids[], reasonCode, remarks, dryRun }`. 1–200 distinct ids, reason must be an active `DELETE_REASON`, remarks ≤ 500. Per row: DONE / SKIPPED (+ blockers) / READY on dry run, warnings, `loginChange`. One `batchId` per call, one transaction |
| Pre-check (shared, reused by Step 4) | Blocks: own record (`UserId` = signed-in user), active not-deleted direct reports, open /task items assigned to the person (`RequestApprovalSteps`, `LeaveRequests` current approver, `SeparationTasks`, `RequisitionApprovals`, `OfferApprovals` with status PENDING, by employee id or user id), company outside `AllowedCompanyIds`, location outside `CanEditLocation`. Warns: own pending leave, open separation case, no login |
| Login | Same rule as Separation release, plus deleted records do not count as "another active record". God-mode role read with a separate query (no `Include(Role)`, which drops users whose role row is missing) |
| `GET /api/employees/deleted` | Developer only (`IsDeveloper`, else 403). Query `companyId`, `search`, `pageNumber`, `pageSize` (≤ 200). Uses `IgnoreQueryFilters([SoftDeleteFilter])`. Reason, remarks and "deleted by" name come from the latest DELETE history row |
| `POST /api/employees/bulk-restore` | Developer only. Body `{ ids[], remarks, dryRun }`. Restores the `PreviousIsActive` of the latest DELETE row (none → inactive + warning). Login back only if active again and the DELETE row says `DISABLED`. Blocks when an active, not-deleted record has the same National ID (any company) or work email (same company). History `reversesHistoryId` = the DELETE row |
| Writes per Done row | `employee_status_history`, `audit_log` (`EMPLOYEE_DELETED` / `EMPLOYEE_RESTORED`, EntityType `Employee`), `employee_job_history` (`EMPLOYMENT_STATUS`, status → "Deleted" and back) |
| Tests | `EmployeeStatusActionTests` (9): delete effects + history/audit/job history; dry run saves nothing; per-row blockers (self, reports, open task, not found); company outside access; shared login kept; bad reason / >200 / empty refused; Developer-only list + restore; restore with login back + `reversesHistoryId`; restore blocked by duplicate National ID. Full suite 609/609 |
| DB | No SQL change. Read-only check on Local2: `employee_status_history` has all 31 columns the entity maps, `DELETE_REASON` 4 rows, 0 deleted employees |
| **Do not use on real data before Step 3** | Screens that show inactive people on purpose (Employee List Inactive filter, reports, Dossier / Assignment by id) still show a deleted employee until Step 3 |

## 8c. Step 3 result (2026-10-22) — deleted employees hidden app-wide

| Item | Done |
|---|---|
| Unfiltered EF reads | `.Where(e => !e.IsDeleted)` added after **152** `Employees.IgnoreQueryFilters()` calls in ~60 files (lists, by-id reads for Dossier / Assignment / Profile, approvals, notifications, attendance, separation, onboarding, imports, Health). Named ignores such as `IgnoreQueryFilters([CompanyFilter])` already keep `SoftDelete`, so they needed nothing |
| `ScopeEmployees()` | Shared access-scope helper in HrmsDbContext now leaves out deleted rows |
| Left seeing deleted on purpose | `EmployeeCodeService` + `EmployeeMasterLookups` (codes stay reserved, import must not reuse a code), `JobHistoryRecorder`, `BlobPathResolver`, `LoginActiveFromSync`, 3 by-id reads in `EmployeeTimeZone` (the by-user one now hides deleted), the access-filter internals in HrmsDbContext, `EmployeeStatusActionService` |
| Login guard | `AuthService`: a user whose employee records are **all** deleted is refused ("This account is disabled") even if the login is still on; the employee-code sign-in path no longer finds deleted records. God-mode roles are never blocked |
| Raw SQL | The one query in `AttendanceService` (month leave CTE) gets `isDeleted = 0` |
| SQL objects (11) | `fn_LeavePayDays`, `sp_FlagUnauthorisedAbsence`, `sp_GetAttendanceAdjustments`, `sp_GetAttendanceExceptions`, `sp_GetAttendanceExceptionsPaged`, `sp_GetBiometricPunchesPaged`, `sp_GetLateArrivalWarnings`, `sp_GetTeamAttendanceSummary`, `sp_GetTeamConfirmationStatus`, `sp_ReportEmployees`, `sp_RunExitEncashment`. Inner joins get `e.isDeleted = 0`; lists that LEFT JOIN employees get `ISNULL(e.isDeleted, 0) = 0`, so punches with no mapped employee still show |
| SQL left as is | `fn_LeaveDayBreakdown`, `fn_EmployeeTimeZoneSql`, `fn_LocationTimeZoneSql` (one employee by id), `sp_AllocateEmployeeCodes` (must see deleted codes) |
| Tests | `EmployeeSoftDeleteHidingTests` (3): `ScopeEmployees` hides deleted; deleted employee refused by username and by code; **source guard** fails the build if new code adds a bare `Employees.IgnoreQueryFilters()` without `IsDeleted` outside the allow-list. Full suite 612/612 |
| DB | `docs/db_changes.sql` block "Step 3", ran on HRMSCore_Local2 (CREATE OR ALTER, re-runnable). Rolled-back check: `sp_ReportEmployees` shows an employee, hides them once `isDeleted = 1`; 0 deleted rows left after |
| Known leftovers | Related rows (leave, attendance) of a deleted employee are hidden where the query joins the employee; screens that read those tables without the employee may still count them (same as for a separated employee). Manager / HRBP name of a deleted person can still show on someone else's record (Delete blocks people with direct reports, so this is rare) |

## 8d. Step 4 result (2026-10-22) — Inactivate API + access gate

| Item | Done |
|---|---|
| `POST /api/employees/bulk-inactivate` | Body `{ ids[], effectiveDate, reasonCode, remarks, dryRun }`. 1–200 ids; reason must be an active `INACTIVE_REASON`; remarks required (≤ 500); effective date required. Per row DONE / SKIPPED / READY with blockers and warnings, one `batchId`, one transaction |
| Row rules | Skipped when: not found / deleted; company or location outside access; no `employment_details` row; not active or status not Probation / Confirmed (Notice → "use Separation"); effective date after the company's today or before the joining date; shared pre-check (self, active direct reports, open /task items). Warns: own pending leave, open separation case, no login, **payroll already finalised** (Approved / Locked / Archived run on or after the effective month with a payslip for the person) |
| Effect | `employment_details`: EmploymentStatus `INACTIVE`, Status `INACTIVE`, `SeparationDate` = effective date. `Employees`: `IsActive = 0`, `AccessBlockedAt` (kept if already set). Login off with the Separation-release rule (`KEPT_SHARED` when another active record shares it or god-mode role) |
| Writes | `employee_status_history` (action INACTIVATE, effective date, reason code + name, remarks, previous status / IsActive / SeparationDate / AccessBlockedAt, login change, accepted warnings), `audit_log` `EMPLOYEE_INACTIVATED`, `employee_job_history` EMPLOYMENT_STATUS (old status → INACTIVE, dated the effective date) |
| Access gate | `EnsureCanAct`: Super Admin, Developer, or the permission (`employees.inactivate` / `employees.deleted`; `employees.reactivate` ready for Step 5). `bulk-delete` lost its `[Authorize(Policy = "employees.deleted")]` and uses the same gate. Deleted list / Restore stay Developer only |
| Tests | 4 new in `EmployeeStatusActionTests`: Inactivate effects + history/audit/job history; skipped rows (notice, manager, no details, self, missing, future date, before joining) and refused requests (wrong reason, no remarks, no date); finalised payroll warning saved in history; access matrix (no permission → 403 on both; each permission opens only its action; Super Admin and Developer with no permissions can do both). Full suite 616/616 |
| DB | No SQL change (permissions, lookups and columns came in Step 1) |

## 8e. Step 5 result (2026-10-22) — Reactivate API

| Item | Done |
|---|---|
| `POST /api/employees/{id}/reactivate` | Body `{ reasonCode, remarks, dryRun }`. Reason must be an active `REACTIVATE_REASON`; remarks optional (≤ 500). Returns the same result shape as the bulk actions with one row (DONE / SKIPPED / READY, blockers, warnings, `loginChange`). Unknown id → 404. Gate `EnsureCanAct(employees.reactivate)` |
| Row rules | Skipped when: company or location outside access; no `employment_details` row; employee still active or status not `INACTIVE` / `SEPARATED`; own record; an active, not-deleted record now has the same National ID (any company) or work email (same company). Deleted employees are not reachable (Restore is the Developer path) |
| Status restored | **Inactive:** from the latest INACTIVATE history row not already undone (`PreviousEmploymentStatus` + `PreviousStatus`); `reversesHistoryId` = that row. **Separated:** from the latest RELEASED case — `IsProbationResignation` → Probation, else Confirmed; Status `ACTIVE` (release saves no earlier status, and the notice period sat between). No history / case (changed outside the app) → Confirmed + warning |
| Effect | `employment_details`: status as above, `SeparationDate` and `LastWorkingDay` cleared. `Employees`: `IsActive = 1`, `AccessBlockedAt` cleared. Login: turned back on, except after an Inactivate that left it on (`KEPT_SHARED` / `NO_LOGIN` in history) |
| Separation case | New status **`REVERSED`** (`SeparationApprovalRules.Reversed`); `CurrentStage` cleared; WAITING / PENDING tracker tasks → SKIPPED ("Separation reversed"), so they leave /task; trail row `RESIGNATION:REVERSED` with the reason. Nothing removed. `REVERSED` added to the closed lists (SeparationService — the employee can resign again; status-action pre-check), the separation export label and the screen badge / trail label |
| Warns only | Case will be marked Reversed; leave CANCELLED with an end date on or after the resignation date is not brought back (listed by type and dates); final settlement already PAID (amount, date) |
| Writes | `employee_status_history` (REACTIVATE, reason, previous values, `separationRequestId`, `reversesHistoryId`, login change, warnings), `audit_log` `EMPLOYEE_REACTIVATED`, `employee_job_history` EMPLOYMENT_STATUS (INACTIVE / SEPARATED → restored status) |
| Tests | 4 new in `EmployeeStatusActionTests`: Inactivate → Reactivate round trip (status, login, history link, audit, job history; second Reactivate skipped); release → Reactivate (Probation back, case Reversed, open task skipped, done task kept, trail row, leave stays cancelled, warnings for leave + paid settlement, dry run saves nothing); skipped / refused cases (active, no details, duplicate National ID, self, outside access, unknown id, wrong reason; no-history fallback warning); access matrix. Full suite 620/620; frontend tsc clean |
| DB | No SQL change (`separation_requests.status` has no check constraint on Local2) |
| Note | The "cancelled at release" list is by date (cancelled leave ending on or after the resignation date), because release does not record which leave it cancelled |

## 8f. Step 6 result (2026-10-22) — Re-join API

| Item | Done |
|---|---|
| `GET /api/employees/{id}/rejoin-draft` | Returns `RejoinDraftDto`: old record facts (code, name, status, hire / separation / last working day, login username), `blockers`, `warnings`, `draft` (a `RejoinEmployeeRequest` = the Add Employee body) and the old record's `documents`. Saves nothing. Unknown or deleted id → 404 |
| Pre-fill | Person, ID, emails, phones, addresses, contacts, time zone; last job (department, location, extra locations, job title, grade, cost centre, employment type, contract type, shift, work schedule); salary + statutory numbers; primary bank; education (certificate keys kept). Joining date = company today, or the day after the old last day. Probation days from the employment type. `EmployeeCode` empty (series). Manager / HRBP / Direct HR not active → blank + warning. Expired ID / documents → warning. `CreateLogin` on; `RoleId` = old login's role when the signed-in user may give it, else default + warning. All documents in `linkDocumentIds` |
| `POST /api/employees/{id}/rejoin` | Body `RejoinEmployeeRequest` (Add Employee body + `remarks` + `linkDocumentIds`). Runs the full Add Employee path (pick lists, login options, `EmployeeCreateValidator`, code series, leave opening) and returns `RejoinResultDto` (new record + credentials + `employeeNumber`, `loginChange`, `linkedDocuments`). Validation errors come back as a list (400) |
| Gate | `EnsureCanAct(employees.reactivate)` **and** Super Admin / Developer / `employees.create` |
| Blocks | Outside company / location access; no `employment_details`; still active or status not `INACTIVE` / `SEPARATED`; own record; already re-joined (a not-deleted record has `RejoinOfEmployeeId` = this id — a re-joined person who leaves again re-joins from the newer record); new joining date on or before the old last working day / separation date |
| Duplicate exception | `EmployeeMasterLookups.LoadAsync(…, rejoinOf)`: National ID, personal email and company work email of **deleted** records and of the **re-join source** are not "already used"; the source's login is left out of the username / login-email sets. Every other record still blocks. Codes stay reserved (deleted included) |
| Login (decision, this step) | `users.email` is unique, so a second login with the same email is impossible. With `CreateLogin` on and an old login present, the old record's `UserId` is cleared (saved first, because `UX_Employees_UserId` is unique) and `EmployeeAccountService.MoveLoginAsync` gives the same `users` row to the new record: username per the form (new code / work email), new password (auto or manual), role, activation, force change, unlocked, login email follows the new personal email when free, company grant added if missing. Old record without a login → a new login as in Add Employee. `CreateLogin` off → old login stays (disabled) on the old record |
| Documents | Chosen documents of the old record get new `employee_documents` rows on the new record pointing at the same `storageKey` (note "Linked on re-join from …"); a certificate already added from education is not added twice. Old rows untouched |
| Writes | New record as Add Employee (`EMPLOYEE_CREATED`, `LOGIN_CREATED` or `LOGIN_MOVED_ON_REJOIN`, leave opening) with `rejoinOfEmployeeId`; on the old record: `employee_status_history` REJOIN (effective = new joining date, remarks, `rejoinEmployeeId`, `separationRequestId` of the released case, `loginChange` MOVED / CREATED / NO_LOGIN), `audit_log` `EMPLOYEE_REJOINED`, `employee_job_history` EMPLOYMENT_STATUS (status → "Re-joined as {code}"). One transaction; credentials email after commit |
| Files | `EmployeeStatusActionService.Rejoin.cs` (new, class now partial), `EmployeeService.Create.cs` (`CreateRejoinEmployeeAsync`; own transaction only when none is open), `EmployeeAccountService.MoveLoginAsync`, `EmployeeMasterLookups`, DTOs, controller, `StatusLoginChanges.Moved / Created` |
| Tests | `EmployeeRejoinTests` (4): duplicate rules (deleted ignored, only the source ignored, codes still reserved); draft → re-join end to end (pre-fill, new code, link, login moved, one user per email, document linked, history / audit / job history, second draft blocked); refusals (active, deleted / unknown 404, joining date overlap, someone else's National ID → validation list, nothing saved); access matrix. Full suite 624/624 |
| DB | No SQL change (`rejoinOfEmployeeId`, REJOIN action already in Step 1; `loginChange` has no check constraint — checked on Local2) |
| Follow-up | Re-adding a **deleted** person through plain Add Employee is still refused by "personal email already used by a login account" when their old login holds that email (`users.email` unique). Options for later: Developer Restore, or move the login as Re-join does |

## 9. Prompts for each step

- **Step 0:** "Read docs/employee_soft_delete.md and do Step 0 only (read-only audit). Fill §8. No code changes."
- **Steps 1–9:** "Read docs/employee_soft_delete.md and do Step N only."
