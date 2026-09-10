# HRMS Enterprise System — Comprehensive Feature Specification & Architecture
**Target Stack:** .NET 10 Core (Repository Pattern) | Next.js (Port: 4000) | Microsoft SQL Server

---

## 1. System Architecture & Multi-Tenancy Foundation

### 1.1 Multi-Company Hierarchy & Scoping
* **Hierarchy:** `Group -> Company -> Division -> Department -> Location -> Employee`
* **Tenancy Rules:**
  * Every tenant-scoped entity must include `CompanyId`.
  * Multi-company users (e.g., Group HR, Group Finance) can view consolidated group data or switch company context via a top-bar company selector.
  * Single-company users are locked strictly to their assigned company.
  * In "All Companies" consolidated view, data mutation (create/update) is prohibited at the API layer. Mutations must be executed within an explicit company context.
  * Cross-tenant record queries must return HTTP 404 (Not Found), never HTTP 403 (Forbidden), preventing ID enumeration.

### 1.2 Target Technology Stack
* **Database:** Microsoft SQL Server 2022+
  * Identity/PK: `NVARCHAR(36)` or `UNIQUEIDENTIFIER` (supporting CUID/UUID).
  * Currency/Monetary fields: `DECIMAL(14, 2)`.
  * Dates: `DATE` for calendar dates (DOB, join date), `DATETIME2(7)` for audit and timestamps.
  * Tenancy Indexes: Composite non-clustered indexes on `(CompanyId, IsActive)` across all tenant tables.
* **Backend:** .NET 10 Core Web API
  * Architecture: Clean Architecture with Repository & Unit of Work Pattern.
  * Projects:
    * `Hrms.Domain`: Pure domain entities, enums, domain events, interface contracts.
    * `Hrms.Application`: DTOs, service interfaces, CQRS/command handlers, FluentValidation rules, business calculations.
    * `Hrms.Infrastructure`: `ApplicationDbContext` (EF Core 10), generic and specialized repositories, SQL Server configurations, global query filters for multi-tenancy, file storage providers (Azure Blob / S3), hashing services.
    * `Hrms.Api`: REST Controllers, JWT authentication, Permission-based authorization filters, Tenancy middleware, Swagger/OpenAPI.
* **Frontend:** Next.js (App Router, Port 4000)
  * Dev server execution: `next dev -p 4000`.
  * Framework: React 19, TypeScript, Vanilla CSS / Tailwind CSS tokens.
  * State Management & API: TanStack Query (React Query) / SWR, Axios/Fetch interceptors handling JWT rotation and `X-Company-Context` header.

---

## 2. Step-by-Step Feature Specification

```
Step 01: Authentication, Authorization & Audit Engine
Step 02: Group, Multi-Company & Division Master
Step 03: Organisation Master Data (Dept, Location, Job, Grade, Cost Centre)
Step 04: Employee Master, History & Inter-Company Transfer
Step 05: Biometric Attendance, Shifts & Holiday Calendar
Step 06: Leave Engine & Workflow (Employment Act 2007)
Step 07: Payroll Computation Engine & Kenya Statutory Returns
Step 08: HR Letters & Document Merge Engine
Step 09: Personnel Records, Clinical Credentials & Compliance Tracker
Step 10: Recruitment Pipeline & Applicant Tracking System (ATS)
Step 11: Multi-Company Analytics, Liability & Executive Dashboards
Step 12: Backup, Restore & Data Migration
Step 13: Technical Implementation Blueprints (.NET 10 & Next.js)
```

---

### STEP 01: Authentication, Authorization & Audit Engine

#### 1. Features & Business Rules
* **Authentication:** Username/Email + password credentials.
* **Password Hashing:** ASP.NET Core PasswordHasher / Argon2id / PBKDF2-SHA256 (120,000+ iterations).
* **Account Lockout:** Lock account after 5 consecutive failed attempts.
* **Session Lifecycle:** 30-minute rolling inactivity timeout; refresh token mechanism; single-session or revocable active sessions.
* **Role-Based Access Control (RBAC):**
  * Matrix of 30+ granular permissions.
  * Data Scopes:
    * `ALL`: Full access across assigned companies.
    * `TEAM`: Reporting line hierarchy (direct and indirect reports).
    * `SELF`: Own record only.
* **Confidential Data Redaction:** API automatically strips `basicSalary`, `bankAccount`, `kraPin`, `nssfNumber`, `shifNumber` if caller lacks `VIEW_SALARY` or `VIEW_BANK`.
* **Append-Only Audit Trail:**
  * Every mutation records: `Timestamp`, `UserId`, `Username`, `Action`, `Module`, `RecordIdentifier`, `OldValue` (JSON), `NewValue` (JSON), `IpAddress`.
  * Zero delete or update endpoints exposed in DB and API.

#### 2. SQL Server Schema
* `Users` (`Id`, `Username`, `Email`, `PasswordHash`, `FullName`, `EmployeeId`, `RoleId`, `IsActive`, `FailedLoginAttempts`, `LockoutEndUtc`, `DefaultCompanyId`, `CreatedAt`, `UpdatedAt`)
* `Roles` (`Id`, `Name`, `Description`, `Scope` [0=Self, 1=Team, 2=All], `IsSystem`, `CreatedAt`, `UpdatedAt`)
* `Permissions` (`Id`, `Code`, `Category`, `Description`)
* `RolePermissions` (`RoleId`, `PermissionId`)
* `UserSessions` (`Id`, `UserId`, `TokenHash`, `ExpiresAtUtc`, `LastActivityUtc`, `IpAddress`, `UserAgent`)
* `AuditLogs` (`Id`, `TimestampUtc`, `UserId`, `Username`, `Action`, `Module`, `RecordId`, `OldValue`, `NewValue`, `IpAddress`)

#### 3. Pre-loaded Permission Matrix
* **Employee Records:** `VIEW_EMPLOYEES`, `CREATE_EMPLOYEES`, `EDIT_EMPLOYEES`, `DELETE_EMPLOYEES`, `IMPORT_EMPLOYEES`
* **Confidential Data:** `VIEW_SALARY`, `EDIT_SALARY`, `VIEW_BANK`
* **Organisation:** `MANAGE_DEPARTMENTS`, `MANAGE_LOCATIONS`, `MANAGE_JOB_TITLES`, `MANAGE_GRADES`, `MANAGE_COST_CENTRES`
* **Administration:** `MANAGE_USERS`, `MANAGE_ROLES`, `MANAGE_SETTINGS`, `VIEW_AUDIT`
* **Reporting:** `VIEW_REPORTS`, `VIEW_MGMT_DASHBOARD`
* **Time & Leave:** `VIEW_LEAVE`, `APPLY_LEAVE`, `APPROVE_LEAVE`, `MANAGE_LEAVE`, `VIEW_ATTENDANCE`, `MARK_ATTENDANCE`, `MANAGE_ATTENDANCE`, `APPROVE_REGULARISATION`, `MANAGE_LEAVE_TYPES`
* **Payroll:** `VIEW_PAYROLL`, `RUN_PAYROLL`, `APPROVE_PAYROLL`, `MANAGE_PAY_STRUCTURE`, `MANAGE_PAYROLL_CONFIG`, `EXPORT_BANK_FILE`
* **Letters:** `VIEW_LETTERS`, `ISSUE_LETTERS`, `MANAGE_LETTER_TEMPLATES`
* **Documents:** `VIEW_DOCUMENTS`, `UPLOAD_DOCUMENTS`, `VERIFY_DOCUMENTS`, `MANAGE_DOCUMENT_TYPES`, `VIEW_COMPLIANCE`
* **Group Management:** `MANAGE_COMPANIES`, `MANAGE_COMPANY_ACCESS`, `VIEW_GROUP_DASHBOARD`, `TRANSFER_INTER_COMPANY`
* **Recruitment:** `VIEW_RECRUITMENT`, `CREATE_REQUISITION`, `APPROVE_REQUISITION`, `MANAGE_VACANCY`, `VIEW_CANDIDATES`, `CREATE_CANDIDATE`, `EDIT_CANDIDATE`, `SCHEDULE_INTERVIEW`, `SUBMIT_EVALUATION`, `CREATE_OFFER`, `APPROVE_OFFER`, `CONVERT_CANDIDATE`, `VIEW_RECRUITMENT_REPORTS`

---

### STEP 02: Group, Multi-Company & Division Master

#### 1. Features & Business Rules
* **Group Entity:** Top-tier holding entity holding legal name, registration number, and default country.
* **Company Entity:**
  * Stable code format: `C001`, `C002`, `C003`.
  * Metadata: Legal name, Registration No, KRA PIN, Currency (default `KES`), Timezone (`Africa/Nairobi`), Phone, Email, Address, Financial year start/end (`01 January` - `31 December`).
  * Approval Configurations:
    * `LeaveApprovalRoute`: `MANAGER_THEN_HR`, `MANAGER_ONLY`, `HR_ONLY`.
    * `RequisitionApproval`: JSON array specifying ordered approval steps (e.g. `['HOD', 'HR', 'Finance', 'CEO']`).
    * `OfferApproval`: JSON array specifying ordered offer approval steps (e.g. `['HR', 'Finance']`).
* **Division Entity:** Sub-unit between Company and Department (e.g., Hospital Operations, Pharmacy, Outpatient Clinics).
* **User Company Grants (`UserCompanyAccess`):** Explicit mapping linking users to 1..N companies or flag for Entire Group.
* **Company Master Cloning:** When provisioning a new company, optional trigger to duplicate master data (Departments, Locations, Job Titles, Grades, Cost Centres) from an existing company template.

#### 2. SQL Server Schema
* `Groups` (`Id`, `Name`, `LegalName`, `RegNumber`, `Country`, `CreatedAtUtc`)
* `Companies` (`Id`, `GroupId`, `Code`, `Name`, `LegalName`, `RegNumber`, `KraPin`, `Country`, `Currency`, `Address`, `Phone`, `Email`, `Website`, `LogoUrl`, `TimeZone`, `FinancialYearStart`, `FinancialYearEnd`, `LeaveApprovalRoute`, `RequisitionApprovalStepsJson`, `OfferApprovalStepsJson`, `IsActive`, `CreatedAtUtc`, `UpdatedAtUtc`)
* `Divisions` (`Id`, `CompanyId`, `Name`, `IsActive`, `CreatedAtUtc`)
* `UserCompanyAccess` (`UserId`, `CompanyId`, `GrantedAtUtc`, `GrantedByUserId`, PRIMARY KEY (`UserId`, `CompanyId`))

---

### STEP 03: Organisation Master Data

#### 1. Features & Business Rules
* **Master Entities:**
  * Departments (Name, Code, CompanyId, IsActive)
  * Locations (Name, Code, Address, CompanyId, IsActive)
  * Job Titles (Name, Code, CompanyId, `IsClinical` flag, IsActive)
    * `IsClinical` flag automatically drives clinical licence and medical malpractice indemnity compliance rules in Step 09.
  * Grades (Name, Code, CompanyId, Level [Int], MinSalary, MaxSalary, IsActive)
  * Cost Centres (Name, Code, CompanyId, IsActive)
* **Lifecycle:** Soft deactivation only (`IsActive = 0`). Hard deletion is prevented if references exist in Employee or Payroll tables.

#### 2. SQL Server Schema
* `Departments` (`Id`, `CompanyId`, `Code`, `Name`, `IsActive`, `CreatedAtUtc`)
* `Locations` (`Id`, `CompanyId`, `Code`, `Name`, `Address`, `IsActive`, `CreatedAtUtc`)
* `JobTitles` (`Id`, `CompanyId`, `Code`, `Name`, `IsClinical`, `IsActive`, `CreatedAtUtc`)
* `Grades` (`Id`, `CompanyId`, `Code`, `Name`, `Level`, `MinSalary`, `MaxSalary`, `IsActive`, `CreatedAtUtc`)
* `CostCentres` (`Id`, `CompanyId`, `Code`, `Name`, `IsActive`, `CreatedAtUtc`)

---

### STEP 04: Employee Master, History & Inter-Company Transfer

#### 1. Features & Business Rules
* **Employee Record Fields:**
  * Personal: Employee Number (Unique per company), First Name, Middle Name, Last Name, Gender (`Female`, `Male`, `Other`, `Prefer not to say`), DOB, National ID / Passport, Marital Status (`Single`, `Married`, `Divorced`, `Widowed`), Personal Email, Phone, Emergency Contact.
  * Employment: CompanyId, DivisionId, DepartmentId, LocationId, JobTitleId, GradeId, CostCentreId, ManagerId (Reporting line), Employment Type (`Permanent`, `Contract`, `Locum`, `Intern`, `Consultant`, `Casual`), Status (`Active`, `Probation`, `On Notice`, `Suspended`, `Inactive`, `Exited`), Date Joined, Confirmation Date, Contract Start, Contract End, Exit Date, Exit Reason.
  * Statutory & Bank: KRA PIN (Validated format: `^[A-Z]\d{9}[A-Z]$`), NSSF Number, SHIF Number, Bank Name, Bank Branch, Bank Account Number.
  * Compensation: Basic Salary (`DECIMAL(14,2)`), Pay Frequency (`Monthly`, `Fortnightly`, `Weekly`).
* **Validation Engine:** Strict regex verification on KRA PIN, Email, Phone, and positive decimal numbers for salary.
* **Employment Posting History (`EmploymentHistory`):**
  * Tracks every assignment: EffectiveFrom, EffectiveTo (NULL for current), Company, Department, Job Title, Grade, Location, Manager, Basic Salary, Reason.
* **Inter-Company Transfer Flow:**
  1. Closes current `EmploymentHistory` row with `EffectiveTo = TransferDate - 1`.
  2. Creates new `EmploymentHistory` row with new `CompanyId`, new Department, Job Title, Location, and Grade.
  3. Updates active `Employee` record `CompanyId`.
  4. Reporting Manager (`ManagerId`) is cleared (manager in old company is invalid in target company).
  5. Leaves and historic payroll remain immutable in their respective company periods.
* **Employee Bulk Import Engine:**
  * Ingests 36-column CSV.
  * Automatic matching/upserting of Departments, Locations, Titles, Grades, and Cost Centres.
  * Resolves Reporting Managers by employee number lookup.
  * Generates validation batch summary: Total rows, Successful imports, Errors, Duplicates.

#### 2. SQL Server Schema
* `Employees` (`Id`, `CompanyId`, `EmployeeNumber`, `FirstName`, `MiddleName`, `LastName`, `Email`, `Phone`, `Gender`, `DateOfBirth`, `NationalId`, `MaritalStatus`, `DivisionId`, `DepartmentId`, `LocationId`, `JobTitleId`, `GradeId`, `CostCentreId`, `ManagerId`, `EmploymentType`, `Status`, `DateJoined`, `ConfirmationDate`, `ContractStart`, `ContractEnd`, `ExitDate`, `ExitReason`, `BasicSalary`, `PayFrequency`, `BankName`, `BankBranch`, `BankAccount`, `KraPin`, `NssfNumber`, `ShifNumber`, `IsActive`, `CreatedAtUtc`, `UpdatedAtUtc`)
* `EmploymentHistories` (`Id`, `EmployeeId`, `CompanyId`, `DepartmentId`, `JobTitleId`, `GradeId`, `LocationId`, `ManagerId`, `BasicSalary`, `EffectiveFrom`, `EffectiveTo`, `Reason`, `RecordedByUserId`, `RecordedAtUtc`)
* `ImportBatches` (`Id`, `CompanyId`, `FileName`, `UploadedByUserId`, `TotalRecords`, `ImportedCount`, `ErrorCount`, `DuplicateCount`, `ErrorReportJson`, `BatchType`, `StartedAtUtc`, `CompletedAtUtc`)

---

### STEP 05: Biometric Attendance, Shifts & Holiday Calendar

#### 1. Features & Business Rules
* **Shift Master:**
  * Name, Start Time (`TIME`), End Time (`TIME`), Contracted Hours (e.g. 8.00), Grace Period (minutes, e.g. 15), IsNightShift (crosses midnight), Weekly Off Days (JSON array or bitmask, e.g. Saturday & Sunday).
* **Kenyan Holiday Calendar:**
  * Date, Holiday Name, IsGazetted, IsFloating (Eid moon-sighting flags).
* **Attendance Derivation Engine (Evaluated dynamically, never manually typed):**
  * `Present`: Clock-in within Shift Start + Grace Period.
  * `Late`: Clock-in after Grace Period; Lateness minutes calculated.
  * `Half day`: Clock-in after Half-day threshold or total worked hours < 50% of contracted.
  * `Absent`: No clock-in record on working day without approved leave.
  * `On leave`: Overridden automatically from approved leave request for the date.
  * `Weekly off`: Shift designated off-day.
  * `Public holiday`: Matches active Holiday calendar entry.
  * `Regularised`: Missed punch corrected via approval workflow.
* **Overtime Computation:** Hours worked beyond contracted hours; supports night shift rollover across midnight.
* **Ingestion Channels:**
  1. Web/Mobile Punch: Geolocation and timestamp clock in/out.
  2. Biometric CSV Import: Device logs (Emp Number, Timestamp, In/Out Direction).
  3. HR Manual Override: Requires audited justification.
  4. Month-End Processing Run: Batch job generating absent records for unpunched non-holiday working days.
* **Regularisation Workflow:**
  * Employee submits correction request with reason.
  * Routes to Manager or HR Approvals inbox.
  * Approval updates record status to `Regularised` and recalculates worked/overtime hours.

#### 2. SQL Server Schema
* `Shifts` (`Id`, `CompanyId`, `Code`, `Name`, `StartTime`, `EndTime`, `ContractedHours`, `GraceMinutes`, `IsNightShift`, `WeeklyOffDaysJson`, `IsActive`, `CreatedAtUtc`)
* `Holidays` (`Id`, `CompanyId`, `Date`, `Name`, `IsGazetted`, `IsFloatingMoonSighting`, `CreatedAtUtc`)
* `AttendanceRecords` (`Id`, `CompanyId`, `EmployeeId`, `Date`, `ShiftId`, `ClockInUtc`, `ClockOutUtc`, `TotalHoursWorked`, `OvertimeHours`, `LateMinutes`, `Status` [Present, Late, HalfDay, Absent, OnLeave, WeeklyOff, PublicHoliday, Regularised], `IsManualOverride`, `Notes`, `CreatedAtUtc`, `UpdatedAtUtc`)
* `AttendanceRegularisations` (`Id`, `AttendanceRecordId`, `RequestedClockInUtc`, `RequestedClockOutUtc`, `Reason`, `Status` [Pending, Approved, Rejected], `ReviewedByUserId`, `ReviewedAtUtc`, `RejectionReason`)

---

### STEP 06: Leave Engine & Workflow (Employment Act 2007)

#### 1. Statutory Types & Rules
* **Annual Leave (s.28):** 21 working days per year; accrues at 1.75 days per completed month of service; working days calculation automatically excludes weekends and public holidays.
* **Sick Leave (s.30):** 14 days maximum per year; 7 days full pay, followed by 7 days half pay; entitlement unlocks only after 2 completed months of service.
* **Maternity Leave (s.29):** 90 calendar days; female employees only; zero entitlement for male employees.
* **Paternity Leave (s.29(8)):** 14 calendar days; male employees only.
* **Compassionate Leave:** 5 working days (Company policy).
* **Study Leave:** 10 working days; requires minimum 12 months completed service (Company policy).
* **Unpaid Leave:** Up to 30 working days; deductions feed directly into the payroll engine.

#### 2. Processing Engine & Workflow
* **Overlap Prevention:** Hard block on overlapping booking dates for the same employee.
* **Closing Balance Preview:** Real-time calculation showing `Current Balance - Requested Days = Projected Balance`.
* **Approval Routing:**
  * Supported modes: `MANAGER_THEN_HR`, `MANAGER_ONLY`, `HR_ONLY`.
  * Fallback: If employee has no assigned manager, request routes immediately to HR.
* **Balance Ledger:** Leave days deduct from balance **only upon final step approval**.
* **Audit Trail:** Permanent recording of approver decision, comments, and decision timestamp.

#### 3. SQL Server Schema
* `LeaveTypes` (`Id`, `CompanyId`, `Code`, `Name`, `AnnualEntitlementDays`, `AccruesMonthly`, `AccrualRatePerMonth`, `IsGenderRestricted`, `ApplicableGender`, `MinServiceMonths`, `HalfPayAfterDays`, `RequiresCertificate`, `IsActive`, `CreatedAtUtc`)
* `LeaveBalances` (`Id`, `CompanyId`, `EmployeeId`, `LeaveTypeId`, `Year`, `OpeningBalance`, `Accrued`, `Taken`, `Pending`, `ClosingBalance`, `UpdatedAtUtc`)
* `LeaveRequests` (`Id`, `CompanyId`, `EmployeeId`, `LeaveTypeId`, `StartDate`, `EndDate`, `TotalDays`, `Reason`, `AttachmentUrl`, `Status` [Draft, PendingManager, PendingHr, Approved, Rejected, Cancelled], `CurrentApprovalStep`, `CreatedAtUtc`, `UpdatedAtUtc`)
* `LeaveDecisions` (`Id`, `LeaveRequestId`, `Step`, `DecidedByUserId`, `Decision` [Approved, Rejected], `Comments`, `DecidedAtUtc`)

---

### STEP 07: Payroll Computation Engine & Kenya Statutory Returns

#### 1. Statutory Deductions & Tax Bands (Kenya Finance Act Engine)
* **PAYE Graduated Tax Bands:**
  * Band 1: First KES 24,000 @ 10%
  * Band 2: Next KES 8,333 (24,001 to 32,333) @ 25%
  * Band 3: Next KES 467,667 (32,334 to 500,000) @ 30%
  * Band 4: Next KES 300,000 (500,001 to 800,000) @ 32.5%
  * Band 5: Above KES 800,000 @ 35%
* **Tax Reliefs:**
  * Personal Relief: KES 2,400 per month (KES 28,800 annually).
  * Insurance Relief: Evaluated per statutory configuration (SHIF treated as pre-tax deduction).
* **NSSF (National Social Security Fund):**
  * Lower Earnings Limit (LEL - Tier I): Up to KES 8,000 @ 6% = Max KES 480.
  * Upper Earnings Limit (UEL - Tier II): KES 8,001 to KES 72,000 @ 6% = Max KES 3,840.
  * Maximum Employee NSSF: KES 4,320 (matched 100% by Employer).
* **SHIF (Social Health Insurance Fund):**
  * 2.75% of Gross Salary; Statutory minimum: KES 300.
* **Affordable Housing Levy (AHL):**
  * Employee: 1.5% of Gross Salary.
  * Employer: 1.5% of Gross Salary (Employer match).
* **Allowable Pre-Tax Deductions:**
  * NSSF (Employee contribution), SHIF, AHL, and qualifying pension contributions (capped at KES 20,000 or allowable limit).
  * `Taxable Pay = Gross Pay - Allowable Deductions`.

#### 2. Pay Component Architecture
* **Earnings:**
  * Basic Salary
  * Custom Allowances: House, Transport, Medical, Utility (Fixed KES or % of Basic).
  * Overtime Pay: Pulled dynamically from attendance records: `OvertimeHours * (BasicSalary / 26 / 8) * OvertimeMultiplier`.
  * Unpaid Leave Deduction: `UnpaidDays * (BasicSalary / 26)`.
  * Half-Pay Sick Leave Adjustment: `HalfPayDays * (BasicSalary / 26) * 0.5`.
* **Recurring Deductions:** Pre-tax vs Post-tax deductions (Staff Loans, Advances, SACCO, Union dues) with balance tracking.

#### 3. Payroll Run Lifecycle
* `Draft` -> `Computed` -> `Approved` -> `Paid`
* **Run Approval Lock:** Approving a payroll run freezes the calculation, generates payslips, and takes an immutable snapshot of statutory rates in effect, ensuring historical pay runs never drift.
* **Export Artifacts:**
  1. **Payroll Register:** Complete itemized spreadsheet of all components per employee.
  2. **Bank Payment File:** Formatted CSV/text for EFT/RTGS; excludes employees missing bank details and flags an exception report.
  3. **KRA P10 Monthly Tax Return:** Layout mapped for iTax portal import.
  4. **NSSF Return File:** Separating Tier I and Tier II contributions with employer matching.
  5. **SHIF Return File:** Member registration numbers, gross earnings, and 2.75% contributions.
  6. **GL Journal File:** Double-entry journal voucher by Cost Centre (Total Debits == Total Credits).

#### 4. SQL Server Schema
* `StatutoryConfigs` (`Id`, `CompanyId`, `EffectiveYear`, `EffectiveMonth`, `WorkingDaysPerMonth` [default 26], `OvertimeMultiplier` [default 1.5], `PersonalRelief`, `NssfLowerLimit`, `NssfUpperLimit`, `NssfRate`, `NssfEmployerMatches`, `ShifRate`, `ShifMinimum`, `AhlEmployeeRate`, `AhlEmployerRate`, `NssfDeductible`, `ShifDeductible`, `AhlDeductible`, `PensionCap`, `IsActive`)
* `PayeBands` (`Id`, `StatutoryConfigId`, `BandOrder`, `UpToAmount`, `TaxRate`)
* `PayComponents` (`Id`, `CompanyId`, `Code`, `Name`, `ComponentType` [Earning, Deduction], `CalculationMethod` [Fixed, Percentage], `DefaultValue`, `IsTaxable`, `IsPreTax`, `IsActive`)
* `EmployeePayComponents` (`Id`, `EmployeeId`, `PayComponentId`, `CustomAmount`, `CustomPercentage`, `BalanceRemaining`, `IsActive`)
* `PayrollRuns` (`Id`, `CompanyId`, `PeriodMonth` [YYYY-MM], `Status` [Draft, Computed, Approved, Paid], `TotalGross`, `TotalNet`, `TotalEmployerCost`, `TotalPaye`, `TotalNssf`, `TotalShif`, `TotalAhl`, `ComputedAtUtc`, `ApprovedAtUtc`, `ApprovedByUserId`, `StatutorySnapshotJson`)
* `PayslipLines` (`Id`, `PayrollRunId`, `EmployeeId`, `EmployeeNumber`, `EmployeeName`, `DepartmentId`, `CostCentreId`, `BankName`, `BankAccount`, `KraPin`, `NssfNumber`, `ShifNumber`, `BasicSalary`, `EarningsJson`, `GrossPay`, `TaxableGross`, `AllowableDeductions`, `TaxablePay`, `GrossTax`, `PersonalRelief`, `Paye`, `NssfEmployee`, `NssfTier1`, `NssfTier2`, `NssfEmployer`, `Shif`, `AhlEmployee`, `AhlEmployer`, `OtherDeductionsJson`, `TotalDeductions`, `NetPay`, `EmployerCost`, `OvertimeHours`, `UnpaidDays`, `DaysInMonth`)

---

### STEP 08: HR Letters & Document Merge Engine

#### 1. Features & Business Rules
* **10 Standard Pre-loaded Templates:**
  1. Offer of Employment
  2. Confirmation After Probation
  3. Salary Revision / Increment
  4. Promotion Letter
  5. Inter-Company / Branch Transfer
  6. Contract Renewal
  7. Written Warning
  8. Certificate of Service (Employment Act 2007, s.51)
  9. Employment Verification Letter
  10. Acceptance of Resignation
* **Merge Fields:** Automatic replacement of `{{employee_name}}`, `{{first_name}}`, `{{employee_id}}`, `{{job_title}}`, `{{department}}`, `{{location}}`, `{{grade}}`, `{{manager}}`, `{{date_joined}}`, `{{confirmation_date}}`, `{{basic_salary}}`, `{{gross_salary}}`, `{{company_name}}`, `{{company_address}}`, `{{company_kra}}`, `{{reference}}`, `{{today}}`.
* **Human Prompt Guard:** Flags warning if unresolved bracketed placeholders (e.g. `[INSERT PERFORMANCE REASON]`) remain in the text before issuance.
* **Sequential Numbering:** Automated reference generation: `HR/{YYYY}/{NNNN}` or `{COMPANY_CODE}/{YYYY}/{NNNN}`.
* **Document Auto-Filing:** Issued letters automatically create an indexed record in the employee's personnel file (`EmployeeDocuments`).

#### 2. SQL Server Schema
* `LetterTemplates` (`Id`, `CompanyId`, `Code`, `Category`, `Name`, `SubjectTemplate`, `BodyTemplate`, `IsActive`, `CreatedAtUtc`, `UpdatedAtUtc`)
* `IssuedLetters` (`Id`, `CompanyId`, `EmployeeId`, `TemplateId`, `ReferenceNumber`, `Subject`, `BodyHtml`, `IssuedByUserId`, `IssuedAtUtc`)

---

### STEP 09: Personnel Records, Clinical Credentials & Compliance Tracker

#### 1. Features & Business Rules
* **19 Standard Document Types:**
  * Clinical Staff Expirables: Practising Licences (KMPDC, NCK, PPB, KMLTTB - 12 months), Professional Indemnity Cover (12 months), Pre-employment Medical (24 months).
  * General Staff Expirables: Certificate of Good Conduct (DCI - 24 months).
  * Permanent Records: Signed Employment Contract, KRA PIN Certificate, NSSF Card, SHIF Card, National ID/Passport, Bank Confirmation, Academic & Professional Certificates.
* **Clinical Enforcement Engine:**
  * Linked to `JobTitle.IsClinical`. Clinical positions mandate practising licences and indemnity cover.
  * System surfaces critical compliance alerts for expired clinical licences due to direct statutory/regulatory liability.
* **Expiry Tracking Buckets:**
  * `Expired`, `Expiring in 30 Days`, `Expiring in 60 Days`, `Expiring in 90 Days`, `Beyond 90 Days`.
* **Verification Lifecycle:**
  * HR uploads -> Marked verified immediately.
  * Employee self-service uploads -> Placed in `Pending Verification` queue.
* **Immutability & Retention Policy:**
  * Documents are **archived with an audited reason, never deleted**.
  * Document retention rules track mandated retention periods (e.g., 7 years post-exit for payroll/statutory records).
* **Storage Provider:** Files stored in Azure Blob Storage or S3; database stores Storage Key, SHA256 checksum, Content Type, and File Size.

#### 2. SQL Server Schema
* `DocumentTypes` (`Id`, `CompanyId`, `Code`, `Name`, `Category`, `RequiresExpiry`, `ValidityMonths`, `IsRequiredForClinicalOnly`, `RetentionYearsAfterExit`, `IsActive`)
* `EmployeeDocuments` (`Id`, `CompanyId`, `EmployeeId`, `DocumentTypeId`, `Title`, `StorageKey`, `FileName`, `ContentType`, `FileSizeBytes`, `FileChecksumSha256`, `IssueDate`, `ExpiryDate`, `Status` [PendingVerification, Verified, Rejected, Archived], `VerifiedByUserId`, `VerifiedAtUtc`, `ArchivedReason`, `ArchivedAtUtc`, `UploadedAtUtc`)

---

### STEP 10: Recruitment Pipeline & Applicant Tracking System (ATS)

#### 1. Features & Lifecycle
* **1. Requisition:**
  * Raised by department: Job title, Department, Location, Vacancies count, Justification, Budgeted Salary.
  * Sequential numbering: `{COMPANY_CODE}/REQ/{YYYY}/{NNN}`.
  * Multi-step approval chain executed per company configuration.
* **2. Vacancy:**
  * Spawned upon requisition final approval.
  * States: `Draft`, `Open`, `On Hold`, `Closed`, `Filled`, `Cancelled`.
* **3. Candidates & Pipeline:**
  * Unique application per Email + Vacancy.
  * 9-Stage Kanban: `Applied -> Screening -> Shortlisted -> Interview -> Second Interview -> Assessment -> Selected -> Offer -> Hired` (plus `Rejected`).
  * Source Attribution: Company Website, LinkedIn, Referral, Recruitment Agency, Walk-in, Job Board.
* **4. Interview Evaluations:**
  * Interview scheduling (Type, Date, Interviewer, Mode, Meeting Link).
  * Standard 5-point evaluation criteria: Technical Skills, Communication, Experience, Leadership, Culture Fit + Overall Rating (1-5) and Comments.
* **5. Offer & Contract Generation:**
  * Sequential code: `{COMPANY_CODE}/OFR/{YYYY}/{NNN}`.
  * Basic salary, benefits, start date, probation period (Employment Act s.42).
  * Routed through company offer approval workflow.
  * Status: `Draft -> Pending Approval -> Approved -> Sent -> Accepted / Rejected / Withdrawn`.
* **6. Candidate-to-Employee Conversion:**
  * Converts `Accepted` offer directly into active `Employee` record.
  * Opens initial `EmploymentHistory` entry referencing the recruitment offer.
  * Validates email uniqueness across employee table; blocks duplicate conversions.

#### 2. SQL Server Schema
* `JobRequisitions` (`Id`, `CompanyId`, `ReferenceNumber`, `JobTitleId`, `DepartmentId`, `LocationId`, `OpeningsCount`, `BudgetBasicSalary`, `TargetDate`, `Justification`, `Status` [Draft, PendingApproval, Approved, Rejected, Closed], `CurrentApprovalStep`, `CreatedByUserId`, `CreatedAtUtc`)
* `RequisitionApprovals` (`Id`, `JobRequisitionId`, `Step`, `StepName`, `Action` [Approved, Rejected], `ReviewedByUserId`, `Comments`, `ReviewedAtUtc`)
* `Vacancies` (`Id`, `CompanyId`, `RequisitionId`, `Code`, `JobTitleId`, `DepartmentId`, `LocationId`, `OpeningsCount`, `FilledCount`, `Status` [Draft, Open, OnHold, Closed, Filled, Cancelled], `OpenDate`, `CloseDate`, `CreatedAtUtc`)
* `RecruitmentSources` (`Id`, `Name`, `IsActive`)
* `Candidates` (`Id`, `CompanyId`, `VacancyId`, `SourceId`, `FirstName`, `LastName`, `Email`, `Phone`, `CurrentEmployer`, `CurrentSalary`, `ExpectedSalary`, `ResumeStorageKey`, `CurrentStage` [Applied, Screening, Shortlisted, Interview, SecondInterview, Assessment, Selected, Offer, Hired, Rejected], `RejectionReason`, `ConvertedEmployeeId`, `CreatedAtUtc`, `UpdatedAtUtc`)
* `CandidateStageHistories` (`Id`, `CandidateId`, `FromStage`, `ToStage`, `MovedByUserId`, `Notes`, `MovedAtUtc`)
* `Interviews` (`Id`, `CandidateId`, `InterviewType`, `ScheduledAtUtc`, `InterviewerUserId`, `LocationOrMeetingUrl`, `Status` [Scheduled, Completed, Cancelled])
* `InterviewEvaluations` (`Id`, `InterviewId`, `TechnicalScore`, `CommunicationScore`, `ExperienceScore`, `LeadershipScore`, `CultureFitScore`, `OverallScore`, `Comments`, `SubmittedByUserId`, `SubmittedAtUtc`)
* `Offers` (`Id`, `CompanyId`, `CandidateId`, `VacancyId`, `ReferenceNumber`, `JobTitleId`, `DepartmentId`, `LocationId`, `GradeId`, `EmploymentType`, `JoiningDate`, `BasicSalary`, `BenefitsDescription`, `ProbationMonths`, `ExpiryDate`, `Status` [Draft, PendingApproval, Approved, Sent, Accepted, Rejected, Expired, Withdrawn], `CurrentApprovalStep`, `LetterBodyHtml`, `CreatedByUserId`, `CreatedAtUtc`, `SentAtUtc`, `RespondedAtUtc`)
* `OfferApprovals` (`Id`, `OfferId`, `Step`, `StepName`, `Action` [Approved, Rejected], `ReviewedByUserId`, `Comments`, `ReviewedAtUtc`)

---

### STEP 11: Multi-Company Analytics, Liability & Executive Dashboards

#### 1. Features & KPIs
* **1. Group Dashboard (Multi-tenant consolidation):**
  * Consolidated headcount across all companies.
  * Headcount distribution by Company, Division, and Department.
  * Monthly payroll gross cost per company.
  * Active recruitment pipeline volume by company.
  * 12-month rolling headcount joiners, exits, and attrition rate.
* **2. HR Executive Dashboard (Company Context):**
  * Active headcount, today's attendance rate (% Present, % Late, % Absent).
  * Who is away today (Leave and Off duty list).
  * Critical compliance alert: Expired and 30-day expiring clinical practising licences.
  * Unactioned items: Pending leave requests, regularisations, and requisitions.
* **3. Management Dashboard:**
  * Team-specific headcount and reporting tree.
  * Monthly department absenteeism and overtime hours.
* **4. Statutory & Financial Reports:**
  * **Leave Liability Accrual Report:** Untaken accrued annual leave days valued at `(BasicSalary / 26) * AccruedDays`, aggregated by Cost Centre and Department.
  * **Leave Balances Master Register:** Comprehensive breakdown per employee per leave type.
  * **Monthly Absenteeism Register:** Heatmap / grid of attendance codes (`P`, `L`, `A`, `V`, `O`, `H`, `R`).
  * **Recruitment Efficiency Report:** Time-to-hire, offer acceptance ratio, applicant channel source conversion.

---

### STEP 12: Backup, Restore & Data Migration

#### 1. Features & Business Rules
* **JSON System Backup:**
  * Complete export containing Group, Companies, Master Data, Employees, Attendance, Leave, Payroll runs, Payslip lines, Documents metadata, Recruitment records, and Audit logs.
  * Payload serialization handles Decimal precision and ISO 8601 timestamps.
* **System Restore:**
  * Structured schema validation before import.
  * Executes within an atomic database transaction.
  * Repopulates foreign keys and triggers validation checks.
* **Audit Enforcement:** Every backup download and restore execution is stamped permanently into the immutable audit trail.

---

### STEP 13: Technical Implementation Blueprints (.NET 10 & Next.js)

#### 1. .NET 10 Core Backend Architecture

##### Repository Pattern Contract
```csharp
// Hrms.Domain/Common/IRepository.cs
namespace Hrms.Domain.Common;

public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(string id, CancellationToken ct = default);
    Task<IReadOnlyList<T>> ListAllAsync(CancellationToken ct = default);
    Task<IReadOnlyList<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken ct = default);
    Task<T> AddAsync(T entity, CancellationToken ct = default);
    Task UpdateAsync(T entity, CancellationToken ct = default);
    Task DeleteAsync(T entity, CancellationToken ct = default);
    Task<int> CountAsync(Expression<Func<T, bool>> predicate, CancellationToken ct = default);
}

// Specialized Domain Repositories
public interface IEmployeeRepository : IRepository<Employee>
{
    Task<IReadOnlyList<Employee>> GetByDepartmentAsync(string companyId, string departmentId, CancellationToken ct = default);
    Task<Employee?> GetWithDetailsAsync(string employeeId, CancellationToken ct = default);
    Task<string> GenerateNextEmployeeNumberAsync(string companyId, CancellationToken ct = default);
}

public interface IPayrollRepository : IRepository<PayrollRun>
{
    Task<PayrollRun?> GetByPeriodWithLinesAsync(string companyId, string periodMonth, CancellationToken ct = default);
    Task<bool> IsPeriodLockedAsync(string companyId, string periodMonth, CancellationToken ct = default);
}

public interface IUnitOfWork : IDisposable
{
    IEmployeeRepository Employees { get; }
    IPayrollRepository Payroll { get; }
    IRepository<LeaveRequest> LeaveRequests { get; }
    IRepository<AttendanceRecord> Attendance { get; }
    IRepository<AuditLog> AuditLogs { get; }
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}
```

##### Tenancy Query Filter & Middleware
```csharp
// In Hrms.Infrastructure/Data/ApplicationDbContext.cs
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);

    // Global Query Filter for multi-company isolation
    foreach (var entityType in modelBuilder.Model.GetEntityTypes())
    {
        if (typeof(ITenantEntity).IsAssignableFrom(entityType.ClrType))
        {
            var method = typeof(ApplicationDbContext)
                .GetMethod(nameof(ConfigureTenantFilter), BindingFlags.NonPublic | BindingFlags.Instance)!
                .MakeGenericMethod(entityType.ClrType);
            method.Invoke(this, new object[] { modelBuilder });
        }
    }
}

private void ConfigureTenantFilter<T>(ModelBuilder builder) where T : class, ITenantEntity
{
    builder.Entity<T>().HasQueryFilter(e => _currentUserService.AllowedCompanyIds.Contains(e.CompanyId));
}
```

##### API Tenancy Middleware
```csharp
// Cross-tenant protection: Resolves user claims and returns 404 for unowned tenant data
public class TenantResolutionMiddleware
{
    private readonly RequestDelegate _next;

    public TenantResolutionMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context, ICurrentUserService currentUserService)
    {
        if (context.User.Identity?.IsAuthenticated == true)
        {
            var requestedCompany = context.Request.Headers["X-Company-Id"].FirstOrDefault();
            currentUserService.SetCurrentCompanyContext(requestedCompany);
        }
        await _next(context);
    }
}
```

#### 2. Next.js Frontend Architecture (Port 4000)

##### Project Structure
```
frontend/
├── package.json               # Configured: "dev": "next dev -p 4000"
├── next.config.mjs
├── tsconfig.json
├── src/
│   ├── app/
│   │   ├── layout.tsx         # Root layout, Theme, React Query Provider
│   │   ├── (auth)/
│   │   │   └── login/page.tsx # Auth login screen with PBKDF2/JWT integration
│   │   └── (dashboard)/
│   │       ├── layout.tsx     # Shell, Sidebar navigation, TopBar company switcher
│   │       ├── dashboard/     # Executive HR Dashboard
│   │       ├── group/         # Consolidated Group Dashboard
│   │       ├── employees/     # Employee master grid, filters, details modal
│   │       ├── employees/[id]/# Profile: Details, History, Leave, Pay, Docs
│   │       ├── attendance/    # Daily grid, Monthly register, Punch clock
│   │       ├── leave/         # Applications, Balances, Calendar
│   │       ├── approvals/     # Unified Inbox (Leave, Missed punches, Requisitions)
│   │       ├── payroll/       # Run list, Engine computation, Bank file, Returns
│   │       ├── recruitment/   # Vacancies, Kanban pipeline, Evaluations, Offers
│   │       ├── documents/     # Compliance dashboard, Expiry tracker, Personnel files
│   │       └── admin/         # Roles & Permissions, Users, Organisation master
│   ├── components/
│   │   ├── ui/                # Data tables, Modals, Badges, Tabs, Form controls
│   │   ├── CompanySwitcher.tsx# Multi-company context toggle
│   │   └── PermissionGate.tsx # Client-side authorization wrapper
│   ├── lib/
│   │   ├── api.ts             # Axios/Fetch client with Auth & Company headers
│   │   └── permissions.ts     # Permission constants & role evaluation
│   └── types/                 # TypeScript DTO models matching .NET backend
```

##### Next.js Navigation Matrix & Route Mapping
| Section | Mockup Page | Next.js Route | Permission Required |
|---|---|---|---|
| **Overview** | `dashboard` | `/dashboard` | `VIEW_REPORTS` |
| | `group` | `/group` | `VIEW_GROUP_DASHBOARD` |
| | `mgmt` | `/mgmt` | `VIEW_MGMT_DASHBOARD` |
| **People** | `employees` | `/employees` | `VIEW_EMPLOYEES` |
| | `add` | `/employees/new` | `CREATE_EMPLOYEES` |
| | `import` | `/employees/import` | `IMPORT_EMPLOYEES` |
| | `me` | `/employees/me` | Logged-in Employee |
| | `profile` | `/employees/[id]` | `VIEW_EMPLOYEES` or Self |
| **Time & Leave** | `attendance` | `/attendance` | `VIEW_ATTENDANCE` / `MARK_ATTENDANCE` |
| | `leave` | `/leave` | `VIEW_LEAVE` / `APPLY_LEAVE` |
| | `approvals` | `/approvals` | `APPROVE_LEAVE` / `APPROVE_REGULARISATION` |
| **Payroll** | `payroll` | `/payroll` | `VIEW_PAYROLL` |
| | `payslips` | `/payroll/my-payslips` | Logged-in Employee |
| | `letters` | `/letters` | `VIEW_LETTERS` / `ISSUE_LETTERS` |
| **Insight** | `reports` | `/reports` | `VIEW_REPORTS` |
| **Recruitment** | `recruitment` | `/recruitment` | `VIEW_RECRUITMENT` |
| | `requisitions` | `/recruitment/requisitions` | `VIEW_RECRUITMENT` |
| | `vacancies` | `/recruitment/vacancies` | `VIEW_RECRUITMENT` |
| | `candidates` | `/recruitment/candidates` | `VIEW_CANDIDATES` |
| **Records** | `documents` | `/documents` | `VIEW_DOCUMENTS` / `VIEW_COMPLIANCE` |
| **Administration**| `company` | `/admin/company` | `MANAGE_SETTINGS` |
| | `org` | `/admin/org` | `MANAGE_DEPARTMENTS` etc. |
| | `users` | `/admin/users` | `MANAGE_USERS` |
| | `roles` | `/admin/roles` | `MANAGE_ROLES` |
| | `audit` | `/admin/audit` | `VIEW_AUDIT` |
| | `timeconfig` | `/admin/timeconfig` | `MANAGE_LEAVE_TYPES` |
| | `payconfig` | `/admin/payconfig` | `MANAGE_PAYROLL_CONFIG` |
| | `companies` | `/admin/companies` | `MANAGE_COMPANIES` |
| | `backup` | `/admin/backup` | `MANAGE_SETTINGS` |

---

## 3. Step-by-Step Implementation Roadmap

1. **Step 1: Database Setup & Migrations (SQL Server)**
   * Provision SQL Server database.
   * Run initial EF Core migration for Group, Companies, Users, Roles, and Permissions.
   * Configure composite indexes and tenant query filters.
2. **Step 2: Core Domain & Data Access Layer (.NET 10)**
   * Implement generic `IRepository<T>` and `UnitOfWork`.
   * Configure EF Core `ApplicationDbContext` with global company query filters.
   * Implement JWT Authentication, password hashing, and user lockout handling.
3. **Step 3: Organisation & Employee Service Layer**
   * Build master data controllers (Departments, Locations, Titles, Grades, Cost Centres).
   * Implement Employee CRUD, transfer logic (`EmploymentHistory`), and CSV bulk import engine.
4. **Step 4: Attendance & Leave Engines**
   * Build Shift master and Holiday calendar endpoints.
   * Implement biometric CSV ingestion and dynamic status derivation algorithm.
   * Implement Employment Act 2007 leave rules, entitlement calculators, and approval workflow.
5. **Step 5: Payroll & Statutory Computation Engine**
   * Implement Kenya statutory calculations: PAYE tax bands, NSSF Tier I & II, SHIF, AHL.
   * Connect attendance overtime and unpaid leave adjustments into payslip generation.
   * Implement payroll run locking, bank payment file export, and KRA/NSSF/SHIF statutory returns.
6. **Step 6: Documents, Letters & Compliance Tracker**
   * Implement merge-field letter generator and sequential reference numbering.
   * Build document upload to object storage (Azure Blob / S3) with expiry tracking and clinical licence alerts.
7. **Step 7: Recruitment / ATS Engine**
   * Build job requisition approval pipeline, vacancies, and 9-stage candidate kanban board.
   * Implement interview evaluations, offer generation, and candidate-to-employee conversion.
8. **Step 8: Next.js Frontend Implementation (Port 4000)**
   * Initialize Next.js 15+ App Router application with dev script listening on port 4000.
   * Implement company context switcher, RBAC permission gates, data tables, and modal workflows matching every mockup view.
9. **Step 9: Integration, Parallel Payroll Verification & Audit Locking**
   * Validate double-entry GL journal balance.
   * Reconcile net pay and tax calculations line-by-line against live test figures.
   * Verify append-only immutability of audit logs.
