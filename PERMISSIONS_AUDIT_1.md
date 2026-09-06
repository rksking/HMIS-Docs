# HIMS Comprehensive Permissions & Endpoint Authorization Audit

> **Audit Status**: Complete & 100% Implemented  
> **Total Endpoints Audited**: 295  
> **Total System Permissions**: 66 (43 Catalog Permissions + 23 Newly Implemented Permissions)  
> **Unprotected Endpoints**: 0 (Excluding 2 public authentication endpoints: login/register)  

---

## Executive Summary

| Metric | Count | Description |
| :--- | :--- | :--- |
| **Total Endpoints** | 295 | All controller action methods across the 21 backend microservices |
| **Policy-Protected Endpoints** | 291 | Gated with granular `[Authorize(Policy = "...")]` attributes |
| **Authenticated User Endpoints** | 2 | Gated with `[Authorize]` for session switching (`/switch-branch`) and caller profile (`/me`) |
| **Public Anonymous Endpoints** | 2 | Gated with explicit `[AllowAnonymous]` (`/login` and `/register`) |
| **Unprotected Endpoints** | 0 | Zero endpoints are left open or exposed without authorization |
| **Pre-existing Catalog Permissions** | 43 | Fully mapped and implemented across all corresponding controllers |
| **New Permissions Created & Implemented** | 23 | Designed and implemented to cover previously bare modules |
| **Total Permissions in System** | 66 | Registered in `SeedData.cs` and seeded into `public.permissions` |

---

## Complete Endpoints & Permissions Authorization Matrix

| Module | Code in System | Action / Intended Scope | Current Controller State | Endpoint | IsPermissionImplemented |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Accounts | `accounts.view` | View Accounts (AccountsController.Accounts) | `[Authorize(Policy = "accounts.view")]` | `GET api/accounts/accounts` | Yes |
| Accounts | `accounts.deposit` | Record Account Deposit (AccountsController.PatientDeposits) | `[Authorize(Policy = "accounts.deposit")]` | `GET api/accounts/deposits/patient/{patientId:guid}` | Yes |
| Accounts | `accounts.deposit` | Record Account Deposit (AccountsController.Deposit) | `[Authorize(Policy = "accounts.deposit")]` | `GET api/accounts/deposits/{id:guid}` | Yes |
| Accounts | `accounts.view` | View Accounts (AccountsController.Entries) | `[Authorize(Policy = "accounts.view")]` | `GET api/accounts/journal` | Yes |
| Accounts | `accounts.view` | View Accounts (AccountsController.Entry) | `[Authorize(Policy = "accounts.view")]` | `GET api/accounts/journal/{id:guid}` | Yes |
| Accounts | `accounts.view` | View Accounts (AccountsController.Ledger) | `[Authorize(Policy = "accounts.view")]` | `GET api/accounts/ledger/{accountId:guid}` | Yes |
| Accounts | `accounts.reports` | View Account Reports (AccountsController.ArAging) | `[Authorize(Policy = "accounts.reports")]` | `GET api/accounts/reports/ar-aging` | Yes |
| Accounts | `accounts.reports` | View Account Reports (AccountsController.IncomeStatement) | `[Authorize(Policy = "accounts.reports")]` | `GET api/accounts/reports/income-statement` | Yes |
| Accounts | `accounts.reports` | View Account Reports (AccountsController.TrialBalance) | `[Authorize(Policy = "accounts.reports")]` | `GET api/accounts/reports/trial-balance` | Yes |
| Accounts | `accounts.deposit` | Record Account Deposit (AccountsController.Statement) | `[Authorize(Policy = "accounts.deposit")]` | `GET api/accounts/statement/patient/{patientId:guid}` | Yes |
| Accounts | `accounts.manage` | Manage Accounts (AccountsController.CreateAccount) | `[Authorize(Policy = "accounts.manage")]` | `POST api/accounts/accounts` | Yes |
| Accounts | `accounts.deposit` | Record Account Deposit (AccountsController.CreateDeposit) | `[Authorize(Policy = "accounts.deposit")]` | `POST api/accounts/deposits` | Yes |
| Accounts | `accounts.manage` | Manage Accounts (AccountsController.CreateEntry) | `[Authorize(Policy = "accounts.manage")]` | `POST api/accounts/journal` | Yes |
| Accounts | `accounts.manage` | Manage Accounts (AccountsController.Void) | `[Authorize(Policy = "accounts.manage")]` | `POST api/accounts/journal/{id:guid}/void` | Yes |
| Accounts | `accounts.manage` | Manage Accounts (AccountsController.SyncBilling) | `[Authorize(Policy = "accounts.manage")]` | `POST api/accounts/ledger/sync-billing` | Yes |
| Accounts | `accounts.deposit` | Record Account Deposit (AccountsController.CreateRefund) | `[Authorize(Policy = "accounts.deposit")]` | `POST api/accounts/refunds` | Yes |
| Accounts | `accounts.manage` | Manage Accounts (AccountsController.UpdateAccount) | `[Authorize(Policy = "accounts.manage")]` | `PUT api/accounts/accounts/{id:guid}` | Yes |
| Admin | `users.manage` | Manage Users (AuthController.GetUsers) | `[Authorize(Policy = "users.manage")]` | `GET api/auth/users` | Yes |
| Admin | `permissions.manage` | Manage Permissions (PermissionsController.GetAll) | `[Authorize(Policy = "permissions.manage")]` | `GET api/permissions` | Yes |
| Admin | `roles.manage` | Manage Roles (RolesController.GetAll) | `[Authorize(Policy = "roles.manage")]` | `GET api/roles` | Yes |
| Admin | `users.manage` | Manage Users (AuthController.AssignRole) | `[Authorize(Policy = "users.manage")]` | `POST api/auth/assign-role` | Yes |
| Admin | `users.manage` | Manage Users (AuthController.AssignLocation) | `[Authorize(Policy = "users.manage")]` | `POST api/auth/users/assign-location` | Yes |
| Admin | `roles.manage` | Manage Roles (RolesController.Create) | `[Authorize(Policy = "roles.manage")]` | `POST api/roles` | Yes |
| Admin | `roles.manage` | Manage Roles (RolesController.UpdatePermissions) | `[Authorize(Policy = "roles.manage")]` | `PUT api/roles/permissions` | Yes |
| Appointments | `appointments.view` | View Appointments (AppointmentsController.Schedule) | `[Authorize(Policy = "appointments.view")]` | `GET api/appointments` | Yes |
| Appointments | `appointments.manage` | Manage Appointments (AppointmentsController.UpdateStatus) | `[Authorize(Policy = "appointments.manage")]` | `PUT api/appointments/{id:guid}/status` | Yes |
| Auth | `-` | Authenticated User (AuthController.Me) | `[Authorize]` | `GET api/auth/me` | Yes |
| Auth | `N/A` | Public Anonymous Access (AuthController.Login) | `[AllowAnonymous]` | `POST api/auth/login` | Yes |
| Auth | `N/A` | Public Anonymous Access (AuthController.Register) | `[AllowAnonymous]` | `POST api/auth/register` | Yes |
| Auth | `-` | Authenticated User (AuthController.SwitchBranch) | `[Authorize]` | `POST api/auth/switch-branch` | Yes |
| Billing | `billing.view` | View Billing (BillingController.GetCharges) | `[Authorize(Policy = "billing.view")]` | `GET api/billing/charges` | Yes |
| Billing | `billing.view` | View Billing (BillingController.GetAll) | `[Authorize(Policy = "billing.view")]` | `GET api/billing/invoices` | Yes |
| Billing | `billing.view` | View Billing (BillingController.GetPatient) | `[Authorize(Policy = "billing.view")]` | `GET api/billing/invoices/patient/{patientId:guid}` | Yes |
| Billing | `billing.view` | View Billing (BillingController.Get) | `[Authorize(Policy = "billing.view")]` | `GET api/billing/invoices/{id:guid}` | Yes |
| Billing | `billing.view` | View Billing (BillingController.Revenue) | `[Authorize(Policy = "billing.view")]` | `GET api/billing/reports/revenue` | Yes |
| Billing | `billing.invoice` | Create Invoices (BillingController.CreateCharge) | `[Authorize(Policy = "billing.invoice")]` | `POST api/billing/charges` | Yes |
| Billing | `billing.invoice` | Create Invoices (BillingController.Create) | `[Authorize(Policy = "billing.invoice")]` | `POST api/billing/invoices` | Yes |
| Billing | `billing.payment` | Record Payments (BillingController.Pay) | `[Authorize(Policy = "billing.payment")]` | `POST api/billing/payments` | Yes |
| Billing | `billing.payment` | Record Payments (VisitsController.MarkConsultationPaid) | `[Authorize(Policy = "billing.payment")]` | `POST api/visits/{id:guid}/consultation-payment` | Yes |
| Billing | `billing.invoice` | Create Invoices (BillingController.UpdateCharge) | `[Authorize(Policy = "billing.invoice")]` | `PUT api/billing/charges/{id:guid}` | Yes |
| Blood Bank | `bloodbank.view` | View Blood Bank (BloodBankController.GetDonors) | `[Authorize(Policy = "bloodbank.view")]` | `GET api/bloodbank/donors` | Yes |
| Blood Bank | `bloodbank.view` | View Blood Bank (BloodBankController.GetDonor) | `[Authorize(Policy = "bloodbank.view")]` | `GET api/bloodbank/donors/{id:guid}` | Yes |
| Blood Bank | `bloodbank.view` | View Blood Bank (BloodBankController.GetRequests) | `[Authorize(Policy = "bloodbank.view")]` | `GET api/bloodbank/requests` | Yes |
| Blood Bank | `bloodbank.view` | View Blood Bank (BloodBankController.PatientRequests) | `[Authorize(Policy = "bloodbank.view")]` | `GET api/bloodbank/requests/patient/{patientId:guid}` | Yes |
| Blood Bank | `bloodbank.view` | View Blood Bank (BloodBankController.GetRequest) | `[Authorize(Policy = "bloodbank.view")]` | `GET api/bloodbank/requests/{id:guid}` | Yes |
| Blood Bank | `bloodbank.view` | View Blood Bank (BloodBankController.RequestTransfusions) | `[Authorize(Policy = "bloodbank.view")]` | `GET api/bloodbank/requests/{id:guid}/transfusions` | Yes |
| Blood Bank | `bloodbank.view` | View Blood Bank (BloodBankController.GetTransfusions) | `[Authorize(Policy = "bloodbank.view")]` | `GET api/bloodbank/transfusions` | Yes |
| Blood Bank | `bloodbank.view` | View Blood Bank (BloodBankController.GetUnits) | `[Authorize(Policy = "bloodbank.view")]` | `GET api/bloodbank/units` | Yes |
| Blood Bank | `bloodbank.view` | View Blood Bank (BloodBankController.Available) | `[Authorize(Policy = "bloodbank.view")]` | `GET api/bloodbank/units/available` | Yes |
| Blood Bank | `bloodbank.view` | View Blood Bank (BloodBankController.Expiring) | `[Authorize(Policy = "bloodbank.view")]` | `GET api/bloodbank/units/expiring` | Yes |
| Blood Bank | `bloodbank.view` | View Blood Bank (BloodBankController.StockSummary) | `[Authorize(Policy = "bloodbank.view")]` | `GET api/bloodbank/units/stock-summary` | Yes |
| Blood Bank | `bloodbank.manage` | Manage Inventory/Donors/Requests (BloodBankController.CreateDonor) | `[Authorize(Policy = "bloodbank.manage")]` | `POST api/bloodbank/donors` | Yes |
| Blood Bank | `bloodbank.manage` | Manage Inventory/Donors/Requests (BloodBankController.CreateRequest) | `[Authorize(Policy = "bloodbank.manage")]` | `POST api/bloodbank/requests` | Yes |
| Blood Bank | `bloodbank.transfuse` | Start/Complete Transfusions (BloodBankController.Start) | `[Authorize(Policy = "bloodbank.transfuse")]` | `POST api/bloodbank/transfusions/start` | Yes |
| Blood Bank | `bloodbank.transfuse` | Start/Complete Transfusions (BloodBankController.Complete) | `[Authorize(Policy = "bloodbank.transfuse")]` | `POST api/bloodbank/transfusions/{id:guid}/complete` | Yes |
| Blood Bank | `bloodbank.manage` | Manage Inventory/Donors/Requests (BloodBankController.AddUnit) | `[Authorize(Policy = "bloodbank.manage")]` | `POST api/bloodbank/units` | Yes |
| Blood Bank | `bloodbank.manage` | Manage Inventory/Donors/Requests (BloodBankController.Discard) | `[Authorize(Policy = "bloodbank.manage")]` | `POST api/bloodbank/units/{id:guid}/discard` | Yes |
| Blood Bank | `bloodbank.manage` | Manage Inventory/Donors/Requests (BloodBankController.Reserve) | `[Authorize(Policy = "bloodbank.manage")]` | `POST api/bloodbank/units/{id:guid}/reserve` | Yes |
| Blood Bank | `bloodbank.manage` | Manage Inventory/Donors/Requests (BloodBankController.UpdateDonor) | `[Authorize(Policy = "bloodbank.manage")]` | `PUT api/bloodbank/donors/{id:guid}` | Yes |
| Blood Bank | `bloodbank.manage` | Manage Inventory/Donors/Requests (BloodBankController.UpdateRequestStatus) | `[Authorize(Policy = "bloodbank.manage")]` | `PUT api/bloodbank/requests/{id:guid}/status` | Yes |
| CDSS | `cdss.alert` | CDSS Alert Evaluation (CDSSController.RecentAlerts) | `[Authorize(Policy = "cdss.alert")]` | `GET api/cdss/alerts` | Yes |
| CDSS | `cdss.alert` | CDSS Alert Evaluation (CDSSController.PatientAlerts) | `[Authorize(Policy = "cdss.alert")]` | `GET api/cdss/alerts/patient/{patientId:guid}` | Yes |
| CDSS | `cdss.view` | View CDSS Insights (CDSSController.AllergyRules) | `[Authorize(Policy = "cdss.view")]` | `GET api/cdss/allergy-rules` | Yes |
| CDSS | `cdss.view` | View CDSS Insights (CDSSController.Classes) | `[Authorize(Policy = "cdss.view")]` | `GET api/cdss/classes` | Yes |
| CDSS | `cdss.view` | View CDSS Insights (CDSSController.DoseRules) | `[Authorize(Policy = "cdss.view")]` | `GET api/cdss/dose-rules` | Yes |
| CDSS | `cdss.view` | View CDSS Insights (CDSSController.Interactions) | `[Authorize(Policy = "cdss.view")]` | `GET api/cdss/drug-interactions` | Yes |
| CDSS | `cdss.view` | View CDSS Insights (CDSSController.Reminders) | `[Authorize(Policy = "cdss.view")]` | `GET api/cdss/reminders` | Yes |
| CDSS | `cdss.view` | View CDSS Insights (CDSSController.Rules) | `[Authorize(Policy = "cdss.view")]` | `GET api/cdss/rules` | Yes |
| CDSS | `cdss.view` | View CDSS Insights (CDSSController.Summary) | `[Authorize(Policy = "cdss.view")]` | `GET api/cdss/summary` | Yes |
| CDSS | `cdss.manage` | Manage CDSS Rules (CDSSController.CreateAlert) | `[Authorize(Policy = "cdss.manage")]` | `POST api/cdss/alerts` | Yes |
| CDSS | `cdss.alert` | CDSS Alert Evaluation (CDSSController.Acknowledge) | `[Authorize(Policy = "cdss.alert")]` | `POST api/cdss/alerts/acknowledge` | Yes |
| CDSS | `cdss.manage` | Manage CDSS Rules (CDSSController.CreateAllergyRule) | `[Authorize(Policy = "cdss.manage")]` | `POST api/cdss/allergy-rules` | Yes |
| CDSS | `cdss.alert` | CDSS Alert Evaluation (CDSSController.Check) | `[Authorize(Policy = "cdss.alert")]` | `POST api/cdss/check` | Yes |
| CDSS | `cdss.manage` | Manage CDSS Rules (CDSSController.CreateClass) | `[Authorize(Policy = "cdss.manage")]` | `POST api/cdss/classes` | Yes |
| CDSS | `cdss.manage` | Manage CDSS Rules (CDSSController.CreateDoseRule) | `[Authorize(Policy = "cdss.manage")]` | `POST api/cdss/dose-rules` | Yes |
| CDSS | `cdss.manage` | Manage CDSS Rules (CDSSController.CreateInteraction) | `[Authorize(Policy = "cdss.manage")]` | `POST api/cdss/drug-interactions` | Yes |
| CDSS | `cdss.alert` | CDSS Alert Evaluation (CDSSController.CheckInteractions) | `[Authorize(Policy = "cdss.alert")]` | `POST api/cdss/drug-interactions/check` | Yes |
| CDSS | `cdss.manage` | Manage CDSS Rules (CDSSController.CreateReminder) | `[Authorize(Policy = "cdss.manage")]` | `POST api/cdss/reminders` | Yes |
| CDSS | `cdss.manage` | Manage CDSS Rules (CDSSController.CreateRule) | `[Authorize(Policy = "cdss.manage")]` | `POST api/cdss/rules` | Yes |
| EMR | `emr.view` | View Records (EMRController.History) | `[Authorize(Policy = "emr.view")]` | `GET api/emr/patients/{patientId:guid}/history` | Yes |
| EMR | `emr.view` | View Records (EMRController.GetByVisit) | `[Authorize(Policy = "emr.view")]` | `GET api/emr/records/visit/{visitId:guid}` | Yes |
| EMR | `emr.view` | View Records (EMRController.GetById) | `[Authorize(Policy = "emr.view")]` | `GET api/emr/records/{id:guid}` | Yes |
| EMR | `emr.view` | View Records (TemplatesController.GetActive) | `[Authorize(Policy = "emr.view")]` | `GET api/emr/templates` | Yes |
| EMR | `emr.view` | View Records (TemplatesController.GetResponse) | `[Authorize(Policy = "emr.view")]` | `GET api/emr/templates/responses/record/{clinicalRecordId:guid}` | Yes |
| EMR | `emr.view` | View Records (TemplatesController.Get) | `[Authorize(Policy = "emr.view")]` | `GET api/emr/templates/{id:guid}` | Yes |
| EMR | `emr.edit` | Edit Records (EMRController.Create) | `[Authorize(Policy = "emr.edit")]` | `POST api/emr/records` | Yes |
| EMR | `emr.edit` | Edit Records (EMRController.AddDiagnosis) | `[Authorize(Policy = "emr.edit")]` | `POST api/emr/records/{id:guid}/diagnoses` | Yes |
| EMR | `emr.edit` | Edit Records (EMRController.AddNote) | `[Authorize(Policy = "emr.edit")]` | `POST api/emr/records/{id:guid}/notes` | Yes |
| EMR | `emr.edit` | Edit Records (EMRController.AddOrder) | `[Authorize(Policy = "emr.edit")]` | `POST api/emr/records/{id:guid}/orders` | Yes |
| EMR | `emr.edit` | Edit Records (EMRController.Sign) | `[Authorize(Policy = "emr.edit")]` | `POST api/emr/records/{id:guid}/sign` | Yes |
| EMR | `emr.edit` | Edit Records (EMRController.AddVitals) | `[Authorize(Policy = "emr.edit")]` | `POST api/emr/records/{id:guid}/vitals` | Yes |
| EMR | `emr.edit` | Edit Records (TemplatesController.Create) | `[Authorize(Policy = "emr.edit")]` | `POST api/emr/templates` | Yes |
| EMR | `emr.edit` | Edit Records (TemplatesController.SaveResponse) | `[Authorize(Policy = "emr.edit")]` | `POST api/emr/templates/responses` | Yes |
| EMR | `emr.edit` | Edit Records (EMRController.Update) | `[Authorize(Policy = "emr.edit")]` | `PUT api/emr/records/{id:guid}` | Yes |
| EMR | `emr.edit` | Edit Records (TemplatesController.Update) | `[Authorize(Policy = "emr.edit")]` | `PUT api/emr/templates/{id:guid}` | Yes |
| Emergency | `emergency.view` | View Emergency Cases (EmergencyController.Queue) | `[Authorize(Policy = "emergency.view")]` | `GET api/emergency/queue` | Yes |
| Emergency | `emergency.view` | View Emergency Cases (EmergencyController.Summary) | `[Authorize(Policy = "emergency.view")]` | `GET api/emergency/summary` | Yes |
| Emergency | `emergency.view` | View Emergency Cases (EmergencyController.Get) | `[Authorize(Policy = "emergency.view")]` | `GET api/emergency/{id:guid}` | Yes |
| Emergency | `emergency.manage` | Manage Emergency Cases (EmergencyController.QuickRegister) | `[Authorize(Policy = "emergency.manage")]` | `POST api/emergency/quick-register` | Yes |
| Emergency | `emergency.manage` | Manage Emergency Cases (EmergencyController.Register) | `[Authorize(Policy = "emergency.manage")]` | `POST api/emergency/register` | Yes |
| Emergency | `emergency.manage` | Manage Emergency Cases (EmergencyController.AssignDoctor) | `[Authorize(Policy = "emergency.manage")]` | `PUT api/emergency/{id:guid}/assign-doctor` | Yes |
| Emergency | `emergency.manage` | Manage Emergency Cases (EmergencyController.Complete) | `[Authorize(Policy = "emergency.manage")]` | `PUT api/emergency/{id:guid}/complete` | Yes |
| Emergency | `emergency.triage` | Perform Emergency Triage (EmergencyController.Triage) | `[Authorize(Policy = "emergency.triage")]` | `PUT api/emergency/{id:guid}/triage` | Yes |
| HR | `hr.view` | View HR Records (HRController.Attendance) | `[Authorize(Policy = "hr.view")]` | `GET api/hr/attendance` | Yes |
| HR | `hr.view` | View HR Records (HRController.AttendanceSummary) | `[Authorize(Policy = "hr.view")]` | `GET api/hr/attendance/summary` | Yes |
| HR | `hr.view` | View HR Records (HRController.Departments) | `[Authorize(Policy = "hr.view")]` | `GET api/hr/departments` | Yes |
| HR | `hr.view` | View HR Records (HRController.Designations) | `[Authorize(Policy = "hr.view")]` | `GET api/hr/designations` | Yes |
| HR | `hr.view` | View HR Records (HRController.List) | `[Authorize(Policy = "hr.view")]` | `GET api/hr/employees` | Yes |
| HR | `hr.view` | View HR Records (HRController.Get) | `[Authorize(Policy = "hr.view")]` | `GET api/hr/employees/{id:guid}` | Yes |
| HR | `hr.view` | View HR Records (HRController.LeaveTypes) | `[Authorize(Policy = "hr.view")]` | `GET api/hr/leave-types` | Yes |
| HR | `hr.view` | View HR Records (HRController.Leaves) | `[Authorize(Policy = "hr.view")]` | `GET api/hr/leaves` | Yes |
| HR | `hr.view` | View HR Records (HRController.Balances) | `[Authorize(Policy = "hr.view")]` | `GET api/hr/leaves/balances` | Yes |
| HR | `hr.payroll` | Process/View Payroll (HRController.EmployeePayrolls) | `[Authorize(Policy = "hr.payroll")]` | `GET api/hr/payroll/employee/{employeeId:guid}` | Yes |
| HR | `hr.payroll` | Process/View Payroll (HRController.Register) | `[Authorize(Policy = "hr.payroll")]` | `GET api/hr/payroll/register` | Yes |
| HR | `hr.payroll` | Process/View Payroll (HRController.GetPayroll) | `[Authorize(Policy = "hr.payroll")]` | `GET api/hr/payroll/{id:guid}` | Yes |
| HR | `hr.view` | View HR Records (HRController.Summary) | `[Authorize(Policy = "hr.view")]` | `GET api/hr/summary` | Yes |
| HR | `hr.manage` | Manage Employees/Attendance/Masters (HRController.RecordAttendance) | `[Authorize(Policy = "hr.manage")]` | `POST api/hr/attendance` | Yes |
| HR | `hr.manage` | Manage Employees/Attendance/Masters (HRController.CreateDepartment) | `[Authorize(Policy = "hr.manage")]` | `POST api/hr/departments` | Yes |
| HR | `hr.manage` | Manage Employees/Attendance/Masters (HRController.CreateDesignation) | `[Authorize(Policy = "hr.manage")]` | `POST api/hr/designations` | Yes |
| HR | `hr.manage` | Manage Employees/Attendance/Masters (HRController.Create) | `[Authorize(Policy = "hr.manage")]` | `POST api/hr/employees` | Yes |
| HR | `hr.manage` | Manage Employees/Attendance/Masters (HRController.CreateLeaveType) | `[Authorize(Policy = "hr.manage")]` | `POST api/hr/leave-types` | Yes |
| HR | `hr.manage` | Manage Employees/Attendance/Masters (HRController.ApplyLeave) | `[Authorize(Policy = "hr.manage")]` | `POST api/hr/leaves` | Yes |
| HR | `hr.payroll` | Process/View Payroll (HRController.Payroll) | `[Authorize(Policy = "hr.payroll")]` | `POST api/hr/payroll/process` | Yes |
| HR | `hr.payroll` | Process/View Payroll (HRController.MarkPaid) | `[Authorize(Policy = "hr.payroll")]` | `POST api/hr/payroll/{id:guid}/pay` | Yes |
| HR | `hr.manage` | Manage Employees/Attendance/Masters (HRController.Update) | `[Authorize(Policy = "hr.manage")]` | `PUT api/hr/employees/{id:guid}` | Yes |
| HR | `hr.leave.approve` | Approve Leave Requests (HRController.Approve) | `[Authorize(Policy = "hr.leave.approve")]` | `PUT api/hr/leaves/approve` | Yes |
| ICU | `icu.view` | View ICU Admissions (ICUController.Active) | `[Authorize(Policy = "icu.view")]` | `GET api/icu/admissions/active` | Yes |
| ICU | `icu.view` | View ICU Admissions (ICUController.Get) | `[Authorize(Policy = "icu.view")]` | `GET api/icu/admissions/{id:guid}` | Yes |
| ICU | `icu.admit` | Admit/Discharge ICU Patients (ICUController.Admit) | `[Authorize(Policy = "icu.admit")]` | `POST api/icu/admissions` | Yes |
| ICU | `icu.admit` | Admit/Discharge ICU Patients (ICUController.Discharge) | `[Authorize(Policy = "icu.admit")]` | `POST api/icu/admissions/{id:guid}/discharge` | Yes |
| ICU | `icu.chart` | Record Vitals/Meds/IO/Notes/Procedures (ICUController.RecordIO) | `[Authorize(Policy = "icu.chart")]` | `POST api/icu/admissions/{id:guid}/intake-output` | Yes |
| ICU | `icu.chart` | Record Vitals/Meds/IO/Notes/Procedures (ICUController.RecordMedication) | `[Authorize(Policy = "icu.chart")]` | `POST api/icu/admissions/{id:guid}/medications` | Yes |
| ICU | `icu.chart` | Record Vitals/Meds/IO/Notes/Procedures (ICUController.AddNote) | `[Authorize(Policy = "icu.chart")]` | `POST api/icu/admissions/{id:guid}/notes` | Yes |
| ICU | `icu.chart` | Record Vitals/Meds/IO/Notes/Procedures (ICUController.RecordProcedure) | `[Authorize(Policy = "icu.chart")]` | `POST api/icu/admissions/{id:guid}/procedures` | Yes |
| ICU | `icu.chart` | Record Vitals/Meds/IO/Notes/Procedures (ICUController.Vitals) | `[Authorize(Policy = "icu.chart")]` | `POST api/icu/admissions/{id:guid}/vitals` | Yes |
| Inventory | `inventory.assets.view` | View Assets (AssetController.List) | `[Authorize(Policy = "inventory.assets.view")]` | `GET api/inventory/assets` | Yes |
| Inventory | `inventory.assets.view` | View Assets (AssetController.Categories) | `[Authorize(Policy = "inventory.assets.view")]` | `GET api/inventory/assets/categories` | Yes |
| Inventory | `inventory.assets.view` | View Assets (AssetController.Search) | `[Authorize(Policy = "inventory.assets.view")]` | `GET api/inventory/assets/search` | Yes |
| Inventory | `inventory.assets.view` | View Assets (AssetController.ServiceDue) | `[Authorize(Policy = "inventory.assets.view")]` | `GET api/inventory/assets/service-due` | Yes |
| Inventory | `inventory.assets.view` | View Assets (AssetController.Summary) | `[Authorize(Policy = "inventory.assets.view")]` | `GET api/inventory/assets/summary` | Yes |
| Inventory | `inventory.assets.view` | View Assets (AssetController.Get) | `[Authorize(Policy = "inventory.assets.view")]` | `GET api/inventory/assets/{id:guid}` | Yes |
| Inventory | `inventory.assets.view` | View Assets (AssetController.MaintenanceLogs) | `[Authorize(Policy = "inventory.assets.view")]` | `GET api/inventory/assets/{id:guid}/maintenance` | Yes |
| Inventory | `inventory.stock.view` | View Stock/Items (InventoryController.Categories) | `[Authorize(Policy = "inventory.stock.view")]` | `GET api/inventory/categories` | Yes |
| Inventory | `inventory.procurement.view` | View Suppliers/POs (ProcurementController.Receipts) | `[Authorize(Policy = "inventory.procurement.view")]` | `GET api/inventory/goods-receipts` | Yes |
| Inventory | `inventory.stock.view` | View Stock/Items (InventoryController.List) | `[Authorize(Policy = "inventory.stock.view")]` | `GET api/inventory/items` | Yes |
| Inventory | `inventory.stock.view` | View Stock/Items (InventoryController.Search) | `[Authorize(Policy = "inventory.stock.view")]` | `GET api/inventory/items/search` | Yes |
| Inventory | `inventory.stock.view` | View Stock/Items (InventoryController.Get) | `[Authorize(Policy = "inventory.stock.view")]` | `GET api/inventory/items/{id:guid}` | Yes |
| Inventory | `inventory.stock.view` | View Stock/Items (InventoryController.ItemTx) | `[Authorize(Policy = "inventory.stock.view")]` | `GET api/inventory/items/{id:guid}/transactions` | Yes |
| Inventory | `inventory.procurement.view` | View Suppliers/POs (ProcurementController.POs) | `[Authorize(Policy = "inventory.procurement.view")]` | `GET api/inventory/purchase-orders` | Yes |
| Inventory | `inventory.procurement.view` | View Suppliers/POs (ProcurementController.PO) | `[Authorize(Policy = "inventory.procurement.view")]` | `GET api/inventory/purchase-orders/{id:guid}` | Yes |
| Inventory | `inventory.stock.view` | View Stock/Items (InventoryController.Expiring) | `[Authorize(Policy = "inventory.stock.view")]` | `GET api/inventory/stock/expiring` | Yes |
| Inventory | `inventory.stock.view` | View Stock/Items (InventoryController.Summary) | `[Authorize(Policy = "inventory.stock.view")]` | `GET api/inventory/summary` | Yes |
| Inventory | `inventory.procurement.view` | View Suppliers/POs (ProcurementController.Suppliers) | `[Authorize(Policy = "inventory.procurement.view")]` | `GET api/inventory/suppliers` | Yes |
| Inventory | `inventory.procurement.view` | View Suppliers/POs (ProcurementController.Supplier) | `[Authorize(Policy = "inventory.procurement.view")]` | `GET api/inventory/suppliers/{id:guid}` | Yes |
| Inventory | `inventory.stock.view` | View Stock/Items (InventoryController.Units) | `[Authorize(Policy = "inventory.stock.view")]` | `GET api/inventory/units` | Yes |
| Inventory | `inventory.assets.manage` | Manage Assets/Maintenance (AssetController.Create) | `[Authorize(Policy = "inventory.assets.manage")]` | `POST api/inventory/assets` | Yes |
| Inventory | `inventory.assets.manage` | Manage Assets/Maintenance (AssetController.CreateCategory) | `[Authorize(Policy = "inventory.assets.manage")]` | `POST api/inventory/assets/categories` | Yes |
| Inventory | `inventory.assets.manage` | Manage Assets/Maintenance (AssetController.LogMaintenance) | `[Authorize(Policy = "inventory.assets.manage")]` | `POST api/inventory/assets/{id:guid}/maintenance` | Yes |
| Inventory | `inventory.assets.manage` | Manage Assets/Maintenance (AssetController.ChangeStatus) | `[Authorize(Policy = "inventory.assets.manage")]` | `POST api/inventory/assets/{id:guid}/status` | Yes |
| Inventory | `inventory.stock.manage` | Manage Stock/Items (InventoryController.CreateCategory) | `[Authorize(Policy = "inventory.stock.manage")]` | `POST api/inventory/categories` | Yes |
| Inventory | `inventory.stock.manage` | Manage Stock/Items (InventoryController.Create) | `[Authorize(Policy = "inventory.stock.manage")]` | `POST api/inventory/items` | Yes |
| Inventory | `inventory.procurement.manage` | Create/Receive POs (ProcurementController.CreatePO) | `[Authorize(Policy = "inventory.procurement.manage")]` | `POST api/inventory/purchase-orders` | Yes |
| Inventory | `inventory.procurement.approve` | Approve Purchase Orders (ProcurementController.Approve) | `[Authorize(Policy = "inventory.procurement.approve")]` | `POST api/inventory/purchase-orders/{id:guid}/approve` | Yes |
| Inventory | `inventory.procurement.manage` | Create/Receive POs (ProcurementController.Cancel) | `[Authorize(Policy = "inventory.procurement.manage")]` | `POST api/inventory/purchase-orders/{id:guid}/cancel` | Yes |
| Inventory | `inventory.procurement.manage` | Create/Receive POs (ProcurementController.Close) | `[Authorize(Policy = "inventory.procurement.manage")]` | `POST api/inventory/purchase-orders/{id:guid}/close` | Yes |
| Inventory | `inventory.procurement.manage` | Create/Receive POs (ProcurementController.Order) | `[Authorize(Policy = "inventory.procurement.manage")]` | `POST api/inventory/purchase-orders/{id:guid}/order` | Yes |
| Inventory | `inventory.procurement.manage` | Create/Receive POs (ProcurementController.Receive) | `[Authorize(Policy = "inventory.procurement.manage")]` | `POST api/inventory/purchase-orders/{id:guid}/receive` | Yes |
| Inventory | `inventory.stock.manage` | Manage Stock/Items (InventoryController.Adjust) | `[Authorize(Policy = "inventory.stock.manage")]` | `POST api/inventory/stock/adjust` | Yes |
| Inventory | `inventory.procurement.manage` | Create/Receive POs (ProcurementController.CreateSupplier) | `[Authorize(Policy = "inventory.procurement.manage")]` | `POST api/inventory/suppliers` | Yes |
| Inventory | `inventory.stock.manage` | Manage Stock/Items (InventoryController.CreateUnit) | `[Authorize(Policy = "inventory.stock.manage")]` | `POST api/inventory/units` | Yes |
| Inventory | `inventory.assets.manage` | Manage Assets/Maintenance (AssetController.Update) | `[Authorize(Policy = "inventory.assets.manage")]` | `PUT api/inventory/assets/{id:guid}` | Yes |
| Inventory | `inventory.stock.manage` | Manage Stock/Items (InventoryController.Update) | `[Authorize(Policy = "inventory.stock.manage")]` | `PUT api/inventory/items/{id:guid}` | Yes |
| Inventory | `inventory.procurement.manage` | Create/Receive POs (ProcurementController.UpdateSupplier) | `[Authorize(Policy = "inventory.procurement.manage")]` | `PUT api/inventory/suppliers/{id:guid}` | Yes |
| Laboratory | `lab.view` | View Lab (LaboratoryController.PatientOrders) | `[Authorize(Policy = "lab.view")]` | `GET api/laboratory/orders/patient/{patientId:guid}` | Yes |
| Laboratory | `lab.view` | View Lab (LaboratoryController.ByStatus) | `[Authorize(Policy = "lab.view")]` | `GET api/laboratory/orders/status/{status}` | Yes |
| Laboratory | `lab.view` | View Lab (LaboratoryController.GetOrder) | `[Authorize(Policy = "lab.view")]` | `GET api/laboratory/orders/{id:guid}` | Yes |
| Laboratory | `lab.view` | View Lab (LaboratoryController.GetTests) | `[Authorize(Policy = "lab.view")]` | `GET api/laboratory/tests` | Yes |
| Laboratory | `lab.view` | View Lab (LaboratoryController.GetTest) | `[Authorize(Policy = "lab.view")]` | `GET api/laboratory/tests/{id:guid}` | Yes |
| Laboratory | `lab.order` | Order Tests (LaboratoryController.CreateOrder) | `[Authorize(Policy = "lab.order")]` | `POST api/laboratory/orders` | Yes |
| Laboratory | `lab.result` | Enter Results (LaboratoryController.GenerateReport) | `[Authorize(Policy = "lab.result")]` | `POST api/laboratory/orders/{id:guid}/report` | Yes |
| Laboratory | `lab.result` | Enter Results (LaboratoryController.EnterResults) | `[Authorize(Policy = "lab.result")]` | `POST api/laboratory/results` | Yes |
| Laboratory | `lab.result` | Enter Results (LaboratoryController.Amend) | `[Authorize(Policy = "lab.result")]` | `POST api/laboratory/results/amend` | Yes |
| Laboratory | `lab.order` | Order Tests (LaboratoryController.CreateTest) | `[Authorize(Policy = "lab.order")]` | `POST api/laboratory/tests` | Yes |
| Laboratory | `lab.order` | Order Tests (LaboratoryController.Collect) | `[Authorize(Policy = "lab.order")]` | `PUT api/laboratory/orders/{id:guid}/collect` | Yes |
| Laboratory | `lab.order` | Order Tests (LaboratoryController.UpdateTest) | `[Authorize(Policy = "lab.order")]` | `PUT api/laboratory/tests/{id:guid}` | Yes |
| Master | `master.view` | View Master Data (MasterController.GetBeds) | `[Authorize(Policy = "master.view")]` | `GET api/master/beds` | Yes |
| Master | `master.view` | View Master Data (MasterController.GetDepts) | `[Authorize(Policy = "master.view")]` | `GET api/master/departments` | Yes |
| Master | `master.view` | View Master Data (MasterController.GetDesig) | `[Authorize(Policy = "master.view")]` | `GET api/master/designations` | Yes |
| Master | `master.view` | View Master Data (MasterController.GetDoctors) | `[Authorize(Policy = "master.view")]` | `GET api/master/doctors` | Yes |
| Master | `master.view` | View Master Data (MasterController.GetDrugCats) | `[Authorize(Policy = "master.view")]` | `GET api/master/drug-categories` | Yes |
| Master | `master.view` | View Master Data (MasterController.GetFeeTypes) | `[Authorize(Policy = "master.view")]` | `GET api/master/fee-types` | Yes |
| Master | `master.view` | View Master Data (MasterController.SearchICD) | `[Authorize(Policy = "master.view")]` | `GET api/master/icd-codes/search` | Yes |
| Master | `master.view` | View Master Data (MasterController.GetSchemes) | `[Authorize(Policy = "master.view")]` | `GET api/master/insurance-schemes` | Yes |
| Master | `master.view` | View Master Data (MasterController.GetLocations) | `[Authorize(Policy = "master.view")]` | `GET api/master/locations` | Yes |
| Master | `master.view` | View Master Data (MasterController.GetLocation) | `[Authorize(Policy = "master.view")]` | `GET api/master/locations/{id:guid}` | Yes |
| Master | `master.view` | View Master Data (MasterController.BillTypes) | `[Authorize(Policy = "master.view")]` | `GET api/master/lookups/bill-types` | Yes |
| Master | `master.view` | View Master Data (MasterController.BloodGroups) | `[Authorize(Policy = "master.view")]` | `GET api/master/lookups/blood-groups` | Yes |
| Master | `master.view` | View Master Data (MasterController.Genders) | `[Authorize(Policy = "master.view")]` | `GET api/master/lookups/genders` | Yes |
| Master | `master.view` | View Master Data (MasterController.LeaveTypes) | `[Authorize(Policy = "master.view")]` | `GET api/master/lookups/leave-types` | Yes |
| Master | `master.view` | View Master Data (MasterController.PaymentMethods) | `[Authorize(Policy = "master.view")]` | `GET api/master/lookups/payment-methods` | Yes |
| Master | `master.view` | View Master Data (MasterController.VisitSubTypes) | `[Authorize(Policy = "master.view")]` | `GET api/master/lookups/visit-sub-types` | Yes |
| Master | `master.view` | View Master Data (MasterController.VisitTypes) | `[Authorize(Policy = "master.view")]` | `GET api/master/lookups/visit-types` | Yes |
| Master | `master.view` | View Master Data (MasterController.GetTestCats) | `[Authorize(Policy = "master.view")]` | `GET api/master/test-categories` | Yes |
| Master | `master.manage` | Manage Master Data (MasterController.CreateBed) | `[Authorize(Policy = "master.manage")]` | `POST api/master/beds` | Yes |
| Master | `master.manage` | Manage Master Data (MasterController.CreateDept) | `[Authorize(Policy = "master.manage")]` | `POST api/master/departments` | Yes |
| Master | `master.manage` | Manage Master Data (MasterController.CreateDesig) | `[Authorize(Policy = "master.manage")]` | `POST api/master/designations` | Yes |
| Master | `master.manage` | Manage Master Data (MasterController.CreateDoctor) | `[Authorize(Policy = "master.manage")]` | `POST api/master/doctors` | Yes |
| Master | `master.manage` | Manage Master Data (MasterController.CreateDrugCat) | `[Authorize(Policy = "master.manage")]` | `POST api/master/drug-categories` | Yes |
| Master | `master.manage` | Manage Master Data (MasterController.CreateFeeType) | `[Authorize(Policy = "master.manage")]` | `POST api/master/fee-types` | Yes |
| Master | `master.manage` | Manage Master Data (MasterController.CreateICD) | `[Authorize(Policy = "master.manage")]` | `POST api/master/icd-codes` | Yes |
| Master | `master.manage` | Manage Master Data (MasterController.CreateScheme) | `[Authorize(Policy = "master.manage")]` | `POST api/master/insurance-schemes` | Yes |
| Master | `master.manage` | Manage Master Data (MasterController.CreateLocation) | `[Authorize(Policy = "master.manage")]` | `POST api/master/locations` | Yes |
| Master | `master.manage` | Manage Master Data (MasterController.CreateTestCat) | `[Authorize(Policy = "master.manage")]` | `POST api/master/test-categories` | Yes |
| Master | `master.manage` | Manage Master Data (MasterController.UpdateBed) | `[Authorize(Policy = "master.manage")]` | `PUT api/master/beds/{id:guid}/status` | Yes |
| Master | `master.manage` | Manage Master Data (MasterController.UpdateDoctor) | `[Authorize(Policy = "master.manage")]` | `PUT api/master/doctors/{id:guid}` | Yes |
| Master | `master.manage` | Manage Master Data (MasterController.UpdateScheme) | `[Authorize(Policy = "master.manage")]` | `PUT api/master/insurance-schemes/{id:guid}` | Yes |
| Master | `master.manage` | Manage Master Data (MasterController.UpdateLocation) | `[Authorize(Policy = "master.manage")]` | `PUT api/master/locations/{id:guid}` | Yes |
| Operation Theatre | `ot.view` | View OT Schedule/Records (OTController.Procedures) | `[Authorize(Policy = "ot.view")]` | `GET api/ot/procedures` | Yes |
| Operation Theatre | `ot.view` | View OT Schedule/Records (OTController.List) | `[Authorize(Policy = "ot.view")]` | `GET api/ot/surgeries` | Yes |
| Operation Theatre | `ot.view` | View OT Schedule/Records (OTController.Patient) | `[Authorize(Policy = "ot.view")]` | `GET api/ot/surgeries/patient/{patientId:guid}` | Yes |
| Operation Theatre | `ot.view` | View OT Schedule/Records (OTController.Today) | `[Authorize(Policy = "ot.view")]` | `GET api/ot/surgeries/today` | Yes |
| Operation Theatre | `ot.view` | View OT Schedule/Records (OTController.Get) | `[Authorize(Policy = "ot.view")]` | `GET api/ot/surgeries/{id:guid}` | Yes |
| Operation Theatre | `ot.view` | View OT Schedule/Records (OTController.Theatres) | `[Authorize(Policy = "ot.view")]` | `GET api/ot/theatres` | Yes |
| Operation Theatre | `ot.manage` | Manage Theatres/Procedures (OTController.CreateProcedure) | `[Authorize(Policy = "ot.manage")]` | `POST api/ot/procedures` | Yes |
| Operation Theatre | `ot.schedule` | Book Surgery (OTController.Schedule) | `[Authorize(Policy = "ot.schedule")]` | `POST api/ot/surgeries` | Yes |
| Operation Theatre | `ot.perform` | Checklist/Notes/Close Case (OTController.Checklist) | `[Authorize(Policy = "ot.perform")]` | `POST api/ot/surgeries/{id:guid}/checklist` | Yes |
| Operation Theatre | `ot.perform` | Checklist/Notes/Close Case (OTController.Close) | `[Authorize(Policy = "ot.perform")]` | `POST api/ot/surgeries/{id:guid}/close` | Yes |
| Operation Theatre | `ot.perform` | Checklist/Notes/Close Case (OTController.Notes) | `[Authorize(Policy = "ot.perform")]` | `POST api/ot/surgeries/{id:guid}/operative-notes` | Yes |
| Operation Theatre | `ot.perform` | Checklist/Notes/Close Case (OTController.PostOp) | `[Authorize(Policy = "ot.perform")]` | `POST api/ot/surgeries/{id:guid}/post-op` | Yes |
| Operation Theatre | `ot.manage` | Manage Theatres/Procedures (OTController.CreateTheatre) | `[Authorize(Policy = "ot.manage")]` | `POST api/ot/theatres` | Yes |
| Operation Theatre | `ot.perform` | Checklist/Notes/Close Case (OTController.Status) | `[Authorize(Policy = "ot.perform")]` | `PUT api/ot/surgeries/{id:guid}/status` | Yes |
| Patients | `patients.delete` | Delete Patients (PatientsController.Delete) | `[Authorize(Policy = "patients.delete")]` | `DELETE api/patients/{id:guid}` | Yes |
| Patients | `patients.view` | View Patients (PatientsController.GetAll) | `[Authorize(Policy = "patients.view")]` | `GET api/patients` | Yes |
| Patients | `patients.view` | View Patients (PatientsController.GetByMRN) | `[Authorize(Policy = "patients.view")]` | `GET api/patients/mrn/{mrn}` | Yes |
| Patients | `patients.view` | View Patients (PatientsController.GetById) | `[Authorize(Policy = "patients.view")]` | `GET api/patients/{id:guid}` | Yes |
| Patients | `patients.view` | View Patients (PatientsController.GetAppointments) | `[Authorize(Policy = "patients.view")]` | `GET api/patients/{patientId:guid}/appointments` | Yes |
| Patients | `patients.create` | Create Patients (PatientsController.Create) | `[Authorize(Policy = "patients.create")]` | `POST api/patients` | Yes |
| Patients | `patients.create` | Create Patients (PatientsController.CreateAppointment) | `[Authorize(Policy = "patients.create")]` | `POST api/patients/appointments` | Yes |
| Patients | `patients.edit` | Edit Patients (PatientsController.Update) | `[Authorize(Policy = "patients.edit")]` | `PUT api/patients/{id:guid}` | Yes |
| Pharmacy | `pharmacy.view` | View Pharmacy (PharmacyController.GetDrugs) | `[Authorize(Policy = "pharmacy.view")]` | `GET api/pharmacy/drugs` | Yes |
| Pharmacy | `pharmacy.view` | View Pharmacy (PharmacyController.LowStock) | `[Authorize(Policy = "pharmacy.view")]` | `GET api/pharmacy/drugs/low-stock` | Yes |
| Pharmacy | `pharmacy.view` | View Pharmacy (PharmacyController.NearExpiry) | `[Authorize(Policy = "pharmacy.view")]` | `GET api/pharmacy/drugs/near-expiry` | Yes |
| Pharmacy | `pharmacy.view` | View Pharmacy (PharmacyController.GetDrugStock) | `[Authorize(Policy = "pharmacy.view")]` | `GET api/pharmacy/drugs/{drugId:guid}/stock` | Yes |
| Pharmacy | `pharmacy.view` | View Pharmacy (PharmacyController.GetAllPrescriptions) | `[Authorize(Policy = "pharmacy.view")]` | `GET api/pharmacy/prescriptions` | Yes |
| Pharmacy | `pharmacy.view` | View Pharmacy (PharmacyController.GetPatientPrescriptions) | `[Authorize(Policy = "pharmacy.view")]` | `GET api/pharmacy/prescriptions/patient/{patientId:guid}` | Yes |
| Pharmacy | `pharmacy.view` | View Pharmacy (PharmacyController.GetPrescription) | `[Authorize(Policy = "pharmacy.view")]` | `GET api/pharmacy/prescriptions/{id:guid}` | Yes |
| Pharmacy | `pharmacy.dispense` | Dispense Drugs (PharmacyController.Dispense) | `[Authorize(Policy = "pharmacy.dispense")]` | `POST api/pharmacy/dispense` | Yes |
| Pharmacy | `pharmacy.manage` | Manage Stock (PharmacyController.CreateDrug) | `[Authorize(Policy = "pharmacy.manage")]` | `POST api/pharmacy/drugs` | Yes |
| Pharmacy | `pharmacy.manage` | Manage Stock (PharmacyController.AddStock) | `[Authorize(Policy = "pharmacy.manage")]` | `POST api/pharmacy/drugs/stock` | Yes |
| Pharmacy | `pharmacy.manage` | Manage Stock (PharmacyController.CreatePrescription) | `[Authorize(Policy = "pharmacy.manage")]` | `POST api/pharmacy/prescriptions` | Yes |
| Radiology | `radiology.view` | View Radiology (RadiologyController.GetModalities) | `[Authorize(Policy = "radiology.view")]` | `GET api/radiology/modalities` | Yes |
| Radiology | `radiology.view` | View Radiology (RadiologyController.Patient) | `[Authorize(Policy = "radiology.view")]` | `GET api/radiology/studies/patient/{patientId:guid}` | Yes |
| Radiology | `radiology.view` | View Radiology (RadiologyController.Pending) | `[Authorize(Policy = "radiology.view")]` | `GET api/radiology/studies/pending` | Yes |
| Radiology | `radiology.view` | View Radiology (RadiologyController.Get) | `[Authorize(Policy = "radiology.view")]` | `GET api/radiology/studies/{id:guid}` | Yes |
| Radiology | `radiology.view` | View Radiology (RadiologyController.GetStudyTypes) | `[Authorize(Policy = "radiology.view")]` | `GET api/radiology/study-types` | Yes |
| Radiology | `radiology.view` | View Radiology (RadiologyController.GetStudyType) | `[Authorize(Policy = "radiology.view")]` | `GET api/radiology/study-types/{id:guid}` | Yes |
| Radiology | `radiology.manage` | Manage Radiology Orders (RadiologyController.CreateModality) | `[Authorize(Policy = "radiology.manage")]` | `POST api/radiology/modalities` | Yes |
| Radiology | `radiology.order` | Order Radiology Examination (RadiologyController.Order) | `[Authorize(Policy = "radiology.order")]` | `POST api/radiology/studies` | Yes |
| Radiology | `radiology.report` | Create/Submit Radiology Report (RadiologyController.Report) | `[Authorize(Policy = "radiology.report")]` | `POST api/radiology/studies/{id:guid}/report` | Yes |
| Radiology | `radiology.manage` | Manage Radiology Orders (RadiologyController.CreateStudyType) | `[Authorize(Policy = "radiology.manage")]` | `POST api/radiology/study-types` | Yes |
| Radiology | `radiology.report` | Create/Submit Radiology Report (RadiologyController.Perform) | `[Authorize(Policy = "radiology.report")]` | `PUT api/radiology/studies/{id:guid}/perform` | Yes |
| Radiology | `radiology.report` | Create/Submit Radiology Report (RadiologyController.Schedule) | `[Authorize(Policy = "radiology.report")]` | `PUT api/radiology/studies/{id:guid}/schedule` | Yes |
| Radiology | `radiology.manage` | Manage Radiology Orders (RadiologyController.UpdateStudyType) | `[Authorize(Policy = "radiology.manage")]` | `PUT api/radiology/study-types/{id:guid}` | Yes |
| Referrals | `referrals.view` | View Referrals (ReferralsController.Inbox) | `[Authorize(Policy = "referrals.view")]` | `GET api/referrals/inbox` | Yes |
| Referrals | `referrals.view` | View Referrals (ReferralsController.Get) | `[Authorize(Policy = "referrals.view")]` | `GET api/referrals/{id:guid}` | Yes |
| Referrals | `referrals.manage` | Create/Respond Referrals (ReferralsController.Create) | `[Authorize(Policy = "referrals.manage")]` | `POST api/referrals` | Yes |
| Referrals | `referrals.manage` | Create/Respond Referrals (ReferralsController.Accept) | `[Authorize(Policy = "referrals.manage")]` | `POST api/referrals/{id:guid}/accept` | Yes |
| Referrals | `referrals.manage` | Create/Respond Referrals (ReferralsController.Decline) | `[Authorize(Policy = "referrals.manage")]` | `POST api/referrals/{id:guid}/decline` | Yes |
| Reports | `reports.view` | View Reports (ReportingController.Dashboard) | `[Authorize(Policy = "reports.view")]` | `GET api/reporting/dashboard` | Yes |
| Reports | `reports.manage` | Manage Reports (ReportingController.List) | `[Authorize(Policy = "reports.manage")]` | `GET api/reporting/definitions` | Yes |
| Reports | `reports.view` | View Reports (ReportingController.Financial) | `[Authorize(Policy = "reports.view")]` | `GET api/reporting/financial` | Yes |
| Reports | `reports.view` | View Reports (ReportingController.History) | `[Authorize(Policy = "reports.view")]` | `GET api/reporting/history` | Yes |
| Reports | `reports.view` | View Reports (ReportingController.HR) | `[Authorize(Policy = "reports.view")]` | `GET api/reporting/hr` | Yes |
| Reports | `reports.view` | View Reports (ReportingController.Operational) | `[Authorize(Policy = "reports.view")]` | `GET api/reporting/operational` | Yes |
| Reports | `reports.manage` | Manage Reports (ReportingController.Create) | `[Authorize(Policy = "reports.manage")]` | `POST api/reporting/definitions` | Yes |
| Reports | `reports.manage` | Manage Reports (ReportingController.Run) | `[Authorize(Policy = "reports.manage")]` | `POST api/reporting/run` | Yes |
| Special Clinics | `specialclinics.manage` | Manage Special Clinics (SpecialClinicsController.RemoveDoctor) | `[Authorize(Policy = "specialclinics.manage")]` | `DELETE api/special-clinics/doctors/{assignmentId:guid}` | Yes |
| Special Clinics | `specialclinics.view` | View Special Clinics (SpecialClinicsController.GetActive) | `[Authorize(Policy = "specialclinics.view")]` | `GET api/special-clinics` | Yes |
| Special Clinics | `specialclinics.view` | View Special Clinics (SpecialClinicsController.Get) | `[Authorize(Policy = "specialclinics.view")]` | `GET api/special-clinics/{id:guid}` | Yes |
| Special Clinics | `specialclinics.view` | View Special Clinics (SpecialClinicsController.Bookings) | `[Authorize(Policy = "specialclinics.view")]` | `GET api/special-clinics/{id:guid}/bookings` | Yes |
| Special Clinics | `specialclinics.view` | View Special Clinics (SpecialClinicsController.Slots) | `[Authorize(Policy = "specialclinics.view")]` | `GET api/special-clinics/{id:guid}/slots` | Yes |
| Special Clinics | `specialclinics.manage` | Manage Special Clinics (SpecialClinicsController.Create) | `[Authorize(Policy = "specialclinics.manage")]` | `POST api/special-clinics` | Yes |
| Special Clinics | `specialclinics.book` | Book Special Clinic Appointment (SpecialClinicsController.Book) | `[Authorize(Policy = "specialclinics.book")]` | `POST api/special-clinics/bookings` | Yes |
| Special Clinics | `specialclinics.manage` | Manage Special Clinics (SpecialClinicsController.AssignDoctor) | `[Authorize(Policy = "specialclinics.manage")]` | `POST api/special-clinics/{id:guid}/doctors` | Yes |
| Special Clinics | `specialclinics.book` | Book Special Clinic Appointment (SpecialClinicsController.UpdateBookingStatus) | `[Authorize(Policy = "specialclinics.book")]` | `PUT api/special-clinics/bookings/{id:guid}/status` | Yes |
| Special Clinics | `specialclinics.manage` | Manage Special Clinics (SpecialClinicsController.Update) | `[Authorize(Policy = "specialclinics.manage")]` | `PUT api/special-clinics/{id:guid}` | Yes |
| Visits | `visits.view` | View Visits (VisitsController.GetAll) | `[Authorize(Policy = "visits.view")]` | `GET api/visits` | Yes |
| Visits | `visits.view` | View Visits (VisitsController.Counts) | `[Authorize(Policy = "visits.view")]` | `GET api/visits/counts` | Yes |
| Visits | `visits.view` | View Visits (VisitsController.ActiveByPatient) | `[Authorize(Policy = "visits.view")]` | `GET api/visits/patient/{patientId:guid}/active` | Yes |
| Visits | `visits.view` | View Visits (VisitsController.ByStage) | `[Authorize(Policy = "visits.view")]` | `GET api/visits/stage/{stage}` | Yes |
| Visits | `visits.view` | View Visits (VisitsController.Get) | `[Authorize(Policy = "visits.view")]` | `GET api/visits/{id:guid}` | Yes |
| Visits | `visits.manage` | Create/Advance Visits (VisitsController.Create) | `[Authorize(Policy = "visits.manage")]` | `POST api/visits` | Yes |
| Visits | `visits.manage` | Create/Advance Visits (VisitsController.Advance) | `[Authorize(Policy = "visits.manage")]` | `POST api/visits/{id:guid}/advance` | Yes |
| Visits | `visits.manage` | Create/Advance Visits (VisitsController.Move) | `[Authorize(Policy = "visits.manage")]` | `POST api/visits/{id:guid}/move` | Yes |
| Visits | `visits.manage` | Create/Advance Visits (VisitsController.Update) | `[Authorize(Policy = "visits.manage")]` | `PUT api/visits/{id:guid}` | Yes |

---

## System Permissions Reference Catalog (66 Total)

| Permission Code | Module | Display Name | Source |
| :--- | :--- | :--- | :--- |
| `accounts.deposit` | Accounts | Record Account Deposit | Newly Implemented (23) |
| `accounts.manage` | Accounts | Manage Accounts | Newly Implemented (23) |
| `accounts.reports` | Accounts | View Account Reports | Newly Implemented (23) |
| `accounts.view` | Accounts | View Accounts | Newly Implemented (23) |
| `appointments.manage` | Appointments | Manage Appointments | Newly Implemented (23) |
| `appointments.view` | Appointments | View Appointments | Newly Implemented (23) |
| `billing.invoice` | Billing | Create Invoices | Excel Catalog (43) |
| `billing.payment` | Billing | Record Payments | Excel Catalog (43) |
| `billing.view` | Billing | View Billing | Excel Catalog (43) |
| `bloodbank.manage` | Blood Bank | Manage Inventory/Donors/Requests | Excel Catalog (43) |
| `bloodbank.transfuse` | Blood Bank | Start/Complete Transfusions | Excel Catalog (43) |
| `bloodbank.view` | Blood Bank | View Blood Bank | Excel Catalog (43) |
| `cdss.alert` | CDSS | CDSS Alert Evaluation | Newly Implemented (23) |
| `cdss.manage` | CDSS | Manage CDSS Rules | Newly Implemented (23) |
| `cdss.view` | CDSS | View CDSS Insights | Newly Implemented (23) |
| `emergency.manage` | Emergency | Manage Emergency Cases | Newly Implemented (23) |
| `emergency.triage` | Emergency | Perform Emergency Triage | Newly Implemented (23) |
| `emergency.view` | Emergency | View Emergency Cases | Newly Implemented (23) |
| `emr.edit` | EMR | Edit Records | Excel Catalog (43) |
| `emr.view` | EMR | View Records | Excel Catalog (43) |
| `hr.leave.approve` | HR | Approve Leave Requests | Excel Catalog (43) |
| `hr.manage` | HR | Manage Employees/Attendance/Masters | Excel Catalog (43) |
| `hr.payroll` | HR | Process/View Payroll | Excel Catalog (43) |
| `hr.view` | HR | View HR Records | Excel Catalog (43) |
| `icu.admit` | ICU | Admit/Discharge ICU Patients | Excel Catalog (43) |
| `icu.chart` | ICU | Record Vitals/Meds/IO/Notes/Procedures | Excel Catalog (43) |
| `icu.view` | ICU | View ICU Admissions | Excel Catalog (43) |
| `inventory.assets.manage` | Inventory | Manage Assets/Maintenance | Excel Catalog (43) |
| `inventory.assets.view` | Inventory | View Assets | Excel Catalog (43) |
| `inventory.procurement.approve` | Inventory | Approve Purchase Orders | Excel Catalog (43) |
| `inventory.procurement.manage` | Inventory | Create/Receive POs | Excel Catalog (43) |
| `inventory.procurement.view` | Inventory | View Suppliers/POs | Excel Catalog (43) |
| `inventory.stock.manage` | Inventory | Manage Stock/Items | Excel Catalog (43) |
| `inventory.stock.view` | Inventory | View Stock/Items | Excel Catalog (43) |
| `lab.order` | Laboratory | Order Tests | Excel Catalog (43) |
| `lab.result` | Laboratory | Enter Results | Excel Catalog (43) |
| `lab.view` | Laboratory | View Lab | Excel Catalog (43) |
| `master.manage` | Master | Manage Master Data | Newly Implemented (23) |
| `master.view` | Master | View Master Data | Newly Implemented (23) |
| `ot.manage` | Operation Theatre | Manage Theatres/Procedures | Excel Catalog (43) |
| `ot.perform` | Operation Theatre | Checklist/Notes/Close Case | Excel Catalog (43) |
| `ot.schedule` | Operation Theatre | Book Surgery | Excel Catalog (43) |
| `ot.view` | Operation Theatre | View OT Schedule/Records | Excel Catalog (43) |
| `patients.create` | Patients | Create Patients | Excel Catalog (43) |
| `patients.delete` | Patients | Delete Patients | Excel Catalog (43) |
| `patients.edit` | Patients | Edit Patients | Excel Catalog (43) |
| `patients.view` | Patients | View Patients | Excel Catalog (43) |
| `permissions.manage` | Admin | Manage Permissions | Excel Catalog (43) |
| `pharmacy.dispense` | Pharmacy | Dispense Drugs | Excel Catalog (43) |
| `pharmacy.manage` | Pharmacy | Manage Stock | Excel Catalog (43) |
| `pharmacy.view` | Pharmacy | View Pharmacy | Excel Catalog (43) |
| `radiology.manage` | Radiology | Manage Radiology Orders | Newly Implemented (23) |
| `radiology.order` | Radiology | Order Radiology Examination | Newly Implemented (23) |
| `radiology.report` | Radiology | Create/Submit Radiology Report | Newly Implemented (23) |
| `radiology.view` | Radiology | View Radiology | Newly Implemented (23) |
| `referrals.manage` | Referrals | Create/Respond Referrals | Excel Catalog (43) |
| `referrals.view` | Referrals | View Referrals | Excel Catalog (43) |
| `reports.manage` | Reports | Manage Reports | Newly Implemented (23) |
| `reports.view` | Reports | View Reports | Newly Implemented (23) |
| `roles.manage` | Admin | Manage Roles | Excel Catalog (43) |
| `specialclinics.book` | Special Clinics | Book Special Clinic Appointment | Newly Implemented (23) |
| `specialclinics.manage` | Special Clinics | Manage Special Clinics | Newly Implemented (23) |
| `specialclinics.view` | Special Clinics | View Special Clinics | Newly Implemented (23) |
| `users.manage` | Admin | Manage Users | Excel Catalog (43) |
| `visits.manage` | Visits | Create/Advance Visits | Excel Catalog (43) |
| `visits.view` | Visits | View Visits | Excel Catalog (43) |
