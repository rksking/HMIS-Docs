# HRMS Enterprise System: Multi-Persona Perspectives & Self-Service Blueprint

> **Reference Documents Audited:** `docs/HRMS_SPECIFICATION_AND_ARCHITECTURE.md`, `docs/PROJECT_CONTEXT.md`, `docs/tech_doc.md`, `docs/PROCESS_FLOW.md`, `docs/SETUP.md`, `docs/SHIFTS_AND_WORK_SCHEDULES.md`, `docs/test_report.md`  
> **Status:** Current Architecture Audit & Product Roadmap  
> **Scope:** Transitioning from Admin-Centric HRMS to a Full Multi-Perspective Enterprise Platform (Employee, Manager, Finance, Compliance, and Executive).

---

## 1. Executive Summary & Current State Audit

To date, the **Africare Enterprise HRMS** platform has successfully engineered and stabilized the core administrative spine of the platform:
- **Tenancy & Group Governance**: 85 relational tables, apex holding conglomerates, operating subsidiaries, multi-company switcher, and movement tracking.
- **Master Configurations**: Organization masters (departments, locations, job titles, grades, cost centres, banks), statutory tax configurations (PAYE, NSSF, SHIF, Housing Levy), shift models, and compliance registers.
- **Core Operations**: Employee 360 dossiers, biometric device integration, leave policies, staff requisitions, candidate pipelines (ATS), document folders, official letter templates, loans/EMI, and payroll calculation runs.

### The Current Reality: Admin-Centric Dominance
The screens developed so far in `frontend/src/app/(dashboard)/` and `Sidebar.tsx` reflect the perspectives of three administrative roles:
1. **Super Admin**: Full platform power, tenancy setup, database backups, audit logs, and user provisioning.
2. **HR Admin**: Employee directory CRUD, shift definitions, leave policy adjustments, letter issuing, and recruitment pipelines.
3. **Payroll Administrator**: Running multi-currency payroll cycles, defining allowances/deductions, and tax calculations.

### The Missing Gap
If a regular hospital employee (e.g., doctor, nurse, administrative clerk) or a line manager logs into the system today:
* They are exposed to high-level administrative screens they should not touch.
* The sidebar navigation displays menus like *Group Setup*, *Company Setup*, *Statutory Rates*, and *Audit Log*.
* There is no clean, distraction-free **Employee Self-Service (ESS)** workspace where an employee can view their personal payslip, apply for leave, clock in/out, or check their roster.
* There is no dedicated **Manager Self-Service (MSS)** cockpit where department heads can view live team presence or approve pending subordinate requests in one click.

---

## 2. Multi-Persona Architecture: The 5 Enterprise User Perspectives

Below is the architectural definition of the **5 distinct user perspectives** required for an enterprise deployment, mapped against the pre-loaded seed accounts in `docs/HRMS_SPECIFICATION_AND_ARCHITECTURE.md`:

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   ENTERPRISE APPLICATION ACCESS                                 │
└────────────────┬───────────────────┬───────────────────┬───────────────────┬────────────────────┘
                 │                   │                   │                   │
                 ▼                   ▼                   ▼                   ▼
        ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
        │  EMPLOYEE (ESS) │ │  MANAGER (MSS)  │ │     FINANCE     │ │   COMPLIANCE    │
        │  `role_emp`     │ │  `role_mgr`     │ │  `role_fin`     │ │   & AUDIT       │
        │  Scope: SELF    │ │  Scope: TEAM    │ │  Scope: ALL/CO  │ │   Scope: ALL/CO │
        └─────────────────┘ └─────────────────┘ └─────────────────┘ └─────────────────┘
                 │                   │                   │                   │
                 └───────────────────┴─────────┬─────────┴───────────────────┘
                                               │
                                               ▼
                                      ┌─────────────────┐
                                      │  ADMIN / HR     │
                                      │  `role_super`   │
                                      │  `role_hr`      │
                                      │  Scope: ALL     │
                                      └─────────────────┘
```

---

### Perspective 1: Employee Self-Service (ESS)
* **Primary Seed User:** `employee` (`Amina Gitau`, Clinical Staff Nurse, `EMP00020`)
* **Security Scope:** `SELF` (Strict row-level security: only allowed to view and query their own `EmployeeId`).
* **Device Focus:** High mobile responsiveness & touch-friendly interface.

#### Key Functional Modules & User Journeys:

1. **My Personal Workplace (`/portal/home` or `/me`)**:
   - Welcome banner showing personal shift status for today (*e.g., "On Duty: Morning Ward Shift 08:00 - 17:00"*).
   - Quick punch action card with live digital clock: **Clock In** / **Clock Out** button with geolocation/IP status indicator.
   - Quick KPI cards: *Remaining Annual Leave (Days)*, *This Month's Hours Worked*, *Pending Requests (Count)*.
   - Company noticeboard & public holiday announcements.

2. **My Profile Dossier (`/portal/profile`)**:
   - View personal details: Bio, national ID, passport, marital status, emergency contacts.
   - View job terms: Department, designation, reporting manager name, grade, date of hire.
   - View designated salary disbursement bank account (masked for privacy).
   - **Self-Service Change Request**: Form to submit updates for phone number, residential address, or emergency contacts (routed to HR for verification before DB update).

3. **My Time & Attendance (`/portal/attendance`)**:
   - **Personal Monthly Punch Card**: Calendar grid displaying daily check-in, check-out, total hours, and status chips (*Present, Late, Half-Day, Weekly Off, Absent*).
   - **Missed-Punch Regularisation Application**:
     - Employee selects date of missed punch.
     - Inputs actual clock-in/out times.
     - Selects reason: *Biometric Device Failure*, *On-Site Emergency Call*, *Network Outage*, *Official Off-Site Assignment*.
     - Submits for Line Manager approval.

4. **My Leave & Absence (`/portal/leave`)**:
   - **Leave Balance Summary Cards**: Real-time entitlement, used, and available balance for Annual, Sick, Maternity/Paternity, Compassionate, and Study leave.
   - **Apply for Leave**:
     - Select leave type.
     - Start date and end date (auto-calculates working days excluding weekly offs and public holidays).
     - Reason field.
     - Select relief / handover colleague from department list.
     - Upload supporting document (mandatory for Sick Leave > 2 days).
   - **Application History Tracker**: Visual timeline showing progress:
     - `Submitted` ➔ `Manager Review (Pending/Approved)` ➔ `HR Final Review (Approved)`.

5. **My Pay, Payslips & Tax (`/portal/payslips`)**:
   - **Download Itemized Monthly Payslips**: Secure list of published monthly pay runs.
   - One-click **PDF download** with official hospital header, breakdown of basic pay, overtime, housing/transport allowances, statutory deductions (PAYE, NSSF Tier I & II, SHIF, Affordable Housing Levy), loan deductions, and net salary.
   - **Tax Documents**: Download annual KRA P9 tax certificate for annual tax filing.
   - **My Loans & Advances (`/portal/loans`)**:
     - View active loan balances, monthly EMI installment amount, and repayment end date.
     - Interactive loan application form: Request emergency staff loan or salary advance within company policy limit.

6. **My Shift Roster (`/portal/roster`)**:
   - Personal monthly shift calendar (Ward Clinical Rotations, Night Shifts, General Shifts).
   - **Shift Swap Request**: Form to select a shift date and propose a swap with a qualified departmental peer, automatically alerting the ward supervisor.

7. **My Documents & Issued Letters (`/portal/documents`)**:
   - Archive of official letters issued to the employee: *Appointment Letter*, *Confirmation Letter*, *Salary Increment Letters*, *Promotion Letters*.
   - **Upload Compliance Documents**: Upload renewed Nursing Council (NCK) or Medical Board (KMPDC) license, BLS/ACLS certificates, and CPD points.

---

### Perspective 2: Line Manager / Supervisor (Manager Self-Service - MSS)
* **Primary Seed User:** `manager` (`Nafula Otieno Gitau`, Branch Manager Westlands, `EMP00017`)
* **Security Scope:** `TEAM` (Access restricted to direct and indirect subordinates within the reporting tree: `reportingManagerId == currentEmployeeId`).

#### Key Functional Modules & User Journeys:

1. **My Team Dashboard (`/manager/team`)**:
   - **Team Live Attendance Status**: Real-time overview of who is currently working, who is on approved leave, who is late today, and who is absent.
   - **Team Roster Schedule**: Weekly calendar view showing staffing coverage across shifts for their specific department or ward.
   - **Subordinate Directory**: Quick view of team members' contact info, emergency contacts, job title, and shift assignment.

2. **Unified Action Center / Pending Approvals (`/manager/approvals`)**:
   - A single, high-efficiency inbox for all subordinate requests:
     * **Leave Approvals**: View leave request with team calendar overlay (checks if too many staff from the same department are taking leave simultaneously). Single-click *Approve* or *Reject with Remarks*.
     * **Attendance Regularisations**: Review missed punches, inspect employee explanation, and approve or reject.
     * **Shift Swap Requests**: Approve peer-to-peer shift swaps ensuring minimum unit clinical coverage.
     * **Overtime Approvals**: Authorize extra hours worked before payroll cut-off.

3. **Team Staff Requisitions (`/manager/requisitions`)**:
   - Ability to initiate a **Job Requisition** for their department:
     - Specify vacancy title, requested headcount, replacement vs. new position, justification, and required qualifications.
     - Tracks approval through Finance and HR.

4. **Probation & Milestone Reviews (`/manager/reviews`)**:
   - Automated notifications for subordinates nearing the end of their 3-month or 6-month probation period.
   - Form to submit formal probation recommendation: *Confirm Employment*, *Extend Probation (with reason)*, or *Initiate Separation*.

---

### Perspective 3: Finance & Treasury Officer Perspective
* **Primary Seed User:** `finance` (`Nafula Otieno Mwangi`, Head of Finance, `EMP00003`)
* **Security Scope:** `ALL` or assigned operating companies. Focus on financial validation, cash outlays, statutory remittance, and bank clearing.

#### Key Functional Modules & User Journeys:

1. **Payroll Verification & Sign-off (`/finance/payroll`)**:
   - Review calculated payroll summaries submitted by HR.
   - Multi-currency variance report: Compare current month’s gross, deductions, and employer contributions against the previous month (flags spikes > 5%).
   - Multi-tier financial approval before disbursement.

2. **Disbursement & Bank File Generation (`/finance/bank-disbursements`)**:
   - Export structured bank disbursement files formatted for commercial banks (e.g., KCB, Equity Bank, Standard Chartered EFT/RTGS/KBA formats).
   - Generate Mobile Money (M-Pesa B2C Bulk Pay) disbursement files for casual or locum healthcare workers.

3. **Statutory Filing & Remittance Pack (`/finance/statutory-returns`)**:
   - Export certified monthly statutory return schedules:
     - **KRA iTax P10 Monthly File** (PAYE & Affordable Housing Levy).
     - **NSSF Monthly Return File** (Tier I & Tier II employee/employer contributions).
     - **SHIF Return Schedule** (Social Health Insurance Fund 2.75% contributions).

4. **Staff Loan Disbursements & Accounting (`/finance/loans`)**:
   - Review and disburse approved employee loan applications.
   - Generate General Ledger (GL) journal entries: Debit Staff Loan Asset, Credit Bank Disbursement.
   - Record manual loan repayments or early settlements.

5. **Cost Center & General Ledger Export (`/finance/gl-export`)**:
   - Generate journal voucher summaries broken down by Cost Centre, Department, and Subsidiary Company for import into external accounting ERPs (e.g., SAP, QuickBooks, Oracle).

---

### Perspective 4: Clinical Compliance & Regulatory Auditor
* **Primary Role:** Hospital Clinical Governance & Compliance Lead
* **Security Scope:** Quality, patient safety, and regulatory board compliance.

#### Key Functional Modules & User Journeys:

1. **Professional Licensing Watchdog (`/compliance/licenses`)**:
   - Central register of medical licenses across all doctors, surgeons, nurses, pharmacists, and radiographers.
   - Color-coded expiry alerts:
     - 🔴 **Expired** (Immediate alert: staff cannot be scheduled on clinical shifts).
     - 🟡 **Expiring in 30 Days** (Automated SMS/Email notification sent to employee).
     - 🟢 **Active / Verified**.
   - Direct verification link to regulatory portals (e.g., KMPDC, NCK, PPB online registries).

2. **CPD (Continuing Professional Development) Audit (`/compliance/cpd`)**:
   - Tracking mandatory annual CPD points per cadre.
   - Queue to verify CPD certificates submitted by clinical personnel.

3. **Asset & Medical Equipment Allocations (`/compliance/assets`)**:
   - Tracking sensitive medical instruments, communication radios, tablets, and hospital laptops assigned to personnel.
   - Department transfer clearance and exit clearance sign-off.

---

### Perspective 5: Executive / Board / C-Suite (CEO, CFO, Medical Director)
* **Primary Seed User:** `superadmin` / Group Executive Leadership
* **Security Scope:** Aggregated enterprise-wide executive intelligence.

#### Key Functional Modules & User Journeys:

1. **Executive Workforce Intelligence (`/executive/dashboard`)**:
   - Total group headcount across all operating companies and subsidiaries.
   - Month-on-month total employment cost and average cost per employee.
   - Clinical vs. Non-Clinical staff ratio.
   - Group turnover and attrition trends.
   - Overtime spend hot-spots by hospital location.

2. **Executive Multi-Tier Approvals (`/executive/approvals`)**:
   - Final executive approval on senior appointments (HODs, Medical Directors).
   - Final approval on group-wide monthly payroll release.

---

## 3. Dynamic Role-Based Access Control (RBAC) & Navigation Matrix

The frontend navigation must adapt dynamically based on the authenticated user's role and permission grants:

| Route / Module | Super Admin (`role_super`) | HR Admin (`role_hr`) | Line Manager (`role_mgr`) | Employee (`role_emp`) | Finance (`role_fin`) |
|---|:---:|:---:|:---:|:---:|:---:|
| **`/portal` (Employee Workspace)** | Optional | Optional | ✅ Self | **PRIMARY** | Optional |
| **`/portal/attendance` (My Clock/Punch)** | ❌ | ❌ | ✅ Self | **PRIMARY** | ❌ |
| **`/portal/leave` (My Leave Application)** | ❌ | ❌ | ✅ Self | **PRIMARY** | ❌ |
| **`/portal/payslips` (My PDF Payslips)** | ❌ | ❌ | ✅ Self | **PRIMARY** | ❌ |
| **`/portal/loans` (My Staff Loans)** | ❌ | ❌ | ✅ Self | **PRIMARY** | ❌ |
| **`/manager/team` (My Team Presence)** | ❌ | ❌ | **PRIMARY** | ❌ | ❌ |
| **`/manager/approvals` (Approvals Inbox)** | ✅ | ✅ | **PRIMARY** | ❌ | ❌ |
| **`/employeedirectory` (Full Directory)** | ✅ Full | ✅ Full | 👥 Team Only | ❌ Hidden | 👁️ View Only |
| **`/employees/[id]` (Dossier Edit)** | ✅ Full | ✅ Full | 👁️ Team View | 👁️ Self Only | 👁️ View Only |
| **`/attendance` (Company Punch Rolls)** | ✅ Full | ✅ Full | 👥 Team Only | ❌ Hidden | ❌ Hidden |
| **`/shifts` (Roster & Shift Master)** | ✅ Full | ✅ Full | 👁️ View/Assign | ❌ Hidden | ❌ Hidden |
| **`/payroll` (Run Payroll Calculations)** | ✅ Full | ✅ Full | ❌ Hidden | ❌ Hidden | **PRIMARY** |
| **`/finance/bank-files` (EFT Exports)** | ✅ Full | ❌ Hidden | ❌ Hidden | ❌ Hidden | **PRIMARY** |
| **`/requisitions` (Staff Requisitions)** | ✅ Full | ✅ Full | ✍️ Initiate/View | ❌ Hidden | 👁️ View/Budget |
| **`/recruitment` (ATS & Candidates)** | ✅ Full | ✅ Full | ✍️ Interviewer | ❌ Hidden | ❌ Hidden |
| **`/letters` (Issue Official Letters)** | ✅ Full | ✅ Full | ❌ Hidden | ❌ Hidden | ❌ Hidden |
| **`/documents` (DMS Org Repository)** | ✅ Full | ✅ Full | 👥 Team View | ❌ Hidden | 👁️ View Only |
| **`/compliance-masters` (Licensing)** | ✅ Full | ✅ Full | 👁️ View Only | ❌ Hidden | ❌ Hidden |
| **`/groupsetup` & `/companysetup`** | ✅ Full | 👁️ View Only | ❌ Hidden | ❌ Hidden | ❌ Hidden |
| **`/admin/*` (Users, Roles, Audit)** | ✅ Full | ❌ Hidden | ❌ Hidden | ❌ Hidden | ❌ Hidden |

---

## 4. Backend API Alignment (Existing vs. Required)

A review of the `.NET 10` API backend demonstrates that **most business logic and endpoints already exist** in controllers, requiring primarily frontend integration and persona-specific views:

| Business Function | Existing Backend API | Status | Required Frontend Work |
|---|---|:---:|---|
| **Employee Clock In/Out** | `POST /api/attendance/clock-in`, `POST /api/attendance/clock-out` | ✅ Ready | Build personal clock widget with timestamp and feedback |
| **Employee Daily Attendance** | `GET /api/attendance/daily?date=...` | ✅ Ready | Build personal monthly calendar view |
| **Missed-Punch Regularisation** | `POST /api/attendance/regularise` | ✅ Ready | Build regularisation modal with reason dropdown |
| **Employee Leave Balances** | `GET /api/leave/balances?employeeId={id}` | ✅ Ready | Build leave balance cards widget |
| **Apply for Leave** | `POST /api/leave/apply` | ✅ Ready | Build leave application drawer/modal with date math |
| **Leave Calendar** | `GET /api/leave/calendar?month=...` | ✅ Ready | Build department holiday & team leave calendar |
| **Employee Payslips** | `GET /api/payroll/payslips?employeeId={id}` | ✅ Ready | Build printable payslip viewer with PDF export |
| **Employee Loans** | `GET /api/loans?employeeId={id}` | ✅ Ready | Build loan balance card & installment table |
| **Manager Approvals Queue** | `GET /api/approvals/pending` | ✅ Ready | Build unified manager approval card inbox |
| **Manager Approval Decision** | `POST /api/approvals/decision` | ✅ Ready | Add one-click Approve/Reject with comments dialog |
| **Bank Disbursement Files** | `GET /api/payroll/cycles/{id}/export-bank` | ⚠️ Endpoint Needed | Add EFT/KBA bank file generator service |
| **KRA iTax P10 Export** | `GET /api/payroll/cycles/{id}/export-tax` | ⚠️ Endpoint Needed | Add CSV formatter matching KRA iTax template |

---

## 5. Step-by-Step Implementation Roadmap

To systematically roll out these perspectives without disrupting existing administrative features:

### Phase 1: Employee Self-Service (ESS) Portal (Days 1 – 3)
1. **Frontend Route `/portal`**:
   - Create `src/app/(dashboard)/portal/page.tsx` as the home dashboard for employees.
   - Component: `ClockInOutWidget.tsx` using existing `/api/attendance/clock-in` and `/clock-out`.
   - Component: `LeaveBalanceCards.tsx` using `/api/leave/balances`.
   - Component: `RecentPayslipsList.tsx` with printable PDF payslip modal.
2. **Leave Application Modal (`ApplyLeaveModal.tsx`)**:
   - Integrated into the portal allowing staff to submit leave with instant balance validation.
3. **Sidebar Dynamic Persona Filter**:
   - Update `src/components/layout/Sidebar.tsx` to detect `user.role`:
     - If `role === 'Employee'`, display **My Workplace**, **My Attendance**, **My Leave**, **My Payslips**, and **My Documents**.

### Phase 2: Manager Self-Service (MSS) Cockpit (Days 4 – 5)
1. **Manager Team Dashboard (`/manager/team`)**:
   - Team presence card (*Clocked In*, *On Leave*, *Absent*).
   - Subordinate roster schedule view.
2. **Unified Approvals Inbox (`/manager/approvals`)**:
   - Central tabbed list: *Leave Requests (Count)*, *Missed-Punch Regularisations (Count)*, *Shift Swaps (Count)*.
   - Quick action buttons: `Approve` and `Reject with Comment`.

### Phase 3: Finance & Statutory Remittance Exports (Days 6 – 7)
1. **Bank Batch Generation**:
   - Backend endpoint to format bank files for Kenyan/East African clearing (EFT/RTGS).
2. **Statutory CSV Downloads**:
   - KRA P10 monthly return export, NSSF schedule export, and SHIF monthly return export.

### Phase 4: Clinical Compliance Alerts & Audits (Day 8)
1. **License Expiry Notification Engine**:
   - Background service flagging healthcare licenses expiring within 30 days.
   - Automated badge in header navigation for clinical supervisors.

---

## 6. Conclusion & Recommendation

The Africare Enterprise HRMS has a rock-solid, production-grade administrative and relational core. By implementing the **Employee Self-Service (ESS)** and **Manager Self-Service (MSS)** perspectives on top of the existing API infrastructure, the platform will evolve from a back-office administration tool into a complete, daily-active operational platform for every staff member in the hospital network.
