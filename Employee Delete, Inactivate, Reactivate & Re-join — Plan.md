# Employee Delete, Inactivate, Reactivate & Re-join — Plan

Branch: `employee-bulk-soft-delete` · Created 2026-10-22 · Status: **Step 0 done 2026-10-22 · Step 1 next**

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
| **Re-join** | Only Separated / Inactive rows. Opens Add Employee pre-filled (personal, ID, bank, education, documents linked) with a **new employee code** and a new joining date. Needs the unique-index change in Step 1 (National ID and work email are unique today). The app check allows a match only with the old Separated / Inactive record. Leave and service start fresh. Sets `RejoinOfEmployeeId`. The new code gives a new login (username = code) |
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
| **4** | Inactivate API `POST /api/employees/bulk-inactivate` `{ ids[], effectiveDate, reasonCode, remarks }` | API + tests |
| **5** | Reactivate API `POST /api/employees/{id}/reactivate` (single, because each case needs a check) + separation request marked Reversed | API + tests |
| **6** | Re-join API: `GET /api/employees/{id}/rejoin-draft` (pre-fill) + create with `RejoinOfEmployeeId`, duplicate-check exception | API + tests |
| **7** | Employee List UI (ui.md): row checkboxes, bulk bar (**Inactivate** / **Delete**), Actions icons only (inactivate, reactivate, bin), drawers in New Request design with pre-check list and per-row results. Re-join button on separated rows → Add Employee pre-filled | HR can use A–D |
| **8** | Developer **Deleted** tab on Employee List (code, name, company, location, deleted on/by, reason; restore icon + bulk Restore) + status history timeline in the Employee Dossier | Developer restore + history |
| **9** | E2E on a throwaway DB clone only: each action → effect on lists, dropdowns, attendance, payroll, login → reverse it → check it is all back. Final docs + artifact + task.xlsx | Ready for browser check |

## 7. Decisions (agreed 2026-10-22, all defaults accepted)

- **O1** — No approval; all actions take effect immediately.
- **O2** — Only Developer sees the Deleted tab (not Superadmin).
- **O3** — Anyone with `employees.inactivate` can set a past date (today or earlier).
- **O4** — Re-join gets a **new employee code** on a new record.
- **O5** — Inactive clean-up is done by selecting rows on Employee List. No Excel upload.

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

**(e) Add Employee duplicate checks.** The service has no duplicate check in code; the database enforces it with unique indexes:
`UQ_Employee_NationalId` (National ID across all companies), `UQ_Employee_Company_WorkEmail`, `UQ_Employee_Company_EmpNum`
and `UX_Employees_UserId`; on users, `email` and `username`. **Re-join, and re-adding a wrongly joined person after Delete,
are blocked today by the National ID and work email indexes.** Step 1 turns them into filtered indexes.

**(f) Job history.** `employee_job_history` stores one changed field per row (FieldKey, From/To, EffectiveDate, ChangedByName, BatchId).
It has no action, reason code, remarks or saved `IsActive`, so the new `employee_status_history` table is still needed. Each action **also** writes
one `employee_job_history` row (FieldKey `EMPLOYMENT_STATUS`), so the change shows in the existing Job History on My Profile.

**Employee List UI today** ([EmployeeDirectoryView.tsx](../frontend/src/modules/employee-directory/EmployeeDirectoryView.tsx), 602 lines).
It has a status filter (`EMPLOYEE_STATUS_OPTIONS`), sorting and permission checks for `employees.export` and `employees.create`. There are **no row
checkboxes and no Actions column** yet, so Step 7 adds both.

## 9. Prompts for each step

- **Step 0:** "Read docs/employee_soft_delete.md and do Step 0 only (read-only audit). Fill §8. No code changes."
- **Steps 1–9:** "Read docs/employee_soft_delete.md and do Step N only."
