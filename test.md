# Staff Attendance Policy — gap analysis and work plan

**Source document:** `docs/artifacts/Staff_Attendance_Policy_HRMS.docx` (Staff Attendance Policy (HRMS), v1.0)
**Compared against:** the system as built through Policies Steps 1–18 + §26 clean-up (see `docs/policies.md`)
**Checked:** 2026-10-03. Every line below was verified in code / SQL, not inferred from docs.

> The earlier plan that lived in this file (Employee Assignment & Document Repository) moved to
> `docs/archive/todo_assignment_documents.md`. Its backend endpoint is built; its four upload drawers are not.

**Rules this plan follows** (`docs/follow.md`): everything DB-driven and configurable per company — no static numbers;
`space-y-6` pages; 50 % `SideDrawer` forms; permissions and menus in the DB; every DB change into `docs/db_changes.sql`;
`dotnet build` and `npx tsc --noEmit` clean before any step is called done.

---

## Part A — Matching: policy lines already delivered

These need no work. 19 of the policy's rules are in place.

| # | Policy line | Where it lives today |
|---|---|---|
| A1 | §2 Standard hours, shift patterns, weekly offs defined per entity / department / roster, configured in the shift master | Shift Master `/shifts/master`, work schedules, weekly offs; a shift can be restricted to departments, locations, job titles, grades, employment types or cost centres (Step 17) |
| A2 | §2 Shift rosters uploaded to the HRMS | Roster planner `/shifts/roster` (DRAFT → PUBLISHED → LOCKED) + the General shift and Roster uploads (Step 6) |
| A3 | §3 Staff mark attendance daily at the start and end of each shift | Web clock in / out (portal card) and biometric terminal sync |
| A4 | §3 Attendance recorded only through the approved method | Clock-in access granted per employee (Step 4b); optional selfie + location check (Step 18); sign-in tied to a registered phone (Step 4b) |
| A5 | §4 Grace period of N minutes, up to N times per month | `graceLateMinutes` + `graceLateDaysPerMonth`, per company and overridable per shift policy |
| A6 | §4 Beyond the grace limit the day is flagged Late | `isLate`, `LATE` status, `lateSequenceInMonth` counted per attendance period |
| A7 | §5 Status codes P, A, HD, WO, PH, L | `PRESENT`, `ABSENT`, `HALF_DAY`, `WEEK_OFF` / `REST_DAY`, `HOLIDAY`, `ON_LEAVE` |
| A8 | §6 Unauthorised absence marked A and treated as loss of pay | `ABSENT` + unpaid absence deduction in payroll |
| A9 | §6 Absence of N or more days / sickness rules configurable per leave type | Leave Master: notice days, minimum service, min / max application days, gap between applications, negative balance |
| A10 | §7 Missed punches, device failures and on-duty cases raised through the HRMS | Regularisation request from the portal → approvals inbox |
| A11 | §7 Maximum N regularizations per month | `maxRegularisationsPerMonth`, counted per attendance period |
| A12 | §7 Requests after the payroll cut-off are not processed in this cycle | Closed-period refusal (`fn_IsAttendanceDateClosed`, `closeMonthMode`, Steps 4 / 4b) |
| A13 | §8 Overtime and work on a weekly off or public holiday identified separately | Overtime types OT1 (working day) and OT2 (weekly off + holiday), each with its own rate (Step 5) |
| A14 | §8 Overtime authorised before it is paid; limits per employee type | `requiresAuthorization` per overtime type + `overtime_limits` per period and per week |
| A15 | §8 Compensatory off credited in the HRMS after approval | Comp-off: manager grants → company HR approves on `/task` → credited with expiry (Step 15) |
| A16 | §9 Approved leave overrides attendance flags for that day | Leave overlay in `fn_EmployeeDaySchedule` and daily processing |
| A17 | §9 Attendance locked on the payroll cut-off date | `cutOffDay` + `closeMonthMode` (payroll approval, or end of the attendance period) |
| A18 | §9 LOP days, overtime and late-coming deductions flow automatically to payroll | Payroll reads the attendance period: unpaid absence, overtime per type at its own rate, late penalty |
| A19 | §10 / §11 Employee raises requests, HR maintains masters, audits exceptions and locks; falsification traceable | Portal, masters, Exceptions Queue `/attendance/exceptions`, close month, audit trail on every change |

---

## Part B — Partly built: the rule exists but does not behave as the policy says

| # | Policy line | What happens today | What is missing |
|---|---|---|---|
| B1 | §6 "Sick absence beyond 2 days requires a medical certificate" | Leave types hold *requires proof* and *proof threshold days*, and a request has a proof document field | **Nothing enforces it.** A 10-day sick leave with no certificate is accepted |
| B2 | §4 "Half-day leave deducted (or LOP if no balance)" | The half-day penalty deducts half a day's **pay** immediately | It never touches the leave balance, so "leave first, LOP only if no balance" does not happen |
| B3 | §3 Field duty / remote work marked via an on-duty request | Out Duty request exists (date, times, type, destination, purpose, transport) and reaches the approvals inbox | It marks the day **plain Present with a text note** — field duty is invisible in reports. Also hardcodes 09:00 / 17:00 / 480 minutes |
| B4 | §5 LOP status code | Exists only as a payroll deduction line | Not a day status code |
| B5 | §7 "Approved by the reporting manager, then verified by HR" | **Single step** — HR *or* the line manager approves | No second HR verification stage (leave requests have two stages; regularization does not) |
| B6 | §7 "Beyond this, HR head approval is required" | The monthly cap **refuses** the request | No escalation path to allow one above the cap with higher approval |
| B7 | §7 Requests after cut-off "processed in the next cycle" | Refused outright | Not deferred or queued to the next cycle |
| B8 | §8 Overtime "must be pre-approved by the manager and HR" | Authorised **after** the work is done | No pre-approval request before the shift |
| B9 | §9 Attendance locked "after manager confirmation" | The lock is by date / payroll approval | No manager monthly confirmation step at all |
| B10 | §9 "Changes after the lock require HR approval and are paid or recovered in the next cycle" | Everything is refused after the lock | No HR override, no next-cycle arrears |
| B11 | §10 Manager "approve or reject within 2 working days" | The inbox flags items overdue after 2 days | The 2 is **hardcoded**, counts **calendar** days, and no reminder is sent |
| B12 | §3 "mobile app" | The portal works on a phone; sign-in can be tied to a registered phone | No native mobile app |
| B13 | §10 "Payroll: process only HRMS-locked attendance" | Payroll reads the period and locks it on approval | Payroll does not *require* a prior lock |

---

## Part C — Missing outright

| # | Policy line | Note |
|---|---|---|
| C1 | §4 Tier "4–5 occurrences: warning / system alert to manager" | The escalation table has **three** tiers; the system has two (within allowance → nothing, beyond → penalty). No late alert of any kind exists — the only email template in the system is the overtime limit one |
| C2 | §4 "Early going without approval is treated the same as late coming" | Early departure minutes are recorded and shown, but carry no penalty, no status and no monthly allowance |
| C3 | §5 OD (On Duty) status code | — |
| C4 | §5 WFH (Work From Home) status code and request | Nothing anywhere |
| C5 | §6 "Absence of 3 or more consecutive days without information" | No consecutive-absence detection, flag or alert |
| C6 | §7 "within 3 working days of the date" | No submission deadline on a regularization |
| C7 | §2 "Rosters published at least 5–7 days before the roster period starts" | No lead-time rule or warning |
| C8 | §2 "Shift swaps need prior approval from the reporting manager" | No swap request feature; HR edits the roster directly, leaving no request trail |
| C9 | §10 Manager confirms monthly attendance before cut-off | — |
| C10 | §3 Proxy / buddy punching detection | The controls exist (selfie, geofence, registered phone) but no report flags suspected proxy punching |

---

## Part D — Work plan (do in this order)

Each step: DB block appended to `docs/db_changes.sql` and applied twice on local; backend; frontend; unit tests;
clone verification; then `docs/policies.md` notes. Every number configurable per company — no static values.

### Phase 1 — Late, early and absence rules (the ones staff feel)

- [ ] **D1. Enforce proof of sickness** (closes B1) — *smallest change, real compliance value*
  - Enforce `leave_types.requiresProof` + `proofThresholdDays` in `LeaveService.ApplyForLeaveAsync`: refuse when
    counted days exceed the threshold and `ProofDocumentUrl` is empty, with a message naming the leave type and threshold.
  - Allow HR to approve without proof only where the leave type says so — add `proofWaiverAllowedByHr` (default off).
  - Portal apply-leave drawer: show the rule and require the attachment once the dates cross the threshold.
  - Tests: under / at / over the threshold, with and without a document.

- [ ] **D2. Late escalation tiers + manager alert** (closes C1, B2)
  - New table `late_penalty_tiers` (company, `fromOccurrence`, `toOccurrence` nullable, `action` in
    `NONE` / `ALERT_MANAGER` / `DEDUCT`, optional `deductionMode` override). No row = today's single-threshold behaviour.
  - Replace `fn_LatePenalty`'s single `@GraceLateDaysPerMonth` branch with the matching tier; keep the current result
    columns so `sp_ProcessDailyAttendance` and payroll need no change beyond the new `latePenaltyType` value.
  - Add `LATE_WARNING` email template + send to the employee's reporting manager when a day lands in an
    `ALERT_MANAGER` tier (reuse the `OT_LIMIT_EXCEEDED` plumbing and `notification_settings`).
  - New `lateDeductionMode` value `LEAVE_THEN_LOP`: deduct a half day from the leave balance of a nominated leave type
    (`latePenaltyLeaveTypeId` on the company policy), and fall back to the pay deduction only when the balance is short.
  - UI: Attendance Policies page → new "Late coming" tier table (add / edit rows) + the leave type selector.
  - Tests: 1–3 / 4–5 / 6+ with the policy's own table; leave balance sufficient vs short.

- [ ] **D3. Early going treated as late coming** (closes C2)
  - Company policy: `earlyGoingGraceMinutes`, `earlyGoingDaysPerMonth`, `earlyGoingDeductionMode` (same options as late,
    blank = follow the late settings so one setting can drive both).
  - `sp_ProcessDailyAttendance`: set `isEarlyGoing`, `earlySequenceInMonth`, `earlyPenaltyType` from
    `earlyDepartureMinutes` the same way late is handled; new `fn_EarlyPenalty` mirroring `fn_LatePenalty`.
  - Payroll: charge `earlyPenaltyType` alongside the late penalty, as its own payslip line.
  - Tests: within grace, beyond grace within allowance, beyond allowance.

- [ ] **D4. Consecutive unauthorised absence** (closes C5)
  - Company policy: `unauthorisedAbsenceAlertDays` (default off / null).
  - SQL view or function over `attendance_records` finding runs of `ABSENT` days with no approved leave and no
    regularization, at or above the threshold; surface on the Exceptions Queue with its own filter.
  - `ABSENCE_ESCALATION` email template to the manager and HR when a run first reaches the threshold (logged so it is
    sent once per run).
  - Tests: a run below, at and above the threshold; a run broken by approved leave or a weekly off.

### Phase 2 — Day types and request integrity

- [ ] **D5. OD and WFH as real day types** (closes C3, C4, B3, B4)
  - Add `ON_DUTY`, `WORK_FROM_HOME` to `AttendanceStatus`; add `LOP` as a reported day code derived from
    `dayCredit = NONE` + unpaid (no new status needed — decide and document which).
  - New `attendance_day_requests` table covering On Duty and Work From Home (type, date range, times, reason,
    destination / location, status, approval trail) — replaces the note-only Out Duty path and keeps its data.
  - Remove the hardcoded `09:00` / `17:00` / `480` from `ApplyOutDutyAsync`: default the times from the employee's
    scheduled shift (`fn_EmployeeDaySchedule`), refuse when there is no schedule.
  - Approval: manager, then HR, through the existing approvals inbox; approved days processed with the new status and
    full day credit.
  - Reports and the monthly view: show OD / WFH distinctly.
  - Tests: request → approve → day status and credit; no schedule → refused; overlapping leave → refused.

- [ ] **D6. Regularization: deadline, two stages, escalation** (closes C6, B5, B6, B7)
  - Company policy: `regularisationWindowDays` (working days, blank = no limit) and
    `regularisationOverQuotaApproverRole` (blank = refuse, as today).
  - Refuse a request raised more than N working days after the attendance date (count with the employee's schedule).
  - Make regularization two-stage (manager → HR verify) like leave: `currentStageOrder` / `totalStages` on
    `Regularisations`, routed through `ApprovalEngine`; HR-raised ones still skip to HR.
  - Over the monthly cap: instead of refusing, route to the configured HR-head role with the reason, when one is set.
  - UI: portal shows the remaining window and quota; inbox shows the stage.
  - Tests: inside / outside the window; stage 1 then stage 2; over quota with and without an escalation role.

- [ ] **D7. Manager monthly attendance confirmation** (closes C9, B9, B13)
  - New `attendance_period_confirmations` (company, employee's manager or department, period, confirmedBy, confirmedAt).
  - Managers get an "Confirm my team's attendance" screen listing their team for the open period with exceptions
    highlighted; confirming records the row.
  - Company policy `requireManagerConfirmationBeforeLock` (default off, so nothing changes until switched on): when on,
    closing the period / approving payroll refuses while any team in scope is unconfirmed, naming them.
  - Tests: confirm → lock allowed; unconfirmed → refused; flag off → unchanged.

### Phase 3 — Roster and after-the-fact changes

- [ ] **D8. Roster publication lead time** (closes C7)
  - Company policy `rosterPublishLeadDays` (blank = no rule).
  - Warn (not block) on the roster planner when publishing later than the lead time, and show the next period's
    publish-by date on the Shifts dashboard; log the lateness so HR can report on it.
  - Tests: published early / late with the rule set and unset.

- [ ] **D9. Shift swap requests** (closes C8)
  - New `shift_swap_requests` (two employees, date(s), their shifts, reason, status, approval trail).
  - Employee raises a swap with a colleague → colleague accepts → reporting manager approves → the roster day rows of
    both employees are exchanged, refused if either shift is restricted for the other employee (Step 17 check reused).
  - Refuse a swap for a date already past, or in a closed / locked period.
  - Tests: happy path; restricted shift; past date; locked period; colleague declines.

- [ ] **D10. Changes after the lock, with arrears** (closes B10)
  - Allow an HR-approved change inside a locked period, recorded as an adjustment against the **next** open payroll
    period (`payroll_arrears`: employee, source, amount or days, reason, target month, status).
  - Payroll picks up arrears rows for the month it runs and shows them as their own payslip lines.
  - Tests: locked-period change → arrears row → paid or recovered in the next run.

- [ ] **D11. Configurable approval SLA** (closes B11)
  - Replace the hardcoded `slaDays = 2` in `ApprovalsService` with a per-company, per-module setting; count **working
    days** using the employee's schedule and the holiday calendar.
  - Daily reminder to the approver (and escalation to HR) once an item is overdue, using the notification settings.
  - Tests: overdue across a weekend and a public holiday.

### Phase 4 — Optional / lower value

- [ ] **D12. Proxy punching report** (closes C10) — flag punches where the location check failed, the photo is missing,
      or several employees clocked in from the same device or coordinates within a short window.
- [ ] **D13. Overtime pre-approval** (closes B8) — a pre-approval request before the shift, which the existing
      post-hoc authorization then settles against.
- [ ] **D14. Native mobile app** (B12) — out of scope for this plan; the portal is responsive today.

---

## Carried over from the §26 clean-up (still open)

- [ ] Go-ahead to run `docs/policies_step2b_merge_duplicate_shifts.sql` (770 shifts → 55; verified on a clone).
- [ ] Go-ahead to run `docs/policies_recount_past_leave.sql` (1 request, 2 → 3 days; verified on a clone).
- [ ] Choose the "no basic salary" mode per company — on the default **Refuse**, payroll stops until salaries exist
      (3,861 of 3,871 employees have none).
- [ ] Optional: remove the remaining silent statutory fallbacks in `PayrollCalculationEngine` (professional tax 200,
      rebate 700,000, UAE / Tanzania / Zambia rates, and the hardcoded `basic / 30` UAE gratuity daily wage).
- [ ] Go-live: browser walk-through, run `db_changes.sql` on the VM, appsettings keys (`ShiftUploads`, `LeaveAccrual`,
      `Notifications`, `ClockEvidence`), real SMTP, HTTPS.
