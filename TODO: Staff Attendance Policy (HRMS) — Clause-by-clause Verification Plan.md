# Staff Attendance Policy (HRMS) — Clause-by-clause Verification Plan

### Follow the instructions of 'docs/follow.md' file

**Source:** `docs/StaffAttendancePolicyHRMS.pdf` (v1.0, 12 sections, shown to the HR group on 2026-10-06).
**Built in:** `docs/policies.md` Phases 1–3 (Steps 1–29, 31–33) and roster go-live §46–49. The gap map is §27.
**Why this plan:** every clause was coded, but Step 30 (the browser walkthrough) was skipped, and most completion notes
say *"Not checked in a browser."* This plan proves each clause works for real users, or records a gap.

## Progress

| Step | Scope (PDF section) | Status |
|---|---|---|
| 0 | Baseline: the policy values vs the configured values per company; known gaps; test setup | ✅ **Done** (2026-10-07) — see §6 |
| 1 | §2 Working hours, shifts, roster lead time, shift swaps | ✅ **Done** (2026-10-07) — V-1.7 fixed; see §6 |
| 2 | §3 Marking attendance (biometric, web / phone, selfie / geofence, OD / WFH, proxy) | ✅ **Done** (2026-10-07) — V-2.5 fixed, OD renamed "Out Duty"; see §6 |
| 3 | §4 Grace, late tiers 1–3 / 4–5 / 6+, manager alert, leave-then-LOP, early going | ✅ **Done** (2026-10-07) — V-3.1–V-3.6; V-3.5 gap → F6; see §6 |
| 4 | §5 Status codes + §6 Absence (A = LOP, 3 consecutive days, sick > 2 days certificate) | ✅ **Done** (2026-10-07) — V-4.1/V-4.2 fixed (status codes), F3 seeded; see §6 |
| 5 | §7 Regularisation (3 working days, manager → HR, max 3 → HR head, after cut-off → next cycle) | ✅ **Done** (2026-10-07) — V-5.5 fixed; V-5.4 Blocker → F8, reasons → F9; see §6 |
| 6 | §8 Overtime and comp-off (pre-approval by manager **and** HR, credit after approval) | ✅ **Done** (2026-10-07) — V-6.1–V-6.15; K1 confirmed → F1; comp-off expiry Blocker V-6.11 → F10; security V-6.15; see §6 |
| 7 | §9 Leave and payroll (leave overrides, manager confirmation → lock on the cut-off, flow to payroll, post-lock arrears) | ✅ **Done** (2026-10-07) — cycle 15th–14th (cut-off 14); Blockers V-7.1–V-7.4, Gap V-7.5; see §6 |
| 8 | §10 Roles (Employee / Manager / HR Ops / Payroll, 2 working days), §11 non-compliance, §12 review | ✅ **Done** (2026-10-07) — /task stages and deadlines OK; Blocker V-8.2 (register readable by any login) → F13; HR Ops role gaps V-8.4; audit Gap V-8.8 → F14; see §6 |
| 9 | Close-out: findings → fix steps, HR walkthrough checklist, sign-off | ✅ **Done** (2026-10-07) — 9 Blockers open, 13 Gaps, 26 Minor, 14 Config; F16–F21 added; order + HR walkthrough in the artifact; see §5.0 / §6 |
| F1 | Overtime **pre-approval** (manager → HR before the shift; unapproved overtime flagged) — K1 | ⏳ after Step 6 |
| F2 | **Disciplinary case log** (HR opens a case from an alert, tracks it to closure) — K4 | ⏳ after Step 8 |
| F4 | Weekly offs: remove the dead **work schedule** option from bulk assignment; weekly offs come from the shift's day rules, the employee's weekday override or the roster (V-1.2) | ⏳ decided 2026-10-07 |
| F5 | **Biometric member-number matching** (V-2.2, V-2.3): match within the device's company, exact number first; devices on the unknown company `comp-001`; remove the sync's fake fallbacks; re-map stored logs | ⏳ decided 2026-10-07 |
| F3 | Sick / annual leave types missing in 9 companies (V-0.6) — master data, HR or script | ✅ **Done** (2026-10-07) — seed-from-template block in db_changes.sql (run on local / VM); see §5 / §6 |
| F6 | **Early-going manager alert** (V-3.5): the warning at the 4–5 tier fires only for late arrivals; extend it (or add a sibling) so an early-going-only warning day also alerts the manager, with early-aware wording | ⏳ decided 2026-10-07 — needs approval before coding |
| F7 | **Dead late columns** (V-3.6 / V-0.12): retire `company_attendance_policies.lateThresholdMinutes` and `shifts.gracePeriodMinutes` (read nowhere in the late calc) — drop or wire | ⏳ decided 2026-10-07 — needs approval before coding |
| F8 | **Approved regularisation survives re-processing** (V-5.4 Blocker, V-5.7): `sp_ProcessDailyAttendance` uses the approved regularisation's times (like an approved manual adjustment), so late / half-day are worked out and a re-process (biometric sync, leave, roster) no longer puts the day back to ABSENT | ⏳ decided 2026-10-07 — plan → approval before coding |
| F9 | **Regularisation reasons master** (V-5.8): per-company reasons HR maintains (seed: missed punch, device failure, out duty), replacing the 8 hardcoded ESS options | ⏳ decided 2026-10-07 — plan → approval before coding |
| F10 | **Comp-off rules** (V-6.6–V-6.8, V-6.10, V-6.11, V-6.13): leave on a date after the credit expires is refused (balance as of the leave date); a credit keeps its full validity across 31 Dec; only the employee's manager grants; OFF_DAY_WORK only on a weekly off / holiday; a part-day grant stops only that part of the day's overtime; company-local dates | ⏳ proposed 2026-10-07 — plan → approval before coding |
| F11 | **Approved leave overrides punches** (V-7.1): full-day leave → ON_LEAVE whatever the punches (no late / half day / late occurrence); half-day leave → the other half is measured against its own half of the shift | ⏳ proposed 2026-10-07 — plan → approval before coding |
| F12 | **Lock only after the period ends** (V-7.3, V-7.11): team confirmation and payroll approval refused before the period's last day has passed (company-local date); payroll run warns / refuses with no attendance | ⏳ proposed 2026-10-07 — plan → approval before coding |
| F13 | **Row and company scoping for reads** (V-8.2, V-8.3 — security): `GET api/attendance/daily` (and the other register reads) and `GET api/employees` return only what the caller's data level allows — self for ESS, own team for a line manager, the companies the user may access for HR / payroll; a company the user cannot access is refused; `ALL` no longer 500s. Review the `portal.* → *.view` implications | ⏳ proposed 2026-10-07 — plan → approval before coding |
| F14 | **Audit trail for policy and access changes** (V-8.8): TA settings, approval deadlines, email settings, role permission grants, user role changes, payroll approve / run, team confirmation → `audit_log` (who, when, old → new); audit viewer for HR (`audit.view` on HR roles, data) | ⏳ proposed 2026-10-07 — plan → approval before coding |
| F15 | **/task completeness** (V-8.9, V-8.7): payroll arrears and attendance-adjustment decisions in the /task inbox (D4); leave HR-stage deadline counts from the manager's decision (`leave_decisions.decidedAt`), not from submission | ⏳ proposed 2026-10-07 — plan → approval before coding |
| F16 | **Sign-in backdoor** (V-6.15 — security): `PasswordHasher.VerifyPassword` accepts six fixed demo passwords and the stored hash for every account; remove them, find accounts whose stored value is not a hash and force a reset | ⏳ proposed 2026-10-07 (Step 9) — plan → approval before coding |
| F17 | **Payroll totals** (V-7.2, V-7.4, V-7.10, V-7.8): absence / half-day LOP deducted once; a LOCKED (approved) run cannot be re-run ("raise an arrear"); approver name from the user record; decide the tax treatment of late / early LOP | ⏳ proposed 2026-10-07 (Step 9) — plan → approval before coding |
| F18 | **Leave on a paid day** (V-7.5): applying / approving leave on a `PayrollLocked` (or closed) day is refused with "raise an arrear", like V-5.5 | ⏳ proposed 2026-10-07 (Step 9) — decision: refuse (recommended) or auto-arrear |
| F19 | **Static-value sweep** (V-7.9, V-8.10): payroll refuses a statutory setup without PAYE bands instead of `GetDefaultKenyaBands()` and the 0.0275 / 300 / 0.015 / 20000 fallbacks; the payroll item on /task uses the country's statutory labels and invariant number format | ⏳ proposed 2026-10-07 (Step 9) — plan → approval before coding |
| F20 | **Role clean-up** (V-8.4, V-8.5, V-6.4, V-7.7): name the HR Ops role (recommended: Company HR) and give it the attendance HR duties; take shift / grace masters and payroll reports away from Line Manager; a `db_changes.sql` block so live matches | ⏳ proposed 2026-10-07 (Step 9) — decision on the role, then a data block |
| F21 | **Attendance small fixes** (V-1.3, V-2.9, V-2.10, V-2.11, V-2.13, V-5.9): company default shift for shared shifts, OD / WFH days show the shift times, second clock-in does not overwrite, remove the HR "Quick Clock" buttons, ESS drawer warns on a closed period, developer sees the manager stage on /task | ⏳ proposed 2026-10-07 (Step 9) — plan → approval before coding |
| F* | More fix steps, added from the findings (one per session) | — |

## 1. Policy values to verify (the bracketed numbers in the PDF)

| # | PDF clause | Policy value | Where it is configured (from policies.md) |
|---|---|---|---|
| P1 | Applies to | Permanent, probationary, contract staff | Employment types master / policy "Applies to" (§48) |
| P2 | §2 Roster published before the period | 5–7 days | Roster publication lead time (Step 26, §37) |
| P3 | §4 Grace | 30 min, 3 times a month | `/masters/shift-grace-policies` (Step 1) |
| P4 | §4 Late tiers | 1–3 none · 4–5 manager alert · 6+ half-day leave, else LOP | `late_penalty_tiers`, `LATE_WARNING`, `LEAVE_THEN_LOP` (Step 20, §29) |
| P5 | §4 Early going | Same as late | Early penalty (Step 21, §30) |
| P6 | §6 Consecutive absence | 3 days | Absence threshold + `ABSENCE_ESCALATION` (Step 22, §33) |
| P7 | §6 Sick certificate | Beyond 2 days | Leave type `requiresProof` / `proofThresholdDays` (Step 19, §28) |
| P8 | §7 Regularisation window | 3 working days | Step 24, §35 |
| P9 | §7 Monthly quota | 3, then HR head | Step 24, §35 |
| P10 | §9 Cut-off | **14th → payroll cycle 15th–14th** (user, 2026-10-07; PDF example 25th); configurable per company | TA general settings / close month (Step 4, 4b) |
| P11 | §10 Manager SLA | 2 working days + reminders | Approval deadlines (Step 29, §40) |

## 2. Known gaps before testing (declared in §27 / §41 as "not planned")

| # | PDF clause | State | Decision (2026-10-07) |
|---|---|---|---|
| K1 | §8 "Overtime … must be **pre-approved** by the manager and HR" | Overtime is authorised **after** the work (D13 not planned) | **Build pre-approval** → F1 |
| K2 | §3 "Proxy or buddy punching is a disciplinary offence" | Controls exist (selfie, geofence, registered phone); no report flags proxy punching (D12) | **Rely on the controls** (selfie, geofence, registered phone) — verified in Step 2, no report |
| K3 | §3 "mobile app" | Portal works in a phone browser; no native app | **Phone browser is the approved mobile method** — tested at phone size in Step 2 |
| K4 | §11 Disciplinary action | No disciplinary case record; only alerts and reports | **Build a disciplinary case log** → F2 |
| K5 | §12 Annual review | No policy version / review date in the system | **Out of scope** |

**P1 (Applies to):** the rules apply to **all** employment types, as today; the PDF list is the minimum, not an exclusion.
**Policy values:** HR enters them through the UI (policy screens) — not by script. Because every clone is a fresh copy
of HRMSCore_Local (where the values are not set), **each step first sets the values it needs on the clone through the
UI or API**, then tests. HR repeats the final configuration on live at sign-off (Step 9).

## 3. How every step is run (same method each time)

1. **Read the clause** in the PDF and the matching completion notes in `docs/policies.md` (only that section).
2. **Configuration check (read-only SQL):** the values for each company match the policy table §1.
3. **Code trace:** find where the rule is enforced (service / stored procedure), and confirm that it is not hardcoded
   (follow.md #1, #10).
4. **Scenario test on a throwaway clone only** ([[e2e-tests-protect-user-data]]): `HRMSCore_StepTest` restored from a
   COPY_ONLY backup, test API on **:5299**, `BiometricGateway__AutoSyncEnabled=false`, the clone dropped and the `.bak`
   removed at the end. Never write to `HRMSCore_Local`, and never touch the user's API on :5197. SQL-only checks go in
   `BEGIN TRAN … ROLLBACK`.
   Each scenario = role (Employee / Line Manager / HR / Payroll) → action → expected result per the PDF → actual result.
   Test the **edge** each time (for example the 3rd late vs the 4th, day 3 vs day 4 of the window, 2 sick days vs 3).
5. **Browser checklist:** add the step's manual tests (log in as, page, action, expected) to the **"Policy
   verification"** section of the management artifact. Do not end with "check in the browser"
   ([[manual-testing-in-artifact]]).
6. **Record findings** in §5 as `V-<step>.<n>`, with a severity (**Blocker** = the policy is broken · **Gap** = not built
   · **Minor** = wording or UI · **OK**).
7. **Fixes:** show the fix plan, and fix only after the user approves (follow.md #9). A small fix (one file, no schema
   change) is made in the same session. Anything bigger becomes an **F-step** in Progress.
8. **Close the session:** update Progress + §5 + `docs/policies.md` (one line per step under §55) + the artifact +
   memory, then print the **next-step prompt** from §4.

## 4. Per-step prompts (start each in a fresh session)

> Common prefix for every prompt: *"Read docs/attendance_policy_verification.md (§1–§3 and the Progress table) and
> docs/follow.md. Follow the method in §3 exactly. Read only the policies.md sections named for the step.
> Start the clone with docs/scripts/stepTestClone.sh up (and ui for browser tests), set this step's policy values on
> the clone first (§2 note), check the step's V-0.x findings in §5, and finish with down."*

- **Step 0:** "…prefix… Do Step 0: (a) read-only SQL — for every active company list the configured value of each
  policy item P1–P11 next to the PDF value, and show the mismatches; (b) ask me the K1–K5 decisions; (c) prepare and
  verify the throwaway-clone script (create / API on :5299 / drop) and save it as docs/scripts/stepTestClone.md;
  (d) add an empty 'Policy verification' tracker section to the management artifact. Record in §5 and update Progress."
- **Step 1:** "…prefix… Do Step 1 (PDF §2): shift master patterns General / Morning / Evening / Night, weekly offs per
  entity / department / roster, roster upload, publication lead time 5–7 days (policies.md §12, §37, §48), shift swap
  colleague → manager → roster updated before the shift (§38). Clone tests + browser checklist, findings in §5."
- **Step 2:** "…prefix… Do Step 2 (PDF §3): biometric sync, web clock in / out, phone sign-in with a registered mobile,
  selfie / geofence flags, clock-in access, OD and WFH requests → day marked OD / WFH with shift times (policies.md §11,
  §25, §34, §45). Proxy punching per the K2 decision. Clone tests + browser checklist, findings in §5."
- **Step 3:** "…prefix… Do Step 3 (PDF §4): grace 30 min × 3 a month; lates 1–3 no action, 4–5 LATE_WARNING to the
  manager, 6th onward half-day from leave then LOP if no balance; early going without approval counted the same
  (policies.md §6, §29, §30). Test the 3rd/4th, 5th/6th boundaries, zero leave balance, and late + early on the same day.
  Clone tests + browser checklist, findings in §5."
- **Step 4:** "…prefix… Do Step 4 (PDF §5 + §6): all nine codes P / A / HD / WO / PH / L / OD / WFH / LOP shown in the
  ESS calendar, muster / reports and export; unauthorised absence = A and LOP in payroll; 3 consecutive days without
  information → ABSENCE_ESCALATION; sick leave over 2 days refused without a certificate, HR waiver
  (policies.md §28, §33, §34, §46). Clone tests + browser checklist, findings in §5."
- **Step 5:** "…prefix… Do Step 5 (PDF §7): missed punch / device failure / on-duty regularisation within 3 working
  days, manager approves → HR verifies, 4th request in a month → HR head, request after the payroll cut-off → next
  cycle (policies.md §35, §39, §45). Clone tests + browser checklist, findings in §5."
- **Step 6:** "…prefix… Do Step 6 (PDF §8): overtime and work on WO / PH per the K1 decision; overtime pay vs comp-off,
  comp-off credited only after manager + HR approval, with expiry; ESS view (policies.md §10, §22, §31, §32).
  Clone tests + browser checklist, findings in §5."
- **Step 7:** "…prefix… Do Step 7 (PDF §9): approved leave overrides the punch flags; team confirmation by managers →
  lock on the cut-off; payroll refuses unlocked attendance; LOP, overtime and late deductions on the payslip; post-lock
  change → HR approval → arrear in the next run (policies.md §26, §36, §39, §44). On the clone, create a statutory setup
  first (payroll refuses without one). Clone tests + browser checklist, findings in §5."
- **Step 8:** "…prefix… Do Step 8 (PDF §10–§12): log in as Employee, Line Manager, HR Ops and Payroll and confirm that
  each can do exactly their duties (and no more) — on /task only ([[approvals-live-in-task-page]]); approval deadline
  2 working days, overdue flag and reminders; audit trail for every change; K4 / K5 decisions
  (policies.md §40, §44, rbac_fixes.md). Clone tests + browser checklist, findings in §5."
- **Step 9:** "…prefix… Do Step 9: summarise all findings in §5 by severity, propose the F-steps (scope, files, DB
  change, test) for my approval, finish the HR walkthrough checklist in the artifact, and add per-step prompts for the
  F-steps to §4."
- **F1 (after Step 6):** "…prefix… Do Step F1 — overtime pre-approval (K1): show me the design (request before the
  shift, manager → HR stages, how worked overtime without one is flagged, payroll effect) before coding."
- **F2 (after Step 8):** "…prefix… Do Step F2 — disciplinary case log (K4): show me the design (open from an alert
  or by hand, statuses, who sees it, menu + permissions in the DB) before coding."
- **F4:** "…prefix… Do Step F4 (V-1.2, V-1.8): plan to remove the work-schedule choice from bulk shift assignment
  (UI + API + any reader), the on-screen note 'weekly offs come from the shift', the readable filter summary, and what
  happens to rows already saved in employee_shift_assignments.workScheduleId — for my approval before coding."
- **F5:** "…prefix… Do Step F5 (V-2.2, V-2.3): plan the biometric member-number matching — match within the device's
  company (`biometric_devices.companyId`) and the exact number first, what to do with the 9 devices on `comp-001` and
  unmapped punches, remove the sync's fake fallbacks, and how to re-map stored logs and reprocess the affected days —
  for my approval before coding."
- **F8:** "…prefix… Do Step F8 (V-5.4, V-5.7): plan how `sp_ProcessDailyAttendance` reads an APPROVED regularisation
  (requested in / out replace or complete the punches, as `attendance_adjustments` do), what `RegularisationRules.ApplyToDay`
  keeps doing (or re-processes the day instead of setting PRESENT), late / half-day for a late requested time, a missed
  out-punch only, PayrollLocked days, and the re-test (approve → re-process → still PRESENT) — for my approval before coding."
- **F9:** "…prefix… Do Step F9 (V-5.8): plan a per-company regularisation reasons master (table, menu under Attendance
  masters, permission, seed missed punch / device failure / out duty, ESS drawer reads it, HR drawer too) — for my
  approval before coding."
- **F11:** "…prefix… Do Step F11 (V-7.1): plan how `sp_ProcessDailyAttendance` lets an approved full-day leave win over
  punches (ON_LEAVE, no late / half day, not a late occurrence, overtime?) and how a half-day leave measures the other half
  (late from the second-half start, half-day credit), then re-test late tiers and payroll — for my approval before coding."
- **F12:** "…prefix… Do Step F12 (V-7.3, V-7.11): plan to refuse team confirmation and payroll approval until the
  attendance period has ended (company-local date, cut-off 14 → from the 15th), what payroll run does for a period with
  no attendance yet, and `fn_IsAttendanceDateClosed` in company time — for my approval before coding."
- **F13:** "…prefix… Do Step F13 (V-8.2, V-8.3 — security): plan row and company scoping for `GET api/attendance/daily`
  (and the other register / summary reads) and `GET api/employees` by data level (self / team / company access), refuse a
  company the user cannot access, fix `ALL` → 500, and review the `portal.* → *.view` implications — for my approval
  before coding."
- **F14:** "…prefix… Do Step F14 (V-8.8): plan the audit trail (old → new, who, when) for TA settings, approval deadlines,
  email settings, role grants / user roles, payroll run / approve and team confirmation, plus an audit viewer for HR —
  for my approval before coding."
- **F15:** "…prefix… Do Step F15 (V-8.9, V-8.7): plan payroll arrears and attendance adjustments in the /task inbox and the
  leave HR-stage deadline from `leave_decisions.decidedAt` — for my approval before coding."
- **F6:** "…prefix… Do Step F6 (V-3.5): plan the early-going manager alert — `sp_GetLateArrivalWarnings` also returns days
  whose `earlyTierAction` is ALERT_MANAGER (or a sibling procedure), the email wording for an early departure (template
  placeholders, a new `EARLY_GOING_WARNING` template or one shared template), the notification setting, and the re-test
  (early-only day #4 → manager alerted once) — for my approval before coding."
- **F7:** "…prefix… Do Step F7 (V-3.6, V-0.12): plan retiring `company_attendance_policies.lateThresholdMinutes` and
  `shifts.gracePeriodMinutes` (every reader in entities / DTOs / screens / procedures, the Shift Master field, the
  `db_changes.sql` drop or keep-but-hide) — for my approval before coding."
- **F10:** "…prefix… Do Step F10 (V-6.6–V-6.8, V-6.10, V-6.11, V-6.13): plan comp-off — balance checked as of the leave
  date (apply and final approval), a credit valid for its full days across 31 Dec (`fn_LeaveEntitlementAsOf`), only the
  employee's manager grants (HR exception?), OFF_DAY_WORK only on a weekly off / holiday, a part-day grant stops only
  that part of the day's overtime, company-local dates — for my approval before coding."
- **F16 (security — first):** "…prefix… Do Step F16 (V-6.15): plan removing the demo-password and hash-as-password
  fallback from `PasswordHasher.VerifyPassword`, a read-only count of users whose stored value is not a real hash (they
  could not sign in afterwards → force reset / temporary password), what the seed / DbInitializer and the follow.md
  credentials table rely on, and the clone re-test (demo password on another account refused; real password works) —
  for my approval before coding."
- **F17:** "…prefix… Do Step F17 (V-7.2, V-7.4, V-7.10, V-7.8): plan deducting absence / half-day LOP once
  (`PayrollService` customDeductions vs `UnpaidAbsenceDeduction`, all country paths in `PayrollCalculationEngine`),
  refusing `RunPayrollAsync` for a LOCKED run, the approver's name from the user record (`PayrollController`), and the
  tax treatment of late / early LOP (before or after tax — my decision) — then re-run October on the clone and compare
  net pay with a hand calculation — for my approval before coding."
- **F18:** "…prefix… Do Step F18 (V-7.5): plan refusing leave apply / approval on a `PayrollLocked` or closed day
  (`LeaveService`, the HR stage on /task, the ESS message pointing to Payroll → Arrears, as `RegularisationRules.PaidDayRefusal`)
  — or my alternative, an automatic arrear — for my approval before coding."
- **F19:** "…prefix… Do Step F19 (V-7.9, V-8.10): plan the static-value sweep — `GetDefaultKenyaBands()` and the SHIF /
  AHL / pension fallbacks in `PayrollCalculationEngine` (refuse with a clear message instead), the Kenyan labels and
  culture-grouped amounts in the /task payroll item (`ApprovalsService`) — for my approval before coding."
- **F20:** "…prefix… Do Step F20 (V-8.4, V-8.5, V-6.4, V-7.7): show me the role × permission matrix for Company HR,
  HR Manager, Group HR, Line Manager, Finance / Finance Manager today, my choice of the HR Ops role, the permissions to
  add / remove (attendance.manage, shifts.*, attendance.dayrequest.hr, notifications.manage, audit.view,
  overtime.settings.manage, attendance.confirm.team) and the idempotent `db_changes.sql` block — for my approval before
  running it on the clone and re-running the Step 8 duty matrix."
- **F21:** "…prefix… Do Step F21 (V-1.3, V-2.9, V-2.10, V-2.11, V-2.13, V-5.9): plan the small attendance fixes — a
  per-company default shift for shared shifts (`shift_companies` or the company policy), OD / WFH days showing the shift
  times in the register, a second clock-in not overwriting the first, removing the HR Quick Clock buttons
  (`AttendanceView`, `AttendanceDashboardView`), the regularisation allowance checking a closed period, and the developer
  seeing manager-stage items on /task — for my approval before coding."
- **F-steps:** "…prefix… Do Step F<n> as written in Progress / §5: plan → approval → code following the existing
  patterns → db_changes.sql block (idempotent) → dotnet build / test, tsc --noEmit → re-run the failed scenario on the
  clone → mark the finding Fixed."

## 5. Findings

### 5.0 Summary by severity (Step 9, 2026-10-07)

100 findings (99 V-rows + F3). Re-checked read-only on HRMSCore_Local on 2026-10-07: every V-0.x config value is
still unset (grace 15 in all 15 companies, 0 late tiers, early going OFF, absence days / cut-off / lead time blank, WFH off,
confirmation off, 0 overtime types need approval); sick leave in 6 companies only (the F3 block is not yet run on local);
`HRMSCore_SrfTest` still on the server.

| Severity | Open | Fixed | Findings → step |
|---|---|---|---|
| **Blocker** | 9 | 3 | V-6.15 → **F16**, V-8.2 → **F13**, V-7.2 + V-7.4 → **F17**, V-7.3 → **F12**, V-7.1 → **F11**, V-5.4 → **F8**, V-2.2 → **F5**, V-6.11 → **F10**. Fixed: V-1.7, V-2.5, V-5.5 |
| **Gap** | 13 | 1 (F3 on clone) | V-8.3 → F13; V-7.5 → F18; V-5.7 → F8; V-6.8, V-6.10 → F10; V-6.2 → F1; V-3.5 → F6; V-1.2 → F4; V-8.4, V-8.5 → F20; V-8.8 → F14; V-8.9 → F15; V-0.6 → F3 (**run on local / VM**) |
| **Minor** | 26 | 5 | V-6.6, V-6.7, V-6.13 → F10; V-6.14 → F1; V-2.3 → F5; V-1.8 → F4; V-3.6, V-0.12 → F7; V-5.8, V-6.12 → F9; V-7.11, V-8.11 → F12; V-8.7 → F15; V-7.8, V-7.10 → F17; V-7.9, V-8.10 → F19; V-7.7 → F20; V-1.3, V-2.9, V-2.10, V-2.11, V-2.13, V-5.9 → F21; V-0.14 (JIVA a test company?) and V-0.15 (drop HRMSCore_SrfTest) → **your decision**. Fixed: V-2.12, V-4.1, V-4.2, V-7.12, V-8.13 |
| **Config** (HR / server at go-live) | 14 | — | V-0.1 – V-0.5, V-0.7 – V-0.11 (policy values P2–P11), V-2.1 (map member numbers), V-5.10 (request emails on), V-6.1 (overtime authorisation), V-6.4 → F20. HR walkthrough Part A in the artifact |
| **OK** | 29 | — | Incl. by decision: V-4.3 (A = LOP one status), V-5.6 (after cut-off = refuse + arrear), V-8.12 (K4 → F2, K5 out); known limits V-1.5 (no reminder before publish-by), V-2.7 (desktop-site mode) |

**Proposed order (security and money first, then data correctness, then process, then polish):**

| Wave | F-steps | Why first |
|---|---|---|
| 1 — before any employee login or payroll | **F16**, **F13**, **F17**, **F12**, **F11** | Anyone can sign in as anyone; any login reads every register; net pay wrong; months locked early; leave deducted as late |
| 2 — attendance data correct | **F8**, **F5**, **F10**, **F18** | Approved corrections lost; punches on the wrong person; comp-off after expiry; leave on paid days |
| 3 — roles, audit, approvals | **F20**, **F14**, **F15**, **F1** | HR Ops can do its job; changes traceable; all approvals on /task; overtime pre-approval |
| 4 — polish | **F6**, **F9**, **F4**, **F7**, **F19**, **F21**, **F2** | Alert / master / dead-setting / static-value clean-ups; disciplinary log |

**F-step proposals (scope · files · DB change · test)** — each needs your approval of its plan before coding:

| Step | Scope | Main files | DB change (`db_changes.sql`) | Re-test on the clone |
|---|---|---|---|---|
| F1 | Overtime request before the shift, manager → HR; worked overtime without one flagged / not paid; not your own | `OvertimeService`, `OvertimeAuthorization`, `ApprovalsService`, ESS request drawer, Overtime page | New table (pre-approval requests) + permissions + notification events | V-6.2 / V-6.14: request → manager → HR → worked → paid; no request → flagged |
| F2 | Disciplinary case log from an alert or by hand, statuses to closure | New service + `modules/discipline` (or under HR) | New tables (case, case events), permissions `discipline.*`, menu row | Open from UNAUTHORISED_ABSENCE / LATE_WARNING, close; who sees it |
| F4 | Remove the bulk work-schedule choice and dead readers; readable filter summary | `ShiftService` (≈ :597), bulk-assignment UI | Optional: null `employee_shift_assignments.workScheduleId` (decision) | V-1.2 / V-1.8: option gone, summary shows "Legal" |
| F5 | Biometric match within the device's company, exact number first; devices on `comp-001`; no fake fallbacks; re-map logs | `BiometricService` (`SyncWithWatermarkAsync`), `BiometricBackgroundSyncService` | Data block: device `companyId`, re-map `biometric_punch_logs`, reprocess affected days | V-2.2: MAKL 20001 punch → MAKL employee; unmapped → exception in the device's company |
| F6 | Early-going-only warning day alerts the manager | `sp_GetLateArrivalWarnings`, `NotificationService`, `NotificationTemplates` | Alter procedure; template / event seed | Early-only #4 → one manager email (dry run) |
| F7 | Retire the two dead grace columns | Entities, DTOs, Shift Master form | Drop (or keep-unused) `lateThresholdMinutes`, `shifts.gracePeriodMinutes` | Build / tsc; simulator unchanged |
| F8 | `sp_ProcessDailyAttendance` uses the approved regularisation's times; late / half day from them | `sp_ProcessDailyAttendance`, `RegularisationRules.ApplyToDay` | Alter procedure | V-5.4 / V-5.7: approve → re-process → still PRESENT; 10:30 → late |
| F9 | Per-company regularisation (and comp-off grant) reasons master | New master service + Attendance masters page; ESS + HR drawers; `CompOffService` | New table + seed (missed punch, device failure, out duty), permission, menu | V-5.8 / V-6.12: drawer lists the master; HR edits it |
| F10 | Comp-off balance on the leave date; full validity across 31 Dec; manager-only grant; OFF_DAY_WORK on WO / PH; part-day grant; local dates | `CompOffService`, `CompOffDecision`, `LeaveService`, `fn_LeaveEntitlementAsOf` | Alter function | V-6.11: 15 Dec refused; V-6.10: 1 Jan still 1.00; V-6.6–6.8 |
| F11 | Approved full-day leave → ON_LEAVE over punches; half-day leave measures the other half | `sp_ProcessDailyAttendance` (+ late sequence) | Alter procedure | V-7.1: 24 / 26 / 28 Sep ON_LEAVE, no late #; FIRST_HALF + 09:00 not late |
| F12 | Confirmation and payroll approval only after the period ends; payroll with no attendance; PERIOD_END in company time | `TeamConfirmationService`, `PayrollService.ApproveCycleAsync`, `fn_IsAttendanceDateClosed` | Alter function | V-7.3 / V-8.11: on 7 Oct refused, from 15 Oct allowed; November refused or warned |
| F13 | Register / summary reads and the employee list by data level and company access; `ALL` no 500; implications review | `AttendanceService.GetDailyAttendanceAsync` (+ other reads), `EmployeeService`, `AttendanceScopeRules`, permission implications | Possibly the implication rows (`portal.* → *.view`) | V-8.2 / V-8.3: employee → own rows; manager → team; other company → 403 |
| F14 | Audit (who, when, old → new) for TA settings, deadlines, email settings, role grants, user roles, payroll run / approve, team confirmation; HR audit viewer | `ConfigService`, `ApprovalSlaService`, `NotificationService`, `AdminService`, `PayrollService`, `TeamConfirmationService` | Grant `audit.view` to HR roles (data) | V-8.8: deadline 2 → 3 appears in the log with the user |
| F15 | Arrears and adjustments on /task; leave HR stage from `leave_decisions.decidedAt` | `ApprovalsService`, `ApprovalSla`, `PayrollArrearService` | None expected | V-8.9 / V-8.7: both listed on /task; HR due 2 days after the manager |
| F16 | Remove the password backdoor; force reset for non-hash accounts | `PasswordHasher`, seed / `DbInitializer` | Read-only check first; possibly set `forcePasswordChange` | V-6.15: `EmpPassword!2026` on `employee` → refused |
| F17 | LOP once; no re-run of a LOCKED run; approver name; late-LOP tax treatment | `PayrollService`, `PayrollCalculationEngine`, `PayrollController` | None | V-7.2: Elian net ≈ 36,555; V-7.4: re-run refused |
| F18 | Leave refused on a paid / closed day | `LeaveService`, approval HR stage | None | V-7.5: 16 Sep leave refused with "raise an arrear" |
| F19 | No built-in Kenyan bands / rate fallbacks; country labels on /task | `PayrollCalculationEngine`, `ApprovalsService` | None | Empty band list → payroll refuses; TZ company shows its own labels |
| F20 | HR Ops role gets its duties; Line Manager loses masters / payroll reports | — (data) | `role_permissions` block (idempotent) | Step 8 duty matrix re-run: Company HR 200s, manager 403 on shift policies |
| F21 | Default shift for shared shifts; OD times; second clock-in; Quick Clock removed; closed-period warning; developer on /task | `fn_EmployeeDaySchedule` / `ShiftService`, `AttendanceService`, `AttendanceView`, `AttendanceDashboardView`, `RegularisationRules`, `ApprovalVisibilityRules` | Possibly a default-shift column (decision) | V-1.3: the 7 resolve a shift; V-2.10: first in-time kept; V-5.9 drawer warns |

| ID | PDF clause | Scenario | Expected | Actual | Severity | Fix / step | Status |
|---|---|---|---|---|---|---|---|
| V-0.1 | §4 Grace | Config, 15 companies | 30 min, 3 / month | `graceLateMinutes` **15** in all companies, and every one of the 55 active shifts has `gracePeriodMinutes` **15**, `earlyGraceMinutes` 10. Times 3 ✔. Which of the two (company vs shift) wins → Step 3 | Config | HR via UI | Open |
| V-0.2 | §4 Tiers | Config | 1–3 none · 4–5 alert · 6+ half day | `late_penalty_tiers` **empty**; `lateDeductionMode` HALF_DAY with `lateBeyondGraceAlwaysDeducts` = 1 → today **every** late beyond grace deducts half a day at once | Config | HR via UI | Open |
| V-0.3 | §4 Leave-then-LOP | Config | Half day from leave, LOP if no balance | `latePenaltyLeaveTypeId` NULL everywhere (a leave type must be chosen for LEAVE_THEN_LOP) | Config | HR via UI | Open |
| V-0.4 | §4 Early going | Config | Same as late | `earlyGoingMode` **OFF** in all | Config | HR via UI | Open |
| V-0.5 | §6 3 consecutive days | Config | Alert at 3 | `unauthorisedAbsenceAlertDays` NULL; scheduled check off (`Notifications:UnauthorisedAbsenceCheckEnabled` false) | Config | HR via UI + appsettings | Open |
| V-0.6 | §6 Sick > 2 days | Master data | Certificate required | Set correctly (`requiresProof`, threshold 2, before approval) on SICK_FULL / SICK_HALF in **6** companies (AFRICARE, AFRIHOSP, LIFECARE, MAKL, NMC, SAINI). **9 companies have only COMP_OFF — no sick or annual leave at all**: AGI, BLISS (1 384 staff), DINLAS, JIVA, LCF, NEWAGE, SDI, SDI_HOLD, SDLI. Leave-then-LOP there always falls to LOP | **Gap** | F3 | Open |
| V-0.7 | §7 Window / quota | Config | 3 working days; 3 / month then HR head | Window NULL (no deadline); cap 3 ✔; over-quota role NULL → the 4th is **refused**, not escalated. Two stages manager → HR are always on (by reporting manager) ✔. Step 5: works once set (V-5.1, V-5.3) | Config | HR via UI | Open |
| V-0.8 | §9 Lock | Config | Cut-off 25th, after manager confirmation | `cutOffDay` NULL, `closeMonthMode` PAYROLL_APPROVAL, `requireManagerConfirmationBeforeLock` 0 | Config | HR via UI | Open |
| V-0.9 | §2 Roster lead time | Config | 5–7 days | `rosterPublishLeadDays` NULL | Config | HR via UI | Open |
| V-0.10 | §10 Manager SLA | Config | 2 working days + reminders | `approval_sla_settings` = **2 working days** for all 6 modules in all 15 companies ✔. But **all four scheduled checks are off** in `backend/Api/appsettings.json` (late warning, unauthorised absence, approval reminder, team confirmation) → no alert or reminder is ever sent. Step 8: deadlines, overdue flag and reminders work once switched on (V-8.6) | Config | appsettings at go-live | Open |
| V-0.11 | §5 WFH | Config | WFH code in use | `workFromHomeEnabled` 0 in all (On Duty 1 ✔) | Config | HR via UI | Open |
| V-0.12 | §4 | Code | No dead settings (follow.md #10) | `company_attendance_policies.lateThresholdMinutes` (= 30 everywhere, DB default) is **read nowhere** — only the entity declares it. Looks like the PDF's 30 minutes but does nothing | Minor | Remove column, or wire it — decide in Step 3 | Open |
| V-0.13 | §4 / §6 / §9 / §10 | Email | Alerts exist | Templates `LATE_ARRIVAL_WARNING`, `UNAUTHORISED_ABSENCE`, `APPROVAL_OVERDUE`, `TEAM_CONFIRMATION_REMINDER`, `ARREAR_RAISED / DECIDED`, regularisation, day request and shift swap — active in all 15 companies. Recipients tested in Steps 3–8 | OK | — | — |
| V-0.14 | — | Master data | — | Company **JIVA** is active with 0 employees, 0 employment types, 1 leave type | Minor | Confirm it is a test company | Open |
| V-0.15 | — | Housekeeping | No leftover clones | Database **HRMSCore_SrfTest** (left by an ATS session) still on the server | Minor | Drop after your OK | Open |
| V-1.1 | §2 Shift patterns | Shift master | General / Morning / Evening / Night exist | 55 active FIXED shifts, one **group pool** owned by LIFECARE and enabled in 13 companies through `shift_companies` (JIVA: none). By times: 9 general (e.g. G1 08:00–17:00), 24 morning, 13 evening, 9 night (`isNightShift`, e.g. N3 19:00–07:00). There is no "pattern" field; a pattern is just the code + times | OK | — | — |
| V-1.2 | §2 Weekly offs per entity / department | Clone: bulk-assign the 5-day work schedule (Sat + Sun off) to Legal (MAKL) | Saturday becomes a weekly off | Bulk commit says "1 assignment created", but Saturday stays a working G1 day. All 55 shifts carry their own 7 day rules (Sunday = WEEKLY_OFF), and the rule beats the work schedule. The schedule picked in bulk is saved in `employee_shift_assignments.workScheduleId`, which `fn_EmployeeDaySchedule` never reads. The company default schedule (Sat + Sun off) is never reached either. Weekly offs that work: the shift's day rules, the employee's weekday override, the roster ✔ | **Gap** | F4 (decided: remove the dead option, HR makes shift variants) | Open |
| V-1.3 | §2 Every employee has a shift | Read-only SQL, today | No one unscheduled | 7 active staff resolve to **no shift** (AFRIHOSP 1, MAKL 6). The company-default fallback only looks at shifts the company owns (`shifts.companyId`), so it works only for LIFECARE. `shift_companies` has no per-company default | Config + Minor | HR assigns a shift to the 7; code → F21 | Open |
| V-1.4 | §2 Roster upload | Clone: `docs/MAKL_Roster_Oct2026.xlsx` | Check → Save → draft; publish → used | 5 rows / 155 cells valid; saved as DRAFT; draft does not change the schedule; after publish the roster wins (e.g. 20001 weekly off Thu 8 Oct, source ROSTER) and past days are re-processed (4 Oct → WEEK_OFF) | OK | — | — |
| V-1.5 | §2 Lead time 5–7 days | Clone: lead time 7 (MAKL, LCH) | Publish-by shown, late flagged | 61 refused (400, clear message). November: publish by **25 Oct**. October published on 7 Oct → **13 days late**, allowed, shown in the Roster Publication report. A company with no rule records nothing. The PDF's range becomes one number (7 recommended). Warning only, never blocked; no reminder email before the date (§37) | OK (Minor: no reminder) | — | — |
| V-1.6 | §2 Swap colleague → manager | Clone: Rahab ⇄ Peter, 8 Oct, manager Bhavdip | Colleague accepts, then manager approves; roster updated | Preview names Bhavdip. Manager sees nothing until Peter accepts; then SW-BB1D0B46 in his inbox (`SHIFT_CHANGE`). Approve → roster days exchanged (Rahab G16, Peter weekly off, remark "Shift swap SW-…"), schedule follows | OK | — | — |
| V-1.7 | §2 Roster updated **before the shift** | Clone: same pair, **today** at 17:25, Peter's G16 09:00–14:00 already over | Refused | **Was approved.** Re-processing then marked Rahab **ABSENT** (she was off) and turned Peter's worked day into a weekly off. Only dates before today were refused | **Blocker** | Fixed in this session: `ShiftSwapRules.StartedRefusal` refuses today's day once either shift has started (company local time), at preview, submit, accept and approve. Re-test: today → "The shift on 07 Oct 2026 has already started (08:30)…", 400; tomorrow → allowed | **Fixed** |
| V-1.8 | — | Bulk assignment preview | Readable summary | "Filtered by: dpt-MAKL-007 · All locations" shows the raw id, not "Legal" (`ShiftService.cs:597`) | Minor | Small fix with F4 | Open |
| V-2.1 | §3 Biometric | Read-only SQL | Punches synced from the devices | Watermark sync runs (`AutoSyncEnabled` true in appsettings; last run 7 Oct). 35 322 logs since 2 Jul, 52 devices. **2 373 logs (6.7 %) match no employee**, 969 `UNMAPPED_EMPLOYEE` and 200 `INVALID_DEVICE` exceptions open. Clone: device punches 08:20 / 18:40 processed into the day ✔ | Config (HR maps member numbers) | HR | Open |
| V-2.2 | §3 Biometric | Read-only SQL + code | A punch goes to the person who punched | `SyncWithWatermarkAsync` looks the member number up in **all companies**, also after stripping "EMP" and leading zeros. MAKL `20001` and AFRICARE `EMP20001` both become "20001": the **10 punches from the MAKL device went to Wanjiku Muthoni (AFRICARE)**; the MAKL employee got none (→ absent) | **Blocker** | F5 | Open |
| V-2.3 | §3 Biometric | Code + data (follow.md #1, #10) | No fake fallbacks | Unreadable punch time → *now*; missing status → guessed by the hour (< 13 = in); device 1, branch "1" by default; unmapped exceptions filed under the **first employee's company**; AutoSync on / 30 min when the setting is missing. 9 devices belong to `comp-001`, which is not a company | Minor | F5 | Open |
| V-2.4 | §3 Clock-in access | Clone: Amina (LCH) | Only employees given access | No access → "Clock in / out is not enabled for you…" (400); HR grants → works. Local: **1 employee** has access (MAKL 20341) | OK (Config) | HR via UI | — |
| V-2.5 | §3 Proxy punching (K2) | Clone: Amina sends her manager's id to clock-in / clock-out | Refused / only herself | **Accepted** — Aditya clocked in and out by Amina, even with ENFORCE + photo on; the clock event did not say who sent it. The portal card sent the id too | **Blocker** | Fixed in this session: clock in / out always resolve the signed-in employee (`AttendanceService`), `employeeId` removed from `ClockInRequest` / `ClockOutRequest` and the frontend payloads. Re-test: same call → punch recorded on Amina, Aditya untouched | **Fixed** |
| V-2.6 | §3 Selfie / geofence | Clone: LCH ENFORCE, 150 m, ±50 m, photo at clock in; Arch Place given coordinates | Refused outside / without photo; WARN flags | No position → refused; 348 m → "You are 348 m from Arch Place…"; ±200 m → "not precise enough"; no photo → "Take a photo…"; fake JPEG → "could not be read"; inside + photo → accepted (INSIDE, photo kept). WARN at 348 m → accepted, OUTSIDE. Local: every company **OFF**, **0 locations** with map coordinates | OK (Config) | HR via UI + Org Masters → Locations | — |
| V-2.7 | §3 Phone (K3) | Clone: `loginWithSpecificMobile` on | First phone registered, others refused | Phone A registered; phone B → "Your account is registered to another phone…"; desktop allowed; HR reset → next phone registers ✔. Limits (known, §11): a phone browser in "desktop site" mode counts as a computer, and the device id is kept by the browser | OK (Minor: limits) | Selfie + geofence are the backstop | — |
| V-2.8 | §3 OD / WFH | Clone: Amina, manager Nafula, HR | Request → manager → HR → day marked | OD 5–6 Oct: manager inbox only → approve → still ABSENT → HR inbox → approve → **ON_DUTY, 540 min, not late**; with device punches 08:20–18:40 → ON_DUTY 640 min. Refusals: WFH switched off, weekly off only, overlap; a started request cannot be cancelled. Partial WFH 09:00–10:30 → ABSENT "below the half-day minimum" | OK | — | — |
| V-2.9 | §3 OD / WFH "with shift times" | Clone | Shift times shown on the day | Day credited 540 min, but `clockIn` / `clockOut` stay **empty** when there are no punches (only real punches are stored, §34) — registers may show no times | Minor | F21 | Open |
| V-2.10 | §3 Web clock | Clone: clock in at 21:02 (G1 08:00–17:00) | Status follows the rules | The record says **PRESENT** at once; daily processing later makes it ABSENT (late 753 min) ✔. A second clock in overwrites the first in-time and clears the clock-out on the record | Minor | F21 | Open |
| V-2.11 | §3 Web clock | Code | One clock card | HR Attendance dashboards have "Quick Clock" buttons (`AttendanceView`, `AttendanceDashboardView`) that send no photo / position (refused under ENFORCE / photo), show the static text "Biometric punch registered in database", and read `message` instead of `errors[0]` | Minor | F21 | Open |
| V-2.12 | §3 / §5 OD wording | User decision 2026-10-07 | "Out Duty" | Every screen, API message and email said "On Duty" | Minor | Fixed: labels → **Out Duty** in 13 frontend + 16 backend files; `db_changes.sql` block renames the permission and the 2 email template names (only if unchanged). Codes stay `ON_DUTY`. The day note "On duty (approved request)" from `sp_ProcessDailyAttendance` changes with the next edit of that procedure | **Fixed** (run db_changes on local / VM) |
| V-2.13 | — | Clone | Developer sees all on /task | `rakesh` (developer) saw the requests only at the HR stage, not at the manager stage | Minor | F21 (re-check) | Open |
| V-3.1 | §4 Grace | Clone MAKL: grace 30, G1 08:00; simulate 08:25 vs 08:40 | Within grace vs beyond | `fn_EvaluateAttendanceDay` sets lateCutoff = shift start + flexi window only; the late **band** uses the **company** `graceLateMinutes` (or a shift-policy override) via `fn_LatePenalty`/`fn_LatePenaltyTiered`. Simulate: effGrace 30, graceCutoff 08:30; 25 m → GRACE, 40 m → BEYOND_GRACE, both covered at #1. The shift's own `gracePeriodMinutes` (15, V-0.1) is **read nowhere in the late calc** — only an editable Shift-Master field. So **company grace wins; the shift column is cosmetic** | OK (resolves V-0.1) | — (shift column: fold into V-3.6 clean-up) | — |
| V-3.2 | §4 Tiers | Clone MAKL: tiers 1–3 NONE / 4–5 ALERT_MANAGER / 6+ DEDUCT LEAVE_THEN_LOP, beyond-grace-always-deducts OFF; simulate #3–#6 | 1–3 none · 4–5 alert · 6+ deduct | Simulate at 08:40 (40 m late): **#3** tier NONE covered · **#4** ALERT_MANAGER covered, no deduction · **#5** ALERT_MANAGER · **#6** DEDUCT, penalty `LEAVE_THEN_LOP`, not covered. Reads the saved `late_penalty_tiers` + company policy, not hardcoded | OK | — | — |
| V-3.3 | §4 Leave-then-LOP | Clone MAKL emp 10961, two LEAVE_THEN_LOP days, `sp_SyncLateLeaveCharges` in a rolled-back tran | Half-day from leave, LOP when short | Annual available **0.5** → oldest day (18 Sep) charged **LEAVE** (balance → 0), next day (19 Sep) **LOP**; available **0** → **both LOP**. Oldest-first spillover and zero-balance → LOP confirmed live. Not eligible / no balance row → LOP | OK | — | — |
| V-3.4 | §4 Early going | Clone MAKL: `earlyGoingMode` WITH_LATE; simulate 08:40 in + 16:40 out (full day, 20 m early) | Early going counted the same | Shared count through the same table: **#6** both late **and** early → DEDUCT `LEAVE_THEN_LOP`; **#4** both → ALERT_MANAGER. Early grace falls back to the shift's `earlyGraceMinutes` (10). A day with both carries **two** charges (LATE + EARLY) in `sp_SyncLateLeaveCharges`. Deduction half of "treated the same" works | OK | — | — |
| V-3.5 | §4 Early-going **alert** | Code: `sp_GetLateArrivalWarnings` + `NotificationService` | Early going treated the same as late → manager alerted at the warning tier | The **only** warning source filters `ar.lateTierAction = 'ALERT_MANAGER'`; it ignores `earlyTierAction`, and there is no early-going warning email. A day on the warning step because of an **early departure only** (not late) → **no manager alert**. The template body is late-centric (`{{LateOccurrence}}`, `{{LateMinutes}}`). So "early going treated the same as late" holds for the **deduction** but **not** the **manager alert** | **Gap** | F6 (proposed) | Open |
| V-3.6 | §4 | Code (follow.md #10) | No dead settings | `company_attendance_policies.lateThresholdMinutes` (V-0.12, = 30 everywhere) is declared only on the entity and **read by no service or procedure** — truly dead. `shifts.gracePeriodMinutes` is editable in Shift Master but unused in the late calc (V-3.1). Both look like "the PDF's 30 minutes" but do nothing | Minor (resolves V-0.12) | F7 (proposed: drop/retire the columns) | Open |
| V-4.1 | §5 Status codes in ESS | ESS calendar legend | P/A/HD/WO/PH/L/OD/WFH/LOP shown | The calendar renders the right day types (badge + 9-chip legend: Present, Late, Absent (LOP), Holiday, Leave, Leave Pending, Off, Out Duty, Work From Home) but **no short codes**. HD had no legend chip at all | Minor (user asked for the codes) | **Fixed** this session: legend chips now carry the policy code (P · Present, A / LOP · Absent (LOP), HD, WO, PH, L, OD, WFH) and an HD chip was added (`PortalAttendanceView.tsx`) | **Fixed** |
| V-4.2 | §5 Export | HR export CSV (detailed) | Readable codes | `ExportAttendanceCsvAsync` wrote the **raw enum** (`ABSENT`, `ON_DUTY`, `WEEK_OFF`) — not the "Out Duty" / "Absent (LOP)" labels or the policy codes | Minor | **Fixed** this session: detailed CSV now has a **Status Code** column (A / LOP, HD, P, WO, PH, L, OD, WFH) + the readable **Status** label; clone export → `A / LOP \| Absent (LOP)`, `WO \| Weekly Off`, etc. (`AttendanceService.cs`) | **Fixed** |
| V-4.3 | §5 A vs LOP | Design | A and LOP as day types | The PDF lists **A** and **LOP** as separate codes, but the system has one `ABSENT` status, labelled "Absent (LOP)" (the §34 decision: no new status; ABSENT is an unpaid scheduled working day). Both PDF codes map to ABSENT; shown as "A / LOP" | OK (by design) | — | — |
| V-4.4 | §6 3 consecutive days | Clone MAKL, `unauthorisedAbsenceAlertDays`=3, `sp_FlagUnauthorisedAbsence` | Alert at 3, days are A | A run of **3+** consecutive scheduled working days ABSENT without information → one **UNAUTHORISED_ABSENCE** HIGH item in the Exceptions Queue with the dates/count (20283 28–30 Sep → "3 consecutive working days absent…"); those days are status **ABSENT**; min flagged run = **3** (edge holds), 2-day runs exist but are not flagged. (The PDF's "ABSENCE_ESCALATION" is the event `UNAUTHORISED_ABSENCE` — queue item + email, recipients proven in §33) | OK | — | — |
| V-4.5 | §6 Unauthorised absence = A and LOP in payroll | Code trace `PayrollService` | A deducted as LOP | `PayrollService` counts `ABSENT` days → *"Unpaid Absence (N days)"* at the day rate, `HALF_DAY` → half the day rate, gated on the run's `IncludeAbsences` flag. No separate "unauthorised" distinction — **every ABSENT day is LOP**. Full payroll end-to-end (statutory setup) is Step 7 | OK | — | — |
| V-4.6 | §6 Sick > 2 days | Clone MAKL config + §28 | Certificate required, HR waiver | MAKL SICK_FULL / SICK_HALF: `requiresProof` 1, `proofThresholdDays` 2, `proofTiming` BEFORE_APPROVAL, `proofWaiverByHr` 0; permission `leave.proof.waive` present. Config intact on the clone; enforcement (submit allowed, **HR final refused** until the certificate, waiver needs the permission + reason) proven end-to-end in **Step 19 / §28** | OK (Config: HR turns on the waiver where wanted) | — | — |
| V-5.1 | §7 Window 3 working days | Clone LCH: window 3 (via `PUT api/config/attendance-policy`), Amina (G1, Sunday off), today Wed 7 Oct | Day 3 allowed, day 4 refused | 2 Oct (4 working days passed: 3, 5, 6, 7 Oct — Sunday skipped) → refused, "Too late to regularise 02 Oct 2026: the limit is 3 working day(s) after the date and 4 have passed." (400); 3 Oct (3 passed, 0 left) → allowed; 5 Oct → 1 left. Counted on the employee's own schedule (`fn_EmployeeDaySchedule`). HR raising is exempt (by design §35) | OK (Config: window blank in all 15, V-0.7) | HR via UI | — |
| V-5.2 | §7 Manager approves → HR verifies | Clone: Amina → manager `aditya` → HR `aisha.kamau` (Company HR) | Two stages, day changes only at the end | Submit → "to the reporting manager, then HR verification"; only the manager's inbox (PENDING_MANAGER); HR deciding stage 1 → "This approval is not assigned to you or your role."; manager approves → PENDING_HR in HR's inbox, day still ABSENT; HR approves → PRESENT with the requested times | OK | — | — |
| V-5.3 | §7 4th in a month → HR head | Clone: cap 3, over-quota role **Group HR**; 3 requests (3, 5, 6 Oct) then a 4th (7 Oct) | 4th goes to the HR head | Allowance: "Above the monthly limit of 3: this one needs approval from Group HR."; routed manager → **PENDING_ESCALATION** in Group HR's inbox only; Company HR deciding it → refused; Group HR (`puneet.singh`) approves → PRESENT. With no role set (today, all 15) the 4th is **refused**. The system has no "HR head" role — HR picks the role (Group HR / HR VP) | OK (Config: role blank, V-0.7) | HR via UI | — |
| V-5.4 | §7 Regularised day stays corrected | Clone: re-run `sp_ProcessDailyAttendance` for 3 Oct and 6 Oct after both were approved | Day stays PRESENT | **Both back to ABSENT** ("No biometric punch recorded"), times cleared; the request still says APPROVED. The procedure reads approved `attendance_adjustments` and day requests, never `regularisations`; the approval only wrote `clockIn` / `clockOut` (not `actualClockIn`, source stays SYSTEM). In live use the biometric sync re-processes the **whole company** for every date with a new punch (`BiometricService.cs:845`), and leave / holiday / roster changes re-process too — so a device-failure correction is lost as soon as the device's late logs arrive | **Blocker** | F8 | Open |
| V-5.5 | §7 / §9 After the cut-off | Clone (all companies use `PAYROLL_APPROVAL`): Amina's 1 Oct / 2 Oct marked `PayrollLocked` (as payroll approval does) | Paid day not changed; correction → next cycle | **Accepted and applied**: HR raised on a locked day, and a request pending when payroll locked was approved → days PRESENT and `lifecycleStatus` **PayrollLocked → Approved** (lock removed, so later re-processing could change the paid day too). A rejection also overwrote the lock ("Rejected"). The closed-period check works only in `PERIOD_END` mode | **Blocker** | **Fixed** this session (approved): `RegularisationRules.PaidDayRefusal` — the ESS allowance / submit and the final approval refuse a `PayrollLocked` day ("… is in a payroll that is already approved … Ask Payroll to raise an arrear for the next payroll (Payroll → Arrears)."); a rejection is allowed and leaves the day alone. Re-test: submit 30 Sep → 400 with that text; approve pending 29 Sep → refused; reject → REJECTED, day still ABSENT / PayrollLocked. Test `PayrollLockedDay_ApprovalRefused_RejectionLeavesTheDayAlone` | **Fixed** |
| V-5.6 | §7 After cut-off → **next cycle** | Design (§35 B7, §39) | Request after the cut-off is processed in the next cycle | No queue: a closed / paid day is refused; the correction is a **manual arrear** raised by Payroll and approved by another person (§39), paid in the next open payroll. Decision 2026-10-07 (user): **keep refuse + manual arrear**; the refusal message now points to Payroll → Arrears (V-5.5) | OK (by decision) | HR walkthrough explains it | — |
| V-5.7 | §7 Regularisation corrects punches, not lateness | Clone: 6 Oct requested 10:30–17:00 on G1 08:00; 5 Oct in 08:05 only | Late / half-day worked out from the corrected times | Final approval sets **PRESENT** directly: 10:30 → `lateMinutes` 0, not late; in-time only → PRESENT full day. Nothing re-evaluates the day (`RegularisationRules.ApplyToDay`) | **Gap** | F8 (same fix) | Open |
| V-5.8 | §7 Reasons: missed punch / device failure / on duty | Code (follow.md #1, #10) | Reasons from a master; PDF reasons | ESS drawer has **8 hardcoded options** (`PortalAttendanceView.tsx` ~1770) incl. "Traffic delay / Road obstruction" (a late excuse) and "On-Site Emergency Clinical Call"; HR drawer is free text. Out duty has its own request (§34). `ApplyToDay` still recognises old out-duty rows by the words "Out Duty" | Minor | F9 (decided: master table) | Open |
| V-5.9 | §7 | Clone, `PERIOD_END` mode, 30 Sep | Drawer tells the employee before submit | Submit → "The attendance period containing 2026-09-30 is closed." ✔, but the allowance does not check a closed period, so inside the window the drawer would say "can raise" and submit then fails (payroll-locked days are now reported, V-5.5) | Minor | F21 | Open |
| V-5.10 | §7 / §10 Emails | Clone + read-only SQL | Manager / HR told of waiting requests | `REGULARISATION_WAITING` / `_DECIDED` templates are active, but **no `notification_settings` row** turns them on in any company (none of the 4 regularisation / day-request events) → no email was logged for the 6 test requests. Inbox on /task works | Config | HR via Email Templates / notification settings at go-live | Open |
| V-5.11 | §7 Max 3 a month | Code | Count of requests | Every request in the attendance period counts, **rejected ones too** (§35, unchanged). Matches "3 requests a month" | OK | — | — |
| V-6.1 | §8 Overtime set-up | Read-only SQL, 15 companies | OT rates; overtime needs approval | OT1 working day ×1.50, OT2 weekly off + holiday ×2.00 in all 15 ✔. `requiresAuthorization` **0** on every type → all **10 898** overtime days are AUTO (paid without approval); no `overtime_limits` | Config | HR via UI (Attendance → Overtime types) | Open |
| V-6.2 | §8 Pre-approval by manager **and** HR (K1) | Clone LCH: OT1 + OT2 set to need approval; Amina punched Sun 4 Oct (weekly off) 09:00–15:00 and Mon 5 Oct 08:00–19:30 | Approved **before** the work, manager then HR | Overtime appears only **after** the work, as PENDING (OT2 360 min, OT1 150 min, payable 0). Manager and HR both see it on /task as "PENDING_MANAGER"; the manager approved 5 Oct alone → APPROVED, paid 150 (no HR stage; HR on /task afterwards → "not assigned"); HR approved 4 Oct straight from the Overtime page, **without the manager** | **Gap** | F1 (pre-approval + two stages) | Open |
| V-6.3 | §8 | Clone | Only approved overtime is paid | PENDING pays 0; HR authorised 240 of 360 min → payable 240 ✔ | OK | — | — |
| V-6.4 | §8 / §10 | Clone | HR sets the overtime rules | Company HR (`aisha.kamau`) → **403** on overtime type settings (`overtime.settings.manage`); developer allowed | Config | F20 | Open |
| V-6.5 | §8 Comp-off credited after manager + HR | Clone: manager `aditya` grants Amina 1 day for 4 Oct, HR `aisha.kamau` approves | Credited only after HR; with expiry | Grant → PENDING_HR, ESS still 0; the granter is refused (API 403, /task "not assigned"); HR approves → credited 7 Oct, **expires 6 Dec** (60 days), ESS 1.00. 1.5 days → "At most 1 day(s)…"; 3 Oct (no clock-in) → "There is no clock-in…". The day's approved overtime (240 min) → REJECTED "Compensated by comp-off", payable 0 (not paid twice) ✔ | OK | — | — |
| V-6.6 | §8 Manager grants | Clone: `puneet.singh` (Group HR, not Amina's manager) grants her 0.5 day | Only her manager grants | **Accepted.** Anyone with `compoff.grant` can grant to any employee in the workspace (`CompOffService.GrantAsync`) | Minor | F10 (proposed) | Open |
| V-6.7 | §8 Work on WO / PH | Clone: reason OFF_DAY_WORK for Tue 6 Oct (ordinary working day) | Refused, or flagged | **Accepted** (HR rejected it by hand). Nothing checks that the date is a weekly off / holiday | Minor | F10 (proposed) | Open |
| V-6.8 | §8 Overtime pay **vs** comp-off | Clone: 0.5-day grant for 5 Oct (150 min approved overtime) | Only the compensated part unpaid | The **whole day's** overtime → REJECTED, payable 150 → **0**, whatever the grant size | **Gap** | F10 (proposed) | Open |
| V-6.9 | §8 Expiry | Clone `fn_LeaveEntitlementAsOf` | Usable up to the expiry day | 6 Dec (expiry day) **1.50**, 7 Dec **0.00** ✔; the daily accrual job (`LeaveAccrual:AutoRunEnabled`) refreshes the balance | OK | — | — |
| V-6.10 | §8 Expiry across the year end | Clone, rolled back: credit moved to 15 Dec (expiry 13 Feb) | Valid for 60 days | 31 Dec **1.00**, 1 Jan **0.00** — credits count only in the year they were credited and COMP_OFF has carry-forward NONE, so a December credit loses most of its 60 days | **Gap** | F10 (proposed) | Open |
| V-6.11 | §8 Expiry enforced | Clone: Amina books comp-off leave for **15 Dec** (credits expire 6 Dec); manager + HR approve | Refused — no balance on that date | **Accepted and approved**: 15 Dec ON_LEAVE (paid); entitlement on 15 Dec 0, taken 1. The balance check uses today's balance, not the balance on the leave date. (2 Dec was refused only because 0.5 was left) | **Blocker** | F10 (proposed) | Open |
| V-6.12 | §8 | Code (follow.md #1, #10) | Reasons from a master | Grant reasons OFF_DAY_WORK / NIGHT_WORK / OTHER hardcoded (`CompOffService.cs:18`, same as V-5.8) | Minor | With F9 (reasons master) | Open |
| V-6.13 | §8 | Code | Company dates | Credit / expiry date and "work date today or earlier" use the **UTC** date (`CompOffDecision.cs:39`, `CompOffService.GrantAsync`), not company local time — off by a day between 00:00 and 03:00 in Kenya | Minor | F10 (proposed) | Open |
| V-6.14 | §8 / §10 | Code | Nobody approves their own overtime | `OvertimeAuthorization.DecideAsync` (Overtime page, `overtime.authorize`) has no "not your own" check; /task hides own items. Not tried on the clone | Minor | F1 | Open |
| V-6.15 | — Security (found in passing) | Code `PasswordHasher.cs:24` | Only the real password signs in | `VerifyPassword` accepts six fixed demo passwords (`Admin@123`, `EmpPassword!2026`, …) for **every** account, and the stored hash itself as a password — in every environment, not only Development | **Blocker** | F16 | Open |
| V-7.1 | §9 Leave overrides punches | Approved full-day Annual Leave on days with punches (Elian 24 Sep LATE, 26 Sep HALF_DAY; Bhavdip 28 Sep one punch), and a FIRST_HALF leave with arrival 09:00 (25 Sep) | Day = ON_LEAVE (L), no late / half-day flag, not counted as a late occurrence | All stayed **LATE / HALF_DAY** ("Missing Clock Out Punch"); 26 Sep counted as late #5 → ALERT_MANAGER; the half day is deducted on the payslip **and** the leave is taken from the balance. First-half leave + 09:00 → LATE 30 min (measured from 08:00, not the second half). `sp_ProcessDailyAttendance` uses `@CurIsLeave` only when there is **no punch** | **Blocker** | F11 | Open |
| V-7.2 | §9 LOP on payslip | October run, 60,000 basic, Elian 8 absent + 1 half day; Bhavdip (Sept) 18 absent | LOP deducted once | Deducted **twice**: `PayrollService` adds absent / half-day LOP to `customDeductions` **and** passes it as `UnpaidAbsenceDeduction`, which the engine already takes off gross (`PayrollCalculationEngine.cs:49`, all 6 country paths). Elian net **16,940.22** (correct ≈ 36,555); Bhavdip net **0** (correct ≈ 14,510). The payslip lines themselves are right; the totals are wrong | **Blocker** | F17 | Open |
| V-7.3 | §9 Lock on the cut-off | Cut-off 14 (period 15 Sep – 14 Oct). On 7 Oct: confirm all teams, approve October payroll; run November | Lock only after the period has ended and every team has confirmed | Confirmation **and** approval accepted on 7 Oct; lock covered 1,812 existing rows **15 Sep – 7 Oct** only — 8–14 Oct are never locked or paid. November (15 Oct – 14 Nov) **ran with no attendance at all** (full pay). No "period ended" check anywhere (`ApproveCycleAsync`, `TeamConfirmationService`) | **Blocker** | F12 | Open |
| V-7.4 | §9 Payroll on locked attendance | `POST payroll/run` for the approved (LOCKED) October | Refused: "approved, raise an arrear" | **Re-ran**: status LOCKED → COMPUTED, payslips rewritten (approvedAt kept), attendance still PayrollLocked; PAID arrears drop out of a re-run; an arrear on October then refused ("October 2026 is still open") while its days stay locked. `RunPayrollAsync` has no status check | **Blocker** | F17 | Open |
| V-7.5 | §9 Post-lock change → HR approval → arrear | Annual Leave applied and approved (both stages) for 16 Sep, a PayrollLocked ABSENT day | Refused with "raise an arrear", or an arrear raised | **Accepted**: 1 day taken from the balance, the day stays ABSENT (already LOP) — the employee loses the leave **and** the pay. Leave services never check `PayrollLocked` / closed periods (regularisation does, V-5.5) | **Gap** | F18 (decision: refuse recommended) | Open |
| V-7.6 | §9 | Confirmation gate, arrears, tiers, overtime | As built (§36, §39) | ✔ Approve refused naming 7 teams, then only "Trevor Mwamburi"; second confirmation refused; period label "15 Sep – 14 Oct 2026". Arrear: raiser self-approve refused, superadmin approves → target **2026-11**, November payslip "Arrears paid – 2026-10 (16 Sep), 1 day(s)" **2,307.69** (= 60,000 / 26). Real punches 08:45 (grace 30): #1–3 NONE, #4–5 ALERT_MANAGER, #6–8 DEDUCT LEAVE_THEN_LOP; zero balance → 1.5 days LOP → "Late Arrival Penalty — loss of pay (1.5 day(s), 3 late arrival(s))" **3,461.54**. Overtime line per type | OK | — | — |
| V-7.7 | §9 / §10 Manager confirms | Ranmeet confirms his own team (19 reports) | Allowed | **403**: his login now has the **Finance Manager** role (no `attendance.confirm.team`); "Give manager access" only moves employee-role logins. HR confirmed on his behalf | Minor (data) | F20 | Open |
| V-7.8 | §9 Late deductions | Late-LOP vs absence-LOP | Same tax treatment | Absence LOP reduces gross / taxable pay; late / early penalties are only after-tax deductions (`customDeductions`) | Minor | F17 (decision) | Open |
| V-7.9 | — Static values (follow.md #1, #10) | Statutory setup saved with an empty PAYE band list | Payroll refuses, or uses only configured bands | Engine silently uses hardcoded `GetDefaultKenyaBands()`, and `> 0 ? configured : 0.0275 / 300 / 0.015 / 20000` fallbacks (SHIF, AHL, pension cap) — the "still open" item of policies.md §26 | Minor | F19 | Open |
| V-7.10 | §10 Audit trail | Approve payroll as `rakesh` | Approver's name recorded | `approvedByName` = **"Executive Approver"** (`User.Identity.Name` is null; static fallback in `PayrollController`) — the id is kept, the name is not | Minor | F17 | Open |
| V-7.11 | §9 | Code read | PERIOD_END closes at the company's midnight | `fn_IsAttendanceDateClosed` compares with `SYSUTCDATETIME()` — Kenya (UTC+3) closes 3 h late | Minor | With F12 | Open |
| V-7.12 | — Tooling | `stepTestClone.sh status` | Shows the clone | "STOP: not on HRMSCore_StepTest" although the API was on the clone: the pid file held the wrapper subshell. **Fixed**: status checks the process listening on :5299 | Minor | Fixed (script) | ✅ Fixed |
| V-8.1 | §10 Approvals on /task | LCH: Amina (employee, manager set to Nafula on the clone) raises leave, punch correction, out duty. 25 decisions tried by the wrong person at each stage (employee on own items; HR / payroll at the manager stage; manager / payroll at the HR stage) | Only the owner of the stage decides | ✔ All 25 → **403** "not assigned to you or your role". Manager inbox: 3 items, *can act*; HR sees the leave read-only until stage 2; employee has no /task; payroll sees only the payroll run (and SRF budget step) for its company. Manager → HR (Aisha) → APPROVED for leave and punch correction; out duty HR stage by Group HR | OK | — | — |
| V-8.2 | §10 "no more than duties" | `GET api/attendance/daily` as **employee** (Amina, ESS) | Own record only | **2,141 rows** — the whole LCH register (names, punch times, late tier, penalties, notes); with `companyId=comp-makl-01` → **79 MAKL rows** (another company); `ALL` → HTTP 500. Same for payroll and line-manager logins (whole company, not the team). Cause: implication `portal.attendance.view → attendance.view` + no data-level / company-access check in `GetDailyAttendanceAsync` | **Blocker** (security) | F13 | Open |
| V-8.3 | §10 | `GET api/employees` as employee | Not available, or self | 2,141 LCH staff with phone, work email, reporting line (salary masked as 0). Confirms RBAC Step 7 finding 3 (`portal.dashboard.view → employees.view`) | **Gap** (privacy) | F13 | Open |
| V-8.4 | §10 HR Ops duties | Company HR (`aisha.kamau`, also `hr`, 2 more logins) does the HR Ops list | Exceptions queue, adjustments, close month, TA settings, shift / grace masters, out duty HR stage, switch on reminders, all-teams confirmation | Role **COMPANY_HR** lacks `attendance.manage`, `shifts.manage`, `shifts.policies.manage`, `attendance.dayrequest.hr`, `notifications.manage`, `audit.view` → all 403, while the sidebar shows Exceptions / Adjustments / Policies. Out duty at the HR stage is in **no** Company HR inbox (LCH: only HR Manager `insurance.hr` — password change pending — and Group HR). HR Manager / Group HR roles hold everything. Approval deadlines + reminders: ✔ (`approvals.sla.manage`) | **Gap** (role data) | F20 (decide which role is "HR Ops") | Open |
| V-8.5 | §10 "no more" | Line Manager login | Approve team requests, confirm team attendance | Also holds `shifts.manage` + `shifts.policies.manage` → can create shifts and **grace / late policies** for the company (empty POST reached validation); sidebar shows Register, Exceptions, Adjustments, Shift Policies and Payroll reports (company statutory totals) | **Gap** (role data) | F20 | Open |
| V-8.6 | §10 2 working days + reminders | Leaves submitted Wed 7, Mon 5, Fri 2 Oct (Amina works Saturdays; Sun = week off) | Due after 2 working days of her schedule; overdue the day after; reminder to the manager at stage 1, HR as escalation | ✔ Wed 7 → due **Fri 9**; Mon 5 → due **Wed 7, not overdue on the 7th** (edge); Fri 2 → due **Mon 5** (Sat counted) → **Overdue**. Counters 2 pending / 1 overdue. *Show overdue* (dry run, `APPROVAL_OVERDUE` on, To = test address): manager stage → To list **+ reporting manager**; HR stage → To list only. Payroll run item: 2 calendar days (as documented, §40) | OK | — | — |
| V-8.7 | §10 | Manager approves the overdue leave on 7 Oct | HR gets its own 2 working days | Leave HR stage counts **from submission** → arrives already overdue (due 5 Oct) and HR is reminded for the manager's delay. `leave_decisions.decidedAt` now has the stage time (punch correction / out duty already count from the manager's decision) | Minor | F15 | Open |
| V-8.8 | §11 Audit trail for every change | HR changes a deadline 2 → 3; superadmin changes grace 15 → 30 | Who / when / old → new recorded | **Nothing recorded**: no `audit_log` row, no "changed by" column (`approval_sla_settings`, `company_attendance_policies`). Code read: TA settings, notification settings, role permission grants (`AdminService` only reads the log), payroll run / approve and team confirmation are not audited either; the whole DB has 10 audit rows. Audit viewer only for admin / superadmin / developer. **Is** traced: leave per stage (`leave_decisions`), punch correction and out duty (manager + HR name, time, remarks), adjustments (adjusted / decided by), shift / grace policies, overtime, arrears, roster, holidays, leave types, clock access, employee edits | **Gap** | F14 | Open |
| V-8.9 | §10 / D4 Approvals on /task only | Payroll arrear, attendance adjustment | In the /task inbox | Not in /task: arrears are approved on Payroll → Arrears, adjustments on Attendance → Adjustments (`ApprovalsService` has neither) | **Gap** | F15 | Open |
| V-8.10 | — Static values (follow.md #4, #10) | Payroll item on /task | Country-neutral | Summary hardcodes Kenya labels "PAYE / SHIF / AHL" for every company; amount grouped by server culture ("23,27,180"). Currency from the company ✔; the invented values flagged in §40 are gone ✔ | Minor | F19 | Open |
| V-8.11 | §9 / §10 | Manager confirms with an empty body | Period must be chosen / ended | Defaults to the current, unfinished period (15 Sep – 14 Oct) and confirms it on 7 Oct — same root as V-7.3 | Minor | F12 | Open |
| V-8.12 | §11 / §12 | K4 / K5 | — | §11 alerts exist (late warning Step 3, absence escalation Step 4); no disciplinary case record → **F2** (K4, unchanged). §12 annual review: **out of scope** (K5) | OK (decided) | F2 | — |
| V-8.13 | — Tooling | `stepTestClone.sh status` after `up` | Shows the clone API | "Test API not running": the pid file's wrapper had exited. **Fixed**: `status`, `api` and `down` use the process listening on :5299 (down now really stops it) | Minor | Fixed (script) | ✅ Fixed |
| F3 | §6 | Master data (V-0.6) | Sick / annual leave in every company | **9 companies had only COMP_OFF** (AGI, BLISS 1 384 staff, DINLAS, JIVA, LCF, NEWAGE, SDI, SDI_HOLD, SDLI) → leave-then-LOP always fell to LOP, sick-certificate rule unusable. Decision 2026-10-07 (user): **seed from a template**. `db_changes.sql` block "Step 4 — F3" copies the 8-type standard set from **LIFECARE** into every active company with no sick/annual (skips codes it already has, e.g. COMP_OFF); HR adjusts days/rules after | **Gap → Fixed** | Seed script (idempotent); **run on local / VM** | **Fixed** (clone: 9 → full 8 types each; 2nd run inserted nothing) |

## 6. Completion notes
_(one subsection per step: date, what was checked, clone used / dropped, findings raised, fixes made)_

### Step 0 — Baseline (2026-10-07)
- **Read-only SQL** on HRMSCore_Local (15 active companies): `company_attendance_policies`, `late_penalty_tiers`,
  `approval_sla_settings`, `shifts`, `shift_policies` (empty), `leave_types`, `employment_types`, `email_templates`;
  plus `appsettings.json` Notifications. **All 15 companies carry identical settings**: every Phase 3 rule is at its
  "today's behaviour" default, i.e. switched off. Nothing in the policy is wrong in code so far — it is not configured.
- **Decisions:** K1 build → F1 · K2 rely on controls · K3 phone browser OK · K4 build → F2 · K5 out of scope ·
  P1 all employment types · values entered by HR through the UI.
- **Clone tooling:** `docs/scripts/stepTestClone.sh` (`up | ui | status | down`) + `docs/scripts/stepTestClone.md`.
  Verified: up in ~16 s, API :5299 on HRMSCore_StepTest, test UI :3000 → :5299, down leaves nothing; :5197 / :4000
  untouched. Supporting: `frontend/next.config.ts` (`distDir` from `NEXT_DIST_DIR`), `frontend/tsconfig.json` include,
  `.gitignore`. `tsc --noEmit` ✔.
- **Artifact:** "Policy verification" section added (tracker + Step 0 manual checklist).
- Clone created twice and dropped; no `.bak` left.

### Step 1 — Shifts, weekly offs, roster, swaps (2026-10-07)
- PDF text could not be extracted (font-encoded, no PDF tools installed); the clause was taken from the Step 1 prompt
  (§4). Read policies.md §12, §37, §38, §48 only.
- **Read-only SQL** on HRMSCore_Local: shift pool and patterns, day rules (all 55 shifts: 7 rules, Sunday off), work
  schedules, assignments, rosters (none yet), swaps (none), `rosterPublishLeadDays` (blank everywhere, V-0.9), and the
  schedule source per active employee today (7 with no shift, V-1.3).
- **Code trace:** `fn_EmployeeDaySchedule` precedence date override → published roster → weekday override →
  assignment → employment → company default; `RosterLeadTimeRules` reads the company setting (no hardcoded value);
  `ShiftSwapRules.DateRefusal` only refused dates before today (V-1.7).
- **Clone** (up → api → down): lead time 7 set through `PUT api/config/attendance-policy` on LCH and MAKL; MAKL October
  sheet uploaded and published; 4 test logins created **on the clone only** (st.rahab, st.peter, st.trevor = role_emp;
  st.bhavdip = role_mgr, all with the demo employee password); swap flows; bulk department schedule. Clone dropped,
  `.bak` removed, :5299 free, :5197 untouched.
- **Fix (approved):** V-1.7 in `backend/Infrastructure/Services/ShiftSwapRules.cs` (+ test
  `ShiftSwapTests.ShiftAlreadyStartedToday_Refused`). `dotnet build` ✔, `dotnet test` **305/305** ✔. No DB or frontend
  change.
- **Decision:** V-1.2 → F4 (remove the dead work-schedule option; weekly offs from shift day rules / weekday override /
  roster).
- **Tooling:** `docs/scripts/stepTestClone.sh api` restarts the test API on new code and keeps the clone.
- **Not run:** browser tests (no browser in this session) → Step 1 checklist in the artifact. Step 0 manual results
  were not supplied.

### Step 2 — Marking attendance (2026-10-07)
- Read policies.md §11, §25, §34, §45 only. PDF clause from the §4 prompt.
- **Read-only SQL** on HRMSCore_Local: clock / phone / geofence / photo / OD / WFH settings (identical in all 15
  companies: all controls OFF, OD on, WFH off, manager → HR), clock-in access (1), registered phones (0), locations with
  coordinates (0), biometric logs / devices / watermark / exceptions, employee-number collisions across companies (1).
- **Code trace:** `BiometricService.SyncWithWatermarkAsync` (V-2.2, V-2.3), `AttendanceService.ClockIn/OutAsync` +
  `ResolveTargetEmployeeIdAsync` (V-2.5), `ClockEvidenceService`, login device check, `DayRequestRules` /
  `sp_ProcessDailyAttendance`.
- **Clone** (up → api → down): LCH set through `PUT api/config/attendance-policy` (ENFORCE / ±50 m / photo at clock in /
  registered phone / WFH on, then WARN); Arch Place and Aditya's location given coordinates; Amina's manager set to
  Nafula **on the clone only**; scenarios V-2.4 – V-2.8, V-2.10.
- **Fixes (approved):** V-2.5 proxy punching (`AttendanceDtos.cs`, `AttendanceService.cs`, `attendance/types.ts`,
  `ClockInOutCard.tsx`); V-2.12 "Out Duty" wording + `db_changes.sql` block (run twice on the clone: 31 rows, then 0;
  **not yet run on HRMSCore_Local**). `dotnet build` ✔, `dotnet test` **305/305** ✔, `tsc --noEmit` ✔.
- **Decision:** V-2.2 / V-2.3 → **F5**.
- **Not run:** browser tests (camera, location and phone size need a real device) → Step 2 checklist in the artifact.

### Step 3 — Grace, late tiers, manager alert, leave-then-LOP, early going (2026-10-07)
- Read policies.md §6 (Step 1 grace/late engine), §29 (Step 20 tiers / warning / leave-then-LOP), §30 (Step 21 early
  going) only. PDF clause from the §4 prompt.
- **Code trace** (`docs/dbscript/tables_proc_all.sql`, read-only): `fn_EvaluateAttendanceDay` → `fn_LatePenaltyTiered`
  (reads `late_penalty_tiers`, company `graceLateMinutes` / `graceLateDaysPerMonth` / `lateBeyondGraceAlwaysDeducts`),
  `sp_SimulateAttendancePolicy`, `sp_SyncLateLeaveCharges` (leave-vs-LOP, oldest first), `sp_GetLateArrivalWarnings`
  (manager recipient via `employment_details.reportingManagerId`). All config-driven, no hardcoded tier/grace values.
- **Clone** (up → api → down, :5299, AutoSync off): MAKL policy set via `PUT /api/config/attendance-policy` — grace 30,
  3/month, beyond-grace-always-deducts **off**, tiers 1–3 NONE / 4–5 ALERT_MANAGER / 6+ DEDUCT `LEAVE_THEN_LOP`, Annual
  Leave as the penalty type, 0.5 day, `earlyGoingMode` **WITH_LATE**. Shift G1 (`shf-g1`, 08:00–17:00).
  - **Simulate** (`POST /api/attendance/settings/simulate`): effGrace 30, graceCutoff 08:30; 08:25 → GRACE, 08:40 →
    BEYOND_GRACE (V-3.1). Occurrence boundaries at 08:40: #3 NONE · #4 ALERT_MANAGER (no deduction) · #5 ALERT_MANAGER ·
    #6 DEDUCT `LEAVE_THEN_LOP` (V-3.2). Late+early same day (08:40 in / 16:40 out, full day): #4 both ALERT_MANAGER, #6
    both DEDUCT `LEAVE_THEN_LOP` (V-3.4).
  - **Rolled-back SQL** (`sp_SyncLateLeaveCharges`, emp 10961, two LEAVE_THEN_LOP days): Annual available 0.5 → oldest day
    LEAVE, next LOP; available 0 → both LOP (V-3.3). No data changed.
- **Findings:** V-3.1 OK (grace: company wins, shift `gracePeriodMinutes` cosmetic — resolves V-0.1); V-3.2, V-3.3,
  V-3.4 OK; **V-3.5 Gap** — early-going warning never reaches the manager (`sp_GetLateArrivalWarnings` filters
  `lateTierAction` only) → **F6**; **V-3.6 Minor** — `lateThresholdMinutes` + `shifts.gracePeriodMinutes` dead
  (resolves V-0.12) → **F7**.
- **Config findings resolved by setting values** (not code defects): V-0.2, V-0.3, V-0.4 — HR enters grace 30 / tiers /
  Annual-leave penalty type / early-going on the policy screen at go-live (Step 9).
- **No code changed this session** (follow.md #9): F6 and F7 need approval before coding.
- **Not run:** browser tests (no browser in this session) → Step 3 checklist in the artifact.

### Step 4 — Status codes + absence (A = LOP, 3 consecutive days, sick > 2 days) (2026-10-07)
- Read policies.md §28 (sick certificate), §33 (consecutive absence), §34 (OD/WFH + "Absent (LOP)"), §46 (ESS calendar)
  only. PDF clause from the §4 prompt.
- **Code trace:** status labels/codes in `attendance/status.tsx`, `PortalAttendanceView.tsx` (ESS calendar legend),
  `StaffSchedulesGrid.tsx` (already OD/WFH), `AttendanceView.tsx` (HR register badge), `AttendanceService.ExportAttendanceCsvAsync`
  (detailed CSV), `reports/attendance/AttendanceReportsView.tsx` (absenteeism **rates**, not a code grid);
  `PayrollService` (ABSENT → "Unpaid Absence" at day rate, HALF_DAY at half, `IncludeAbsences`); `sp_FlagUnauthorisedAbsence`
  / `fn_UnauthorisedAbsenceRuns` (event `UNAUTHORISED_ABSENCE`, not "ABSENCE_ESCALATION"). No hardcoded status values.
- **Read-only SQL** on HRMSCore_Local: leave-type coverage per active company → confirmed **V-0.6 / F3**: 9 companies have
  only COMP_OFF (AGI, BLISS 1 384, DINLAS, JIVA, LCF, NEWAGE, SDI, SDI_HOLD, SDLI); 6 have the full set; `duplicate-masters`
  copies departments/titles/grades/cost-centres **but not leave types** (`CompanyService.cs`), so no bulk path existed.
- **Clone** (up → api → down, :5299, AutoSync off):
  - **Export** (`GET api/attendance/export?format=detailed`, MAKL Sep): before → raw enum (ABSENT / HALF_DAY / WEEK_OFF),
    V-4.2; after the fix → `Status Code` + `Status` columns (`A / LOP \| Absent (LOP)`, `HD \| Half-Day`, `P \| Present`,
    `P \| Late Arrival`, `WO \| Weekly Off`).
  - **Unauthorised absence** (`unauthorisedAbsenceAlertDays`=3 via `PUT api/config/attendance-policy`; `sp_FlagUnauthorisedAbsence`
    MAKL): 85 runs flagged; 20283 28–30 Sep = "3 consecutive working days absent…", those days **ABSENT**; min flagged
    run = **3**, 2-day runs not flagged (V-4.4). A → LOP in payroll is code-traced (V-4.5; full run is Step 7).
  - **Sick > 2 days**: MAKL config intact (requiresProof, threshold 2, BEFORE_APPROVAL, waiver off; `leave.proof.waive`
    present) — enforcement proven in Step 19 / §28 (V-4.6).
  - **F3 seed**: applied the `db_changes.sql` "Step 4 — F3" block on the clone → the 9 companies went from COMP_OFF only to
    the full **8-type** set (2 sick, 1 annual, …); **second run inserted nothing** (idempotent).
- **Fixes (user-approved):** V-4.1 ESS calendar legend codes + HD chip (`PortalAttendanceView.tsx`); V-4.2 export Status
  Code + label (`AttendanceService.cs`); **F3** seed-from-template block (`db_changes.sql`, **run on local / VM**).
  `dotnet build` ✔, `dotnet test` **305/305** ✔, `tsc --noEmit` ✔.
- **Decision:** F3 → seed from a template (user, 2026-10-07). `sno` is an IDENTITY column, so the seed does not insert it.
- **Design note:** A and LOP are one `ABSENT` status labelled "Absent (LOP)" (§34), shown as "A / LOP" (V-4.3).
- **Not run:** browser tests (no browser in this session) → Step 4 checklist in the artifact.


### Step 5 — Regularisation (3 working days, manager → HR, 4th → HR head, after cut-off) (2026-10-07)
- Read policies.md §35 (Step 24), §39 (Step 28 arrears), §45 (Step 33 emails) only. PDF clause from the §4 prompt.
- **Read-only SQL** on HRMSCore_Local: all 15 companies window NULL, cap 3, over-quota role NULL, `PAYROLL_APPROVAL`,
  cut-off NULL (V-0.7 / V-0.8); 4 regularisations ever (all APPROVED, 1 of 1); no `PayrollLocked` days yet; no
  `notification_settings` row for the regularisation / day-request emails (V-5.10).
- **Code trace:** `AttendanceService.SubmitRegularisationAsync` / `BuildRegularisationAllowanceAsync`, `RegularisationRules`
  (stages, window via `fn_EmployeeDaySchedule`, quota per period, `ApplyToDay`), `ApprovalsService` → `ApprovalEngine`
  (`REGULARISATION`), `AttendancePeriod` / `fn_IsAttendanceDateClosed` (closed only in `PERIOD_END`), `sp_ProcessDailyAttendance`
  (reads adjustments and day requests, not regularisations), `BiometricService` sync re-process. No hardcoded window / cap.
- **Clone** (up → api → down, :5299, AutoSync off): LCH set through `PUT api/config/attendance-policy` — window **3**, cap
  **3**, over-quota role **Group HR**. Clone-only logins: `aditya` (Amina's manager), `aisha.kamau` (Company HR),
  `puneet.singh` (Group HR) given the demo employee password. Scenarios V-5.1 – V-5.5, V-5.7, V-5.9; `PERIOD_END` tried
  and set back.
- **Fix (approved):** V-5.5 — `RegularisationRules.PaidDayRefusal` used by the allowance / submit and the final approval;
  `ApplyToDay` leaves a paid day alone (`RegularisationRules.cs`, `AttendanceService.cs`, test in `RegularisationTests.cs`).
  `dotnet build` ✔, `dotnet test` **306/306** ✔. No DB or frontend change.
- **Decisions (user, 2026-10-07):** V-5.4 / V-5.7 → **F8**; V-5.6 keep refuse + manual arrear; V-5.8 → **F9** (reasons
  master).
- **Not run:** browser tests (no browser in this session) → Step 5 checklist in the artifact.

### Step 6 — Overtime and comp-off (pre-approval, manager + HR, credit with expiry) (2026-10-07)
- Read policies.md §10, §22, §31, §32 only. PDF clause from the §4 prompt.
- **Read-only SQL** on HRMSCore_Local: OT1 ×1.50 / OT2 ×2.00 in all 15, no type needs approval (10 898 days AUTO), no
  limits; COMP_OFF in every company (60-day expiry, max 1 per grant, clock-in needed, overtime not paid, carry-forward NONE).
- **Code trace:** `OvertimeService` / `OvertimeAuthorization` (one decision, after the work), `ApprovalsService` 2b (manager
  or HR on /task), `CompOffService.GrantAsync`, `CompOffDecision` (credit + expiry + overtime → REJECTED +
  `sp_SyncAttendanceOvertime`), `fn_LeaveEntitlementAsOf` (CREDITS: same year only, lapse after `expiresOn`),
  `sp_RunLeaveAccrual` (daily refresh), `MyExtraTimeService` (ESS).
- **Clone** (up → api → down, :5299, AutoSync off). The first `up` failed when the Mac's disk filled; Docker had to be
  restarted. Cause: `HRMSCore_Local_log.ldf` is **5.9 GB** for 276 MB of data, and every restore copies it. A clone left
  in RESTORING also makes `up` fail (`SET SINGLE_USER` is not allowed). The restore was finished by hand.
  LCH: OT1 / OT2 set to need approval (developer — Company HR gets 403); test punches for Amina 4–6 Oct in
  `biometric_punch_logs` + `sp_ProcessDailyAttendance`. Cast: Amina (`employee`), manager `aditya`, Company HR
  `aisha.kamau`, Group HR `puneet.singh`. Scenarios V-6.2 – V-6.11; year end in a rolled-back transaction.
- **No code changed.** Decisions needed: F10 plan; V-6.15 security fix; shrink `HRMSCore_Local`'s log (local only).
- **Not run:** browser tests (no browser in this session) → Step 6 checklist in the artifact; `GET me/overtime` not
  exercised on the clone (in the checklist).

### Step 7 — Leave and payroll (leave overrides, confirmation → lock, payroll, arrears) (2026-10-07)
- Read policies.md §26, §36, §39, §44 only. PDF clause from the §4 prompt. **Payroll cycle 15th–14th** (user): set as
  attendance policy **cut-off day 14** (Attendance Policies → General settings); `fn_AttendancePeriod` then gives
  15 Sep – 14 Oct for the **October** payroll (the month the period ends in). Already configurable per company; no code needed.
- **Code trace:** `fn_AttendancePeriod` / `fn_IsAttendanceDateClosed` (UTC), `AttendancePeriod.ForMonthAsync`,
  `sp_ProcessDailyAttendance` (leave only when no punch; never touches `PayrollLocked` rows), `PayrollService.RunPayrollAsync`
  (no status / period-ended check; absent + half-day LOP into `customDeductions` and `UnpaidAbsenceDeduction`),
  `PayrollCalculationEngine` (gross minus unpaid; hardcoded Kenya bands fallback), `ApproveCycleAsync` (confirmation +
  arrears gates, locks existing rows), `TeamConfirmationRules`, `PayrollArrearRules` / `PayrollArrearService`. Leave
  services never check the lock.
- **Clone** (up → down, :5299, AutoSync off), MAKL, set through the API: cut-off **14**, `PAYROLL_APPROVAL`, confirmation
  **required**, grace 30 × 3, tiers 1–3 NONE / 4–5 ALERT / 6+ DEDUCT LEAVE_THEN_LOP (Annual, 0.5); KE statutory setup via
  `PUT config/statutory-rates`; payroll "no basic salary" = **60,000** (nobody in MAKL has a salary). Clone-only data:
  4 approved leave rows on punched days, 14 late punches for Bhavdip 1–9 Sep, his Annual balance set to 0. Period
  re-processed. Cast: `rakesh` (HR / payroll), `ranmeet` (manager), `superadmin` (second approver).
- **Results:** V-7.1 – V-7.12 in §5. Blockers: leave does not override punches (V-7.1), LOP deducted twice (V-7.2),
  approval / confirmation before the period ends (V-7.3), a locked payroll can be re-run (V-7.4). Gap: leave approved on
  a locked day (V-7.5). Works: confirmation gate, arrears end-to-end, late tiers → late-LOP line, overtime line (V-7.6).
- **Changed:** `docs/scripts/stepTestClone.sh` status check only (V-7.12). **No app code changed** (follow.md #9).
- **Decisions needed:** fix V-7.2 and V-7.4 now (each one file, no DB change); F11 / F12 plans; V-7.5 refuse vs
  auto-arrear; V-7.8 tax treatment of late penalties; V-7.10 approver name.
- **Not run:** browser tests (no browser in this session) → Step 7 checklist in the artifact.

### Step 8 — Roles, 2 working days, audit trail, K4 / K5 (2026-10-07)
- Read policies.md §40, §44 and rbac_fixes.md (§0 decisions, §3 design, Step 7 re-test) only. PDF clause from the §4
  prompt and the §27 gap map (§10 duties per role; §11 audit trail and disciplinary action; §12 annual review).
- **Code trace:** `TaskController` (no permission policy on pending / decide; access in the service) →
  `ApprovalsService.EnsureCanActAsync` (= the inbox's *can act*) → `ApprovalVisibilityRules` (assignee / role / team;
  payroll = `payroll.approve`); `ResolveApproverContextAsync` decides "HR or admin" from hardcoded role codes **and**
  `roleLevel` (FINANCE / FINANCE_MANAGER are `COMPANY` level, so count as HR there — not exploitable in the tests, all
  refused). `ApprovalSla` (working days on the employee's schedule; leave HR stage from submission),
  `NotificationService.CheckApprovalRemindersAsync`. `audit_log` writers: 18 services, none of them settings / RBAC /
  payroll run.
- **Clone** (up → down, :5299, AutoSync off), LCH. Policy values set first: approval deadlines 2 working days (already
  seeded) and email `APPROVAL_OVERDUE` switched on through the API (To = `hr-test@example.invalid`, line manager ticked);
  reminders only as **dry run** (no mail sent). Clone-only data: Amina's manager → Nafula (`manager`); Grace Wambui's
  temporary-password flag cleared; 2 leave rows back-dated (2 and 5 Oct) for the deadline edge. Cast: `employee`,
  `manager`, `aisha.kamau` (Company HR), `puneet.singh` (Group HR), `finance` (Finance Manager), `grace.wambui`
  (Finance), `ranmeet` (MAKL payroll approver), `superadmin`.
- **Duty matrix (API, 16 duty endpoints × 5 logins):** employee — own attendance, leave, punch correction, out duty only;
  manager — team inbox + team confirmation, **plus** shift / grace masters (V-8.5); Company HR — deadlines and reminders,
  but **none** of the attendance HR Ops endpoints (V-8.4); payroll — payroll view / run / approve only, no attendance
  or approvals. Everyone with a portal login reads the whole daily register, other companies included (V-8.2).
- **Results:** V-8.1 – V-8.13 in §5. Works: stage ownership on /task (25 wrong-person decisions refused), deadlines and
  the overdue edge, reminders to manager / HR, request-level trail. Blocker: V-8.2 (security). Gaps: V-8.3 directory,
  V-8.4 HR Ops role, V-8.5 manager extras, V-8.8 audit of settings / access, V-8.9 arrears and adjustments outside /task.
- **Changed:** `docs/scripts/stepTestClone.sh` (`status` / `api` / `down` use the :5299 listener, V-8.13). **No app code
  changed** (follow.md #9).
- **Decisions needed:** F13 (security — first), F14, F15 plans; which role is "HR Ops" and the manager role clean-up
  (V-8.4 / V-8.5 are role data, done in Roles & Permissions — or a `db_changes.sql` block if you prefer).
- **Not run:** browser tests (no browser in this session) → Step 8 checklist in the artifact.

### Step 9 — Close-out: findings → fix steps, HR walkthrough, sign-off (2026-10-07)
- Read §1–§3, Progress, §5, §6 and follow.md; policies.md §55 only. No clone was started: Step 9 has no write
  scenarios, and every check was read-only on HRMSCore_Local (the V-0.x values — all still unset, F3 not yet run, SrfTest
  still there) or a code read (V-6.15 `PasswordHasher.cs:24`, V-7.2 `PayrollService.cs:629` + `PayrollCalculationEngine.cs:49`,
  V-7.4 `PayrollService.RunPayrollAsync`, V-7.10 `PayrollController.cs:82` — all still present).
- **Summary:** §5.0 — 9 Blockers open (3 fixed), 13 Gaps, 26 Minor, 14 Config, 29 OK.
- **New fix steps:** F16 password backdoor, F17 payroll totals, F18 leave on a paid day, F19 static-value sweep,
  F20 role clean-up, F21 attendance small fixes. Every open finding now names a step or "your decision" (V-0.14, V-0.15).
  Order proposed in four waves (§5.0). Prompts for F6, F7, F10, F16–F21 added to §4.
- **Artifact:** Step 9 summary, F16–F21 rows and the **HR walkthrough and sign-off checklist** (Part A policy values on
  live, Part B role walkthrough after Wave 1–2, Part C go-live server / data, Part D sign-off per clause).
- **Changed:** docs only. **No app code changed** (follow.md #9).
- **Decisions needed:** approve the F-step list and order; F18 refuse vs auto-arrear; F20 which role is HR Ops; V-7.8 tax
  treatment (in F17); V-0.14 JIVA; V-0.15 drop `HRMSCore_SrfTest`.
