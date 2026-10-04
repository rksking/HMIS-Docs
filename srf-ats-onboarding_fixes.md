# Requisitions (SRF), Recruitment (ATS) & Onboarding — Test Findings & Fix Plan

> Status: **revision 1 (2026-10-04) — testing done, no code changed yet. Waiting for answers in §0 and approval of Step 1.**
> Each step is reviewed/approved before code (follow.md #9). Every DB change goes to `docs/db_changes.sql` as a dated,
> idempotent block (follow.md #15). New actions get a `permissions` row in `<module>.<action>` form (follow.md #8).
> After each step: zero `tsc --noEmit` errors, zero `dotnet build` errors, `dotnet test` green (follow.md #11), and the
> E2E flow in §6 re-run on a throwaway DB clone (follow.md #12).

## Progress

| Step | Scope | Status |
|---|---|---|
| 1 | Security: permission checks on Onboarding + Approval-matrix endpoints | ⏳ Waiting for approval |
| 2 | SRF decisions only through the approval rules (no step skipping, no self-approval) | ⏳ Pending |
| 3 | Offer approval routing (designated user, all roles, same company, no hard-coded chain) | ⏳ Pending |
| 4 | SRF approval routing (department head = submitter, matched rule, no hard-coded chain) | ⏳ Pending |
| 5 | Reject / Return for revision / Resubmit for SRF and Offer | ⏳ Pending |
| 6 | Notifications for SRF and Offer steps | ⏳ Pending |
| 7 | Onboarding: form declaration, activation defaults, activation validation | ⏳ Pending |
| 8 | After activation: vacancy filled count, leave balances | ⏳ Pending |
| 9 | Small data fixes + static values + duplicate frontend module | ⏳ Pending |
| 10 | MAKL set-up (data through the UI) + live test users | ⏳ Pending (needs your go-ahead for live writes) |
| 11 | Full end-to-end re-test across all roles + report | ⏳ Pending |

---

## 0. Decisions needed (defaults used if no reply)

| # | Question | Default (recommended) |
|---|---|---|
| D1 | No REQUISITION / OFFER matrix rule matches the company. What happens? | **Refuse the submission** with "No approval matrix set up for <company> — ask HR to add one on /task". Same approach as payroll refusing when a company has 0 statutory configs. Removes the hard-coded chains (follow.md #1). |
| D2 | The SRF submitter is the head of the department. Who approves the Department Head step? | The head's own reporting manager; if none, the company's Company HR approver. Shown as "auto-escalated" with the reason. |
| D3 | Should `/requisitions` still have Approve/Reject buttons? | **No.** All approvals live in `/task`. The `/requisitions/{id}/approval-decision`, `budget-validate` and `hr-review` endpoints use the same rules as `/task` (or are removed if the UI no longer calls them). |
| D4 | A returned SRF is edited and resubmitted. Where does it restart? | **From step 1**, because the content changed. Previous decisions stay in the history. |
| D5 | Leave balances for a new hire created by activation | Create them the same way Add Employee does (current leave master and accrual rules). No new rules. |
| D6 | Duplicate frontend modules `src/modules/requisition/` and `src/modules/requisitions/` | Keep `requisition/` (the routes import it). Delete `requisitions/` (nothing imports it). |
| D7 | Interview score scale | 1–5 per criterion (the formula already divides by 25). Reject values outside 1–5. |

---

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
| F10 | 🟠 | "Return for revision" in /task sets the SRF to **REJECTED**; there is no edit/resubmit | SRF-2026-0719B → `REJECTED`. `/requisitions` sets `DRAFT` for the same action. `RequisitionsController` has no update or resubmit endpoint. |
| F11 | 🟠 | Rejection with empty remarks is accepted (SRF, Offer) | SRF-2026-DAC6D was rejected with `remarks:""`. `ats.md` step 3 says remarks are mandatory. |
| F12 | 🟠 | No notifications for any SRF/Offer event | 0 rows in `notifications` for all 6 users after the full flow. |
| F13 | 🟠 | Activation crashes with a raw DB error when the NSSF number is missing | `salary_details.nssfNumber` is NOT NULL. The user sees "An error occurred while saving the entity changes". |
| F14 | 🟠 | Activation form defaults are "first item in the list", not the SRF/offer/onboarding values | `OnboardingService` lines ~1599–1605. Job title Accountant (should be IT Technician), grade A0 (A1), reporting manager Achieng, a plain employee (Wanjiru was chosen at initiate), cost centre Clinical (ICT), probation 3 (offer: 6). |
| F15 | 🟠 | Activation silently falls back to another department/location/grade/cost centre, even another company's | `OnboardingService` lines ~1738–1768: `?? FirstOrDefault(company) ?? FirstOrDefault()` with `IgnoreQueryFilters()`. |
| F16 | 🟠 | The candidate's declaration is not enforced | Form submitted with `declarationConfirmed=false` and accepted. |
| F17 | 🟠 | Vacancy still OPEN with `filledCount=0` after its only opening was filled; no leave balances for the new hire | VAC-2026-E6F39 after activating employee 70063. `PROCESS_FLOW.md` §3.1 step 7 expects leave balances. |
| F18 | 🟡 | `/task` tier counter is one behind for offers (step 2 shows 1/3, step 3 shows 2/3); assigned name shows the wrong person | Same resolver as F5. |
| F19 | 🟡 | Offer currency saved empty when not sent | OFF-2026-0101 `currency=""`. The SRF already falls back to the company currency. |
| F20 | 🟡 | Approver name stored as the login code ("70059") instead of the person's name | `requisition_approvals.decidedByName`, `finalApprovedByName`. |
| F21 | 🟡 | Interview scores not range-checked | Sending 1–10 values gave an overall score of 164%. |
| F22 | 🟡 | `positionTitle` sent on offer create is not saved | The public offer page shows `positionTitle: null`. |
| F23 | 🟡 | Onboarding "proposed reporting manager" defaults to an unrelated employee | `initiate-data` proposed Elian Njuguna for an IT hire. It should be the SRF requester or the department head. |
| F24 | 🟡 | Duplicate frontend module | `src/modules/requisitions/` is a copy of `requisition/` and nothing imports it. |

### MAKL set-up gaps (data, not code)

- **No MAKL department has a head.** Every Department Head step stalls. On the clone, Kamau → Contact Centre and Wanjiru → IT were set via `PUT /api/org/departments/{id}`.
- **No OFFER rule.** `rule-offer-makl` is typed REQUISITION, is named "Offer", and has two CUSTOM steps with no user.
- **No onboarding templates.** The only 4 belong to `comp-lch-01`. Initiate accepted another company's template.

### > Code correction and static values (follow.md #10)

| Where | Static value | Fix |
|---|---|---|
| `RequisitionService.GetFallbackStepDefinitions` | Hard-coded 3-step chain | Remove; refuse (D1) |
| `ApprovalsService` ~950 | `"MANAGING_DIRECTOR"` / `"HR_REVIEW"` fallback, `totalSteps = 2` | Use the stored offer steps only |
| `ApprovalsService.ResolveOfferApproverAsync` | Job-title text match `"Director"/"Managing"/"Chief"/"CEO"`, global search | Use the matrix step (role / designated user) through `ApprovalEngine`, limited to the company |
| `RequisitionService` 510 / 519 | `"System User"`, `Region = "AFRICA_KE"` | Current user's name; region from the company's country master |
| `OnboardingService` 209 / 713 / 1126 | `"Healthcare Professional"` | Vacancy job title; empty if none |
| `OnboardingService` 1599–1605 | `FirstOrDefault()` defaults, `ProbationPeriodMonths = 3` | Values from the onboarding profile → offer → SRF |
| `OnboardingService` 1738–1768 | Silent fallbacks to any company's masters | Validation error naming the bad field |
| `GetDefaultRoleAssigneeName` | Display names like "Finance Controller" | OK as labels; replaced by the real resolved name once Step 4 is done |

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
- Tests: designated user gets the step; FINANCE step goes to the company finance user; no cross-company approver; creator refused.

### Step 4 — SRF approval routing (F7, F8-SRF, F9)
- Store `job_requisitions.matrixRuleId` at submit. `ResolveRequisitionApproverAsync` reads that rule's step, not "highest priority".
- DEPT_HEAD when the head is the requester → escalate per D2 and show the reason (`isSelfApprovalEscalated`).
- Remove `GetFallbackStepDefinitions` (D1). Make the tie-break deterministic.
- Save the resolved approver on `requisition_approvals` when the step becomes active (same as Step 3).
- DB: `job_requisitions.matrixRuleId` → `docs/db_changes.sql`.
- Tests: head = requester escalates; designated user from the matched rule; no rule → refused.

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
  - final approve / reject / return → the requester (SRF) or creator (Offer)
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
- Activation increments `vacancies.filledCount`; when `filledCount >= openingsCount`, the vacancy becomes FILLED/CLOSED (use the existing `VacancyStatus` value).
- Create leave balances the way Add Employee does (D5). Reuse that service; no new rules.
- Tests: vacancy closes on the last hire; the new hire has leave balances.

### Step 9 — Small fixes, static values, cleanup (F19–F22, F24, §2 static values)
- Offer currency → company currency master when empty (same as SRF).
- `decidedByName` / `finalApprovedByName` → the employee's full name (look up by user id).
- Scorecard values must be 1–5 (D7) → 400 otherwise.
- Save `positionTitle` on the offer.
- Replace `"Healthcare Professional"`, `"System User"`, `"AFRICA_KE"` per the §2 static-values table.
- Delete `frontend/src/modules/requisitions/` (D6).

### Step 10 — MAKL set-up + live users (data through the UI/API, after your go-ahead)
- Set department heads for MAKL departments (Org Masters).
- Fix `rule-offer-makl`: retype it to OFFER, or delete it and add a proper OFFER rule on /task with real approvers.
- Add MAKL onboarding templates (or copy Lifecare's).
- Create the 6 test users from §1 on the live DB (Add Employee + Admin › Users). Employee codes will differ from the clone.

### Step 11 — Full re-test (follow.md #12)
- Re-run §6 on a fresh clone with all 6 users. Update the §1 results and write completion notes per step below.

---

## 5. Open risks
- Changing `ResolveOfferApproverAsync` and `ResolveRequisitionApproverAsync` affects `/task` for every company. The Step 3/4 tests must cover `comp-lch-01` and `comp-afri-01` as well as MAKL.
- Existing PENDING SRFs and offers have no stored approver or rule. Step 3/4 must handle `matrixRuleId = null` (resolve once from the current rule, then save it).

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
