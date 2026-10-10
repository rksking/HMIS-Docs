# Separation (Resignation) Module — Design & Implementation Plan

> Status: **revision 12 (2026-10-10). Steps 1–8 and 10 ✅ done — resignation flow to release, Reports → Separation, and the /letters clean-up (docs/letters_fixes.md, policies §95). Next: Step 9 blob-path clean-up (prompt §11.1).**
> Scope: **RESIGNATION only** (probation confirmation, termination and the confirmation workflow are out of scope).
> Sources: `docs/Sepration Workflow (1).docx`, `docs/Staff Exit Clearance Form.docx`, HR notes (Memo 9/10/26),
> reference task-tracker screenshot (2026-10-10).
> Artifact: **sep-workflow** — https://claude.ai/artifact/4HWTJrHZYeU2xobzMRaUyF (tab **Workflow Guide**: who does
> what after HR approval; tasks stay WAITING until the clearance start date = LWD − lead days, assignees resolve on opening)
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

On HR approval: `employees.accessBlockedAt` = **00:00 company time on the day after the LWD** (user, Step 4 — the
employee works the notice and does the exit interview first). From then on, sign-in and token refresh are refused once
every active employee record of the user is blocked (god roles never). A later LWD change moves it. Full deactivation
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
  `SeparationService` / `SeparationController` (`api/separation`) / `SeparationDtos.cs` ✅ (Step 3); Step 4 ✅ `NotificationService.Separation.cs`,
  `SeparationApprovalRules.cs`; Step 5 ✅ `SeparationTaskRules.cs`, `ISeparationTaskService` / `SeparationTaskService`,
  `SeparationTasksController` (`api/separation-tasks`), `SeparationTaskBackgroundService`, `SeparationTaskDtos.cs`.
- **Frontend:** `src/modules/separation/` — `masters/SeparationConfigView.tsx` ✅, `components/` ✅
  (SeparationStagesTab, StageMappingDrawer, SeparationGeneralTab, SeparationChecklistTab, ChecklistItemDrawer),
  `api.ts`, `types.ts`, `constants.ts`, `index.ts` ✅; Step 3 ✅: `portal/PortalSeparationView.tsx`, `components/`
  SeparationRequestDrawer (tabs Resign / Change LWD / Withdraw → InitiateResignationForm, LwdRequestForm,
  WithdrawResignationForm), SeparationCaseCard, EmploymentFactsCard, SeparationTimeline; Step 5 ✅: `tracker/SeparationTrackerView.tsx`,
  components SeparationCaseHeader, ClearanceTaskDrawer, SeparationTrackerDrawer; Step 6 ✅: SeparationTaskDrawer (router used by
  /task), FnfProcessDrawer, FnfPaymentDrawer, FnfSummary, ClearancesReadOnly, TaskDocuments, TaskDrawerParts, useSeparationTask;
  to come: ExitInterviewDrawer.
  Step 8 ✅ (report, own folder per follow.md #19): `src/modules/reports/separation/` — SeparationReportView, components
  SeparationReportKpiCards, SeparationReportFilterCard, SeparationReportTable, ExitInsightsPanel; backend
  `SeparationReportController` (`api/reports/separation`), `ISeparationReportService` / `SeparationReportService`,
  `DTOs/Reports/SeparationReportDtos.cs`, `sp_ReportSeparationExitStats`.
  Routes: `src/app/(dashboard)/reports/separation/page.tsx` ✅, `src/app/(dashboard)/masters/separation-config/page.tsx` ✅, `src/app/(dashboard)/portal/separation/page.tsx` ✅, `src/app/(dashboard)/employees/separation-tracker/page.tsx` ✅.

---

## 8. Build steps

| Step | Scope | Status |
|---|---|---|
| 1 | DB + domain foundations: script, entities, DbContext, menus, permissions, grants, seeds; verified on a DB clone (2 runs, idempotent) and applied to `HRMSCore_Local2`. Add Employee auto-fill confirmed. | ✅ Done 2026-10-10 |
| 2 | **Separation Config** (Masters & Configuration): API + screen — Stages & mapping (employee mapping drawer), General, Checklists; `sp_SeedSeparationConfig`; view/manage permissions; grants fixed to real role ids. | ✅ Done 2026-10-10 |
| 3a | **Employee lifecycle in days:** probation days everywhere (months removed), `employmentStatus`, probation end / separation / LWD dates, probation notice days on the Employment Type master. | ✅ Done 2026-10-10 |
| 3b | **ESS Initiate resignation:** portal page, Separation Request drawer (Resign / Change LWD / Withdraw), LWD calc (confirmed / probation), LWD by agreement, recovery days, withdraw. | ✅ Done 2026-10-10 |
| 4 | **Approvals + notifications:** line manager + LWD request + HR on `/task`; email + bell per stage mapping; `IsApproverAsync` for mapped employees; login block; `employmentStatus = NOTICE`. | ✅ Done 2026-10-10 |
| 5 | **Clearance tracker:** task generation, trigger date job, parallel / sequential, checklist + handover / KT upload, retrigger. | ✅ Done 2026-10-10 |
| 6 | **HR final clearance + F&F Process + F&F Paid.** | ✅ Done 2026-10-10 |
| 7 | **Exit interview + Experience cum Relieving Letter + release.** Full cross-role E2E + report. | ✅ Done 2026-10-10 |
| 8 | **Separation Reports** (D1) — `/reports/separation`, `modules/reports/separation/` (follow.md #19): filters, cards, paged list, exit-interview insights, XLSX export. | ✅ Done 2026-10-10 |
| 9 | **Blob path clean-up** (all modules, `docs/blob_paths.md`) — after the user reviews the folder layout. | ⏳ Planned |
| 10 | **/letters clean-up** (`docs/letters_fixes.md`, L1–L8). | ⏳ Planned |

## 9. Decisions (answered 2026-10-10)

| # | Answer |
|---|---|
| D1 | HR-side screen → **Reports → Separation** (`/reports/separation`), Step 8 ✅. HR roles only; statistics cover every case matching the filters. |
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

### Step 4 — Approvals + notifications (2026-10-10)
- User answers: login blocked **after the LWD** (not at HR approval); reject remarks **required and shown** to the
  employee; a pending LWD request is **closed with** a rejected resignation; default e-mail templates **seeded and enabled**.
- `SeparationApprovalRules.cs` (who acts, /task items, decisions). `ApprovalsService`: block 2c'' in the inbox, 2d'' in
  `DecideApprovalAsync`, review details (facts, LWDs, recovery days, trail), tab `SEPARATION`. Item ids: case id
  (`SEPARATION`), LWD request trail id (`SEPARATION_LWD`). `ApprovalDecisionRequest` + `AcceptRequestedLwd`, `OverrideLwd`.
- Decisions: line manager approve → `PENDING_HR` (pending LWD accepted or kept official in the same step); reject →
  `REJECTED` + dates cleared. HR approve → `HR_APPROVED`, `currentStage` = next active stage, `NOTICE`, clearance trigger
  = LWD − lead days, `accessBlockedAt` = day after LWD (company tz, stored UTC). Later LWD approval re-computes both.
  Override (`separation.lwd.override`, superadmin / developer) → trail `OVERRIDDEN`. Every decision → trail + `audit_log`.
- `UserAccessResolver.IsApproverAsync`: mapped to an active stage, or line manager of an open case → approver grants.
- `AuthService`: login + refresh refused after `accessBlockedAt`.
- `NotificationService.Separation.cs` + 4 events in `NotificationTemplates` (catalog, placeholders, bell defaults);
  `NotifyUsersAsync` got `bell` / `email` switches (stage `notifyBell` / `notifyEmail`). Sent from `SeparationService`
  (submit, LWD request) and after each decision. `sp_SeedSeparationNotifications` (db_changes.sql) — also run by
  `SeparationConfigService` for a company seeded on first open.
- Frontend: `/task` tab **Separation**, entity styles, `TaskDecisionDrawer` separation fields, Accept / Keep LWD choice,
  override date (permission-gated), no Return; `onDecide` now takes an extras object.
- Checks: `dotnet build` 0 errors, `tsc --noEmit` 0 errors, tests 322/322; clone E2E **34/34** (3 cases: approve path with
  earlier LWD + later LWD request, reject with pending LWD, developer override; login/refresh block; mapped ess → task.view;
  14 e-mails SENT). Script ×2 on the clone (60 / 60 rows), then applied to dev (employees 3,876; no cases; nobody mapped).

> **Code correction and static values** (follow.md #10)
> - Bell text falls back to `NotificationTemplates.BellDefaults` (code) when a company has no active template — existing
>   pattern, separation lines added the same way. Stage label "Change of last working day" for LWD items is in code.
> - `ApprovalsService.ResolveApproverContextAsync` still decides "HR" from role codes (`HR_MANAGER`, `COMPANY_HR`, …) for
>   other modules; separation does not use it (mapped employees only).
> - ~~`/task` History tab does not list separation decisions yet~~ — done 2026-10-10: `GetApprovalHistoryAsync` block 2d
>   lists line manager / HR / LWD decisions and overrides (module badge "Separation", action OVERRIDDEN styled).

### Step 5 — Clearance tracker (2026-10-10)
- User answers: a **"No" blocks submit** (resolve first, then Yes / N/A); tracker is a **new HR page** —
  **Employee Management → Separation Tracker** (`/employees/separation-tracker`, `separation.tracker.view`: role_hr,
  company_hr, group_hr); **Retrigger** = `separation.task.retrigger` holders on PENDING (re-notify) or COMPLETED (re-open)
  tasks; the exit-interview task opens with the clearances but is **not notified until Step 7**.
- `SeparationTaskRules.cs`: on HR approval one WAITING task per active stage after `HR_APPROVAL`
  (`configuredTriggerDate` = case trigger date). HR approval is **refused** while an active post-approval stage resolves
  to nobody (message names the stages). `AdvanceAsync` (company date via `CompanyClock`): PARALLEL opens every clearance,
  SEQUENTIAL the next one when none is open; exit interview opens with them; **HR final** opens when every clearance is
  COMPLETED (F&F, letter stay WAITING). Runs after HR approval / LWD decisions, after each completion, and from
  `SeparationTaskBackgroundService` (appsettings `SeparationTasks`: AutoRunEnabled, RunIntervalMinutes = 60).
  An LWD change moves `configuredTriggerDate` of WAITING tasks. Only active stages get tasks, so SEQUENTIAL skips
  switched-off stages.
- Complete: every active checklist line YES / NO / NA (+ remark), any NO → refused; handover / KT notes required where
  the stage `requiresHandover` (L1); answers replace earlier ones in `separation_clearance_responses`; audit
  `TASK_COMPLETED`. Retrigger: stamps `retriggeredDate`, re-resolves assignees, clears completed-by, puts a PENDING HR
  final back to WAITING; audit `TASK_RETRIGGERED`.
- Handover documents: `api/separation-tasks/{id}/attachments` → blob `employee/<companyId>/separation/<separationId>/HANDOVER/`
  (appsettings `SeparationDocuments`: MaxBytes 10 MB, AllowedTypes PDF / JPEG / PNG / DOCX / XLSX, StorageFolder
  `employee`; signature check reused from `LeaveProofStorage.Validate`). Download: tracker viewers in the workspace and
  the task's assignees.
- /task: entity `SEPARATION_TASK` (Separation tab, badge "Clearance"), for the people the stage resolves to;
  **Open Clearance** opens `ClearanceTaskDrawer` (case header, checklist Yes / No / N/A + remark, handover notes +
  uploads, remarks); clearance items are excluded from batch approve. Notices: `NotifySeparationTasksAsync` —
  `SEPARATION_STAGE_ASSIGNED` per open clearance task (stage bell / e-mail switches), key `case|TASK|task|retriggerTicks`.
- Tracker page `SeparationTrackerView` (status / company / search, task counts) → `SeparationTrackerDrawer` (header:
  location, joining, notice, resignation, LWD, recovery days, clearance start; per task: status, configured / actual /
  retriggered dates, assigned to, owner, completed by, answers, notes, documents, Retrigger).
- **Fix to Step 4:** `ApproverEmployeeIdsAsync` now uses each stage's configured "handled by" (`DIRECT_MANAGER` →
  reporting manager, else mapped) instead of hard-wiring the reporting manager to `LINE_MANAGER_APPROVAL` only — L1
  clearance never resolved to the manager before. `IsApproverAsync` also counts the line manager of an HR_APPROVED case.
- DB (`docs/db_changes.sql`, Step 5 block): menu `menu-emp-separation-tracker`, permission `separation.tracker.view`,
  3 grants. No schema change.
- Checks: `dotnet build` 0 errors, `tsc --noEmit` 0 errors, tests 322/322; script ×2 on a clone; clone E2E **50/50**
  (parallel case: refusal while unmapped, 10 tasks, /task routing incl. ess-role mapped people, No / missing answer
  refused, handover upload + bad type + blob path, download rights, HR final opening, tracker, retrigger + re-notify,
  audit, bells) + **12/12** (sequential case: future trigger, LWD change moves dates, scheduled job opens L1 + exit only,
  completion opens the next clearance). 9 `SEPARATION_STAGE_ASSIGNED` e-mails SENT. Applied to dev (employees 3,876;
  no cases; nobody mapped). Pre-change backup `/var/opt/mssql/data/s5.bak`.

> **Code correction and static values** (follow.md #10)
> - The drawer hint "PDF, Word, Excel or image" mirrors `SeparationDocuments:AllowedTypes`; the server enforces the list.
> - `/task` review (`GetTaskReviewAsync`) only serves items still pending, so the tracker has its own read endpoint.
> - DOCX / XLSX uploads are not signature-checked (only PDF / JPEG / PNG are, as for leave proof).
> - Tracker list is capped at 500 cases per query (no paging yet) — Separation Reports (later) will add paging.

### Step 6 — HR final clearance + F&F (2026-10-10)
- User answers: HR final **"No" allowed with a required remark** (shown on the tracker / F&F); **one net F&F amount, may be
  negative** (employee owes the company); amount **entered manually** with reference facts (recovery days, notice, LWD,
  leave balances of the LWD year) — auto-calculation later with payroll; F&F amounts / documents need the new
  **`separation.fnf.view`** (role_hr, company_hr, group_hr, role_fin, finance; superadmin / developer always).
- `SeparationTaskRules`: `AdvanceAsync` opens **F&F Process** when HR final is completed and **F&F Paid** when F&F Process
  is completed. `CompleteAsync` serves clearance + HR groups (HR: No needs a remark). `SubmitFnfAsync` — amount required,
  currency from `companies.currency` (refused if blank), latest `FNF_STATEMENT` upload required → `separation_fnf`
  SUBMITTED, audit `FNF_SUBMITTED`. `PayFnfAsync` — fnf SUBMITTED, paid amount, reference, payment date ≤ company today,
  latest `PAYMENT_EVIDENCE` upload required → PAID, audit `FNF_PAID`. Upload category from the task (L1 → HANDOVER,
  F&F Process → FNF_STATEMENT, F&F Paid → PAYMENT_EVIDENCE). Retrigger puts later PENDING tasks back to WAITING
  (clearance → HR final + F&F, HR final → F&F, F&F Process → F&F Paid) and steps the F&F row back (PAID → SUBMITTED,
  SUBMITTED → PENDING). `PendingAsync` / `NotifySeparationTasksAsync` cover CLEARANCE, HR and FNF groups.
- API `POST api/separation-tasks/{id}/fnf`, `POST {id}/fnf-payment`; task detail adds group, upload category, currency,
  read-only clearances (HR final / F&F), F&F summary, leave balances. Tracker: `canViewFnf`, F&F block; without the
  permission F&F amounts and documents are hidden and their download refused (assignees still open their own).
- Frontend: `/task` "Open Task" → `SeparationTaskDrawer` picks the drawer by task code (from `detailsPayloadJson`):
  checklist (clearance / HR final with read-only clearances), `FnfProcessDrawer`, `FnfPaymentDrawer`; tracker drawer F&F block.
- DB (`docs/db_changes.sql`, Step 6 block): `separation_fnf.paymentDate`, `remarks`, `paymentRemarks`; permission
  `separation.fnf.view` (menu Separation Tracker) + 5 grants.
- Checks: `dotnet build` 0 errors, `tsc --noEmit` 0 errors, tests 322/322; script ×2 on a clone; clone E2E **44/44**
  (mapped HR, Finance, line manager, employee, **developer** god role without an employee record: HR final routing +
  bell, read-only clearances, No without / with remark, F&F refusals — no statement, blank currency, no amount — negative
  amount, routing to Finance, payment refusals — no evidence, future date, blank reference — PAID row, audit, tracker
  with / without `separation.fnf.view`, downloads, retrigger chain) + Step 5 regressions **50/50** and **12/12**.
  30 `SEPARATION_STAGE_ASSIGNED` e-mails SENT on the clone. Applied to dev (employees 3,876). Backup `/var/opt/mssql/data/s6.bak`.

> **Code correction and static values** (follow.md #10)
> - Statement / evidence are the latest upload of the task; earlier uploads stay listed.
> - Leave balances shown are for the year of the LWD from `leave_balances.remaining` (reference only, no encashment maths).
> - The HR final seed line "Exit interview held" is usually "No" with a remark until Step 7 adds the exit interview.

### Step 7 — Exit interview, Experience cum Relieving Letter, release (2026-10-10)
- User answers (all recommendations approved): exit interview opens with the clearances, fillable until sign-in is blocked,
  **HR may skip it with a reason**; answers only for **`separation.exit.view`** (HR roles), not the line manager; **new
  combined template**, primary authorised signatory; **text-only PDF** (no logo / signature image) accepted; **IsActive = 0
  at release** (later payroll runs skip them; F&F carries the final dues); **login deactivated** unless the user has another
  active employee record (god roles never); **leave balances frozen, pending leave cancelled**, release notice lists direct
  reports and stage mappings (nothing reassigned automatically); **letter e-mailed to the personal e-mail** (exit interview,
  else profile); form seeded as the docx, two extra questions switched off. Blob: letter + copies of every document of the
  employee under **`separation/{employeeNumber_employeeName}/final/`** (user request), path clean-up as a later task.
- **Exit interview** — `SeparationExitRules.cs`: the form = company rows of `separation_exit_questions` (REASON / RATING /
  TEXT / SCALE; `sp_SeedSeparationExitQuestions`, also run when Separation Config opens for a company without rows).
  Submit: employee only, task PENDING, case HR_APPROVED, before `accessBlockedAt`; ≥ 1 reason or other reason; every active
  rating on the scale; text optional (2,000 chars); mobile or e-mail (valid). Answers (with the question text) in
  `separation_exit_interviews` (`reasonsJson`, `otherReason`, `ratingsJson`, `textAnswersJson`, `contactJson`), task
  COMPLETED, audit `EXIT_INTERVIEW_COMPLETED`. HR **Skip** (`POST api/separation-tasks/{id}/skip`, retrigger permission,
  reason required) → SKIPPED, audit `EXIT_INTERVIEW_SKIPPED`; **Retrigger** also re-opens a SKIPPED exit interview.
  `ActionableGroups` + EXIT: notice `SEPARATION_STAGE_ASSIGNED` with link `/portal/separation`; /task item flagged
  **`IsSelfTask`** (new on `PendingApprovalDto`; `ApprovalVisibilityRules` rule 0 — only the employee sees / acts; the
  "never your own submission" rule hid it); `UserAccessResolver.IsApproverAsync` counts an employee with an open exit
  interview (so they get `task.view`; permissions are in the token, refreshed at sign-in / refresh). API
  `GET/POST api/separation/{id}/exit-interview` (employee; HR with exit.view read-only).
- **Letter + release** — `SeparationReleaseService.TryReleaseAsync` (after every task action, exit submit / skip, the
  hourly job, and **Retry letter** = Retrigger on the System task). Ready when F&F Paid COMPLETED, exit COMPLETED / SKIPPED,
  **company date > LWD** (added after the E2E showed a case paid during the notice being released before its LWD), letter
  not issued. `LettersService.IssueSystemLetterAsync` (template by code `TPL_EXPERIENCE_RELIEVING`, primary signatory else
  template default, refused when missing — no sample fallbacks; same reference returns the same letter) with extra fields
  `last_working_date`, `relieving_date`, `resignation_date`, `notice_period_days`; reference `<case ref>-ECRL`.
  `Pdf/LetterPdf.cs` (SimplePdf: letterhead text, ref + date, subject, wrapped body, footer note) → blob
  `{SeparationDocuments:FinalStorageFolder}/{empNo_name}/final/experience-cum-relieving-letter-<ref>.pdf`
  (`BlobKeyBuilder.SeparationFinalFolder` / `SafeFileName`), `separation_attachments` EXPERIENCE_LETTER,
  `LettersService.AttachStoredLetterAsync` → `employee_documents` row **with the storage key** (PDF). Archive: candidate
  (`candidates.convertedEmployeeId`), onboarding (`candidate_onboarding_profiles.convertedEmployeeId`), employee documents
  and separation attachments copied (download + upload) to `final/documents/<source>/NNN_<file>`, one `separation_attachments`
  ARCHIVE row each, unreadable files logged and skipped. Release (one save): case RELEASED + `releasedAt`, letter task
  COMPLETED by "System", `employmentStatus` SEPARATED, `status` RESIGNED, `employees.isActive` 0, `users.isActive` 0 (no other
  active record, not god mode), pending leave cancelled via `LeaveService.CancelLeaveRequestAsync`, audit `CASE_RELEASED`
  (details: letter, archive count, leave, login, direct reports, mappings). Failure → letter task PENDING with
  "Not issued yet: <reason>". Notices: `NotifySeparationReleasedAsync` — `SEPARATION_LETTER_ISSUED` bell to the employee +
  e-mail to the personal address **with the PDF attached** (`SendOnceAsync` takes attachments now); `SEPARATION_RELEASED` to
  the HR final clearance people (their stage switches). Download `GET api/separation/{id}/letter` (employee, tracker
  viewers in the workspace, superadmin / developer).
- Frontend: `ExitInterviewDrawer` (portal case card button + /task via `SeparationTaskDrawer`, `separationId` from the
  payload), `ExitInterviewAnswersView`, case card letter download; tracker drawer: release / archive banner, exit answers
  (exit.view), **Skip** with reason, **Retry letter**; Separation Config tab **Exit Interview** (`SeparationExitFormTab`,
  `ExitQuestionDrawer`, section labels in `constants.ts`).
- DB (`docs/db_changes.sql`, Step 7 block): table + seed proc + seeding (450 rows, 15 companies), exit interview columns,
  template for the 5 companies with letter templates (others get it from `LettersSeedData` on first /letters use),
  `sp_SeedSeparationNotifications` re-created with the two new events (30 settings), permission + 3 grants.
- Checks: `dotnet build` 0 errors, `tsc --noEmit` 0 errors, tests 322/322; script ×2 on a clone; **clone E2E 90/90** (report
  below). Applied to dev (employees 3,876; no cases). Pre-change backup `/var/opt/mssql/data/s7pre.bak`.
- **E2E report** (clone `HRMSCore_SepTest`, API :5299, local blob folder, Ethereal SMTP): config (30 seeded lines, company
  name in the statement, add / edit / reorder, 6th scale point refused, employee 403). **Case A** (william.gitau): resign →
  line manager (trevor) → HR (hr) → exit interview on the employee's /task + bell → L1 (handover upload), Admin + IT
  (peter), Finance (christine), Audit (douglas) → HR final → F&F Process → exit interview (line manager 403, HR read-only,
  HR submit 403, 4 validation refusals, submit) → letter waits → F&F Paid (ranmeet) → **not released before LWD**, Retry
  refused with the reason → LWD over → released: status / employment / employee + user inactive, letter in /letters with
  PDF in employee documents, merged text, PDF on disk, downloads (developer, HR; line manager 403), 4 documents archived
  (missing one skipped), tracker, e-mails (letter to the exit e-mail with PDF, release to HR), bells, audit, pending leave
  cancelled, sign-in refused. **Case B** (milton.olenyo): F&F Paid with exit open → not released; Finance skip 403, skip
  without reason refused; signatory inactive → skip → letter PENDING "no active authorised signatory", Retry refused;
  retrigger skipped exit → employee submits (mobile only); signatory back → Retry letter → released, same release checks.

> **Code correction and static values** (follow.md #10)
> - `LettersService.RenderLetterAsync` / `IssueLetterAsync` (manual /letters) still fall back to sample values ("Sarah
>   Omolo", "Director of Human Capital", "Consultant Physician", "Internal Medicine", "EMP-00101", "HR Operations Manager")
>   and a random `LTR-HRMS-yyyy-####` reference that can repeat; `PdfUrl` points at a non-existent endpoint. The System
>   path does not use them — left for a /letters clean-up.
> - The seeded default signatory "Sarah Omolo" and letterhead contacts (hr@hospital.co.ke) are seed data, not real
>   company data — HR should set real signatories / letterheads in /letters before go-live.
> - "Employment Letters" document type is still created in code when missing (`AttachToEmployeeDossierInternal`).
> - Contact fields of the exit interview (P.O. Box, postal code, town, mobile, e-mail) are a fixed structure, not a master
>   (the letter e-mail depends on the e-mail field).
> - The section labels of the Exit Interview tab (`EXIT_QUESTION_TYPES`) are display text for the type codes.
> - After release the employee cannot sign in, so the portal download is for superadmin / developer; the employee gets
>   the letter by e-mail.

### Step 8 — Reports › Separation (2026-10-10)
- User: report at **`/reports/separation`**, UI in its own folder **`modules/reports/separation/`** — now a rule for every
  report (follow.md #19); permissions `reporting.separation.view` / `.export` (Reports naming); **HR roles only**;
  statistics cover **every case matching the filters**.
- API `api/reports/separation` (`SeparationReportController` → `SeparationReportService`): `GET filters` (workspace
  companies; departments, locations, stages that occur on cases), `GET` (paged list + summary, page size ≤ 100),
  `GET exit-stats` (`separation.exit.view`, else 403), `GET export` (XLSX, `reporting.separation.export`, audit row
  `SeparationReport` / `EXPORT`). One filter query feeds list, summary, statistics and export. F&F fields are blanked
  without `separation.fnf.view`. Stage names come from `separation_stages`.
- `sp_ReportSeparationExitStats @SeparationIds` — OPENJSON over `reasonsJson` / `ratingsJson` of COMPLETED interviews;
  current question text and order from `separation_exit_questions` (answer copy as fallback).
- Screen `SeparationReportView`: unboxed header + Refresh / Export XLSX, filter card, 6 cards, tabs Resignations /
  Exit Interview Insights (only with exit.view), paged table; a row opens `SeparationTrackerDrawer` (now exported
  from `modules/separation`) for `separation.tracker.view` holders.
- DB (`docs/db_changes.sql`, Step 8 block): menu `menu-rep-separation`, 2 permissions, 6 grants, the procedure.
- Checks: `dotnet build` 0 errors, `tsc --noEmit` 0 errors, tests 322/322; script ×2 on a clone; clone E2E **36/36**
  (6 seeded cases in two companies — HR, line manager with a clone-only view grant, employee, developer). Applied to dev
  (employees 3,876; no cases). Pre-change backup `/var/opt/mssql/data/s8pre.bak`.

> **Code correction and static values** (follow.md #10)
> - The XLSX status column uses a status → label map in `SeparationReportService` (same text as the screen's
>   `SEPARATION_STATUS`); the F&F status shows the stored code (PENDING / SUBMITTED / PAID).
> - With a group workspace, stage names come from the first company that defines the stage code.
> - The Separation Tracker list still caps at 500 cases (no paging); the report has paging.

### Fix — General "days before LWD" applies to approved cases not yet in clearance (2026-10-10)
- User report: lead days set to 30 after SEP-2026-0001 was HR-approved; the tracker still showed the start date 24 Oct
  (fixed at approval with 7). Decision: saving General **re-dates HR-approved cases with no task triggered yet** (case +
  WAITING tasks) and opens what is now due with notices (`RunForCasesAsync`); cases already in clearance unchanged.
  Count shown on save and written to the audit row. policies.md §94.

### Fix — separation document upload returned 415 in the browser (2026-10-10)
- `separationTasksApi.upload` (L1 handover, F&F statement, payment evidence) sent the FormData with the `apiClient`
  default `Content-Type: application/json` → ASP.NET 415. Now sends `multipart/form-data`, like the other modules'
  uploads. The clone E2E had called the API directly, so it did not catch this.

### Fix — portal Status timeline ignored the tracker tasks (2026-10-10)
- `/portal/separation` timeline was positional on `currentStage`, which stays on the first post-approval stage after HR
  approval, so it showed "L1 Manager clearance — In progress" with everything later waiting even when all tasks were done.
  Now (`SeparationService.TimelineState`): once tasks exist (open or RELEASED case) a stage with a task is DONE when
  COMPLETED / SKIPPED, CURRENT ("In progress", several in parallel) when PENDING, else WAITING; approval stages before the
  first task are DONE. Before HR approval the positional rule is kept.

## 11. Next-step prompts — paste each into a new chat, in this order

Order (user, 2026-10-10): **Step 8 Separation Reports** ✅ (`/reports/separation`, follow.md #19) →
**Step 9 blob-path clean-up** → **Step 10 /letters clean-up**.

### 11.1 Step 9 — blob-path clean-up

```
Read docs/follow.md and follow every rule. Read docs/blob_paths.md (inventory, proposed target, migration method),
docs/sepration.md §10 Step 7 (final/ archive folder) and memory "separation-module-plan" + "keep-usage-lean".
Task: clean up every blob path in all modules.
1) Re-check the inventory in docs/blob_paths.md §1 against the code (every writer of a storage key: BlobKeyBuilder,
   DocumentsService, Recruitment/Onboarding, EmployeeService certificates, AuthorizedSignatories, LeaveProofStorage,
   ClockEvidence, SeparationTaskService, SeparationReleaseService, LettersService) and list anything missing.
2) Present the final folder layout + key format per file type + which DB column holds each key, flag static values
   (follow.md #10), and wait for my approval. Folder roots come from appsettings, not code.
3) After approval: one BlobKeyBuilder for all writers; a migration (copy → update key → verify the file opens → delete
   old only after verify) run on a throwaway DB clone + copied blob folder first; DB changes in docs/db_changes.sql
   (idempotent); a dry-run report (files found / moved / missing).
4) Verify: tsc --noEmit + dotnet build clean, tests pass, downloads work per role (employee, line manager, HR) for each
   file type; then apply to dev with a backup.
Docs after: docs/blob_paths.md (mark done), docs/policies.md (next section), docs/sepration.md step table, the
sep-workflow artifact (Build steps + manual test), memory. End with the prompt for Step 10 (docs/sepration.md §11.2).
```

### 11.2 Step 10 — /letters clean-up

```
Read docs/follow.md and follow every rule. Read docs/letters_fixes.md (L1–L8), docs/blob_paths.md (final layout from
Step 9) and docs/sepration.md §10 Step 7 (System letter path: LettersService.IssueSystemLetterAsync, Pdf/LetterPdf.cs —
already clean, reuse it). Memories "separation-module-plan", "no-static-values-use-masters", "keep-usage-lean".
Task: clean up the manual /letters module.
1) Verify each item L1–L8 in code and data (counts from the dev DB, read-only), add anything else found.
2) Present the plan (follow.md #9) with a "Code correction and static values" section and wait for my approval.
   Ask me about L8 (template wording) before touching HR text.
3) After approval: no sample fallbacks (refuse with the missing field), per-company letter numbering, every issued letter
   stored as a PDF under the Step 9 layout and attached to employee documents with its key, a real permission-checked
   download, "Employment Letters" document type seeded, seed signatory / letterhead not looking real + readiness banner.
   Back-fill empty employee_documents.storageKey rows on a clone first. DB changes in docs/db_changes.sql (idempotent).
4) Verify: tsc --noEmit + dotnet build clean, tests pass, clone E2E as HR (issue, preview, download, refusals) and
   employee (own letters only), Experience cum Relieving Letter regression (release path).
Docs after: docs/letters_fixes.md (tick items), docs/policies.md (next section), memory.
```
