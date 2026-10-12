# Reports — Status & Backlog

Last updated: 2026-10-12

This file tracks which reports exist in the system and which are planned. New reports are picked up from the backlog one at a time.

---

## 1. Standard report features (reference: `/reports/employee-reports`)

Every new report must work the same way as the Employee Report (`frontend/src/modules/reports/employee-reports`, `backend/Api/Controllers/EmployeeReportController.cs`).

| # | Feature | How Employee Report does it |
|---|---|---|
| F1 | **Filter bar** | Company, department, job title, employment type, location, status. Filter options come from a `GET filters` endpoint (masters only, nothing hardcoded) |
| F2 | **Date filter** | Picker for which date field to filter on, plus From/To dates and an **All time** toggle |
| F3 | **Apply-first loading** | No data loads until the user applies filters |
| F4 | **Search** | Free-text search across the main columns |
| F5 | **Column chooser** | Columns are grouped (e.g. Employee / Job / Dates / Personal / Account / Pay). Some are visible by default, others optional. Hidden columns are saved in `localStorage` |
| F6 | **Sorting** | Click a column header to sort (`SortBy` sent to the server) |
| F7 | **Server-side paging** | Page size comes from list settings (default 50; options 10/25/50/100). Shows "x–y of N" |
| F8 | **Export** | Excel (`xlsx`) and CSV through a `GET export` endpoint. Exports only the visible columns (`Columns` param) with the same filters applied |
| F9 | **Row drill-down** | Clicking a row opens the related drawer, e.g. `EmployeeDossierDrawer` |
| F10 | **RBAC** | Shown through a DB-driven menu entry. Without permission the user sees "Access to This Report Is Restricted". Data is limited to companies in the user's scope |
| F11 | **No static values** | Columns, filters and labels come from masters: pay components, leave types, statutory configs, currency, country |

**Backend pattern:** a `<Name>ReportController` with `GET filters`, `GET` (data, paged) and `GET export`. A `<Name>ReportQuery` DTO carries `CompanyId, DateField, DateFrom, DateTo, AllTime, Search, <filters>, SortBy, Page, PageSize, Columns, Format`. Logic lives in a `<Name>ReportService`.

---

## 2. Reports already in the system

| Area | Report | Location | Std features (§1) |
|---|---|---|---|
| Employee | Employee Report | `reports/employee-reports`, `EmployeeReportController` | ✅ Reference |
| Attendance | Attendance Register | `reports/attendance-register`, `AttendanceRegisterController` | Mostly |
| Attendance | Attendance Summary | `reports/attendance-summary`, `AttendanceSummaryController` | Mostly |
| Attendance | Roster vs Actual | `reports/shift-overtime`, `TimeAttendanceReportsController` | Partial |
| Attendance | Absenteeism & Punctuality | `reports/attendance`, `ReportsController /absenteeism` | Partial |
| Shift / OT | Overtime Register | `reports/shift-overtime`, `TimeAttendanceReportsController` | Partial |
| Leave | Statutory Leave Liability Valuation | `reports/leave-liability`, `ReportsController /leave-liability` | Partial |
| Workforce | Headcount Growth & Attrition | `reports/workforce`, `ReportsController /headcount` | Partial |
| Payroll | Statutory Tax & Remittance Audit | `reports/payroll`, `ReportsController /statutory` | Partial |
| Payroll | Cycle export / payslip PDF / currency summary | `PayrollController` | n/a (not a report screen) |
| Separation | Exit Stats & Export | `reports/separation`, `SeparationReportController` | Mostly |
| Recruitment | Recruitment Pipeline Velocity | `ReportsController /available` | Partial |
| Compliance | Regulatory Credential Compliance | `ReportsController /available` | Partial |

> The "Std features" column is a first-pass estimate. Before closing a report, check it against every item in §1 (F1–F11).

---

## 3. Backlog — reports to build later

Status key: ⬜ Not started · 🟨 In progress · ✅ Done

### P1 — Monthly HR & payroll close

| # | Report | Purpose | Main data source | Status |
|---|---|---|---|---|
| R1 | Payroll Register (salary sheet) | One row per employee with each pay component, gross, deductions and net | Payroll cycle, pay components | ⬜ |
| R2 | Payroll Variance (MoM) | This month vs last month: joiners, leavers, component changes | Payroll cycles | ⬜ |
| R3 | Statutory Remittance Schedules | One schedule per configured statutory deduction (from compliance masters) | Compliance masters, payroll | ⬜ |
| R4 | Bank Transfer / Net Pay Advice | Net pay by bank and branch, ready to upload | Payroll, employee bank details | ⬜ |
| R5 | Leave Balance Report | Opening, accrued, taken, lapsed and closing balance per employee and leave type | Leave, leave accrual | ⬜ |
| R6 | Leave Transactions | Leave applied, approved and rejected in a date range | Leave | ⬜ |
| R7 | Joiners & Leavers | New hires and exits by company, department and grade | Employees, separation | ⬜ |

### P2 — Operations (hospital-specific)

| # | Report | Purpose | Main data source | Status |
|---|---|---|---|---|
| R8 | Night Allowance Register | Night shifts and allowance per employee | NightAllowance | ⬜ |
| R9 | Comp-Off Register | Comp-off earned, used and expired | CompOff | ⬜ |
| R10 | Late In / Early Out / Missed Punch | Attendance exceptions with regularisation status | Attendance | ⬜ |
| R11 | Shift Coverage | Rostered vs required per ward/department per shift | Roster, shifts | ⬜ |
| R12 | Biometric Punch Log | Raw and unmatched punches, for troubleshooting | Biometrics | ⬜ |
| R13 | Loan Outstanding | Loan amount, EMIs deducted, balance | Loans | ⬜ |
| R14 | Benefits Enrolment | Enrolment per benefit and its cost | Benefits | ⬜ |

### P3 — Lifecycle & compliance

| # | Report | Purpose | Main data source | Status |
|---|---|---|---|---|
| R15 | Probation / Confirmation Due | Upcoming confirmations (ties to the future Confirmation workflow) | Employees, TeamConfirmation | ⬜ |
| R16 | Document / License Expiry | Licences and IDs expiring in 30/60/90 days (clinical staff) | Employee documents | ⬜ |
| R17 | Onboarding Status | Pending onboarding tasks per joiner | Onboarding | ⬜ |
| R18 | Requisition (SRF) Status | Open and filled positions, time to fill | Requisitions | ⬜ |
| R19 | Offer Acceptance / Source of Hire | Recruitment funnel by source | Recruitment | ⬜ |
| R20 | Approval SLA / Pending Ageing | Tasks pending longer than N days, by approver | Task, ApprovalSla | ⬜ |
| R21 | Audit Trail Report | Who changed what in employee, pay and masters data | Audit | ⬜ |

### P4 — Analytics

| # | Report | Purpose | Main data source | Status |
|---|---|---|---|---|
| R22 | Attrition Analysis | Attrition by department, grade and tenure band, with resignation reasons | Separation, employees | ⬜ |
| R23 | Cost to Company by Department | Total employee cost per department | Payroll | ⬜ |
| R24 | Overtime Cost Trend | Monthly overtime cost | Overtime, payroll | ⬜ |
| R25 | Diversity | Gender, age band, nationality | Employees | ⬜ |

### Upgrade existing reports to the standard (§1)

| # | Item | Status |
|---|---|---|
| U1 | Bring the "Partial" reports in §2 up to F1–F11 (column chooser, server paging, xlsx/csv export of visible columns, drill-down) | ⬜ |

---

## 4. Recommended order

1. **R1 Payroll Register**, **R5 Leave Balance**, **R16 Document/License Expiry**
2. The rest of P1 (R2–R4, R6, R7)
3. P2, then P3, then P4
4. U1, done alongside or after the above

## 5. Definition of done (per report)

- [ ] Meets every item in §1 (F1–F11)
- [ ] Menu entry is DB-driven under Reports, with RBAC permission seeded (script added to `docs/db_changes.sql`)
- [ ] Filters and columns read from masters; nothing hardcoded
- [ ] Export matches the on-screen columns and filters
- [ ] Browser test steps added to the management artifact checklist
- [ ] Status updated in this file and in `docs/policies.md`
