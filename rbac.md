# Users, Roles, Menus & Permissions (RBAC) — Findings & Fix Plan

Pages in scope: `/user`, `/rolemanagemnt`, `/menumanagement`, `/permissionmanagemnt`
Analysis date: 2026-10-07 (live DB `HRMSCore_Local`, read-only queries)

### Follow the instructions of 'docs/follow.md' file

## Progress

| Step | Title | Status |
|---|---|---|
| 0 | Backup (DB + RBAC table snapshots) | Done 2026-10-07 |
| 1 | Menu data cleanup | Done 2026-10-07 |
| 2 | Permission data cleanup | Done 2026-10-07 |
| 3 | Link menus ↔ permissions (FKs) | Done 2026-10-07 |
| 4 | Backend: policies, dynamic Task menu, God-mode flag | Not started |
| 5 | Frontend: route guard, delete duplicate routes/views | Not started |
| 6 | UI: role grants by menu tree, merge Permission page into Menus | Not started |
| 7 | Full re-test (follow.md #12) | Not started |

## 0. Decisions (answered 2026-10-07)

- D1 — Back up first so we can restore in the worst case (Step 0).
- D2 — Model: **menu → controller → permissions** (§3). Approved.
- D3 — Four pages become three: Users, Roles, Menus & Permissions (developer only). Approved.
- D4 — All approvals live on `/task`. The Task menu must be **dynamic** (DB-driven), not created or special-cased in code.
- D5 — Each step = `docs/db_changes.sql` script + `tsc --noEmit` + `dotnet build` (follow.md #11, #15).

## 1. Current state (counts from live DB)

- `menus`: 107 rows (103 active, 100 visible). `permissions`: 157 rows in 25 free-text modules. `role_permissions`: 860 rows.
- Visibility logic: [MenuService.cs](../backend/Infrastructure/Services/MenuService.cs) `GetAuthorizedNavigationAsync` — role → `role_permissions` → menu `permissionCode`, else falls back to a `permissionModule` string match.
- API enforcement: `[Authorize(Policy = "<code>")]` on 48 of 50 controllers via `PermissionPolicyProvider`.
- Page enforcement: **none**. [AuthGuard.tsx](../frontend/src/components/layout/AuthGuard.tsx) only checks login.

## 2. Findings

### Pages
- F1 — Each admin page is routed twice: `menumanagemnt`/`menumanagement`, `rolemanagemnt`/`rolemanagement`, `permissionmanagemnt`/`permissionmanagement`. Older copies too: `/admin/users`, `/admin/roles`, `/admin/roles/create`, `/admin/users/create`. The sidebar links the misspelled ones.
- F2 — Menu `permissionCode` is a free-text input in `CreateMenuDrawer`/`EditMenuDrawer`. A typo hides the menu from everyone (`task.view` is on a menu but not in `permissions`).
- F3 — `menus.permissionModule` values do not match `permissions.module` ("leave" vs "Leave Management", "Letters" vs "HR Letters", "System" vs "Administration", "Masters", "reports", "Documents"), so the module fallback silently fails.
- F4 — Role grants are a flat list of codes; the screen does not show which menu/page each code opens.

### Permissions vs controllers
- F5 — 55 permissions are checked nowhere (no controller, menu or UI):
  `attendance.dayrequest.hr attendance.edit attendance.regularisation.hr attendance.regularise attendance.team audit.export backup.manage banks.manage companies.duplicate costcentres.manage currencies.view departments.manage dev.maintenance_mode documents.verify employees.bank.view employees.delete employees.salary.edit employees.salary.view employees.terminate employees.transfer employmenttypes.manage grades.manage groups.manage groups.realign groups.view jobtitles.manage leave.approval leave.cancel leave.policies.manage leave.proof.waive leave.team letters.signatories.manage letters.signatories.upload loans.manage locations.manage payroll.config.manage payroll.edit payroll.process permissions.create permissions.delete permissions.edit permissions.manage recruitment.post recruitment.vacancy_notify reporting.dashboard reports.export requisitions.approve roles.create roles.delete roles.edit roles.view settings.view shifts.assign shifts.overrides.manage users.create users.delete users.edit users.view`
  Example: [AdminController.cs](../backend/Api/Controllers/AdminController.cs) only checks `roles.manage` / `users.manage`, so ticking `roles.create` does nothing.
- F6 — 10 permissions control menus/UI but no controller checks them: `portal.attendance.view portal.dashboard.view portal.leave.view portal.me.view payroll.payslips.view permissions.view dashboard.employees.view attendance.approve leave.types.manage letters.signatories.view reports.view`.
- F7 — Menu permission ≠ API permission: `SYS_PERM` menu needs `permissions.view`, its API needs `roles.manage` (menu shows, page fails). Reports uses both `reports.view` and `reporting.view`.
- F8 — **Security**: [MenusController.cs](../backend/Api/Controllers/MenusController.cs) create/update/delete/reorder/toggle have no policy — any logged-in user can edit the sidebar. `AuthorizedSignatoriesController` has no policy.
- F9 — Hidden menu ≠ blocked page: typing a URL opens any page shell.

### Task / approvals (D4)
- F10 — [MenuService.cs:87-115](../backend/Infrastructure/Services/MenuService.cs#L87-L115) inserts the TASK menu from code (hardcoded id, icon, sort, `task.view`) on every nav GET.
- F11 — TASK visibility is special-cased in code (`Code == "TASK"` / route `/task` / module `Task`) and unlocks on "has direct reports" or `approvals.view`. It ignores the actual approvers in `approval_matrices.approverSequenceJson`, `requisition_approvals`, `offer_approvals`.
- F12 — Old approval menus `APPROVALS`, `APPROVALS_MATRIX` (both `/approvals`, hidden) and the `/approvals` page are superseded by `/task`.

### Menus — relevant / irrelevant
- F13 — Duplicates: `LEAVE_MANAGEMENT` = `LEAVE_APPS` (`/leave`); `APPROVALS` = `APPROVALS_MATRIX` (`/approvals`); `REP_WORKFORCE` = `ONBOARD_REPORTS` (`/reports`); `LEAVE_HOLIDAYS` = `MST_LEAVE_HOLIDAYS` (`/leave-holidays`).
- F14 — Empty parents: `REQUISITIONS`, `ONBOARDING`, `DOCS_COMPLIANCE`. `PERFORMANCE` parent inactive but children active.
- F15 — Menus that open the "Under construction" catch-all: `/employees/add`, `/employees/details`, `/employees/bulk-upload`, `/leave/apply`, `/onboarding/configuration`, `/performance/reviews`, `/performance/goals`, `/training/programs`.
- F16 — Inactive leftovers: `MST_SRF_MATRICES`, `ONBOARD_CONFIG`.
- F17 — Wrong place: `SYS_USER` under Administration while Roles/Menus/Permissions are under System Configuration; `SYS_AUDIT`, `SYS_SETTINGS` under Developer Studio.
- F18 — Typo: "Attandance Policies" (`MST_SHIFT_GRACE`).
- F19 — Live pages with no menu: `/shifts/roster` (roster go-live). To check: `/me`, `/shifts/lookup`, `/shifts/master`, `/shifts/schedules`, `/companysetup/routing`.
- F20 — Duplicate pages to delete: `/authorizedsignatories` (= `/authorized-signatories`), `/letter` (= `/letters`), the `*managemnt` routes, `/admin/users*`, `/admin/roles*`.

### > Code correction and static values (follow.md #10)
- S1 — Hardcoded role ids/names: [permissions.ts](../frontend/src/lib/permissions.ts) (`role_dev`, `DEVELOPER`, `role_super`, `"Super Admin"`…) and `MenuService` God-mode check. → `roles.isGodMode` column.
- S2 — TASK menu row built in code (F10).
- S3 — `CreateMenuDrawer` defaults `permissionModule: "Common"`, and "Common" menus are shown to everyone.
- S4 — Free-text permission code/module inputs (F2, F3).

## 3. Target design

```
menus                      one row per screen; Parent rows are only groups
  permissionId  (FK)       the screen's *.view permission (Child rows: required)
permissions
  menuId        (FK)       the screen this action belongs to (replaces free-text module)
role_permissions           unchanged
roles.isGodMode (bit)      replaces hardcoded developer ids
```

Rules:
1. One controller = one permission prefix = one menu. `LeaveController` → `leave.*` → Leave menus.
2. Every `[Authorize(Policy)]` code must exist in `permissions`; every permission must belong to a menu and be checked by an endpoint (or be deleted).
3. Child menu visible ⇔ role has its view permission. Parent visible ⇔ any child visible. No module fallback, no "Common".
4. Page allowed ⇔ its route is in the user's `/api/menus/nav`; else a 403 view.
5. **Task menu (D4)** — an ordinary DB row with `task.view`, visible when:
   the role has `task.view`, **or** the user (or the user's role) is an approver in any active `approval_matrices.approverSequenceJson`, **or** the user has an open row in a pending approval table (`requisition_approvals`, `offer_approvals`, leave/attendance approvals).
   All read from the DB — no code-created menu, no `Code == "TASK"` check, no "has direct reports" guess. New approval types only need data, not code.
6. Screens: **Users** (assign role) · **Roles** (grant on a menu tree, each menu's actions as checkboxes) · **Menus & Permissions** (developer only; permissions edited in each menu's drawer). All three under one "Access Control" parent.

## 4. Implementation steps (one approval / PR each)

### Step 0 — Backup (before any change)
1. Full DB backup:
   `BACKUP DATABASE HRMSCore_Local TO DISK = '/var/opt/mssql/data/HRMSCore_Local_before_rbac.bak' WITH INIT, COMPRESSION, CHECKSUM;`
   then `RESTORE VERIFYONLY FROM DISK = '...before_rbac.bak'`, and `docker cp sqlserver:/var/opt/mssql/data/HRMSCore_Local_before_rbac.bak docs/`.
2. Quick-restore snapshots: `SELECT * INTO bak_rbac_menus / bak_rbac_permissions / bak_rbac_role_permissions / bak_rbac_roles FROM ...`.
3. Write the restore script (both options) at the top of the Step 0 notes in §7.
Done when: `.bak` verified and copied, 4 snapshot tables have the same row counts as the live tables (107 / 157 / 860 / roles).

### Step 1 — Menu data cleanup (F13–F19)
Delete or merge F13–F16 rows (re-point any `role` data first); fix `SYS_ROLE`/`SYS_PERM` routes to the correct spelling; fix "Attendance Policies"; move menus per F17; add a Roster menu (`/shifts/roster`, `shifts.roster.view`); decide the "to check" pages in F19. Script in `db_changes.sql`.

### Step 2 — Permission data cleanup (F5–F7)
Add `task.view`. Merge `reports.view` → `reporting.view` (move `role_permissions` first). For each F5 code: wire it into its controller (e.g. `roles.create/edit/delete`, `users.create/edit/delete` on `AdminController`) or delete it with its `role_permissions`. For each F6 code: add the policy to the matching endpoint. Align `SYS_PERM` with its API.

### Step 3 — Link menus ↔ permissions (F2–F4)
Add `permissions.menuId` and `menus.permissionId` (FK, nullable for Parent rows), backfill from the controller → menu map, then retire `permissions.module` / `menus.permissionModule` / `menus.permissionCode` from the code (keep the columns until Step 7 passes).

### Step 4 — Backend (F8, F10, F11, S1, D4)
- Policies on `MenusController` writes (developer) and `AuthorizedSignatoriesController`.
- Remove the TASK auto-insert and the TASK special case; implement rule 5 of §3 in `MenuService` (stored proc `sp_*` if the approver lookup gets complex — follow.md #3).
- `roles.isGodMode`; God-mode checks read it (backend + session payload).
- Nav: parent visible only when a child is.

### Step 5 — Frontend guard & cleanup (F1, F9, F20, S1)
Route guard in the dashboard layout using the nav response (403 view). `permissions.ts` uses `isGodMode` from the session. Delete duplicate routes and old views (`RolesView`, `RoleWizardView` if unused, `/admin/users*`, `/admin/roles*`, `*managemnt`, `/authorizedsignatories`, `/letter`, `/approvals`, `/me`). Guard must allow sub-routes of a granted menu route (tabs: `/shifts/lookup|master|schedules|roster`, `/companysetup/routing`). Remove the hardcoded `NAV_SECTIONS` fallback and the `OVERVIEW` special case in `Sidebar.tsx` (static values, found in Step 1).

### Step 6 — UI (F2, F4, D3)
Roles: grant on a menu tree with action checkboxes. Menus & Permissions: permission select (no free text), permissions tab inside each menu drawer; remove the Permission Management page and menu. Users / Roles / Menus under one "Access Control" parent.

### Step 7 — Full re-test (follow.md #12)
Log in as developer, superadmin, HR, line manager, employee: sidebar = granted menus only; typed URL of a non-granted page → 403; API returns 403 for the same; Task menu appears for a matrix approver with no `task.view`, and disappears when removed from the matrix. Then drop the retired columns and `bak_rbac_*` tables.

## 5. Open risks
- Removing F5 permissions changes what existing roles hold — list role grants per code before deleting.
- Superadmin currently relies on `role_permissions` only; check its grants after Step 2 so it does not lose admin pages.
- `approverSequenceJson` shape must be confirmed before writing the Task lookup (Step 4).

## 6. Per-step prompts (start each in a fresh session)

### NOTE: EVERYTHING MUST BE DYNAMIC.

- **Step 0:** "Read docs/rbac_fixes.md. Do Step 0 (backup) exactly as written, verify it, record the restore script and counts in §7, update Progress."
- **Step 1:** "Read docs/rbac_fixes.md. Confirm Step 0 is done. Do Step 1 (menu cleanup): show me the delete/merge/move list with role impact first, then write the script in db_changes.sql, apply, record in §7."
- **Step 2:** "Read docs/rbac_fixes.md. Do Step 2 (permission cleanup): per F5/F6 code, show wire-or-delete with role impact, then implement, dotnet build, record in §7."
- **Step 3:** "Read docs/rbac_fixes.md. Do Step 3 (menu ↔ permission FKs + backfill), update EF entities/configs, dotnet build, record in §7."
- **Step 4:** "Read docs/rbac_fixes.md. Do Step 4 (backend policies, dynamic Task menu per §3 rule 5, roles.isGodMode). Confirm approverSequenceJson shape first. dotnet build, record in §7."
- **Step 5:** "Read docs/rbac_fixes.md. Do Step 5 (route guard, isGodMode in permissions.ts, delete duplicate routes/views). tsc --noEmit, record in §7."
- **Step 6:** "Read docs/rbac_fixes.md. Do Step 6 (Roles menu-tree grants, permissions inside Menu drawer, remove Permission page, Access Control group). tsc + dotnet build, record in §7." 
- **Step 7:** "Read docs/rbac_fixes.md. Do Step 7 re-test on a DB clone across all roles, then drop retired columns and bak_rbac_* tables after my go-ahead."

## 7. Completion notes
_(one subsection per step: date, what changed, scripts, counts, how to restore)_

### Step 0 — Backup (2026-10-07)

**Restore script**

Option A — full database (worst case; replaces everything in the DB, including data added after the backup):
```sql
USE master;
ALTER DATABASE HRMSCore_Local SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
RESTORE DATABASE HRMSCore_Local FROM DISK = '/var/opt/mssql/data/HRMSCore_Local_before_rbac.bak' WITH REPLACE, CHECKSUM;
ALTER DATABASE HRMSCore_Local SET MULTI_USER;
```
If the file is missing from the container: `docker cp docs/HRMSCore_Local_before_rbac.bak sqlserver:/var/opt/mssql/data/`.

Option B — RBAC tables only (other data stays as it is). All four tables use identity keys, so columns are listed and `IDENTITY_INSERT` is used. Roles are referenced by users, so they are reset in place, not deleted. If a later step adds a column (e.g. `roles.isGodMode`), it keeps its current value. Dry-run tested on 2026-10-07 inside a rolled-back transaction (107 / 156 / 860 / 15).
```sql
USE HRMSCore_Local;
SET XACT_ABORT ON;
BEGIN TRAN;
DELETE FROM role_permissions;
UPDATE menus SET parentId = NULL;
DELETE FROM menus;
DELETE FROM permissions;

SET IDENTITY_INSERT permissions ON;
INSERT INTO permissions (id, code, module, name, description, updatedAt, createdAt, sno, actionType, isActive, isSystem, category)
SELECT id, code, module, name, description, updatedAt, createdAt, sno, actionType, isActive, isSystem, category FROM bak_rbac_permissions;
SET IDENTITY_INSERT permissions OFF;

SET IDENTITY_INSERT menus ON;
INSERT INTO menus (sno, id, name, code, icon, parentId, route, permissionModule, permissionCode, type, sortOrder, isActive, isVisible, createdAt, updatedAt)
SELECT sno, id, name, code, icon, parentId, route, permissionModule, permissionCode, type, sortOrder, isActive, isVisible, createdAt, updatedAt FROM bak_rbac_menus;
SET IDENTITY_INSERT menus OFF;

-- roles are referenced by users, so they are not deleted: re-add missing ones and reset the rest
SET IDENTITY_INSERT roles ON;
INSERT INTO roles (id, name, description, scope, isSystem, createdAt, updatedAt, sno, code, roleType, roleLevel, parentRoleId, isActive, isMultiUser)
SELECT b.id, b.name, b.description, b.scope, b.isSystem, b.createdAt, b.updatedAt, b.sno, b.code, b.roleType, b.roleLevel, b.parentRoleId, b.isActive, b.isMultiUser
FROM bak_rbac_roles b WHERE NOT EXISTS (SELECT 1 FROM roles r WHERE r.id = b.id);
SET IDENTITY_INSERT roles OFF;
UPDATE r SET r.name = b.name, r.description = b.description, r.scope = b.scope, r.isSystem = b.isSystem, r.code = b.code,
       r.roleType = b.roleType, r.roleLevel = b.roleLevel, r.parentRoleId = b.parentRoleId, r.isActive = b.isActive, r.isMultiUser = b.isMultiUser, r.updatedAt = b.updatedAt
FROM roles r JOIN bak_rbac_roles b ON b.id = r.id;

SET IDENTITY_INSERT role_permissions ON;
INSERT INTO role_permissions (id, roleId, permissionId, createdAt, updatedAt, sno)
SELECT id, roleId, permissionId, createdAt, updatedAt, sno FROM bak_rbac_role_permissions;
SET IDENTITY_INSERT role_permissions OFF;
COMMIT;
```

**What was done**
- `BACKUP DATABASE ... WITH INIT, COMPRESSION, CHECKSUM` → 26,650 pages. `RESTORE VERIFYONLY ... WITH CHECKSUM` → "The backup set on file 1 is valid."
- Copied to `docs/HRMSCore_Local_before_rbac.bak` (52 MB, ignored by `.gitignore` `*.bak` — not committed).
- Snapshot tables created; script logged in `docs/db_changes.sql` (RBAC Step 0 block, idempotent).

**Counts (live = snapshot)**

| Table | Live | `bak_rbac_*` |
|---|---|---|
| menus | 107 | 107 |
| permissions | 156 | 156 |
| role_permissions | 860 | 860 |
| roles | 15 | 15 |

Note: permissions is 156, not 157 as written in §1. Use 156 as the baseline for Steps 1–7.

### Step 1 — Menu data cleanup (2026-10-07)

Script: `docs/db_changes.sql` → "RBAC Step 1" block (one transaction, idempotent; re-run changed 0 rows). Restore: Step 0 Option B.

**Deleted (19)** — no role lost access to a working page:
- Duplicates (F13/F12): `LEAVE_MANAGEMENT`, `APPROVALS`, `APPROVALS_MATRIX` (hidden), `ONBOARD_REPORTS` (all `reports.view` roles also hold `reporting.view`, used by `REP_WORKFORCE`), `LEAVE_HOLIDAYS` (its `leave.types.manage` roles all hold `leave.manage`, used by the kept `MST_LEAVE_HOLIDAYS`).
- Empty/inactive parents (F14): `REQUISITIONS`, `ONBOARDING`, `DOCS_COMPLIANCE`, `PERFORMANCE`, `TRAINING`.
- Under construction (F15): `PERF_REV`, `PERF_GOALS`, `TRAIN_PROG`, `EMP_ADD`, `EMP_DETAILS`, `EMP_BULK` (add/bulk live in `/employeedirectory`, details at `/employees/[id]`), `LEAVE_APPLY` (staff use `/portal/leave`).
- Inactive (F16): `MST_SRF_MATRICES`, `ONBOARD_CONFIG`.

**Moved (F17)** to System Configuration: `SYS_USER` (from Administration), `SYS_AUDIT`, `SYS_SETTINGS` (from Developer Studio). Order: Users 1, Roles 2, Menus 3, Permissions 4, Audit 5, Settings 6. No visibility change — `users.manage`, `audit.view`, `settings.manage` are held by the same roles (Admin, Developer, Super Admin).

**Fixed:** `SYS_ROLE` → `/rolemanagement`, `SYS_PERM` → `/permissionmanagement` (F1); "Attendance Policies" (F18); Masters sort order (Approval Matrices 11, Deadlines 12, Country 13).

**Added (F19):** `ATT_ROSTER` "Staff Roster" under Attendance, `/shifts/roster`, `shifts.roster.view` (8 roles), icon `CalendarRange`.

**F19 "to check" — decided:** `/shifts/lookup|master|schedules` are tabs of `/shifts`; `/companysetup/routing` is a tab of `/companysetup` → no menu (Step 5 guard allows sub-routes). `/me` is superseded by `/portal/me` → added to the Step 5 delete list.

**Counts:** menus 107 → **89** (89 active, 89 visible). 0 orphans, 0 empty parents, 0 duplicate routes. `permissions` / `role_permissions` / `roles` unchanged (156 / 860 / 15).

**Carried forward:** Step 5 — `Sidebar.tsx` hardcoded `NAV_SECTIONS` fallback (incl. `/admin/backup`, which has no menu) and `OVERVIEW` special case. Step 3 — `permissionModule` values still mismatched (e.g. `ATT_OVERTIME` "Overtime", `ATT_ROSTER` "Shift Management").

### Step 2 — Permission data cleanup (2026-10-07)

Script: `docs/db_changes.sql` → "RBAC Step 2" block (one transaction, idempotent; re-run changed 0 rows). Restore: Step 0 Option B. `dotnet build` 0 errors, `tsc --noEmit` clean.

**F5 correction:** 4 codes were not unused. Code checks them as constants: `attendance.dayrequest.hr` (DayRequestRules), `attendance.regularisation.hr` (RegularisationRules), `leave.proof.waive` (LeaveProofRules), `recruitment.vacancy_notify` (NotificationService.Recruitment). Kept.

**Wired.** Each new code was granted to every role holding the endpoint's old code, so access is unchanged:

| Code | Endpoint | Grant change |
|---|---|---|
| `permissions.view` / `.manage` | `AdminController` permissions GETs / writes (F7: `SYS_PERM` menu now matches its API) | from `roles.manage` |
| `roles.view` / `.create` / `.edit` / `.delete` | roles GETs / `POST roles/wizard` / PUT, permissions, users, status / DELETE | from `roles.manage` |
| `users.view` / `.create` / `.edit` | users GETs + `users/assignable` / POST + wizard / PUT | from `users.manage`; `users.create` revoked from COMPANY_HR (it had no effect before) |
| `employees.transfer` | `POST employees/{id}/transfer` | +HR_MANAGER |
| `dashboard.employees.view` | `GET employees/dashboard-metrics` (handler alias still lets `employees.view` pass) | — |
| `companies.duplicate` | `POST companies/duplicate-masters` | — |
| `payroll.config.manage` | `PUT payroll/settings` | +FINANCE_MANAGER |
| `backup.manage` | `GET config/backup`, `POST config/restore` | — |
| `settings.view` | `GET config/company` | — |
| `portal.attendance.view` | `GET attendance/my-record` (had no policy) | — (all 15 roles hold it) |
| `attendance.approve` | `POST attendance/adjustments/decide` (was `attendance.manage`) | **Approved change:** LINE_MANAGER, EMS and COMPANY_HR can now decide; ADMIN can no longer decide (it had no menu for it) |
| `letters.signatories.manage` / `.upload` | `AuthorizedSignatoriesController` writes / `upload-signature` (part of F8). GETs stay `[Authorize]`, because the offer and letter drawers used by LM/COMPANY_HR call them | — |
| `task.view` | **new permission**, granted to the 11 `approvals.view` roles (same as the handler alias). Visibility rule is Step 4 | — |

Menus: `SYS_ROLE` → `roles.view`, `SYS_USER` → `users.view`. Sidebar fallback codes were updated to match (that fallback is deleted in Step 5).

**Deleted (43, with 194 grants).** No role lost access; every holder still passes through the endpoint's real code:
`roles.manage users.manage users.delete permissions.create/edit/delete reports.view (all holders already had reporting.view) reports.export reporting.dashboard audit.export attendance.edit attendance.regularise attendance.team banks/costcentres/departments/employmenttypes/grades/jobtitles/locations.manage (OrgMasters uses orgmasters.manage) currencies.view groups.manage groups.realign groups.view dev.maintenance_mode documents.verify employees.bank.view employees.delete employees.salary.view employees.salary.edit employees.terminate leave.approval leave.cancel leave.policies.manage leave.types.manage leave.team loans.manage payroll.edit payroll.process recruitment.post requisitions.approve shifts.assign shifts.overrides.manage`.
`PermissionAuthorizationHandler` aliases no longer mention `reporting.dashboard`, `groups.view` or `recruitment.post`.

**Counts:** permissions 156 → **114**; role_permissions 860 → **678** (+13 grants, −1 revoke, −194 deleted). 0 orphan grants, 0 menus with an unknown code, 0 controller policies missing from `permissions`.

**Codes left without a policy (by design):** the 4 rule constants above; `portal.dashboard.view` and `portal.leave.view` (handler aliases + `PermissionGate`); `letters.signatories.view` (menu only); `task.view` (Step 4); `payroll.payslips.view` (see below).

**Carried forward:**
- **Payslips (follow-up):** the `PAY_SLIPS` menu needs `payroll.payslips.view` (ESS holds it), but the page calls `payroll.view` endpoints, so ESS gets 403. Wiring it to `cycles/{m}/payslips/{employeeId}` would let ESS fetch anyone's payslip. A self-only "my payslips" endpoint is needed.
- **Step 4:** `PermissionAuthorizationHandler` still has hardcoded aliases (portal → module, task ⇔ approvals, org-master read for recruiters) and the God-mode role-id check. Replace them with data, or keep them explicitly. `MenusController` writes still have no policy (F8).
- **Risk noted:** employee salary/bank fields are not field-gated. Their codes were deleted because nothing checked them; field-level gating is a separate feature.

### Step 3 — Link menus ↔ permissions (2026-10-07)

Script: `docs/db_changes.sql` → "RBAC Step 3" block (schema + one transaction, idempotent; re-run changed 0 rows). Restore: Step 0 Option B. If needed, drop the new links first: `ALTER TABLE menus DROP CONSTRAINT FK_menus_permission; ALTER TABLE permissions DROP CONSTRAINT FK_permissions_menu;` (Option B lists columns, so the new ones stay NULL). `dotnet build` 0 errors, `tsc --noEmit` clean, `RecruitmentNotificationTests` 9/9.

**Schema:** `menus.permissionId` and `permissions.menuId` (`nvarchar(36) NULL`), with foreign keys `FK_menus_permission` / `FK_permissions_menu` and indexes. `permissions.module` was made nullable so the code no longer writes it.

**Backfill (approved mapping):**
- `menus.permissionId` comes from `permissionCode`, on the 75 menus that have a route. Group rows without a route stay NULL. TASK keeps `task.view`.
- `permissions.menuId`: every one of the 114 permissions is linked, one screen each. A shared code goes to the main screen: `attendance.view/.manage/...` → ATT_DASH, `payroll.view/.run/...` → PAY_RUNS, `approvals.view/.manage` + `task.view` → TASK, `settings.* backup.manage` → SYS_SETTINGS (`/admin/backup` has no menu). The full map is in the script's `@map`.
- Checks: 0 permissions without a menu, 0 routed menus without a permission, 0 menus whose FK code differs from the old `permissionCode`.

**Code:**
- Entities: `Menu.PermissionId` / `Permission` / `Permissions`, `Permission.MenuId` / `Menu`. `Menu.PermissionModule`, `Menu.PermissionCode` and `Permission.Module` were removed from the entities. The columns stay until Step 7.
- `MenuService` sidebar: a screen is visible only when the role holds `menu.Permission.Code`. The module fallback and the "Common = everyone" rule (S3) are removed. Before/after check of each role's visible menus: **529 = 529 role–menu pairs, no difference**.
- `MenuService`: the TASK auto-insert (F10) was removed early. It referenced the retired columns, and the row already exists in the DB. The TASK visibility special case stays until Step 4.
- Menu create/update still take `permissionCode` from the drawers, but an unknown code is now rejected (F2). Deleting a menu that still has permissions is blocked ("move them first").
- Permissions API: `module` is now derived from the menu's group name (for example "Attendance"). New fields `menuId` and `menuName`. Create/update accept `menuId`; otherwise the sent module name is resolved to a menu. Deleting a permission that a menu uses as its view permission is blocked. Five copied DTO mappers were merged into `ToPermissionDto`.
- `MenuDto.permissionModule` is now the group name (for the badges on the current Menu Management page). It was removed from `MenuNavDto` and from the create/update requests.

**Counts:** menus 89, permissions 114, role_permissions 678 (unchanged).

**> Code correction and static values (found in Step 3, for Step 4):**
- `AdminService.CreatePermissionAsync` auto-grants new permissions to hardcoded role ids `superadmin`, `role_super`, `admin` (S1).
- `AdminService.GetAllPermissionsAsync` falls back to the in-code `SeedPermissions` list when the query fails (a fake fallback, follow.md #1). The `DefaultRolePermissions` dictionary is also hardcoded.

**Carried forward:** Step 6 replaces the free-text module and permission-code inputs with menu and permission selects, and then removes the derived `module` / `permissionModule` fields. Payslips self-only endpoint, handler aliases and `MenusController` policies stay as listed in Step 2 (user confirmed 2026-10-07).

