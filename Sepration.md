# Separation (Resignation) Module — Design & Implementation Plan

> Status: **revision 5 (2026-10-10). Steps 1–2 ✅ done. Next: Step 3 — ESS Initiate resignation (prompt in §11).**
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

- **Confirmed employee:** LWD = resignation date + notice period days.
- **Probation employee:** LWD = resignation date + `separation_config.probationNoticeDays` (7, configurable per company).
- Notice and probation days come from the **Employment Type master**, copied onto the employee at hire. Add Employee
  already fills them from the selected type and lets HR edit them (`CreateEmployeeModal.tsx` — no change needed).
- **LWD change:** the employee requests a date + reason → the line manager approves → LWD is fixed.
  superadmin / developer can set it directly (`separation.lwd.override`).
- **Recovery days** = notice days not served (notice period − days between resignation and LWD); shown in the tracker
  header and carried into F&F.

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
2. **General** — probation notice days (≥ 1), days before LWD to start clearance (≥ 0), Parallel / Sequential.
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
| `separation_config` | Per company: `probationNoticeDays`, `clearanceTriggerLeadDays`, `clearanceMode`. |
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
  `Api/Controllers/SeparationConfigController.cs` ✅ (`api/separation-config`); to come: `ISeparationService` /
  `SeparationService`, `NotificationService.Separation.cs`, `SeparationController`.
- **Frontend:** `src/modules/separation/` — `masters/SeparationConfigView.tsx` ✅, `components/` ✅
  (SeparationStagesTab, StageMappingDrawer, SeparationGeneralTab, SeparationChecklistTab, ChecklistItemDrawer),
  `api.ts`, `types.ts`, `constants.ts`, `index.ts` ✅; to come: portal view + InitiateResignationDrawer,
  LwdRequestDrawer, TaskActionDrawer, HandoverDrawer, FnfDrawer, FnfPaymentDrawer, ExitInterviewDrawer, SeparationTracker.
  Routes: `src/app/(dashboard)/masters/separation-config/page.tsx` ✅, `src/app/(dashboard)/portal/separation/page.tsx` (Step 3).

---

## 8. Build steps

| Step | Scope | Status |
|---|---|---|
| 1 | DB + domain foundations: script, entities, DbContext, menus, permissions, grants, seeds; verified on a DB clone (2 runs, idempotent) and applied to `HRMSCore_Local2`. Add Employee auto-fill confirmed. | ✅ Done 2026-10-10 |
| 2 | **Separation Config** (Masters & Configuration): API + screen — Stages & mapping (employee mapping drawer), General, Checklists; `sp_SeedSeparationConfig`; view/manage permissions; grants fixed to real role ids. | ✅ Done 2026-10-10 |
| 3 | **ESS Initiate resignation:** portal page, Initiate drawer, LWD calc (confirmed / probation), recovery days, LWD change request, withdraw. | ⏳ Next |
| 4 | **Approvals + notifications:** line manager + LWD request + HR on `/task`; email + bell per stage mapping; `IsApproverAsync` for mapped employees; login block. | |
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

## 11. Next-step prompt (Step 3) — paste into a new chat

```
Read docs/follow.md and follow every rule. Read docs/sepration.md (plan, §1, §8 step table, §10 step log) and memory
"separation-module-plan". Steps 1–2 of the Separation (resignation-only) module are done.

Do Step 3 — ESS "Initiate resignation" on Self Service → Separation (/portal/separation, menu-separation, permission
separation.view; actions separation.initiate / .withdraw / .lwd.request).

Scope:
1. Backend: ISeparationService + SeparationService + SeparationController (api/separation), DTOs in
   Application/DTOs/Separation. Endpoints: my separation (current case + my employment facts), LWD preview, initiate,
   request LWD change, withdraw (only before HR approval).
2. Facts come from the DB, never literals: notice days / probation months / status / confirmation date are on
   EmploymentDetail (NoticePeriodDays, ProbationPeriodMonths, Status, ConfirmationDate); probation notice days from
   separation_config. Confirmed: LWD = resignation date + notice days. Probation: + probationNoticeDays.
   Recovery days = notice days not served. Reference number: follow the existing numbering pattern used elsewhere.
3. Refuse initiation (clear message) when: an open case exists; the line-manager stage cannot be resolved (no reporting
   manager and nobody mapped) or HR_APPROVAL has nobody mapped (point to Masters & Configuration → Separation Config).
4. Status after submit = PENDING_L1 with a separation_approvals trail. /task routing, emails and bell notices are Step 4 —
   do not build them here.
5. Frontend: thin route src/app/(dashboard)/portal/separation/page.tsx; portal view in src/modules/separation; page root
   space-y-6, unboxed header, one action bar; InitiateResignationDrawer and LwdRequestDrawer as 50% SideDrawers (no
   modals); show employment facts, computed LWD, status timeline, Withdraw.

Before coding, present the Step 3 plan for approval (follow.md #9), including these open points:
- Is LWD = resignation date + N days, or + N − 1 (the HR note reads "resign 1 Oct → LWD 30 Oct" for 30 days)?
- What counts as "probation": EmploymentDetail.Status = PROBATION, or no ConfirmationDate / confirmation date in future?
- Can the employee pick a resignation date in the past or future, or is it always today?

Every DB change goes in docs/db_changes.sql (idempotent; live schema differs from docs/dbscript/tables_proc_all.sql —
check INFORMATION_SCHEMA). Test writes only on a throwaway DB clone. After the step: tsc --noEmit and dotnet build clean,
role E2E (employee, line manager, HR), update docs/sepration.md (step table, step log, next prompt), docs/policies.md
(next section after §86), the sep-workflow artifact (Build steps + manual test), memory, and write the Step 4 prompt.
```
