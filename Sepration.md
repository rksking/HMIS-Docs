# Separation (Resignation) Module — Design & Implementation Plan

> Status: **revision 6 (2026-10-10). Steps 1–3 ✅ done (3 = 3a lifecycle in days + 3b ESS resign). Next: Step 4 — approvals + notifications (prompt in §11).**
> Scope: **RESIGNATION only** (probation confirmation, termination and the confirmation workflow are out of scope).
> Sources: `docs/Sepration Workflow (1).docx`, `docs/Staff Exit Clearance Form.docx`, HR notes (Memo 9/10/26),
> reference task-tracker screenshot (2026-10-10).
> Artifact: **sep-workflow** — https://claude.ai/artifact/4HWTJrHZYeU2xobzMRaUyF
> Rules: `docs/follow.md` — DB-driven (no hardcoded data), clean readable code, `sp_*` for complex calc, modules under
> `src/modules/<domain>/components/` + `index.ts`, page root `space-y-6` with unboxed header and one action bar,
> **SideDrawer (50%) for forms — never modals**, permissions `<module>.<action>` + menu rows, plan before code,
> flag static values, zero `tsc` / `dotnet build` errors, real-role E2E, every DB change in `docs/db_changes.sql`.

---

## 1. Scope and flow

```
Employee ─► Line Manager ─► Company HR ─► [login blocked] ─► Clearance ─► HR final clearance ─► F&F Process ─► F&F Paid ─► Exit interview ─► Experience cum Relieving Letter ─► Released
(ESS)       approval         approval                         (7 days before LWD, configurable; parallel or sequential)
```

Every stage is a row in **Separation Config** (§3) with the employees mapped to it. When a stage opens, the people
resolved for it get an **email**, a **bell notification** and a **/task entry** (§3.3).

| # | Stage (code) | Who acts | Where |
|---|---|---|---|
| 1 | Initiate resignation | Employee (ESS) | `/portal/separation` |
| 2 | `LINE_MANAGER_APPROVAL` (+ LWD change request) | Employee's reporting manager | `/task` |
| 3 | `HR_APPROVAL` → **login blocked**, clearance scheduled | Mapped Company HR | `/task` |
| 4 | `L1_CLEARANCE` — checklist + **handover / KT notes + document upload** | Reporting manager | `/task` |
| 5 | `ADMIN_CLEARANCE`, `FINANCE_CLEARANCE`, `AUDIT_LEGAL_CLEARANCE`, `IT_CLEARANCE` (`SACCO_CLEARANCE` off) | Mapped employees | `/task` |
| 6 | `HR_FINAL_CLEARANCE` — checklist, comment, submit (opens when all clearances are done) | Mapped HR | `/task` |
| 7 | `FNF_PROCESS` — amount + F&F statement upload | Mapped HR | `/task` |
| 8 | `FNF_PAID` — paid amount, payment reference, evidence | Mapped Finance | `/task` |
| 9 | `EXIT_INTERVIEW` — feedback form | Employee | `/portal/separation` + `/task` |
| 10 | `EXPERIENCE_LETTER` — "Experience cum Relieving Letter" via the letters module | System | auto |

### 1.1 Last working day (LWD)

- **Official LWD = resignation date + N − 1** (the resignation day is day 1: resign 1 Oct, 30 days → 30 Oct).
  - Confirmed employee: N = the employee's notice period days.
  - On probation (`employmentStatus = PROBATION`, or confirmation date after the resignation date): N = the employment
    type's **probation notice days** (`employment_types.probationNoticeDays`).
- Notice, probation days and probation notice days come from the **Employment Type master** (days only — no months);
  notice and probation days are copied onto the employee at hire. Confirmation date = joining date + probation days.
- **Resignation date:** today or later (company time zone). Back-dating only for superadmin / developer.
- **LWD by agreement:** on the resign form (or later via Change LWD) the employee can enter a different LWD — earlier or
  later than official — with a reason; the line manager approves it and it is fixed. superadmin / developer can set it
  directly (`separation.lwd.override`, Step 4).
- **Recovery days** = days the agreed LWD is earlier than the official LWD; shown in the tracker header and carried
  into F&F.

### 1.2 Clearance timing

- Clearance tasks get **configured trigger date = LWD − `clearanceTriggerLeadDays`** (7, configurable).
- They fire on that date (or immediately if HR approves later than it). **Actual trigger date** is stamped when they fire.
- `clearanceMode`: **PARALLEL** (all clearance stages at once) or **SEQUENTIAL** (one at a time, by `sequenceOrder`).
- `HR_FINAL_CLEARANCE` opens when every active clearance task is completed; then F&F Process → F&F Paid.
  The exit interview opens with the clearance tasks. The letter is issued after F&F Paid and the exit interview.

### 1.3 Login block

On HR approval: `employees.accessBlockedAt = now` and the login path refuses that employee's user. Full deactivation
(`Status = RESIGNED`, `IsActive = 0`) happens at release.

---

## 2. Task tracker (reference screenshot)

After HR approval the case is driven by `separation_tasks` — one row per post-approval stage:

| Column in the screen | Field |
|---|---|
| Task | `taskName` |
| Status | `status` (WAITING / PENDING / COMPLETED / SKIPPED) |
| Configured Trigger Date | `configuredTriggerDate` |
| Actual Trigger Date | `actualTriggerDate` |
| Retriggered Date | `retriggeredDate` |
| Assigned To | `assigneeName` (resolved person) or the mapped employees |
| Owner | `ownerLabel` (stage name, e.g. "Finance Clearance", "HRBP") |
| Completed By | `completedByName` + `completedAt` |
| Documents / Download | `documentUrl` + `separation_attachments` |
| Action | **Retrigger** (`separation.task.retrigger`) — stamps `retriggeredDate`, re-opens the task, re-sends notifications |

Header: office location, date of joining, notice period, date of resignation, LWD, recovery days.

---

## 3. Separation Config (Masters & Configuration) — built in Step 2

**Menu:** Masters & Configuration (`menu-masters`) → **Separation Config** → `/masters/separation-config`.
Permissions (Masters pattern): `separation.config.view` (menu, read) + `separation.config.manage` (edit).
All separation configuration lives here.

### 3.1 Screen (follow.md #6, #7)

Root `space-y-6`, unboxed header ("Separation Config" + subtitle), one action bar (company picker when the workspace
is a group, Refresh). A readiness banner lists active stages that have nobody mapped. Three tabs:

1. **Stages & mapping** — one row per stage: order, name, group, who handles it (+ mapped employee chips), email / bell,
   status. Active MAPPED stages without anyone show **"Not mapped"**. **Edit** opens a **SideDrawer**: handled by
   (reporting manager / mapped employees — employee and system stages keep their type), employee search + chips,
   email and bell switches, and On/Off for clearance stages only.
2. **General** — days before LWD to start clearance (≥ 0), Parallel / Sequential. (Probation notice days moved to
   the Employment Type master in Step 3a.)
3. **Checklists** — pick a clearance or HR stage; add / edit / switch off lines (SideDrawer), move up / down.

Rules enforced by the API: company must be in the user's workspace; only active employees of that company can be
mapped; core stages (approval, HR, F&F, exit, letter) cannot be switched off; checklists only on clearance and HR
stages. Every change writes an `audit_log` row (`entityType = SeparationConfig`). A company created after the script
ran is seeded on first open by `sp_SeedSeparationConfig`.

### 3.2 Who a stage goes to

| `assigneeType` | Resolved to |
|---|---|
| `DIRECT_MANAGER` | The employee's reporting manager. If none: the stage's mapped employees. |
| `MAPPED` | The stage's mapped employees (`separation_stage_assignees`). Any one of them can act. |
| `EMPLOYEE` | The resigning employee. |
| `SYSTEM` | Nobody — done automatically. |

If nobody can be resolved for an active stage, the action that needs it is **refused** with a message naming the stage
and pointing to *Masters & Configuration → Separation Config* (same rule as SRF/Offer "no approval matrix"). No
hard-coded fallback person.

### 3.3 What a mapped person receives

When a stage opens for them (and again on Retrigger):

- **Email** — through `IEmailService.SendEmailAsync`, using an email template per event code
  (`SEPARATION_STAGE_ASSIGNED`, `SEPARATION_SUBMITTED`, `SEPARATION_DECIDED`…), only if `notifyEmail` is on.
- **Bell** — a `Notification` row (`Kind = "separation"`, body, `Link` to the task), only if `notifyBell` is on.
  Written from a new `NotificationService.Separation.cs` partial, like `NotificationService.Recruitment.cs`.
- **/task entry** — the task appears in their **Awaiting me** bucket (always).

`/task` access for mapped people: `UserAccessResolver.IsApproverAsync` is extended to count an employee mapped to any
active stage (or holding an open separation task) as an approver, so they receive the `grantedToApprovers`
permissions (`task.view`, `separation.task.act`, …) whatever their role. Superadmin / developer see everything.

---

## 4. Data model (Step 1 — applied)

Entities: `backend/Domain/Entities/SeparationEntities.cs`; mapped in `HrmsDbContext` (camelCase column convention,
tenant query filters on the company-scoped tables). Schema in `docs/db_changes.sql`.

| Table | Purpose |
|---|---|
| `separation_config` | Per company: `clearanceTriggerLeadDays`, `clearanceMode` (`probationNoticeDays` moved to `employment_types` in Step 3a). |
| `separation_stages` | Per company, 13 stages: `code`, `name`, `stageGroup`, `sequenceOrder`, `assigneeType`, `requiresHandover`, `notifyEmail`, `notifyBell`, `isActive`. |
| `separation_stage_assignees` | Employees mapped to a stage (`stageId`, `employeeId`, `isActive`). Not seeded — HR maps people. |
| `separation_checklist_items` | Yes/No/N/A lines per stage (seeded from the Exit Clearance Form). |
| `separation_requests` | The case: dates, notice, `recoveryDays`, official / requested / approved LWD, status, stage, `accessBlockedAt`, `releasedAt`. |
| `separation_approvals` | Decisions on `LINE_MANAGER_APPROVAL`, `HR_APPROVAL`, `LWD_REQUEST`. |
| `separation_tasks` | The tracker (§2). |
| `separation_clearance_responses` | Checklist answers per task. |
| `separation_fnf` | Amount, statement, paid amount, reference, evidence. |
| `separation_exit_interviews` | Reasons, ten 1–5 ratings, comments, contact details. |
| `separation_attachments` | Handover / KT docs, F&F statement, payment evidence. Blob path `employee/<companyId>/separation/<separationId>/<category>/`. |
| `employees.accessBlockedAt` | Login block. |

### 4.1 Seeded checklists (per company)

| Stage | Lines |
|---|---|
| L1 Manager clearance | Office inventory correct · Working tools, equipment returned · Keys returned · Handover notes completed & surrendered |
| Admin Clearance | Company mobile phone · Company simcard · Company car · Company house keys · House items checklist done |
| Finance Clearance | No pending issue (money given and not accounted for) |
| Audit & Legal | No pending issue |
| IT clearance | Network & email ID deleted · System access & passwords disabled · ICT equipment returned |
| SACCO *(inactive)* | Outstanding loans |
| HR final clearance | Staff ID · Uniform · Notice period served · Years of service · Leave balance · Leave encashment · Certificate of service issued · Exit interview held |

Payroll "Final dues paid" from the form is the F&F Paid stage.

---

## 5. Menus and permissions (Step 1 — applied)

| Menu | Parent | Route | Menu permission |
|---|---|---|---|
| Separation | Self Service | `/portal/separation` | `separation.view` |
| Separation Config | Masters & Configuration | `/masters/separation-config` | `separation.config.view` |

| Permission | Menu | Granted to |
|---|---|---|
| `separation.view`, `.initiate`, `.withdraw`, `.lwd.request`, `.exit.submit`, `.task.act` | Separation (self service) | **every role that has My Profile (`portal.me.view`)** — anyone can resign (15 roles today) |
| `separation.approve`, `.lwd.approve` | Task Hub, **granted to approvers** | role_mgr (Line Manager), ems; approve also role_hr, company_hr, group_hr (+ any mapped employee via IsApproverAsync, Step 4) |
| `separation.task.act` | Task Hub, granted to approvers | role_fin, finance, admin (+ self-service roles above) |
| `separation.task.retrigger`, `.view.all` | Task Hub | role_hr, company_hr, group_hr |
| `separation.lwd.override` | Task Hub | superadmin / developer (god mode) |
| `separation.config.view`, `separation.config.manage` | Separation Config | role_hr, company_hr, group_hr, admin |

---

## 6. Approval-engine integration

Same per-entity-type pattern as leave / requisition in `ApprovalsService.GetPendingApprovalsAsync` / `DecideTaskAsync`.
New entity types on `/task`: `SEPARATION` (line manager + HR approval, LWD request) and `SEPARATION_TASK` (every tracker
task; the stage name is the title). Extend the `entityType` union in `frontend/src/modules/task/types.ts`.

## 7. Module structure

- **Backend:** `Domain/Entities/SeparationEntities.cs` ✅, `Application/DTOs/Separation/SeparationConfigDtos.cs` ✅,
  `Application/Common/Interfaces/ISeparationConfigService.cs` ✅, `Infrastructure/Services/SeparationConfigService.cs` ✅,
  `Api/Controllers/SeparationConfigController.cs` ✅ (`api/separation-config`); `ISeparationService` /
  `SeparationService` / `SeparationController` (`api/separation`) / `SeparationDtos.cs` ✅ (Step 3); to come:
  `NotificationService.Separation.cs`, approval / task services.
- **Frontend:** `src/modules/separation/` — `masters/SeparationConfigView.tsx` ✅, `components/` ✅
  (SeparationStagesTab, StageMappingDrawer, SeparationGeneralTab, SeparationChecklistTab, ChecklistItemDrawer),
  `api.ts`, `types.ts`, `constants.ts`, `index.ts` ✅; Step 3 ✅: `portal/PortalSeparationView.tsx`, `components/`
  SeparationRequestDrawer (tabs Resign / Change LWD / Withdraw → InitiateResignationForm, LwdRequestForm,
  WithdrawResignationForm), SeparationCaseCard, EmploymentFactsCard, SeparationTimeline; to come: TaskActionDrawer,
  HandoverDrawer, FnfDrawer, FnfPaymentDrawer, ExitInterviewDrawer, SeparationTracker.
  Routes: `src/app/(dashboard)/masters/separation-config/page.tsx` ✅, `src/app/(dashboard)/portal/separation/page.tsx` ✅.

---

## 8. Build steps

| Step | Scope | Status |
|---|---|---|
| 1 | DB + domain foundations: script, entities, DbContext, menus, permissions, grants, seeds; verified on a DB clone (2 runs, idempotent) and applied to `HRMSCore_Local2`. Add Employee auto-fill confirmed. | ✅ Done 2026-10-10 |
| 2 | **Separation Config** (Masters & Configuration): API + screen — Stages & mapping (employee mapping drawer), General, Checklists; `sp_SeedSeparationConfig`; view/manage permissions; grants fixed to real role ids. | ✅ Done 2026-10-10 |
| 3a | **Employee lifecycle in days:** probation days everywhere (months removed), `employmentStatus`, probation end / separation / LWD dates, probation notice days on the Employment Type master. | ✅ Done 2026-10-10 |
| 3b | **ESS Initiate resignation:** portal page, Separation Request drawer (Resign / Change LWD / Withdraw), LWD calc (confirmed / probation), LWD by agreement, recovery days, withdraw. | ✅ Done 2026-10-10 |
| 4 | **Approvals + notifications:** line manager + LWD request + HR on `/task`; email + bell per stage mapping; `IsApproverAsync` for mapped employees; login block; `employmentStatus = NOTICE`. | ⏳ Next |
| 5 | **Clearance tracker:** task generation, trigger date job, parallel / sequential, checklist + handover / KT upload, retrigger. | |
| 6 | **HR final clearance + F&F Process + F&F Paid.** | |
| 7 | **Exit interview + Experience cum Relieving Letter + release.** Full cross-role E2E + report. | |
| later | **Separation Reports** (HR-side list / reports) — D1. | |

## 9. Decisions (answered 2026-10-10)

| # | Answer |
|---|---|
| D1 | HR-side screen → **"Separation Reports"**, later. |
| D2 | Letter via the letters module, named **"Experience cum Relieving Letter"**, issued by System. |
| D3 | Exit interview is filled by the **employee**; HR can retrigger it. |
| D4 | Active stages as in the reference tracker (L1, Admin, Finance, Audit & Legal, IT, HR final, F&F, exit, letter). SACCO off. |
| D5 | Employee can withdraw until HR approval. |
| D6 | Every stage has configurable employee mapping in **Masters & Configuration → Separation Settings**; mapped employees get email + bell + `/task` entry. |

## 10. Step log

### Step 1 — DB + domain foundations (2026-10-10)
- `docs/db_changes.sql`: 11 tables + `employees.accessBlockedAt`, 2 menus, 12 permissions, 22 role grants,
  per-company seeds (15 companies → 15 config rows, 195 stages, 345 checklist lines). No employee mappings seeded.
- Verified on a throwaway restore (`HRMSCore_SepTest`) — run twice, identical counts, no errors — then applied to
  `HRMSCore_Local2` (employees unchanged: 3,876). Clone dropped; pre-change backup kept at
  `/var/opt/mssql/data/sep_clone.bak` in the `sqlserver` container.
- `SeparationEntities.cs` (11 entities), `Employee.AccessBlockedAt`, DbSets + mappings + tenant filters in
  `HrmsDbContext`. `dotnet build` 0 errors (warnings unchanged at 51).

> **Code correction and static values** (follow.md #10)
> - `docs/dbscript/tables_proc_all.sql` is **out of date**: it shows `menus.permissionModule/permissionCode` and
>   `permissions.module`, but the live DB uses `menus.permissionId`, `permissions.menuId/category/isSelfService/
>   grantedToApprovers`. The first clone run failed on this; the script now follows the live schema. The reference
>   schema file should be regenerated.
> - New menus use existing icons (`ClipboardList`, `Sliders`) because `menuIconHelper.tsx` has a fixed icon list;
>   unknown names fall back to a file icon.

### Step 2 — Separation Config screen (2026-10-10)
- **Menu** renamed to **Masters & Configuration → Separation Config** (`/masters/separation-config`). Permissions split
  into `separation.config.view` (menu) + `separation.config.manage` (edit), like other Masters menus.
- **API** `api/separation-config` (`SeparationConfigController` → `SeparationConfigService`): companies of the workspace,
  config, employee search, save general, save stage + mapping, add / edit / reorder checklist lines. Audit row per change.
- **Screen** `SeparationConfigView` with tabs Stages & mapping / General / Checklists, SideDrawers for stage mapping and
  checklist lines, readiness banner ("N of M active stages have nobody mapped").
- **DB** (`docs/db_changes.sql`, same block, re-runnable): menu rename + permission split for DBs that ran Step 1;
  `sp_SeedSeparationConfig @companyId` now holds the default stages / checklists and is run for every company;
  **grants fixed** — real users are on `ess`/`role_emp` (employees), `role_mgr` (line managers; `ems` is inactive),
  `role_hr`, `role_fin`; self-service separation permissions now go to every role that has `portal.me.view`.
- **Fix to Step 1:** `AccessBlockedAt` had been added to `EmploymentDetail` instead of `Employee`; queries on employment
  details would have failed at runtime. Moved to `Employee`.
- **Checks:** `dotnet build` 0 errors, `tsc --noEmit` 0 errors. Script: fresh clone ×2 (13 perms, 25 grants before
  the grant fix), seed procedure on a wiped company (13 stages / 12 active / 23 lines), upgrade on dev. API run against a
  throwaway clone (temporary password set in the clone only) as **hr** and **employee**: 16/17 checks passed; the one miss
  was the test's own count (8 MAPPED stages, not 7). Employee gets 403; outside-workspace company refused; mapping,
  un-mapping, guard rails, general settings, checklist add / switch off / reorder all correct; 6 audit rows written.
  Clone dropped; dev DB: employees unchanged (3,876), 111 separation grants, 15 roles see Separation, 0 mappings.

### Step 3a — Employee lifecycle in days (2026-10-10)
- User decisions: LWD = date + N − 1; probation in **days only**; lifecycle columns on `employment_details` (one row
  per company); probation notice days on the Employment Type master; Confirmation workflow = its own module later.
- `docs/db_changes.sql` new block: `employment_types.defaultProbationDays` + `probationNoticeDays`;
  `employment_details.probationPeriodDays`, `probationEndDate`, `employmentStatus`, `separationDate`, `lastWorkingDay`;
  `offers.probationDays`; `candidate_onboarding_profiles.probationDays`; one-time conversion; month columns and
  `separation_config.probationNoticeDays` dropped (`sp_DropColumnIfExists`). Step 1 block no longer creates
  `probationNoticeDays`.
- Code: `EmploymentDetail.ApplyProbation(days, confirmationDate?)` + `EmploymentStatuses`; ~35 backend / frontend files
  moved from months to days (Employment Type master + drawer, Add Employee, Assignment, Onboarding activation, offer
  drawers and letter snapshot, `/task` offer details, ESS profile, import template "Probation (Days)", letters'
  `{{probation_end_date}}`). Dashboards count probation from `employmentStatus`. Employment-type API validates
  probation ≥ 0 and probation notice ≥ 1.
- Verified on a clone (script ×2, 8/8 API checks), then applied to dev: 278 PROBATION / 3,595 CONFIRMED, employees
  3,876 unchanged, no month columns left. Pre-change backup `/var/opt/mssql/data/s3a.bak` in the `sqlserver` container.

### Step 3b — Self Service → Separation (2026-10-10)
- API `api/separation`: `GET me`, `GET lwd-preview`, `POST` (initiate), `POST {id}/lwd-request`, `POST {id}/withdraw`;
  policies `separation.view` / `.initiate` / `.lwd.request` / `.withdraw`. The caller's record is found by user id or
  e-mail (same rule as login), in the picked employment or the request's company; other people's cases → 404.
- Rules: resignation date ≥ company-local today unless superadmin / developer; reason required; one open case; line
  manager = reporting manager or mapped employee; HR approval needs a mapped employee; a different LWD (earlier or
  later) needs a reason and stays PENDING; withdraw only in PENDING_L1 / PENDING_HR. Reference `SEP-{year}-{0001}` per
  company. Trail in `separation_approvals` (RESIGNATION SUBMITTED / WITHDRAWN, LWD_REQUEST REQUESTED); audit
  `entityType = Separation`; `employment_details.separationDate` / `lastWorkingDay` set on submit, cleared on withdraw.
- Screen `/portal/separation` (layout of `/portal/attendance`) with the **Separation Request** drawer (like New
  Request; tabs Resign / Change LWD / Withdraw shown only when allowed).
- Fix: `separationErrorMessage` now reads `errors[0]` (ApiResponse.Fail), so Step 2 refusals also show their text.
- Bug found in testing and fixed: "today" used UTC (EAT is +3) → now `CompanyClock.TodayAsync`.
- Checks: `dotnet build` 0 errors, `tsc --noEmit` 0 errors, backend tests 322/322, clone E2E 28/28 (employee, line
  manager, HR, superadmin: preview confirmed / probation, refusals, initiate, trail, open case, LWD request, other users
  blocked, withdraw, numbering, back-date). Clone dropped. Dev DB has no separation cases and **nobody mapped** — HR
  must map "Company HR approval" (and a line-manager fallback for staff without a manager) before anyone can resign.

> **Code correction and static values** (follow.md #10)
> - `employmentStatus` is set at hire / backfill; nothing moves it to CONFIRMED when the confirmation date passes — the
>   Confirmation workflow module (later) will. Separation also counts "confirmation date after the resignation date"
>   as probation, so a stale PROBATION status is the only risk.
> - `LettersService` template preview still uses a sample `probation_end_date` = joining + 6 months (sample data only).
> - One "Extension of probation" letter template's own wording says "three (3) months" — HR text, not changed.
> - Offer numbers start at `OFF-{year}-0101` (hard-coded 100 base) and loan numbers use a random suffix — not copied.
> - Assignment / Add Employee still default notice days to 30 when the request has none (existing behaviour).

## 11. Next-step prompt (Step 4) — paste into a new chat

```
Read docs/follow.md and follow every rule (design reference: /portal/attendance page + its New Request drawer).
Read docs/sepration.md (§1, §3.2–3.3, §6, §8 step table, §10 step log) and memories "separation-module-plan" and
"employee-lifecycle-rules". Steps 1–3 of the Separation (resignation-only) module are done.

Do Step 4 — approvals + notifications on /task:
1. /task entity types SEPARATION (LINE_MANAGER_APPROVAL, HR_APPROVAL) and SEPARATION_LWD (LWD_REQUEST) in
   ApprovalsService.GetPendingApprovalsAsync / DecideTaskAsync, same per-entity pattern as leave / requisition; extend
   the entityType union in frontend/src/modules/task/types.ts and the TaskDecisionDrawer details (employment facts,
   resignation date, official / requested LWD, recovery days, reason, trail).
2. Who acts: LINE_MANAGER_APPROVAL → reporting manager, else the stage's mapped employees; HR_APPROVAL → mapped
   employees (any one can act). Extend UserAccessResolver.IsApproverAsync so mapped employees get the
   grantedToApprovers permissions. superadmin / developer see everything.
3. Decisions: line manager approve → PENDING_HR (a pending LWD request is decided in the same step: approve → fixed
   ApprovedLwd + RecoveryDays; reject → official LWD stays); reject → REJECTED (remarks required). HR approve →
   HR_APPROVED, employees.accessBlockedAt = now (login refused), employment_details.employmentStatus = NOTICE,
   ClearanceTriggerDate = LWD − clearanceTriggerLeadDays; HR reject → REJECTED. LWD requests raised after L1 approval
   go to the line manager on their own. separation.lwd.override lets superadmin / developer set the LWD directly.
   Every decision writes separation_approvals + audit_log.
4. Notifications per stage mapping (notifyEmail / notifyBell): email via IEmailService with templates per event code
   (SEPARATION_SUBMITTED, SEPARATION_STAGE_ASSIGNED, SEPARATION_DECIDED, SEPARATION_LWD_DECIDED) from the email
   template tables (no hard-coded text); bell via a new NotificationService.Separation.cs partial (Kind "separation",
   Link to the /task item). The employee is told of every decision.
5. Login block: AuthService refuses a user whose employee has accessBlockedAt set (clear message).

Before coding, present the Step 4 plan for approval (follow.md #9) with open points, e.g.: is a reason required and
shown to the employee on rejection; should the login block start at HR approval or on the LWD; what happens to an open
case's LWD request when the line manager rejects the resignation.

Every DB change goes in docs/db_changes.sql (idempotent; check INFORMATION_SCHEMA — the live schema differs from
docs/dbscript/tables_proc_all.sql). Test writes only on a throwaway DB clone (method in memory); map test people on the
clone only. After the step: tsc --noEmit and dotnet build clean, role E2E (employee, line manager, HR), update
docs/sepration.md (step table, step log, next prompt), docs/policies.md (next section after §88), the sep-workflow
artifact (Build steps + manual test), memory, and write the Step 5 prompt.
```
