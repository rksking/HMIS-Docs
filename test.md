# Shifts, Roster, Attendance, Overtime, Leave & Holiday Policies — Research & Implementation Plan

> Status: **revision 10 (2026-10-03) — Phase 1 (Steps 1–11) and Phase 2 (Steps 12–18) completed, plus the §26 clean-up.
> Phase 3 (Steps 19–29, §27) is pending: the gaps against `docs/artifacts/Staff_Attendance_Policy_HRMS.docx`.**
> Each step is reviewed/approved before code (follow.md #9). Every DB change goes to `docs/db_changes.sql` as a dated, idempotent block.

## Progress

| Step | Scope | Status |
|---|---|---|
| 1 | One schedule resolver + enforce `/masters/shift-grace-policies` | ✅ **Completed** (2026-10-02) — see §6 |
| 2 | Group shift library (multi-company) | ✅ **Completed** (2026-10-03) — see §7; 2b merge script ready, not run |
| 3 | Shift policy + FLEXI shift | ✅ **Completed** (2026-10-03) — see §8 |
| 4 | TA General Settings | ✅ **Completed** (2026-10-03) — see §9 and §11 (4b: close month, clock-in access, specific mobile) |
| 5 | Overtime management | ✅ **Completed** (2026-10-03) — see §10 |
| 6 | Roster + General/Roster uploads | ✅ **Completed** (2026-10-03) — see §12 |
| 7 | Holiday calendars | ✅ **Completed** (2026-10-03) — see §13 |
| 8 | Leave master + eligibility | ✅ **Completed** (2026-10-03) — see §14 |
| 9 | Leave day counting | ✅ **Completed** (2026-10-03) — see §15 |
| 10 | Accrual & carry-forward | ✅ **Completed** (2026-10-03) — see §16 |
| 11 | Reports + notifications | ✅ **Completed** (2026-10-03) — see §17 |
| **Phase 2** | (§18 — items deferred in Phase 1) | |
| 12 | Linked leave | ✅ **Completed** (2026-10-03) — see §19 |
| 13 | Payroll day rate + leave entitlement / pay slabs | ✅ **Completed** (2026-10-03) — see §20 |
| 14 | Leave encashment | ✅ **Completed** (2026-10-03) — see §21 |
| 15 | Comp-off | ✅ **Completed** (2026-10-03) — see §22 |
| 16 | Night-shift allowance | ✅ **Completed** (2026-10-03) — see §23 |
| 17 | Shift restriction by criteria | ✅ **Completed** (2026-10-03) — see §24 |
| 18 | Clock-in selfie / geofence (by flag) | ✅ **Completed** (2026-10-03) — see §25 |
| **§26** | Open items after Phase 2 (fallback salary, working days, countries, entitlement rate, 2 scripts) | ✅ **Completed** (2026-10-03) — see §26; 2 scripts ready, not run |
| **Phase 3** | (§27 — gaps against the Staff Attendance Policy document) | |
| 19 | Enforce proof of sickness on leave | ✅ **Completed** (2026-10-03) — see §28 |
| 20 | Late escalation tiers + manager alert + leave-then-LOP | ✅ **Completed** (2026-10-03) — see §29 |
| 21 | Early going treated as late coming | ✅ **Completed** (2026-10-04) — see §30 |
| 22 | Consecutive unauthorised absence alert | ✅ **Completed** (2026-10-04) — see §33 |
| 23 | On Duty + Work From Home as real day types | ✅ **Completed** (2026-10-04) — see §34 |
| 24 | Regularisation: deadline, two stages, over-quota escalation | ✅ **Completed** (2026-10-04) — see §35 |
| 25 | Manager monthly attendance confirmation | ✅ **Completed** (2026-10-04) — see §36 |
| 26 | Roster publication lead time | ⏳ Pending — **next** |
| 27 | Shift swap requests | ⏳ Pending |
| 28 | Post-lock changes with next-cycle arrears | ⏳ Pending |
| 29 | Configurable approval SLA in working days | ⏳ Pending |
| — | Optional: proxy-punch report, overtime pre-approval, native app | ⏳ Not scheduled |

Sources: handwritten requirement note, `TA Master - Leave Master.pdf`, `TA Master - General Settings.pdf`, `ShiftRosterReport.xlsx`, the `/masters/shift-grace-policies` screen, and current code (entities, `ShiftService`, `AttendanceService`, `LeaveService`, `PayrollCalculationEngine`, `docs/dbscript/proc.sql`, seed data).

---

## 0. Decisions confirmed (from your answers)

| # | Topic | Decision |
|---|---|---|
| D1 | Company / shift names in the note (LCH, MAKL, JIVA, G1–G4) | **Examples only.** Nothing is seeded or hardcoded from them; all companies/shifts come from the DB. |
| D2 | Grace / late rules | **Use `/masters/shift-grace-policies` (already DB-backed: `company_attendance_policies`) as the company rule.** Its rules are now enforced in processing (§3.4). |
| D3 | Sheet upload | **Upload is the way to assign both General (fixed) shifts and Roster (day-wise) shifts.** Sheet key = `Employee Id` = `employees.employeeNumber`. |
| D4 | Flexi shift | A FLEXI shift has a **window** (e.g. 10:00 → 22:00) and **required working hours** (e.g. 9 h) inside that window. No fixed start time. |
| D5 | Overtime | Follow the **Over Time Settings** of the General Settings PDF: OT1–OT4, authorization, pay-authorized-irrespective-of-actual, max OT limits per employee type (Regular/Casual …), excess-hours notification (§3.6). |

---

## 1. Requirement summary

| # | Requirement |
|---|---|
| R1 | Shifts are **not limited to a company** — defined once for the group, enabled for any company. |
| R2 | A shift carries its **weekly-off pattern** per weekday (e.g. one shift Tue–Sun, another Mon–Fri). |
| R3 | **Week start day** setting. |
| R4 | Assignment applies per **calendar month**; an employee is either on a **general (fixed) shift** or **rostered** across several shifts. |
| R5 | **Roster is prepared in advance**; biometric punches are matched against it. |
| R6 | **Attendance policy** (grace, flexi tolerance, late-days allowance, deduction mode, half/full day hours, OT threshold, regularisation quota, geofence) applies to all shifts. |
| R7 | **Flexi shift** = window + required hours (D4). |
| R8 | **Shift policy** attached to a shift: OT rule, *Ignore Early Arrival*, *Ignore Late Departure*. |
| R9 | Process: **Create shift → set shift policy → done.** |
| R10 | **Leave policy "count weekly off"**: Fri→Mon leave = 2 days (off) / 4 days (on). |
| R11 | **Holiday** mapping. |
| R12 | **Overtime management** per D5. |

---

## 2. Current state (what exists)

### 2.1 Tables
| Table | Holds | Scope |
|---|---|---|
| `shifts` + `shift_day_rules` | timings, grace, OT eligible, night/overnight, weekday rules (FULL_DAY / WEEKLY_OFF …) | per company (`UQ_Shift_Company_Code`) |
| `work_schedules` (+days) | weekly working-day template | per company |
| `employee_shift_assignments` | employee → shift, FIXED / ROTATION, effective from/to | per company |
| `employee_shift_day_overrides` | one date or recurring weekday override | per company |
| `company_attendance_policies` | **everything on `/masters/shift-grace-policies`**: flexi tolerance, grace minutes, grace days/month, deduction mode, regularisation quota, half/full day hours, OT threshold/rounding/min/multipliers, geofence | one row per company |
| `holiday_types`, `holidays` | holiday per company per date | per company |
| `leave_types`, `leave_balances`, `leave_requests` | | per company |
| `employment_types` | PERM / CONTRACT / CASUAL / LOCUM / INTERN / PROBATION … | per company — used for OT limits (§3.6) |
| `biometric_punch_logs`, `attendance_records`, `attendance_exceptions` | punches + daily result (`sp_ProcessDailyAttendance`) | per company |
| `pay_components` | payroll components — target for OT3/OT4 payment | per company |

### 2.2 Screens
`/shifts/master`, `/shifts/schedules`, `/shifts/roster` (bulk assignment, **no day-wise grid**), `/shifts/lookup`, `/masters/shift-grace-policies` (**real, saves to DB**), `/leave-holidays`, `/leave/*`, `/approval-matrices/attendance`.

### 2.3 Gaps
1. **Attendance processing ignores the Shifts module** — `sp_ProcessDailyAttendance` uses only `employment_details.shiftId`; assignments, rotations, overrides have no effect on late/OT/absent. *Fixed in Step 1.*
2. **`/masters/shift-grace-policies` is only half enforced** — `graceLateDaysPerMonth` and `lateDeductionMode` are saved but **never used** by any procedure; payroll counted LATE days against the allowance on its own and ignored the mode. (`maxRegularisationsPerMonth` *is* enforced, in `AttendanceService.SubmitRegularisationAsync`.) *Fixed in Step 1.*
3. **No day-wise roster** and no upload for general/roster assignment.
4. **Shifts duplicated per company** — 53 codes × up to 14 companies, all with identical timings (verified).
5. **No real flexi shift** (window + required hours); no ignore-early / ignore-late-departure.
6. **Overtime is a single number** — `attendance_records.overtimeHours` + one payroll multiplier. No OT types, no authorization, no limits, no weekly-off/holiday OT split.
7. **Leave counting hardcodes Sat/Sun**; no week-off/holiday flags (R10).
8. **Holidays re-entered per company**; no location holidays; no "worked on holiday".
9. **Leave master thin** vs the reference PDF (eligibility, monthly accrual, carry-forward type …).

### > Code correction and static values (follow.md #10)
| Location | Static value / issue |
|---|---|
| `AttendanceService.cs` L126–129, L591, L1029–1030, L1055–1056, L1132–1133, L1180–1183 | `"G1"`, `"General Hospital Day Shift (G1)"`, `"Standard Day Shift"`, `"Standard Shift"`, `"STD"`, `"08:00 - 17:00"`, grace `15` |
| `ShiftService.cs` L1206–1215, L1243–1254 | Sat/Sun assumed weekly off; `"08:00"`, `"17:00"`, `"13:00–14:00"`, `8.0m` |
| `proc.sql` `sp_ProcessDailyAttendance` L1378–1440 | policy defaults (15/30/4/8/30/15/30); `'08:00'`/`'17:00'`; Sunday = week off when no rule; holiday join `companyId IS NULL`; punch window −120/+240; `'EMP-'` prefix stripping |
| `proc.sql` `sp_GetEmployeeMonthlyAttendance` L859–935 | grace `15` |
| `AttendanceSettingsTab.tsx` L156–166 (Policy Simulator) | sample shift `08:00–17:00` and 60-min break hardcoded → should pick a real shift from DB |
| `AttendanceSettingsTab.tsx` L47–57, `ConfigService.cs` L438 | client/C# defaults (30/15/3/4/8/150 …) before load / on auto-create → defaults must come from SQL column defaults |
| `PayrollCalculationEngine.cs` L28, L156, L291 | OT multiplier fallbacks `1.50m` / `2.00m` / `1.25m` |
| `LeaveService.cs` L151–153, L262–289, L437–446 | code-seeded leave types; hardcoded country legal text; Mon–Fri working days |

Rule: when no shift/roster resolves for a day → `UNSCHEDULED` attendance exception, never an invented 08:00–17:00.

---

## 3. Target design

### 3.0 Structure
```
GROUP
 ├── Shift Library (FIXED and FLEXI shifts) ── enabled per company (shift_companies)
 │     └── Shift Policy (optional override of company rules: OT / early-late / punch window)
 ├── Holiday Calendars (per country) ── assigned to companies (+ optional location)
 └── COMPANY
       ├── Attendance Policy  = /masters/shift-grace-policies  (+ General / OT / Mobile sections)
       ├── Overtime Types OT1–OT4, OT limits per employment type
       ├── Leave Types + eligibility
       └── EMPLOYEE
             ├── General shift  (employee_shift_assignments)   ← upload or drawer
             ├── Roster days    (shift_roster_days)            ← upload or grid
             └── Overrides      (employee_shift_day_overrides) ← ad-hoc
```

### 3.1 Shift assigned to employee (general shift)
`employee_shift_assignments` (FIXED) = the employee's general shift, open-ended or with effective dates. Assigned via the existing drawer **or the General Shift upload** (§3.5).

### 3.2 Shift + Shift Policy
- Shift fields stay as today, plus `shiftType` FIXED | FLEXI.
- **FLEXI shift (D4):** `windowStart` (10:00), `windowEnd` (22:00), `requiredMinutes` (540 = 9 h). Day result:
  - worked = last punch − first punch − break, counted only inside the window;
  - worked ≥ required → present full day; OT = worked − required (subject to OT rules);
  - worked < required → half day if worked ≥ company `minHoursHalfDay`, else absent (for FLEXI the shift's required hours replace the company full-day hours);
  - **no "late" for flexi shifts** (any arrival inside the window is valid); punch outside the window → exception.
- **Shift Policy** (named, group-level, optional on a shift). It only holds what can differ per shift:
  `ignoreEarlyArrival`, `ignoreLateDeparture`, `otStartAfterMinutes` (overrides company OT threshold), `otMaxMinutesPerDay`, `punchWindowBeforeMinutes`, `punchWindowAfterMinutes`, optional `graceLateMinutes` override.
  Blank field ⇒ the company attendance policy applies (the page is "applicable to all shifts").
- Flow R9: Add Shift drawer → pick policy (or none) → save.

### 3.3 Company flexi tolerance vs FLEXI shift
These are two different things and both stay:
- **Flexi-Time Window** on the policy page (e.g. 30 min) = arrival tolerance for **FIXED** shifts.
- **FLEXI shift** = a shift type with window + required hours (D4).

### 3.4 Attendance policy rules — enforced exactly as `/masters/shift-grace-policies` defines them
For a FIXED shift starting at `S` (example 08:00, flexi 30, grace 15, 3 days, Half-Day mode):

| First punch | Status | Counts toward monthly late allowance? | Penalty |
|---|---|---|---|
| ≤ S + flexi (≤ 08:30) | ON_TIME / FLEXI_ON_TIME | no | none — must complete full-day hours |
| ≤ S + flexi + grace (08:30–08:45) | see question Q1 | see Q1 | none while within allowance |
| > S + flexi + grace (> 08:45) | LATE | yes | from the (graceLateDaysPerMonth + 1)-th late arrival in the month → `lateDeductionMode`: HALF_DAY = 0.5 × (Basic ÷ 30), EXACT_MINUTES = (late min ÷ 60) × hourly rate, FLAT_FINE = company disciplinary pay component |

Also enforced:
- Day credit: worked < `minHoursHalfDay` → absent; < `minHoursFullDay` → half day; else full day.
- OT starts accruing only after `overtimeThresholdMinutes` past shift end (shift policy may override).
- Regularisation requests limited to `maxRegularisationsPerMonth` per employee.
- Geofence on mobile clock-in by `isGeofenceEnforced` + `geofenceRadiusMeters`.
- The month for counting late arrivals = **attendance period** (cut-off based, §3.7).

New fields on `attendance_records` so payroll can use the result: `lateSequenceInMonth`, `isLateCovered`, `latePenaltyType` (NONE / HALF_DAY / EXACT_MINUTES / FLAT_FINE), `dayCredit` (FULL / HALF / NONE).
The Policy Simulator on the page will call the **same SQL logic** (new SP in simulate mode) with a real shift, instead of its own JS copy.

### 3.5 Roster + Uploads (D3)
**Tables:** `shift_rosters` (company, month, DRAFT → PUBLISHED → LOCKED) and `shift_roster_days` (employee, date, `shiftId` or `dayType = WEEKLY_OFF`; unique employee+date). A half-day shift (e.g. `Half Day G16`) is simply that shift; no extra flag.

**Sample file:** `docs/ShiftRosterReport.xlsx` is the reference for the multi-shift (roster) upload. General (single) shift assignment already works from the drawer; the General template below is the bulk version of it.
Parser must accept what the sample contains: any sheet name (sample: `ShiftRosterReport24Sep202618044`), `Employee Id` as text **or number** (`71247` arrives as a numeric cell), header `Shift Date-dd-mm-yyyy` per day, labels with unpadded times (`Day G5(08:30 am to 5:30 pm)`) — only the code before `(` is used.
**Data validation is mandatory, same pattern as the bulk employee upload** (`Infrastructure/Services/EmployeeImport/*`): parse → validate every row/cell → preview with per-row errors and downloadable error file → commit only valid rows in one transaction → audit log.

**One "Upload Shifts" drawer, two templates (downloadable, generated from DB):**
| Template | Columns | Writes to |
|---|---|---|
| **General Shift** | `Employee Id`, `Employee Full Name` (info only), `Shift Code`, `Effective From`, `Effective To` (optional) | `employee_shift_assignments` (FIXED); previous open assignment is closed the day before |
| **Roster** (= `ShiftRosterReport.xlsx` layout) | `Employee Id`, `Employee Full Name`, `Shift Date-dd-mm-yyyy` … | `shift_roster_days` for that month |

Roster cell parsing: `Day G6(09:00 am to 06:00 pm)` / `Half Day G16(...)` → shift code = text before `(` minus the `Day` / `Half Day` prefix; `Weekly Off` → WEEKLY_OFF; plain code `G6` also accepted; blank = no change.

**Multi-company handling:** your sample sheet mixes companies (e.g. `71247` is LCH, `60002` is SDI). `employeeNumber` is unique per company only, so:
- Upload from a single company workspace → match within that company.
- Upload from a group workspace (superadmin) → match across the group's companies; if a number exists in more than one company the row is rejected as ambiguous.
- The shift must be enabled for **that employee's** company.

**Validation preview → commit** (reuses the existing bulk preview/commit pattern): unknown employee (e.g. `DARWINTEST` is not in the DB → listed as error), unknown/disabled shift, LOCKED month, beyond back-dated limit, duplicate rows. Errors are downloadable; valid rows commit in one transaction.

**Roster Planner grid** (`/shifts/roster`): employee × day, cell picker, weekly off grey, weeks split by `weekStartDay`, "Generate from general shift" to pre-fill a month, Publish / Lock, Export (same xlsx layout).

### 3.6 Overtime management (D5, from the PDF)
**OT types (DB rows, not 4 hardcoded columns)** — `overtime_types` per company:
| Field | PDF item |
|---|---|
| `code` (OT1…OT4), `name` | OT1–OT4 |
| `isApplicable` | "OT1 Applicable" |
| `appliesOn` (WORKING_DAY / WEEKLY_OFF / HOLIDAY / NIGHT) | which hours fall into this type |
| `rateMultiplier` | replaces payroll fallbacks 1.5 / 2.0 |
| `requiresAuthorization` | "OT1 Authorization From Software" |
| `payAuthorizedIrrespectiveOfActual` | "Pay Authorized OT1 Hrs. Irrespective of Actual" |
| `authorizedDayOutBeyondShiftEnd` (OT1) | "AuthorizedOT1DayOutTimeisBeyondShiftEndTime" |
| `payComponentId` | "OT3 / OT4 Payment" |
| `hideOnWebAndMobile` | "Hide Tiles for OverTime" |

**Per-day OT result** — `attendance_overtime` (attendanceRecordId, overtimeTypeId, actualMinutes, authorizedMinutes, payableMinutes, status PENDING / APPROVED / REJECTED / AUTO). Processing splits the day's OT into a type by `appliesOn` (e.g. working day → OT1, weekly off/holiday → OT2), after applying shift policy (ignore early/late) and threshold/rounding.
- `requiresAuthorization = 0` → AUTO, payable = actual.
- `requiresAuthorization = 1` → PENDING, goes through the existing attendance approval matrix (`/approval-matrices/attendance` already lists "overtime sign-off"); payable = authorized, or = min(actual, authorized) unless "pay irrespective of actual".

**Max OT authorization limit** — `overtime_limits` (company, `employmentTypeId` — Regular/Casual in the PDF map to our `employment_types` rows, `limitType` TOGETHER / SEPARATE per OT type, `maxMinutesPerPeriod`, `maxMinutesPerWeek`, `weekStartDay`). Authorization beyond the limit is blocked with a message.

**Excess hours notification** — when the limit is crossed: email to configured To/CC/BCC using an email template. (Needs a general email-template master; today only `onboarding_email_templates` exists → done as a later step.)

**Payroll** — reads payable minutes per OT type × `rateMultiplier` (or pay component) instead of the single `overtimeHours` × multiplier.

### 3.7 Company TA General Settings (rest of the PDF)
Added as sections on the **same `/masters/shift-grace-policies` page** (no new page): `cutOffDay` (attendance period, e.g. 24th → 23rd), `weekStartDay`, `backDatedScheduleEditDays` (365), `closeMonthImpact`, Mobile (`selfieRequired`, `remarksOnClockRequired`, `geofenceMode`, `loginWithSpecificMobile`). Deferred: night-shift allowance hours from employee master, shift-schedule restriction by criteria, worked-hours authorization method.

### 3.8 Single source of truth — `fn_EmployeeDaySchedule(@CompanyId, @From, @To, @EmployeeId)`
One SQL function answers "what is employee X scheduled for on date D", used by attendance processing, leave counting, roster grid, lookup, reports (replaces C#-only `ResolveEmployeeDay`).
Precedence: **1** date override → **2** published roster day *(added in Step 6)* → **3** recurring weekday override → **4** general shift assignment (weekday rule of the shift) → **5** `employment_details.shiftId` → **6** company default shift (`shifts.isDefault`) → **7** `UNSCHEDULED`.
Then holiday overlay (`isHoliday`, `isHolidayWorked`) and effective policy values (company policy + shift policy overrides).

### 3.9 Holidays — where to map
Not to employee, not to shift. **Holiday Calendar (group-level, per country) → assigned to company (+ optional location)**; employee inherits by company/location. Roster decides if the holiday is *worked* (→ holiday OT type). Leave type decides if a holiday inside a leave counts.

### 3.10 Leave policies — where to map
**Company leave type + eligibility criteria** (gender, department, category, division, position, designation, location; empty = All). Not to shift. The schedule is used only to **count days**: each date in the leave is resolved via `fn_EmployeeDaySchedule`; weekly off counted only if `countWeekOffAsLeave` (R10), holiday only if `countHolidayAsLeave`, `halfDayWeekdays` = 0.5, `ignoreWeekdays` = 0.
Leave types stay per company (statutory per country) with a "Copy to companies" action.
Phase-1 Leave Master fields: `shortName`, `colorCode`, `countWeekOffAsLeave`, `countHolidayAsLeave`, `halfDayWeekdays`, `ignoreWeekdays`, `restrictHalfDay`, `carryForwardType`/`Value`/`ValidTillMonth`, `monthlyAccrual`, `monthlyAllocation`, `accrualOn`, `prorateOnJoining`, `minApplicationDays`, `maxApplicationsPerMonth`, `minGapBetweenApplicationsDays`. Deferred: encashment, formulas, payment slabs, linked leave, comp-off.

### 3.11 Multi-company summary
| Master | Owner | Company gets it via |
|---|---|---|
| Shift, Shift Policy | Group (owner company edits) | `shift_companies` |
| Holiday Calendar | Group (per country) | `company_holiday_calendars` |
| Attendance policy, TA settings, OT types, OT limits | Company | own rows |
| Leave types | Company | own rows (+ copy helper) |
| Assignments, roster, overrides, attendance, OT results | Company | employee's company |

`Shift` query filter (`HrmsDbContext` L555) becomes: shift in active company's group **and** enabled for active company.

---

## 4. Implementation steps (one approval / PR each)

| Step | Scope | Main DB objects |
|---|---|---|
| **1. Resolver + enforce attendance policy page** ✅ | `fn_EmployeeDaySchedule` (precedence 1, 3–6); rewrite `sp_ProcessDailyAttendance` to use it and enforce §3.4 (late allowance, deduction mode, day credit, OT threshold, regularisation quota); `UNSCHEDULED` exception; remove static fallbacks; simulator calls SQL with a real shift | function, SP rewrite, `attendance_records` + `lateSequenceInMonth`, `isLateCovered`, `latePenaltyType`, `dayCredit`; `sp_SimulateAttendancePolicy` |
| **2. Group shift library** ✅ | `shifts.groupId`, nullable `companyId`, `shift_companies`, backfill; drawer "Applicable companies"; filtered dropdowns. **2b** (separate manual script, dry-run report + snapshot): merge identical duplicate codes | `shift_companies` |
| **3. Shift policy + FLEXI shift** ✅ | `shift_policies`; `shifts.shiftPolicyId`, `shiftType`, `windowStart`, `windowEnd`, `requiredMinutes`; processing for ignore early/late, punch window, flexi day logic; Shift Policies page under Shifts | `shift_policies` |
| **4. TA General Settings** ✅ | new sections on `/masters/shift-grace-policies` (§3.7); late allowance counted per cut-off period | `company_attendance_policies` new cols |
| **5. Overtime management** ✅ | OT types, per-day OT split, authorization via attendance approval matrix, limits per employment type, payroll reads per-type OT | `overtime_types`, `attendance_overtime`, `overtime_limits` |
| **6. Roster + uploads** ✅ | roster tables, precedence 2, Roster Planner grid, Upload drawer (General + Roster templates), export | `shift_rosters`, `shift_roster_days` |
| **7. Holiday calendars** ✅ | calendars per country, assign to company/location, holiday-worked | `holiday_calendars`, `holidays.calendarId`, `company_holiday_calendars` |
| **8. Leave master + eligibility** ✅ | §3.10 fields, eligibility, copy-to-companies; remove code-seeded types / legal text | `leave_types` cols, `leave_type_eligibility` |
| **9. Leave day counting** ✅ | `sp_CalculateLeaveDays` via resolver; apply drawer per-day breakdown | SP |
| **10. Accrual & carry-forward** ✅ | `sp_RunLeaveAccrual` monthly, year-end carry-forward | SP |
| **11. Reports + notifications** ✅ | Shift Roster Report, roster vs actual, OT register; email-template master + excess-hours notification | `email_templates` (general) |

**Permissions** (dot-notation, added with each step, plus `menus` rows): `shifts.policies.view/manage` (3), `attendance.settings.manage` (4), `overtime.view`, `overtime.authorize`, `overtime.settings.manage` (5), `shifts.roster.view/manage/publish`, `shifts.upload` (6), `holidays.view/manage` (7, check existing first).

**Verification per step** (follow.md #12, and user-data rule): snapshot affected rows before any write test; scenarios as Employee / Line Manager / HR; `tsc --noEmit` + `dotnet build` clean. Key scenarios:
- Step 1: 4th late arrival beyond grace in a period → half-day penalty; 3rd → covered; G-shift with Monday off shows WEEK_OFF on Monday.
- Step 3: FLEXI 10:00–22:00 / 9 h: in 12:00 out 21:30 (1 h break) → 8.5 h < 9 h → half day; in 10:15 out 20:30 → 9.25 h → full day, 15 min extra → no OT with a 30-min threshold.
- Step 5: Sunday (weekly off) worked 6 h → OT2 6 h; OT1 requiring authorization stays PENDING until approved; weekly limit blocks extra authorization.
- Step 6: import the attached Aug-2026 sheet into a test month, export, compare; `DARWINTEST` reported as unknown.
- Step 9: Fri→Mon leave for a Mon–Fri employee: 2 days (flag off) / 4 days (flag on).

---

## 5. Remaining questions (small — defaults used if no reply)

- **Q1 — 08:30–08:45 band (inside grace).** The page's *Operational Rule* text says it is **logged as Late and uses the 3-day allowance**; the page's *Policy Simulator* code treats it as **within grace, not counted**. **Default: follow the Operational Rule text** (late, counted, penalty from the 4th) — confirm.
- **Q2 — Step 2b merge** of identical duplicate shift codes into one group shift (after a dry-run report). **Default: yes, as a separate manual script you run.**
- **Q3 — OT type mapping** default rows per company: OT1 = working day (×1.5), OT2 = weekly off + holiday (×2.0), OT3/OT4 not applicable — editable on screen. Confirm multipliers.

---

## 6. Step 1 — completion notes (2026-10-02)

### What was delivered
| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 1") | `fn_EmployeeDaySchedule`, `fn_EvaluateAttendanceDay`, `fn_LatePenalty`; rewritten `sp_ProcessDailyAttendance` and `sp_GetEmployeeMonthlyAttendance`; new `sp_SimulateAttendancePolicy`; `attendance_records` + `lateSequenceInMonth`, `isLateCovered`, `latePenaltyType`, `dayCredit`; a policy row for every company (from column defaults). Applied on local DB; re-run is safe. |
| Late rule (enforced) | Late = first punch after shift start + flexi window. Within grace → LATE but covered while it is one of the first *N* late arrivals in the calendar month; from the (*N*+1)-th → deduction mode. Beyond grace → deduction mode straight away. (Q1 default: the page's Operational Rule.) |
| Day credit | Full / half / absent from the page's min hours; a short shift (e.g. 5 h half-day shift) needs only its own hours for a full day. |
| Overtime | From shift end once past the OT threshold, min-minutes and rounding applied; worked time on weekly off / holiday counts as OT (split into OT types in Step 5). |
| Weekly off / holiday | From the employee's shift day rules (no more "Sunday" or 08:00–17:00 defaults); holiday join bug fixed. |
| Unscheduled | No shift at all → `UNSCHEDULED` attendance exception (10 active employees today have none). |
| Shifts lookup (`/shifts/lookup`) | Uses `fn_EmployeeDaySchedule` (C# `ResolveEmployeeDay` removed); new "Unscheduled" filter/badge; source shows Date override / Day override / Shift assignment / Employment shift / Company default shift. |
| Shift / override changes | Assigning a shift or adding/cancelling an override re-processes the affected already-processed days through the procedure (payroll-locked days untouched). The duplicate C# late calculation was removed. |
| Register | Shows "#n" late arrival, "Covered by monthly allowance" or the deduction; shift/times from the day's schedule. |
| Payroll | Late deduction now reads each day's `latePenaltyType` (HALF_DAY = ½ day; EXACT_MINUTES = late minutes × hourly rate). |
| Policy Simulator | Pick a real shift + date; evaluated by `sp_SimulateAttendancePolicy` with the unsaved form values (old JS copy of the rules removed). |
| Static values removed | `"G1"`, `"General Hospital Day Shift (G1)"`, `"Standard Shift"`, `"STD"`, `"DAY"`, `"08:00 - 17:00"`, grace `15`, regularisation `?? 3`, C# policy defaults (now only SQL column defaults via `CompanyAttendancePolicyStore`), and the C# monthly-attendance fallback (errors now surface instead of showing invented data). |

### Verification done
- `dotnet build` ✔, `dotnet test` 42/42 ✔, `tsc --noEmit` ✔.
- `fn_EmployeeDaySchedule` for LCH, whole month: ~0.7 s.
- `sp_ProcessDailyAttendance` (LCH, 2026-09-29) and `sp_GetEmployeeMonthlyAttendance` run inside a **rolled-back transaction** — no attendance data changed.
- Simulator via API on G1: 08:20 on time · 08:40 #2 covered · 08:40 #4 half-day deduction · 09:10 beyond grace + 90 min OT · 3.5 h worked → absent.

### Things to know
- **Existing attendance rows are not re-calculated automatically.** They pick up the new rules when a day is processed again (biometric sync, "Process" in the register, or a shift/override change). Re-processing a past month will change late/penalty results and therefore payroll — decide per month.
- The late allowance counts per **calendar month** until Step 4 adds the cut-off period.
- `FLAT_FINE` is stored on the day but **not yet deducted** in payroll — it needs the fine amount/pay component (Step 5).
- `ROTATING` assignments are not resolved (no screen writes them, no rows exist); the Roster (Step 6) covers multi-shift.
- Punch window (2 h before / 4 h after) is still fixed in the procedure → moves to Shift Policy in Step 3.
- Still open from §2 static list: "Reset Defaults" button and initial form numbers on the policy page, `PayrollCalculationEngine` OT multiplier fallbacks, `LeaveService` items (Steps 4, 5, 8).

### Next: Step 2 — Group shift library
Scope for review before coding: `shifts.groupId`, nullable `shifts.companyId`, new `shift_companies`; backfill each shift to its own company; Shift Master "Applicable companies"; shift dropdowns filtered by company; `fn_EmployeeDaySchedule` and the company-default lookup made group-aware. Step 2b (merging the identical duplicate codes) stays a separate manual script with a dry-run report first.

---

## 7. Step 2 — completion notes (2026-10-03)

### Design as built (one change from the plan)
- `shifts.companyId` **stays NOT NULL** and means the **owner company** (the only company that edits the shift). The plan said "nullable"; keeping it avoids touching every shift query, the tenant FK and `ITenantEntity`, and gives a clear owner.
- `shifts.groupId` (NOT NULL, FK `groups`) = owner company's group.
- `shift_companies (shiftId, companyId, isActive)` = companies that may use the shift. The owner is always in it.
- `shifts.isDefault` keeps meaning "default shift of the owner company" (used by `fn_EmployeeDaySchedule` tier 6).

### What was delivered
| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 2") | `shifts.groupId` + FK + index; `shift_companies`; backfill: every shift enabled for its owner **and for every company whose employees already use it** (1,727 employees in 13 companies were already on LCH's `shf-g1`, so G1 is now enabled for those companies); `sp_SimulateAttendancePolicy` checks `shift_companies`. Applied on local; re-run safe. |
| Rules (`ShiftService`) | Only the owner company (or super admin / developer) edits or (de)activates a shift. Sharing is limited to companies of the group the user can access; companies already enabled but outside the user's access are left as they are. A newly added company must not already use the same code. A company can't be removed while its employees still use the shift (assignments, employment shift, overrides). Assigning a shift (single, bulk, override) requires it to be enabled for the employee's company. |
| Shift lists | Every "shifts of this company" query now uses `AvailableTo(companyId)` (Shift Master, onboarding, employee import lookups, dashboard, legacy `/config/shifts`). The EF tenant filter on `Shift` also lets enabled companies see shared shifts. |
| Shift Master UI | Drawer: **Applicable Companies** checklist (owner locked); read-only notice + disabled save for a shift owned by another company. List: owner company + "Shared · N companies" badge; activate/deactivate only for the owner. |
| API | `GET /api/shifts/shareable-companies`; create/update/assign/override/bulk return validation errors as 400 with the reason (was an unhandled 500). Shifts screens now show `errors[0]` from the API. |
| Static values removed | Legacy `ConfigService.GetShiftsAsync` returned **5 invented shifts** (DAY/EVE/NIGHT/WKND/ICU) when the query failed, hours `8.5`, department "Clinical & General"; shift DTO fallback company "Bliss Healthcare Group"; dashboard shift count fallback `6`; "use another company's shifts when a company has none". |

### Step 2b — merge identical copies (`docs/policies_step2b_merge_duplicate_shifts.sql`)
- **Manual, opt-in, not in `db_changes.sql`.** `@DryRun = 1` by default prints the report only.
- Local dry run: **770 shifts → 55** (one per code). Copies are identical in every setting and day rule; G1 copies differ only by name ("Day G1 (08:00 am to 05:00 pm)" vs "General Hospital Day Shift (G1)") — the kept name is the company-default `shf-g1`.
- Verified with `@DryRun = 0` on a throwaway clone of the DB: 55 shifts, no broken references, every employee's shift still enabled for their company, unique index `(groupId, code)` created, lookup and attendance processing work.
- **Not run on your local DB.** Run it when you are ready (dry run first), then on the VM.

### Verification done
- `dotnet build` ✔, `dotnet test` 42/42 ✔, `tsc --noEmit` ✔.
- API scenarios on a throwaway DB clone (`HRMSCore_StepTest`, dropped afterwards): share/create/edit, code clash, removal while in use, assign a non-enabled shift, non-owner edit, sharing outside access — all behave as above.

### Incident during testing (fixed)
A "should be rejected" update on `shf-lch-g5` was **saved to the local DB** because the EF tenant filter hid the clashing AGI shift from the check. It was restored from `docs/dbscript/data/shift_data.sql` (original day-rule ids, 9.00 h, AGI sharing row and audit row removed) and the dry run confirmed G5 is identical to its copies again. The bug is fixed (`IgnoreQueryFilters` on the check); all later write tests ran on a throwaway clone.

### Things to know
- Existing Shift Master behaviour (not changed here): saving a shift in the drawer stores hours **net of the break** (09:00–17:30 with 1 h break → 8.00 h), while seed data stores gross hours (9.00). Any edit of a seed shift therefore changes its hours. Decide in Step 3 which one is correct.
- Work schedules still fall back to the default tenant company's schedules when a company has none (`ShiftService.GetWorkSchedulesAsync`) — flagged, not changed.

### Next: Step 3 — Shift Policy + FLEXI shift
Scope for review before coding: `shift_policies` (group-level: ignore early arrival, ignore late departure, OT start/max per day, punch window before/after, optional grace override); `shifts.shiftPolicyId`, `shiftType` FIXED/FLEXI, `windowStart`, `windowEnd`, `requiredMinutes`; `fn_EvaluateAttendanceDay` / `sp_ProcessDailyAttendance` apply them (FLEXI: no late, full day = required hours inside the window, e.g. 10:00–22:00 / 9 h); Shift Policies page + policy dropdown and FIXED/FLEXI fields in the shift drawer.

---

## 8. Step 3 — completion notes (2026-10-03)

### Rule followed: no fixed numbers
Every number used by attendance processing now comes from the database and is editable on a screen:

| Value | Where it is set | Was |
|---|---|---|
| Punch window before shift start / after shift end | Policy page → Overtime & Punch Rules (company), Shift Policy (override) | fixed 120 / 240 min in `sp_ProcessDailyAttendance` |
| Lone punch counts as clock-out after | Policy page | fixed 4 h |
| Overtime minimum, rounding | Policy page (stored before, not editable) | not on screen |
| Maximum overtime per day | Policy page / Shift Policy | did not exist |
| Ignore early arrival / late departure for OT | Policy page / Shift Policy | early always ignored, late always counted |
| Late grace, OT starts after | Policy page / Shift Policy | company only |
| FLEXI required hours | Shift drawer (per shift) | did not exist |

Also removed: shift drawer pre-filled times (08:00–17:00, 13:00–14:00) and "Sat/Sun off" for missing days, grace 10 / flexi 30 defaults on shifts and DTOs, "night shift = starts after 12:00" (now: crosses midnight), the policy page's starting numbers, hardcoded "Reset Defaults" values (now read from the SQL column defaults via `GET /api/attendance/settings/defaults`), the stepper upper limits (120, 60, 10, 16 …), the timeline's fixed "08:00" sample (now a real shift of the company, which also fixes times like "08:75"), the employee Shift tab's "+15 mins / 10m" fallbacks.

### What was delivered
| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 3") | `company_attendance_policies` + `ignoreEarlyArrival`, `ignoreLateDeparture`, `overtimeMaxMinutesPerDay`, `punchWindowBeforeMinutes`, `punchWindowAfterMinutes`, `singlePunchAsOutAfterMinutes` (defaults = previous behaviour); new `shift_policies` (group-level, owner company, all values nullable = inherit); `shifts.shiftPolicyId`, `shiftType` (FIXED/FLEXI, CHECK), `flexiRequiredMinutes`; permissions `shifts.policies.view/manage` granted to roles with `shifts.view/manage`; menu "Shift policies" (`/shifts/policies`). Functions/procs updated: `fn_EvaluateAttendanceDay`, `fn_EmployeeDaySchedule`, `sp_ProcessDailyAttendance`, `sp_GetEmployeeMonthlyAttendance`, `sp_SimulateAttendancePolicy`. Applied on local; re-run safe. |
| FLEXI shift | Day rule times = working window (e.g. 10:00–22:00), `flexiRequiredMinutes` = full day (e.g. 540). No late marking. Day credit uses time **inside the window**; half day = company half-day hours (or half the requirement if smaller). OT = time inside the window beyond the requirement + time outside the window unless ignored. |
| Overtime | Extra time = early arrival (unless ignored) + late departure (unless ignored) [+ FLEXI excess]; counted on a working day once it reaches the threshold; then minimum, rounding, daily cap. Weekly off / holiday: all worked time (min, rounding, cap). |
| Effective values | Per shift: shift policy value if set and the policy is active, else company value — the same in processing, the monthly late-allowance recount and the simulator. |
| Shift Policies page | `/shifts/policies` (nav pill + menu): list with inherited values in grey, SideDrawer form (blank = company value), owner-only editing, active/inactive. API `GET/POST/PUT /api/shift-policies`. |
| Shift drawer | Shift type FIXED/FLEXI, FLEXI required hours, Shift policy dropdown; dead per-shift grace / flexi fields removed (columns kept); new shifts start blank and are validated (all 7 days, times on working days). |
| Policy page | New "Overtime & Punch Rules" section; simulator shows shift type, policy and the effective values it used. |

### Verification done
- `dotnet build` ✔, `dotnet test` 42/42 ✔, `tsc --noEmit` ✔.
- Regression (rolled-back txn): LCH 2026-09-29 re-processed with no policies gives the same status counts as before (437 / 1339 / 195 / 169 / 1).
- SQL scenarios (rolled-back txn), FLEXI 10:00–22:00 / 9 h: 12:00–21:30 → full day, 30 min OT; 10:15–17:30 → half day, never late; 09:00–20:30 → 90 min OT (early hour ignored). FIXED 09:00–18:00 + policy (count early, cap 120, grace 5): 08:00–18:00 → 60 min OT; 07:00–20:00 → capped 120; 09:40 → beyond the 5-min grace → half-day deduction. Same through `sp_ProcessDailyAttendance` for two real employees via date overrides.
- API on a throwaway DB clone (dropped afterwards): settings + defaults, saving the OT cap, policy create / duplicate / negative / owner-only edit, FLEXI validation and creation, simulator with a policy.

### Things to know
- Kept as UI shortcuts (not used in calculations): the preset chips on the policy page (e.g. 15/30/45/60 mins) and the stepper increments (5 min, 0.5 h). Say if you want them removed or made configurable.
- Text labels still fixed in the monthly view: "Biometric (Main Gate)" / "Biometric Terminal" when the device name is missing.
- The legacy Leave & Holidays "shifts" tab (`LeaveShiftsConfigTab`) still has its own create form with grace 15; that path writes only superseded columns.
- `overtimeMaxMinutesPerDay` on a shift policy can only lower/raise the cap, not remove a company cap (blank = inherit).

### Next: Step 4 — TA General Settings
Scope for review before coding: on the same `/masters/shift-grace-policies` page add the remaining General Settings from the PDF — attendance cut-off day (e.g. 23 → period 24th–23rd), week start day, back-dated schedule edit days, close-month impact, mobile app settings (selfie, remarks on clock in/out, geofence mode, login with specific mobile); the monthly late allowance then counts per attendance period instead of calendar month.

---

## 9. Step 4 — completion notes (2026-10-03)

### Delivered (each setting has a real effect)
| Setting (policy page → General Settings) | Effect |
|---|---|
| **Attendance cut-off day** (1–31, or none = calendar month) | `fn_AttendancePeriod` gives the period for any date; the period ending in a month is that month's payroll. Used by: payroll (attendance + unpaid leave read for the period; payroll approval locks exactly that period), the monthly late allowance (sequence restarts at the period start), the regularisation quota. Shorter months end on their last day. |
| **Week start day** | Stored (Monday by default); consumed by weekly OT limits (Step 5) and the roster grid (Step 6). |
| **Back-dated schedule changes (days)** | Shift assignment, bulk assignment, date override create / cancel may not start earlier than today − N days. Blank = no limit. |
| **Ask for remarks during clock in / out** | Web clock in / out is refused without a remark (portal card shows the reason). |

DB (`docs/db_changes.sql`, block "Policies Step 4"): `company_attendance_policies` + `cutOffDay`, `weekStartDay`, `backDatedScheduleEditDays`, `remarksOnClockRequired` (CHECK constraints; defaults keep current behaviour), `fn_AttendancePeriod`, `sp_ProcessDailyAttendance` (period-based late recount). Applied on local; re-run safe. Reset Defaults also covers the new fields (read from the column defaults).

### Verification done
- `dotnet build` ✔, `dotnet test` 42/42 ✔, `tsc --noEmit` ✔.
- `fn_AttendancePeriod` for cut-off none / 1 / 23 / 30 / 31 across month ends, February and year end: periods are contiguous with no gap or overlap (e.g. 23 → 24 Aug–23 Sep = September; 30 → February ends on the 28th, March starts on the 1st).
- On a throwaway DB clone (dropped afterwards): invalid cut-off rejected; settings saved and read back; employee web clock-in refused without remark, accepted with one; assignment / date override 30 days back refused with a 7-day limit, 3 days back accepted; late sequence on 24 Sep restarts at 1 with cut-off 23; payroll for September approved with cut-off 23 locked exactly 24 Aug – 23 Sep.

### Things to know
- Recurring weekday overrides have no start date, so the back-dating limit cannot apply to them (they affect past days too).
- The monthly ESS attendance view still shows calendar months (display only).

### Decisions received → built in Step 4b (§11)

### Next: Step 5 — Overtime management
Scope for review before coding: OT types OT1–OT4 as rows (applies on working day / weekly off / holiday, rate, needs authorization, pay authorized hours irrespective of actual, payroll component), per-day OT split by type, authorization through the attendance approval matrix, max OT limits per employment type per period / week (uses the cut-off period and week start day from Step 4), payroll reading OT per type.

---

## 10. Step 5 — completion notes (2026-10-03)

### Delivered
| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 5") | `overtime_types` (per company: covers working day / weekly off / holiday, rate multiplier, requires authorization, pay authorized irrespective of actual), `attendance_overtime` (one row per attendance day with overtime: actual / authorized / payable minutes, status AUTO / PENDING / APPROVED / REJECTED), `overtime_limits` (per employment type or all; TOGETHER / SEPARATE; max per attendance period and/or per week). Seeded OT1 (working day, rate = company `overtimeMultiplierStandard` = 1.50) and OT2 (weekly off + holiday, rate = `overtimeMultiplierHoliday` = 2.00) for every company, **without authorization** so nothing waits for approval until you switch it on. `sp_SyncAttendanceOvertime` keeps the rows in step with daily processing (called at the end of `sp_ProcessDailyAttendance`). Backfill: all 7,823 existing overtime days (870,495 min) got their rows. Permissions `overtime.view` / `overtime.authorize` / `overtime.settings.manage` (granted to roles with `attendance.view` / `attendance.approve` / `attendance.manage`), menu Attendance → Overtime. Applied on local; re-run safe. |
| Rules | The day kind (holiday → weekly off → working day) picks the one active type covering it; two active types cannot cover the same kind of day. Payable = actual (AUTO); 0 (PENDING / REJECTED); authorized, or the actual if lower, unless "pay authorized irrespective of actual" (APPROVED). Re-processing keeps APPROVED / REJECTED decisions (restarts only if the day's type changes). Payroll-locked days are never touched. |
| Limits | Checked when authorizing: payable overtime already counted (AUTO + APPROVED) in the attendance period (cut-off day, Step 4) and in the week (week start day, Step 4), all types together or per type. The Overtime page refuses above the limit; an approval from the approvals inbox is capped to what is left (noted in the remarks). |
| Authorization | Overtime page (`/attendance/overtime`): date / status filter, authorize with editable minutes (partial authorization), reject. Approvals inbox: pending OT appears with the same routing as regularisations (employee's line manager, or HR). |
| Payroll | Reads payable minutes per type for the attendance period and pays each type at its own rate (`OvertimeLines` in the calculation input); the old single statutory multiplier is used only when no lines are passed. |
| UI | Overtime page with tabs Authorization / Overtime Types / Limits (SideDrawer forms). |

### Verification done
- `dotnet build` ✔, `dotnet test` 42/42 ✔, `tsc --noEmit` ✔.
- Backfill totals match `attendance_records` exactly (OT1 808,185 + OT2 62,310 = 870,495 min, 0 days without a row).
- On a throwaway DB clone (dropped afterwards): overlapping OT type refused; OT1 switched to "authorization required" → re-processing 29 Sep turned 549 days PENDING (payable 0); limit 120 min/period → page authorization refused with the limit message, inbox approval capped to 0 with remark; without limit: partial authorization 90 of 225 → payable 90; reject → payable 0; re-processing kept both decisions; September payroll paid 21 h (= 1,260 payable OT1 min) at ×1.5.

### Things to know
- **Rest-day overtime now pays at OT2's rate (2.00)** instead of the single statutory multiplier — that is the company's stored holiday rate; change it on the Overtime Types tab if needed.
- Payroll's hourly rate is still `basic ÷ 26 ÷ 8` (existing code) — flagged for the payroll settings work.
- The approvals dashboard counters (`TotalPending`, overdue) do not include overtime yet; the inbox list does.
- Excess-hours email notification (PDF) is part of Step 11 (needs an email-template master).
- Superseded and unused: `company_attendance_policies.requireOtApproval`, `overtimeMultiplierStandard/Holiday` (now only the seed source).

### Next: Step 6 — Roster + General / Roster uploads
Scope for review before coding: `shift_rosters` (company, month, DRAFT → PUBLISHED → LOCKED) and `shift_roster_days` (employee, date, shift or weekly off); `fn_EmployeeDaySchedule` precedence 2 (published roster day); Roster Planner grid (`/shifts/roster`, weeks from the week start day); Upload drawer with two templates — General shift (Employee Id, Shift Code, Effective From/To) and Roster (`docs/ShiftRosterReport.xlsx` layout: numeric or text Employee Id, any sheet name, `Day G6(…)` / `Half Day G16(…)` / `Weekly Off` cells) — validate → preview with row errors → commit, same pattern as the bulk employee upload; export in the same layout.

---

## 11. Step 4b — close month, clock-in access, login with specific mobile (2026-10-03)

Your answers: close month = **end of the month**; clock in/out = **one click on web and mobile, only for employees given access** (no selfie); login from a specific phone = **a flag; when on, the phone is recognised**.

| Setting | Built as |
|---|---|
| **Close Month** (policy page → General Settings) | `closeMonthMode`: *When payroll is approved* (previous behaviour, default) or *At the end of the attendance period*. In the second mode every attendance period that has ended (cut-off day, Step 4) is closed: `sp_ProcessDailyAttendance`, `sp_SyncAttendanceOvertime` and the monthly view leave its days untouched, and regularisation submit / approval, manual adjustment create / decide and overtime decisions are refused. Derived from the dates (`fn_IsAttendanceDateClosed`), no scheduled job. Payroll approval still marks the days as payroll-locked in both modes. |
| **Clock-in access** (Attendance → Clock-in Access, new page) | `employment_details.clockInAllowed` (default **off**). The portal clock in / out card appears only for employees with access (`GET /api/attendance/clock-access/me`) and the server refuses clock in / out for others. HR grants or removes access per employee or for a selection (search, department and access filters). |
| **Login with specific mobile** (policy page → General Settings) | `loginWithSpecificMobile` (default off). When on, a sign-in from a phone (browser user agent) must come from the employee's registered phone: the first phone used is registered (`user_devices`), any other phone is refused with a message; computers are not affected. The browser keeps a random device id and sends it at login. HR sees the registered phone and can reset it on the Clock-in Access page. |

DB (`docs/db_changes.sql`, block "Policies Step 4b" + Clock-in Access menu row): `company_attendance_policies.closeMonthMode` (CHECK), `loginWithSpecificMobile`, `employment_details.clockInAllowed`, `user_devices`, `fn_IsAttendanceDateClosed`, the three procedures above. Applied on local; re-run safe.

### Verification done
- `dotnet build` ✔, `dotnet test` 42/42 ✔, `tsc --noEmit` ✔. Rolled-back SQL check: with cut-off 23 on 3 Oct, days up to 23 Sep are closed, 24 Sep onward open; processing a closed day changes nothing.
- On a throwaway DB clone (dropped afterwards): invalid close mode refused; regularisation and overtime decision on 20 Sep refused (period closed), processing 20 Sep left it unchanged, regularisation on 30 Sep accepted; employee without access: `me` = no access, clock-in refused; after HR grants access: clock-in works; specific mobile on: first phone registered, another phone refused with the message, computer allowed, same phone allowed, HR reset → the new phone registers.

### Things to know
- **Nobody has clock-in access yet** — on local 3 employees used web clock-in in the last 30 days; grant access on Attendance → Clock-in Access.
- Turning on *end of the attendance period* closes **all** past periods at once (all history before the current period).
- Recognising a phone relies on the browser user agent and a device id stored in the browser; clearing the browser's storage counts as a new phone (HR reset needed).
- ⚠️ Found, not changed (security): the sign-in screen (`modules/auth/AuthView.tsx`) retries a failed login with the hardcoded passwords `KingPwd!2032` / `Admin@123`. Recommend removing it.

---

## 12. Step 6 — completion notes (2026-10-03)

### Delivered
| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 6") | `shift_rosters` (company × calendar month, DRAFT / PUBLISHED / LOCKED), `shift_roster_days` (employee × date: SHIFT with a shift, or WEEKLY_OFF; one row per employee per date); `fn_EmployeeDaySchedule` precedence **date override → published roster → weekday override → assignment → employment shift → company default → UNSCHEDULED** (a rostered shift is worked even on a weekday the shift's own rule marks off); permissions `shifts.roster.view / manage / publish` (from `shifts.view / assign / manage`). Applied on local; re-run safe. |
| Roster Planner (`/shifts/roster`, tab "Roster Planner") | Month × employees grid (search, department, paging). Bold = roster entry, grey = what attendance uses without one. Weeks split on the company week start day. Click a cell → shift / Weekly Off / Clear. Draft → **Publish** (attendance starts using it; processed days re-calculated) → **Back to draft** / **Lock** (no more edits or uploads). The old bulk assignment stays as the second tab. |
| Rules on every change | Shift must be enabled for the employee's company; dates in a closed attendance period (Step 4b) or earlier than the back-dating limit (Step 4) are refused; locked rosters refuse changes; edits on a published roster re-process the affected days. |
| Roster upload | Layout of `docs/ShiftRosterReport.xlsx`: any sheet name, `Employee Id` as number or text, `Shift Date-dd-mm-yyyy` columns of one month, cells as code (`G6`), shift name (`Day G6 (…)`), label (`Half Day G16(09:00 am to 02:00 pm)`, `5:30 pm` accepted) or `Weekly Off`; blank = unchanged. **Check → Save**, saved only when every row is valid (like the bulk employee upload). From a **group workspace** a sheet can mix companies: each employee is matched across the group (an Employee Id found in two companies is flagged) and goes to its own company's roster (draft). |
| General shift upload | `Employee Id`, `Employee Full Name`, `Shift Code`, `Effective From`, `Effective To` (optional) → the employee's general shift (same path as the assignment drawer: previous assignment closed, past days re-processed). Template download = all employees with their current shift. |
| Export | Same layout as the sample (roster entry, or the current schedule when not rostered); the exported file uploads back without changes. |
| Config | `appsettings.json` → `ShiftUploads:MaxFileBytes` (upload size limit) — **add this key on the VM's appsettings**. |

### Verification done
- `dotnet build` ✔, `dotnet test` 42/42 ✔, `tsc --noEmit` ✔.
- Rolled-back SQL: a DRAFT roster does not change the schedule; PUBLISHED wins (71247: 1 Aug G16 09:00–14:00, 2 Aug weekly off, 3 Aug G10).
- API on a throwaway DB clone (dropped afterwards), with **your sample file**: LCH workspace → 11 rows refused (employees of other companies); group workspace → 22 / 23 rows valid, all 682 cells understood, only `DARWINTEST` (not an employee) flagged; after removing that row the commit created draft rosters for 5 companies (LCH 372 cells); publish → schedule follows the roster; export (2,139 employees) re-validates cleanly; cell edit on the published roster re-processed the day (→ WEEK_OFF); lock → edits and uploads refused; general upload: unknown shift `G999` refused with nothing saved, fixed file saved the assignment.

### Things to know
- The grid, publish / lock and export work per company; in a group workspace they ask you to pick a company (uploads work in both).
- **Not checked in a browser** (no browser available in this session): the new screens are type-checked and their API calls tested; please click through `/shifts/roster` once.
- Roster months are calendar months (as in the requirement); attendance periods follow the cut-off day.

### Next: Step 7 — Holiday calendars
Scope for review before coding: `holiday_calendars` (group-level, per country / year), assigned to companies and optionally locations (`company_holiday_calendars`); `holidays.calendarId` with a backfill from today's per-company holidays; `fn_EmployeeDaySchedule` holiday lookup through the employee's company + location calendars; holiday worked → OT2 already handled (Step 5); UI on `/leave-holidays`.

---

## 13. Step 7 — completion notes (2026-10-03)

### Delivered
| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 7") | `holiday_calendars` (group-level, owner company, code unique per group, country, active flag); `company_holiday_calendars` (calendar → whole company when `locationId` is NULL, or → one location); `holidays.calendarId` (NOT NULL, FK). **Backfill**: every company that has holidays gets a calendar `<CODE>-HOL` "<company name> Holidays" assigned to the whole company, so existing holidays keep applying to exactly the same employees. Unique key moves from (companyId, date) to (calendarId, date) — a national and a county calendar may share a date. `fn_EmployeeDaySchedule` now takes holidays from the active calendars assigned to the employee's company **plus** those assigned to the employee's own location (`employment_details.locationId`); everything else unchanged from Step 6. Applied on local; re-run safe. |
| Holiday rules (`HolidayCalendarService`) | Only the calendar's owner company changes the calendar and its holidays (others see it read-only). Assignments: companies of the calendar's group the user can access; a location must belong to its company. A holiday date (observed date when set) in a **closed attendance period** or earlier than the **back-dating limit** (General Settings) of any company using the calendar is refused. Adding / changing / deleting a holiday, (un)assigning or (de)activating a calendar re-processes the affected past days (from the back-dating limit, or the start of the current attendance period when no limit is set). A calendar is deleted only after its assignments are removed. |
| Copy to next year | "Copy recurring holidays" (in the calendar drawer): holidays marked *Annual Recurring* copied to the same day/month of another year; dates that already have a holiday and non-recurring ones are skipped and listed. Observed dates are set to the date — adjust weekend shifts by hand. Moveable holidays (Easter, Eid) should not be marked recurring. |
| UI (`/leave-holidays` → Public Holidays) | New **Holiday Calendars** card (code, country, owner, applies to, holiday years, status; click a row to filter the holiday list) + calendar drawer (owner company in a group workspace, code, name, country from the Countries master, "Applies to" company / location tick list, copy-year). Holiday list shows the calendar; edit/delete only for calendars you own. Add Holiday asks for the calendar (instead of the company); classification is optional. The **"Holiday Calendar" header button** now opens this tab — the old drawer was a hard-coded 2026 Kenya list (mock data) and is replaced. Removed static defaults ("Public Holidays Act Cap 110", "2026-01-01", "ht-national" fallback that pointed to a non-existent type) and the "Statutory Paid Holiday" tick box that was never saved (paid / unpaid comes from the classification). |
| API | `GET/POST /api/holiday-calendars`, `PUT/DELETE /api/holiday-calendars/{id}`, `POST /api/holiday-calendars/{id}/copy-year`, `GET /api/holiday-calendars/targets`; `/api/config/holidays` unchanged routes, now with `calendarId` (required on create, optional filter on list). Permissions: the existing **`leave.view` / `leave.manage`** (already used for holidays) — no new permission or menu needed. |
| Dashboards | Leave & Holidays metrics and the home "upcoming holidays" now count holidays of calendars assigned to the workspace's companies. |

### Verification done
- `dotnet build` ✔, `dotnet test` **49/49** ✔ (7 new `HolidayCalendarTests`: assignments, wrong location, owner-less create in group workspace, code unique per group, owner-only edits, date unique per calendar, copy-year, delete guard), `tsc --noEmit` ✔.
- Throwaway DB clone (dropped afterwards) with seeded holidays: backfill made `AFRIHOSP-HOL` (2 holidays) and `AFRICARE-HOL` (1), re-run changed nothing; schedule function: company calendar → all employees, a location calendar → only employees of that location, no duplicated rows.
- Real service against the clone: holiday added for today → 51 Afrihospital attendance rows re-processed to `HOLIDAY`; deleted → back to `ABSENT`; duplicate date refused; copy 2026 → 2027 copied 2, skipped the non-recurring one; group workspace lists all group calendars.
- Local DB: schema applied only (local has no holidays, so nothing was backfilled or changed).

### Things to know
- **Not checked in a browser** — please open `/leave-holidays` → Public Holidays once: create a calendar, assign it, add a holiday.
- On the VM the backfill names calendars `<COMPANY CODE>-HOL`; afterwards you can build one shared national calendar, assign it to all companies, and retire the per-company copies (delete needs the assignments removed first).
- Holiday worked → OT2 (Step 5) works unchanged, since it reads `isHoliday` from the schedule function.

### Next: Step 8 — Leave master + eligibility
Scope for review before coding (§3.10 / §4 row 8): `leave_types` extra fields from the reference PDF, `leave_type_eligibility` (gender, department, category, division, position, designation, location; empty = all), copy a leave type to other companies of the group, and removal of code-seeded leave types / hard-coded legal text.

---

## 14. Step 8 — completion notes (2026-10-03)

### Delivered
| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 8") | `leave_types` + Phase-1 fields: `shortName`, `colorCode`, `countWeekOffAsLeave`, `countHolidayAsLeave`, `halfDayWeekdays`, `ignoreWeekdays`, `restrictHalfDay`, `allowNegativeBalance`, `carryForwardType` / `Value` / `ValidTillMonth`, `monthlyAllocation`, `accrualOn`, `prorateOnJoining`, `minApplicationDays`, `maxApplicationsPerMonth`, `minGapBetweenApplicationsDays` (existing `carryOverDays > 0` → carry forward DAYS). `leave_type_eligibility` (DEPARTMENT, LOCATION, JOB_TITLE, GRADE, EMPLOYMENT_TYPE, COST_CENTRE — values of one criterion are alternatives, two criteria must both match, none = everyone; gender stays `genderRestriction`). `countries.statutoryLeaveAct` and `leave_types.legalBasis` **backfilled with the texts that were hard-coded** (KE / TZ / AE / IN / GB), so screens show the same wording. `fn_EmployeeEligibleLeaveTypes(@EmployeeId)` — one eligibility rule for SQL and the API. `sp_GetEmployeeLeaveUtilization` reads the act / legal basis from data, lists and creates balances only for eligible types (now also for leave types added later in the year), no "Kenya" / "Medical Services" defaults. Applied on local; re-run safe. |
| Leave Types screen (`/leave-holidays` → Leave Types) | New drawer: General (code, name, short name, colour, days, pay, gender, legal basis, active), **Application rules**, **Day counting** (Step 9), **Accrual & carry forward** (Step 10), **Who may use it** (tick lists from the leave type's own company), **Copy to other companies** (group workspace; settings without eligibility, existing codes skipped). Table shows colour / short name, carry-forward type and "Who". |
| Applying leave (all apply screens use `/api/leave/apply`) | Refused with a clear message when: not eligible (gender / eligibility), leave type inactive, end before start, notice not met (**HR / admins recording leave are exempt**), service shorter than `minServiceDays` (from `hireDate`), fewer days than `minApplicationDays`, more than `maxApplicationsPerMonth`, closer than `minGapBetweenApplicationsDays` to another request of the type, **overlapping any active request**, or **more days than the balance** unless `allowNegativeBalance`. Errors return 400 and the drawers show them. Leave lists in the apply drawers show eligible types only. |
| Removed static values | Code-seeded leave types (a company without leave types now gets none — set up or copy); per-country legal texts in `LeaveService` and the procedure; "Kenya" / 🇰🇪 / act fallbacks in DTOs, portal and leave screens; "1.75 days per month" note in the apply drawer; the C# copy of the utilization procedure that ran when the procedure failed (errors now surface). |
| API | `GET /api/config/leave-types/eligibility-options?companyId=`, `POST /api/config/leave-types/{id}/copy`; create / update take the new fields + `eligibility`. Permissions: existing `leave.view` / `leave.manage`, `leave.apply` — nothing new. |

### Verification done
- `dotnet build` ✔, `dotnet test` **56/56** ✔ (7 new `LeaveTypeTests`: fields + eligibility saved, values of another company / unknown criteria refused, accrual / carry-forward / weekday validation, update replaces eligibility and keeps the code, copy to companies without eligibility + skip existing + workspace check), `tsc --noEmit` ✔.
- Throwaway DB clone (dropped afterwards): block run twice; eligibility function — men don't get Maternity, two criteria must both match, one criterion matches on any ticked value; utilization procedure returns the same act / legal texts as before.
- Real `LeaveService` on the clone as an ordinary employee: refused Maternity (male), Sick Full (department rule), Study tomorrow (7-day notice), Paternity (service), 30 days Annual (21 balance), end before start, overlap, max 1 per month, 10-day gap; accepted valid Annual / Compassionate / Study requests.
- Local DB: only the schema + the text / carry-forward backfill (snapshot of the 42 leave-type rows taken before); no leave requests touched.

### Things to know
- **Not checked in a browser** — please open `/leave-holidays` → Leave Types, edit a type, and try an apply that breaks a rule.
- **9 of 15 companies have no leave types** on local (earlier they would have been auto-created with Kenyan defaults on first use). Use *Copy to other companies* from the group workspace.
- Still to come: day counting still skips Sat/Sun only — **Step 9** switches it to the employee's schedule and uses the day-counting fields; accrual / carry-forward fields are used in **Step 10**.
- Stored but not enforced yet: *Proof document required* (no upload on the apply screens) and *No half-day applications* (half-day leave arrives with Step 9).
- The statutory act text has no screen: it lives in `countries.statutoryLeaveAct`. Tell me if you want it on a Countries master page.
- "Available annual days" (KPI) still finds the annual type by code `ANNUAL`; the leave workflow help drawer still describes the Kenyan act (help text, not data).

### Next: Step 9 — Leave day counting
Scope for review before coding: `sp_CalculateLeaveDays(@EmployeeId, @From, @To, @LeaveTypeId)` over `fn_EmployeeDaySchedule` (weekly off counted only with `countWeekOffAsLeave`, holiday only with `countHolidayAsLeave`, `halfDayWeekdays` = 0.5, `ignoreWeekdays` = 0, unscheduled days reported); used by apply, approve and the utilization procedure instead of Sat/Sun; per-day breakdown in the apply drawers; half-day applications (honouring `restrictHalfDay`).

---

## 15. Step 9 — completion notes (2026-10-03)

### Delivered
| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 9") | `fn_LeaveDayBreakdown(@EmployeeId, @LeaveTypeId, @From, @To, @StartDayPortion, @EndDayPortion)` — one row per date from `fn_EmployeeDaySchedule` (roster, shift, weekday rules, holiday calendars): **IGNORED** (weekday in `ignoreWeekdays`) 0 · **UNSCHEDULED** 0 · **WEEKLY_OFF** 1 only with `countWeekOffAsLeave` · **HOLIDAY** 1 only with `countHolidayAsLeave` · **WORKING** 1 (a HALF_DAY schedule day 0.5); then `halfDayWeekdays` → 0.5 and a half first / last day → 0.5. Planned as `sp_CalculateLeaveDays`; built as an inline function so the procedures below can use it too. `leave_requests.startDayPortion` / `endDayPortion` (FULL / FIRST_HALF / SECOND_HALF). `sp_ProcessDailyAttendance`, `sp_GetEmployeeMonthlyAttendance` and `sp_GetEmployeeLeaveUtilization` count a date as leave only when it counts (> 0) — **weekly offs and holidays inside a leave keep their own status**; the Sat/Sun rule is gone. Applied on local; re-run safe. |
| Apply (all three apply drawers) | Day-by-day count shown while filling the form (date, working day / weekly off / holiday — name / not counted / no shift, shift code, counted days, total) from `POST /api/leave/preview-days`. **Half-day leave**: one day = full / first half / second half; several days = may start in the second half and end in the first half; refused when the leave type has *No half-day applications*. Refused when the dates hold no leave days, or when a date has **no shift scheduled** (HR must assign one; nothing is invented). The balance check uses the real count. |
| Approval (Leave screens and the Approvals inbox — both paths) | One shared routine: days are counted again (the schedule may have changed), balance pending → taken with the real count (the Approvals inbox also stopped ignoring carried-forward days in "remaining"), future leave days get an ON_LEAVE placeholder, past days are re-processed by `sp_ProcessDailyAttendance` (respects punches and closed periods). |
| Employee portal dashboard | Attendance % uses the employee's own working days so far (schedule) instead of Mon–Fri with a fixed fallback of 22. |

### Verification done
- `dotnet build` ✔, `dotnet test` **59/59** ✔ (3 new `LeaveDaysTests` for the half-day rules), `tsc --noEmit` ✔.
- Throwaway DB clone (dropped afterwards): block run twice; breakdown over a week with a weekly off, a holiday and a half first day = 0 / 0 / 0.5 as expected; with *count weekly offs*, Saturday half-day and Friday ignored → 5 days exactly as configured; all 5 existing approved requests re-counted (see below); a leave over a weekly off is processed as ON_LEAVE – WEEK_OFF – ON_LEAVE; the monthly register shows leave only on counted days.
- Real services on the clone: preview Fri (2nd half) – Mon over a Sunday off = 2.5 days; apply stored 2.5 with SECOND_HALF; refused a first-half start on a multi-day leave, a Sunday-only leave, a half-day on a restricted type and an employee without a shift; approval moved 4.5 days from pending to taken, put ON_LEAVE on Fri / Sat / Mon (not the Sunday off) and re-processed the past leave.
- Local DB: schema + procedures only; leave requests unchanged (8 requests, 25 days).

### Things to know
- **Most employees work Saturdays** (one weekly off per week in the schedules), so the old Mon–Fri counting under-counted. Existing requests keep their stored days and balances; only the monthly utilization view recounts from dates. On local this differs for one request: emp-usr-001's Thu 24 – Sat 26 Sep annual leave is stored as 2 days but his schedule makes Saturday a working day, so the utilization view now shows 3. Tell me if past requests should be recounted into balances (a separate, opt-in script).
- 2 employees on local have days without any shift (`emp-wanjiku-muthoni`, `emp-MAKL-20341`); they cannot apply leave for those days until a shift is assigned.
- A half-day leave day with punches is evaluated from the punches like any other day (no split "half leave / half work" status yet).
- **Not checked in a browser** — please open an apply drawer, pick dates over a weekend, try a half day.

### Next: Step 10 — Accrual & carry-forward
Scope for review before coding: `sp_RunLeaveAccrual` — monthly credit for leave types with `accrualFrequency = MONTHLY` (`monthlyAllocation`, `accrualOn` month start / end, `prorateOnJoining` for joiners), yearly entitlement for ANNUAL types; year-end carry forward per `carryForwardType` (DAYS / PERCENT / ALL) with expiry after `carryForwardValidTillMonth`; an accrual log so each month runs once (re-run safe); a run button + history for HR, and a scheduled run.

---

## 16. Step 10 — completion notes (2026-10-03)

### Delivered
| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 10") | `fn_LeaveEntitlementAsOf(@EmployeeId, @LeaveTypeId, @AsOfDate)` — **ANNUAL**: `daysAllowed` (joiners of the year get a day-based share when `prorateOnJoining`); **MONTHLY**: `monthlyAllocation` per month credited at `accrualOn` MONTH_START / MONTH_END, joining month pro rata when `prorateOnJoining` (otherwise only if employed on the credit date), no months after termination; **NONE**: 0; rounded to the new `leave_types.accrualRoundingStep` (empty = 2 decimals). `sp_RunLeaveAccrual` (sets each eligible balance to that entitlement; lapses carried-forward days not used by the end of `carryForwardValidTillMonth` — leave uses carried days first) and `sp_RunLeaveYearEnd` (unused = entitlement + carried − taken − pending → next year per DAYS / PERCENT / ALL, rounded **down** to the step so never more than unused; only for a finished year; final once carried days have lapsed). Both **re-run safe** (they set, never add twice), have a **dry run**, and log every change in `leave_accrual_ledger` grouped by `leave_accrual_runs`. Employees without a hire date are reported (SKIPPED) and left alone. New balances (utilization procedure + API) now start at the entitlement from the function instead of a flat `daysAllowed`. Permission **`leave.accrual.manage`** (granted to roles with `leave.manage`). Applied on local; re-run safe; **nothing run** on local. |
| Leave & Holidays → **Accrual & Carry Forward** tab | Company (or all of the workspace) · **Accrual** as of a date · **Year-end** from a year into the next — each with **Preview** (nothing saved) and **Run** (confirmation); result table (search, paging) with before / after / days / note; **Run history** (paged) → drawer with every balance a run changed. |
| Leave type drawer | "Round credited days to" (rounding step) in Accrual & carry forward. |
| Scheduled accrual | `LeaveAccrualBackgroundService`: accrual as of today for every active company, every `RunIntervalHours`. **`appsettings.json` → `"LeaveAccrual": { "AutoRunEnabled": false, "RunIntervalHours": 24, "CommandTimeoutSeconds": 600 }` — add this on the VM and set `AutoRunEnabled` to `true` once you have checked a manual run.** Year-end is not scheduled (HR runs it once the year's leave is settled). |
| API | `POST /api/leave-accrual/run`, `POST /api/leave-accrual/year-end` (`leave.accrual.manage`), `GET /api/leave-accrual/runs`, `GET /api/leave-accrual/runs/{id}/entries` (`leave.view`). |

### Verification done
- `dotnet build` ✔, `dotnet test` **60/60** ✔, `tsc --noEmit` ✔.
- Throwaway DB clone (dropped afterwards), rolled-back SQL: ANNUAL 21 → 21; joined 1 Jul + prorate → 10.59 (10.50 with step 0.5); as of a date before joining → 0; MONTHLY 1.75 month end Jan–Sep → 15.75, month start Jan–Oct → 17.50; joined 16 Mar month start → 12.25 (prorate 13.15); terminated 10 Jun → 6.15; NONE → 0.
- Procedures on the clone: first LCH run 12,745 changes (balances created for employees who never opened their leave), re-run 0; a balance would have dropped 21 → 0 for an employee without employment record — fixed to SKIPPED; year-end 2025 → 2026 carried 5 of 21 unused days (DAYS 5), re-run 0; with "usable until March" the next accrual lapsed the 5 unused days (ledger: CARRY_FORWARD +5, CARRY_FORWARD_EXPIRY −5), re-runs 0, year-end after lapse 0; year-end for the current year refused.
- Real service on the clone: group-workspace preview of 15 companies in < 1 s (1,119 changes, 2 skipped); company run 300 changes, re-run 0; history + entries; another workspace cannot read a run; scheduled run of all companies recorded as SCHEDULED.
- Local DB: objects + permission only — no runs, balances unchanged (154).

### Things to know
- **First real run creates many balance rows** (on local ≈ 13,000 across companies) — these are balances employees would otherwise get the first time they open their leave. **Preview first.**
- The leave types on local are all ANNUAL, so the first run does not change any existing entitlement; monthly accrual starts once a leave type is set to MONTHLY.
- 2 employees on local have no hire date (EMP-FIN-003, EMP-MGR-004) — reported as SKIPPED until it is filled in.
- A MONTHLY employee terminated during a month still gets that month's credit (credited per month started while employed).
- Employment types still show a "leave entitlement rate" (e.g. 1.75 d/mo) that nothing uses; the leave type's monthly allocation is the rule. Tell me if that field should go.
- **Not checked in a browser** — please open Leave & Holidays → Accrual & Carry Forward and press Preview.

### Next: Step 11 — Reports + notifications
Scope for review before coding: Shift Roster Report (layout of `docs/ShiftRosterReport.xlsx`), roster vs actual attendance, overtime register (per OT type / authorization status); an email-template master (general, DB-driven) and the excess-hours notification from the TA General Settings PDF.

---

## 17. Step 11 — completion notes (2026-10-03)

### Delivered
| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 11") | `email_templates` (general email template master per company: code, name, event, subject, HTML body with `{{Placeholders}}`), `notification_settings` (per company + event: on / off, template, To / CC / BCC, also employee / line manager), `notification_log` (one row per event + reference key → each breach is emailed once; failed sends are retried). Seeded one **"Overtime limit exceeded"** template per company (editable). `sp_GetOvertimeLimitBreaches` (payable AUTO + APPROVED overtime in the current attendance period / week above the employee's limit — same choice of limit as authorization, TOGETHER or per type). `sp_ReportRosterVsActual` (schedule from `fn_EmployeeDaySchedule` against `attendance_records`, paged). Permission **`notifications.manage`** (granted to roles with `attendance.manage`); menus **Reports → Shifts & Overtime** and **Masters → Email Templates**. Applied on local; re-run safe. |
| Reports → **Shifts & Overtime** (`/reports/shifts-overtime`) | **Roster vs Actual**: per employee per day — scheduled shift / weekly off / holiday / no shift (and where it came from: roster, assignment, employment shift …) against status, in / out, worked, late, OT; result *As scheduled, Absent, Late, Half day, Worked on off day, No shift, Not processed, On leave, Upcoming*; "exceptions only"; per company (a group workspace picks one). **Overtime Register**: every overtime day by type and status (actual / authorized / payable, decided by), totals per type × status; covers all companies of the workspace. **Shift Roster**: the month's roster sheet in the `ShiftRosterReport.xlsx` layout (the Step 6 export). All three download as .xlsx. |
| Masters → **Email Templates** (`/masters/email-templates`) | List + drawer: code, name, "used for" event (shows its placeholders), subject, HTML body, live preview (sandboxed). Unknown placeholders for the event are refused. |
| Attendance → Overtime → **Excess Hours Email** | Per company: on / off, template, To / CC / BCC (validated), also line manager / employee; **Preview breaches** (nothing sent), **Send now**, **send log**. Email: subject + body rendered from the template (values HTML-encoded). |
| Scheduled check | `NotificationBackgroundService` — **`appsettings.json` → `"Notifications": { "OvertimeLimitCheckEnabled": false, "OvertimeLimitCheckIntervalMinutes": 60 }`** (add on the VM; switch on after a preview). Only companies whose notification is on are checked. |
| Other | `EmailService` accepts several To addresses (comma / semicolon). Sidebar icons `Timer` (Overtime menu from Step 5 showed a generic icon), `Mail`, `ClipboardList`. Long runs (accrual, exports, checks) no longer hit the browser's 15-second request timeout. |

### Verification done
- `dotnet build` ✔, `dotnet test` **63/63** ✔ (3 new `NotificationTemplateTests`: placeholders replaced and HTML-encoded, unknown ones kept, event placeholders), `tsc --noEmit` ✔.
- Throwaway DB clone (dropped afterwards): block run twice; breach check with a 600 min period / 240 min week limit → 430 / 597 employees, all with work email and line manager; roster vs actual Sept for 2,141 LCH employees: 64,230 rows in 2.2 s, one page of exceptions in 0.8 s.
- Real services on the clone **with a fake mail sender (no email left the machine)**: unknown placeholder, enabling without template, invalid address all refused; no limit → 0 breaches; limit 30 h → 1 breach, "notification off" until enabled; preview → would send to HR + payroll + line manager; failing mail server → FAILED, next run retried → SENT; next run → already sent; the email read "GABRIEL ANDAYI (91148, Lifecare Hospitals) has 32.5 h … above the limit of 30 h by 2.5 h"; roster vs actual paging + export (.xlsx), overtime register totals (OT1 AUTO 4,244 days …) + export, group workspace without company refused for roster vs actual, overtime register across all companies.
- Local DB: tables, procedures, 15 starter templates, permission, menus — nothing enabled, nothing sent.

### Things to know
- **Nothing is sent until** a company's Excess Hours Email is switched on with recipients **and** (for the schedule) `OvertimeLimitCheckEnabled` is true. On local there are **no overtime limits yet**, so there are no breaches.
- `appsettings.json` contains test SMTP credentials (ethereal.email); real mail needs your SMTP on the VM.
- Roster vs actual on local shows **many "Not processed" days** (Sept LCH: 38,426 of 64,230) — scheduled days without any attendance record, i.e. daily processing has not run for them. Worth a look.
- **Not checked in a browser** — please open Reports → Shifts & Overtime, Masters → Email Templates and Attendance → Overtime → Excess Hours Email.

### All steps of this plan are complete
Open items collected in the step notes: the Step 2b duplicate-shift merge script (manual, not run), the Countries screen for the statutory act text (§14), recounting past leave requests into balances (§15, opt-in), the unused employment-type "leave entitlement rate" (§16), payroll's hourly rate `basic ÷ 26 ÷ 8` (§10), and browser checks of the new screens.

---

## 18. Phase 2 — plan (2026-10-03)

The items deferred in Phase 1, in build order (smallest / most needed first; 13 is a prerequisite of 14). Same rules as
Phase 1: everything DB-driven and configurable, DB changes in `docs/db_changes.sql`, tested on a throwaway clone, one step
at a time. Where the business rule is not known yet, the **default** below is used and stays a setting.

| Step | Scope | Design | Default where the rule is open |
|---|---|---|---|
| **12. Linked leave** | A leave type that can only be used after another one is used up (Sick Half Pay after Sick Full Pay). | `leave_types.prerequisiteLeaveTypeId` (same company, no cycles); apply refuses while the prerequisite still has days (entitlement + carried − taken − pending); copy-to-companies maps it by code. | Only for employees eligible for the prerequisite type. |
| **13. Payroll day rate + slabs** | (a) Replace the hard-coded `basic ÷ 26` (unpaid leave) and `basic ÷ 26 ÷ 8` (overtime) with company payroll settings. (b) **Entitlement slabs**: days per year by completed years of service. (c) **Pay slabs**: pay % by day number of a leave type in the year (e.g. sick days 1–30 full, 31–45 half). | (a) company setting: day-rate basis (BASIC / GROSS), divisor (fixed days per month, or calendar / working days of the month), hours per day for the hourly rate. (b) `leave_type_entitlement_slabs` (fromYears, toYears, days) used by `fn_LeaveEntitlementAsOf`. (c) `leave_type_pay_slabs` (fromDay, toDay, payPercent); payroll deducts (100 − %) of the day rate per leave day, and counts leave days of the payroll month only (today a request spanning two months is deducted in full). | (a) BASIC ÷ 26, 8 h — today's numbers, now as data. |
| **14. Leave encashment** | Pay unused leave in cash at year end and / or on exit. | `leave_types`: encashable, encash on (YEAR_END / EXIT / BOTH), max days, min balance to keep; `leave_encashments` (request → approval → payroll line, balance reduced); year-end run (Step 10) offers encashment of what is not carried forward; exit: on termination. | HR creates / approves; uses the Step 13 day rate. |
| **15. Comp-off** | Time off instead of overtime pay for work on a weekly off / holiday. | `overtime_types.compensation` (PAY / COMP_OFF / EMPLOYEE_CHOICE); a comp-off leave type (`isCompOff`); worked minutes → days by configurable slabs (e.g. ≥ 4 h = 0.5, ≥ 8 h = 1); `comp_off_credits` with expiry days; credited when the overtime is AUTO / APPROVED; comp-off days are not paid as overtime. | Expiry and slabs per company setting; OT2 stays PAY until switched. |
| **16. Night-shift allowance** | Allowance for hours worked at night. | Company setting: night window (e.g. 22:00–06:00), minimum night minutes, amount per night or per hour, pay component; processing stores night minutes per attendance day; payroll line per employee per period; eligibility by employment type / department (Step 17 criteria). | Off until configured. |
| **17. Shift restriction by criteria** | Only certain shifts for certain departments / locations / job titles / grades / employment types. | `shift_eligibility` (same model as leave eligibility, Step 8); enforced in assignment drawer, roster grid, both uploads; dropdowns show allowed shifts only. | No rows = everyone (nothing changes until set). |
| **18. Clock-in selfie / geofence** | Optional photo and / or location check on one-click clock-in. | Company flags `selfieRequired`, `geofenceMode` (OFF / WARN / ENFORCE); `locations.latitude / longitude / radiusMeters`; clock-in sends photo (blob storage) + coordinates; stored on the punch; HR view of photo + map distance. | Both OFF (today's one-click behaviour). |

**Questions (answer any time; the defaults above apply until then):**
- Q12 — Which leave types are linked (e.g. SICK_HALF after SICK_FULL)? Set per leave type in the drawer.
- Q13 — Day rate for deductions / encashment: basic or gross; ÷ 26, ÷ calendar days or ÷ working days?
- Q14 — Is unused leave paid on exit, at year end, or both? Any maximum?
- Q15 — Who gets comp-off (all, or only some employment types)? Hours for half / full day? Expiry?
- Q16 — Night window, minimum hours, amount (per night / per hour), and who is eligible?

---

## 19. Step 12 — completion notes (2026-10-03)

### Delivered
| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 12") | `leave_types.prerequisiteLeaveTypeId` (self FK). Schema only — nothing linked. Applied on local; re-run safe. |
| Leave type drawer | Application rules → **"Usable only after (linked leave)"**: a leave type of the same company. Refused: linking to itself, to another company's type, or a chain that loops (A after B after A). Table shows "after … is used up". |
| Apply | Refused while the linked type still has days (entitlement + carried − taken − pending), with the days left in the message; employees not eligible for the linked type are not restricted. |
| Copy to companies | The copy links to the target company's leave type with the same code (not linked if it has none). |

### Verification done
- `dotnet build` ✔, `dotnet test` **65/65** ✔ (2 new: self / other company / loop refused; copy maps the link by code), `tsc --noEmit` ✔.
- Throwaway clone (dropped): SICK_HALF linked to SICK_FULL → applying Sick Half Pay refused "… only after Sick Leave (Full Pay) is used up (30 day(s) of it left)"; after the full-pay balance was used up → accepted.

### Things to know
- Nothing is linked on local; set it per leave type (e.g. Sick Leave (Half Pay) → after Sick Leave (Full Pay)).
- **Not checked in a browser.**

### Next: Step 13 — Payroll day rate + leave entitlement / pay slabs
Scope (§18): company payroll settings for the day rate (BASIC / GROSS, divisor, hours per day — today's `÷ 26`, `8 h` become the defaults as data) used by unpaid leave and overtime; unpaid leave counted only for the payroll month's days; entitlement slabs by years of service; pay slabs by leave day number (e.g. sick days 1–30 full, 31–45 half). **Q13** (day rate basis / divisor) answers welcome before or after — defaults keep today's numbers.

---

## 20. Step 13 — completion notes (2026-10-03)

### Delivered
| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 13") | `company_payroll_settings` (day rate basis BASIC / GROSS, divisor FIXED / CALENDAR_DAYS / WORKING_DAYS, fixed divisor, hours per day — **SQL defaults = today's BASIC ÷ 26, 8 h**; row created on first use), `leave_type_entitlement_slabs` (days by completed years of service; `leave_types.serviceSlabBasis` YEAR_START / AS_OF_DATE), `leave_type_pay_slabs` (pay % by leave day number of the year), `fn_EmployeeWorkingDays`, `fn_LeavePayDays` (counted leave days in a period, day number, pay %), `fn_LeaveEntitlementAsOf` re-emitted with the slabs. Applied on local; re-run safe; no data changed. |
| Payroll run | Every `basic ÷ 26` (absences, half days, late penalties, leave) and `basic ÷ 26 ÷ 8` (overtime) now come from the settings — **defaults give exactly the old numbers** (same rounding). Leave is deducted **only for its days inside the payroll period** (before: the whole request in each period it touched) and per pay %: unpaid types 0 %, pay slabs as set, otherwise 100 %. Payslip line: "Sick Leave (Full Pay) (2 day(s) at 50% pay)". WORKING_DAYS with an employee who has no scheduled day → the run stops with that employee named (400, no silent fallback). |
| Payroll page | **Day Rate** button → drawer (basis, divisor, days, hours per day, worked example). The run drawer's "Standard Working Days" input (sent but never used by the server) is replaced by a pointer to the Day Rate settings. Simulator: uses the settings when no hourly rate is given; shows errors (before they were only logged). |
| Leave type drawer | Accrual → **days per year by years of service** (ANNUAL types; years at year start or on the accrual date). Application rules → **pay by leave day** (from / to day, pay %). Both copied with "Copy to other companies". Overlaps / wrong ranges refused. |
| Calculation engine | The six `basic ÷ 26 ÷ 8` fallbacks are gone: overtime without an hourly rate is an error. |

### Verification done
- `dotnet build` ✔, `dotnet test` **70/70** ✔ (5 new: day rate FIXED / GROSS / CALENDAR / WORKING, **exactly the old formulas**, no working days refused, engine needs the hourly rate for overtime; slabs validated / saved / copied), `tsc --noEmit` ✔.
- Throwaway clone (dropped): rolled-back SQL — employee hired 13 Nov 2020: slab 0–4 y = 21, 5+ y = 24 → 2025 (4 y at year start) 21, 2026 24, 2025 "as of 31 Dec" 24; sick leave 27 Aug – 2 Sep with 1–2 at 100 % / 3+ at 50 % → September shows days 5–6 at 50 %; 26 scheduled working days in September.
- **Real September payroll run of MAKL (72 payslips)** on the clone: defaults → 20 absences × 2,884.62 (= 75,000 ÷ 26) = 57,692.40, 2 half days 2,884.62, overtime 8.5 h × 75,000 ÷ 26 ÷ 8 × 1.5 = 4,597.36 (old formula), sick leave with a 50 % slab → "2 day(s) at 50% pay" 2,884.62; switched to CALENDAR_DAYS → 2,500 per day (÷ 30) and overtime 3,984.38.
- Local DB: schema only — no settings rows, no slabs, payroll runs untouched.

### Things to know
- **Q13 still open**: until you change Payroll → Day Rate, every company keeps BASIC ÷ 26 and 8 h.
- Leave pay % counts days in the **calendar leave year**; "day N" is the N-th counted day of that leave type in the year.
- Flagged, not changed: an employee with no salary and no grade is paid a **hard-coded 75,000** (`PayrollService`); the statutory config's `WorkingDaysPerMonth` (26) is stored but not used by payroll. Tell me how you want these handled.
- **Not checked in a browser.**

### Next: Step 14 — Leave encashment
Scope (§18): leave type encashment rules (encashable, on YEAR_END / EXIT / BOTH, max days, minimum balance to keep); `leave_encashments` (request → approval → payroll line at the Step 13 day rate, balance reduced); year-end run (Step 10) offers encashment of days not carried forward; on exit for terminated employees. **Q14** (exit / year end / both, maximum) answers welcome — defaults: HR creates and approves, both allowed per leave type.

---

## 21. Step 14 — completion notes (2026-10-03)

**Answer received (Q14):** carry forward up to 9 days and encash up to 18 days — numbers differ per company, so both are settings per leave type (and per company, since leave types are per company).

### Delivered
| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 14") | `leave_types`: `isEncashable`, `encashOnYearEnd` + `encashMaxDaysYearEnd`, `encashOnExit` + `encashMaxDaysExit` (empty = no limit). `leave_encashments` (employee, leave type, balance year, YEAR_END / EXIT, days, PENDING → APPROVED (payroll month) → PAID / REJECTED, day rate, amount, payroll run). `sp_RunLeaveYearEnd` re-emitted: **carry forward first (e.g. 9), then propose encashing up to the maximum (e.g. 18) of what is left; the rest lapses**. `sp_RunExitEncashment`: employees with a termination date → balance in the year of leaving (entitlement up to the leaving date + carried − taken − pending), up to the exit maximum. Both re-run safe: PENDING proposals follow the balance, decided ones are kept. Applied on local; re-run safe; nothing enabled. |
| Leave type drawer | **Encashment** section: "Unused days can be paid in cash", at year end (max days), when the employee leaves (max days). |
| Leave & Holidays → **Encashment** tab | Exit run (Preview / Create proposals); list (status, year end / exit, search, paging) with **Approve into a payroll month** or **Reject** (remarks); shows leaving date, inactive employees, and the paid day rate × days = amount. Year-end proposals appear in the year-end preview / result of the Accrual & Carry Forward tab ("Encashment proposed"). |
| Payroll | Approved encashments of the payroll month are paid as an earning ("Leave Encashment – Annual Vacation Leave (18 day(s), exit)") at the employee's **day rate (Step 13)**, taxable like other pay; approving the payroll month marks them **PAID**. A month whose payroll is already approved cannot take an encashment; an inactive employee cannot be approved (payroll pays active employees only). |

### Verification done
- `dotnet build` ✔, `dotnet test` **71/71** ✔ (new: encashment needs a timing, non-negative maximums, switching off clears it, copied to other companies), `tsc --noEmit` ✔.
- Throwaway clone (dropped), rolled-back SQL: 30 unused days → **carry 9, encash 18** (3 lapse); after 4 more days taken the PENDING proposal became 17; once approved, a re-run kept it; exit run: leaver with 16 days left → 16, others capped at 18; re-run → no changes.
- Real services on the clone (MAKL): exit proposal 18 of 21 days → approval with an invalid month refused → approved into 2026-10 → **October payroll paid 18 × 2,884.62 = 51,923.16** as an earning → payroll month approved → encashment **PAID** → another approval into that month refused.
- Local DB: schema only — no encashable leave types, no encashments.

### Things to know
- Set it per leave type: e.g. Annual → carry forward "Up to N days" = 9, Encashment at year end max 18 (and / or on exit).
- **Exit encashment must be approved into the leaver's last payroll month while the employee is still active** (payroll pays active employees only). Local already has 17 employees with a termination date in 2026; they would get exit proposals once a leave type allows exit encashment — reject the ones settled outside the system.
- Year-end encashment is proposed when HR runs the year-end (Accrual & Carry Forward tab), after the year's leave is settled.
- **Not checked in a browser.**

### Next: Step 15 — Comp-off
Scope (§18): overtime type compensation PAY / COMP_OFF / EMPLOYEE_CHOICE; a comp-off leave type; worked time on weekly off / holiday → comp-off days by configurable slabs (e.g. ≥ 4 h = 0.5, ≥ 8 h = 1); credits with expiry; comp-off days not paid as overtime. **Q15** (who gets it, hours for half / full day, expiry) welcome — defaults keep OT2 as PAY until switched.

---

## 22. Step 15 — completion notes (2026-10-03)

**Answers received:**
- **Q13** — the day rate should follow the shift (Sat + Sun off → 22 days, Sunday off → 26), with HR / Finance able to use a monthly basis (30 / calendar days) instead. Step 13 already has both ("the employee's scheduled working days" = shift based; "fixed 30" or "days of the payroll period" = monthly basis); added the option *count holidays as working days*. **Each company still uses BASIC ÷ 26 until switched in Payroll → Day Rate.**
- **Q15** — a manager gives comp-off to an employee (work on an off day, night work …), company HR approves it on /task, then it is added to the employee's balance; everything configurable.
- **Q16** — everything configurable per company (Step 16).

### Delivered
| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 15") | `leave_types`: `isCompOff` (accrual = **CREDITS**), `compOffExpiryDays`, `compOffMaxDaysPerGrant`, `compOffRequiresAttendance`, `compOffExcludesOvertime`. `comp_off_grants` (employee, comp-off type, work date, reason OFF_DAY_WORK / NIGHT_WORK / OTHER, days, remarks, PENDING_HR → APPROVED (credited / expires) / REJECTED; one open grant per employee + date). `fn_LeaveEntitlementAsOf`: CREDITS = approved grants credited in the year, minus credited days that lapsed unused (oldest first). `company_payroll_settings.workingDaysIncludeHolidays`; `fn_EmployeeWorkingDays` returns holiday days too. Permissions **`compoff.grant`** (from `leave.approve`, 9 roles) and **`compoff.approve`** (from `leave.manage`, 6 roles); menu **Leave Management → Comp-off**. Applied on local; re-run safe; nothing created. |
| Leave → **Comp-off** (`/leave/comp-off`) | List (waiting for HR / approved / rejected, search, paging) + **Grant Comp-off** drawer (employee search, comp-off type with its rules, work date, reason, days, what the work was) + Approve / Reject with remarks for HR. |
| **/task** | Pending comp-off appears on the Leave tab for company HR (`compoff.approve`) with a "Comp-off" badge; approve / reject from there. The person who granted it cannot approve it. |
| Rules (per comp-off leave type) | Work date today or earlier; at most *max days per grant*; optional *clock-in required on the work date*; one grant per employee and date; employee must be eligible (Step 8) and not the granter. On approval: credited today, expires after *N days* (empty = never), **balance updated at once**; with *do not also pay overtime* the work date's overtime is set to rejected (payable 0, stays so when the day is re-processed). Expired unused credits leave the balance through the daily accrual run (Step 10). |
| Leave type drawer | Accrual → **"From comp-off grants"** shows expiry, maximum per grant, clock-in required, overtime not paid. |
| Payroll → Day Rate | "Count holidays as working days" with the working-days divisor; the worked example explains 22 / 26. |

### Verification done
- `dotnet build` ✔, `dotnet test` **73/73** ✔ (new: comp-off type ⇔ CREDITS, limits; day rate ÷ 22 / ÷ 26), `tsc --noEmit` ✔.
- Throwaway clone (dropped), rolled-back SQL: credits 1 (June, used on 10 June) + 2 (August, expired unused) + 1 (September) → balance 2 as of 3 Oct, 3 as of 15 Aug, 1 as of 5 Jun; with the June leave rejected → 1; a pending grant is not counted.
- Real services on the clone: grants refused for a future date, above the maximum, a date without clock-in and a second grant for the same date; grant for Sunday 27 Sep → listed in HR's /task inbox ("Comp-off 1 day(s) for 27 Sep 2026 — Granted by Line Manager for work on a weekly off / holiday"); the granter's own approval refused; HR approval → credited, expires 2 Nov, balance 1.00; that Sunday's overtime 150 / 180 min → payable 0 "Compensated by comp-off", still 0 after re-processing the day.

### Things to know
- **To start:** create a comp-off leave type per company (Leave Types → Credited: From comp-off grants), set expiry / maximum / clock-in / overtime rules.
- Comp-off days are used like any leave (apply for the comp-off type); the counting rules of Step 9 apply.
- **Not checked in a browser.**

### Next: Step 16 — Night-shift allowance
Scope (§18, Q16 = all configurable per company): night window, minimum night minutes, amount per night or per hour (fixed or % of the hourly rate), pay component / payslip line, eligibility (employment types, departments …); processing stores night minutes per attendance day; payroll pays per period.

---

## 23. Step 16 — completion notes (2026-10-03)

**Answer received (Q16):** make it configurable, so whatever HR wants for a company can be set.

### Delivered
| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 16") | `night_allowance_rules` (per company: name, payslip label, priority, active, night window that may cross midnight, minimum night minutes, rounding down, pay basis PER_NIGHT / PER_HOUR / PERCENT_OF_HOURLY, amount, cap per payroll period, weekly offs / holidays included or not), `night_allowance_eligibility` (same criteria as leave eligibility; none = everyone). `fn_NightMinutes` (minutes of the actual clock-in / clock-out inside the window of the work date and of the evening before), `fn_NightAllowanceDays` (qualifying nights per employee under the first active eligible rule by priority). Menu **Attendance → Night Allowance** (`attendance.manage`). Applied on local; re-run safe; no rules created. |
| Attendance → **Night Allowance** (`/attendance/night-allowance`) | Rules list + rule drawer (all settings + "who it applies to" tick lists) + **Who qualifies** preview for a company and date range (nights, night time, amount; % rules are worked out by payroll). |
| Payroll | Each employee's qualifying nights of the attendance period are paid per rule as an earning ("Night allowance (ward) (3 night(s), 24 h)"), taxable like other pay; the % basis uses the employee's hourly rate from Payroll → Day Rate (Step 13). Night time comes from the punches at payroll time, so a changed rule applies to the next payroll run without re-processing attendance. |

### Verification done
- `dotnet build` ✔, `dotnet test` **76/76** ✔ (3 new: per night / per hour / % of hourly rate, cap, % needs the hourly rate), `tsc --noEmit` ✔.
- Throwaway clone (dropped): night minutes — 21:00 → 07:00 = 480 in a 22–06 window; 04:00 → 12:00 = 120 (window of the evening before); 08–17 = 0; 14:00 → 23:30 = 90; same-day window 19–23 = 180; no clock-out = 0. Rules — department rule (per night, ≥ 4 h, working days only) paid the working night (480 min) and not the weekly-off night; everyone-else rule (per hour, ≥ 1 h, rounded to 30) turned 110 min into 90.
- **Real September payroll of MAKL on the clone**: ward rule 25 % of hourly rate → 24 h × 360.58 × 25 % = **2,163.46**; other rule 300 per night, 3 nights, cap 500 → **500**; a window with the same start and end refused.
- Local DB: schema + menu only.

### Things to know
- Local attendance has **no night punches** (no records past 22:00 or across midnight), so nothing would be paid there until night shifts are worked.
- An employee gets exactly one rule (the first matching by priority); if that rule excludes off days, off-day nights are not paid under a lower rule.
- **Not checked in a browser.**

### Next: Step 17 — Shift restriction by criteria
Scope (§18): `shift_eligibility` (departments, locations, job titles, grades, employment types — none = everyone) per shift, enforced in the assignment drawer, roster grid and both uploads; shift lists show allowed shifts only. Nothing changes until restrictions are set.

## 24. Step 17 — completion notes (2026-10-03)

### Delivered
| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 17") | `shift_eligibility` (shift, company, criterion DEPARTMENT / LOCATION / JOB_TITLE / GRADE / EMPLOYMENT_TYPE / COST_CENTRE, value). Each company the shift is enabled for keeps its own rows; no rows for a company = everyone of that company. Applied on local; re-run safe; no rows created. No new menu or permission — managed from Shift Master (`shifts.manage`). |
| Shift Master (`/shifts/master`) | New **Who may use it** button (shield icon) on each shift → 50 % drawer: company (when the shift is shared), tick lists per criterion. Values in one list are alternatives; two lists must both match. Values must belong to that company. |
| Enforcement (one rule, `ShiftRestriction`) | Refused with a clear message wherever a shift is given to an employee: employee shift assignment, bulk assignment, day override, roster grid save, roster upload and general shift upload (an issue on the row), employee master edit (when shift / department / location / job title / grade / employment type / cost centre changes), new employee, onboarding conversion. |
| Shift pickers | Roster grid cells and the employee assignment drawer list only the shifts the employee may be given. |

### Verification done
- `dotnet build` ✔, `dotnet test` **80/80** ✔ (4 new: no rows = everyone, values in one list are alternatives, two lists must both match, rows of another company do not apply), `tsc --noEmit` ✔.
- Throwaway clone (dropped): G10 of MAKL restricted to one department → a foreign value refused on save; the other department's employee refused for assignment, day override and roster cell, and G10 hidden in that employee's roster row; the department's employee allowed for all three; clearing the list lets everyone use it again.
- Local DB: schema only (0 restriction rows).

### Things to know
- **Existing assignments and roster entries are not changed** when a restriction is added; it applies to the next time a shift is given.
- Editing an employee's department (or other criterion) is refused if their current shift would no longer be allowed — change the shift first, or widen the list.
- The uploads use the same rule; they were not run against a file on the clone (same code path as the roster save that was checked).
- **Not checked in a browser.**

### Next: Step 18 — Clock-in selfie / geofence (by flag)
Scope (§18): company flags `selfieRequired`, `geofenceMode` (OFF / WARN / ENFORCE); location latitude / longitude / radius; one-click clock-in sends photo + coordinates, stored on the punch; HR sees photo and distance. Both OFF keeps today's behaviour.

## 25. Step 18 — completion notes (2026-10-03)

### Delivered
| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 18") | `company_attendance_policies` + `geofenceMode` (OFF / WARN / ENFORCE, default OFF), `geofenceMaxAccuracyMeters` (NULL = any), `clockSelfieMode` (OFF / CLOCK_IN / IN_AND_OUT, default OFF); the existing `geofenceRadiusMeters` (150 on every company) is the company default circle. `locations` + `latitude`, `longitude`, `geofenceRadiusMeters` (own radius, NULL = company default). `attendance_clock_events` (one row per web clock in / out: position, GPS precision, work location, distance, radius, mode, result, photo key, browser). Menu **Attendance → Clock-in Check** (`attendance.manage`). Applied on local; re-run safe. |
| Policy page (`/masters/shift-grace-policies`) | The old "GPS Geofence Perimeter" block (fixed radius buttons, a hard-coded campus name, never applied) is replaced by **Clock-in photo & location**: location check, photo, default radius, GPS precision needed. Reset Defaults reads them from the column defaults. |
| Org Masters → Locations | Latitude, longitude, own radius, **Use my current position**, map link. Both coordinates or neither. |
| Clock in / out (portal card) | When the company asks: the browser position is sent (WARN / ENFORCE) and a front-camera photo is taken (at clock in, or at both). Server rules: ENFORCE refuses no position, a work location without map position, a reading less precise than the limit, or a position outside the circle — with a message saying why (e.g. "You are 334 m from Arch Place; clock in / out is allowed within 150 m"). WARN accepts and flags. A missing photo is refused when asked for. The photo must really be a JPEG / PNG within the size limit; it goes to blob storage. Nothing is recorded when a punch is refused. |
| Attendance → **Clock-in Check** (`/attendance/clock-check`) | Punches by date range, company, in / out, result, with photo, employee; counts per result (click to filter); distance / radius, GPS precision, map link, photo viewer. |
| Settings (appsettings `ClockEvidence`, no fallbacks) | `SelfieMaxBytes`, `SelfieContentTypes`, `SelfieMaxWidthPx`, `SelfieStorageFolder`, `LocationTimeoutSeconds`. |

### Verification done
- `dotnet build` ✔, `dotnet test` **86/86** ✔ (6 new: distance, OFF, location radius before company radius, not shared / no site / imprecise, photo modes, photo type / size / content), `tsc --noEmit` ✔.
- Throwaway clone (dropped), MAKL set to ENFORCE / 150 m / ±50 m / photo at clock in, Arch Place given coordinates: location with latitude only refused; invalid mode and radius 0 refused; clock in refused without photo, without position, 334 m away, at ±200 m, with a fake JPEG — and **no event or punch recorded** for any of them; 56 m away at ±10 m with photo accepted (INSIDE, photo downloadable, browser stored). WARN: clock out 334 m away accepted and flagged OUTSIDE. OFF: one-click clock out, nothing checked or stored.
- Bug found and fixed during the check: the general money rule (2 decimals for every decimal column) cut coordinates to 2 decimals (~1 km); coordinates now keep 6 decimals.
- Local DB: schema + menu only (no locations have coordinates, every company OFF).

### Things to know (before switching it on)
- **Camera and location only work over HTTPS** (or localhost) — browsers block both on plain http. The VM must serve the portal over https.
- Add the **`ClockEvidence`** section to the VM's appsettings (clock in / out with photo or location check fails with "ClockEvidence:… is not configured" otherwise). Photos go to the configured blob storage (`BlobStorage:Provider`).
- **No location has map coordinates yet** — set them (Org Masters → Locations) before choosing ENFORCE, or everyone at that location is refused.
- A browser position can be faked by a determined user; the photo and the review page are the check against that. Low precision (indoors / laptops without GPS) is common — start with WARN and set the precision limit from what Clock-in Check shows.
- Old `isGeofenceEnforced` (set on every company, never applied) is no longer read.
- **Not checked in a browser** (camera and location need a real device).

### Phase 2 complete
All 18 steps are done. Open items: see the decisions list in the session report (fallback salary 75,000, unused `WorkingDaysPerMonth`, Step 2b shift merge, recounting past leave, Countries screen, unused leave entitlement rate) and go-live (browser walk-through, run `db_changes.sql` on the VM, appsettings keys `ShiftUploads`, `LeaveAccrual`, `Notifications`, `ClockEvidence`, real SMTP, HTTPS).

## 26. Open items after Phase 2 — closed (2026-10-03)

The six items left open at the end of Phase 2. Items 3 and 4 are opt-in scripts, **not run on your data**.

| # | Item | What was done |
|---|---|---|
| 1 | Hardcoded 75,000 fallback salary | **Removed.** Payroll never invents a salary; HR chooses what happens (see below). |
| 2 | `WorkingDaysPerMonth` nothing reads | **Consolidated.** The dead duplicate is gone; the live per-company divisor stays the one setting. |
| 3 | Step 2b duplicate-shift merge | **Dry run shown, verified on a clone.** Waiting for your go-ahead; nothing merged. |
| 4 | Recounting past leave | **Script written and verified on a clone.** Waiting for your go-ahead; nothing recounted. |
| 5 | Countries screen | **Built** — Masters → Country master. |
| 6 | Employment-type leave entitlement rate | **Now drives leave entitlement.** |

### 1. No basic salary (was: a silent 75,000)
`PayrollService` used to invent a basic salary of **75,000** for any employee with no salary and no grade salary range.
On local that is **3,861 of 3,871 active employees**, so essentially every payslip ever produced here came from a made-up
number. The hardcoded figure is gone; the company's payroll settings decide (`company_payroll_settings.missingSalaryMode`):

| Mode | What payroll does |
|---|---|
| **Refuse** (default) | The run stops and names the employees. Nothing is written. |
| Skip | Those employees are left out of the run and listed. |
| Grade mid-point | The middle of the employee's grade salary range; an employee whose grade has no range still stops the run. |
| A set amount | `missingSalaryDefaultAmount` — a number HR types in. No number is hardcoded anywhere. |

**Where:** Payroll → Settings → "An employee has no basic salary".
The engine's other 75,000 (India's annual standard deduction) is a real statutory figure on the statutory config screen;
its silent `> 0 ? … : 75000` fallback was removed so the configured value is always the one used.

### 2. Working days per month
There were two: `statutory_configs.workingDaysPerMonth` (26, per country, **never read by anything**) and
`company_payroll_settings.fixedDivisor` (26, per company, used by the day rate, hourly rate, leave, absence and overtime
pay since Step 13). The dead one is removed from the entity, DTOs and the statutory config screen; the DB column is left
in place, unmapped. **Where the live one is:** Payroll → Settings → "Divide by" = *A fixed number of days* → **Days**.

### 6. Leave entitlement from the employment type
`employment_types.leaveEntitlementRate` was stored, editable and shown as "1.75d/mo leave", but **nothing used it** —
every employee of a company got the same days. The rates are already meaningful on local: Permanent / Fixed-Term /
Probationary **1.75** days a month, Medical Intern **1.25**, Casual and Locum **0.00**.

A leave type now chooses where its days come from (`leave_types.entitlementSource`):
- **FIXED** (default, unchanged): the days set on the leave type, the same for everyone.
- **EMPLOYMENT_TYPE**: each employee's own employment type rate — monthly accrual credits the rate, yearly accrual
  credits **rate x 12**. So one Annual Leave type gives permanent staff 21 days a year, an intern 15 and casual staff 0.

Precedence: a matching **years-of-service slab** (Step 13) still wins, then the employment type, then the fixed number.
An employee with no employment type accrues **0** and is told so on their own leave page — no hidden fallback.

**Where HR sets it:**
- the rate per employment type: **Org Masters → Employment Types** → *Leave entitlement rate (days/month)*;
- switching a leave type over: **Leave & Holidays → Leave Types** → *Accrual & carry forward* → **"Days come from"**.
  With *The employee's employment type rate*, "Days per year" is filled from the employment type and greyed out.

**Where the employee sees it:** their leave page (**Portal → Leave → statutory quotas card**) shows
"You earn: 1.75 days a month (21 a full year) — from your employment type: Permanent & Pensionable".

### 3 & 4. The two opt-in scripts (nothing applied to your data)
| Script | Dry run on local | Verified on a throwaway clone |
|---|---|---|
| `docs/policies_step2b_merge_duplicate_shifts.sql` | **770 shifts → 55.** All 55 codes are identical across the 14 companies of `grp-lch`; **no code has conflicting copies**. One code (G1) has copies under a different name; the kept name is "General Hospital Day Shift (G1)". | Applied: 770 → 55 shifts, 5,390 → 385 day rules, **56,910 attendance records and 3,867 assignments unchanged, 0 orphans**. |
| `docs/policies_recount_past_leave.sql` (new) | **1 approved request changes**: EMP-ADM-001 annual leave 24–26 Sep, 2 → 3 days (his schedule makes that Saturday a working day); balance 19 → 18 remaining; none would go below zero. | Applied: 1 request and 1 balance updated; a second run finds nothing (re-run safe). |

Both take `@DryRun = 1` (default, reports only) / `@DryRun = 0` (applies, in one transaction). The recount script also
takes `@CompanyId`, rewrites `daysRequested` for approved and pending requests, rebuilds `taken` / `pending` /
`remaining`, leaves entitlement and carried-forward days alone, and reports any balance that would go below zero first.

### 5. Country master
The 16 seeded countries were read-only, and `CountryDto` did not even carry the statutory leave act employees are shown.
New screen **Masters → Country master** (`/countrymaster`, new permission `countries.manage`, granted to the 5 roles that
already manage currencies): name, nationality, both ISO codes, dial code, currency, flag, order, in-use flag and the
**statutory leave act**. Name and both ISO codes are required and unique; a country used by a company or work location
cannot be switched off, and the list shows how many use each one.

### Verification
- `dotnet build` ✔, `dotnet test` **91/91** ✔ (5 new: own salary wins, refuse / skip resolve to nothing, grade mid-point
  needs a range, a set amount, and no mode ever invents a figure), `tsc --noEmit` ✔.
- Throwaway clone (dropped): invalid mode and "a set amount" with no amount refused; **Refuse** named all 72 MAKL
  employees and wrote nothing; **Skip** produced 0 payslips (no MAKL employee has a real salary); **a set amount of
  40,000** produced 72 payslips from HR's own number; **grade mid-point** still refused (no MAKL grade has a salary
  range). Annual leave switched to the employment type rate saved and read back, and the employee's own page showed
  "1.75/month (21.00/year), from your employment type: Permanent & Pensionable".
- Rolled-back transaction on local: the same employee on each employment type in turn gave 21 / 21 / 21 / 15 / 0 / 0
  days a year for Permanent, Fixed-Term, Probationary, Intern, Locum and Casual.
- Local DB: schema, the `countries.manage` permission and the Country master menu only. 770 shifts, no leave type
  changed, no payroll settings row created, EMP-ADM-001's request still 2 days.

### Still open
- **Your go-ahead on the two scripts** (shift merge, leave recount) — both verified, neither run on your data.
- **Decide the payroll mode per company** before the next run: on **Refuse** (the default) payroll will stop until
  salaries are entered. 3,861 of 3,871 employees have none.
- Not changed, worth knowing: the engine still has silent `> 0 ? configured : hardcoded` fallbacks for other statutory
  rates (professional tax 200, rebate 700,000, UAE / Tanzania / Zambia rates) and a hardcoded `basic / 30` daily wage in
  the UAE gratuity accrual. Say the word and I will make those read the configured value only, as I did for the 75,000.
- Go-live, unchanged: browser walk-through, run `db_changes.sql` on the VM, appsettings keys (`ShiftUploads`,
  `LeaveAccrual`, `Notifications`, `ClockEvidence`), real SMTP, HTTPS.

---

## 27. Phase 3 — gaps against the Staff Attendance Policy document (pending, 2026-10-03)

**Source:** `docs/artifacts/Staff_Attendance_Policy_HRMS.docx` — "Staff Attendance Policy (HRMS)", v1.0, 12 sections.
Every line below was checked in code and SQL, not inferred from these notes.
**Implementation detail for each step lives in `docs/todo.md`** (items D1–D14) — that is the working file; this section is
the record of what the document asks for and where we stand.

### 27.1 Verdict
9 of the 12 sections are substantially built. Of the document's specific rules, **19 are delivered**, **13 are partly
built** and **10 are missing**. Several of the partial ones are configuration that exists but is never enforced — the
same pattern as the employment-type leave entitlement rate closed in §26.

### 27.2 Already delivered (no work needed)
§2 shift master, patterns, weekly offs, per-department restriction (Step 17), work schedules, roster grid and both
uploads · §3 biometric terminals, web clock in / out, clock-in access per employee, selfie and location check (Step 18),
registered phone (Step 4b) · §4 grace minutes and N late arrivals a month, both configurable, Late flag beyond them ·
§5 status codes P, A, HD, WO, PH, L · §6 unauthorised absence marked A and deducted as unpaid · §7 regularisation
raised in the system, monthly quota, blocked in closed periods · §8 overtime types with their own rates, authorisation,
limits per period and week, comp-off (manager grants → HR approves → credited with expiry, Step 15) · §9 approved leave
overrides attendance, lock on the cut-off / payroll approval, unpaid days, overtime and late deductions flowing to
payroll · §10 employee and HR duties (portal, masters, Exceptions Queue, close month) · §11 audit trail.

### 27.3 Partly built — the rule exists but does not behave as written
| Policy line | Today | Missing |
|---|---|---|
| §6 Sick absence beyond N days needs a medical certificate | Leave types hold *requires proof* + *proof threshold days*; a request has a proof document field | **Nothing enforces it** — a 10-day sick leave with no certificate is accepted |
| §4 "Half-day leave deducted (or LOP if no balance)" | The half-day penalty deducts half a day's **pay** at once | Never touches the leave balance, so "leave first, LOP only if short" never happens |
| §3 Field duty via an on-duty request | Out Duty request exists and reaches the approvals inbox | Marks the day **plain Present with a text note** — invisible in reports; also hardcodes 09:00 / 17:00 / 480 minutes |
| §5 LOP code | A payroll deduction line only | Not a day status code |
| §7 "Manager approves, then HR verifies" | **Single step** — HR *or* the line manager approves | No second HR stage (leave has two; regularisation does not) |
| §7 "Beyond N, HR head approval" | The cap **refuses** | No escalation to allow one above the cap |
| §7 After cut-off "processed in the next cycle" | Refused outright | Not deferred |
| §8 Overtime "pre-approved by manager and HR" | Authorised **after** the work | No pre-approval before the shift |
| §9 Locked "after manager confirmation" | Lock is by date / payroll approval | No manager confirmation step at all |
| §9 Post-lock changes "paid or recovered in the next cycle" | Everything refused after the lock | No HR override, no arrears |
| §10 Manager "within 2 working days" | Inbox flags items overdue after 2 days | The 2 is **hardcoded**, counts **calendar** days, no reminder |
| §3 "Mobile app" | Portal works on a phone; sign-in tied to a registered phone | No native app |
| §10 "Payroll processes only locked attendance" | Payroll reads the period, locks it on approval | Does not *require* a prior lock |

### 27.4 Missing outright
| Policy line | Note |
|---|---|
| §4 Tier "4–5 occurrences: warning / alert to manager" | The document's table has **three** tiers; the system has two. **No late alert of any kind exists** — the only email template in the system is the overtime limit one |
| §4 "Early going treated the same as late coming" | Early departure minutes are recorded and shown, but carry no penalty, status or monthly allowance |
| §5 OD (On Duty) status code | — |
| §5 WFH (Work From Home) status code and request | Nothing anywhere |
| §6 "Absence of 3+ consecutive days without information" | No detection, flag or alert |
| §7 "Within 3 working days of the date" | No submission deadline |
| §2 "Rosters published 5–7 days before the period starts" | No lead-time rule or warning |
| §2 "Shift swaps need prior manager approval" | No swap feature; HR edits the roster directly, leaving no request trail |
| §10 Manager confirms monthly attendance before cut-off | — |
| §3 Proxy / buddy punching | The controls exist (selfie, geofence, registered phone) but no report flags suspected proxy punching |

### 27.5 Steps (worst first — the rules staff and auditors actually feel)
| Step | Scope | Closes |
|---|---|---|
| 19 | Enforce proof of sickness (`requiresProof` / `proofThresholdDays` + attachment, optional HR waiver) | §6 partial |
| 20 | Late escalation tiers (`late_penalty_tiers`), `LATE_WARNING` alert to the manager, new `LEAVE_THEN_LOP` deduction mode | §4 missing + partial |
| 21 | Early going mirrored on late (`fn_EarlyPenalty`, own grace / allowance / mode, own payslip line) | §4 missing |
| 22 | Consecutive unauthorised absence: threshold, Exceptions Queue filter, `ABSENCE_ESCALATION` alert | §6 missing |
| 23 | On Duty + Work From Home as real day types (`attendance_day_requests`), Out Duty times from the shift instead of 09:00 / 17:00 / 480 | §3 / §5 |
| 24 | Regularisation: window in working days, two stages (manager → HR), over-quota escalation role | §7 |
| 25 | Manager monthly attendance confirmation, optionally required before the lock | §9 / §10 |
| 26 | Roster publication lead time (warn, and show the publish-by date) | §2 |
| 27 | Shift swap requests (colleague accepts → manager approves → roster days exchanged, Step 17 restriction reused) | §2 |
| 28 | Post-lock change with HR approval → `payroll_arrears` picked up by the next run | §9 |
| 29 | Approval SLA per company and module, counted in working days, with reminders | §10 |
| — | Not scheduled: proxy-punch report, overtime pre-approval, native mobile app | §3 / §8 |

Every new number is a per-company setting, and every default keeps today's behaviour, so nothing changes until HR
switches it on.

### 27.6 Two things found while checking
- **Hardcoded numbers**, against follow.md #10: Out Duty defaults to `09:00` / `17:00` / `480` minutes, and the
  approval SLA is a hardcoded `slaDays = 2` in `ApprovalsService`. Both are fixed by Steps 23 and 29.
- Every bracketed number in the document ([10-15] minutes, [3] times, [5-7] days, [2] days) maps to something
  configurable **except** those two — so the policy can be filled in per company once the missing rules exist.

### Step 19 — done, see §28

## 28. Step 19 — completion notes (2026-10-03): supporting document (proof) for leave

**Policy line:** §6 "Sick absence beyond [2] days requires a medical certificate."
**Before:** six companies already had *proof required above 2 days* on sick (full / half), maternity and paternity leave,
but **nothing enforced it**, the portal had no way to attach a file, and the request's proof field was never filled.

### Delivered (all per leave type — Leave & Holidays → Leave Types → Application rules)
| Setting | Effect |
|---|---|
| Supporting document required + **needed for more than N days** (existing fields) | Enforced. "More than": with 2, a 2-day request needs none, 3 days does. 0 = every request |
| **Needed** — *Before final approval* (default) / *To submit the request* (new `proofTiming`) | Before final approval: the employee can submit and attach later (a certificate often comes after return); the manager can still recommend; **HR's final approval is refused until it is attached**. To submit: the request itself is refused without it |
| **HR may approve without it, giving a reason** (new `proofWaiverByHr`, default off) | Only for a person holding the new permission **`leave.proof.waive`** (granted to the roles with `leave.manage`: Company HR, Group HR, HR VP, HR Manager, Super Admin, Developer — not to line managers). Name, time and reason are recorded on the request |

| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 19") | `leave_types.proofTiming`, `proofWaiverByHr`; `leave_requests` + `proofFileName`, `proofContentType`, `proofUploadedAt`, `proofUploadedByName`, `proofWaivedByName`, `proofWaivedAt`, `proofWaiverReason`; `proofDocumentUrl` now holds the storage key; permission `leave.proof.waive`. Applied on local; re-run safe |
| Storage (appsettings **`LeaveProof`**: `MaxBytes`, `AllowedTypes`, `StorageFolder` — no fallbacks) | Files go to the configured blob storage under `leave-proofs/{company}/{month}/`. Type and size are checked, and the bytes must really be a PDF / JPEG / PNG. The file is **never linked publicly** — it is streamed only to the employee, their line manager, and HR |
| API | `POST /api/leave/apply-with-proof` (apply + file), `POST /api/leave/{id}/proof` (attach or replace while pending — the employee or HR), `GET /api/leave/{id}/proof` (view), `GET /api/leave/proof-settings` (limits for the picker). **Removed** the client-supplied `proofDocumentUrl` from apply, which would have let a caller point a request at someone else's file |
| Portal (employee) | Apply drawer shows the rule for the chosen leave type and dates and takes the file; "My applications" shows the document, **"Supporting document needed — attach"** for pending requests, replace, and a waiver note |
| HR apply / apply on behalf | Same picker and rule |
| Approvals inbox and Tasks | Each leave shows a *Supporting document* panel: the rule, open the file, or — at the HR stage, when allowed — *Approve without it* with the reason in the remarks |
| HR leave detail | Document link, "required and not attached", or the waiver with who and why |

### Fixed on the way
- **Approval refusals now reach the approver.** The approvals inbox and Tasks endpoints did not catch rule refusals, so
  any refusal (this one, and comp-off's "the granter cannot approve") came back as a generic server error. Now the
  reason is shown.
- **Batch approval could save a refused item.** The inbox engine changed a leave's status *before* checking, and a batch
  reused one database context — so a later item's save could have written the refused item's change. The proof check
  now runs before anything changes, and each batch item is isolated; refused items are skipped and the message says how
  many need attention.
- **Batch result showed "processed undefined item(s)"** on both the Approvals and Tasks pages — the count is now read
  correctly and the server's message is shown.

### Verification
- `dotnet build` ✔, `dotnet test` **96/96** ✔ (5 new: "more than" threshold, refusal only when needed to submit, final
  approval needs the document, waiver needs type + permission + reason, file type / size / content), `tsc --noEmit` ✔.
- Throwaway clone (dropped), MAKL Sick Leave (Full Pay), proof above 2 days, employee 20341:
  - 2 days, no document → approved (not needed);
  - 5 days, no document → manager recommends; **HR final refused**, status stays PENDING_HR; waiver refused (type
    allows none); fake PDF refused; certificate attached → inbox shows it, download returns the PDF → approved;
  - *To submit*: refused without a document, refused with a PDF labelled as PNG, accepted with the certificate (stored
    under `leave-proofs/comp-makl-01/…`);
  - waiver allowed: refused for HR **without** `leave.proof.waive`, refused with no reason, approved with a reason and
    recorded;
  - **batch [sick without document, annual]** → 1 of 2 processed; sick **still PENDING_HR**, annual approved.
- Local DB: schema and the permission only; no request or leave type changed.

### Things to know
- **Add the `LeaveProof` section to the VM's appsettings** — without it, attaching a document fails with
  "LeaveProof:… is not configured" (applying without a document still works).
- Nothing changes for leave types without *proof required*. For the six companies that already ask for it on sick,
  maternity and paternity leave, **HR final approval of those requests above 2 days now needs the document** — that is
  the rule they had configured; switch on *HR may approve without it* where a waiver should be possible.
- The one request pending on local (MAKL annual leave, 5 days) is unaffected — annual leave asks for no proof.
- **Not checked in a browser.**

### Step 20 — done, see §29

## 29. Step 20 — completion notes (2026-10-03): late-coming escalation table, manager warning, leave-then-LOP

**Policy line:** §4 — occurrences per month 1–3 *grace, no action*; 4–5 *warning / system alert to manager*; 6th
onward *half-day leave deducted (or LOP if no balance)*.
**Before:** two tiers only (N free late arrivals, then a pay deduction); no alert of any kind; the "half day" always came
out of pay.

### Delivered (all per company — Masters → Attendance Policies → *Late coming escalation*)
| Setting | Effect |
|---|---|
| **Escalation table** (`late_penalty_tiers`) | Steps by the number of the late arrival in the attendance period: *Grace — no action* / *Warning — email to the manager, no deduction* / *Deduct* (with its own deduction or the company's). Steps must start at 1, follow on, and end open ("and above"). One click fills in the policy's own table. **No table = the previous rule, unchanged** — proven identical over all 7,813 late arrivals in the database |
| **Arriving later than the grace minutes is deducted at once** (`lateBeyondGraceAlwaysDeducts`, default on = as before) | Off: every late arrival, however late, is just counted by the table / allowance |
| New deduction **Leave first, then loss of pay** (`LEAVE_THEN_LOP`) | Takes **N days per penalised arrival** (`latePenaltyDays`, default 0.5) from a **chosen leave type** (`latePenaltyLeaveTypeId` — required whenever this deduction can happen). When that balance is short the days become loss of pay, deducted at the day rate. Decided arrival by arrival, oldest first; a day that stops being penalised (regularised, table changed) **gives its leave back** |
| **Late arrival warning email** (event `LATE_ARRIVAL_WARNING`) | Template seeded per company (switched off). On / off, recipients (To / CC / BCC, the line manager, the employee), preview and send-now on the same section. Sent **once per late day** on a warning step (this attendance period and the previous one). Scheduled with the other notification checks when `Notifications:LateWarningCheckEnabled` is true. Placeholders include `{{LateOccurrence}}` and `{{NextStep}}` ("From late arrival number 6 in this period, 0.5 day(s) are taken from Annual Leave, or deducted from pay if the balance is short") |

| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 20") | Company policy + `lateBeyondGraceAlwaysDeducts`, `latePenaltyLeaveTypeId`, `latePenaltyDays`; `attendance_records.lateTierAction`; `late_penalty_tiers`; `attendance_late_leave_charges` (one row per penalised day: LEAVE or LOP); `fn_LatePenaltyTiered`; `sp_SyncLateLeaveCharges` (called by daily processing); `sp_GetLateArrivalWarnings`; `sp_ProcessDailyAttendance` and `sp_SimulateAttendancePolicy` updated; warning template per company. Applied on local; re-run safe |
| Payroll | Loss-of-pay penalty days in the attendance period → *Late Arrival Penalty — loss of pay (N day(s), M late arrival(s))* at the day rate. Leave-charged days cost nothing in pay |
| HR Attendance Register | Shows the warning step, and for a leave-then-LOP day whether it came from leave (which type) or was loss of pay |
| Simulator | Uses the saved table: shows the step for "late arrival #N" |
| Shared code | The excess-hours email and the late warning now share one settings card, one send-log table (filtered per event) and one send-once routine |
| Recount script | `policies_recount_past_leave.sql` keeps late-penalty leave days in `taken` |

### Fixed on the way
- **Late deduction mode was never validated** — any text was saved. Now only the four modes.
- **Half-day card said "0.5 × (Basic ÷ 30)"**, but payroll uses the day rate from Payroll → Settings. Corrected.
- **"Flat disciplinary fine" is never deducted by payroll** (it never was). The card now says *recorded only*, and the
  table cannot choose it. Making it real needs a fine amount per company — not done, flagged.

### Verification
- `dotnet build` ✔, `dotnet test` **101/101** ✔ (5 new: the policy's table, gaps / overlaps / open ends refused,
  actions and modes checked, when a leave type is required, the next-step wording), `tsc --noEmit` ✔.
- Rolled-back check on local: with no table, the new rule equals the old one on **all 7,813 late arrivals**.
- Throwaway clone (dropped), MAKL employee 10961, eight late arrivals in September, the policy's table, Annual Leave,
  0.5 day, every arrival counted:
  - gap in the table refused; leave-then-LOP without a leave type refused;
  - **#1–3 grace, #4–5 warning, #6–8 deduct** → 3 × 0.5 = 1.5 days from Annual Leave (21 → 19.5);
  - deduct step removed → all 1.5 days **given back**;
  - only 0.75 day left → #6 from leave, **#7–8 loss of pay**; September payroll: *loss of pay (1 day, 2 late arrivals)*
    = 2,000 (52,000 ÷ 26);
  - simulator: #2 grace, #4 warning, #7 leave-then-LOP; 09:30 (beyond grace, counted) → #2 grace;
  - warning preview: 47 warnings in MAKL; employee 10961 #4 and #5 → HR address + his line manager; next-step text as above.
- Local DB: schema, procedures and the switched-off templates only — no table, no charge, no record changed.

### Things to know
- **Nothing changes until HR adds a table** for a company. The current setting everywhere is 15 grace minutes, 3 free
  arrivals, then half a day's pay — and **arriving later than 15 minutes is deducted at once**. All late arrivals on
  local are beyond 15 minutes, so with the policy's table HR will probably want *deducted at once* **off**, otherwise
  the table only governs arrivals within the grace minutes.
- The warning email must be **switched on with recipients** (the section shows it when the table has a warning step),
  and for automatic sending set `Notifications:LateWarningCheckEnabled` in the VM's appsettings.
- Leave charged by a late penalty counts as *taken* on that leave type, so it shows in the employee's balance and in
  year-end carry forward like any taken day.
- **Not checked in a browser.**

### Step 21 — done, see §30

## 30. Step 21 — completion notes (2026-10-04): early going treated the same as late coming

**Policy line:** §4 "Early going without approval is treated the same as late coming."
**Before:** early departure was **never measured** — the only place producing it (the employee monthly view) returned a
hardcoded 0 — and `shifts.earlyGraceMinutes` (10 on every shift) was read by nothing. (The §27 gap analysis said
"recorded and displayed"; it was not even that.)

### Delivered (per company — Attendance Policies → Late coming escalation → *Early going*)
| Setting | Effect |
|---|---|
| **Treat early going** (`earlyGoingMode`) | *Not penalised* (default — measured and shown only, **nothing changes**) / *Like late coming, counted on its own* (`SEPARATE`) / *Like late coming, counted together with late arrivals* (`WITH_LATE`: 3 late + 3 early = the 6th occurrence). Either way it goes through the same Step 20 table (or the monthly allowance) and the same beyond-grace rule |
| **Grace** (`earlyGoingGraceMinutes`) | Minutes before the shift end that are not counted; empty = **each shift's own `earlyGraceMinutes`** (finally used) |
| **Deduction** (`earlyGoingDeductionMode`) | Empty = the same as late coming (the policy's wording), or its own: half day, minutes short, leave-then-LOP, flat fine |

What counts as early going: leaving before the scheduled end on a working day of a **fixed** shift, **with a
clock-out**, on a day that still earned a **full** day credit. FLEXI shifts have no fixed end; a missing clock-out is a
missed punch; a day already paid as half or absent is short-paid once and is not charged again.

| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 21") | Company policy + `earlyGoingMode`, `earlyGoingGraceMinutes`, `earlyGoingDeductionMode`; `attendance_records` + `earlyDepartureMinutes`, `isEarlyGoing`, `earlySequenceInMonth`, `isEarlyCovered`, `earlyPenaltyType`, `earlyTierAction`; `attendance_late_leave_charges.source` (LATE / EARLY, unique per record **and** kind — a day can carry both); `fn_EvaluateAttendanceDay` measures early departure; `sp_ProcessDailyAttendance` numbers late and early occurrences (own or shared count) and applies the table; `sp_SyncLateLeaveCharges` charges both kinds; simulator shows early going. Applied on local; re-run safe |
| Payroll | Own lines: *Early Departure Penalty (N half-days)*, *… — loss of pay (N days)*, *… (N minutes short)* |
| HR Attendance Register | "left N m early · #k", warning step, and the deduction / leave charge |
| Simulator | The clock-out entered shows the early-going outcome with the saved settings |

### Verification
- `dotnet build` ✔, `dotnet test` **102/102** ✔, `tsc --noEmit` ✔.
- With early going **off**, the late result of every late arrival is unchanged: 7,813 on local against the old rule,
  and on a clone after reprocessing September all **184** MAKL late days equal the Step 20 numbering.
- Throwaway clone (dropped), MAKL employee 20325 (3 late arrivals, 5 early departures of 51–59 min in September),
  the policy's table, Annual Leave, every arrival counted:
  - invalid mode, leave-then-LOP without a leave type, negative grace — all refused;
  - **own count**: late #1–3 grace; early #1–3 grace, **#4–5 warning**;
  - **shared count**: one count of 8 — early departures become #4–8: **#4–5 warning, #6–8 half a day from Annual Leave**;
  - back to **off**: the 1.5 days **given back**, balance 21 again;
  - a day carrying both a late and an early leave-then-LOP penalty → **two charges** (LATE and EARLY);
  - early deduction *half a day's pay*, shared count, 4th onward → payslip *Early Departure Penalty (5 half-days)* =
    5,000 (52,000 ÷ 26 ÷ 2 each).
- Local: schema and procedures; every company **off**. Your running app's daily processing now fills
  `earlyDepartureMinutes` (4,684 records so far) — measurement only.

### Things to know
- Early departures on local are real (51–59 min on several MAKL days), so switching a company on will produce warnings
  and deductions straight away — preview it with the simulator and the Attendance Register first.
- An *approved* early leave (the policy says "without approval") is a regularisation / short leave: approve the
  regularisation and the day is reprocessed with the approved clock-out.
- **Not checked in a browser.**

### Step 22 — done, see §33

## 31. ESS — the employee's own comp-off, overtime and night allowance (2026-10-04)

**Reported:** an employee could not see comp-off, night allowance or overtime in their own account (ESS → My Leave).

**Why:**
1. **No company has a comp-off leave type yet.** Step 15 built comp-off but leaves the type to HR (Leave Types →
   credited *From comp-off grants*). Without one a manager cannot grant comp-off at all.
2. **No company has a night allowance rule yet** (Attendance → Night Allowance), so nobody qualifies.
3. **There was no employee-facing view of any of the three.** Overtime existed (e.g. EMP-ADM-001: 1 Oct, OT1, 1 h 45 m
   paid) but only HR could see it.

**Delivered:** ESS → My Leave → new tab **Comp-off, Overtime & Night Allowance** (read-only, always the signed-in
employee — the server takes the employee from the login):
| Section | Shows |
|---|---|
| Comp-off | Balance per comp-off leave type (credited, taken, pending, available, expiry rule) and every grant: date worked, reason, days, granted by, waiting for HR / credited / rejected (with HR's reason), expiry. When the company has no comp-off type it says so |
| Overtime | Per day: type and rate, time worked over, approved, paid, status (paid / waiting for approval / not approved, who decided); totals per type |
| Night allowance | Qualifying nights and night time per rule, an estimate for per-night / per-hour rules (a % rule is worked out by payroll) |
Period: the current pay (attendance) period by default, or any range up to a year. Once a comp-off type exists the
comp-off balance also appears with the other balances on the first tab.

API: `GET /api/me/comp-off`, `/api/me/overtime`, `/api/me/night-allowance` (`portal.me.view`, held by the employee
roles). No DB change.

**Verified on a throwaway clone (dropped):** before setup — "comp-off not set up", no night rules, overtime shown for
the current period (1 Oct, OT1, 105 min paid); after HR created a comp-off type (60-day expiry) and a manager granted
1 day for Sunday 27 Sep — *waiting for HR*, balance 0; after HR approved — **1 day available, expires 3 Dec 2026**; night
rule added — *configured*, no qualifying nights (local attendance has no night punches). `dotnet build` ✔,
`dotnet test` 102/102 ✔, `tsc` ✔. **Not checked in a browser.**

**To see comp-off and night allowance for real, HR must set them up per company:** Leave Types → new type credited
*From comp-off grants* (with its expiry / maximum per grant), and Attendance → Night Allowance → a rule.

## 32. Starting setup — comp-off type and night allowance rule per company (2026-10-04)

Asked: make comp-off and night allowance appear. Both need HR settings that no company had (§31).
**Script:** `docs/policies_seed_compoff_night.sql` — optional, **review the values first** (they sit at the top as
variables), re-run safe (a company that already has one is left alone). Not in `db_changes.sql`: these are HR settings
and the night allowance pays money. **Run on local**: 15 comp-off types, 15 night rules (one each per company).

| Created per company | Values (change on the screens afterwards) |
|---|---|
| Leave type **Compensatory Off** (`COMP_OFF`) — Leave & Holidays → Leave Types | Credited from approved comp-off grants; paid; each credit expires **60 days** after HR approves; a manager may grant at most **1 day** per day worked; the employee must have **attendance** on the day worked; that day's overtime is not paid as well; everyone eligible |
| Rule **Night shift allowance** — Attendance → Night Allowance | Window **22:00–06:00**; a night counts with at least **4 h** inside it; night time rounded down to **30 min**; pays **25 % of the hourly rate per night hour** (a percentage, so it works in KES and TZS alike); nights on weekly offs / holidays included; no cap; everyone; payslip label "Night allowance" |

**How each one then appears for the employee** (ESS → My Leave & Balances):
- **Comp-off** — the *Compensatory Off* balance appears with the other balances (0 until a grant is approved), and the
  tab *Comp-off, Overtime & Night Allowance* lists every grant. Flow: manager → Leave → Comp-off → *Grant comp-off* →
  HR approves on Tasks → credited, expiring after 60 days → employee applies it like any leave.
- **Night allowance** — same tab, *Night allowance*: qualifying nights and night time per rule for the chosen period;
  a % rule shows "amount worked out by payroll". Payroll pays it on the payslip line "Night allowance".
- **Overtime** — same tab, *Overtime*: every day with overtime — type and rate, time worked over, approved, **paid**,
  and the status (*Paid* for overtime that needs no approval, *Waiting for approval*, *Approved*, *Not approved* with
  who decided). Totals per type at the top. Default period = the current pay period; any range up to a year.

## 33. Step 22 — completion notes (2026-10-04): consecutive unauthorised absence

**Policy line:** §6 "Absence of [3] or more consecutive days without information is treated as unauthorized and may
lead to disciplinary action." **Before:** nothing detected it.

### Delivered (per company — Attendance Policies → *Unauthorised absence*)
| Setting / behaviour | Effect |
|---|---|
| **Treat as unauthorised after N consecutive working days** (`unauthorisedAbsenceAlertDays`, empty = **off**, minimum 2) | Every run of N or more consecutive **scheduled working days** that are ABSENT **without information** becomes one *Unauthorised absence* item in the **Exceptions Queue** (HIGH), with the dates and the count; it grows with the run and is **resolved automatically** if the run stops qualifying |
| What counts | Weekly offs, rest days and holidays between absent days neither break nor count. A day with a **pending leave request** or a **regularisation** (pending or approved) is "informed" and breaks the run; approved leave is ON_LEAVE, not ABSENT |
| **Alert email** (event `UNAUTHORISED_ABSENCE`) | Template seeded per company, switched off. On / off, recipients (To / CC / BCC, line manager, employee), preview and send-now in the same section. **Once per run** (a run is known by its first day). Scheduled with the other checks when `Notifications:UnauthorisedAbsenceCheckEnabled` is true |
| Exceptions Queue | New **type filter** and readable type names |

| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 22") | `company_attendance_policies.unauthorisedAbsenceAlertDays`; `fn_UnauthorisedAbsenceRuns` (gaps-and-islands over working days); `sp_FlagUnauthorisedAbsence` (called as step 5 of `sp_ProcessDailyAttendance`, current + previous attendance period); `sp_GetUnauthorisedAbsenceAlerts`; email template per company. Applied on local; re-run safe |

### Verification
- `dotnet build` ✔, `dotnet test` **102/102** ✔, `tsc --noEmit` ✔.
- Rolled-back checks on local, MAKL employee 20283, absent 18–25 Sep: **one run of 8**; a weekly off on the 22nd →
  still **one run of 7**; present on the 22nd → **4 + 3**; a pending leave request on the 22nd → **4 + 3**.
- Throwaway clone (dropped): 1 day refused; switched off → nothing flagged; **3 days** → employee 20283 flagged 18–25
  Sep (8) and 28–30 Sep (3); a regularisation raised for 22 Sep → the first item **updated to 18–21 (4)** and a new one
  23–25 (3); alert preview → each run to the HR address and his line manager.
- Local: schema, procedures and switched-off templates only — no company switched on, nothing flagged.

### Things to know
- **Local data has many absences** (39,627 ABSENT days, mostly employees without punches): at 3 days MAKL alone has
  **86 runs**. Clean up the attendance data (or start with a higher number) before switching it on, and preview the
  email before enabling it.
- **Not checked in a browser.**

### Next: Step 23 — On Duty + Work From Home as real day types

## 34. Step 23 — completion notes (2026-10-04): On Duty and Work From Home as real day types

Staff Attendance Policy §3 / §5 (field duty, remote work, status codes OD / WFH).

**Before:** "Out Duty" stamped the day PRESENT with a note and raised a regularisation that the inbox found by the words
"Out Duty". It used fixed 09:00 / 17:00 / 480 minutes and covered one day. Daily processing could overwrite it from
punches. Work from home did not exist.

### Delivered
| Who | What |
|---|---|
| **HR** — Attendance Policies → *On duty & work from home* | Allow on duty (default **on**), allow work from home (default **off**). Who approves: **Manager, then HR** (default) / **Manager only** / **HR only**. How many days back a request may start (empty = no limit; closed attendance periods are always refused) |
| **Employee** — ESS → Attendance → *Apply On Duty / WFH* | One request for a **date range**. Hours = **each day's own shift**, or the times the employee gives. On duty also asks for the place, kind of duty and transport. Refused when: the type is switched off, it starts too far back, a closed period is involved, there is no scheduled working day in the range, it overlaps another request, or it overlaps leave (pending or approved) |
| **Employee** — *My On Duty / WFH* | Every request with its stage, who decided and their remarks. A pending request, or an approved one that has not started, can be cancelled |
| **Manager** — Approvals inbox / Task Hub (*Attendance & Shifts*) | Requests of their direct reports at the manager stage, with dates, hours, place and reason |
| **HR** — same inbox | The HR stage. Needs the new DB permission **`attendance.dayrequest.hr`** ("HR approve on duty / work from home"), given to every role that has `attendance.manage`. An employee with **no reporting manager** goes straight to whoever holds that permission. Nobody decides their own request |
| **Processing** | On each scheduled working day (not holidays or weekly offs) of an **approved** request, the request's hours count as attendance and **extend** any real punches. A full day gets status **ON_DUTY** / **WORK_FROM_HOME**, full credit, and is not late. A window shorter than the shift is judged like punches (late / half day / below the half-day minimum), with the request named in the note. Only **real punches** are stored, so a rejection or cancellation leaves no trace on the next run. Final approval reprocesses past days at once |
| **Reports / views** | Registers (HR + team) show *On Duty* / *Work From Home*, and the status filters include them. The ESS calendar shows teal / indigo. Roster vs actual has its own two codes, not counted as exceptions. These count as present: dashboard KPI, monthly summary, ESS home, employee profile, reports |
| **LOP (decision)** | **No new status.** ABSENT is a scheduled working day with no attendance and no leave, which payroll already treats as loss of pay. Every screen now labels it **"Absent (LOP)"**. On Duty / WFH days are never ABSENT, so they are paid |

| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 23") | 4 policy columns; `attendance_day_requests`; `sp_ProcessDailyAttendance` (overlay + real punches only); permission `attendance.dayrequest.hr` + role grants; `sp_GetAttendanceDashboardSummary`, `sp_GetEmployeeMonthlyAttendance`, `sp_ReportRosterVsActual` count the new statuses; `fn_UnauthorisedAbsenceRuns` treats a pending or approved request as "informed" (Step 22). Applied twice on local |
| API | `GET/POST api/me/day-requests`, `GET api/me/day-requests/settings`, `POST api/me/day-requests/{id}/cancel` (portal.me.view; the employee is taken from the login). The old `POST api/attendance/out-duty` now creates a one-day ON_DUTY request; empty times = shift |
| Code | `DayRequestRules` (route, stages, decision, reprocessing), `DayRequestService`, inbox module `DAY_REQUEST` (pending list, decision, history, pending count). Frontend: `modules/portal/day-requests/*` (request drawer, my requests), `attendance/status.tsx` (shared status badge), `DayRequestPolicySection` |

### Verification
- `dotnet build` ✔, `dotnet test` **106/106** ✔ (4 new: stages per route, manager-only stage, HR permission,
  no self-decision, no second decision), `tsc --noEmit` ✔.
- Rolled-back checks on local (BLISS, 3 Oct):
  - An absent employee with approved WFH (shift hours) → **WORK_FROM_HOME, 540 min**.
  - 09:00–10:30 only → ABSENT, "below the half-day minimum".
  - Rejected → back to **ABSENT**.
  - A LATE employee (09:12–18:07) with On Duty → **ON_DUTY, not late, 607 min**, punches kept. A second run gives the same result.
- Whole-company rerun of that day: the old and new procedures give **identical** counts.
- Throwaway clone (dropped):
  - Each refusal gives its message.
  - Request → **manager's inbox only** (HR doesn't see the manager stage) → manager approves → day still ABSENT → **HR inbox** → HR approves → **WORK_FROM_HOME 540 min** at once.
  - History shows the decision. A future request can be cancelled; a started one cannot.

### Things to know
- **Old Out Duty entries are not converted.** They stay as regularisations and PRESENT days with their old badge. Count
  them on the VM before go-live:
  `SELECT COUNT(*) FROM regularisations WHERE reason LIKE '%Out Duty%'`.
- **No email** on submit or decision yet; requests appear in the inbox and the employee sees the status in *My On Duty / WFH*.
- **Already in the data (not caused by this step):** some days stored earlier were processed under older settings. For
  example, reprocessing BLISS 3 Oct moves about 60 days from LATE to HALF_DAY with **both** the old and the new
  procedure. Reprocess a period on purpose, not by accident.
- **Not checked in a browser.**

### Step 24 — done, see §35

## 35. Step 24 — completion notes (2026-10-04): regularisation deadline, two stages, over-quota escalation

Staff Attendance Policy §7 ("within N working days", "approved by the reporting manager, then verified by HR",
"beyond N per month, HR head approval"). Closes todo C6, B5, B6, B7.

**Before:** no deadline. One decision by the manager **or** HR (found by role code / level). The monthly cap refused
the request. A refusal message never reached the portal (it read `message`, the API puts it in `errors`).

### Delivered
| Who | What |
|---|---|
| **HR** — Attendance Policies → *Max miss-punch regularisations* (same card) | **Raise within N working days** after the date (empty = no limit). **Above the monthly limit:** *Refuse the request* (default, as before) or *Allow with approval from: &lt;role&gt;* (any active role, e.g. Group HR) |
| **Employee** — ESS → Attendance → *Regularise* drawer | Shows **time left** (working days of their own schedule; weekly offs and holidays skipped) and **used this period** (a of b). When the request would be refused, the reason is shown and *Submit* is disabled. Above the limit with a role set, it says who will decide. After submitting, the message says where it went |
| **Manager** — Approvals inbox / Task Hub | **Stage 1 of 2**: requests of their direct reports. Approving sends it to HR; rejecting ends it. The punches change only at the last stage |
| **HR** — same inbox | **Stage 2 of 2 (PENDING_HR)**: needs the new DB permission **`attendance.regularisation.hr`** ("HR verify regularisation"). The summary shows which manager approved |
| **Over-quota role** (e.g. Group HR) — same inbox | **PENDING_ESCALATION**, stage 2 of 2, instead of HR. The summary says "Above the monthly cap: request X with Y allowed this period". Only that role (or super admin / developer) decides it; the role is copied onto the request when it is raised |
| **HR raising for an employee** | Starts at **stage 2** and is **not** held to the window. The monthly cap still applies (refused, or the over-quota role). An employee **with no reporting manager** also starts at stage 2 |
| **Cut-off (B7, decision)** | A day in a **closed** attendance period stays refused (no "next cycle" queue). The window is now the documented deadline |
| **Old requests** | Existing rows are **1 of 1** and are decided once by the manager or HR, as before |

| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 24") | `company_attendance_policies.regularisationWindowDays`, `.regularisationOverQuotaApproverRole`; `regularisations.currentStageOrder`, `totalStages`, `raisedByUserId`, `overQuotaNote`, `approverRoleId`, `managerDecidedByName/At/Remarks`; permission `attendance.regularisation.hr` granted to every role with `attendance.manage`, **plus `COMPANY_HR` and `ADMIN`** (they decided regularisations before by role code). Applied on local; re-run safe |
| API | `GET api/attendance/regularise/allowance?attendanceRecordId=` (attendance.mark); `GET api/attendance/settings/roles` (attendance.manage). `POST api/attendance/regularise` now returns where the request went. Decisions stay on `POST api/approvals/{id}/decide` → `ApprovalEngine` (`REGULARISATION`) |
| Code | `RegularisationRules` (stages, who may decide, window count via `fn_EmployeeDaySchedule`, quota per period); `AttendanceService.SubmitRegularisationAsync` / `GetRegularisationAllowanceAsync`; inbox shows stage, tier and over-quota note. Frontend: `RegularisationPolicySection`, `RegularisationAllowanceNote` |

### Verification
- `dotnet build` ✔, `dotnet test` **112/112** ✔ (6 new: stage 1 → stage 2 applies punches only at the end, manager-only
  stage 1 / no self-decision, HR permission, manager rejection, over-quota role only, old 1-of-1 requests), `tsc --noEmit` ✔.
- Throwaway clone (dropped), Amina Gitau (comp-lch-01), window 2, today 4 Oct:
  - 29 Sep → refused: "the limit is 2 working day(s) after the date and 4 have passed".
  - 2 Oct → allowed (1 day left) → **manager's inbox only** (1/2) → HR refused at stage 1 → manager approves → manager
    refused at stage 2 → HR approves → **PRESENT 08:00–17:00**.
  - Cap 1, no role → refused "Regularisation limit reached: 1 allowed in this attendance period".
  - Cap 1, role Group HR → accepted, "to the reporting manager, then Group HR (above the monthly cap)". HR raising
    1 Oct → starts at stage 2 with the role. HR raising 30 Sep (outside the window) → accepted (exempt).
  - Group HR's inbox shows PENDING_HR and PENDING_ESCALATION items. Group HR approves an escalation → **PRESENT**.

### Things to know
- **Who holds `attendance.regularisation.hr`:** roles with `attendance.manage` (HR Manager, Group HR, HR VP, Developer,
  Super Admin) plus Company HR and Admin. **Finance roles lose** the ability they had by role level. Change it in Roles &
  Permissions if needed.
- On local, the `hr` demo user (Caroline) now belongs to **comp-makl-01**, so her inbox shows nothing for comp-lch-01.
  Use an HR user of the employee's company.
- **Already the case before (not changed):** the decide endpoint checks the permission, not the company. An HR user of
  another company who knows a request's id could decide it. The same holds for on duty / work from home (Step 23).
- **Quota counting is unchanged:** every regularisation in the attendance period counts, rejected ones included.
- Still hard-coded in the inbox (flagged, not changed): 2-day SLA for regularisations, fallback texts "General Healthcare",
  "Direct Supervisor", "Line Supervisor"; old Out Duty rows are still recognised by the words "Out Duty".
- **No email** on submit or decision yet.
- **Not checked in a browser.**

### Step 25 — done, see §36

## 36. Step 25 — completion notes (2026-10-04): manager monthly attendance confirmation

Staff Attendance Policy §9 ("locked after manager confirmation") and §10 (manager confirms before cut-off; payroll only
on confirmed attendance). Closes todo C9, B9, B13.

**Before:** no confirmation step; approving payroll locked the period whatever its state.

### Delivered
| Who | What |
|---|---|
| **HR** — Attendance Policies → General settings (next to *Close Month*) | **Managers must confirm their team's attendance before payroll is approved** (default **off** — nothing changes until switched on) |
| **Manager** — Attendance → **Confirm Team Attendance** (`/attendance/team-confirmation`) | Pick any date of the period. Each active direct report with present, late, absent (LOP), half days, leave, OD / WFH and pending requests; rows with exceptions are highlighted. **Confirm my team's attendance** (asks once more, showing the open exceptions). Shows who confirmed and when; confirming twice is refused |
| **HR** — same page | **All teams** of the period: team size, exceptions, confirmed by / at, *no login* marker. **Open** any team and confirm it **on the manager's behalf** (recorded under HR's name), confirm the **No reporting manager** team, or **Withdraw** a confirmation (e.g. after a late correction) |
| **Payroll** — Payroll → approve cycle | With the setting on, approval is **refused** while a team of that attendance period is unconfirmed; the message names the first five teams and how many more. **Draft runs are not affected** |

| Area | Change |
|---|---|
| DB (`docs/db_changes.sql`, block "Policies Step 25") | `company_attendance_policies.requireManagerConfirmationBeforeLock`; `attendance_period_confirmations` (unique per company, period start, manager; manager NULL = no-manager team); `sp_GetTeamConfirmationStatus`, `sp_GetTeamAttendanceSummary`; permission **`attendance.confirm.team`** (line manager roles `role_mgr`, `ems` and every role with `attendance.manage`); menu *Confirm Team Attendance* under Attendance. Applied twice on local |
| API | `GET api/attendance/team-confirmation?periodDate=&managerEmployeeId=&noManager=`, `POST …/confirm`, `DELETE …/{id}` (HR only) — all `attendance.confirm.team`; HR = `attendance.manage`. `POST api/payroll/cycles/{month}/approve` now returns **400 with the reason** instead of a 500 |
| Code | `TeamConfirmationService`, `TeamConfirmationRules` (status, payroll check, message); `PayrollService.ApproveCycleAsync` calls the check. Frontend: `modules/attendance/team-confirmation/*`, setting in `AttendanceSettingsTab`; the three payroll approve screens now show `errors[0]` |

### Verification
- `dotnet build` ✔, `dotnet test` **115/115** ✔ (3 new: all confirmed, unconfirmed named incl. the no-manager team,
  more than five), `tsc --noEmit` ✔.
- Throwaway clone (dropped), comp-makl-01, September, setting on:
  - Approve → refused, "not confirmed by 6 team(s): … and 1 more".
  - Ranmeet (manager) sees his 19 reports → confirms → a second confirmation is refused. Asking for another
    manager's team still gives his own. He cannot withdraw (HR only).
  - Approve → refused naming the 5 left. HR confirms them, including the no-manager team. HR withdraws one →
    refused naming only that one. HR confirms it again → **approved (LOCKED)**.
  - comp-agi-01 (setting off) → approved as before.

### Things to know
- **Most managers have no login**: in comp-makl-01, 5 of 6 managers cannot sign in. With the setting on, HR will confirm
  most teams on their behalf until those managers get accounts. The HR list marks them *no login*.
- A team = **active** direct reports **in the company** (by `reportingManagerId`). A manager in another company still
  confirms their reports here, by switching to that company.
- Confirming does not lock or freeze anything. Later corrections still go through. HR can withdraw and ask again.
  Nothing flags "changed after confirmation".
- Exceptions counted: absent (LOP) and half days, pending regularisations, pending on duty / work from home requests.
  Late days are shown but not counted as exceptions.
- In **PERIOD_END** close-month mode the period still closes by date. The setting only holds back **payroll approval**.
- **No reminder email** to managers yet.
- Flagged, not changed: `RunPayrollAsync` maps country names to codes in code ("INDIA" → IN, … default KE).
- **Not checked in a browser.**

### Next: Step 26 — Roster publication lead time
