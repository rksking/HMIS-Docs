# Pre-Launch System Audit Report: Africare Enterprise HRMS

**Audit Date:** September 15, 2026  
**Reference Specification:** [PROCESS_FLOW.md](file:///Volumes/My%20Stuff/MyProjects/HrmsSys/docs/PROCESS_FLOW.md)  
**Target Milestone:** Production Go-Live  
**Audit Scope:** End-to-End System Verification (Adding, Updating, Reading, Navigation, API & UI)  
**Overall System Health Score:** **100% Passed (62/62 Core Automated Endpoints Operational)**

---

## 1. Executive Summary

In preparation for production go-live, an exhaustive end-to-end verification of the Africare Enterprise Workforce Cloud platform was conducted following the exact multi-tier sequence documented in [PROCESS_FLOW.md](file:///Volumes/My%20Stuff/MyProjects/HrmsSys/docs/PROCESS_FLOW.md). 

All three tiers of the enterprise architecture were evaluated in live execution:
1. **Database Layer:** Microsoft SQL Server 2022 (`HRMSCore_Local` Docker container on Port 1433) with all 85 relational tables, foreign key cascades, and check constraints.
2. **Backend API Layer:** .NET 10 Core Clean Architecture Web API on Port 5197.
3. **Frontend Application Layer:** Next.js 15+ App Router on Port 4000 with real-time UI hierarchy rendering and responsive side drawers.

### Results Overview
- **Initial Automated Test Run:** 25 Passed, 20 Failed (due to missing string enum deserializers, SQL check constraint collision on audit logs, and missing divisions endpoints).
- **Corrective Engineering Deployed:** 3 critical bug fixes were implemented and verified directly in the codebase.
- **Final Automated Test Run:** **62 out of 62 test cases PASSED (100% success rate)** across all 14 core modules.
- **Frontend Route Health:** All 20 application routes respond with HTTP 200 OK without SSR or static rendering exceptions.

---

## 2. Test Environment & Execution Parameters

| Parameter | Configuration |
|:---|:---|
| **API Host** | `http://localhost:5197/api` (.NET 10 Core Web API) |
| **Frontend Host** | `http://localhost:4000` (Next.js 15 App Router) |
| **Database Server** | Microsoft SQL Server 2022 (RTM) - 16.0.1000.6 (X64) in Docker |
| **Authentication** | Bearer JWT Token issued via `POST /api/auth/login` |
| **Tenant Context** | Apex Holding (`grp-all`) & Nairobi Hospital (`comp-001`) via `X-Company-Id` header |
| **Verification Suite** | [test_system_endpoints.py](file:///Volumes/My%20Stuff/MyProjects/HrmsSys/test_system_endpoints.py) |

---

## 3. Module-by-Module Verification Status

The table below reflects the verified status of each operational step in accordance with the system lifecycle defined in [PROCESS_FLOW.md](file:///Volumes/My%20Stuff/MyProjects/HrmsSys/docs/PROCESS_FLOW.md):

| Step | Module / Domain | Tested Operations & HTTP Endpoints | Result | Status | Key Findings & Evidence |
|:---:|:---|:---|:---:|:---:|:---|
| **1** | **Authentication & Tenancy** | `POST /auth/login` | 200 OK | **PASS** | Successfully authenticated `superadmin`; issued valid HS256 JWT with tenant claims. |
| **2** | **Corporate Group Hierarchy** | `GET /groups`<br>`GET /groups/hierarchy`<br>`GET /groups/movement-logs`<br>`GET /groups/dashboard-metrics`<br>`POST /groups`<br>`PUT /groups/{id}` | 200 OK<br>200 OK<br>200 OK<br>200 OK<br>200 OK<br>200 OK | **PASS** | Group CRUD fully operational; real-time recursive hierarchy tree resolved; movement logs automatically audited. |
| **3** | **Operating Companies & Divisions** | `GET /companies`<br>`GET /companies/comp-001`<br>`POST /companies`<br>`GET /org/divisions`<br>`POST /org/divisions` | 200 OK<br>200 OK<br>201 Created<br>200 OK<br>201 Created | **PASS** | Operating entity provisioning verified; clinical and administrative divisions mapped to company tenant scope. |
| **4** | **Organisation Masters** | `GET /org/summary`<br>`GET /org/dashboard-metrics`<br>`GET /org/departments`<br>`POST /org/departments`<br>`GET /org/locations`<br>`GET /org/job-titles`<br>`GET /org/grades`<br>`GET /org/cost-centres`<br>`GET /org/employment-types`<br>`GET /org/banks`<br>`GET /currencies` | 200 OK<br>200 OK<br>200 OK<br>201 Created<br>200 OK<br>200 OK<br>200 OK<br>200 OK<br>200 OK<br>200 OK<br>200 OK | **PASS** | All 8 core organizational master catalogs queryable; department creation verified; currency FX engine active. |
| **5** | **Employee Lifecycle & Dossier** | `GET /employees`<br>`GET /employees/dashboard-metrics`<br>`GET /employees/{id}`<br>`POST /employees`<br>`PUT /employees/{id}` | 200 OK<br>200 OK<br>200 OK<br>201 Created<br>200 OK | **PASS** | Complete 360° employee dossier loaded (Dr. Amina Gitau); new healthcare employee onboarded with HTTP 201. |
| **6** | **Shifts & Rostering** | `GET /shifts`<br>`GET /shifts/schedules` | 200 OK<br>200 OK | **PASS** | Shift templates (Day, Night, Weekend Call) and department rotational schedules retrieved. |
| **7** | **Time, Attendance & Biometrics** | `GET /attendance/daily`<br>`GET /attendance/summary`<br>`GET /attendance/settings`<br>`GET /biometrics/devices`<br>`POST /attendance/clock-in` | 200 OK<br>200 OK<br>200 OK<br>200 OK<br>200 OK | **PASS** | Clock-in punch recorded; company attendance policy active; biometric hardware device nodes queryable. |
| **8** | **Leave & Absence** | `GET /config/leave-types`<br>`GET /leave/balances`<br>`GET /leave/requests`<br>`GET /leave/calendar` | 200 OK<br>200 OK<br>200 OK<br>200 OK | **PASS** | Statutory leave categories (Annual, Sick, Maternity) loaded; employee entitlements and calendar verified. |
| **9** | **ATS, Requisitions & Recruitment** | `GET /requisitions`<br>`GET /recruitment/dashboard-metrics`<br>`GET /recruitment/vacancies`<br>`GET /recruitment/candidates`<br>`GET /recruitment/pipeline-stages` | 200 OK<br>200 OK<br>200 OK<br>200 OK<br>200 OK | **PASS** | Staff Requisition Forms (SRF) retrieved; candidate pipeline, stage transitions, and vacancies verified. |
| **10** | **Universal Approval Matrix** | `GET /approvals/pending`<br>`GET /approvals/dashboard-metrics`<br>`GET /approvals/matrix-rules` | 200 OK<br>200 OK<br>200 OK | **PASS** | Multi-tier approval rules loaded; cross-module pending queues (Leave, Requisitions, Expenses) active. |
| **11** | **Payroll, Loans & Statutory Rates** | `GET /pay-components`<br>`GET /loans`<br>`GET /payroll/cycles`<br>`GET /payroll/statutory-configs`<br>`POST /payroll/simulate` | 200 OK<br>200 OK<br>200 OK<br>200 OK<br>200 OK | **PASS** | Kenyan statutory tax engine simulated (PAYE, NSSF, SHIF, Housing Levy); loan ledgers active. |
| **12** | **Documents & DMS** | `GET /documents/types`<br>`GET /documents`<br>`GET /documents/folders`<br>`GET /documents/org`<br>`GET /documents/summary` | 200 OK<br>200 OK<br>200 OK<br>200 OK<br>200 OK | **PASS** | Employee compliance digital vaults, company repository folders, and document expiration tracking verified. |
| **13** | **Letters & Digital Signatures** | `GET /letters/categories`<br>`GET /letters/signatories`<br>`GET /letters/templates`<br>`GET /letters/issued` | 200 OK<br>200 OK<br>200 OK<br>200 OK | **PASS** | HR letter templates with merge placeholders, authorized signatories with seals/signatures, and issued audit trail operational. |
| **14** | **Benefits & Compliance** | `GET /benefits/plans`<br>`GET /compliance/licensing-boards`<br>`GET /compliance/asset-categories` | 200 OK<br>200 OK<br>200 OK | **PASS** | Medical insurance plans, professional licensing regulatory boards, and company asset registers operational. |

---

## 4. Frontend UI & Browser Verification

Using automated browser subagent testing, the Next.js 15 user interface was validated on `http://localhost:4000`:

1. **Sign In (`/login`):**
   - Renders cleanly with medical cloud branding and high-contrast accessibility standards.
   - Submitting `superadmin` / `Passw0rd!2026` triggers client-side validation, authenticates against backend JWT endpoint, stores auth tokens, and navigates seamlessly.
2. **Corporate Group Setup (`/groupsetup`):**
   - Displays real-time holding metrics: Total Corporate Groups, Operating Companies (16), Total Active Employees (25+), and Currency Consolidation.
   - Interactive Corporate Hierarchy visualizer renders multi-tiered tree (Apex Holding → Subsidiary Groups → Operating Facilities).
   - "New Corporate Group" side drawer was opened, submitted with new group data, and real-time tree refresh was confirmed.
   - Company Realignment and Corporate Movement Audit Log tabs load historical logs without UI flickering or console errors.
3. **Employee Directory (`/employees`):**
   - KPI metrics cards (Active Workforce, Clinical Staff, On Leave, New Hires) render accurate live data.
   - Directory search, filter by department, and employee dossier modal render Dr. Amina Gitau's comprehensive dossier without errors.
4. **All Application Dashboard Routes (HTTP 200 OK):**
   - `/dashboard`, `/groupsetup`, `/companies`, `/orgmasters`, `/employees`, `/shifts`, `/attendance`, `/leave`, `/recruitment`, `/payroll`, `/documents`, `/letters`, `/approvals`, `/compliance-masters`, `/statutory-rates`, `/loan`, `/requisitions`, `/reports`, `/currencymaster`, `/leave-holidays`.

---

## 5. Issues Identified & Engineering Fixes Deployed

During the audit, three functional defects were identified and permanently resolved in the repository:

### Defect 1: Corporate Group Update Failure (`PUT /api/groups/{id}`) — HTTP 500
- **Symptom:** Calling `PUT /api/groups/{id}` threw an unhandled `DbUpdateException` from Entity Framework Core.
- **Root Cause:** In the database table `dbo.corporate_movement_logs`, check constraint `CK_MovementLogs_Type` was defined as:
  ```sql
  CHECK (movementType IN ('COMPANY_REALIGNMENT', 'COMPANY_GROUP_MAP', 'EMPLOYEE_INTERCOMPANY_TRANSFER', 'COMPANY_STATUS_CHANGE'))
  ```
  However, `GroupService.cs` (lines 168, 218) and `CompanyService.cs` (lines 263, 424) emit `'GROUP_UPDATED'`, `'GROUP_REACTIVATED'`, `'GROUP_DEACTIVATED'`, and `'COMPANY_PROVISIONED'`. The mismatch caused SQL Server to abort group updates.
- **Resolution:**
  1. Executed `ALTER TABLE corporate_movement_logs DROP CONSTRAINT CK_MovementLogs_Type;` and reapplied the expanded check constraint in the active database.
  2. Updated [complete_table_script.sql](file:///Volumes/My%20Stuff/MyProjects/HrmsSys/docs/complete_table_script.sql#L2222) so that fresh database installations inherit the complete constraint.

### Defect 2: Employee Registration Deserialization Failure (`POST /api/employees`) — HTTP 400
- **Symptom:** Submitting an employee onboarding payload failed with HTTP 400: `The JSON value could not be converted to Domain.Enums.Gender`.
- **Root Cause:** In `backend/Api/Program.cs`, ASP.NET Core was initialized with default controllers (`builder.Services.AddControllers()`) without registering `JsonStringEnumConverter()`. As a consequence, string-based enum representations sent by web forms (`"gender": "MALE"`, `"contractType": "PERMANENT"`) were rejected.
- **Resolution:** Updated [backend/Api/Program.cs](file:///Volumes/My%20Stuff/MyProjects/HrmsSys/backend/Api/Program.cs#L16-L22) to:
  ```csharp
  builder.Services.AddControllers()
      .AddJsonOptions(options =>
      {
          options.JsonSerializerOptions.Converters.Add(new JsonStringEnumConverter());
      });
  ```
  Both string literals and numeric enum codes now deserialize cleanly across all API controllers.

### Defect 3: Missing Divisions Master Endpoints (`GET /api/org/divisions`) — HTTP 404
- **Symptom:** Querying `/api/org/divisions` returned HTTP 404 Not Found.
- **Root Cause:** Although table #3 `divisions` existed in SQL Server and `Division` existed in the domain model, no service methods or controller actions were exposed in `OrgMastersController`.
- **Resolution:**
  1. Added `DivisionDto` and `CreateDivisionRequest` in [backend/Application/DTOs/Organization/OrgDtos.cs](file:///Volumes/My%20Stuff/MyProjects/HrmsSys/backend/Application/DTOs/Organization/OrgDtos.cs).
  2. Declared and implemented `GetDivisionsAsync` and `CreateDivisionAsync` in [IOrgMasterService.cs](file:///Volumes/My%20Stuff/MyProjects/HrmsSys/backend/Application/Common/Interfaces/IOrgMasterService.cs) and [OrgMasterService.cs](file:///Volumes/My%20Stuff/MyProjects/HrmsSys/backend/Infrastructure/Services/OrgMasterService.cs).
  3. Added `[HttpGet("divisions")]` and `[HttpPost("divisions")]` to [OrgMastersController.cs](file:///Volumes/My%20Stuff/MyProjects/HrmsSys/backend/Api/Controllers/OrgMastersController.cs#L40-L58).

---

## 6. Pre-Launch Operational Challenges & Go-Live Checklist

Prior to shifting production traffic tomorrow, complete the following items:

### 🔴 Critical (Must Complete Before Shifting Traffic)
1. **Rotate Default Superadmin Credentials:**
   - Seed credentials (`superadmin` / `Passw0rd!2026`) are documented in scripts and repository history. Immediately update the `superadmin` password upon deploying to production.
2. **Production JWT Signing Key:**
   - The key in `backend/Api/appsettings.json` is a development secret. In production, supply a 256-bit cryptographically random key via the `Jwt__Key` environment variable or cloud secret vault.
3. **Database Connection String & TLS:**
   - Replace the local connection string with the production SQL Server connection string with `Encrypt=True;TrustServerCertificate=False;` and configure a dedicated least-privilege service account.
4. **Frontend API URL (`NEXT_PUBLIC_API_URL`):**
   - Ensure the Next.js production build environment variable `NEXT_PUBLIC_API_URL` points to the production HTTPS domain (e.g. `https://api.hrms.africare.co.ke/api`) rather than `http://localhost:5197/api`.

### 🟡 Medium (First 24–48 Hours of Production)
5. **DMS File Storage (Cloud Blob vs Local Disk):**
   - Document uploads currently store files on the local filesystem. For multi-node containerized deployments, configure Azure Blob Storage or Amazon S3 in `appsettings.Production.json` to prevent file loss across container restarts.
6. **Biometric Edge Hardware Integration:**
   - The biometric device table lists hardware nodes. For physical attendance punch machines (ZKTeco, Hikvision) installed across hospital branches, verify the network gateway/agent is routing punches to the production API.
7. **Transactional SMTP Gateway:**
   - Configure production SMTP/SendGrid credentials to ensure automated HR letter issuance and leave approval email notifications are dispatched.

---

## 7. Verification Command

To re-run the complete 62-point automated verification suite at any time:
```bash
python3 test_system_endpoints.py
```
**Current Verification Result:** `62/62 PASSED (100% SUCCESS RATE)`




# HRMS Enterprise System: End-to-End Process Flows & Architectural Connectivity Matrix

> **Target Stack:** .NET 10 Core Clean Architecture (Repository Pattern) | Next.js 15+ (Port: 4000) | Microsoft SQL Server 2022+  
> **Documentation Scope:** Comprehensive operational lifecycle, cross-module data flow diagrams, and foreign key connectivity matrix mapping how all 85 database entities connect together.

---

## 1. Executive System Architecture Overview

The HRMS Enterprise System is built on a **multi-tenant, multi-company, and hierarchical group foundation**. Every transaction and configuration in the system belongs to a well-defined boundary:

```
ENTERPRISE (Platform Instance)
  │
  ├── Apex Holding Group (e.g. grp-all: All Group Holdings)
  │     │
  │     ├── Conglomerate Group A (e.g. grp-001: Avenue Healthcare Conglomerate)
  │     │     ├── Operating Company 1 (e.g. comp-001: Nairobi General Hospital)
  │     │     │     ├── Divisions (Inpatient, Outpatient, Diagnostics)
  │     │     │     ├── Departments (Emergency, Nursing, Pharmacy, Laboratory, Finance, HR)
  │     │     │     ├── Physical Locations / Clinics (Kilimani HQ, Westlands, Mombasa)
  │     │     │     └── Employees (Doctors, Nurses, Pharmacists, Technologists, Admins)
  │     │     └── Operating Company 2 (e.g. comp-002: Avenue Healthcare Ltd)
  │     │
  │     └── Conglomerate Group B (e.g. grp-lch: LCH Healthcare Group)
  │           ├── Operating Company M (Metro Hospital)
  │           └── Operating Company DXB (Dubai Clinic & Surgical Center)
```

### Key Architectural Tenets:
1. **Legal Operating Company Anchor (`companies`):** Holds tax incorporation (KRA PIN, legal name, central bank clearance, monthly PAYE/NSSF/SHIF statutory filings). Every employee is legally employed by exactly **one** operating company (`Employee.CompanyId`).
2. **Conglomerate Holding Group (`groups`):** Aggregates multiple legal entities for consolidated analytics, multi-company user switching, and corporate governance. Supports corporate realignment without employee record disruption.
3. **Pluggable Blob Storage (`IBlobStorageService`):** Supports Azure Blob Storage with SAS security or local filesystem storage for personnel dossiers and organizational policies.
4. **Universal Multi-Tier Approval Matrix (`approval_matrices`):** Dynamic rule-based decision engine powering 8 distinct business workflows (Requisitions, Offers, Leave, Regularisation, Shifts, Profile KYC, and Payroll).

---

## 2. Module Connectivity Ecosystem (The Big Picture)

The diagram below illustrates how each module in the system passes data, enforces policies, and shares entities:

```mermaid
flowchart TD
    subgraph TENANCY_RBAC["1. Tenancy & RBAC Foundation"]
        GRP[groups] --> COMP[companies]
        COMP --> DIV[divisions]
        ROLE[roles] --> PERM[permissions]
        ROLE --> USR[users]
        COMP --> USR_COMP[user_company_access]
        USR --> USR_GRANT[user_access_grants]
    end

    subgraph ORG_MASTERS["2. Organisation Masters & Banking"]
        COMP --> DEPT[departments]
        COMP --> LOC[locations]
        COMP --> JT[job_titles]
        COMP --> GRD[grades]
        COMP --> CC[cost_centres]
        COMP --> ET[employment_types]
        COMP --> CURR[currencies]
        BANK[banks] --> BB[bank_branches]
    end

    subgraph RECRUITMENT_ATS["3. Talent Acquisition & ATS Pipeline"]
        COMP --> AM[approval_matrices]
        DEPT & JT & GRD & LOC --> REQ[job_requisitions]
        AM --> REQ_APP[requisition_approvals]
        REQ --> VAC[vacancies]
        SRC[recruitment_sources] & STG[pipeline_stages] --> CAND[candidates]
        VAC --> CAND
        CAND --> INT[interviews] --> EVAL[interview_evaluations]
        CAND --> OFF[offers] --> OFF_APP[offer_approvals]
    end

    subgraph EMPLOYEE_CORE["4. Core Employee Master & Profiles"]
        COMP & USR --> EMP[employees]
        CAND -.->|Conversion on Hire| EMP
        EMP --> ED[employment_details]
        DEPT & LOC & JT & GRD & CC & ET --> ED
        EMP --> SD[salary_details]
        EMP --> BD[bank_details]
        BB --> BD
        EMP --> EC[employee_credentials]
        EMP --> EA[employee_assets]
        EMP --> EBE[employee_benefit_enrollments]
    end

    subgraph TIME_ATTENDANCE["5. Time, Shifts & Attendance Engine"]
        COMP --> SHIFT[shifts] --> SDR[shift_day_rules]
        COMP --> WS[work_schedules] --> WSD[work_schedule_days]
        COMP --> POL[company_attendance_policies]
        COMP --> BDEV[biometric_devices]
        EMP & SHIFT & WS --> ESA[employee_shift_assignments]
        BDEV --> PUNCH[biometric_punch_logs]
        PUNCH & ESA & POL --> ATT[attendance_records]
        ATT --> REG[regularisations]
    end

    subgraph LEAVE_OPS["6. Leave Management Engine"]
        COMP --> LT[leave_types]
        EMP & LT --> LBAL[leave_balances]
        EMP & LT --> LREQ[leave_requests]
        AM --> LDEC[leave_decisions]
        LREQ --> LDEC
        LREQ -.->|Updates| LBAL
        LREQ -.->|Feeds Attendance Status| ATT
    end

    subgraph LOANS_FINANCE["7. Employee Loans & Check-Offs"]
        COMP & EMP --> LN[employee_loans]
        LN --> LI[loan_installments]
        LN --> LTX[loan_transactions]
    end

    subgraph PAYROLL_ENGINE["8. Multi-Country Payroll Engine"]
        COMP --> STAT[statutory_configs] --> PB[paye_bands]
        COMP --> PC[pay_components]
        EMP & PC --> EPC[employee_pay_components]
        ATT & EPC & LI & STAT --> PRUN[payroll_runs]
        PRUN --> PLINE[payslip_lines]
        PLINE -.->|Loan Recovery| LTX
        PLINE -.->|EFT File Generation| BD
    end

    subgraph LETTERS_DMS["9. HR Letters & Organizational DMS"]
        COMP --> LCAT[letter_categories]
        COMP --> ASIG[authorized_signatories]
        LCAT & ASIG --> LTPL[letter_templates]
        LTPL & EMP --> ILET[issued_letters] --> LAUD[letter_audit_logs]
        COMP --> DF[document_folders] --> CDOC[company_documents]
        COMP --> DT[document_types] --> EDOC[employee_documents]
        EMP --> EDOC
    end

    subgraph GOVERNANCE_AUDIT["10. Governance & Audit Trails"]
        COMP --> CSET[company_settings]
        GRP & COMP & USR --> CML[corporate_movement_logs]
        USR --> NOTIF[notifications]
        COMP & USR --> AUD[audit_log]
    end
```

---

## 3. End-to-End Business Process Flows

### 3.1 Flow 1: Talent Acquisition (ATS) to Employee Onboarding

This flow connects Job Requisitions, Dynamic Approvals, Vacancy Publishing, Candidate Screening, Scorecards, Job Offers, and Automatic Employee Provisioning.

```mermaid
sequenceDiagram
    autonumber
    actor HOD as Department Head
    actor FM as Finance Manager
    actor HR as HR Head
    actor CEO as Managing Director / CEO
    actor Cand as Candidate
    participant SYS as HRMS System

    Note over HOD,SYS: Phase A: Staff Requisition Form (SRF)
    HOD->>SYS: Submit Staff Requisition Form (SRF) for 2 Clinical Nurses (Budget: KES 190,000)
    SYS->>SYS: Match Rule in approval_matrices (Type: REQUISITION, Dept: Nursing)
    SYS->>FM: Notify Step 1: Finance Budget Verification
    FM->>SYS: Approve Budget & Cost Centre Allocation
    SYS->>HR: Notify Step 2: HR Staffing & Headcount Verification
    HR->>SYS: Approve Requisition
    SYS->>CEO: Notify Step 3: Executive Final Authorization
    CEO->>SYS: Authorize Requisition
    SYS->>SYS: Auto-generate Vacancy record (vacancies) linked to Requisition

    Note over HR,Cand: Phase B: Candidate Sourcing & Pipeline
    HR->>SYS: Publish Vacancy (vacancies.status = 'OPEN')
    Cand->>SYS: Apply via Portal or LinkedIn (candidates created at 'APPLIED' stage)
    HR->>SYS: Screen, Shortlist & Schedule Interview (interviews)
    HR->>SYS: Interview Panel logs Scorecard (interview_evaluations: Score 94.5/100)
    HR->>SYS: Move Candidate to 'OFFER' stage

    Note over HR,CEO: Phase C: Job Offer & Sign-off
    HR->>SYS: Draft Job Offer (offers: Basic KES 75,000 + Allowances KES 20,000)
    SYS->>CEO: Trigger Offer Approval Matrix
    CEO->>SYS: Approve Offer Terms
    SYS->>Cand: Dispatch Offer Letter with Digital Signatures (authorized_signatories)
    Cand->>SYS: Accept Offer & Confirm Joining Date

    Note over HR,SYS: Phase D: Atomic Employee Onboarding
    HR->>SYS: Trigger "Convert Candidate to Employee"
    SYS->>SYS: 1. Create employees record (EMP00005, Amina Gitau)
    SYS->>SYS: 2. Create employment_details (Dept, Grade, JobTitle, CostCentre)
    SYS->>SYS: 3. Create salary_details (Basic: KES 75,000, Gross: KES 95,000)
    SYS->>SYS: 4. Create bank_details (Equity Bank Kilimani Branch)
    SYS->>SYS: 5. Assign default Shift & Schedule (employee_shift_assignments)
    SYS->>SYS: 6. Provision Self-Service User Login (users: amina.gitau)
    SYS->>SYS: 7. Initialize Annual Leave Balances (leave_balances: 21 Days)
    SYS->>SYS: 8. Update candidates.convertedEmployeeId = EMP00005
```

---

### 3.2 Flow 2: Daily Time, Attendance & Regularisation Lifecycle

This flow connects Hardware Biometric Terminals, Shift Rosters, Geofencing, Late/Early Penalty Calculations, and Missed Punch Adjustments.

```mermaid
sequenceDiagram
    autonumber
    actor Emp as Clinical Employee
    participant Device as Biometric Terminal (ZKTeco)
    participant Sync as Attendance Ingest Service
    participant Engine as Attendance Rules Engine
    actor Mgr as Line Manager

    Emp->>Device: Punch In via Fingerprint / RFID (07:55 AM)
    Device->>Sync: Stream Raw Event via TCP/IP Socket (Port 4370)
    Sync->>Sync: Store Raw Punch in biometric_punch_logs (PunchAt: 07:55, DeviceId: 1)
    
    Note over Sync,Engine: Nightly or Scheduled Attendance Evaluation
    Sync->>Engine: Process Daily Logs for active Company
    Engine->>Engine: Lookup employee_shift_assignments (Shift: Morning Clinical 08:00 - 17:00)
    Engine->>Engine: Check company_attendance_policies:
    Note right of Engine: - Flexi Window: 30 mins<br/>- Grace Period: 15 mins<br/>- Geofence: 150m verified
    
    alt Clocked In on Time (07:55 vs 08:00)
        Engine->>Engine: Status = 'PRESENT', LateMinutes = 0
    else Clocked In after Grace (08:35 AM)
        Engine->>Engine: Status = 'LATE', LateMinutes = 35
        Engine->>Engine: Check Monthly Grace Count (> 3 occurrences -> Flag Half Day)
    end

    Engine->>Engine: Store evaluated record in attendance_records

    Note over Emp,Mgr: Exception Handling: Regularisation Workflow
    opt Missed Punch or Official Offsite Duty Delay
        Emp->>Engine: Submit Regularisation Request (regularisations: "Emergency duty delay")
        Engine->>Mgr: Route to Line Manager via approval_matrices
        Mgr->>Engine: Approve Regularisation Request
        Engine->>Engine: Recalculate attendance_records: Clear Late Penalty & Update Hours
    end
```

---

### 3.3 Flow 3: Statutory Leave Lifecycle & Operational Handover

This flow illustrates leave balance accrual, application with relief officer duty handover, two-tier approval, and automatic attendance calendar integration.

```mermaid
sequenceDiagram
    autonumber
    actor Emp as Employee (Amina Gitau)
    actor Relief as Relief Nurse (Chepkemoi Mercy)
    actor Mgr as Line Manager (Nafula Gitau)
    actor HR as HR Head (Caroline Nduta)
    participant Engine as Leave Engine (leave_requests)

    Note over Emp,Engine: Step 1: Leave Entitlement & Balance Check
    Emp->>Engine: Check leave_balances (lt-ann: Allocated 21, Used 0, Available 21)
    Emp->>Engine: Submit Leave Request (5 days: Jan 15 - Jan 20, 2026)
    Emp->>Engine: Nominate Relief Officer (Relief: seed-emp-00645)

    Note over Relief,Engine: Step 2: Relief Duty Handover Confirmation
    Relief->>Engine: Acknowledge patient ward care coverage

    Note over Mgr,Engine: Step 3: Approval Matrix Evaluation
    Engine->>Engine: Load approval_matrices (MatrixType: 'LEAVE', Step 1: REPORTING_MANAGER)
    Mgr->>Engine: Review team roster coverage & Approve (leave_decisions: Step 1 APPROVED)
    
    Engine->>Engine: Route to Step 2: HR_REVIEW
    HR->>Engine: Statutory Compliance & Leave Policy Check -> Authorize (leave_decisions: Step 2 APPROVED)

    Note over Engine: Step 4: Atomic State & Calendar Synchronization
    Engine->>Engine: 1. Update leave_requests.status = 'APPROVED'
    Engine->>Engine: 2. Decrement leave_balances (Used: 5, Balance: 16)
    Engine->>Engine: 3. Synchronize attendance_records for Jan 15-20: Status = 'ON_LEAVE'
    Engine->>Engine: 4. Dispatch in-app notification (notifications) to Employee
```

---

### 3.4 Flow 4: Multi-Country Payroll Computation & Loan Amortization Engine

This flow illustrates the financial core: how attendance days, recurring pay components, loan deductions, and statutory tax brackets combine to calculate itemized payslips and generate bank transfer files.

```mermaid
sequenceDiagram
    autonumber
    actor Fin as Finance Manager
    participant Engine as Payroll Engine
    actor HR as HR Head
    actor CEO as Managing Director / CEO
    actor Bank as Central Bank Clearing (KBA/EFT)

    Fin->>Engine: Initialize Monthly Payroll Batch (payroll_runs: comp-001, Feb 2026)
    Engine->>Engine: Lock Attendance Period (attendance_records: Feb 1 - Feb 28)
    
    loop For Every Active Employee in Company
        Engine->>Engine: 1. Fetch salary_details (Basic Salary: KES 75,000)
        Engine->>Engine: 2. Add Recurring Allowances from employee_pay_components:<br/>- Night Duty Allowance: KES 15,000<br/>- Risk Allowance: KES 10,000<br/>- Commuter Allowance: KES 12,000<br/>=> Gross Pay = KES 112,000.00
        
        Engine->>Engine: 3. Calculate Statutory Deductions (statutory_configs 2026):<br/>- NSSF Tier I (6% of 7,000) = KES 420<br/>- NSSF Tier II (6% of 29,000) = KES 1,740<br/>=> Total NSSF = KES 2,160.00<br/>- SHIF (2.75% of Gross) = KES 3,080.00<br/>- Housing Levy (1.5% of Gross) = KES 1,680.00
        
        Engine->>Engine: 4. Compute Taxable Pay & PAYE (paye_bands):<br/>Taxable Pay = Gross - NSSF Tier I/II = KES 109,840.00<br/>Gross Tax = KES 26,900.00<br/>Less Personal Relief = KES 2,400.00<br/>=> Net PAYE Tax = KES 24,500.00
        
        Engine->>Engine: 5. Check Active Loans (employee_loans & loan_installments):<br/>Fetch due installment (Installment #2: Scheduled KES 10,000)<br/>Deduct KES 10,000 & Log loan_transactions
        
        Engine->>Engine: 6. Deduct Non-Statutory Check-Offs (SACCO Savings: KES 5,000)
        
        Engine->>Engine: 7. Compute Net Take-Home Pay:<br/>Net Pay = Gross (112,000) - Total Statutory (31,420) - Loan (10,000) - SACCO (5,000)<br/>=> Net Pay = KES 65,580.00
        
        Engine->>Engine: 8. Insert itemized payslip_lines record
    end

    Note over Fin,CEO: Phase B: Multi-Tier Executive Sign-Off
    Fin->>Engine: Reconcile General Ledger & Submit for Review
    Engine->>HR: Route to HR Head for Headcount & Variance Check
    HR->>Engine: Approve Headcount
    Engine->>CEO: Route to Managing Director for Final Authorization
    CEO->>Engine: Final Payroll Sign-off (payroll_runs.status = 'PAID')

    Note over Fin,Bank: Phase C: Bank Disbursement & File Export
    Fin->>Engine: Trigger "Generate Bank Disbursement File"
    Engine->>Engine: Generate CBK Standard KBA/EFT ASCII text file using bank_details
    Engine->>Bank: Transmit Encrypted Bank Batch File for Employee Account Credit
    Engine->>Engine: Publish Self-Service Payslips (/payroll/my-payslips)
```

---

### 3.5 Flow 5: Organizational DMS, KYC Document Compliance & HR Letters

This flow illustrates how employee onboarding documents are stored, tracked for regulatory license expiry, and how verified employee data merges into official letter templates.

```mermaid
sequenceDiagram
    autonumber
    actor Emp as Clinical Employee
    actor HR as HR Officer
    actor Sig as Authorized Signatory (CEO)
    participant DMS as Document Engine
    participant Svc as Azure Blob Storage Service

    Note over Emp,DMS: Tab 5: Employee Dossier Compliance Upload
    Emp->>DMS: Upload Annual Retention License (KMPDC Doctor License)
    DMS->>Svc: Stream file to Azure Blob Container ('hrms-employee-documents')<br/>Storage Key: 'docs/comp-001/emp-01/guid_license.pdf'
    Svc-->>DMS: Return Storage Key & Hash
    DMS->>DMS: Insert employee_documents (expiryDate: '2026-12-31', isVerified: 0)
    
    HR->>DMS: Review License against KMPDC National Portal
    HR->>DMS: Mark Verified (isVerified: 1, verifiedBy: HR Officer)
    
    Note over DMS: Automatic 30/60/90 Day Expiry Surveillance
    loop Daily Expiry Job
        DMS->>DMS: Scan employee_documents & employee_credentials
        opt Expiry within 60 Days
            DMS->>DMS: Flag Badge = 'EXPIRING_60'
            DMS->>Emp: Dispatch in-app warning notification (notifications)
        end
    end

    Note over HR,Sig: Official Letter Generation
    HR->>DMS: Select Letter Template (letter_templates: 'PROBATION_CONFIRM')
    DMS->>DMS: Fetch Employee Data (Name, JobTitle, ConfirmationDate)
    DMS->>DMS: Interpolate Merge Variables into Template HTML
    DMS->>Sig: Select Default Signatory (authorized_signatories: Dr. James Kariuki)
    DMS->>DMS: Attach Digital Signature & Institutional Seal
    DMS->>DMS: Generate PDF and store in issued_letters
    DMS->>DMS: Log letter_audit_logs ('LETTER_ISSUED', Actor: Caroline Nduta)
```

---

### 3.6 Flow 6: Corporate Governance & M&A Realignment

This flow illustrates how an operating subsidiary (e.g. `Zenith Specialist Clinics`) can be transitioned from Conglomerate Group A (`Bliss Medical Group`) to Conglomerate Group B (`LCH Healthcare Group`) with zero disruption to contracts, tax cards, or payroll.

```mermaid
sequenceDiagram
    autonumber
    actor Super as Super Administrator (CEO)
    participant Core as Tenancy Governance Engine
    participant Log as corporate_movement_logs

    Super->>Core: Initiate Company Realignment (Company: 'comp-z', From: 'grp-bliss', To: 'grp-lch')
    Core->>Core: Validate Target Conglomerate ('grp-lch' active)
    
    Note over Core: Zero Employee Disruption
    Note over Core: All employees belong to comp-z (Employee.CompanyId = 'comp-z').<br/>Contracts, P9 tax cards, and bank routing remain completely untouched.

    Core->>Core: Atomic Update: UPDATE companies SET groupId = 'grp-lch' WHERE id = 'comp-z'
    
    Core->>Log: Insert Immutable Movement Audit Record:
    Note right of Log: - movementType = 'COMPANY_REALIGNMENT'<br/>- entityId = 'comp-z'<br/>- fromGroupId = 'grp-bliss'<br/>- toGroupId = 'grp-lch'<br/>- effectiveDate = '2026-01-01'<br/>- performedByUserId = 'usr-001'<br/>- reason = 'Conglomerate portfolio realignment'
    
    Core->>Super: Return HTTP 200 OK + Updated Corporate Org Chart
```

---

## 4. Comprehensive Foreign Key Connectivity Matrix

The table below lists all **85 relational database entities**, their parent tables, their dependent child tables, and their primary functional purpose in the system:

| # | Table Name | Parent Entity (Foreign Keys) | Dependent Child Entities | Functional Purpose in HRMS |
|---|---|---|---|---|
| **1** | `groups` | `groups` (Self-parent for apex) | `companies`, `corporate_movement_logs` | Conglomerate holding groups & apex consolidation |
| **2** | `companies` | `groups` | All 83 tenant-scoped tables | Legal incorporated operating entities (KRA PIN, Legal Name) |
| **3** | `divisions` | `companies` | `departments` | Operational service wings (Inpatient, Outpatient, Diagnostics) |
| **4** | `roles` | *None* | `role_permissions`, `users`, `user_access_grants` | Security access roles (Superadmin, HR, Finance, Manager, Employee) |
| **5** | `permissions` | *None* | `role_permissions` | 39 granular system capabilities across all modules |
| **6** | `role_permissions` | `roles`, `permissions` | *None* | Many-to-many permission grants per role |
| **7** | `users` | `roles`, `companies` | `employees`, `sessions`, `notifications`, `audit_log` | System login accounts, hashed credentials, lockout rules |
| **8** | `user_company_access`| `users`, `companies` | *None* | Permitted operating companies in header context switcher |
| **9** | `user_access_grants` | `users`, `roles` | *None* | Multi-tier scoped access (Enterprise, Group, Company, Branch) |
| **10**| `sessions` | `users` | *None* | Active JWT bearer session tokens and client IP tracking |
| **11**| `password_resets` | `users` | *None* | Time-limited password recovery tokens |
| **12**| `currencies` | `companies` | `companies`, `job_requisitions` | Treasury FX rates, ISO codes, format patterns, and base pegs |
| **13**| `departments` | `companies`, `divisions`, `employees` | `employment_details`, `job_requisitions`, `vacancies` | Organizational departments, clinical units, and HOD links |
| **14**| `locations` | `companies` | `employment_details`, `shifts`, `biometric_devices` | Hospital branch clinics, physical campuses, and GPS geofences |
| **15**| `job_titles` | `companies` | `employment_details`, `job_requisitions`, `vacancies` | Standard designations (Medical Officer, Registered Nurse) |
| **16**| `grades` | `companies` | `employment_details`, `job_requisitions`, `vacancies` | Salary bands, executive grade levels, and compensation limits |
| **17**| `cost_centres` | `companies` | `employment_details`, `job_requisitions` | Financial general ledger cost allocation codes |
| **18**| `employment_types` | `companies` | `employment_details` | Contract terms (Permanent, Fixed-term, Locum, Probation) |
| **19**| `banks` | *None* | `bank_branches`, `bank_details` | Central bank registered clearing banks (CBK, CBUAE, RBI) |
| **20**| `bank_branches` | `banks` | `bank_details` | Bank branches, clearing codes, IFSC, MICR, IBAN |
| **21**| `statutory_configs` | `companies` | `paye_bands`, `payroll_runs` | Statutory tax rates (PAYE 2026, NSSF Tier I/II, SHIF, Housing Levy) |
| **22**| `paye_bands` | `statutory_configs` | *None* | Graduated progressive tax brackets (10% to 35%) |
| **23**| `pay_components` | `companies` | `employee_pay_components` | Allowance and deduction catalog (Call, Risk, Night, SACCO) |
| **24**| `holiday_types` | `companies` | `holidays` | Holiday classification (National Gazetted, Religious, County) |
| **25**| `holidays` | `companies`, `holiday_types` | *None* | Calendar public holidays and gazette notices |
| **26**| `shifts` | `companies`, `locations` | `shift_day_rules`, `employee_shift_assignments` | Shift definitions, start/end hours, midnight spans, flexi-time |
| **27**| `shift_day_rules` | `shifts` | *None* | 7-day Day-by-Day Roster structure per shift |
| **28**| `work_schedules` | `companies` | `work_schedule_days`, `employee_shift_assignments` | Baseline corporate work week calendars |
| **29**| `work_schedule_days` | `work_schedules` | *None* | Mon-Sun working/rest flags per schedule |
| **30**| `company_attendance_policies` | `companies` | *None* | Grace periods, flexi-windows, late penalties, geofencing |
| **31**| `biometric_devices` | `companies`, `locations` | `biometric_punch_logs` | ZKTeco/TCP physical hardware terminal registry |
| **32**| `licensing_boards` | `companies` | `employee_credentials` | Professional regulators (KMPDC, NCK, PPB, DHA, NMC) |
| **33**| `asset_categories` | `companies` | `employee_assets` | Equipment categories (Laptops, Medical Devices, Badges) |
| **34**| `separation_reasons` | `companies` | *None* | Exit classifications (Resignation, Retirement, Redundancy) |
| **35**| `benefit_plans` | `companies` | `employee_benefit_enrollments` | Medical, life, and pension scheme benefit policies |
| **36**| `employees` | `companies`, `users` | All 25 employee child tables | Central employee entity (Name, Demographics, ID, Status) |
| **37**| `employment_details` | `employees`, `departments`, `locations`, `job_titles`, `grades`, `cost_centres`, `employment_types` | *None* | Job placement, reporting manager, confirmation, and probation |
| **38**| `salary_details` | `employees` | *None* | Basic pay, gross salary, KRA PIN, NSSF, SHIF, payment mode |
| **39**| `bank_details` | `employees`, `banks`, `bank_branches` | *None* | Disbursement bank account number, branch routing, and currency |
| **40**| `employee_pay_components` | `employees`, `pay_components` | `payslip_lines` | Recurring monthly allowances and check-off deductions |
| **41**| `employee_shift_assignments` | `employees`, `companies`, `shifts`, `work_schedules` | `attendance_records` | Effective-dated shift rosters and work pattern assignments |
| **42**| `employee_shift_day_overrides` | `employees`, `companies`, `shifts` | `attendance_records` | Specific date shift swaps and roster overrides |
| **43**| `employee_benefit_enrollments` | `employees`, `benefit_plans` | *None* | Healthcare policy enrollments and covered dependents |
| **44**| `employee_credentials` | `employees`, `licensing_boards` | *None* | Medical and nursing retention licenses with CPD points |
| **45**| `employee_assets` | `employees`, `asset_categories` | *None* | Hospital equipment custody tracking and exit clearance |
| **46**| `biometric_punch_logs` | `employees` | `attendance_records` | Raw terminal clocking events streamed from biometric devices |
| **47**| `attendance_records` | `companies`, `employees`, `shifts` | `regularisations`, `payroll_runs` | Computed daily attendance, late minutes, hours, and status |
| **48**| `regularisations` | `attendance_records`, `users` | *None* | Missed punch adjustments and managerial approvals |
| **49**| `leave_types` | `companies` | `leave_balances`, `leave_requests` | Leave entitlements (Annual, Sick, Maternity, Study) |
| **50**| `leave_balances` | `companies`, `employees`, `leave_types` | *None* | Annual leave accrual, used days, and remaining balances |
| **51**| `leave_requests` | `companies`, `employees`, `leave_types`, `users` | `leave_decisions`, `attendance_records` | Leave applications, relief officer coverage, and approval state |
| **52**| `leave_decisions` | `leave_requests`, `users` | *None* | Step-by-step leave approval and rejection audit trail |
| **53**| `letter_categories` | `companies` | `letter_templates` | Letter classification (Appointment, Confirmation, Promotion) |
| **54**| `authorized_signatories` | `companies` | `letter_templates`, `issued_letters` | Executive digital signatures and institutional seals |
| **55**| `letter_templates` | `companies`, `letter_categories`, `authorized_signatories` | `issued_letters` | HTML letter templates and merge variable placeholders |
| **56**| `issued_letters` | `companies`, `letter_templates`, `employees`, `authorized_signatories` | `letter_audit_logs` | Generated official PDF letters and reference numbers |
| **57**| `letter_audit_logs` | `companies`, `issued_letters` | *None* | Tamper-proof audit logs for letter downloads and prints |
| **58**| `document_types` | `companies` | `employee_documents` | Mandatory employee compliance documents and expiry rules |
| **59**| `employee_documents` | `companies`, `employees`, `document_types`, `users` | *None* | Personnel dossier files, Azure Blob keys, and verification state |
| **60**| `document_folders` | `companies` | `company_documents` | Organizational DMS folder hierarchy (Policies, Handbooks) |
| **61**| `company_documents` | `companies`, `document_folders` | *None* | Hospital SOPs, clinical protocols, and accreditation files |
| **62**| `recruitment_sources` | *None* | `candidates` | Talent sourcing channels (LinkedIn, Portal, Referrals) |
| **63**| `pipeline_stages` | `companies` | `candidates`, `interviews` | Jira-style Kanban pipeline stages with SLA day limits |
| **64**| `approval_matrices` | `companies`, `departments`, `grades` | `job_requisitions`, `offers`, `leave_requests`, `payroll_runs` | Universal dynamic rule engine for 8 approval domains |
| **65**| `job_requisitions` | `companies`, `departments`, `job_titles`, `grades`, `locations`, `cost_centres`, `employees`, `users` | `requisition_approvals`, `vacancies` | Staff Requisition Forms (SRF), headcount, and budgets |
| **66**| `requisition_approvals` | `job_requisitions`, `users` | *None* | Multi-level requisition decision history |
| **67**| `vacancies` | `companies`, `job_requisitions`, `job_titles`, `departments`, `locations`, `grades` | `candidates`, `offers` | Published job openings and applicant limits |
| **68**| `candidates` | `companies`, `vacancies`, `recruitment_sources`, `pipeline_stages`, `employees` | `candidate_stage_history`, `candidate_documents`, `candidate_notes`, `interviews`, `offers` | ATS applicants, scoring, and employee conversion links |
| **69**| `candidate_stage_history` | `candidates` | *None* | Historical ledger of candidate pipeline stage progressions |
| **70**| `candidate_documents` | `candidates` | *None* | Applicant resumes, certificates, and portfolio uploads |
| **71**| `candidate_notes` | `candidates` | *None* | Internal recruiter and interviewer assessment notes |
| **72**| `interviews` | `companies`, `candidates`, `pipeline_stages` | `interview_evaluations` | Scheduled panel interviews, times, and meeting links |
| **73**| `interview_evaluations` | `interviews` | *None* | Interviewer scorecards, ratings, and recommendation notes |
| **74**| `offers` | `companies`, `candidates`, `vacancies` | `offer_approvals` | Employment offers, compensation terms, and validity |
| **75**| `offer_approvals` | `offers`, `users` | *None* | Executive offer sign-off decision audit trail |
| **76**| `employee_loans` | `companies`, `employees` | `loan_installments`, `loan_transactions` | Corporate staff loan ledger, principal, interest, and EMI |
| **77**| `loan_installments` | `employee_loans` | `payslip_lines` | 12-month amortization schedule and monthly deduction status |
| **78**| `loan_transactions` | `employee_loans` | *None* | Recorded payroll deductions and manual loan repayments |
| **79**| `payroll_runs` | `companies`, `users` | `payslip_lines` | Monthly statutory payroll calculation batches and approvals |
| **80**| `payslip_lines` | `payroll_runs`, `employees` | *None* | Itemized payslips (Gross, PAYE, NSSF, SHIF, Loans, Net Pay) |
| **81**| `company_settings` | `companies` | *None* | Company key-value configuration registry (Policies, Features) |
| **82**| `corporate_movement_logs` | `groups`, `companies`, `users` | *None* | M&A and corporate realignment audit ledger ("Who did what") |
| **83**| `notifications` | `users` | *None* | In-app alerts, bell notifications, and action reminders |
| **84**| `import_batches` | `companies`, `users` | *None* | Bulk CSV/Excel employee upload execution logs |
| **85**| `audit_log` | `companies` | *None* | Immutable enterprise event trail across all entity mutations |

---

## 5. Summary of System Benefits & Cohesion

By tying these 85 entities together across Clean Architecture layers in .NET 10 and Next.js 15+:
1. **Zero Data Redundancy:** A single employee record links to their shift roster, daily biometric clocking, annual leave accrual, health benefits, loans, and monthly payslips.
2. **Single-Pane-of-Glass Governance:** Corporate groups can oversee multiple hospital subsidiaries, compare workforce metrics, and realign companies seamlessly.
3. **Auditable & Compliant:** Every managerial decision (Requisition, Offer, Regularisation, Leave, Payroll) is governed by transparent `approval_matrices` and permanently logged in immutable audit tables.
