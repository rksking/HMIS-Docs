# TrencoCare HIMS — Comprehensive Permissions & Endpoint Audit Matrix

Status: Complete Endpoint-to-Permission Mapping  
Scope: All 295 backend controller endpoints across 17 microservices + IdentityServer compared against system permission catalog.  

## Summary Statistics

- **Total Endpoints Audited:** 295
- **Policy-Enforced Endpoints (`IsPermissionImplemented = Yes`):** 131 (44.4%)
- **Unprotected / Missing Endpoints (`IsPermissionImplemented = No`):** 145 (49.2%)
- **Partial (Role-Only or Bare Auth without Policy):** 15 (5.1%)
- **Public / Session Endpoints:** 4

---

## Permissions Audit Table

| Module | Code in System | Action / Intended Scope | Current Controller State | Endpoint | IsPermissionImplemented |
|---|---|---|---|---|:---:|
| Accounts | None (Missing in system) | Accounts (Accounting / Ledger / Deposit) | No `[Authorize]` | `GET /api/accounts/accounts` | No |
| Accounts | None (Missing in system) | CreateAccount (Accounting / Ledger / Deposit) | No `[Authorize]` | `POST /api/accounts/accounts` | No |
| Accounts | None (Missing in system) | UpdateAccount (Accounting / Ledger / Deposit) | No `[Authorize]` | `PUT /api/accounts/accounts/{id:guid}` | No |
| Accounts | None (Missing in system) | CreateEntry (Accounting / Ledger / Deposit) | No `[Authorize]` | `POST /api/accounts/journal` | No |
| Accounts | None (Missing in system) | Entries (Accounting / Ledger / Deposit) | No `[Authorize]` | `GET /api/accounts/journal` | No |
| Accounts | None (Missing in system) | Entry (Accounting / Ledger / Deposit) | No `[Authorize]` | `GET /api/accounts/journal/{id:guid}` | No |
| Accounts | None (Missing in system) | Void (Accounting / Ledger / Deposit) | No `[Authorize]` | `POST /api/accounts/journal/{id:guid}/void` | No |
| Accounts | None (Missing in system) | Ledger (Accounting / Ledger / Deposit) | No `[Authorize]` | `GET /api/accounts/ledger/{accountId:guid}` | No |
| Accounts | None (Missing in system) | TrialBalance (Accounting / Ledger / Deposit) | No `[Authorize]` | `GET /api/accounts/reports/trial-balance` | No |
| Accounts | None (Missing in system) | IncomeStatement (Accounting / Ledger / Deposit) | No `[Authorize]` | `GET /api/accounts/reports/income-statement` | No |
| Accounts | None (Missing in system) | CreateDeposit (Accounting / Ledger / Deposit) | No `[Authorize]` | `POST /api/accounts/deposits` | No |
| Accounts | None (Missing in system) | CreateRefund (Accounting / Ledger / Deposit) | No `[Authorize]` | `POST /api/accounts/refunds` | No |
| Accounts | None (Missing in system) | Deposit (Accounting / Ledger / Deposit) | No `[Authorize]` | `GET /api/accounts/deposits/{id:guid}` | No |
| Accounts | None (Missing in system) | PatientDeposits (Accounting / Ledger / Deposit) | No `[Authorize]` | `GET /api/accounts/deposits/patient/{patientId:guid}` | No |
| Accounts | None (Missing in system) | Statement (Accounting / Ledger / Deposit) | No `[Authorize]` | `GET /api/accounts/statement/patient/{patientId:guid}` | No |
| Accounts | None (Missing in system) | ArAging (Accounting / Ledger / Deposit) | No `[Authorize]` | `GET /api/accounts/reports/ar-aging` | No |
| Accounts | None (Missing in system) | SyncBilling (Accounting / Ledger / Deposit) | No `[Authorize]` | `POST /api/accounts/ledger/sync-billing` | No |
| Appointments | None (Missing in system) | Schedule (Appointment scheduling) | No `[Authorize]` | `GET /api/appointments` | No |
| Appointments | None (Missing in system) | UpdateStatus (Appointment scheduling) | No `[Authorize]` | `PUT /api/appointments/{id:guid}/status` | No |
| Admin - Users | None (Public) | Register (Public authentication) | No `[Authorize]` | `POST /api/auth/register` | N/A (Public) |
| Admin - Users | None (Public) | Login (Public authentication) | No `[Authorize]` | `POST /api/auth/login` | N/A (Public) |
| Admin - Users | `users.manage` | GetUsers | `[Authorize(Roles = "Admin")]` | `GET /api/auth/users` | Partial (Role-only) |
| Admin - Users | `users.manage` | AssignRole | `[Authorize(Roles = "Admin")]` | `POST /api/auth/assign-role` | Partial (Role-only) |
| Admin - Users | `users.manage` | AssignLocation | `[Authorize(Roles = "Admin")]` | `POST /api/auth/users/assign-location` | Partial (Role-only) |
| Authentication and users | None (Session-level) | SwitchBranch | `[Authorize]` (Class-level) | `POST /api/auth/switch-branch` | N/A (Authenticated) |
| Authentication and users | None (Session-level) | Me | `[Authorize]` (Class-level) | `GET /api/auth/me` | N/A (Authenticated) |
| Billing | `billing.invoice` | Create | `[Authorize(Policy = "billing.invoice")]` | `POST /api/billing/invoices` | Yes |
| Billing | `billing.view` | Get | `[Authorize(Policy = "billing.view")]` | `GET /api/billing/invoices/{id:guid}` | Yes |
| Billing | `billing.view` | GetPatient | `[Authorize(Policy = "billing.view")]` | `GET /api/billing/invoices/patient/{patientId:guid}` | Yes |
| Billing | `billing.view` | GetAll | `[Authorize(Policy = "billing.view")]` | `GET /api/billing/invoices` | Yes |
| Billing | `billing.view` | GetCharges | `[Authorize(Policy = "billing.view")]` | `GET /api/billing/charges` | Yes |
| Billing | `billing.invoice` | CreateCharge | `[Authorize(Policy = "billing.invoice")]` | `POST /api/billing/charges` | Yes |
| Billing | `billing.invoice` | UpdateCharge | `[Authorize(Policy = "billing.invoice")]` | `PUT /api/billing/charges/{id:guid}` | Yes |
| Billing | `billing.payment` | Pay | `[Authorize(Policy = "billing.payment")]` | `POST /api/billing/payments` | Yes |
| Billing | `billing.view` | Revenue | `[Authorize(Policy = "billing.view")]` | `GET /api/billing/reports/revenue` | Yes |
| Blood bank | `bloodbank.manage` | AddUnit | `[Authorize(Policy = "bloodbank.manage")]` | `POST /api/bloodbank/units` | Yes |
| Blood bank | `bloodbank.view` | GetUnits | `[Authorize(Policy = "bloodbank.view")]` | `GET /api/bloodbank/units` | Yes |
| Blood bank | `bloodbank.view` | Available | `[Authorize(Policy = "bloodbank.view")]` | `GET /api/bloodbank/units/available` | Yes |
| Blood bank | `bloodbank.view` | Expiring | `[Authorize(Policy = "bloodbank.view")]` | `GET /api/bloodbank/units/expiring` | Yes |
| Blood bank | `bloodbank.view` | StockSummary | `[Authorize(Policy = "bloodbank.view")]` | `GET /api/bloodbank/units/stock-summary` | Yes |
| Blood bank | `bloodbank.manage` | Discard | `[Authorize(Policy = "bloodbank.manage")]` | `POST /api/bloodbank/units/{id:guid}/discard` | Yes |
| Blood bank | `bloodbank.manage` | Reserve | `[Authorize(Policy = "bloodbank.manage")]` | `POST /api/bloodbank/units/{id:guid}/reserve` | Yes |
| Blood bank | `bloodbank.manage` | CreateRequest | `[Authorize(Policy = "bloodbank.manage")]` | `POST /api/bloodbank/requests` | Yes |
| Blood bank | `bloodbank.view` | GetRequests | `[Authorize(Policy = "bloodbank.view")]` | `GET /api/bloodbank/requests` | Yes |
| Blood bank | `bloodbank.view` | GetRequest | `[Authorize(Policy = "bloodbank.view")]` | `GET /api/bloodbank/requests/{id:guid}` | Yes |
| Blood bank | `bloodbank.view` | PatientRequests | `[Authorize(Policy = "bloodbank.view")]` | `GET /api/bloodbank/requests/patient/{patientId:guid}` | Yes |
| Blood bank | `bloodbank.manage` | UpdateRequestStatus | `[Authorize(Policy = "bloodbank.manage")]` | `PUT /api/bloodbank/requests/{id:guid}/status` | Yes |
| Blood bank | `bloodbank.view` | RequestTransfusions | `[Authorize(Policy = "bloodbank.view")]` | `GET /api/bloodbank/requests/{id:guid}/transfusions` | Yes |
| Blood bank | `bloodbank.transfuse` | Start | `[Authorize(Policy = "bloodbank.transfuse")]` | `POST /api/bloodbank/transfusions/start` | Yes |
| Blood bank | `bloodbank.transfuse` | Complete | `[Authorize(Policy = "bloodbank.transfuse")]` | `POST /api/bloodbank/transfusions/{id:guid}/complete` | Yes |
| Blood bank | `bloodbank.view` | GetTransfusions | `[Authorize(Policy = "bloodbank.view")]` | `GET /api/bloodbank/transfusions` | Yes |
| Blood bank | `bloodbank.view` | GetDonors | `[Authorize(Policy = "bloodbank.view")]` | `GET /api/bloodbank/donors` | Yes |
| Blood bank | `bloodbank.view` | GetDonor | `[Authorize(Policy = "bloodbank.view")]` | `GET /api/bloodbank/donors/{id:guid}` | Yes |
| Blood bank | `bloodbank.manage` | CreateDonor | `[Authorize(Policy = "bloodbank.manage")]` | `POST /api/bloodbank/donors` | Yes |
| Blood bank | `bloodbank.manage` | UpdateDonor | `[Authorize(Policy = "bloodbank.manage")]` | `PUT /api/bloodbank/donors/{id:guid}` | Yes |
| CDSS | None (Missing in system) | Check (Clinical rules / alerts) | No `[Authorize]` | `POST /api/cdss/check` | No |
| CDSS | None (Missing in system) | CreateAlert (Clinical rules / alerts) | No `[Authorize]` | `POST /api/cdss/alerts` | No |
| CDSS | None (Missing in system) | PatientAlerts (Clinical rules / alerts) | No `[Authorize]` | `GET /api/cdss/alerts/patient/{patientId:guid}` | No |
| CDSS | None (Missing in system) | RecentAlerts (Clinical rules / alerts) | No `[Authorize]` | `GET /api/cdss/alerts` | No |
| CDSS | None (Missing in system) | Acknowledge (Clinical rules / alerts) | No `[Authorize]` | `POST /api/cdss/alerts/acknowledge` | No |
| CDSS | None (Missing in system) | Interactions (Clinical rules / alerts) | No `[Authorize]` | `GET /api/cdss/drug-interactions` | No |
| CDSS | None (Missing in system) | CreateInteraction (Clinical rules / alerts) | No `[Authorize]` | `POST /api/cdss/drug-interactions` | No |
| CDSS | None (Missing in system) | CheckInteractions (Clinical rules / alerts) | No `[Authorize]` | `POST /api/cdss/drug-interactions/check` | No |
| CDSS | None (Missing in system) | AllergyRules (Clinical rules / alerts) | No `[Authorize]` | `GET /api/cdss/allergy-rules` | No |
| CDSS | None (Missing in system) | CreateAllergyRule (Clinical rules / alerts) | No `[Authorize]` | `POST /api/cdss/allergy-rules` | No |
| CDSS | None (Missing in system) | DoseRules (Clinical rules / alerts) | No `[Authorize]` | `GET /api/cdss/dose-rules` | No |
| CDSS | None (Missing in system) | CreateDoseRule (Clinical rules / alerts) | No `[Authorize]` | `POST /api/cdss/dose-rules` | No |
| CDSS | None (Missing in system) | Classes (Clinical rules / alerts) | No `[Authorize]` | `GET /api/cdss/classes` | No |
| CDSS | None (Missing in system) | CreateClass (Clinical rules / alerts) | No `[Authorize]` | `POST /api/cdss/classes` | No |
| CDSS | None (Missing in system) | Reminders (Clinical rules / alerts) | No `[Authorize]` | `GET /api/cdss/reminders` | No |
| CDSS | None (Missing in system) | CreateReminder (Clinical rules / alerts) | No `[Authorize]` | `POST /api/cdss/reminders` | No |
| CDSS | None (Missing in system) | Rules (Clinical rules / alerts) | No `[Authorize]` | `GET /api/cdss/rules` | No |
| CDSS | None (Missing in system) | CreateRule (Clinical rules / alerts) | No `[Authorize]` | `POST /api/cdss/rules` | No |
| CDSS | None (Missing in system) | Summary (Clinical rules / alerts) | No `[Authorize]` | `GET /api/cdss/summary` | No |
| Emergency | None (Missing in system) | Register (Emergency department workflow) | No `[Authorize]` | `POST /api/emergency/register` | No |
| Emergency | None (Missing in system) | QuickRegister (Emergency department workflow) | No `[Authorize]` | `POST /api/emergency/quick-register` | No |
| Emergency | None (Missing in system) | Triage (Emergency department workflow) | No `[Authorize]` | `PUT /api/emergency/{id:guid}/triage` | No |
| Emergency | None (Missing in system) | AssignDoctor (Emergency department workflow) | No `[Authorize]` | `PUT /api/emergency/{id:guid}/assign-doctor` | No |
| Emergency | None (Missing in system) | Complete (Emergency department workflow) | No `[Authorize]` | `PUT /api/emergency/{id:guid}/complete` | No |
| Emergency | None (Missing in system) | Queue (Emergency department workflow) | No `[Authorize]` | `GET /api/emergency/queue` | No |
| Emergency | None (Missing in system) | Get (Emergency department workflow) | No `[Authorize]` | `GET /api/emergency/{id:guid}` | No |
| Emergency | None (Missing in system) | Summary (Emergency department workflow) | No `[Authorize]` | `GET /api/emergency/summary` | No |
| Electronic medical record | `emr.edit` | Create | `[Authorize(Policy = "emr.edit")]` | `POST /api/emr/records` | Yes |
| Electronic medical record | `emr.view` | GetById | `[Authorize(Policy = "emr.view")]` | `GET /api/emr/records/{id:guid}` | Yes |
| Electronic medical record | `emr.view` | GetByVisit | `[Authorize(Policy = "emr.view")]` | `GET /api/emr/records/visit/{visitId:guid}` | Yes |
| Electronic medical record | `emr.view` | History | `[Authorize(Policy = "emr.view")]` | `GET /api/emr/patients/{patientId:guid}/history` | Yes |
| Electronic medical record | `emr.edit` | Update | `[Authorize(Policy = "emr.edit")]` | `PUT /api/emr/records/{id:guid}` | Yes |
| Electronic medical record | `emr.edit` | Sign | `[Authorize(Policy = "emr.edit")]` | `POST /api/emr/records/{id:guid}/sign` | Yes |
| Electronic medical record | `emr.edit` | AddDiagnosis | `[Authorize(Policy = "emr.edit")]` | `POST /api/emr/records/{id:guid}/diagnoses` | Yes |
| Electronic medical record | `emr.edit` | AddVitals | `[Authorize(Policy = "emr.edit")]` | `POST /api/emr/records/{id:guid}/vitals` | Yes |
| Electronic medical record | `emr.edit` | AddNote | `[Authorize(Policy = "emr.edit")]` | `POST /api/emr/records/{id:guid}/notes` | Yes |
| Electronic medical record | `emr.edit` | AddOrder | `[Authorize(Policy = "emr.edit")]` | `POST /api/emr/records/{id:guid}/orders` | Yes |
| Clinical templates | `emr.edit` | Create | `[Authorize(Policy = "emr.edit")]` | `POST /api/emr/templates` | Yes |
| Clinical templates | `emr.view` | GetActive | `[Authorize(Policy = "emr.view")]` | `GET /api/emr/templates` | Yes |
| Clinical templates | `emr.view` | Get | `[Authorize(Policy = "emr.view")]` | `GET /api/emr/templates/{id:guid}` | Yes |
| Clinical templates | `emr.edit` | Update | `[Authorize(Policy = "emr.edit")]` | `PUT /api/emr/templates/{id:guid}` | Yes |
| Clinical templates | `emr.edit` | SaveResponse | `[Authorize(Policy = "emr.edit")]` | `POST /api/emr/templates/responses` | Yes |
| Clinical templates | `emr.view` | GetResponse | `[Authorize(Policy = "emr.view")]` | `GET /api/emr/templates/responses/record/{clinicalRecordId:guid}` | Yes |
| Human resources | `hr.manage` | Create | `[Authorize(Policy = "hr.manage")]` | `POST /api/hr/employees` | Yes |
| Human resources | `hr.view` | List | `[Authorize(Policy = "hr.view")]` | `GET /api/hr/employees` | Yes |
| Human resources | `hr.view` | Get | `[Authorize(Policy = "hr.view")]` | `GET /api/hr/employees/{id:guid}` | Yes |
| Human resources | `hr.manage` | Update | `[Authorize(Policy = "hr.manage")]` | `PUT /api/hr/employees/{id:guid}` | Yes |
| Human resources | `hr.manage` | RecordAttendance | `[Authorize(Policy = "hr.manage")]` | `POST /api/hr/attendance` | Yes |
| Human resources | `hr.view` | Attendance | `[Authorize(Policy = "hr.view")]` | `GET /api/hr/attendance` | Yes |
| Human resources | `hr.view` | AttendanceSummary | `[Authorize(Policy = "hr.view")]` | `GET /api/hr/attendance/summary` | Yes |
| Human resources | `hr.manage` | ApplyLeave | `[Authorize(Policy = "hr.manage")]` | `POST /api/hr/leaves` | Yes |
| Human resources | `hr.leave.approve` | Approve | `[Authorize(Policy = "hr.leave.approve")]` | `PUT /api/hr/leaves/approve` | Yes |
| Human resources | `hr.view` | Leaves | `[Authorize(Policy = "hr.view")]` | `GET /api/hr/leaves` | Yes |
| Human resources | `hr.view` | Balances | `[Authorize(Policy = "hr.view")]` | `GET /api/hr/leaves/balances` | Yes |
| Human resources | `hr.payroll` | Payroll | `[Authorize(Policy = "hr.payroll")]` | `POST /api/hr/payroll/process` | Yes |
| Human resources | `hr.payroll` | MarkPaid | `[Authorize(Policy = "hr.payroll")]` | `POST /api/hr/payroll/{id:guid}/pay` | Yes |
| Human resources | `hr.payroll` | GetPayroll | `[Authorize(Policy = "hr.payroll")]` | `GET /api/hr/payroll/{id:guid}` | Yes |
| Human resources | `hr.payroll` | Register | `[Authorize(Policy = "hr.payroll")]` | `GET /api/hr/payroll/register` | Yes |
| Human resources | `hr.payroll` | EmployeePayrolls | `[Authorize(Policy = "hr.payroll")]` | `GET /api/hr/payroll/employee/{employeeId:guid}` | Yes |
| Human resources | `hr.view` | Departments | `[Authorize(Policy = "hr.view")]` | `GET /api/hr/departments` | Yes |
| Human resources | `hr.manage` | CreateDepartment | `[Authorize(Policy = "hr.manage")]` | `POST /api/hr/departments` | Yes |
| Human resources | `hr.view` | Designations | `[Authorize(Policy = "hr.view")]` | `GET /api/hr/designations` | Yes |
| Human resources | `hr.manage` | CreateDesignation | `[Authorize(Policy = "hr.manage")]` | `POST /api/hr/designations` | Yes |
| Human resources | `hr.view` | LeaveTypes | `[Authorize(Policy = "hr.view")]` | `GET /api/hr/leave-types` | Yes |
| Human resources | `hr.manage` | CreateLeaveType | `[Authorize(Policy = "hr.manage")]` | `POST /api/hr/leave-types` | Yes |
| Human resources | `hr.view` | Summary | `[Authorize(Policy = "hr.view")]` | `GET /api/hr/summary` | Yes |
| ICU | `icu.admit` | Admit | `[Authorize(Policy = "icu.admit")]` | `POST /api/icu/admissions` | Yes |
| ICU | `icu.view` | Get | `[Authorize(Policy = "icu.view")]` | `GET /api/icu/admissions/{id:guid}` | Yes |
| ICU | `icu.view` | Active | `[Authorize(Policy = "icu.view")]` | `GET /api/icu/admissions/active` | Yes |
| ICU | `icu.chart` | Vitals | `[Authorize(Policy = "icu.chart")]` | `POST /api/icu/admissions/{id:guid}/vitals` | Yes |
| ICU | `icu.admit` | Discharge | `[Authorize(Policy = "icu.admit")]` | `POST /api/icu/admissions/{id:guid}/discharge` | Yes |
| ICU | `icu.chart` | RecordMedication | `[Authorize(Policy = "icu.chart")]` | `POST /api/icu/admissions/{id:guid}/medications` | Yes |
| ICU | `icu.chart` | RecordIO | `[Authorize(Policy = "icu.chart")]` | `POST /api/icu/admissions/{id:guid}/intake-output` | Yes |
| ICU | `icu.chart` | AddNote | `[Authorize(Policy = "icu.chart")]` | `POST /api/icu/admissions/{id:guid}/notes` | Yes |
| ICU | `icu.chart` | RecordProcedure | `[Authorize(Policy = "icu.chart")]` | `POST /api/icu/admissions/{id:guid}/procedures` | Yes |
| Inventory and procurement | `inventory.stock.manage` | Create | `[Authorize(Policy = "inventory.stock.manage")]` | `POST /api/inventory/items` | Yes |
| Inventory and procurement | `inventory.stock.view` | List | `[Authorize(Policy = "inventory.stock.view")]` | `GET /api/inventory/items` | Yes |
| Inventory and procurement | `inventory.stock.view` | Search | `[Authorize(Policy = "inventory.stock.view")]` | `GET /api/inventory/items/search` | Yes |
| Inventory and procurement | `inventory.stock.view` | Get | `[Authorize(Policy = "inventory.stock.view")]` | `GET /api/inventory/items/{id:guid}` | Yes |
| Inventory and procurement | `inventory.stock.manage` | Update | `[Authorize(Policy = "inventory.stock.manage")]` | `PUT /api/inventory/items/{id:guid}` | Yes |
| Inventory and procurement | `inventory.stock.view` | ItemTx | `[Authorize(Policy = "inventory.stock.view")]` | `GET /api/inventory/items/{id:guid}/transactions` | Yes |
| Inventory and procurement | `inventory.stock.manage` | Adjust | `[Authorize(Policy = "inventory.stock.manage")]` | `POST /api/inventory/stock/adjust` | Yes |
| Inventory and procurement | `inventory.stock.view` | Expiring | `[Authorize(Policy = "inventory.stock.view")]` | `GET /api/inventory/stock/expiring` | Yes |
| Inventory and procurement | `inventory.stock.view` | Summary | `[Authorize(Policy = "inventory.stock.view")]` | `GET /api/inventory/summary` | Yes |
| Inventory and procurement | `inventory.stock.view` | Categories | `[Authorize(Policy = "inventory.stock.view")]` | `GET /api/inventory/categories` | Yes |
| Inventory and procurement | `inventory.stock.manage` | CreateCategory | `[Authorize(Policy = "inventory.stock.manage")]` | `POST /api/inventory/categories` | Yes |
| Inventory and procurement | `inventory.stock.view` | Units | `[Authorize(Policy = "inventory.stock.view")]` | `GET /api/inventory/units` | Yes |
| Inventory and procurement | `inventory.stock.manage` | CreateUnit | `[Authorize(Policy = "inventory.stock.manage")]` | `POST /api/inventory/units` | Yes |
| Inventory - Procurement | `inventory.procurement.view` | Suppliers (View suppliers / POs / GRNs) | No `[Authorize]` | `GET /api/inventory/suppliers` | No |
| Inventory - Procurement | `inventory.procurement.view` | Supplier (View suppliers / POs / GRNs) | No `[Authorize]` | `GET /api/inventory/suppliers/{id:guid}` | No |
| Inventory - Procurement | `inventory.procurement.manage` | CreateSupplier (Manage suppliers / POs) | No `[Authorize]` | `POST /api/inventory/suppliers` | No |
| Inventory - Procurement | `inventory.procurement.manage` | UpdateSupplier (Manage suppliers / POs) | No `[Authorize]` | `PUT /api/inventory/suppliers/{id:guid}` | No |
| Inventory - Procurement | `inventory.procurement.manage` | CreatePO (Manage suppliers / POs) | No `[Authorize]` | `POST /api/inventory/purchase-orders` | No |
| Inventory - Procurement | `inventory.procurement.view` | POs (View suppliers / POs / GRNs) | No `[Authorize]` | `GET /api/inventory/purchase-orders` | No |
| Inventory - Procurement | `inventory.procurement.view` | PO (View suppliers / POs / GRNs) | No `[Authorize]` | `GET /api/inventory/purchase-orders/{id:guid}` | No |
| Inventory - Procurement | `inventory.procurement.approve` | Approve (Approve PO) | No `[Authorize]` | `POST /api/inventory/purchase-orders/{id:guid}/approve` | No |
| Inventory - Procurement | `inventory.procurement.manage` | Order (Manage suppliers / POs) | No `[Authorize]` | `POST /api/inventory/purchase-orders/{id:guid}/order` | No |
| Inventory - Procurement | `inventory.procurement.manage` | Cancel (Manage suppliers / POs) | No `[Authorize]` | `POST /api/inventory/purchase-orders/{id:guid}/cancel` | No |
| Inventory - Procurement | `inventory.procurement.manage` | Close (Manage suppliers / POs) | No `[Authorize]` | `POST /api/inventory/purchase-orders/{id:guid}/close` | No |
| Inventory - Procurement | `inventory.procurement.manage` | Receive (Manage suppliers / POs) | No `[Authorize]` | `POST /api/inventory/purchase-orders/{id:guid}/receive` | No |
| Inventory - Procurement | `inventory.procurement.view` | Receipts (View suppliers / POs / GRNs) | No `[Authorize]` | `GET /api/inventory/goods-receipts` | No |
| Inventory assets | `inventory.assets.view` | Categories | `[Authorize(Policy = "inventory.assets.view")]` | `GET /api/inventory/assets/categories` | Yes |
| Inventory assets | `inventory.assets.manage` | CreateCategory | `[Authorize(Policy = "inventory.assets.manage")]` | `POST /api/inventory/assets/categories` | Yes |
| Inventory assets | `inventory.assets.manage` | Create | `[Authorize(Policy = "inventory.assets.manage")]` | `POST /api/inventory/assets` | Yes |
| Inventory assets | `inventory.assets.view` | List | `[Authorize(Policy = "inventory.assets.view")]` | `GET /api/inventory/assets` | Yes |
| Inventory assets | `inventory.assets.view` | Search | `[Authorize(Policy = "inventory.assets.view")]` | `GET /api/inventory/assets/search` | Yes |
| Inventory assets | `inventory.assets.view` | Summary | `[Authorize(Policy = "inventory.assets.view")]` | `GET /api/inventory/assets/summary` | Yes |
| Inventory assets | `inventory.assets.view` | ServiceDue | `[Authorize(Policy = "inventory.assets.view")]` | `GET /api/inventory/assets/service-due` | Yes |
| Inventory assets | `inventory.assets.view` | Get | `[Authorize(Policy = "inventory.assets.view")]` | `GET /api/inventory/assets/{id:guid}` | Yes |
| Inventory assets | `inventory.assets.manage` | Update | `[Authorize(Policy = "inventory.assets.manage")]` | `PUT /api/inventory/assets/{id:guid}` | Yes |
| Inventory assets | `inventory.assets.manage` | ChangeStatus | `[Authorize(Policy = "inventory.assets.manage")]` | `POST /api/inventory/assets/{id:guid}/status` | Yes |
| Inventory assets | `inventory.assets.manage` | LogMaintenance | `[Authorize(Policy = "inventory.assets.manage")]` | `POST /api/inventory/assets/{id:guid}/maintenance` | Yes |
| Inventory assets | `inventory.assets.view` | MaintenanceLogs | `[Authorize(Policy = "inventory.assets.view")]` | `GET /api/inventory/assets/{id:guid}/maintenance` | Yes |
| Laboratory | `lab.order` | CreateTest (Order test / collect specimen) | No `[Authorize]` | `POST /api/laboratory/tests` | No |
| Laboratory | `lab.view` | GetTests (View lab catalog / orders) | No `[Authorize]` | `GET /api/laboratory/tests` | No |
| Laboratory | `lab.view` | GetTest (View lab catalog / orders) | No `[Authorize]` | `GET /api/laboratory/tests/{id:guid}` | No |
| Laboratory | `lab.order` | UpdateTest (Order test / collect specimen) | No `[Authorize]` | `PUT /api/laboratory/tests/{id:guid}` | No |
| Laboratory | `lab.order` | CreateOrder (Order test / collect specimen) | No `[Authorize]` | `POST /api/laboratory/orders` | No |
| Laboratory | `lab.view` | GetOrder (View lab catalog / orders) | No `[Authorize]` | `GET /api/laboratory/orders/{id:guid}` | No |
| Laboratory | `lab.view` | PatientOrders (View lab catalog / orders) | No `[Authorize]` | `GET /api/laboratory/orders/patient/{patientId:guid}` | No |
| Laboratory | `lab.view` | ByStatus (View lab catalog / orders) | No `[Authorize]` | `GET /api/laboratory/orders/status/{status}` | No |
| Laboratory | `lab.order` | Collect (Order test / collect specimen) | No `[Authorize]` | `PUT /api/laboratory/orders/{id:guid}/collect` | No |
| Laboratory | `lab.result` | EnterResults (Enter/amend lab results) | No `[Authorize]` | `POST /api/laboratory/results` | No |
| Laboratory | `lab.result` | GenerateReport (Enter/amend lab results) | No `[Authorize]` | `POST /api/laboratory/orders/{id:guid}/report` | No |
| Laboratory | `lab.result` | Amend (Enter/amend lab results) | No `[Authorize]` | `POST /api/laboratory/results/amend` | No |
| Master Data | None (Missing in system) | CreateDept (Hospital master catalogs) | No `[Authorize]` | `POST /api/master/departments` | No |
| Master Data | None (Missing in system) | GetDepts (Hospital master catalogs) | No `[Authorize]` | `GET /api/master/departments` | No |
| Master Data | None (Missing in system) | CreateDesig (Hospital master catalogs) | No `[Authorize]` | `POST /api/master/designations` | No |
| Master Data | None (Missing in system) | GetDesig (Hospital master catalogs) | No `[Authorize]` | `GET /api/master/designations` | No |
| Master Data | None (Missing in system) | CreateICD (Hospital master catalogs) | No `[Authorize]` | `POST /api/master/icd-codes` | No |
| Master Data | None (Missing in system) | SearchICD (Hospital master catalogs) | No `[Authorize]` | `GET /api/master/icd-codes/search` | No |
| Master Data | None (Missing in system) | CreateDrugCat (Hospital master catalogs) | No `[Authorize]` | `POST /api/master/drug-categories` | No |
| Master Data | None (Missing in system) | GetDrugCats (Hospital master catalogs) | No `[Authorize]` | `GET /api/master/drug-categories` | No |
| Master Data | None (Missing in system) | CreateTestCat (Hospital master catalogs) | No `[Authorize]` | `POST /api/master/test-categories` | No |
| Master Data | None (Missing in system) | GetTestCats (Hospital master catalogs) | No `[Authorize]` | `GET /api/master/test-categories` | No |
| Master Data | None (Missing in system) | CreateBed (Hospital master catalogs) | No `[Authorize]` | `POST /api/master/beds` | No |
| Master Data | None (Missing in system) | GetBeds (Hospital master catalogs) | No `[Authorize]` | `GET /api/master/beds` | No |
| Master Data | None (Missing in system) | UpdateBed (Hospital master catalogs) | No `[Authorize]` | `PUT /api/master/beds/{id:guid}/status` | No |
| Master Data | None (Missing in system) | GetLocations (Hospital master catalogs) | No `[Authorize]` | `GET /api/master/locations` | No |
| Master Data | None (Missing in system) | GetLocation (Hospital master catalogs) | No `[Authorize]` | `GET /api/master/locations/{id:guid}` | No |
| Master Data | None (Missing in system) | CreateLocation (Hospital master catalogs) | No `[Authorize]` | `POST /api/master/locations` | No |
| Master Data | None (Missing in system) | UpdateLocation (Hospital master catalogs) | No `[Authorize]` | `PUT /api/master/locations/{id:guid}` | No |
| Master Data | None (Missing in system) | GetDoctors (Hospital master catalogs) | No `[Authorize]` | `GET /api/master/doctors` | No |
| Master Data | None (Missing in system) | CreateDoctor (Hospital master catalogs) | No `[Authorize]` | `POST /api/master/doctors` | No |
| Master Data | None (Missing in system) | UpdateDoctor (Hospital master catalogs) | No `[Authorize]` | `PUT /api/master/doctors/{id:guid}` | No |
| Master Data | None (Missing in system) | GetFeeTypes (Hospital master catalogs) | No `[Authorize]` | `GET /api/master/fee-types` | No |
| Master Data | None (Missing in system) | CreateFeeType (Hospital master catalogs) | No `[Authorize]` | `POST /api/master/fee-types` | No |
| Master Data | None (Missing in system) | GetSchemes (Hospital master catalogs) | No `[Authorize]` | `GET /api/master/insurance-schemes` | No |
| Master Data | None (Missing in system) | CreateScheme (Hospital master catalogs) | No `[Authorize]` | `POST /api/master/insurance-schemes` | No |
| Master Data | None (Missing in system) | UpdateScheme (Hospital master catalogs) | No `[Authorize]` | `PUT /api/master/insurance-schemes/{id:guid}` | No |
| Master Data | None (Missing in system) | BloodGroups (Hospital master catalogs) | No `[Authorize]` | `GET /api/master/lookups/blood-groups` | No |
| Master Data | None (Missing in system) | Genders (Hospital master catalogs) | No `[Authorize]` | `GET /api/master/lookups/genders` | No |
| Master Data | None (Missing in system) | LeaveTypes (Hospital master catalogs) | No `[Authorize]` | `GET /api/master/lookups/leave-types` | No |
| Master Data | None (Missing in system) | PaymentMethods (Hospital master catalogs) | No `[Authorize]` | `GET /api/master/lookups/payment-methods` | No |
| Master Data | None (Missing in system) | VisitTypes (Hospital master catalogs) | No `[Authorize]` | `GET /api/master/lookups/visit-types` | No |
| Master Data | None (Missing in system) | VisitSubTypes (Hospital master catalogs) | No `[Authorize]` | `GET /api/master/lookups/visit-sub-types` | No |
| Master Data | None (Missing in system) | BillTypes (Hospital master catalogs) | No `[Authorize]` | `GET /api/master/lookups/bill-types` | No |
| Operation theatre | `ot.view` | Theatres | `[Authorize(Policy = "ot.view")]` | `GET /api/ot/theatres` | Yes |
| Operation theatre | `ot.manage` | CreateTheatre | `[Authorize(Policy = "ot.manage")]` | `POST /api/ot/theatres` | Yes |
| Operation theatre | `ot.view` | Procedures | `[Authorize(Policy = "ot.view")]` | `GET /api/ot/procedures` | Yes |
| Operation theatre | `ot.manage` | CreateProcedure | `[Authorize(Policy = "ot.manage")]` | `POST /api/ot/procedures` | Yes |
| Operation theatre | `ot.schedule` | Schedule | `[Authorize(Policy = "ot.schedule")]` | `POST /api/ot/surgeries` | Yes |
| Operation theatre | `ot.view` | Get | `[Authorize(Policy = "ot.view")]` | `GET /api/ot/surgeries/{id:guid}` | Yes |
| Operation theatre | `ot.view` | List | `[Authorize(Policy = "ot.view")]` | `GET /api/ot/surgeries` | Yes |
| Operation theatre | `ot.view` | Today | `[Authorize(Policy = "ot.view")]` | `GET /api/ot/surgeries/today` | Yes |
| Operation theatre | `ot.view` | Patient | `[Authorize(Policy = "ot.view")]` | `GET /api/ot/surgeries/patient/{patientId:guid}` | Yes |
| Operation theatre | `ot.perform` | Status | `[Authorize(Policy = "ot.perform")]` | `PUT /api/ot/surgeries/{id:guid}/status` | Yes |
| Operation theatre | `ot.perform` | Checklist | `[Authorize(Policy = "ot.perform")]` | `POST /api/ot/surgeries/{id:guid}/checklist` | Yes |
| Operation theatre | `ot.perform` | Notes | `[Authorize(Policy = "ot.perform")]` | `POST /api/ot/surgeries/{id:guid}/operative-notes` | Yes |
| Operation theatre | `ot.perform` | PostOp | `[Authorize(Policy = "ot.perform")]` | `POST /api/ot/surgeries/{id:guid}/post-op` | Yes |
| Operation theatre | `ot.perform` | Close | `[Authorize(Policy = "ot.perform")]` | `POST /api/ot/surgeries/{id:guid}/close` | Yes |
| Patients | `patients.create` | Create (Register new patient) | No `[Authorize]` | `POST /api/patients` | No |
| Patients | `patients.view` | GetById (View patient record) | No `[Authorize]` | `GET /api/patients/{id:guid}` | No |
| Patients | `patients.view` | GetByMRN (View patient record) | No `[Authorize]` | `GET /api/patients/mrn/{mrn}` | No |
| Patients | `patients.view` | GetAll (View patient record) | No `[Authorize]` | `GET /api/patients` | No |
| Patients | `patients.edit` | Update (Edit patient details) | No `[Authorize]` | `PUT /api/patients/{id:guid}` | No |
| Patients | `patients.delete` | Delete (Delete patient record) | No `[Authorize]` | `DELETE /api/patients/{id:guid}` | No |
| Patients | `patients.create` | CreateAppointment (Book appointment) | No `[Authorize]` | `POST /api/patients/appointments` | No |
| Patients | `patients.view` | GetAppointments (View appointments) | No `[Authorize]` | `GET /api/patients/{patientId:guid}/appointments` | No |
| Admin - Permissions | `permissions.manage` | GetAll (List all permissions) | No `[Authorize]` | `GET /api/permissions` | No |
| Pharmacy | `pharmacy.manage` | CreateDrug (Manage drug/stock) | No `[Authorize]` | `POST /api/pharmacy/drugs` | No |
| Pharmacy | `pharmacy.view` | GetDrugs (View drugs/stock/prescriptions) | No `[Authorize]` | `GET /api/pharmacy/drugs` | No |
| Pharmacy | `pharmacy.manage` | AddStock (Manage drug/stock) | No `[Authorize]` | `POST /api/pharmacy/drugs/stock` | No |
| Pharmacy | `pharmacy.view` | LowStock (View drugs/stock/prescriptions) | No `[Authorize]` | `GET /api/pharmacy/drugs/low-stock` | No |
| Pharmacy | `pharmacy.view` | NearExpiry (View drugs/stock/prescriptions) | No `[Authorize]` | `GET /api/pharmacy/drugs/near-expiry` | No |
| Pharmacy | `pharmacy.manage` | CreatePrescription (Prescribe drugs) | No `[Authorize]` | `POST /api/pharmacy/prescriptions` | No |
| Pharmacy | `pharmacy.view` | GetPrescription (View drugs/stock/prescriptions) | No `[Authorize]` | `GET /api/pharmacy/prescriptions/{id:guid}` | No |
| Pharmacy | `pharmacy.view` | GetPatientPrescriptions (View drugs/stock/prescriptions) | No `[Authorize]` | `GET /api/pharmacy/prescriptions/patient/{patientId:guid}` | No |
| Pharmacy | `pharmacy.view` | GetAllPrescriptions (View drugs/stock/prescriptions) | No `[Authorize]` | `GET /api/pharmacy/prescriptions` | No |
| Pharmacy | `pharmacy.view` | GetDrugStock (View drugs/stock/prescriptions) | No `[Authorize]` | `GET /api/pharmacy/drugs/{drugId:guid}/stock` | No |
| Pharmacy | `pharmacy.dispense` | Dispense (Dispense prescription) | No `[Authorize]` | `POST /api/pharmacy/dispense` | No |
| Radiology | None (Missing in system) | GetModalities (Radiology imaging / reports) | No `[Authorize]` | `GET /api/radiology/modalities` | No |
| Radiology | None (Missing in system) | CreateModality (Radiology imaging / reports) | No `[Authorize]` | `POST /api/radiology/modalities` | No |
| Radiology | None (Missing in system) | GetStudyTypes (Radiology imaging / reports) | No `[Authorize]` | `GET /api/radiology/study-types` | No |
| Radiology | None (Missing in system) | GetStudyType (Radiology imaging / reports) | No `[Authorize]` | `GET /api/radiology/study-types/{id:guid}` | No |
| Radiology | None (Missing in system) | CreateStudyType (Radiology imaging / reports) | No `[Authorize]` | `POST /api/radiology/study-types` | No |
| Radiology | None (Missing in system) | UpdateStudyType (Radiology imaging / reports) | No `[Authorize]` | `PUT /api/radiology/study-types/{id:guid}` | No |
| Radiology | None (Missing in system) | Order (Radiology imaging / reports) | No `[Authorize]` | `POST /api/radiology/studies` | No |
| Radiology | None (Missing in system) | Get (Radiology imaging / reports) | No `[Authorize]` | `GET /api/radiology/studies/{id:guid}` | No |
| Radiology | None (Missing in system) | Pending (Radiology imaging / reports) | No `[Authorize]` | `GET /api/radiology/studies/pending` | No |
| Radiology | None (Missing in system) | Patient (Radiology imaging / reports) | No `[Authorize]` | `GET /api/radiology/studies/patient/{patientId:guid}` | No |
| Radiology | None (Missing in system) | Schedule (Radiology imaging / reports) | No `[Authorize]` | `PUT /api/radiology/studies/{id:guid}/schedule` | No |
| Radiology | None (Missing in system) | Perform (Radiology imaging / reports) | No `[Authorize]` | `PUT /api/radiology/studies/{id:guid}/perform` | No |
| Radiology | None (Missing in system) | Report (Radiology imaging / reports) | No `[Authorize]` | `POST /api/radiology/studies/{id:guid}/report` | No |
| Referrals | `referrals.manage` | Create | `[Authorize(Policy = "referrals.manage")]` | `POST /api/referrals` | Yes |
| Referrals | `referrals.view` | Inbox | `[Authorize(Policy = "referrals.view")]` | `GET /api/referrals/inbox` | Yes |
| Referrals | `referrals.view` | Get | `[Authorize(Policy = "referrals.view")]` | `GET /api/referrals/{id:guid}` | Yes |
| Referrals | `referrals.manage` | Accept | `[Authorize(Policy = "referrals.manage")]` | `POST /api/referrals/{id:guid}/accept` | Yes |
| Referrals | `referrals.manage` | Decline | `[Authorize(Policy = "referrals.manage")]` | `POST /api/referrals/{id:guid}/decline` | Yes |
| Reporting | None (Missing in system) | Dashboard (Business & clinical reports) | No `[Authorize]` | `GET /api/reporting/dashboard` | No |
| Reporting | None (Missing in system) | Financial (Business & clinical reports) | No `[Authorize]` | `GET /api/reporting/financial` | No |
| Reporting | None (Missing in system) | Operational (Business & clinical reports) | No `[Authorize]` | `GET /api/reporting/operational` | No |
| Reporting | None (Missing in system) | HR (Business & clinical reports) | No `[Authorize]` | `GET /api/reporting/hr` | No |
| Reporting | None (Missing in system) | Create (Business & clinical reports) | No `[Authorize]` | `POST /api/reporting/definitions` | No |
| Reporting | None (Missing in system) | List (Business & clinical reports) | No `[Authorize]` | `GET /api/reporting/definitions` | No |
| Reporting | None (Missing in system) | Run (Business & clinical reports) | No `[Authorize]` | `POST /api/reporting/run` | No |
| Reporting | None (Missing in system) | History (Business & clinical reports) | No `[Authorize]` | `GET /api/reporting/history` | No |
| Admin - Roles | `roles.manage` | GetAll (List system roles) | No `[Authorize]` | `GET /api/roles` | No |
| Admin - Roles | `roles.manage` | Create | `[Authorize(Roles = "Admin")]` | `POST /api/roles` | Partial (Role-only) |
| Admin - Roles | `roles.manage` | UpdatePermissions | `[Authorize(Roles = "Admin")]` | `PUT /api/roles/permissions` | Partial (Role-only) |
| Special Clinics | None (Missing in system) | Create | `[Authorize]` (Class-level) | `POST /api/special-clinics` | Partial (Auth-only) |
| Special Clinics | None (Missing in system) | GetActive | `[Authorize]` (Class-level) | `GET /api/special-clinics` | Partial (Auth-only) |
| Special Clinics | None (Missing in system) | Get | `[Authorize]` (Class-level) | `GET /api/special-clinics/{id:guid}` | Partial (Auth-only) |
| Special Clinics | None (Missing in system) | Update | `[Authorize]` (Class-level) | `PUT /api/special-clinics/{id:guid}` | Partial (Auth-only) |
| Special Clinics | None (Missing in system) | AssignDoctor | `[Authorize]` (Class-level) | `POST /api/special-clinics/{id:guid}/doctors` | Partial (Auth-only) |
| Special Clinics | None (Missing in system) | RemoveDoctor | `[Authorize]` (Class-level) | `DELETE /api/special-clinics/doctors/{assignmentId:guid}` | Partial (Auth-only) |
| Special Clinics | None (Missing in system) | Slots | `[Authorize]` (Class-level) | `GET /api/special-clinics/{id:guid}/slots` | Partial (Auth-only) |
| Special Clinics | None (Missing in system) | Bookings | `[Authorize]` (Class-level) | `GET /api/special-clinics/{id:guid}/bookings` | Partial (Auth-only) |
| Special Clinics | None (Missing in system) | Book | `[Authorize]` (Class-level) | `POST /api/special-clinics/bookings` | Partial (Auth-only) |
| Special Clinics | None (Missing in system) | UpdateBookingStatus | `[Authorize]` (Class-level) | `PUT /api/special-clinics/bookings/{id:guid}/status` | Partial (Auth-only) |
| Visits and OPD workflow | `visits.manage` | Create | `[Authorize(Policy = "visits.manage")]` | `POST /api/visits` | Yes |
| Visits and OPD workflow | `visits.view` | GetAll | `[Authorize(Policy = "visits.view")]` | `GET /api/visits` | Yes |
| Visits and OPD workflow | `visits.view` | ByStage | `[Authorize(Policy = "visits.view")]` | `GET /api/visits/stage/{stage}` | Yes |
| Visits and OPD workflow | `visits.view` | Counts | `[Authorize(Policy = "visits.view")]` | `GET /api/visits/counts` | Yes |
| Visits and OPD workflow | `visits.view` | Get | `[Authorize(Policy = "visits.view")]` | `GET /api/visits/{id:guid}` | Yes |
| Visits and OPD workflow | `visits.view` | ActiveByPatient | `[Authorize(Policy = "visits.view")]` | `GET /api/visits/patient/{patientId:guid}/active` | Yes |
| Visits and OPD workflow | `visits.manage` | Update | `[Authorize(Policy = "visits.manage")]` | `PUT /api/visits/{id:guid}` | Yes |
| Visits and OPD workflow | `visits.manage` | Advance | `[Authorize(Policy = "visits.manage")]` | `POST /api/visits/{id:guid}/advance` | Yes |
| Visits and OPD workflow | `visits.manage` | Move | `[Authorize(Policy = "visits.manage")]` | `POST /api/visits/{id:guid}/move` | Yes |
| Visits and OPD workflow | `billing.payment` | MarkConsultationPaid | `[Authorize(Policy = "billing.payment")]` | `POST /api/visits/{id:guid}/consultation-payment` | Yes |
