
### STEP 07: Payroll Computation Engine & Kenya Statutory Returns

#### 1. Statutory Deduction Formulas (Kenya Finance Act Engine)
* **Monthly PAYE Tax Bands:**
  1. Up to KES 24,000: **10%** (Max KES 2,400)
  2. KES 24,001 – KES 32,333: **25%** (Max KES 2,083.25)
  3. KES 32,334 – KES 500,000: **30%** (Max KES 140,300)
  4. KES 500,001 – KES 800,000: **32.5%** (Max KES 97,500)
  5. Above KES 800,000: **35%**
* **Tax Reliefs:**
  * Personal Relief: **KES 2,400 per month** (KES 28,800/yr).
  * Insurance Relief: 15% of premium up to KES 5,000/mo (SHIF treated as allowable deduction).
* **NSSF (National Social Security Fund):**
  * Tier I (LEL): 6% on earnings up to KES 8,000 = **KES 480**.
  * Tier II (UEL): 6% on earnings between KES 8,001 and KES 72,000 = **KES 3,840**.
  * Total Max Employee NSSF: **KES 4,320** (Matched 100% by Employer).
* **SHIF (Social Health Insurance Fund):**
  * **2.75%** of Gross Pay; Statutory Minimum: **KES 300**.
* **Affordable Housing Levy (AHL):**
  * Employee: **1.5%** of Gross Pay.
  * Employer Match: **1.5%** of Gross Pay.
* **Allowable Deductions:**
  * `Allowable = NSSF (Employee) + SHIF + AHL + Min(Pension, KES 20,000)`.
  * `Taxable Pay = Gross Pay - Allowable Deductions`.
  * `Net Pay = Gross Pay - (Statutory Deductions + Recurring Deductions)`.

#### 2. Pay Components & Direct Attendance Integration
* **Earnings:**
  * Basic Salary.
  * Overtime Pay: `OvertimeHours * (BasicSalary / 26 / 8) * OvertimeMultiplier` (default 1.5x).
  * Unpaid Leave Deduction: `UnpaidDays * (BasicSalary / 26)`.
  * Half-Pay Sick Leave Adjustment: `HalfPayDays * (BasicSalary / 26) * 0.5`.
* **Recurring Deductions:**
  * `LOAN`: Staff loan repayment (Post-tax, has balance tracking).
  * `ADV`: Salary advance (Post-tax, has balance tracking).
  * `PENS`: Voluntary pension (Pre-tax, allowable up to cap).
  * `WELF`: Staff welfare fund (Post-tax).

#### 3. Payroll Outputs & Exports
* **Payroll Register:** Detailed line-by-line component spreadsheet.
* **Bank Payment File:** Standard EFT/RTGS CSV (`AccountName`, `BankCode`, `BranchCode`, `AccountNumber`, `NetPay`, `Narrative`). Drops employees with missing bank details and flags an exception report.
* **KRA P10 Monthly Return:** Mapped to KRA iTax macro upload structure.
* **NSSF Monthly Return:** With Tier I and Tier II member-by-member breakdown.
* **SHIF Monthly Return:** Member KRA PIN, ID, Gross Earnings, 2.75% Contribution.
* **GL Journal Export:** Double-entry journal voucher by Cost Centre (`Debit Gross Salary & Employer Cost`, `Credit Net Pay Payable & Statutory Payables`). Balanced control: `Debits == Credits`.
