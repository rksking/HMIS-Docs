# Add Employee page (4-step wizard)

Branch: `add-new-emp-page` · Started 2026-10-14 · Follows `docs/follow.md` and `docs/ui.md`.

**Status: PLAN, waiting for approval (Step 0).** Nothing is built yet.

## 1. Goal

Replace the 9-step drawer **"Register New Healthcare Employee"** (`modules/employees/components/CreateEmployeeModal.tsx`, opened from `/employeedirectory`) with a full page **Employee Management → Add Employee** at `/employees/add`. The page has exactly **four steps**:

| # | Step | Sub-text | Content |
| :-- | :-- | :-- | :-- |
| 1 | Personal & Contact | Basic information | Photo, Basic Information, Identification Details, Contact Information, Address Information |
| 2 | Employment & Organisation | Job, department & location | Employment Details, Organisation & Position, Role, Reporting & Team, Shift & Work Schedule |
| 3 | Pay, Documents & Access | Salary, documents & user account | Salary & Banking, Statutory & Compliance, Employee Code, Educational Qualifications, Compliance Documents, Emergency & Next of Kin, User Account Creation, Additional Information |
| 4 | Review & Confirm | Verify and complete | Read-only cards with **Edit** links, a confirmation tick box, then **Complete Onboarding** |

The existing drawer is renamed **"Register Employee & their user access account"** (§4, decision D8).

## 2. What already exists (reuse, do not rebuild)

| Area | Where | Notes |
| :-- | :-- | :-- |
| Stepper | `modules/admin/components/user-wizard/UserWizardStepper.tsx` (`/user/create`) | Arrow-segment strip: green check when done, green title and underline when active, click to go back. Copy it to a shared component. |
| Wizard layout | `modules/admin/views/CreateUserWizardView.tsx` + `CreateUserStep1..4.tsx` + `WizardFooter.tsx` | Same 4-file split is used for the new page |
| Create API | `POST /api/employees` → `EmployeeService.Create.cs` → `CreateEmployeesAsync` | Creates employee, employment, salary, bank, education, shift assignment (`EffectiveFrom = HireDate`), login, leave balances, credentials email |
| Login creation | `EmployeeAccountService.AddLoginAsync` | Username = **employee code**, role = default employee role, `ForcePasswordChange = true` always, `IsActive = true` always |
| Employee code | `GET /api/employees/next-code` + code-availability check | Series preview + manual mode already in the drawer (step 7) |
| Education upload | `employeesApi.uploadEducationCertificate` → staging folder before the person exists | Copied into `{person}/documents/education/` on save |
| Compliance docs | `documentsApi.uploadDocumentFile` after create | Failed uploads are listed, employee still saved |
| Masters | countries, currencies (`useCurrencies`), banks + branches, departments, locations, job titles, grades, cost centres, employment types, shifts, work schedules, document types, education lookups (`useEducationLookups`), genders + contract types (`formLookups`) | All loaded from the DB already |
| Personnel search | `modules/assignment/components/PersonnelAutocomplete.tsx` | Has a `search` prop, so it can be reused for Reporting Manager / Direct HR / HRBP |
| Blob layout | `Application/Common/BlobKeyBuilder.cs` + appsettings `BlobPaths` | `{companyCode}/people/{employeeNumber_name}/…`, start-up refuses blank names |
| Menu | `menus` row `menu-emp-add` "Add Employee" → `/employees/add`, `employees.create`, **isVisible = 0** | Route not built yet. Turn visible in Step 3 |

## 3. Gap analysis: what is missing

Legend: **DB** = new column/table (script goes in `docs/db_changes.sql`), **API** = backend change, **UI** = frontend only.

### Step 1: Personal & Contact
| # | Item (from your spec / mock-up) | Today | Needed |
| :-- | :-- | :-- | :-- |
| G1 | **Upload Photo**, JPG/PNG, max **1 MB** | No photo upload anywhere. `users.avatarUrl` exists but nothing writes it; no `employees.photoUrl` | DB `employees.photoUrl`; appsettings `BlobPaths.Profile = "profile"`; `BlobKeyBuilder.ProfilePhoto` → `{companyCode}/people/{employeeNumber_name}/profile/photo_{ticks}.{ext}`; upload to staging before save, then move on create; API size and type check from appsettings (no hardcoded 1 MB) |
| G2 | Same photo upload on **My Profile** (`/portal/me`) | Not there | `POST /api/me/photo` with the same path and rule; shows on the profile header |
| G3 | **ID Type**, **Country of Issue**, **Date of Issue**, **Date of Expiry** | Only `nationalIdNumber` | DB 4 columns on `employees`; ID Type list from a master (D3) |
| G4 | Salutation and Marital Status lists | **Hardcoded** `<option>`s in the drawer | Read from a master / lookup (see §6) |
| G5 | Phone with country dial code (`+254 ▾`) | Plain text box | Dial code from `countries` master. Default = company's country |
| G6 | Postal = "Same as residential" tick box | Not there | UI only |
| G7 | "Auto-fill from National ID" button | No ID-reading service exists | **Drop** (D9) |

### Step 2: Employment & Organisation
| # | Item | Today | Needed |
| :-- | :-- | :-- | :-- |
| G8 | Company → **Work Station / Location(s)**: multi-select with check boxes and chips, one primary | Single `employment_details.locationId` | DB `employee_locations` (`employeeId, locationId, isPrimary`); primary is also written to `employment_details.locationId`, so every existing report still works. List is filtered by the chosen company. No "+ Add Another Location" link |
| G9 | Department, Designation, Job Grade, Cost Centre as **autocomplete** (large lists); short lists stay dropdowns | All plain `<select>` | Shared `SearchSelect` UI component (client-side filter of master list). Employment Type / Contract Type stay dropdowns |
| G10 | Order: **Reporting Manager (Level 1)** → **Direct HR** → **HRBP**, all autocomplete | Present but as selects in another order | Reuse `PersonnelAutocomplete` with a company-scoped search |
| G11 | **Team / Unit** | No teams master or column | D4: build a Teams master, or drop the field |
| G12 | Assigned Work Shift as **autocomplete**; Work Schedule Policy as **dropdown**; Working Hours per Week (read-only, from shift) | Selects; hours not shown | UI. Hours come from the shift master |
| G13 | Shift **Effective From** date | Always = Date of Joining | API accepts `shiftEffectiveFrom` (default joining date, not earlier) |
| G14 | Expected Confirmation Date auto = joining + probation | Partly (entity `ApplyProbation`) | UI shows the calculated date, read-only |

### Step 3: Pay, Documents & Access
| # | Item | Today | Needed |
| :-- | :-- | :-- | :-- |
| G15 | Salary & bank fields **all optional** | API/validator treats basic salary, bank etc. as required in places | Relax `EmployeeCreateValidator` (keep "if bank chosen, account number required" style rules) |
| G16 | Monthly CTC (estimated) tile | Not there | UI: base salary; currency from currency master |
| G17 | **Tax Residence Country**, **Tax Status**, **NHIF/SHIF Category**, **Pension Scheme** | No columns, no masters | D5: add columns + list values from masters, or drop for now |
| G18 | "Statutory deductions will be calculated as per *Kenya* configuration" | — | Text from the company's country master, never "Kenya" in code |
| G19 | **Employee Code** card (series / manual + preview) | Exists in drawer step 7 | Move as-is |
| G20 | **Educational Qualifications** + **Compliance Documents** | Exist in drawer step 8 | Move as-is, restyle to `docs/ui.md` |
| G21 | **Emergency & Next of Kin** | One contact: 3 columns on `employees` | D6: keep one contact, or a new `employee_contacts` table for several (emergency / next of kin, relation, phone, email, address) |
| G22 | **User Account Creation**: Create User Account toggle, Login Username, **User Role**, Initial Password (auto + email / set manually), **Account Activation** (now / on joining date), **Password forced to change** tick box | Always creates a login, fixed role, always forced, always active | API: `createLogin`, `roleId`, `passwordMode`, `manualPassword`, `forcePasswordChange`, `activation`. Role list from `roles` (only roles the creator may grant). Audit every choice |
| G23 | Login username rule | Username = employee code (mock-up 5 says the same) but mock-up 3 says "Use Work Email as Username" | D1 |
| G24 | Additional Information → **Notes** (500 chars) | `employment_details.notes` exists, not on the form | UI + payload |
| G25 | "Onboarding Checklist (Optional)" tick boxes | — | **Drop**: they are not saved anywhere (static UI) (D9) |

### Step 4 and page-level
| # | Item | Today | Needed |
| :-- | :-- | :-- | :-- |
| G26 | Review cards with **Edit** links to the step + confirmation tick box | Drawer has a review step | New review layout; Edit jumps to the step and keeps all values |
| G27 | **Save as Draft** | Not there | D7: DB `employee_drafts` (`id, companyId, createdBy, payloadJson, updatedAt`) + list/resume, or drop |
| G28 | Shared stepper documented in `docs/ui.md` | Not documented | Add §12 "Stepper" to `docs/ui.md` |
| G29 | Page route, menu, permission | Menu row hidden, no route | `app/(dashboard)/employees/add/page.tsx`; make `menu-emp-add` visible; reuse `employees.create` |

## 4. Decisions needed from you (Step 0)

| # | Question | My recommendation |
| :-- | :-- | :-- |
| D1 | Login username: employee code (today's rule) or work email? | **Employee code** stays the default. Show "Use work email as username" as an option only if you want it |
| D2 | Where is the photo URL stored? | New `employees.photoUrl` (works even without a login); the topbar avatar reads it through the user's employee |
| D3 | ID Type list (National ID, Passport, Alien ID…) from where? | A small `id_types` master per country (Masters & Configuration), not hardcoded |
| D4 | Team / Unit | **Drop for now** (no teams master). Add later with a Teams master |
| D5 | Tax Residence, Tax Status, NHIF/SHIF Category, Pension Scheme | Add the 4 columns to `salary_details`. Values come from the country's statutory masters. Payroll does not use them yet |
| D6 | Emergency & Next of Kin: one contact or several? | **New `employee_contacts` table** (several rows, type = Emergency / Next of Kin). Copy the current 3 columns in as the first row |
| D7 | Save as Draft | **Yes**, `employee_drafts` table. Drafts show on the Add Employee page ("Resume draft") and are deleted on save |
| D8 | Old 9-step drawer | Rename it as you asked. The directory's **Add Employee** button opens the new page; delete the drawer once the page is live (Step 10). Edit still uses the existing edit screens |
| D9 | Drop "Auto-fill from National ID" and the "Onboarding Checklist" tick boxes? | **Yes**: there is no ID-reading service, and the checklist is not saved anywhere |
| D10 | Salutation and Marital Status lists | Move both into a lookup master (like Gender) |

## 5. Plan (one step per fresh chat)

Every step: `tsc --noEmit` + `dotnet build` clean, DB changes in `docs/db_changes.sql`, audit rows (follow.md 20), update `docs/policies.md` + this file's status, browser test steps added to the management artifact, next-step prompt at the end.

| Step | Work | Status |
| :-- | :-- | :-- |
| 0 | You review §3–§4 and answer D1–D10 | ⏳ Waiting |
| 1 | **DB + backend base:** columns/tables from D2, D3, D5, D6, (D7), `employee_locations`; entities + EF config; `BlobPaths.Profile = "profile"` + `ProfilePhotoMaxBytes` / allowed types in appsettings; `BlobKeyBuilder.ProfilePhoto`; extend `CreateEmployeeRequest` + validator (salary/bank optional, G15) | ☐ |
| 2 | **Photo upload:** staging upload for a new employee, move on create; `POST /api/me/photo` for `/portal/me`; 1 MB and JPG/PNG check from config; audit | ☐ |
| 3 | **Page shell:** route `/employees/add`, menu visible, `modules/employees/add-employee/` (`AddEmployeeView.tsx`, `components/`, `useAddEmployeeForm.ts`), shared `components/ui/Stepper.tsx` (from `UserWizardStepper`, which then uses it), shared `SearchSelect.tsx`, footer (Cancel · Back · Next Step / Complete), `docs/ui.md` §12 Stepper | ☐ |
| 4 | **Step 1 UI:** Basic (photo), Identification, Contact (dial codes), Address (same-as tick box); masters; field and banner validation | ☐ |
| 5 | **Step 2 UI:** Employment Details (auto confirmation date), Organisation & Position (autocomplete, company → locations multi-select chips with primary), Reporting Manager L1 → Direct HR → HRBP autocomplete, Shift autocomplete + hours, Schedule dropdown, Effective From | ☐ |
| 6 | **Step 3 UI, part A:** Salary & Banking (optional, CTC tile), Statutory & Compliance, Employee Code card | ☐ |
| 7 | **Step 3 UI, part B:** Educational Qualifications, Compliance Documents, Emergency & Next of Kin, User Account Creation (toggle, role, password mode, activation, force-change tick box), Notes | ☐ |
| 8 | **Step 4:** Review cards + Edit links + confirmation; submit, upload documents, show credentials panel; backend saves locations, contacts, login options; audit | ☐ |
| 9 | **Save as Draft** (if D7 = yes): save / resume / delete | ☐ |
| 10 | **Wire-up and clean-up:** directory Add Employee → page, rename/retire the drawer (D8), end-to-end test as HR on a throwaway DB clone, docs | ☐ |

## 6. Code corrections and static values

> Code correction and static values

- `CreateEmployeeModal.tsx:1085–1088`: salutations `Dr./Mr./Mrs.…` are hardcoded `<option>`s → master (D10).
- `CreateEmployeeModal.tsx:1179–1180` (and around): marital statuses hardcoded → master (D10).
- `CreateEmployeeModal.tsx:133`: `contractType` defaults to `"PERMANENT"`; `:197` `paymentMethod` defaults to `"BANK_TRANSFER"`; notice period defaults to `30` → take defaults from the master / company settings, or leave empty.
- `CreateEmployeeModal.tsx`: 0 `standered-*` classes and 109 `rounded-lg/xl/2xl`. It does not follow `docs/ui.md`. The new page is built to `ui.md`, and the drawer is retired (D8).
- `CreateEmployeeModal.tsx:717 / 889 / 901`: step guards mix `< 8`, `=== 8` and `< 9`. Not carried over.
- Mock-ups show `+254`, `KES`, `Kenya (KE)`, `NSSF` as values. On the page all of these come from the company's country, currency and statutory masters, never from code.
- `EmployeeAccountService.AddLoginAsync`: role, `ForcePasswordChange` and `IsActive` are fixed in code. They become request options (G22) with the current values as defaults.
