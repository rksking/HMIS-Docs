# HRMS Enterprise System: Implementation TaskSheet

> **Reference Blueprint:** `docs/user_perspectives.md` / `docs/HRMS_SPECIFICATION_AND_ARCHITECTURE.md`  
> **Target Stack:** .NET 10 Core Clean Architecture (Backend Port: 5197) | Next.js 15 App Router (Frontend Port: 4000) | Microsoft SQL Server 2022+  
> **Objective:** Comprehensive, step-by-step operational task list to expand the HRMS from an admin-only platform into a full multi-persona enterprise system (Employee Self-Service, Manager Self-Service, Finance, Compliance, and Executive).

---

## Task Progress Overview

| Phase | Milestone / Focus | Status | Estimated Duration |
|:---:|---|:---:|:---:|
| **Step 01** | Role-Based Access Control (RBAC) & Dynamic Navigation Engine | `Pending` | 1 Day |
| **Step 02** | Employee Self-Service (ESS): Personal Dashboard & Punch Clock | `Pending` | 1.5 Days |
| **Step 03** | Employee Self-Service (ESS): Leave Application & Holiday Calendar | `Pending` | 1 Day |
| **Step 04** | Employee Self-Service (ESS): Itemized Payslips & Loan Statements | `Pending` | 1 Day |
| **Step 05** | Employee Self-Service (ESS): Roster, Shift Swap & Documents | `Pending` | 1 Day |
| **Step 06** | Manager Self-Service (MSS): "My Team" Presence Dashboard | `Pending` | 1 Day |
| **Step 07** | Manager Self-Service (MSS): Unified Approvals Action Center | `Pending` | 1.5 Days |
| **Step 08** | Manager Self-Service (MSS): Staff Requisitions & Probation Reviews | `Pending` | 1 Day |
| **Step 09** | Finance & Treasury: Commercial Bank Disbursement Batch Generator (EFT/KBA) | `Pending` | 1.5 Days |
| **Step 10** | Finance & Treasury: Statutory Tax Return Schedules (KRA P10, NSSF, SHIF) | `Pending` | 1 Day |
| **Step 11** | Clinical Compliance: Medical License Expiry Watchdog & CPD Audit | `Pending` | 1 Day |
| **Step 12** | Executive / C-Suite: Consolidated Group Workforce Intelligence | `Pending` | 1 Day |
| **Step 13** | End-to-End Persona Verification & Production Sign-Off | `Pending` | 1 Day |

---

## Detailed Step-by-Step Task Breakdown

---

### STEP 01: Role-Based Access Control (RBAC) & Dynamic Navigation Engine
> **Goal:** Ensure each user logging in sees only the modules, menus, and data appropriate for their assigned role (`role_super`, `role_hr`, `role_mgr`, `role_emp`, `role_fin`).

- [ ] **Task 1.1: Backend User Context Enrichment**
  - Verify that `backend/Api/Controllers/AuthController.cs` returns the user's `roleId`, `roleName`, `scope` (`ALL`, `TEAM`, `SELF`), and linked `employeeId` in the login JWT response payload.
  - File: `backend/Application/DTOs/Auth/AuthDtos.cs`
- [ ] **Task 1.2: Frontend Auth State & Role Hook**
  - Update `frontend/src/modules/auth/AuthContext.tsx` to store `roleId`, `scope`, and `employeeId`.
  - Create a helper hook `useUserRole()` returning boolean flags: `isSuperAdmin`, `isHR`, `isManager`, `isEmployee`, `isFinance`.
- [ ] **Task 1.3: Dynamic Sidebar Navigation Filtering**
  - Modify `frontend/src/components/layout/Sidebar.tsx`:
    - Add `allowedRoles?: string[]` to `NavItem`.
    - If logged in as `Employee` (`role_emp`), hide all Administrative, Setup, and Org Master menus; show only **My Workplace**, **My Attendance**, **My Leave**, **My Payslips**, and **My Documents**.
    - If logged in as `Manager` (`role_mgr`), display **My Team**, **Approvals**, and employee self-service options, hiding system configuration.
    - If logged in as `Finance` (`role_fin`), expose Payroll runs, Bank files, Statutory rates, and Loan ledgers.
- [ ] **Task 1.4: Route Protection Middleware**
  - Update `frontend/src/middleware.ts` (or client-side route guards) to prevent unauthorized URL access (e.g. an employee navigating directly to `/companysetup` or `/admin/users` gets redirected to `/portal`).

---

### STEP 02: Employee Self-Service (ESS) — Personal Dashboard & Web Punch Clock
> **Goal:** Deliver the primary landing page for hospital employees with live shift status, one-click clock-in/out, and personal KPIs.

- [ ] **Task 2.1: Employee Portal Route Setup**
  - Create the base layout and page for the employee portal:
    - `frontend/src/app/(dashboard)/portal/page.tsx`
    - `frontend/src/modules/portal/PortalView.tsx`
- [ ] **Task 2.2: Live Clock-In / Clock-Out Component**
  - Create `frontend/src/modules/portal/components/ClockInOutCard.tsx`:
    - Digital live clock displaying local time.
    - Status badge: *Not Clocked In*, *Clocked In (08:02 AM)*, *Clocked Out (17:05 PM)*.
    - One-click **Clock In** and **Clock Out** buttons calling existing backend endpoints:
      - `POST /api/attendance/clock-in`
      - `POST /api/attendance/clock-out`
    - Display assigned shift timings for today (*e.g., "Day Shift: 08:00 - 17:00"*).
- [ ] **Task 2.3: Quick KPI Widgets**
  - Add summary cards to the portal home:
    - *Available Annual Leave Days* (fetching from `/api/leave/balances`).
    - *Hours Worked This Month* (calculated from attendance log).
    - *Latest Net Salary Preview* (masked with a toggle icon for privacy).
    - *Active Shift Roster Card*.
- [ ] **Task 2.4: Personal Monthly Attendance Log & Missed-Punch Regularisation**
  - Create `frontend/src/app/(dashboard)/portal/attendance/page.tsx`:
    - Monthly calendar/table showing daily punches, hours worked, late arrival minutes, and status chips.
    - **"Request Regularisation" button**:
      - Opens a modal to submit a missed punch explanation.
      - Calls `POST /api/attendance/regularise` with fields: `attendanceId`, `requestedClockIn`, `requestedClockOut`, `reason`.

---

### STEP 03: Employee Self-Service (ESS) — Leave Application & Holiday Calendar
> **Goal:** Empower staff to view live leave entitlements, apply for leave with automatic validation, and view the company holiday calendar.

- [ ] **Task 3.1: Personal Leave Balances Display**
  - Create `frontend/src/app/(dashboard)/portal/leave/page.tsx`.
  - Display cards for each statutory leave type:
    - Annual Leave (*Entitled: 21 | Used: 5 | Remaining: 16*)
    - Sick Leave (*Entitled: 30 | Used: 2 | Remaining: 28*)
    - Maternity / Paternity Leave
    - Compassionate & Study Leave
- [ ] **Task 3.2: Self-Service Leave Application Modal**
  - Create `frontend/src/modules/portal/components/ApplyLeaveModal.tsx`:
    - Leave type dropdown (filters out exhausted categories).
    - Start date & End date pickers (auto-calculates number of days, skipping Sundays/public holidays).
    - Relief/Handover colleague selector (dropdown of department peers).
    - Reason text field.
    - Document attachment upload (required for Sick Leave > 2 days).
    - Submits via `POST /api/leave/apply`.
- [ ] **Task 3.3: Leave Request Status Tracker**
  - Display a history table of submitted applications with interactive status chips:
    - `SUBMITTED` ➔ `PENDING_MANAGER` ➔ `APPROVED` or `REJECTED`.
    - Ability to cancel a pending request before manager approval.
- [ ] **Task 3.4: Company Holiday Calendar**
  - Tab or side drawer showing official national and gazetted holidays for the operating company.

---

### STEP 04: Employee Self-Service (ESS) — Itemized Payslips & Loan Statements
> **Goal:** Give employees private, instant access to their monthly PDF payslips, tax certificates, and staff loan statements.

- [ ] **Task 4.1: My Payslips Screen**
  - Create `frontend/src/app/(dashboard)/portal/payslips/page.tsx`.
  - Fetch published payslips for the authenticated employee via `GET /api/payroll/payslips?employeeId={currentId}`.
- [ ] **Task 4.2: Printable Official PDF Payslip View**
  - Create `frontend/src/modules/portal/components/PayslipViewerModal.tsx`:
    - Hospital letterhead with company logo and KRA PIN.
    - Employee summary (Name, Number, Department, Grade, Bank Account Masked).
    - Two-column breakdown:
      - **Earnings**: Basic Salary, House Allowance, Medical Allowance, Overtime.
      - **Deductions**: PAYE, NSSF Tier I & II, SHIF, Housing Levy, Staff Loan EMI.
    - Net Pay highlighted in large bold text.
    - One-click **Print / Download PDF** button using browser print CSS.
- [ ] **Task 4.3: My Staff Loans & EMI Schedule**
  - Create `frontend/src/app/(dashboard)/portal/loans/page.tsx`:
    - Card showing active loan: Total Borrowed, Amount Repaid, Remaining Balance.
    - Monthly installment schedule table: Due Date, Principal, Interest, Status (*Paid / Upcoming*).
    - Button to **"Apply for Staff Loan / Advance"** opening an application form.

---

### STEP 05: Employee Self-Service (ESS) — Roster, Shift Swap & Documents
> **Goal:** Enable shift workers (doctors, nurses, technicians) to view upcoming rosters, swap shifts with peers, and access issued official letters.

- [ ] **Task 5.1: My Shift Roster View**
  - Create `frontend/src/app/(dashboard)/portal/roster/page.tsx`:
    - Weekly and monthly calendar showing assigned shifts (*e.g., Morning Shift 08:00–17:00, Night Duty 20:00–08:00, Weekly Off*).
- [ ] **Task 5.2: Shift Swap Request Workflow**
  - Add "Request Swap" action on any upcoming assigned shift:
    - Select target colleague (filtered to same cadre/grade).
    - Specify swap date.
    - Submits a request alerting the colleague and ward supervisor.
- [ ] **Task 5.3: My Letters & Official Documents**
  - Create `frontend/src/app/(dashboard)/portal/documents/page.tsx`:
    - List of official letters issued to the employee (*Appointment Letter, Confirmation, Increment*).
    - One-click view/download of authorized PDF letters.
    - Upload area for personal compliance certificates (Nursing License, KMPDC Renewal, CPD points).

---

### STEP 06: Manager Self-Service (MSS) — "My Team" Presence Dashboard
> **Goal:** Provide line managers and department heads with an instant operational overview of their unit's staffing.

- [ ] **Task 6.1: Manager Team Route & Data Hook**
  - Create `frontend/src/app/(dashboard)/manager/team/page.tsx`.
  - Backend API: Enhance `GET /api/employees?managerId={id}` or `GET /api/employees` to support `scope=TEAM` returning only direct and indirect reports.
- [ ] **Task 6.2: Real-Time Team Presence Widget**
  - Display live attendance counters:
    - 🟢 *Clocked In Today (Count)*
    - 🟡 *Late Arrival (Count)*
    - 🔵 *On Approved Leave (Count)*
    - 🔴 *Absent / Not Checked In (Count)*
- [ ] **Task 6.3: Team Roster & Coverage Calendar**
  - Visual weekly timeline showing shift coverage across all team members in the department.
- [ ] **Task 6.4: Team Members Contact Directory**
  - Card view of subordinates with emergency contact, current shift assignment, and employment status.

---

### STEP 07: Manager Self-Service (MSS) — Unified Approvals Action Center
> **Goal:** A consolidated, high-efficiency inbox where managers can review and decide on all subordinate requests in seconds.

- [ ] **Task 7.1: Unified Approvals Inbox Screen**
  - Create `frontend/src/app/(dashboard)/manager/approvals/page.tsx`.
  - Tabs:
    - 📑 **Leave Requests** (Badge showing pending count).
    - ⏰ **Attendance Regularisations** (Badge showing pending count).
    - 🔄 **Shift Swap Requests** (Badge showing pending count).
    - 💼 **Overtime Approvals** (Badge showing pending count).
- [ ] **Task 7.2: Leave Approval Card & Overlap Checker**
  - Display subordinate leave details: Employee name, leave type, date range, total days, relief staff assigned.
  - **Team Overlap Warning**: Alert if another member of the same department is already on approved leave for overlapping dates.
  - Action buttons: **Approve** (green) and **Reject** (red, prompts for mandatory comments).
  - Calls existing backend: `POST /api/approvals/decision`.
- [ ] **Task 7.3: Attendance Regularisation Approval**
  - View requested punch times vs. biometric logs, with employee's submitted justification.
  - Approve or reject with a single click.

---

### STEP 08: Manager Self-Service (MSS) — Staff Requisitions & Probation Reviews
> **Goal:** Allow department heads to initiate staffing requisitions and conduct timely probation confirmation reviews.

- [ ] **Task 8.1: Manager Staff Requisition Form**
  - Create `frontend/src/app/(dashboard)/manager/requisitions/page.tsx`:
    - Button: **"Request New Staff / Replacement"**.
    - Form: Designation, requested headcount, replacement vs. new post, justification, required qualifications.
    - Submits via `POST /api/requisitions`.
    - Tracks requisition through HR and Finance approval stages.
- [ ] **Task 8.2: Subordinate Probation Review Cockpit**
  - Automated alert card on Manager Dashboard when a subordinate's probation ends within 30 days.
  - One-click evaluation form:
    - Recommendation: *Confirm Employment*, *Extend Probation (1–3 Months)*, or *Initiate Separation*.
    - Performance rating and supervisor remarks.

---

### STEP 09: Finance & Treasury — Commercial Bank Disbursement Batch Generator
> **Goal:** Enable the Finance team to export compliant electronic funds transfer (EFT/KBA) batch payment files for direct bank upload.

- [ ] **Task 9.1: Backend Bank Batch Export Service**
  - Create `backend/Infrastructure/Services/BankExportService.cs`:
    - Formats finalized payroll cycle net salaries into standard commercial bank specifications:
      - **KBA (Kenya Bankers Association) Format** (Standard 120-character fixed-width EFT).
      - **KCB Bank / Equity Bank Bulk Upload CSV**.
      - **M-Pesa B2C Bulk Disbursement CSV** for casuals and locum staff.
  - Endpoint: `GET /api/payroll/cycles/{id}/export-bank?format={kba|csv|mpesa}`.
- [ ] **Task 9.2: Frontend Finance Disbursement Screen**
  - Create `frontend/src/app/(dashboard)/payroll/disbursements/page.tsx` (or tab inside `/payroll`):
    - Select finalized payroll cycle.
    - View total payout amount, employee count, and bank branch summary.
    - Download **Bank EFT Batch File** button.
    - Mark cycle as *Disbursed / Paid*.

---

### STEP 10: Finance & Treasury — Statutory Tax Return Schedules
> **Goal:** Provide ready-to-file statutory schedules matching East African revenue authority templates.

- [ ] **Task 10.1: KRA iTax P10 Monthly Schedule Export**
  - Backend endpoint: `GET /api/payroll/cycles/{id}/export-kra-p10`.
  - Generates CSV matching the exact Kenya Revenue Authority iTax payroll upload format (Employee PIN, Basic, Allowances, Gross, PAYE, Housing Levy, Insurance Relief).
- [ ] **Task 10.2: NSSF & SHIF Monthly Schedules**
  - Export NSSF return schedule: Member NSSF Number, Tier I Contribution, Tier II Contribution, Total Employer + Employee.
  - Export SHIF return schedule: National ID, SHIF Number, 2.75% Gross Salary Contribution.
- [ ] **Task 10.3: Cost Centre & General Ledger (GL) Export**
  - Summary report breaking down total employment cost, gross, and deductions grouped by Cost Centre and Hospital Department.

---

### STEP 11: Clinical Compliance — Medical License Watchdog & CPD Audit
> **Goal:** Safeguard hospital accreditation by proactively tracking clinical licenses and professional development points.

- [ ] **Task 11.1: Clinical License Expiry Dashboard**
  - Enhance `frontend/src/app/(dashboard)/compliance-masters/page.tsx`:
    - Tab: **License Expiry Watchdog**.
    - Color-coded register of doctors and nurses:
      - 🔴 *Expired*
      - 🟡 *Expiring in < 30 Days*
      - 🟢 *Valid / Current*
- [ ] **Task 11.2: Proactive Notification Triggers**
  - Backend scheduled task or query flagging credentials expiring within 30 days.
  - Automatically raises an in-app notification to the employee and clinical supervisor.
- [ ] **Task 11.3: CPD Points Audit Queue**
  - Screen for clinical auditor to review and approve CPD certificates uploaded by medical staff.

---

### STEP 12: Executive / C-Suite — Consolidated Group Workforce Intelligence
> **Goal:** Provide senior leadership (Group CEO, CFO, Board) with high-level workforce analytics and final sign-off authority.

- [ ] **Task 12.1: Executive Summary Dashboard**
  - Enhance `frontend/src/app/(dashboard)/group/page.tsx` for executive roles:
    - Group-wide headcount breakdown by subsidiary company.
    - Total monthly payroll expenditure vs. budget trendline.
    - Turnover rate and absenteeism index across hospital branches.
    - Clinical vs. Non-clinical staff ratio.
- [ ] **Task 12.2: Executive Authorization Queue**
  - High-level approval view for group-wide monthly payroll release and senior leadership appointments.

---

### STEP 13: End-to-End Persona Verification & Production Sign-Off
> **Goal:** Validate all 5 user perspectives with automated integration tests and persona logins.

- [ ] **Task 13.1: Persona Login Verification Test**
  - Verify seamless login and screen rendering across all 5 seed accounts:
    - `superadmin` (`role_super`) ➔ Unrestricted admin view.
    - `hr` (`role_hr`) ➔ Full HR management view.
    - `finance` (`role_fin`) ➔ Payroll, bank files, and statutory returns view.
    - `manager` (`role_mgr`) ➔ Team presence, roster, and approvals view.
    - `employee` (`role_emp`) ➔ Personal self-service portal view.
- [ ] **Task 13.2: Automated System Verification Suite Update**
  - Update `test_system_endpoints.py` to test self-service endpoints (`clock-in`, `apply-leave`, `payslips`, `team-attendance`).
  - Verify that 100% of endpoints pass without regression.
- [ ] **Task 13.3: Production Documentation Update**
  - Update `docs/test_report.md` with multi-persona verification results and sign-off.

---

## Technical File Mapping Matrix

| Persona | Frontend Routes | Primary Components | Backend Controllers |
|---|---|---|---|
| **Employee (ESS)** | `/portal`<br>`/portal/attendance`<br>`/portal/leave`<br>`/portal/payslips`<br>`/portal/loans`<br>`/portal/roster`<br>`/portal/documents` | `ClockInOutCard.tsx`<br>`ApplyLeaveModal.tsx`<br>`PayslipViewerModal.tsx`<br>`PersonalRosterCalendar.tsx`<br>`RegularisationModal.tsx` | `AttendanceController.cs`<br>`LeaveController.cs`<br>`PayrollController.cs`<br>`LoansController.cs`<br>`LettersController.cs` |
| **Manager (MSS)** | `/manager/team`<br>`/manager/approvals`<br>`/manager/requisitions`<br>`/manager/reviews` | `TeamPresenceWidget.tsx`<br>`ApprovalCard.tsx`<br>`OverlapWarningBadge.tsx`<br>`ProbationReviewModal.tsx` | `ApprovalsController.cs`<br>`EmployeesController.cs`<br>`AttendanceController.cs`<br>`RequisitionsController.cs` |
| **Finance** | `/payroll`<br>`/payroll/disbursements`<br>`/payroll/statutory-reports`<br>`/loan` | `BankDisbursementDrawer.tsx`<br>`KraP10ExportButton.tsx`<br>`LoanLedgerTable.tsx` | `PayrollController.cs`<br>`LoansController.cs`<br>`PayComponentsController.cs` |
| **Compliance** | `/compliance-masters`<br>`/documents` | `LicenseWatchdogTable.tsx`<br>`CpdAuditQueue.tsx`<br>`AssetAllocationModal.tsx` | `ComplianceMastersController.cs`<br>`DocumentsController.cs` |
| **Admin / HR** | `/employeedirectory`<br>`/orgmasters`<br>`/shifts`<br>`/recruitment`<br>`/letters`<br>`/groupsetup`<br>`/companysetup`<br>`/admin/*` | Full existing admin component suite | `CompaniesController.cs`<br>`GroupsController.cs`<br>`OrgMastersController.cs`<br>`AdminController.cs` |
