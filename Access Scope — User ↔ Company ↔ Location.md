# Access Scope — User ↔ Company ↔ Location

Last updated: 2026-10-12 · Branch: `role-wise-system-check`

**Goal:** decide which employees each user can see and act on. Access comes from **company + location**, plus the user's own reporting line. Superadmin and developer have no boundaries.

---

## 1. Progress

| Step | What | Status |
|---|---|---|
| 0 | Backup, snapshots and baseline data report | ✅ Done 2026-10-12 |
| 1 | Security: validate `X-Company-Id`; make Admin company-specific | ⬜ Not started |
| 2 | Data model and scope SP (`sp_access_resolve_scope`) with a per-request cached `IAccessScopeService` | ⬜ Not started |
| 3 | User create/edit: locations under each company | ⬜ Not started |
| 4 | Global enforcement: EF query filters on Employee and employee-linked tables | ⬜ Not started |
| 5 | Sweep A: `IgnoreQueryFilters` sites in employees, manager picker, reports, exports, dashboard | ⬜ Not started |
| 6 | Sweep B: Task approvals, bell, attendance, roster, leave, letters, separation, onboarding, loans | ⬜ Not started |
| 7 | Payroll and masters rules (§4) | ⬜ Not started |
| 8 | Login gate: "Location/unit not mapped" screen | ⬜ Not started |
| 9 | Workspace header shows locations; `/portal/me` shows Location | ⬜ Not started |
| 10 | End-to-end verification and close-out | ⬜ Not started |

Status keys: ⬜ Not started · 🟡 In progress · ✅ Done · ⏸ Blocked

---

## 2. Rules (user decisions, 2026-10-12)

| # | Rule |
|---|---|
| R1 | Fix both gaps found in the research: the `X-Company-Id` header must be validated, and every `IgnoreQueryFilters` employee query must apply the scope |
| R2 | Scope is resolved by a **stored procedure** (SQL Server), called once per request and cached. Row filtering stays in EF and uses the resolved lists (see §3.2) |
| R3 | **Superadmin and developer see everything** by default. The design must allow restricting them to specific locations later (`users.LocationRestricted`; no UI yet) |
| R4 | **Admin (GLOBAL) becomes company-specific.** Only superadmin and developer bypass company access. The GLOBAL/GROUP bypass in `CompanyScopeService` is removed |
| R5 | **Line manager (EMS)** sees (a) their whole reporting line in any location and any department, and (b) every employee of their own department within their mapped locations |
| R6 | **Company-wide roles** (Company HR, Admin, Finance…) see employees of their mapped company + locations only |
| R7 | **ESS** sees self only (plus their reporting line if they manage people). Location mapping never widens ESS |
| R8 | **Cross-location line manager:** the manager picker lists only employees the editor can see. Only superadmin, developer or a user mapped to both locations can set a manager from another location |
| R9 | **Login gate:** a user whose employee record has no location is blocked after login with: *"Your location/unit has not been mapped. Please contact your HR."* Superadmin and developer are exempt (§3.6) |
| R10 | `/portal/me` → Employment & Organisation shows the employee's Location (it is missing today) |
| R11 | The workspace header shows the assigned locations next to the company |
| R12 | No hardcoded companies, locations or roles. Everything is read from masters (see `no-static-values`) |

---

## 3. Design

### 3.1 Data model (Step 2)

| Change | Purpose |
|---|---|
| `user_company_access.AllLocations bit NOT NULL DEFAULT 1` | Every location of the company, including ones added later |
| New `user_location_access (Id, UserId, CompanyId, LocationId, CreatedAt…)`, unique on (UserId, CompanyId, LocationId) | Explicit location picks; used when `AllLocations = 0` |
| `users.LocationRestricted bit NOT NULL DEFAULT 0` | Superadmin/developer only: 0 = no boundaries; 1 = follow their mapping like anyone else (future use, R3) |
| Roles: Admin `RoleLevel` GLOBAL → COMPANY | R4 |

Migration: existing rows get `AllLocations = 1`, so nothing changes for existing users on go-live. The unused `user_access_grants` table is **not** reused, because it carries a role and effective dates per grant, which this design doesn't need.

### 3.2 Scope resolution — SP and service

`sp_access_resolve_scope @UserId` returns three result sets:
1. **Locations:** `(CompanyId, LocationId)`. `LocationId` NULL means all locations of that company.
2. **Own employee ids:** the user's employee records.
3. **Visible-by-relation employee ids:** the full reporting line, walked recursively with a CTE (R5a), plus department-in-location peers for line managers (R5b).

`IAccessScopeService` (it replaces the internals of `CompanyScopeService`) calls the SP once per request and caches the result for that request. It exposes `IsUnbounded`, `AllowedCompanyIds`, `AllowedLocationIds(companyId)`, `RelationEmployeeIds`, and `ScopeEmployees(IQueryable<Employee>)`.

**Why not SPs for everything:** EF query filters can't call SPs, and there are 278 employee queries. The SP computes the hard part (recursive reporting line, location set) once per request. EF then filters with `Contains` on those lists.

### 3.3 Visibility rule (single source)

Employee X is visible to user U when any of these holds:
1. U is superadmin or developer and `LocationRestricted = 0`.
2. X is U's own record.
3. X is in U's relation set (reporting line, or department-in-location for line managers).
4. U's role has company-wide reach, X's company is in U's companies, and X's location is allowed (or that company has `AllLocations = 1`).

### 3.4 Enforcement

- **Step 1:** middleware rejects an `X-Company-Id` outside `AllowedCompanyIds` with `403 COMPANY_NOT_ALLOWED`. `ALL` is allowed only for unbounded users.
- **Step 4:** the `Employee` query filter and employee-linked tenant tables (leave, attendance, roster, letters, separation, loans, payroll lines…) filter on `EmployeeId ∈ scope`.
- **Steps 5–6:** every `Employees.IgnoreQueryFilters()` site (123 today) either calls `ScopeEmployees()` or carries a comment explaining why it is intentionally unscoped (e.g. login, background jobs).

### 3.5 UI

- **Create/Edit User, Step 3:** under each ticked company, list its locations from the Locations master, with an *All locations (incl. future)* toggle (on by default). The side summary shows "Entities x/y · Locations a/b".
- **Workspace header:** `LIFECARE – Lifecare Hospitals · Arch Place, Diani Bliss`, or `· All locations`.
- **`/portal/me` → Employment & Organisation:** add a Location row.

### 3.6 Login gate (Step 8)

Login still succeeds, but `/me/access-status` returns `LOCATION_NOT_MAPPED` and the API refuses other calls with that code. The frontend shows a blocking screen with the R9 message and a Sign-out button. The gate is set when:
- the user has an employee record whose `EmploymentDetail.LocationId` is empty, **or**
- the user's role has company-wide reach and they have no location access in any company.

Superadmin and developer are exempt. When a user account is created from an employee, the employee's own company and location are mapped automatically.

---

## 4. Payroll and masters (recommendation — confirm before Step 7)

**Payroll stays a company-level process.** A payroll run, statutory filings (PAYE, pension, health levy) and the bank file belong to the company as one legal entity. Running payroll for one location would split statutory totals and bank files.

| Action | Location-scoped HR | Company-wide user (`AllLocations = 1` for that company, with payroll permission) |
|---|---|---|
| Payroll inputs (variable pay, overtime, deductions, loans) | ✅ For their locations' employees | ✅ |
| Payslips and payroll register | 👁 Their locations' rows only | ✅ |
| Run, recalculate, approve, finalise, bank file, statutory export | ❌ | ✅ |
| Company masters (departments, job titles, grades, leave types, shifts, pay components, statutory) | 👁 Read (needed for pickers) | ✅ Edit |
| Location-owned items (location holiday calendar, biometric devices, location roster) | ✅ Edit for mapped locations | ✅ |

---

## 5. Risks and baseline checks (Step 0)

- Count of employees with an empty or invalid `LocationId`, per company. These users will hit the login gate.
- Users with company access but no employee record (service or admin accounts).
- Hardcoded superadmin checks in `CurrentUserService.IsSuperAdmin` (username or role-id strings). Keep them as they are; do not widen them.
- Current company access of Admin-role users. They may lose companies after R4, so list them before the change.

---

## 6. Prompts for each step (fresh chat per step)

**Step 0**
> Read docs/access.md. Do Step 0: full DB backup plus `bak_access_*` snapshots of users, user_company_access and roles. Produce the baseline report from §5 (counts only, small output) and record it in §7. Change no data. Update the Progress table.

**Step 1**
> Read docs/access.md. Do Step 1 (R1, R4): validate `X-Company-Id` against the user's accessible companies (403 COMPANY_NOT_ALLOWED; ALL only for superadmin/developer). Remove the GLOBAL/GROUP bypass in CompanyScopeService and AttendanceScopeRules. Set the Admin role to COMPANY level through an idempotent db_changes.sql block. Add tests. Update the Progress table, §7, policies.md and the management artifact.

**Step 2**
> Read docs/access.md. Do Step 2 (§3.1–3.3): schema changes in db_changes.sql (idempotent; existing rows AllLocations = 1), `sp_access_resolve_scope`, and `IAccessScopeService` with a per-request cache. Make CompanyScopeService use it. No behaviour change yet beyond Step 1. Add tests for the R5 line-manager set. Update docs.

**Step 3**
> Read docs/access.md. Do Step 3 (§3.5): Create/Edit User Step 3 lists locations under each ticked company, with an All-locations toggle. API saves to user_location_access, and the side summary shows location counts. Auto-map the employee's own company and location when the user is created from an employee. Add the browser test steps to the artifact checklist. Update docs.

**Step 4**
> Read docs/access.md. Do Step 4 (§3.4): apply IAccessScopeService in the EF query filters for Employee and employee-linked tenant tables. Run E2E on a throwaway DB clone only. Update docs.

**Step 5**
> Read docs/access.md. Do Step 5: sweep `Employees.IgnoreQueryFilters()` sites in employees, dossier, manager picker (R8), reports, exports and dashboard. Each site calls ScopeEmployees() or carries a "why unscoped" comment. Update docs.

**Step 6**
> Read docs/access.md. Do Step 6: same sweep for Task approvals, bell rules, AttendanceScopeRules/AttendanceReportScope, roster, leave, letters, separation, onboarding and loans. Update docs.

**Step 7**
> Read docs/access.md. Do Step 7: apply the §4 payroll and masters matrix as confirmed by the user. Update docs.

**Step 8**
> Read docs/access.md. Do Step 8 (R9, §3.6): `/me/access-status`, the API refusal code LOCATION_NOT_MAPPED, and a frontend blocking screen with the message and Sign-out. Superadmin and developer are exempt. Update docs.

**Step 9**
> Read docs/access.md. Do Step 9 (R10, R11): the workspace header shows assigned locations (or "All locations"), and `/portal/me` → Employment & Organisation shows Location. Update docs.

**Step 10**
> Read docs/access.md. Do Step 10: run E2E per role (developer, superadmin, Admin, Company HR with one location, EMS cross-location manager, ESS, user with no location) on a DB clone. Record the results in §7 and close the plan.

---

## 7. Completion notes

### Step 0 — Backup and baseline (2026-10-12)

**Backup:** `HRMSCore_Local2` → `/var/opt/mssql/data/HRMSCore_Local2_before_access.bak` (COPY_ONLY, COMPRESSION, CHECKSUM; VERIFYONLY valid). A copy is at `docs/dbscript/HRMSCore_Local2_before_access.bak` (32 MB).
Restore (worst case; replaces the whole DB): `RESTORE DATABASE HRMSCore_Local2 FROM DISK='/var/opt/mssql/data/HRMSCore_Local2_before_access.bak' WITH REPLACE;`

**Snapshots** (quick restore of single tables): `bak_access_users` (71), `bak_access_user_company_access` (125), `bak_access_roles` (15). Counts match the live tables. Drop them at Step 10.

**Baseline (read-only):**

| Check | Result |
|---|---|
| Active employees with no location | 3: LIFECARE 2, MAKL 1 |
| Active employees whose location is invalid or belongs to another company | 2: LIFECARE 1, SDI 1 |
| Active login users who would hit the gate (R9) | 3 — COMPANY_HR 1, FINANCE_MANAGER 1, LINE_MANAGER 1 |
| Active users with no employee record | 4 — DEVELOPER, EMPLOYEE, HR_MANAGER, ROLE_CLINICAL_SUP (one each) |
| Companies with no active location | JIVA (0). BLISS has 69 locations, so Step 3 needs search and collapse |
| GLOBAL-level non-god roles with active users | ADMIN 0, **HR_MANAGER 3, FINANCE_MANAGER 2, GROUP_HR 1, SUPER_ADMIN 3** (SUPER_ADMIN is matched by code, `isGodMode = 0`) |
| GLOBAL users with **0 company grants** (they would lose everything if the bypass is removed) | `finance` (FINANCE_MANAGER), `insurance.hr` (HR_MANAGER) |

**Open for Step 1:** R4 names only Admin. HR_MANAGER, FINANCE_MANAGER and GROUP_HR are also GLOBAL and currently bypass company access. Before the bypass is removed, decide whether they become company-specific too and which companies `finance` and `insurance.hr` get.

