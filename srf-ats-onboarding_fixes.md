# Requisitions (SRF), Recruitment (ATS) & Onboarding — Test Findings & Fix Plan

> Status: **revision 5 (2026-10-05) — Steps 1–4 done (see §7). Next: Step 5 (prompt in §8).**
> Artifact: **srf-ats-ob** (link in §9). Reference: `docs/HRMS User Manual.pdf` §6–§8, §19.
> Each step is reviewed/approved before code (follow.md #9). Every DB change goes to `docs/db_changes.sql` as a dated,
> idempotent block (follow.md #15). New actions get a `permissions` row in `<module>.<action>` form (follow.md #8).
> After each step: zero `tsc --noEmit` errors, zero `dotnet build` errors, `dotnet test` green (follow.md #11), and the
> E2E flow in §6 re-run on a throwaway DB clone (follow.md #12).

## Progress

| Step | Scope | Status |
|---|---|---|
| 1 | Security: permission checks on Onboarding + Approval-matrix endpoints + requisition matrix drawer | ✅ Done 2026-10-04 |
| 2 | SRF decisions only through the approval rules (no step skipping, no self-approval) + one rule drawer on all 5 matrix pages | ✅ Done 2026-10-05 |
| 3 | Offer approval routing (designated user, all roles, same company, no hard-coded chain) | ✅ Done 2026-10-05 |
| 4 | SRF approval routing (department head = submitter, matched rule, no hard-coded chain, rule-save validation) | ✅ Done 2026-10-05 |
| 5 | Reject / Return for revision / Resubmit for SRF and Offer (+ vacancy openings, standard SRF rule per company) | ✅ Done 2026-10-05 |
| 6 | Notifications for SRF and Offer steps | ⏳ Next |
| 7 | Onboarding: form declaration, activation defaults, activation validation | ⏳ Pending |
| 8 | After activation: leave balances (vacancy filled count done in Step 5) | ⏳ Pending |
| 9 | Small data fixes + static values + merge `requisition` → `requisitions` + retire `/masters/srf-matrices` | ⏳ Pending |
| 10 | MAKL set-up (data through the UI) + live test users | ⏳ Pending (needs your go-ahead for live writes) |
| 11 | Full end-to-end re-test across all roles + report | ⏳ Pending |

---

## 0. Decisions (answered 2026-10-04)

| # | Question | Decision |
|---|---|---|
| D1 | No REQUISITION / OFFER matrix rule matches the company | **Refuse the submission** ("No approval matrix set up for <company> — ask HR to add one"). Rules are set per company on **`/approval-matrices/requisitions`** and **`/approval-matrices/offers`**. Each step names a **role or a specific employee**, and the step must land in exactly that person's / role's **`/task`** bucket. No hard-coded chains (follow.md #1). |
| D2 | The SRF submitter is the head of the department | The head's own reporting manager; if none, the company's Company HR approver. Shown as "auto-escalated" with the reason. |
| D3 | Approve/Reject buttons on `/requisitions` | **None.** All approvals live in `/task`. The `/requisitions/{id}/approval-decision`, `budget-validate` and `hr-review` endpoints apply the same rules as `/task`, or are removed once the UI no longer calls them. |
| D4 | Where a returned SRF restarts after edit + resubmit | **Step 1.** Previous decisions stay in the history. |
| D5 | Leave balances for a new hire created by activation | Created the same way Add Employee does (current leave master and accrual rules). The manual (§8) also asks for "opening leave balances". |
| D6 | Duplicate frontend modules | **Merge into `src/modules/requisitions/`, then delete `src/modules/requisition/`** (see §0.2). |
| D7 | Interview score scale | Accepted as 1–5, **but the User Manual §6 says 1–10** across 5 competencies. Confirm at Step 9 (default there: follow the manual, 1–10, so the overall score is the sum ÷ 50 × 100). |

### 0.1 Standing rules for this work
- **Superadmin and developer (god mode) can see and do everything.** Every new check in this plan (permissions, "not your step", "not your own request", company limits) lets `IsSuperAdmin || IsDeveloper` through, and the audit row says it was an override. This matches the /task rule in memory "approvals live in /task".
- Each step runs in a **new chat** using the prompt in §8. When a step is done, its completion notes go in §7, the progress table is updated, the artifact **srf-ats-ob** is updated (§9), and the prompt for the next step is checked.

### 0.2 D6 — what exists today (checked 2026-10-04)

| Item | Uses | Notes |
|---|---|---|
| Route `/requisitions` (Requisition Dashboard menu) | `modules/requisition` → `RequisitionDashboardView` | keep the route |
| Route `/requisitions/srf` (All Requisitions menu) | `modules/requisition` → `RequisitionsSrfView` + `srf/drawers/*` | keep the route |
| Route `/masters/srf-matrices` (SRF Approval Matrices menu) | `modules/requisition/masters/SrfApprovalMatricesView` → API `/requisitions/approval-matrices` | **edits the same `approval_matrices` table as `/approval-matrices/requisitions`** (API `/task/matrix-rules`). Two screens for one thing. |
| `modules/requisitions/` (`RequisitionsView.tsx` 779 lines + 6 components, `api.ts`, `types.ts`) | imported by **no route** | an older copy; its `types.ts` differs from the live one |
| URL `/requisition` | no route exists → "Page Under Construction" (screenshot) | nothing to keep |

Merge plan (Step 9):
1. Move `dashboard/`, `srf/`, `api.ts`, `types.ts` and `index.ts` from `modules/requisition/` into `modules/requisitions/`.
2. Take anything useful in the old `RequisitionsView`/components that the live screens lack, then delete the old copies.
3. Point the two route files at `@/modules/requisitions`.
4. Retire `/masters/srf-matrices`: hide its menu row (`menu-mst-srf-matrices`, script in `docs/db_changes.sql`) and redirect the route to `/approval-matrices/requisitions`.
5. Delete `modules/requisition/`.
6. `tsc --noEmit` clean.

### 0.3 Alignment with `docs/HRMS User Manual.pdf` (§6, §7, §8, §19)
- **SRF (§7, §19.2):**
  - HOD raises → Finance (BUDGET_AVAILABLE / **BUDGET_EXCEEDED**) → HR technical review → **MD/CEO only for executive grades, unbudgeted posts or high salary** → vacancy auto-created.
  - Reject/return at any stage → remarks logged, **HOD notified to revise or cancel** (Steps 5, 6).
  - The MD/CEO condition is expressed as matrix rules (salary range / grade), not code.
- **Offer (§19.3):** HR Manager → MD/CEO. Configured on `/approval-matrices/offers` per company (Step 3).
- **ATS (§6):** the 5-stage board (Applications → OL Creation → Offer → Hired → Final) and the public acceptance page `/offer/accept/[token]`. Scores are 1–10 (D7).
- **Onboarding (§8):**
  - 4 Kanban stages, a 7-step candidate portal and a token valid for **14 days**.
  - Send Back emails the candidate.
  - Activation, done atomically: employee code, employee record, login with a temporary password, **opening leave balances**, and salary + bank records (Steps 7, 8).

## 1. How the testing was done (2026-10-04)

- Throwaway DB copy `HRMSCore_SrfTest`, with a test API on :5299 and biometric sync off. Your DB and your API on :5197 were not touched.
- Company: **Medical Administrators Kenya Ltd** (`comp-makl-01`).
- Users were created with the real Add Employee API, and roles and passwords were set with the Admin › Users API. Password: `MaklTest@2026`.

| Persona | Login | Name | Role | Department | Reports to |
|---|---|---|---|---|---|
| Employee 1 | 70061 | Achieng Testemp | Staff Employee | Contact Centre | Kamau |
| Employee 2 | 70062 | Baraka Testemp | Staff Employee | IT | Wanjiru |
| Manager 1 | 70057 | Kamau Testmgr | Line Manager, head of Contact Centre | Contact Centre | – |
| Manager 2 | 70058 | Wanjiru Testmgr | Line Manager, head of IT | IT | – |
| Finance approver | 70059 | Mutua Testfin | Finance | Finance & Accounts | – |
| Company HR approver | 70060 | Njeri Testhr | Company HR | Human Resources | – |

These users exist **only in the clone**. Creating them on the live DB was blocked by the permission guard and needs your go-ahead (Step 10).

**What passed:**
- SRF: Finance → HR → Department Head routing and /task visibility for each role. "Not assigned to you" for everyone else. The vacancy was created automatically on final approval.
- ATS:
  - Candidates, interviews, scorecards and stage moves.
  - Offer approval and automatic dispatch with a single-use link.
  - The public offer page: decline needs remarks, a reused link is refused, and a bad link returns 404.
  - Finance rejection moved the candidate to FINAL.
- Onboarding:
  - Starting onboarding again revokes the old link.
  - The candidate can fill in and submit the form.
  - HR send-back (remarks required) and resubmit.
  - HR approve, then activate: employee, job, salary, bank details and login created, with 7 audit rows.
- Automated tests: 154/154 pass.

---

## 2. Findings

Severity: 🔴 security / blocks the flow · 🟠 wrong result · 🟡 minor.

| # | Sev | Finding | Evidence / where |
|---|---|---|---|
| F1 | 🔴 | Any logged-in user can create/delete approval-matrix rules | `TaskController` `POST/DELETE matrix-rules` only has class-level `[Authorize]`. Employee 70061 created and deleted a rule. |
| F2 | 🔴 | `OnboardingController` has no permission policy on any of its 21 endpoints | Employee 70061 read `/onboarding/pipeline` and `/onboarding/review/{id}`, including the candidate's bank account number. |
| F3 | 🔴 | The SRF requester can approve the last step directly and skip Finance and HR | `RequisitionService.ProcessApprovalDecisionAsync` (line ~691) does not check the current step or the approver. Kamau sent step 3 → APPROVED and a vacancy was created. The frontend calls it from `modules/requisition/api.ts:71`. `budget-validate` and `hr-review` have the same gap. |
| F4 | 🔴 | The offer creator can approve their own offer | HR 70060 created OFF-2026-0101 and approved step 1. The "never your own submission" check does not cover offers (`SubmittedById` is not the creator). |
| F5 | 🔴 | Offer steps ignore the designated user from the Approval Matrix page; CUSTOM and FINANCE roles go to "first employee in the company" | `ApprovalsService.ResolveOfferApproverAsync` (line ~105) only handles HR / Director / Dept. The Wanjiru step went to Elian Njuguna, who couldn't act on it. |
| F6 | 🔴 | Offer step assigned to an employee of **another company** | The MD step of OFF-2026-0101 went to Winfred Ndichu `emp-AFRI-12914`. The fallback searches globally. |
| F7 | 🔴 | When the requester is the head of the department, the SRF Department Head step can't be actioned by anyone except superadmin | `ApprovalsService.ResolveRequisitionApproverAsync` (line ~274) has no self-approval escalation (the `ApprovalEngine` DEPT_HEAD case does). |
| F8 | 🟠 | Hard-coded approval chains are used when no rule exists | `RequisitionService.GetFallbackStepDefinitions` (Finance → HR → Dept Head). Offer: `ApprovalsService` line ~950 (HR → Managing Director). Breaks follow.md #1. |
| F9 | 🟠 | The designated-user lookup reads a different rule than the one the SRF matched | `ResolveRequisitionApproverAsync` step 2 takes the REQUISITION rule with the highest `Priority`; `MatchApprovalMatrixRuleAsync` takes the lowest. The SRF does not store which rule it used. Two MAKL rules both have priority 100, so the order is undefined. |
| F10 | 🟠 | ✅ Step 5 — "Return for revision" in /task sets the SRF to **REJECTED**; there is no edit/resubmit | SRF-2026-0719B → `REJECTED`. `/requisitions` sets `DRAFT` for the same action. `RequisitionsController` has no update or resubmit endpoint. |
| F11 | 🟠 | ✅ Step 5 — Rejection with empty remarks is accepted (SRF, Offer) | SRF-2026-DAC6D was rejected with `remarks:""`. `ats.md` step 3 says remarks are mandatory. |
| F12 | 🟠 | No notifications for any SRF/Offer event | 0 rows in `notifications` for all 6 users after the full flow. |
| F13 | 🟠 | Activation crashes with a raw DB error when the NSSF number is missing | `salary_details.nssfNumber` is NOT NULL. The user sees "An error occurred while saving the entity changes". |
| F14 | 🟠 | Activation form defaults are "first item in the list", not the SRF/offer/onboarding values | `OnboardingService` lines ~1599–1605. Job title Accountant (should be IT Technician), grade A0 (A1), reporting manager Achieng, a plain employee (Wanjiru was chosen at initiate), cost centre Clinical (ICT), probation 3 (offer: 6). |
| F15 | 🟠 | Activation silently falls back to another department/location/grade/cost centre, even another company's | `OnboardingService` lines ~1738–1768: `?? FirstOrDefault(company) ?? FirstOrDefault()` with `IgnoreQueryFilters()`. |
| F16 | 🟠 | The candidate's declaration is not enforced | Form submitted with `declarationConfirmed=false` and accepted. |
| F17 | 🟠 | Vacancy part ✅ Step 5; leave balances → Step 8 — Vacancy still OPEN with `filledCount=0` after its only opening was filled; no leave balances for the new hire | VAC-2026-E6F39 after activating employee 70063. `PROCESS_FLOW.md` §3.1 step 7 expects leave balances. |
| F18 | 🟡 | `/task` tier counter is one behind for offers (step 2 shows 1/3, step 3 shows 2/3); assigned name shows the wrong person | Same resolver as F5. |
| F19 | 🟡 | Offer currency saved empty when not sent | OFF-2026-0101 `currency=""`. The SRF already falls back to the company currency. |
| F20 | 🟡 | Approver name stored as the login code ("70059") instead of the person's name | `requisition_approvals.decidedByName`, `finalApprovedByName`. |
| F21 | 🟡 | Interview scores not range-checked | Sending 1–10 values gave an overall score of 164%. |
| F22 | 🟡 | `positionTitle` sent on offer create is not saved | The public offer page shows `positionTitle: null`. |
| F23 | 🟡 | Onboarding "proposed reporting manager" defaults to an unrelated employee | `initiate-data` proposed Elian Njuguna for an IT hire. It should be the SRF requester or the department head. |
| F24 | 🟡 | Duplicate frontend modules + duplicate matrix screen | See §0.2: `modules/requisitions/` is an unused older copy; `/masters/srf-matrices` and `/approval-matrices/requisitions` edit the same table. |
| F25 | 🟠 | An approval-matrix rule can be saved with no name and no steps | Found in Step 1: `POST /task/matrix-rules` and `/requisitions/approval-matrices` with `{}` saved empty REQUISITION rules (clone only, deleted). `ApprovalsService.SaveMatrixRuleAsync` does not validate. Fix in Step 4. |
| F26 | 🟠 | Leave, Attendance & Shifts and Payroll matrix rules are saved but never applied | Found in Step 2: `ApprovalEngine.MatchMatrixRuleAsync` has no callers; only REQUISITION and OFFER rules are read. Leave/regularisation/swap route by reporting manager → HR, payroll by `payroll.approve`. Those 3 drawers now say so. Outside SRF/ATS/OB — needs its own piece of work. |

### MAKL set-up gaps (data, not code)

- **No MAKL department has a head.** Every Department Head step stalls. On the clone, Kamau → Contact Centre and Wanjiru → IT were set via `PUT /api/org/departments/{id}`.
- **No OFFER rule.** `rule-offer-makl` is typed REQUISITION, is named "Offer", and has two CUSTOM steps with no user.
- **No onboarding templates.** The only 4 belong to `comp-lch-01`. Initiate accepted another company's template.
- **Found in Step 3 (all companies, live DB, read-only check):** the 13 live OFFER rules all read HR_REVIEW → MANAGING_DIRECTOR. No active employee holds the `MANAGING_DIRECTOR` role code, and 11 of the 13 companies have no HR-role holder. The old code found the MD by job-title text; that guess is gone (D1), so **these companies can't submit an offer until HR fixes the rule**: make the MD step a designated employee on `/approval-matrices/offers`, or give the MD a role with that code. There were 0 pending offers, so nothing in flight broke. MAKL has no OFFER rule at all (above).

- **Found in Step 4 (live DB, read-only check):** only **2 of 15 companies** have an active REQUISITION rule (Afrihospital Holdings "Senior Consultant & Executive (> 200k)", MAKL "Staff Personnel (< 200k)"). No "ALL" rules exist. Since Step 4 (D1) the other **13 companies can't submit an SRF** until HR adds a rule on `/approval-matrices/requisitions`. MAKL's rule stops at 200k, so a MAKL SRF above 200k is refused too. There is 1 pending live SRF (SRF-2026-CF393, JIVA); its approver is resolved from its own steps the first time it shows in /task.

- **Found in Step 5 (live DB, read-only check, 2026-10-05):** the standard SRF rule now exists for all 15 companies (13 added: Finance → HR → Department Head, by role). But **no live company has a department head**, only LIFECARE has a Finance-role holder, and only 4 companies have an HR-role holder. Until HR fills these in (Step 10), raising an SRF is still refused with "Nobody in the company can take step 1 (Budget Validation)…". MAKL's own rule stops at 200k, so a MAKL SRF above 200k has no rule.

### > Code correction and static values (follow.md #10)

| Where | Static value | Fix |
|---|---|---|
| `RequisitionService.GetFallbackStepDefinitions` | Hard-coded 3-step chain | ✅ Removed in Step 4; refuse (D1) |
| `ApprovalsService` ~950 | `"MANAGING_DIRECTOR"` / `"HR_REVIEW"` fallback, `totalSteps = 2` | Use the stored offer steps only |
| `ApprovalsService.ResolveOfferApproverAsync` | Job-title text match `"Director"/"Managing"/"Chief"/"CEO"`, global search | Use the matrix step (role / designated user) through `ApprovalEngine`, limited to the company |
| `RequisitionService` 510 / 519 | `"System User"`, `Region = "AFRICA_KE"` | Current user's name; region from the company's country master |
| `OnboardingService` 209 / 713 / 1126 | `"Healthcare Professional"` | Vacancy job title; empty if none |
| `OnboardingService` 1599–1605 | `FirstOrDefault()` defaults, `ProbationPeriodMonths = 3` | Values from the onboarding profile → offer → SRF |
| `OnboardingService` 1738–1768 | Silent fallbacks to any company's masters | Validation error naming the bad field |
| `GetDefaultRoleAssigneeName` | Display names like "Finance Controller" | ✅ Removed in Step 4; the step's `approverName` holds the resolved person |
| `RequisitionDetailDrawer` (SRF timeline) | Synthesised Budget → HR → Approval chain with `"superadmin"`, `"Finance Controller"` names when steps were missing | ✅ Step 4: shows the SRF's own steps and the assigned approver |
| `SrfTable` (SRF list) | Three fixed stage columns (Budget / HR / Final) | Step 9: show the SRF's own steps (rules may have any number) |
| `companies.offerApproval` default `["HR_MANAGER","MANAGING_DIRECTOR"]` (entity, Org/Config DTOs, `CompanyService`, `ConfigService`) | Legacy chain column; offers no longer read it (Step 3) | Step 9: stop defaulting it / drop it |
| `sp_ApproveOfferStep` (proc.sql) | Offer decision with no step/approver check | No longer called since Step 3; drop in Step 9 |
| `RecruitmentService.CreateOfferAsync` | First department / job title / location of the company, then `"dept-001"` / `"jt-001"` / `"loc-001"`, when the vacancy has none; letter body `"Formal Offer of Employment for …"` | Step 9: take them from the vacancy only, refuse otherwise |
| `ApprovalsService` /task history (offers) | `"Talent Acquisition"`, `"General Operations"`, `"OFF-N/A"` | Step 9: empty instead of invented labels |
| `TaskHubView` batch decide / `StaffingPresenceDrawer` | Made-up remarks "Batch … processed via Task Hub", "Evaluated staffing presence (…% coverage)"; server default "Bulk sign-off action executed" for any action | ✅ Step 5: approver's own remarks; the server default is kept for approvals only |
| `RaiseSrfDrawer` footer note | Invented chain "Line Manager/HOD → Finance → HR → Executive / CEO" | ✅ Step 5: "steps come from the company's approval matrix rule" |
| `ApprovalsService.GetMatrixRulesAsync` | Company label "Hospital Operating Entity" when the rule has no company | ✅ Step 5: "All companies" for an `ALL` rule, else empty |
| `RecruitmentService` dashboard "open positions" | `OpeningsCount + FilledCount` counted as total | ✅ Step 5: total = openings, open = openings − filled |
| `RecruitmentService.ConvertCandidateToEmployeeAsync` (direct hire) | Bank `"Standard Healthcare Bank"` / `"01128492019"` | Step 9: empty, HR fills in |
| `SrfFilterBar` | Region / type / status option lists typed in the component | Step 9: from masters / server |
| `OnboardingController.ActivateCandidate` | `"sys-user"` / `"HR Manager"` when the claims are missing | Step 9: refuse without a signed-in user |
| `RequisitionService` budget-validate | `BUDGET_EXCEEDED` marks the step REJECTED but leaves the SRF pending with no active step | Step 9: decide (reject, or return) |
| `ApprovalEngine.ResolveApproverForStepAsync` (leave/attendance) | Job-title / department-name text matches, global CEO/GROUP fallbacks | Not used by offers since Step 3 (they use the company-limited `ResolveMatrixStepAsync`); fix with F26 |

---

## 3. Target design

1. **One gate for every decision.** `/task decide` and every `/requisitions` decision endpoint go through `ApprovalVisibilityRules.IsActionableBy`, and only for the **current** pending step. The submitter or creator is never the approver (SRF: `CreatedById`; Offer: `CreatedById`). Superadmin/developer keep their override; the audit row records that it was an override.
2. **One approver resolver.** SRF and Offer steps both use `ApprovalEngine.ResolveApproverForStepAsync`:
   - The designated user on the matrix step comes first.
   - Then the role, always inside the submitter's company (Group HR is the only cross-company role).
   - Then self-approval escalation (D2).
   - The resolved approver is **saved on the step row** when the step becomes active, so /task, notifications and the tier counter all read the same value.
3. **The rule used is stored.** `job_requisitions.matrixRuleId` and `offers.matrixRuleId` are filled at submit. No re-matching afterwards. Ties are broken by `priority`, then `createdAt`.
4. **No rule → refuse (D1).** The message points HR to the /task Approval Matrix page.
5. **Return → requester edits → resubmit (D4).** Status `RETURNED` (new enum value, or `DRAFT` with `hrStatus=RETURNED_FOR_REVISION`; decided in Step 5 review). Remarks are mandatory on Reject and Return.
6. **Activation reads what was agreed.** Defaults come from the onboarding profile (set at initiate), then the offer, then the SRF/vacancy. Activate validates every id against **the target company only** and returns a field-level error instead of falling back.

---

## 4. Implementation steps (one approval / PR each)

### Step 1 — Security: permission checks (F1, F2)
- `OnboardingController`: add `[Authorize(Policy = …)]` per endpoint, using the permissions that already exist:
  - `onboarding.view`: metrics, pipeline, candidates/{id}, timeline, templates, configuration
  - `onboarding.initiate`: initiate-data, initiate, move-stage
  - `onboarding.review`: review/*, verify-document, document view/download
  - `onboarding.activate`: activation-init, activate
  - The 4 `public/form/{token}` endpoints stay `[AllowAnonymous]`.
- `TaskController`: `POST/DELETE matrix-rules` → `approval_matrix.manage`; `GET matrix-rules` → `approval_matrix.view`.
- Check `RecruitmentController` for the 2 endpoints without a policy (only the 2 public offer endpoints may be anonymous).
- DB: none expected (the permissions exist). If a policy is missing from `role_permissions` for a role that needs it, add it to `docs/db_changes.sql`.
- Tests: a new `Tests/RecruitmentSecurityTests.cs` checks that each endpoint has the right attribute (reflection, like the existing RBAC tests).
- Verify on the clone: employee 70061 gets 403 on all of the above; HR 70060 still works.

### Step 2 — SRF decisions only through the rules (F3, F4 for SRF)
- `RequisitionService.ProcessApprovalDecisionAsync`, `ValidateBudgetAsync`, `ReviewHrAsync`:
  - reject when `request.Step != requisition.CurrentStep`
  - reject when the caller is the requester
  - reject when `IsActionableBy` is false
- Per D3, move the frontend `/requisitions` decision buttons to a "Open in Tasks" link (`modules/requisition/`).
- Tests: out-of-order step → refused; requester → refused; correct approver → passes.

### Step 3 — Offer approval routing (F4, F5, F6, F8-offer, F18)
- Offer create: no OFFER rule → refuse (D1); store `matrixRuleId`; for every step, save `designatedUserId`.
- On the step becoming active, resolve and save `ApproverEmployeeId / ApproverUserId / ApproverName` on `offer_approvals` through `ApprovalEngine`, limited to the company.
- Remove the hard-coded HR → MD fallback and the global job-title search from `ResolveOfferApproverAsync` (or remove the method).
- The creator can't approve (F4); `SubmittedById` = offer `CreatedById`.
- Tier = the active step number.
- DB: `offers.matrixRuleId`, and approver columns on `offer_approvals` if missing → `docs/db_changes.sql`.
- Match the OFFER rule on department / grade / salary (the offers drawer saves them since Step 2), lowest priority first.
- Any offer decision endpoint outside `/task` goes through `IApprovalActionGate` + the active-step check, like the SRF ones in Step 2.
- Tests: designated user gets the step; FINANCE step goes to the company finance user; no cross-company approver; creator refused.

### Step 4 — SRF approval routing (F7, F8-SRF, F9)
- Reuse what Step 3 added: `ApprovalEngine.ResolveMatrixStepAsync` (designated employee → role inside the company → never the requester, escalate to their manager), `ApprovalVisibilityRules.RoleMayTakeStep`, and the rule pick in `OfferApprovalRouting.PickRule` (fit, then priority, then specificity, then oldest).
- Store `job_requisitions.matrixRuleId` at submit. `ResolveRequisitionApproverAsync` reads that rule's step, not "highest priority".
- DEPT_HEAD when the head is the requester → escalate per D2 and show the reason (`isSelfApprovalEscalated`).
- Remove `GetFallbackStepDefinitions` (D1). Make the tie-break deterministic.
- Save the resolved approver on `requisition_approvals` when the step becomes active (same as Step 3).
- DB: `job_requisitions.matrixRuleId` → `docs/db_changes.sql`.
- F25: `SaveMatrixRuleAsync` refuses a rule with no name, no steps, a step with no role/designated user, a designated user from another company, or min salary > max salary (400 with the field).
- Tests: head = requester escalates; designated user from the matched rule; no rule → refused; empty rule → 400.

### Step 5 — Reject / Return / Resubmit (F10, F11)
- `ApprovalEngine` REQUISITION `RETURNED` → returned status (not REJECTED), same as `/requisitions`.
- New `PUT /api/requisitions/{id}` (edit while returned) and `POST /api/requisitions/{id}/resubmit` (D4: steps reset, history kept).
  - Permission `requisitions.create` (owner only).
- Frontend: the SRF drawer opens in edit mode for a returned SRF with a "Resubmit" action (follow.md #7 SideDrawer).
- `/task decide`: remarks mandatory for REJECTED and RETURNED on REQUISITION and OFFER (server-side and in the UI).
- Tests: return → edit → resubmit restarts at step 1; empty remarks refused.

### Step 6 — Notifications (F12)
- Use the existing `INotificationService` and `NotificationTemplates`:
  - new pending step → the resolved approver
  - final approve / reject / return → the requester (SRF) or creator (Offer); the return notice carries the remarks and links to the SRF (Edit & Resubmit, Step 5)
  - resubmit → the step-1 approver of the new round
  - vacancy created → HR
- No fixed text in code beyond the templates; links go to `/task` or the SRF.
- Tests: one notification per event with the right user.

### Step 7 — Onboarding: declaration, activation defaults, validation (F13, F14, F15, F16, F23)
- Public form submit: refuse when `declarationConfirmed` is false (and when the template's required documents are missing, if the template has any).
- `initiate-data`: proposed reporting manager = the SRF requester or the department head, not the first manager.
- `activation-init`: fill each field from the onboarding profile → offer → SRF/vacancy. No `FirstOrDefault()` defaults; empty when unknown. Probation from the offer.
- `activate`:
  - Validate department, location, grade, job title, cost centre, shift and manager **within the target company**; return field errors (no silent fallback, no `IgnoreQueryFilters` cross-company search).
  - Store `nssfNumber`/`nhifNumber` as `""` when blank, or require them if the company's country statutory master marks them mandatory (follow.md #4).
- Tests: activation defaults equal the offer values; bad id → field error; missing NSSF → saved (or clear error), never a 500.

### Step 8 — After activation (F17)
- ~~Activation increments `vacancies.filledCount`; when `filledCount >= openingsCount`, the vacancy becomes FILLED/CLOSED~~ — done in Step 5 (`VacancyOpenings`).
- Create leave balances the way Add Employee does (D5). Reuse that service; no new rules.
- Tests: the new hire has leave balances.

### Step 9 — Small fixes, static values, cleanup (F19–F22, F24, §2 static values)
- Offer currency → company currency master when empty (same as SRF).
- `decidedByName` / `finalApprovedByName` → the employee's full name (look up by user id).
- Scorecard values must be 1–5 (D7) → 400 otherwise.
- Save `positionTitle` on the offer.
- Replace `"Healthcare Professional"`, `"System User"`, `"AFRICA_KE"` per the §2 static-values table.
- D6 merge per §0.2 (keep `modules/requisitions/`, delete `modules/requisition/`, retire `/masters/srf-matrices`). Its drawer look now lives in `approval-matrices/requisitions/RequisitionMatrixDrawer.tsx` (Step 1), so `requisition/srf/drawers/ApprovalMatrixDrawer.tsx` can be deleted with it.
- `MatrixRuleBuilderDrawer.tsx` default chain (Finance → HR → Dept Head) → start with one empty step, like the requisition drawer.
- D7: confirm the score scale (manual: 1–10).
- Step 3 leftovers (see the static-values table): `companies.offerApproval` default, drop `sp_ApproveOfferStep`, offer-create master fallbacks (`dept-001`…), /task offer history labels.

### Step 10 — MAKL set-up + live users (data through the UI/API, after your go-ahead)
- Set department heads for MAKL departments (Org Masters).
- Fix `rule-offer-makl`: retype it to OFFER, or delete it and add a proper OFFER rule on /approval-matrices/offers with real approvers.
- The other 13 companies' OFFER rules end in a `MANAGING_DIRECTOR` step nobody holds (§2, found in Step 3): with your go-ahead, point that step at a designated employee (or assign the role) so their offers can be submitted.
- Add MAKL onboarding templates (or copy Lifecare's).
- Create the 6 test users from §1 on the live DB (Add Employee + Admin › Users). Employee codes will differ from the clone.

### Step 11 — Full re-test (follow.md #12)
- Re-run §6 on a fresh clone with all 6 users. Update the §1 results and write completion notes per step below.

---

## 5. Open risks
- Changing `ResolveOfferApproverAsync` and `ResolveRequisitionApproverAsync` affects `/task` for every company. The Step 3/4 tests must cover `comp-lch-01` and `comp-afri-01` as well as MAKL.
- Existing PENDING SRFs (Step 4, done): the DB block moves the designated employee out of `note`; the approver is resolved from the step's own role/designee the first time /task lists it, then saved. `matrixRuleId` stays null for them (no guessing). Offers: the live DB had 0 pending offers at Step 3; any old pending offer still works through its step's role bucket in /task, and its next step is resolved and saved when it becomes active.

- F26: Leave / Attendance & Shifts / Payroll rules on `/approval-matrices/*` are not applied (see §2). The drawers say so; wiring them changes routing for every company and needs its own plan.

- Line Manager (`role_mgr`) holds all 5 `onboarding.*` permissions, including `onboarding.review` (shows candidate bank details) and `onboarding.activate`. Left unchanged in Step 1 (no decision given). Decide before Step 10; the fix is a `role_permissions` block in `docs/db_changes.sql`.

## 6. E2E re-test checklist (run on a clone after each step)
1. Kamau raises an SRF for IT → Finance (Mutua) → HR (Njeri) → IT head (Wanjiru) → vacancy created. Only the current approver sees it in /task.
2. Kamau raises an SRF for his own department → the Department Head step escalates (D2).
3. Finance rejects without remarks → refused; with remarks → REJECTED, Kamau notified.
4. HR returns → Kamau edits and resubmits → restarts at step 1.
5. Kamau calls `/requisitions/{id}/approval-decision` for step 3 → refused.
6. HR adds a candidate → interview → scorecard (1–5) → OL Creation → offer under the MAKL OFFER rule → the designated user gets the step → Finance → dispatched → candidate accepts.
7. HR initiates onboarding → candidate submits without the declaration → refused → submits → HR sends back → resubmit → HR approves.
8. Activation defaults match the offer → activate → employee, login and leave balances created; vacancy filled/closed.
9. Employee 70061: 403 on onboarding, matrix-rules and requisitions endpoints.
10. `dotnet build`, `dotnet test`, `tsc --noEmit` all clean.

---

## 7. Completion notes
_(added per step as each one is approved and delivered)_

### Step 1 — done 2026-10-04 (F1, F2 closed)
- **Onboarding:** each of the 17 staff endpoints in `OnboardingController` has a policy:
  - `onboarding.view`: metrics, pipeline, candidates, timeline, templates, configuration
  - `onboarding.initiate`: initiate-data, initiate, move-stage
  - `onboarding.review`: review, approve, send-back, verify-document, document view/download
  - `onboarding.activate`: activation-init, activate
  - Only the 4 `public/form/{token}` endpoints are anonymous.
- **Approval matrix: one gate for both screens.**
  - `TaskController` `GET matrix-rules` needs `approval_matrix.view`; `POST`/`DELETE` need `approval_matrix.manage`.
  - `RequisitionsController` `approval-matrices` GET/POST/DELETE moved from `requisitions.view/manage` to `approval_matrix.view/manage`. Before this, Line Managers held `requisitions.manage` and could edit rules from `/masters/srf-matrices`.
- **New `GET /api/task/matrix-roles`** (`approval_matrix.view`) lists active roles from the roles master with their `code`, which is what the engine matches on. Both matrix drawers use it instead of `/admin/roles`, which needs `roles.manage` and returned 403 for Company HR.
- **`RecruitmentController`:** already correct; only the 2 `offers/public/{token}` endpoints are anonymous. No change.
- **Requisition approval page, as you asked:** `/approval-matrices/requisitions` now opens `RequisitionMatrixDrawer`, which has the `/masters/srf-matrices` drawer layout:
  - a list of the company's rules, and an edit mode with numbered step cards, move up/down, and a live chain preview;
  - per step: title, **Assign to: Role** (organisation-structure codes plus roles master) **or Specific employee** (that company's employees), action, SLA days;
  - department and grade come from the org masters, currency from the company master;
  - a new rule starts with one empty step (no ready-made chain), and region is no longer offered (an existing value is kept on save);
  - Add/Edit/Delete are hidden without `approval_matrix.manage`, here and in the list on every matrix page.
  - The routing-role list moved to `components/matrixRoleOptions.ts` and is shared with `MatrixRuleBuilderDrawer`.
- **DB:** none. `superadmin`, `admin`, `company_hr`, `group_hr`, `role_hr` and `role_dev` already hold these permissions, and `hr_vp` holds the matrix ones.
- **Tests:**
  - `Tests/RecruitmentSecurityTests.cs` adds 27 reflection checks (the Tests project now references Api).
  - `dotnet build` 0 errors, `dotnet test` **181/181**, `tsc --noEmit` clean.
- **Clone check** (`HRMSCore_SrfTest`, API :5299, biometric sync off):

  | User | Onboarding (17) | Matrix rules + matrix-roles | Note |
  |---|---|---|---|
  | Employee 70061 | 403 | 403 | |
  | Line Manager 70057 | pass | 403 (all 6 rule calls) | |
  | Company HR 70060 | pass | pass | Saved a rule as the drawer sends it: currency filled from the company (KES), designated employee kept. Then deleted it. |
  | superadmin | pass | pass | |

  - "pass" means the request got past the permission check (200, or 400/404 for dummy ids).
  - Public onboarding/offer links with a bad token → 404 (anonymous, not 401).
  - The clone's `superadmin` password was set to the test password (clone only).
- **Not done:** a visual browser check of the drawer. Playwright isn't installed, and a second dev server would clash with yours on :4000. Open `/approval-matrices/requisitions` on your :4000 to see it.
- **Found:** F25 (empty rule accepted; scheduled in Step 4). Open item on `role_mgr` onboarding rights → §5.

### Step 2 — done 2026-10-05 (F3 closed, F4 closed for SRF)
- **One gate for /task and the SRF endpoints.**
  - New `IApprovalActionGate.EnsureCanActAsync`. `ApprovalsService` implements it with the check `/task decide` already used ("CanAct on this item"), so the same `IsActionableBy` rules apply everywhere, including "never your own submission".
  - Superadmin/developer pass, and the method returns `true` so the caller can log the override.
- **`RequisitionService.ValidateBudgetAsync` / `ReviewHrAsync` / `ProcessApprovalDecisionAsync`**, via `LoadForDecisionAsync` and the new `RequisitionDecisionRules`:
  - 400 unless the SRF is `PENDING_APPROVAL` and the step is the active one (the lowest PENDING step, equal to `currentStep`).
  - `approval-decision`: `step` must be that step, and the action must be APPROVED, REJECTED or RETURNED.
  - `budget-validate`: only while the active step is `FINANCE_BUDGET`; status must be BUDGET_AVAILABLE, BUDGET_EXCEEDED or REJECTED.
  - `hr-review`: only while the active step is `HR_REVIEW`; status must be APPROVED, REJECTED or RETURNED_FOR_REVISION.
  - 403 when the gate refuses. The controller now returns 400/403/404 instead of 500.
  - The decider's name is the caller's name. The `"Finance Controller"`, `"HR Reviewer"` and `"Authorized Approver"` fallbacks are gone.
- **Override audit.** A superadmin/developer decision on an SRF writes `audit_log.action = REQUISITION_DECISION_OVERRIDE` with the step and the source (`/task`, `approval-decision`, `budget-validate` or `hr-review`).
- **Endpoint permission fixed.**
  - `budget-validate` and `hr-review` needed `requisitions.manage`. On MAKL only Line Managers (the requesters) hold it; Finance and Company HR (the approvers) don't.
  - All three endpoints now need `approvals.manage`, and the gate decides which step a person may act on.
- `ApprovalsService.DecideApprovalAsync` no longer falls back to `"usr-001"` / `"Administrator"`; there is no decision without a signed-in user.
- **Frontend (D3).**
  - No approve buttons were left on `/requisitions`.
  - Removed the unused `BudgetValidationDrawer` and the three decision calls from `modules/requisition/api.ts`.
  - Pending rows in the SRF table now show **Open in Tasks** (`/task?module=REQUISITION`). The detail drawer already linked there.
  - The old `modules/requisitions/` copy still has the calls; it is deleted in Step 9.
- **Matrix drawers, as you asked:** the Requisition drawer layout is now on all 5 pages.
  - `components/MatrixRulesDrawer.tsx`, configured per page in `components/matrixDrawerConfigs.ts`:

    | Page | Criteria |
    |---|---|
    | Requisitions | type, department, grade, salary |
    | Job Offers | department, grade, salary |
    | Leave | department, grade, min/max days; first step preset to Reporting manager |
    | Attendance & Shifts | rule type (Attendance regularisation / Shift swap), department, grade; first step preset to Reporting manager |
    | Payroll | whole company |

  - Leave, Attendance and Payroll show an amber note: the rules are not applied yet (F26).
  - The `/approval-matrices` overview opens the right drawer for the rule being edited; new rules are added from each type's page.
  - The old 1,209-line `MatrixRuleBuilderDrawer` is deleted.
- **DB:** none.
- **Tests:**
  - `Tests/RequisitionDecisionGateTests.cs` (7): wrong step, gate refusal, wrong kind of step, finished SRF / unknown action, correct approver, superadmin override + audit, requester never actionable.
  - `RecruitmentSecurityTests` +3: the three endpoints need `approvals.manage`.
  - `dotnet build` 0 errors, `dotnet test` **191/191**, `tsc --noEmit` clean.
- **Clone check** (`HRMSCore_SrfTest`, API :5299, biometric sync and SMTP off). Kamau raised SRF-2026-13256 for IT:

  | # | Call | Result |
  |---|---|---|
  | 1 | Kamau `approval-decision` step 3 | 400 "waiting on step 1 (Budget Validation)" |
  | 2 | Kamau step 1 | 403 |
  | 3 | Njeri `hr-review` while on Finance | 400 |
  | 4 | Njeri `budget-validate` | 403 |
  | 5 | Employee 70061 | 403 |
  | 6 | Mutua `budget-validate` | 200 |
  | 7 | Mutua again | 400 (now on HR) |
  | 8 | Unknown action | 400 |
  | 9 | Njeri `hr-review` | 200 |
  | 10 | Kamau step 3 on his own SRF | 403 |
  | 11 | superadmin step 3 | 200, vacancy created, 1 override audit row |
  | 12 | Wanjiru afterwards | 400 "is APPROVED" |

  - /task path (SRF-2026-EAC44): at each step only the current approver had `canAct` (Mutua → Njeri → Wanjiru). Each approved in /task; the SRF reached APPROVED and the vacancy was created.
- **Not done:** a visual browser check of the 4 new drawers. Open `/approval-matrices/leave`, `/attendance`, `/offers` and `/payroll` on your :4000.
- **Found:** F26 (Leave / Attendance / Payroll rules not applied); see §2 and §5.

### Step 3 — done 2026-10-05 (F4, F5, F6, F8-offer, F18 closed)
- **Offers follow their OFFER rule only (D1).**
  - New `OfferApprovalRouting.PickRule`: active OFFER rules of the offer's company. Department, grade and salary must fit; an empty criterion fits anything. Then lowest priority, then the more specific rule, then the oldest.
  - No fitting rule → 400 "No offer approval matrix set up for <company> that fits this offer — ask HR to add one on Approval Matrices › Job Offers". A rule with no steps → 400.
  - `offers.matrixRuleId` is saved; the steps come only from that rule. The HR → Managing Director chain is gone from `RecruitmentService` and from /task.
  - The offer now gets its grade from the vacancy (or its SRF), so grade rules can match. Before, offers had no grade.
- **One company-limited step resolver** (`ApprovalEngine.ResolveMatrixStepAsync`; Step 4 reuses it for the SRF):
  1. the designated employee (must be active in the offer's company), else
  2. `DEPT_HEAD` = the head of the offer's department, `LINE_MANAGER` = the creator's manager, any other code = role holders in the company (`GROUP_HR`/`HRBP` across the group), first by employee number.
  - Never the creator: if only the creator holds the step, their reporting manager gets it (`escalationReason` saved, shown in /task as auto-escalated); else the offer is refused naming the step.
  - No job-title or department-name guessing, no global "first employee". `ResolveOfferApproverAsync` is deleted.
  - Who is *assigned* and who *may act* use the same `ApprovalVisibilityRules.RoleMayTakeStep`, so they can't drift apart. Any holder of the step's role in the company can act (you confirmed); a designated-employee step only that person (plus god mode).
- **Saved on each step** (`offer_approvals`): `approverEmployeeId`, `approverUserId`, `approverName`, `escalationReason`, `designatedEmployeeId`. All steps are resolved at submit (a gap fails early) and the next step is resolved again when it becomes active.
- **F4:** `offers.createdById` + the creator's real name (was `"Talent Team"`). /task now sets `SubmittedById` = creator, so "never your own submission" applies.
- **F18:** tier = the active step's number, total = the step count (`CurrentStep` starts at the first step, not 0).
- **Endpoints outside /task:**
  - `PATCH /recruitment/offers/{id}/approve-step`: `approvals.manage` (was `recruitment.manage`), active step only (400), `IApprovalActionGate` (403), then the same engine decision as /task. `sp_ApproveOfferStep` is no longer called.
  - `PATCH /recruitment/offers/{id}/status`: never `APPROVED`; a pending offer can only be withdrawn; `PENDING_APPROVAL` only from DRAFT (runs the same submit); `SENT`/`ACCEPTED` only once approved.
  - Superadmin/developer decisions write `audit_log` `OFFER_DECISION_OVERRIDE` (from /task and approve-step).
  - Create / status / approve-step return 400/403/404 with the reason instead of 500.
- **Frontend (D3):** the Approve/Reject buttons in the offer drawer (`ViewOfferLetterDrawer`) are now **Open in Tasks**; it shows the assigned approver. `approveOfferStep` removed from `recruitment/api.ts`. The Create Offer drawer shows the server's reason (e.g. no matrix).
- **DB:** `docs/db_changes.sql` block "2026-10-05 — SRF/ATS Step 3" (7 nullable columns, idempotent; run twice on the clone). Note: the working copy of `docs/db_changes.sql` had been cut down to its header before this step; the block is appended after it.
- **Tests:** `Tests/OfferApprovalRoutingTests.cs` (25): rule pick ×3, status changes ×11, designated user, finance role in the company only (not the other company's holder), designated outsider refused, creator escalated/refused, creator can't act, no rule refused, rule followed + approvers saved, unresolvable step refused, wrong step 400, gate 403, tier moves, override audit. `RecruitmentSecurityTests` +1 (approve-step → `approvals.manage`). `dotnet build` 0 errors, `dotnet test` **217/217**, `tsc --noEmit` clean.
- **Clone check** (`HRMSCore_SrfTest`, API :5299, biometric sync off, SMTP pointed at a dead port). Clone rule "MAKL Offer Approval (test)" set to COMPANY_HR → designated Wanjiru → FINANCE. Njeri (70060) created OFF-2026-0103:

  | # | Call | Result |
  |---|---|---|
  | 1 | Create | steps: Caroline Nduta (the other Company HR holder, not Njeri) → Wanjiru → Mutua, all MAKL |
  | 2 | /task step 1 | only Caroline can act, tier 1/3; Njeri, Wanjiru, Mutua, 70061 don't see it |
  | 3 | Njeri approve-step / /task decide | 403 "not assigned to you" |
  | 4 | Wanjiru approve-step 2 while on 1 | 400 |
  | 5 | Employee 70061 approve-step | 403 |
  | 6 | Njeri status → APPROVED / SENT | 400 "decide it in Tasks or withdraw it" |
  | 7 | Caroline /task approve | 200 → Wanjiru, tier 2/3 |
  | 8 | Wanjiru approve-step 2 | 200 → Mutua, tier 3/3 |
  | 9 | Mutua /task approve | 200 → offer APPROVED and dispatched (mail fell back to the local mailbox; file removed) |
  | 10 | Rule switched off → create | 400 "No offer approval matrix set up for Medical Administrators Kenya Ltd…" (rule switched back on) |
  | 11 | superadmin approve-step 1 and /task step 2 on OFF-2026-0104 | 200, 2 `OFFER_DECISION_OVERRIDE` rows |
- **Found:** the live OFFER rules can't be used as they are (MD step with no role holder) — see §2 set-up gaps; fix in Step 10 with your go-ahead.
- **Not done:** a browser check of the offer drawer's "Open in Tasks" link and the Create Offer error message on your :4000.

### Step 4 — done 2026-10-05 (F7, F8-SRF, F9, F25 closed)
- **SRFs follow their REQUISITION rule only (D1).**
  - New `RequisitionApprovalRouting.PickRule`: department, grade, salary budget (max), requisition type and region must fit (empty or `ALL` fits anything). **A fitting rule of the SRF's own company always wins; an `ALL`-company rule is used only when the company has none** (your answer). Within a group the shared `ApprovalMatrixRuleChecks.Best` decides: lowest priority, then more specific, then oldest (also used by offers now, so both pick the same way). Two equal-priority rules no longer depend on row order (F9).
  - No fitting rule → 400 "No requisition approval matrix set up for <company> that fits this SRF — ask HR to add one on Approval Matrices › Requisitions". A rule with no usable steps → 400.
  - `job_requisitions.matrixRuleId` is saved; the steps are copied from that rule at submit and never re-matched. The designated employee is read from the step itself (`designatedEmployeeId`), not from "the highest-priority rule" (F9).
  - Removed: `GetFallbackStepDefinitions` (Finance → HR → Dept Head), `MatchApprovalMatrixRuleAsync` (fell back to any rule), `GetDefaultRoleAssigneeName`, and `ApprovalsService.ResolveRequisitionApproverAsync` (job-title "Director/CEO" and "Finance"/"Human Resources" department-name guessing).
- **Approver resolution** (`ApprovalEngine.AssignRequisitionStepAsync`, built on Step 3's company-limited `ResolveMatrixStepAsync`): designated employee → `DEPT_HEAD` = head of the SRF's department, `LINE_MANAGER` = requester's manager, other codes = role holders in the company (`RoleMayTakeStep`).
  - **D2, every SRF step** (your choice): the requester never approves. If only the requester holds the step → their reporting manager; no manager → the company's `COMPANY_HR` holder; neither → the SRF is refused naming the step. The reason is saved (`escalationReason`) and /task shows the item as auto-escalated (F7).
  - All steps are resolved at submit (a gap fails early, naming the step); the next step is resolved again when it becomes active (/task decide and the old `budget-validate` / `hr-review` / `approval-decision` paths).
  - Saved on `requisition_approvals`: `approverEmployeeId`, `approverUserId`, `approverName`, `escalationReason`, `designatedEmployeeId`. `DecidedByName` stays empty until someone decides.
- **/task (REQUISITION):** assigned person, tier (= active step) and total (= step count) come from the saved step. Removed the labels `"CORP-REQ"`, `"General Operations"`, `"Department Governance"` and the "3 steps" default. **Old pending SRFs:** a step with no saved approver is resolved once from its own role/designee when /task lists it, then saved.
- **F25:** `ApprovalMatrixRuleChecks.ValidateAsync` runs on both save paths (`POST /task/matrix-rules`, `POST /requisitions/approval-matrices`) for all 5 matrix types. 400 for: no name, no steps, a step with neither role nor designated employee, a designated employee not active in the rule's company, minimum salary above maximum.
- **Also fixed:** an SRF with no grade gave a raw 500 (`job_requisitions.gradeId` is NOT NULL) → 400 "Choose a grade for this SRF."; a blank cost centre is saved as null.
- **Frontend:** the SRF detail drawer shows the SRF's own steps with the assigned approver and "auto-escalated: <reason>" (the made-up Budget → HR → Approval chain with "superadmin" names is gone). The rule drawers and the Raise SRF drawer show the server's reason (`errors[0]`).
- **DB:** `docs/db_changes.sql` block "2026-10-05 — SRF/ATS Step 4" (6 nullable columns + moving `ASSIGNED_EMP:` notes on pending steps into `designatedEmployeeId`). Ran twice on the clone.
- **Tests:** `Tests/RequisitionApprovalRoutingTests.cs` (17): rule pick ×3 (own company beats ALL, ALL fallback, type/region/priority/specificity/age), no rule refused, matched rule + designee saved (not the other rule's), head = requester → manager, → Company HR, → refused, role held only in another company → refused, next step resolved on advance, old pending step resolved, F25 ×5 + a complete rule saves. Shared `Tests/TestStubs.cs` (email stub). `dotnet build` 0 errors, `dotnet test` **234/234**, `tsc --noEmit` clean.
- **Clone check** (`HRMSCore_SrfTest`, API :5299, biometric sync off, SMTP to a dead port; your :5197 untouched):

  | # | Call | Result |
  |---|---|---|
  | 1 | /task as each user, old pending SRF-2026-E95A0 (no saved approver) | only Mutua sees it, tier 1/3; approver saved on first listing |
  | 2 | Kamau raises SRF-2026-1E523 for IT | rule `rule-lch-req-std` (the older of the two priority-100 rules) saved; Mutua → Njeri → Wanjiru |
  | 3 | /task visibility for SRF-2026-1E523 | only Mutua; Kamau decide → 403; Wanjiru `approval-decision` step 3 early → 400 |
  | 4 | Mutua, Njeri, Wanjiru approve in /task | step 2 → step 3 → APPROVED, vacancy created |
  | 5 | Kamau raises SRF-2026-1F838 for his own Contact Centre | Dept Head step → Njeri (Company HR), "no reporting manager; auto-escalated to COMPANY_HR" |
  | 6 | Budget 300k (falls to the mistyped `rule-offer-makl`) | 400 "Nobody in the company can take step 2 (HR Review) of 'Standard Employment Offer…'" |
  | 7 | Both MAKL rules switched off | 400 "No requisition approval matrix set up for Medical Administrators Kenya Ltd…" (switched back on) |
  | 8 | F25: 5 bad rules on both endpoints | 10 × 400 with the field; rule count unchanged (35) |
  | 9 | Employee 70061 decides | 403 |
  | 10 | SRF without grade | 400 "Choose a grade for this SRF." |
- **Found:** only 2 of 15 live companies have a REQUISITION rule — see §2 set-up gaps (Step 10).
- **Not done:** a browser check of the SRF drawer timeline and the drawers' error text on your :4000.

### Step 5 — done 2026-10-05 (F10, F11 closed; F17 vacancy part closed)
- **Return for revision (F10).** /task Return, `hr-review` RETURNED_FOR_REVISION and `approval-decision` RETURNED all set the new status **`RETURNED`** (was REJECTED from /task, DRAFT from the old endpoints). The steps after the returning one become `SUPERSEDED`, so nothing is left pending. A return at the budget step leaves `budgetStatus` = PENDING_VALIDATION.
- **Edit + resubmit (D4).**
  - `PUT /api/requisitions/{id}` edits a returned SRF; `POST /api/requisitions/{id}/resubmit` starts it again. Both use policy `requisitions.create` and only the requester; superadmin/developer always, with a `REQUISITION_DECISION_OVERRIDE` audit row. Anyone else → 403; an SRF that is not RETURNED → 400.
  - Resubmit picks the rule again (`RequisitionApprovalRouting.PickRule`), builds the steps from step 1 and resolves each one (`AssignRequisitionStepAsync`), as raise does. A gap or no fitting rule → 400 and the SRF stays RETURNED.
  - New column `requisition_approvals.round`. Earlier rounds stay as history. The final-step check, the /task tier total and the SRF table only count the latest round.
  - Raise and edit share one field mapping (`ApplyRequestAsync`) and one step builder (`BuildApprovalStepsAsync`).
- **Remarks mandatory (F11)** for REJECTED and RETURNED on REQUISITION and OFFER, checked in `ApprovalEngine` before anything changes. The old `budget-validate` / `hr-review` / `approval-decision` endpoints check them too.
  - **Offers can't be returned** ("Offers can only be approved or rejected"); /task hides Return for offers.
  - /task batch Reject/Return asks for remarks in the batch bar. The made-up batch and Staffing Presence remarks are gone; the server's "Bulk sign-off" default is kept for approvals only.
- **Your decisions (all yes):** SRF create now needs `requisitions.create` (was `requisitions.manage`, so COMPANY_HR and EMS could not raise one); Return refused on offers; old SRFs wrongly REJECTED by the old Return are left alone.
- **Vacancy openings (your request, F17 vacancy part).** New `VacancyOpenings`:
  - onboarding activation refuses a hire when the vacancy is cancelled or all its openings are filled, otherwise `filledCount + 1`;
  - the vacancy stays OPEN until the last opening is filled, then CLOSED;
  - an opening is used only at activation, so a candidate rejected or declining at any earlier step never uses one, and the vacancy stays open.
  - The older direct-hire path uses the same helper. The recruitment dashboard's "open positions" = openings − filled (was openings + filled). The vacancy picker in Add Candidate shows "N of M openings left".
- **SRF rules for every company (your request).** An `ALL`-company rule can't be stored (`approval_matrices.companyId` has a foreign key to `companies`), so, per your choice, each company with no REQUISITION rule got "Standard SRF approval": Finance → HR → Department Head, by role, the company's own currency, priority 100. Added on live and on the clone (13 each). See §2 "Found in Step 5": the role holders are still missing on live.
- **Live 500 on `/requisitions/srf` fixed.** The Step 3 and Step 4 blocks of `docs/db_changes.sql` had never been run on `HRMSCore_Local`. Run on 2026-10-05 with your go-ahead, together with the Step 5 block (columns only added, 0 rows changed apart from the 13 new rules).
- **Frontend.**
  - The SRF detail drawer shows a "Returned for revision" banner with who returned it and their remarks, plus **Edit & Resubmit** for the requester and superadmin/developer. Earlier rounds show under "Earlier submissions".
  - `RaiseSrfDrawer` has an edit mode, filled from the SRF with the company locked, and two actions: "Save changes" and "Save & Resubmit". Regional details are kept unless re-entered.
  - The SRF list shows a "Returned for revision" badge and the status filter has "Returned for Revision".
- **DB:** `docs/db_changes.sql` block "2026-10-05 — SRF/ATS Step 5" adds the `round` column and the standard rules. Ran twice on the clone: the second run changed 0 rows.
- **Tests:** `Tests/RequisitionReturnResubmitTests.cs` (13):
  - return → RETURNED + SUPERSEDED; empty remarks refused ×2; hr-review return; offer Return / empty reject refused ×2;
  - only the owner edits, and a superadmin edit is audited; a pending SRF can't be edited or resubmitted; resubmit = round 2 at step 1 with history kept, ending in a vacancy; budget change → other rule; no rule → refused and still RETURNED;
  - vacancy fills to closed then refuses; cancelled vacancy refused.
  - `dotnet build` 0 errors, `dotnet test` **247/247**, `tsc --noEmit` clean.
- **Clone check** (`HRMSCore_SrfTest`, API :5299, biometric sync off, SMTP to a dead port; your :5197 untouched). SRF-2026-7ADD5 (Kamau, IT, rule `rule-lch-req-std`):

  | # | Call | Result |
  |---|---|---|
  | 1 | Mutua rejects with `""` / batch-rejects with no remarks | 400 "Remarks are required…" / 0 of 1 processed |
  | 2 | Mutua approves; Njeri returns with `"  "` then with remarks | 400, then RETURNED; steps APPROVED / RETURNED / SUPERSEDED |
  | 3 | Wanjiru PUT / resubmit; employee 70061 PUT | 403 / 403 / 403 |
  | 4 | Kamau edits (budget 90k, skills) then resubmits | RETURNED with the new values → PENDING_APPROVAL, step 1, round 2: Mutua → Njeri → Wanjiru; 6 step rows (round 1 kept) |
  | 5 | Resubmit again while pending | 400 |
  | 6 | /task | only Mutua sees round 2, total 3 |
  | 7 | Round 2 approved by Mutua, Njeri, Wanjiru | APPROVED, VAC-2026-67EE6 |
  | 8 | Superadmin PUT on the approved SRF | 400 "only an SRF returned for revision can be edited" |
  | 9 | SRF-2026-2806A returned by Mutua, resubmitted by superadmin, then rejected by Mutua with remarks | 200 + audit row "resubmitted by superadmin/developer override"; REJECTED |
- **Not done:** a browser check of the new drawer buttons on your :4000. Vacancy openings were checked by unit tests, not by a clone activation. **Restart your API on :5197** to load the Step 5 code.

---

## 8. Step prompts (copy-paste one into a new chat)

Each prompt is self-contained. Start a fresh chat per step and paste the prompt. At the end of a step the assistant gives back the next prompt (and updates it here if anything changed).

### Prompt — Step 1: Security: permission checks on Onboarding + Approval-matrix endpoints

```text
SRF / ATS / Onboarding fixes — Step 1 of 11: Security: permission checks on Onboarding + Approval-matrix endpoints.
Findings covered: F1, F2.
Goal: Add [Authorize(Policy=...)] to every OnboardingController endpoint (keep the 4 public/form/{token} endpoints anonymous), protect POST/DELETE /api/task/matrix-rules with approval_matrix.manage and GET with approval_matrix.view, and check the 2 RecruitmentController endpoints without a policy. Superadmin/developer always pass.
Read first: docs/follow.md (rules), docs/ats_fixes.md (§0 decisions + §0.1 standing rules, §2 findings, §4 Step 1, §6 checklist), docs/HRMS User Manual.pdf §6–§8 and §19 if needed, and memory (ats-fixes-plan, e2e-tests-protect-user-data, approvals-live-in-task-page, no-static-values-use-masters, keep-usage-lean).
Rules: superadmin/developer can do everything; all approvals live in /task; no static values (read masters); keep tool output small.
Do:
1. Present the plan for Step 1 (follow.md #9, with the "> Code correction and static values" section) and wait for my approval.
2. Implement; DB changes as a dated idempotent block in docs/db_changes.sql; new permissions/menus registered (follow.md #8).
3. dotnet build, dotnet test and tsc --noEmit must be clean; add tests for this step.
4. Verify on a throwaway DB clone (never the live DB, never my API on :5197), using the MAKL test users from docs/ats_fixes.md §1 (they exist in clone HRMSCore_SrfTest; on a fresh clone recreate them as described there).
5. Update docs/ats_fixes.md (progress table + completion notes in §7) and the artifact srf-ats-ob: read https://claude.ai/artifact/7yKeUEJ5A9aqNUobFx9Hzm, edit docs/artifacts/srf-ats-ob.html (progress, findings, How to operate, flow diagram, counts), publish to that url.
6. Give me the copy-paste prompt for Step 2 from docs/ats_fixes.md §8 (update it first if this step changed anything).
```

### Prompt — Step 2: SRF decisions only through the rules

```text
SRF / ATS / Onboarding fixes — Step 2 of 11: SRF decisions only through the rules.
Findings covered: F3, F4 (SRF).
Goal: Make /requisitions/{id}/approval-decision, budget-validate and hr-review accept only the current step, never the requester, and only someone IsActionableBy allows (superadmin/developer override, logged). Per D3, replace the approve buttons on /requisitions with an 'Open in Tasks' link.
Read first: docs/follow.md (rules), docs/ats_fixes.md (§0 decisions + §0.1 standing rules, §2 findings, §4 Step 2, §6 checklist), docs/HRMS User Manual.pdf §6–§8 and §19 if needed, and memory (ats-fixes-plan, e2e-tests-protect-user-data, approvals-live-in-task-page, no-static-values-use-masters, keep-usage-lean).
Rules: superadmin/developer can do everything; all approvals live in /task; no static values (read masters); keep tool output small.
Do:
1. Present the plan for Step 2 (follow.md #9, with the "> Code correction and static values" section) and wait for my approval.
2. Implement; DB changes as a dated idempotent block in docs/db_changes.sql; new permissions/menus registered (follow.md #8).
3. dotnet build, dotnet test and tsc --noEmit must be clean; add tests for this step.
4. Verify on a throwaway DB clone (never the live DB, never my API on :5197), using the MAKL test users from docs/ats_fixes.md §1 (they exist in clone HRMSCore_SrfTest; on a fresh clone recreate them as described there).
5. Update docs/ats_fixes.md (progress table + completion notes in §7) and the artifact srf-ats-ob: read https://claude.ai/artifact/7yKeUEJ5A9aqNUobFx9Hzm, edit docs/artifacts/srf-ats-ob.html (progress, findings, How to operate, flow diagram, counts), publish to that url.
6. Give me the copy-paste prompt for Step 3 from docs/ats_fixes.md §8 (update it first if this step changed anything).
```

### Prompt — Step 3: Offer approval routing

```text
SRF / ATS / Onboarding fixes — Step 3 of 11: Offer approval routing.
Findings covered: F4, F5, F6, F8 (offer), F18.
Goal: Offer create refuses when the company has no OFFER rule (D1, set on /approval-matrices/offers). Match the rule on department/grade/salary (saved by the offers drawer), store matrixRuleId; resolve each step through ApprovalEngine (named employee first, then role, inside the company only); save the resolved approver on offer_approvals; block the creator; fix the tier counter; remove the hard-coded HR -> MD chain and the global job-title search. Any offer decision endpoint outside /task uses the Step 2 gate (IApprovalActionGate + active-step check).
Read first: docs/follow.md (rules), docs/ats_fixes.md (§0 decisions + §0.1 standing rules, §2 findings, §4 Step 3, §6 checklist), docs/HRMS User Manual.pdf §6–§8 and §19 if needed, and memory (ats-fixes-plan, e2e-tests-protect-user-data, approvals-live-in-task-page, no-static-values-use-masters, keep-usage-lean).
Rules: superadmin/developer can do everything; all approvals live in /task; no static values (read masters); keep tool output small.
Do:
1. Present the plan for Step 3 (follow.md #9, with the "> Code correction and static values" section) and wait for my approval.
2. Implement; DB changes as a dated idempotent block in docs/db_changes.sql; new permissions/menus registered (follow.md #8).
3. dotnet build, dotnet test and tsc --noEmit must be clean; add tests for this step.
4. Verify on a throwaway DB clone (never the live DB, never my API on :5197), using the MAKL test users from docs/ats_fixes.md §1 (they exist in clone HRMSCore_SrfTest; on a fresh clone recreate them as described there).
5. Update docs/ats_fixes.md (progress table + completion notes in §7) and the artifact srf-ats-ob: read https://claude.ai/artifact/7yKeUEJ5A9aqNUobFx9Hzm, edit docs/artifacts/srf-ats-ob.html (progress, findings, How to operate, flow diagram, counts), publish to that url.
6. Give me the copy-paste prompt for Step 4 from docs/ats_fixes.md §8 (update it first if this step changed anything).
```

### Prompt — Step 4: SRF approval routing

```text
SRF / ATS / Onboarding fixes — Step 4 of 11: SRF approval routing.
Findings covered: F7, F8 (SRF), F9, F25.
Goal: Reuse the Step 3 pieces (ApprovalEngine.ResolveMatrixStepAsync, ApprovalVisibilityRules.RoleMayTakeStep, the OfferApprovalRouting.PickRule ordering). Store job_requisitions.matrixRuleId at submit and read named approvers from that rule; DEPT_HEAD when the head is the requester escalates per D2; remove GetFallbackStepDefinitions (D1, rules on /approval-matrices/requisitions); deterministic tie-break; save the resolved approver on requisition_approvals; handle existing pending SRFs with no stored rule. F25: refuse saving a matrix rule with no name, no steps, a step with no approver, a designated employee from another company, or min salary > max salary.
Read first: docs/follow.md (rules), docs/ats_fixes.md (§0 decisions + §0.1 standing rules, §2 findings, §4 Step 4, §6 checklist), docs/HRMS User Manual.pdf §6–§8 and §19 if needed, and memory (ats-fixes-plan, e2e-tests-protect-user-data, approvals-live-in-task-page, no-static-values-use-masters, keep-usage-lean).
Rules: superadmin/developer can do everything; all approvals live in /task; no static values (read masters); keep tool output small.
Do:
1. Present the plan for Step 4 (follow.md #9, with the "> Code correction and static values" section) and wait for my approval.
2. Implement; DB changes as a dated idempotent block in docs/db_changes.sql; new permissions/menus registered (follow.md #8).
3. dotnet build, dotnet test and tsc --noEmit must be clean; add tests for this step.
4. Verify on a throwaway DB clone (never the live DB, never my API on :5197), using the MAKL test users from docs/ats_fixes.md §1 (they exist in clone HRMSCore_SrfTest; on a fresh clone recreate them as described there).
5. Update docs/ats_fixes.md (progress table + completion notes in §7) and the artifact srf-ats-ob: read https://claude.ai/artifact/7yKeUEJ5A9aqNUobFx9Hzm, edit docs/artifacts/srf-ats-ob.html (progress, findings, How to operate, flow diagram, counts), publish to that url.
6. Give me the copy-paste prompt for Step 5 from docs/ats_fixes.md §8 (update it first if this step changed anything).
```

### Prompt — Step 5: Reject / Return for revision / Resubmit

```text
SRF / ATS / Onboarding fixes — Step 5 of 11: Reject / Return for revision / Resubmit.
Findings covered: F10, F11.
Goal: RETURNED keeps the SRF editable (not REJECTED); add PUT /api/requisitions/{id} and POST /api/requisitions/{id}/resubmit (owner only, restarts at step 1 per D4, history kept; resubmit re-picks the rule with RequisitionApprovalRouting.PickRule if the edit changed department/grade/salary/type, and re-resolves each step through ApprovalEngine.AssignRequisitionStepAsync, as Step 4 does at submit); SRF drawer edit + Resubmit; remarks mandatory for REJECTED and RETURNED on REQUISITION and OFFER in /task (server and UI).
Read first: docs/follow.md (rules), docs/ats_fixes.md (§0 decisions + §0.1 standing rules, §2 findings, §4 Step 5, §6 checklist), docs/HRMS User Manual.pdf §6–§8 and §19 if needed, and memory (ats-fixes-plan, e2e-tests-protect-user-data, approvals-live-in-task-page, no-static-values-use-masters, keep-usage-lean).
Rules: superadmin/developer can do everything; all approvals live in /task; no static values (read masters); keep tool output small.
Do:
1. Present the plan for Step 5 (follow.md #9, with the "> Code correction and static values" section) and wait for my approval.
2. Implement; DB changes as a dated idempotent block in docs/db_changes.sql; new permissions/menus registered (follow.md #8).
3. dotnet build, dotnet test and tsc --noEmit must be clean; add tests for this step.
4. Verify on a throwaway DB clone (never the live DB, never my API on :5197), using the MAKL test users from docs/ats_fixes.md §1 (they exist in clone HRMSCore_SrfTest; on a fresh clone recreate them as described there).
5. Update docs/ats_fixes.md (progress table + completion notes in §7) and the artifact srf-ats-ob: read https://claude.ai/artifact/7yKeUEJ5A9aqNUobFx9Hzm, edit docs/artifacts/srf-ats-ob.html (progress, findings, How to operate, flow diagram, counts), publish to that url.
6. Give me the copy-paste prompt for Step 6 from docs/ats_fixes.md §8 (update it first if this step changed anything).
```

### Prompt — Step 6: Notifications for SRF and Offer steps

```text
SRF / ATS / Onboarding fixes — Step 6 of 11: Notifications for SRF and Offer steps.
Findings covered: F12.
Goal: Use INotificationService + NotificationTemplates: notify the resolved approver when a step becomes pending (including step 1 of a resubmitted SRF's new round), and the requester/creator on final approve, reject or return (the return notice carries the approver's remarks and links to the SRF, where the requester uses Edit & Resubmit); notify HR when a vacancy is created and when its last opening is filled. Template text only, links to /task or the SRF.
Read first: docs/follow.md (rules), docs/ats_fixes.md (§0 decisions + §0.1 standing rules, §2 findings, §4 Step 6, §6 checklist), docs/HRMS User Manual.pdf §6–§8 and §19 if needed, and memory (ats-fixes-plan, e2e-tests-protect-user-data, approvals-live-in-task-page, no-static-values-use-masters, keep-usage-lean).
Rules: superadmin/developer can do everything; all approvals live in /task; no static values (read masters); keep tool output small.
Do:
1. Present the plan for Step 6 (follow.md #9, with the "> Code correction and static values" section) and wait for my approval.
2. Implement; DB changes as a dated idempotent block in docs/db_changes.sql; new permissions/menus registered (follow.md #8).
3. dotnet build, dotnet test and tsc --noEmit must be clean; add tests for this step.
4. Verify on a throwaway DB clone (never the live DB, never my API on :5197), using the MAKL test users from docs/ats_fixes.md §1 (they exist in clone HRMSCore_SrfTest; on a fresh clone recreate them as described there).
5. Update docs/ats_fixes.md (progress table + completion notes in §7) and the artifact srf-ats-ob: read https://claude.ai/artifact/7yKeUEJ5A9aqNUobFx9Hzm, edit docs/artifacts/srf-ats-ob.html (progress, findings, How to operate, flow diagram, counts), publish to that url.
6. Give me the copy-paste prompt for Step 7 from docs/ats_fixes.md §8 (update it first if this step changed anything).
```

### Prompt — Step 7: Onboarding: declaration, activation defaults, validation

```text
SRF / ATS / Onboarding fixes — Step 7 of 11: Onboarding: declaration, activation defaults, validation.
Findings covered: F13, F14, F15, F16, F23.
Goal: Refuse form submit without the declaration (and missing required template documents); initiate-data proposes the SRF requester or department head as manager; activation-init fills fields from onboarding profile -> offer -> SRF/vacancy (no FirstOrDefault defaults, probation from offer); activate validates every id inside the target company and returns field errors (no silent or cross-company fallback); blank NSSF/NHIF never cause a 500 (country statutory master decides if required).
Read first: docs/follow.md (rules), docs/ats_fixes.md (§0 decisions + §0.1 standing rules, §2 findings, §4 Step 7, §6 checklist), docs/HRMS User Manual.pdf §6–§8 and §19 if needed, and memory (ats-fixes-plan, e2e-tests-protect-user-data, approvals-live-in-task-page, no-static-values-use-masters, keep-usage-lean).
Rules: superadmin/developer can do everything; all approvals live in /task; no static values (read masters); keep tool output small.
Do:
1. Present the plan for Step 7 (follow.md #9, with the "> Code correction and static values" section) and wait for my approval.
2. Implement; DB changes as a dated idempotent block in docs/db_changes.sql; new permissions/menus registered (follow.md #8).
3. dotnet build, dotnet test and tsc --noEmit must be clean; add tests for this step.
4. Verify on a throwaway DB clone (never the live DB, never my API on :5197), using the MAKL test users from docs/ats_fixes.md §1 (they exist in clone HRMSCore_SrfTest; on a fresh clone recreate them as described there).
5. Update docs/ats_fixes.md (progress table + completion notes in §7) and the artifact srf-ats-ob: read https://claude.ai/artifact/7yKeUEJ5A9aqNUobFx9Hzm, edit docs/artifacts/srf-ats-ob.html (progress, findings, How to operate, flow diagram, counts), publish to that url.
6. Give me the copy-paste prompt for Step 8 from docs/ats_fixes.md §8 (update it first if this step changed anything).
```

### Prompt — Step 8: After activation: opening leave balances

```text
SRF / ATS / Onboarding fixes — Step 8 of 11: After activation: opening leave balances.
Findings covered: F17.
Goal: Create opening leave balances for the new hire at activation, the same way Add Employee does (D5, manual §8). The vacancy filled count is already done (Step 5, VacancyOpenings); keep it in the same transaction.
Read first: docs/follow.md (rules), docs/ats_fixes.md (§0 decisions + §0.1 standing rules, §2 findings, §4 Step 8, §6 checklist), docs/HRMS User Manual.pdf §6–§8 and §19 if needed, and memory (ats-fixes-plan, e2e-tests-protect-user-data, approvals-live-in-task-page, no-static-values-use-masters, keep-usage-lean).
Rules: superadmin/developer can do everything; all approvals live in /task; no static values (read masters); keep tool output small.
Do:
1. Present the plan for Step 8 (follow.md #9, with the "> Code correction and static values" section) and wait for my approval.
2. Implement; DB changes as a dated idempotent block in docs/db_changes.sql; new permissions/menus registered (follow.md #8).
3. dotnet build, dotnet test and tsc --noEmit must be clean; add tests for this step.
4. Verify on a throwaway DB clone (never the live DB, never my API on :5197), using the MAKL test users from docs/ats_fixes.md §1 (they exist in clone HRMSCore_SrfTest; on a fresh clone recreate them as described there).
5. Update docs/ats_fixes.md (progress table + completion notes in §7) and the artifact srf-ats-ob: read https://claude.ai/artifact/7yKeUEJ5A9aqNUobFx9Hzm, edit docs/artifacts/srf-ats-ob.html (progress, findings, How to operate, flow diagram, counts), publish to that url.
6. Give me the copy-paste prompt for Step 9 from docs/ats_fixes.md §8 (update it first if this step changed anything).
```

### Prompt — Step 9: Small fixes, static values, merge requisition -> requisitions

```text
SRF / ATS / Onboarding fixes — Step 9 of 11: Small fixes, static values, merge requisition -> requisitions.
Findings covered: F19, F20, F21, F22, F24 + static-values table.
Goal: Offer currency from company master; approver full names instead of login codes; scorecard range check (confirm D7 scale with me first - manual says 1-10); save positionTitle; replace the static values in the §2 static-values table (incl. the Step 3 rows: companies.offerApproval default, drop sp_ApproveOfferStep, offer-create master fallbacks, /task offer history labels); D6 merge per §0.2 (move modules/requisition into modules/requisitions, repoint routes, retire /masters/srf-matrices menu + redirect to /approval-matrices/requisitions, delete modules/requisition).
Read first: docs/follow.md (rules), docs/ats_fixes.md (§0 decisions + §0.1 standing rules, §2 findings, §4 Step 9, §6 checklist), docs/HRMS User Manual.pdf §6–§8 and §19 if needed, and memory (ats-fixes-plan, e2e-tests-protect-user-data, approvals-live-in-task-page, no-static-values-use-masters, keep-usage-lean).
Rules: superadmin/developer can do everything; all approvals live in /task; no static values (read masters); keep tool output small.
Do:
1. Present the plan for Step 9 (follow.md #9, with the "> Code correction and static values" section) and wait for my approval.
2. Implement; DB changes as a dated idempotent block in docs/db_changes.sql; new permissions/menus registered (follow.md #8).
3. dotnet build, dotnet test and tsc --noEmit must be clean; add tests for this step.
4. Verify on a throwaway DB clone (never the live DB, never my API on :5197), using the MAKL test users from docs/ats_fixes.md §1 (they exist in clone HRMSCore_SrfTest; on a fresh clone recreate them as described there).
5. Update docs/ats_fixes.md (progress table + completion notes in §7) and the artifact srf-ats-ob: read https://claude.ai/artifact/7yKeUEJ5A9aqNUobFx9Hzm, edit docs/artifacts/srf-ats-ob.html (progress, findings, How to operate, flow diagram, counts), publish to that url.
6. Give me the copy-paste prompt for Step 10 from docs/ats_fixes.md §8 (update it first if this step changed anything).
```

### Prompt — Step 10: MAKL set-up + live test users

```text
SRF / ATS / Onboarding fixes — Step 10 of 11: MAKL set-up + live test users.
Findings covered: MAKL set-up gaps (2).
Goal: Ask me before any live write. Then, through the app's own APIs: set MAKL department heads, fix/replace rule-offer-makl and add a proper OFFER rule on /approval-matrices/offers, fix the MANAGING_DIRECTOR step of the other companies' OFFER rules (nobody holds that role; see §2), add MAKL onboarding templates, and create the 6 test users of §1 on the live DB (Add Employee + Admin > Users, password MaklTest@2026). Report the real live logins and update §1 and the artifact.
Read first: docs/follow.md (rules), docs/ats_fixes.md (§0 decisions + §0.1 standing rules, §2 findings, §4 Step 10, §6 checklist), docs/HRMS User Manual.pdf §6–§8 and §19 if needed, and memory (ats-fixes-plan, e2e-tests-protect-user-data, approvals-live-in-task-page, no-static-values-use-masters, keep-usage-lean).
Rules: superadmin/developer can do everything; all approvals live in /task; no static values (read masters); keep tool output small.
Do:
1. Present the plan for Step 10 (follow.md #9, with the "> Code correction and static values" section) and wait for my approval.
2. Implement; DB changes as a dated idempotent block in docs/db_changes.sql; new permissions/menus registered (follow.md #8).
3. dotnet build, dotnet test and tsc --noEmit must be clean; add tests for this step.
4. Verify on a throwaway DB clone (never the live DB, never my API on :5197), using the MAKL test users from docs/ats_fixes.md §1 (they exist in clone HRMSCore_SrfTest; on a fresh clone recreate them as described there).
5. Update docs/ats_fixes.md (progress table + completion notes in §7) and the artifact srf-ats-ob: read https://claude.ai/artifact/7yKeUEJ5A9aqNUobFx9Hzm, edit docs/artifacts/srf-ats-ob.html (progress, findings, How to operate, flow diagram, counts), publish to that url.
6. Give me the copy-paste prompt for Step 11 from docs/ats_fixes.md §8 (update it first if this step changed anything).
```

### Prompt — Step 11: Full end-to-end re-test

```text
SRF / ATS / Onboarding fixes — Step 11 of 11: Full end-to-end re-test.
Findings covered: all.
Goal: Run every item of §6 on a fresh clone with the 6 MAKL users across all roles; record pass/fail, close or reopen findings in §2, write the final report in §7 and on the artifact.
Read first: docs/follow.md (rules), docs/ats_fixes.md (§0 decisions + §0.1 standing rules, §2 findings, §4 Step 11, §6 checklist), docs/HRMS User Manual.pdf §6–§8 and §19 if needed, and memory (ats-fixes-plan, e2e-tests-protect-user-data, approvals-live-in-task-page, no-static-values-use-masters, keep-usage-lean).
Rules: superadmin/developer can do everything; all approvals live in /task; no static values (read masters); keep tool output small.
Do:
1. Present the plan for Step 11 (follow.md #9, with the "> Code correction and static values" section) and wait for my approval.
2. Implement; DB changes as a dated idempotent block in docs/db_changes.sql; new permissions/menus registered (follow.md #8).
3. dotnet build, dotnet test and tsc --noEmit must be clean; add tests for this step.
4. Verify on a throwaway DB clone (never the live DB, never my API on :5197), using the MAKL test users from docs/ats_fixes.md §1 (they exist in clone HRMSCore_SrfTest; on a fresh clone recreate them as described there).
5. Update docs/ats_fixes.md (progress table + completion notes in §7) and the artifact srf-ats-ob: read https://claude.ai/artifact/7yKeUEJ5A9aqNUobFx9Hzm, edit docs/artifacts/srf-ats-ob.html (progress, findings, How to operate, flow diagram, counts), publish to that url.
6. Give me the copy-paste prompt for Step none - the work is finished; give me a closing summary instead from docs/ats_fixes.md §8 (update it first if this step changed anything).
```

---

## 9. Artifact — srf-ats-ob

- Link: https://claude.ai/artifact/7yKeUEJ5A9aqNUobFx9Hzm (private; share from the page's Share menu).
- Source: `docs/artifacts/srf-ats-ob.html`. Edit this file, then publish it to the link above (read the artifact first in a new chat).
- Contents: flow diagram (SRF → /task approvals → vacancy → ATS → offer → candidate portal → onboarding → activation), fix progress, decisions, **How to operate** per role with screen paths, company set-up checklist, findings, test users.
- Update it at the end of every step: progress pill, findings closed, How to operate (remove the bracketed "[Step n]" caveats once fixed), counts in the header, the diagram if the flow changed.
