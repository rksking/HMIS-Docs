# HRMS Solution — Remaining Feature Separation & Modularization TODO

> **STRICT COMPLIANCE DIRECTIVE (`docs/follow.md`)**:
> 1. **100% Database-Driven**: Menus, permissions, and operational data must originate strictly from SQL Server. Zero hardcoded items, fake fallbacks, or client-side mock arrays.
> 2. **Layout Standards**: Root layout must be `<div className="space-y-6">` with an unboxed header, inline title/subtitle, and single horizontal action bar (`flex items-center gap-2`).
> 3. **50% Slide-Over Drawers (Rule 7)**: All forms, modals, creation, and inspection flows must use `SideDrawer.tsx` (50% desktop width). Centered popups are strictly prohibited.
> 4. **No Static Values (Rule 10)**: All KPI cards, summaries, and stats must be computed from real database data. All discovered mock arrays and fake strings must be eliminated.
> 5. **Permissions & RBAC (Rule 8)**: Standard dot-notation `<module>.<action>` registered in the `permissions` table, with all submenus registered in the `Menus` table under their respective parent.
> 6. **Build Integrity**: Zero TypeScript errors (`npx tsc --noEmit`) and zero backend build errors (`dotnet build`).

---

## Executive Summary of Remaining Modules

| # | Module | Primary Monolithic File | Current Lines | Current Internal Tabs | Proposed Sub-Routes | Priority |
|---|---|---|---|---|---|---|
| **1** | **Leave Management** | `frontend/src/modules/leave/` | Modularized | 3 modular sub-views | `/leave`, `/leave/calendar`, `/leave/balances` | ✅ **Completed** |
| **2** | **Organisation Masters** | `frontend/src/modules/org-masters/` | Modularized | 8 modular sub-views | `/orgmasters`, `/orgmasters/departments`, `/orgmasters/facilities`, `/orgmasters/job-titles`, `/orgmasters/grades`, `/orgmasters/employment-types`, `/orgmasters/pay-components`, `/orgmasters/cost-centres`, `/orgmasters/banks` | ✅ **Completed** |
| **3** | **Shifts & Schedules** | `frontend/src/modules/shifts/ShiftsAndSchedulesView.tsx` | **468 lines** | 5 tabs (`overview`, `master`, `schedules`, `bulk`, `search`) | `/shifts`, `/shifts/master`, `/shifts/schedules`, `/shifts/roster`, `/shifts/lookup` | **Medium** |
| **4** | **Company Setup** | `frontend/src/modules/company-setup/CompanySetupView.tsx` | **461 lines** | 3 tabs (`dashboard`, `all_companies`, `approval_routes`) | `/companysetup`, `/companysetup/companies`, `/companysetup/routing` | **Medium** |
| **5** | **Onboarding** | `frontend/src/modules/onboarding/` | Modularized | 3 modular sub-views | `/onboarding`, `/onboarding/configuration`, `/onboarding/templates` | ✅ **Completed** |
| **6** | **Reports Hub** | `frontend/src/modules/reports/` | Modularized | 4 modular sub-views | `/reports`, `/reports/attendance`, `/reports/payroll`, `/reports/leave-liability` | ✅ **Completed** |

---

## Detailed Plans & Walkthroughs for Review

---

### Module 1: Leave Management (`/leave`) — ✅ Completed

#### 1. Implemented Architecture & Database Alignment
- **Database Menu Alignment (`menus` table in SQL Server)**:
  - Parent: `Leave Management` (`id: menu-leave`, `code: LEAVE`, `route: -`, `icon: Calendar`)
  - **Child 1**: `Applications Register` (`id: menu-leave-req`, `code: LEAVE_APPS`, `route: /leave`, `sortOrder: 1`, `icon: FileText`)
  - **Child 2**: `Staff Leave Calendar` (`id: menu-leave-cal`, `code: LEAVE_CALENDAR`, `route: /leave/calendar`, `sortOrder: 2`, `icon: CalendarDays`)
  - **Child 3**: `Balances & Entitlements` (`id: menu-leave-bal`, `code: LEAVE_BAL`, `route: /leave/balances`, `sortOrder: 3`, `icon: Layers`)
  - Redundant duplicate entries (`menu-b7816ab4faf54`, `menu-6ea0e02f0ac24`, `menu-91f1fcebb7c74`, `menu-leave-apply`) deactivated (`isVisible: 0`).
  - **Centralized Approvals Architecture**: Approvals are unified under `/task` (Enterprise Task Hub with `entityType === "LEAVE"` and `TaskDecisionDrawer`). Header features live counter badge fast-linking directly to `/task?tab=LEAVE`.

#### 2. Implemented Frontend Modular Structure
```
frontend/src/modules/leave/
├── LeaveView.tsx                     # Clean wrapper delegating to LeaveApplicationsView
├── applications/
│   └── LeaveApplicationsView.tsx     # Workforce Leave Applications Register with dynamic filters, status badges, row inspect & cancel
├── calendar/
│   └── LeaveCalendarView.tsx         # Monthly Interactive Calendar Grid with Gazetted Holidays & Department Absence Roster
├── balances/
│   └── LeaveBalancesView.tsx         # Organization-wide Balances Ledger, Monthly Utilisation Heatmap, Statutory Policies Catalog
├── components/
│   ├── LeaveNavHeader.tsx            # Multi-company filter, FY selector, pending approvals badge, sub-route navigation pills
│   ├── LeaveKpiCards.tsx             # 100% computed dynamic metrics (zero static mocks)
│   ├── ApplyOnBehalfDrawer.tsx       # 75% slide-over right drawer for HR & Managers to proxy-submit leave with contract entitlement verification
│   ├── LeaveAdjustmentDrawer.tsx     # 50% slide-over right drawer for HR balance manual credit/debit audit adjustments
│   ├── LeaveDetailDrawer.tsx         # Slide-over right drawer for inspecting timeline, relief handover, and cancellation
│   ├── LeaveWorkflowGuideDrawer.tsx  # 75% slide-over right drawer for Employment Act 2007 (s.28-30) and approval matrix guidance
│   └── MonthlyUtilizationTable.tsx   # Annual 12-month leave utilization matrix with quota burn progress
└── index.ts                          # Clean public exports
```

#### 3. App Router Endpoints
- `src/app/(dashboard)/leave/page.tsx` -> renders `<LeaveApplicationsView />`
- `src/app/(dashboard)/leave/applications/page.tsx` -> renders `<LeaveApplicationsView />`
- `src/app/(dashboard)/leave/calendar/page.tsx` -> renders `<LeaveCalendarView />`
- `src/app/(dashboard)/leave/balances/page.tsx` -> renders `<LeaveBalancesView />`

#### 4. Design Standards & Width Directive
- All drawers use `SideDrawer.tsx` with light/modern chrome.
- Drawers with rich multi-section content (`ApplyOnBehalfDrawer`, `LeaveWorkflowGuideDrawer`) use 75% width per user preference.
- All KPIs are 100% dynamic and computed from live SQL Server backend data. Zero mock data.
- Built and validated with 0 TypeScript errors (`npx tsc --noEmit`) and 0 backend errors (`dotnet build HrmsSys.sln`).

---

### Module 2: Organisation Masters (`/orgmasters`) — ✅ **Completed**

#### 1. Current State & Modularization Summary
- **Monolithic File Refactored**: `frontend/src/modules/org-masters/OrgMastersView.tsx` reduced from **2,180 lines** down to **~200 lines**, delegating to dedicated sub-views and `OrgMastersDashboard`.
- **Modular Sub-Views Implemented**:
  1. `frontend/src/modules/org-masters/departments/DepartmentsMasterView.tsx`: Full tree hierarchy, parent division apex indicators, department head assignments, active headcount, search, pagination, and `DepartmentDrawer`.
  2. `frontend/src/modules/org-masters/facilities/FacilitiesMasterView.tsx`: Hospital branch registry, city, street address, assigned staff, and `LocationDrawer`.
  3. `frontend/src/modules/org-masters/job-titles/JobTitlesMasterView.tsx`: Designations, clinical vs corporate filters/badges, active headcount, and `JobTitleDrawer`.
  4. `frontend/src/modules/org-masters/grades/GradesMasterView.tsx`: Compensation bands, salary floor/ceiling (`salaryMin`/`salaryMax`), and `GradeDrawer`.
  5. `frontend/src/modules/org-masters/employment-types/EmploymentTypesMasterView.tsx`: Contract types, statutory NSSF/SHIF badges, pension scheme, probation/notice days, and `EmploymentTypeDrawer`.
  6. `frontend/src/modules/org-masters/pay-components/PayComponentsMasterView.tsx`: Earnings vs Deductions pills, tax/pensionable flags, fixed vs % calculation, and `PayComponentDrawer`.
  7. `frontend/src/modules/org-masters/cost-centres/CostCentresMasterView.tsx`: GL account codes, expense descriptions, linked staff, and `CostCentreDrawer`.
  8. `frontend/src/modules/org-masters/banks/BanksMasterView.tsx`: Commercial banks & clearing branches toggle, multi-country filter (Kenya, UAE, India), SWIFT/BIC, CBK EFT/IFSC clearing codes, and `BankDrawer` & `BankBranchDrawer`.
- **Navigation & Layout Adherence**:
  - `OrgMastersNavHeader.tsx`: Single horizontal action bar with company selector, refresh, cross-company duplicate masters drawer, setup guide drawer, and clean horizontal navigation pills across all 9 sub-masters.
  - All drawers adhere strictly to `SideDrawer.tsx` (50% desktop slide-over right drawer).
  - 100% database-driven with zero static mock values.
  - Verified 0 TypeScript compilation errors (`npx tsc --noEmit`) and 0 backend build errors (`dotnet build HrmsSys.sln`).
  - `contract-types`: Full-time, locum, probation, intern, consultant contract types
  - `pay-components`: Earning vs deduction formulas, taxable flags, statutory rules
  - `cost-centres`: Accounting codes, cost allocations
  - `banks`: Commercial banks, clearing codes, branch directories
  - `overview`: Master statistics overview
- **Database Menu Discrepancies**:
  - The `Menus` table currently has only a single entry: `MST_ORG -> /orgmasters`.

#### 2. Target Database Menu Architecture (100% DB-Driven)
Parent Menu: `Masters & Configuration` (`code: MASTERS_CONFIG`, `icon: Settings`, `sortOrder: 15`, `route: -`)
- **Child 1**: `Master Overview` (`code: MST_ORG_OVERVIEW`, `route: /orgmasters`, `sortOrder: 1`, `icon: LayoutDashboard`)
- **Child 2**: `Departments & Units` (`code: MST_DEPARTMENTS`, `route: /orgmasters/departments`, `sortOrder: 2`, `icon: Network`)
- **Child 3**: `Hospital Facilities & Branches` (`code: MST_FACILITIES`, `route: /orgmasters/facilities`, `sortOrder: 3`, `icon: MapPin`)
- **Child 4**: `Job Titles & Designations` (`code: MST_JOB_TITLES`, `route: /orgmasters/job-titles`, `sortOrder: 4`, `icon: Briefcase`)
- **Child 5**: `Salary Grades & Bands` (`code: MST_GRADES`, `route: /orgmasters/grades`, `sortOrder: 5`, `icon: Award`)
- **Child 6**: `Contract & Employment Types` (`code: MST_CONTRACT_TYPES`, `route: /orgmasters/employment-types`, `sortOrder: 6`, `icon: FileSpreadsheet`)
- **Child 7**: `Pay Components & Allowances` (`code: MST_PAY_COMPONENTS`, `route: /orgmasters/pay-components`, `sortOrder: 7`, `icon: DollarSign`)
- **Child 8**: `Cost Centres & Units` (`code: MST_COST_CENTRES`, `route: /orgmasters/cost-centres`, `sortOrder: 8`, `icon: Landmark`)
- **Child 9**: `Banks & Clearing Branches` (`code: MST_BANKS`, `route: /orgmasters/banks`, `sortOrder: 9`, `icon: Building2`)

#### 3. Planned Frontend Modular Structure
```
frontend/src/modules/org-masters/
├── OrgMastersView.tsx                # Clean executive dashboard with fast links to each master
├── departments/
│   └── DepartmentsMasterView.tsx     # Tree hierarchy, parent departments, department heads
├── facilities/
│   └── FacilitiesMasterView.tsx      # Multi-facility registry, branch codes, bed capacity
├── job-titles/
│   └── JobTitlesMasterView.tsx       # Clinical vs non-clinical titles, active counts
├── grades/
│   └── GradesMasterView.tsx          # Salary bands, grade levels
├── employment-types/
│   └── EmploymentTypesMasterView.tsx # Contract definitions, probation durations
├── pay-components/
│   └── PayComponentsMasterView.tsx   # Allowances, deductions, statutory flags
├── cost-centres/
│   └── CostCentresMasterView.tsx     # Accounting cost allocations
├── banks/
│   └── BanksMasterView.tsx           # Commercial banks and branch clearing codes
├── components/
│   ├── OrgMastersKpiCards.tsx        # 100% dynamic master counts from database
│   ├── DepartmentDrawer.tsx          # 50% slide-over right drawer
│   ├── LocationDrawer.tsx            # 50% slide-over right drawer
│   ├── JobTitleDrawer.tsx            # 50% slide-over right drawer
│   └── GradeDrawer.tsx               # 50% slide-over right drawer
└── index.ts                          # Public exports
```

#### 4. App Router Endpoints
- `src/app/(dashboard)/orgmasters/page.tsx` -> renders `<OrgMastersView />`
- `src/app/(dashboard)/orgmasters/departments/page.tsx` -> renders `<DepartmentsMasterView />`
- `src/app/(dashboard)/orgmasters/facilities/page.tsx` -> renders `<FacilitiesMasterView />`
- `src/app/(dashboard)/orgmasters/job-titles/page.tsx` -> renders `<JobTitlesMasterView />`
- `src/app/(dashboard)/orgmasters/grades/page.tsx` -> renders `<GradesMasterView />`
- `src/app/(dashboard)/orgmasters/employment-types/page.tsx` -> renders `<EmploymentTypesMasterView />`
- `src/app/(dashboard)/orgmasters/pay-components/page.tsx` -> renders `<PayComponentsMasterView />`
- `src/app/(dashboard)/orgmasters/cost-centres/page.tsx` -> renders `<CostCentresMasterView />`
- `src/app/(dashboard)/orgmasters/banks/page.tsx` -> renders `<BanksMasterView />`

---

### Module 3: Shifts & Schedules (`/shifts`)

#### 1. Current State & Pain Points
- **Monolithic File**: `frontend/src/modules/shifts/ShiftsAndSchedulesView.tsx` (468 lines).
- **Internal Tabs**:
  - `overview`: Shifts Overview & Dashboard KPIs
  - `master`: Shift Master Definitions (start, end, break, grace periods)
  - `schedules`: Work Schedules & Shift Rotation Patterns (e.g. 24/7 hospital rotations, night shifts)
  - `bulk`: Bulk Shift Roster Assignment by Department/Location
  - `search`: Individual Employee Shift Lookup & Roster Search
- **Database Menu Discrepancies**:
  - Currently represented as a single link `MST_SHIFTS -> /shifts` under `Masters & Configuration`.

#### 2. Target Database Menu Architecture (100% DB-Driven)
Parent Menu: `Masters & Configuration` (or dedicated `Shift Scheduling` parent)
- **Child 1**: `Shifts & Roster Overview` (`code: SHIFT_OVERVIEW`, `route: /shifts`, `sortOrder: 1`, `icon: LayoutDashboard`)
- **Child 2**: `Shift Master Definitions` (`code: SHIFT_MASTER`, `route: /shifts/master`, `sortOrder: 2`, `icon: Clock`)
- **Child 3**: `Work Schedules & Patterns` (`code: SHIFT_SCHEDULES`, `route: /shifts/schedules`, `sortOrder: 3`, `icon: Calendar`)
- **Child 4**: `Bulk Roster Assignment` (`code: SHIFT_ROSTER`, `route: /shifts/roster`, `sortOrder: 4`, `icon: Users`)
- **Child 5**: `Employee Shift Lookup` (`code: SHIFT_LOOKUP`, `route: /shifts/lookup`, `sortOrder: 5`, `icon: Search`)

#### 3. Planned Frontend Modular Structure
```
frontend/src/modules/shifts/
├── ShiftsAndSchedulesView.tsx        # Overview dashboard with roster KPIs
├── master/
│   └── ShiftMasterView.tsx           # Shift hours, grace periods, night differentials
├── schedules/
│   └── WorkSchedulesView.tsx         # Weekly/monthly rotating schedule templates
├── roster/
│   └── BulkRosterAssignmentView.tsx  # Multi-employee hospital shift planner
├── lookup/
│   └── EmployeeShiftLookupView.tsx   # Per-staff shift search and calendar inspector
├── components/
│   ├── ShiftsKpiCards.tsx            # 100% dynamic real-time shift stats
│   ├── AddEditShiftDrawer.tsx        # 50% slide-over right drawer
│   └── AddEditScheduleDrawer.tsx     # 50% slide-over right drawer
└── index.ts                          # Public exports
```

---

### Module 4: Company Setup & Multi-Tenant Registry (`/companysetup`)

#### 1. Current State & Pain Points
- **Monolithic File**: `frontend/src/modules/company-setup/CompanySetupView.tsx` (461 lines).
- **Internal Tabs**:
  - `dashboard`: Multi-company analytics, employee distribution, compliance status
  - `all_companies`: Healthcare facility & legal entity table
  - `approval_routes`: Cross-entity approval routing and matrices

#### 2. Target Database Menu Architecture (100% DB-Driven)
Under `Masters & Configuration` -> `Company Setup`:
- **Child 1**: `Company Analytics & Hierarchy` (`code: COMP_DASHBOARD`, `route: /companysetup`, `sortOrder: 1`, `icon: LayoutDashboard`)
- **Child 2**: `Entities & Hospitals Registry` (`code: COMP_REGISTRY`, `route: /companysetup/companies`, `sortOrder: 2`, `icon: Building2`)
- **Child 3**: `Cross-Company Routing` (`code: COMP_ROUTING`, `route: /companysetup/routing`, `sortOrder: 3`, `icon: GitMerge`)

---

### Module 5: Onboarding Pipeline & Templates (`/onboarding`) — ✅ **COMPLETED**

#### 1. Architecture & Feature Separation Executed
- **Database-Driven Menus (`docs/follow.md`)**: Fully verified with parent `menu-onboarding` and 5 children in SQL Server `menus` table:
  - `ONBOARD_PIPELINE`: Onboarding Pipeline (`/onboarding`)
  - `ONBOARD_CONFIG`: Onboarding Configuration (`/onboarding/configuration`)
  - `ONBOARD_TEMPLATES`: Onboarding Templates (`/onboarding/templates`)
  - `ONBOARD_DOCS`: Document Types (`/documents`)
  - `ONBOARD_REPORTS`: Onboarding Reports (`/reports`)
- **Backend Endpoints Added & Verified**:
  - `GET /api/Onboarding/templates`: Returns real workflow templates (`TPL-ONB-GEN`, `TPL-ONB-CLIN`, `TPL-ONB-NURSE`, `TPL-ONB-NONCLIN`) with step count, SLA expiry days, reminder intervals, and default flags.
  - `GET /api/Onboarding/configuration`: Returns database transition rules (`fromStage`, `toStage`, `allowedRole`, `conditionDescription`), automated email invitation/reminder templates, and SLA policy data.
- **Dedicated Modular Views Created**:
  - `frontend/src/modules/onboarding/templates/OnboardingTemplatesView.tsx`: Displays multi-template card grid, real-time search, step counters, and 50% slide-over right drawer (`SideDrawer.tsx`) for inspecting 7-step candidate onboarding journeys and statutory guardrails.
  - `frontend/src/modules/onboarding/configuration/OnboardingConfigView.tsx`: Tabbed configuration hub for stage progression rules & RBAC permissions table, automated email triggers & merge tag inspector, and SLA validity policies.
- **App Router Pages Connected**:
  - `src/app/(dashboard)/onboarding/page.tsx` -> renders `<OnboardingView />` (Pipeline)
  - `src/app/(dashboard)/onboarding/templates/page.tsx` -> renders `<OnboardingTemplatesView />` (Templates)
  - `src/app/(dashboard)/onboarding/configuration/page.tsx` -> renders `<OnboardingConfigView />` (Configuration)
- **UI/UX Compliance**:
  - Root layout adheres to `<div className="space-y-6">` with unboxed headers.
  - Single horizontal action bar (`flex items-center gap-2`).
  - 50% slide-overs via `SideDrawer.tsx`. Zero centered popups.
  - Zero mock data; 100% sourced from backend EF Core services and SQL Server.
  - Build status: `dotnet build` (0 errors), `npx tsc --noEmit` (0 errors), all endpoints returning HTTP 200.

---

### Module 6: Reports Hub (`/reports`) — ✅ **COMPLETED**

#### 1. Architecture & Feature Separation Executed
- **Database-Driven Menus (`docs/follow.md`)**: Configured parent `REPORTS` in SQL Server with 4 active database children:
  - `REP_WORKFORCE`: Workforce Analytics (`/reports`)
  - `REP_ATTEND`: Attendance Heatmap (`/reports/attendance`)
  - `REP_PAYROLL`: Payroll Reconciliation (`/reports/payroll`)
  - `REP_LEAVE_LIAB`: Leave Liability Audit (`/reports/leave-liability`)
- **Dedicated Modular Views Created**:
  - `frontend/src/modules/reports/workforce/WorkforceAnalyticsView.tsx`: Real-time workforce census, department distributions, gender diversity, contract types, and turnover rates.
  - `frontend/src/modules/reports/attendance/AttendanceReportsView.tsx`: Biometric punch & roster aggregation, punctuality metrics, 14-day daily trends table with pagination and date range filter.
  - `frontend/src/modules/reports/payroll/PayrollReportsView.tsx`: Month-selectable statutory audit ledger (KRA PAYE, NSSF, SHIF, Affordable Housing Levy) with detailed authority remittance table and pagination.
  - `frontend/src/modules/reports/leave-liability/LeaveLiabilityReportView.tsx`: Untaken leave pool valuation, department liability breakdown, and individual employee balance sheet provisioning ledger.
- **App Router Pages Connected**:
  - `src/app/(dashboard)/reports/page.tsx` -> renders `<WorkforceAnalyticsView />`
  - `src/app/(dashboard)/reports/attendance/page.tsx` -> renders `<AttendanceReportsView />`
  - `src/app/(dashboard)/reports/payroll/page.tsx` -> renders `<PayrollReportsView />`
  - `src/app/(dashboard)/reports/leave-liability/page.tsx` -> renders `<LeaveLiabilityReportView />`
- **UI/UX Compliance**:
  - Root layout adheres to `<div className="space-y-6">` with unboxed headers.
  - Single horizontal action row for Export CSV, Architecture Guide, and Refresh.
  - All drawers use `SideDrawer.tsx` (50% desktop slide-over). Zero centered modals.
  - 100% dynamic KPI stats computed from backend API; zero mock values.
  - Build status: `dotnet build` (0 errors), `npx tsc --noEmit` (0 errors), all 4 endpoints returning HTTP 200.

---

## HRMS Domain Questions for User Alignment

Before commencing execution on the above plans, please confirm:
1. **Execution Order**: Should we proceed with **Module 1 (Leave Management)** first, followed by **Module 2 (Organisation Masters)**?
2. **Leave Approvals Hierarchy**: In the Leave Approvals Queue (`/leave/approvals`), do you require 2-tier approval (Line Manager followed by HRBP/HR Manager), or single-tier based on department matrix?
3. **Organisation Masters Submenus**: For Organisation Masters, would you prefer the sub-masters (Departments, Facilities, Job Titles, Grades, Pay Components, Banks) to appear directly as sub-items in the sidebar under `Masters & Configuration`, or keep `/orgmasters` as an umbrella parent with child links?
