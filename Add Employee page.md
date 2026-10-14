# Add Employee page (4-step wizard)

Branch: `add-new-emp-page` · Started 2026-10-14 · Follows `docs/follow.md` and `docs/ui.md`. Management artifact: https://claude.ai/artifact/FR3XhYGHenVQ49uk1rH9V4 (source `docs/artifacts/add-employee-page.html`; progress, how to operate, manual test checklist).

**Status: Steps 0–6 done 2026-10-14. Next: Step 7 (Step 3 UI, part B: Education, Compliance Documents, Emergency & Next of Kin, User Account, Notes).** DB + backend base, photo upload, the 4-step page, its steps 1 (Personal & Contact) and 2 (Employment & Organisation) and the first half of step 3 (Salary & Banking, Statutory, Employee Code) are built; nothing is saved from the page yet (Step 8).

## 1. Goal

Replace the 9-step drawer **"Register New Healthcare Employee"** (`modules/employees/components/CreateEmployeeModal.tsx`, opened from `/employeedirectory`) with a full page **Employee Management → Add Employee** at `/employees/add`. The page has exactly **four steps**:

| # | Step | Sub-text | Content |
| :-- | :-- | :-- | :-- |
| 1 | Personal & Contact | Basic information | Photo, Basic Information, Identification Details, Contact Information, Address Information |
| 2 | Employment & Organisation | Job, department & location | Employment Details, Organisation & Position, Role, Reporting & Team, Shift & Work Schedule |
| 3 | Pay, Documents & Access | Salary, documents & user account | Salary & Banking, Statutory & Compliance, Employee Code, Educational Qualifications, Compliance Documents, Emergency & Next of Kin, User Account Creation, Additional Information |
| 4 | Review & Confirm | Verify and complete | Read-only cards with **Edit** links, a confirmation tick box, then **Complete Onboarding** |

The existing drawer is renamed **"Register Employee & their user access account"** and stays until both are compared side by side (§4, D8).

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
| Menu | `menus` row `menu-emp-add` "Add Employee" → `/employees/add`, `employees.create` | Visible since Step 3 |

## 3. Gap analysis: what is missing

Legend: **DB** = new column/table (script goes in `docs/db_changes.sql`), **API** = backend change, **UI** = frontend only.

### Step 1: Personal & Contact
| # | Item (from your spec / mock-up) | Today | Needed |
| :-- | :-- | :-- | :-- |
| G1 | **Upload Photo**, JPG/PNG, max **1 MB** | No photo upload anywhere. `users.avatarUrl` exists but nothing writes it | DB new column `employees.avatar` (stores the blob URL, D2); appsettings `BlobPaths.Profile = "profile"`; `BlobKeyBuilder.ProfilePhoto` → `{companyCode}/people/{employeeNumber_name}/profile/photo_{ticks}.{ext}`; upload to staging before save, then move on create; API size and type check from appsettings (no hardcoded 1 MB) |
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
| G11 | ~~Team / Unit~~ | No teams master or column | **Removed** (D4). Add later with a Teams master |
| G12 | Assigned Work Shift as **autocomplete**; Work Schedule Policy as **dropdown**; Working Hours per Week (read-only, from shift) | Selects; hours not shown | UI. Hours come from the shift master |
| G13 | Shift **Effective From** date | Always = Date of Joining | API accepts `shiftEffectiveFrom` (default joining date, not earlier) |
| G14 | Expected Confirmation Date auto = joining + probation | Partly (entity `ApplyProbation`) | UI shows the calculated date, read-only |

### Step 3: Pay, Documents & Access
| # | Item | Today | Needed |
| :-- | :-- | :-- | :-- |
| G15 | Salary & bank fields **all optional** | API/validator treats basic salary, bank etc. as required in places | Relax `EmployeeCreateValidator` (keep "if bank chosen, account number required" style rules) |
| G16 | Monthly CTC (estimated) tile | Not there | UI: base salary; currency from currency master |
| G17 | ~~Tax Residence Country, Tax Status, NHIF/SHIF Category, Pension Scheme~~ | No columns, no masters | **Removed** (D5). The "Statutory & Compliance" card shows only the existing KRA PIN / NSSF / SHIF numbers (`salary_details`) |
| G18 | ~~"Statutory deductions will be calculated as per Kenya configuration"~~ | — | **Removed** with G17 |
| G19 | **Employee Code** card (series / manual + preview) | Exists in drawer step 7 | Move as-is |
| G20 | **Educational Qualifications** + **Compliance Documents** | Exist in drawer step 8 | Move as-is, restyle to `docs/ui.md` |
| G21 | **Emergency & Next of Kin** | One contact: `employees.emergencyContactName / emergencyContactPhone / emergencyContactRel`, read by My Profile, employee edit, Employee Reports and Onboarding. No next of kin anywhere | D6: new `employee_contacts` table (type Emergency / Next of Kin, name, relation, phone, email, address, isPrimary). The primary emergency contact is also written to the 3 old columns, so existing screens and reports keep working |
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

## 4. Decisions (Step 0, answered 2026-10-14)

| # | Question | Decision |
| :-- | :-- | :-- |
| D1 | Login username | **Employee code** stays the default. "Use work email as username" is an option on the form |
| D2 | Where the photo URL is stored | New column **`employees.avatar`** holds the blob URL |
| D3 | ID Type list | New **`id_types` master per country** under Masters & Configuration |
| D4 | Team / Unit | **Removed** |
| D5 | Tax Residence, Tax Status, NHIF/SHIF Category, Pension Scheme | **Removed**: no columns or masters exist |
| D6 | Emergency & Next of Kin | **Confirmed 2026-10-14:** `employee_contacts` table (several rows), the primary emergency contact kept in sync with the 3 old `employees` columns |
| D7 | Save as Draft | **Yes**, `employee_drafts` table; "Resume draft" on the page, deleted on save |
| D8 | Old 9-step drawer | **Rename now, keep it.** Remove later, after you compare both side by side |
| D9 | "Auto-fill from National ID" and the "Onboarding Checklist" | **Dropped** |
| D10 | Salutation and Marital Status lists | **Lookup master** (like Gender) |

## 5. Plan (one step per fresh chat)

Every step: `tsc --noEmit` + `dotnet build` clean, DB changes in `docs/db_changes.sql`, audit rows (follow.md 20), update `docs/policies.md` + this file's status, browser test steps added to the management artifact, next-step prompt at the end.

| Step | Work | Status |
| :-- | :-- | :-- |
| 0 | You review §3–§4 and answer D1–D10 | ✅ Done 2026-10-14 |
| 1 | **DB + backend base:** `employees.avatar` + ID Type / Country of Issue / Issue / Expiry columns, `id_types` master (+ Masters screen), salutation + marital-status lookups, `employee_contacts` (D6), `employee_drafts`, `employee_locations`; entities + EF config; `BlobPaths.Profile = "profile"` + `ProfilePhotoMaxBytes` / allowed types in appsettings; `BlobKeyBuilder.ProfilePhoto`; extend `CreateEmployeeRequest` + validator (salary/bank optional, G15) | ✅ Done 2026-10-14 (§7) |
| 2 | **Photo upload:** staging upload for a new employee, move on create; `POST /api/me/photo` for `/portal/me`; 1 MB and JPG/PNG check from config; audit | ✅ Done 2026-10-14 (§8) |
| 3 | **Page shell:** route `/employees/add`, menu visible, `modules/employees/add-employee/` (`AddEmployeeView.tsx`, `components/`, `useAddEmployeeForm.ts`), shared `components/ui/Stepper.tsx` (from `UserWizardStepper`, which then uses it), shared `SearchSelect.tsx`, footer (Cancel · Back · Next Step / Complete), `docs/ui.md` §12 Stepper | ✅ Done 2026-10-14 (§9) |
| 4 | **Step 1 UI:** Basic (photo), Identification, Contact (dial codes), Address (same-as tick box); masters; field and banner validation | ✅ Done 2026-10-14 (§10) |
| 5 | **Step 2 UI:** Employment Details (auto confirmation date), Organisation & Position (autocomplete, company → locations multi-select chips with primary), Reporting Manager L1 → Direct HR → HRBP autocomplete, Shift autocomplete + hours, Schedule dropdown, Effective From | ✅ Done 2026-10-14 (§11) |
| 6 | **Step 3 UI, part A:** Salary & Banking (optional, CTC tile), statutory numbers (KRA PIN / NSSF / SHIF), Employee Code card | ✅ Done 2026-10-14 (§12) |
| 7 | **Step 3 UI, part B:** Educational Qualifications, Compliance Documents, Emergency & Next of Kin, User Account Creation (toggle, role, password mode, activation, force-change tick box), Notes | ☐ |
| 8 | **Step 4:** Review cards + Edit links + confirmation; submit, upload documents, show credentials panel; backend saves locations, contacts, login options; audit | ☐ |
| 9 | **Save as Draft**: save / resume / delete | ☐ |
| 10 | **Wire-up:** directory Add Employee → page, rename the drawer (kept until you compare both, D8), end-to-end test as HR on a throwaway DB clone, docs | ☐ |

## 6. Code corrections and static values

> Code correction and static values

- `CreateEmployeeModal.tsx:1085–1088`: salutations `Dr./Mr./Mrs.…` are hardcoded `<option>`s → master (D10).
- `CreateEmployeeModal.tsx:1179–1180` (and around): marital statuses hardcoded → master (D10).
- `CreateEmployeeModal.tsx:133`: `contractType` defaults to `"PERMANENT"`; `:197` `paymentMethod` defaults to `"BANK_TRANSFER"`; notice period defaults to `30` → take defaults from the master / company settings, or leave empty.
- `CreateEmployeeModal.tsx`: 0 `standered-*` classes and 109 `rounded-lg/xl/2xl`. It does not follow `docs/ui.md`. The new page is built to `ui.md`; the drawer is renamed and kept for comparison (D8).
- `CreateEmployeeModal.tsx:717 / 889 / 901`: step guards mix `< 8`, `=== 8` and `< 9`. Not carried over.
- Mock-ups show `+254`, `KES`, `Kenya (KE)`, `NSSF` as values. On the page all of these come from the company's country, currency and statutory masters, never from code.
- `EmployeeService.Create.cs:216 / 218`: a blank account type / payment method is saved as `"SALARY"` / `"BANK_TRANSFER"` (also the entity and DTO defaults). The page sends nothing when they are left empty, so these fallbacks still apply — decide in Step 8 whether a blank stays blank or uses a lookup flagged as default.
- `useCurrencies` (`modules/currency-master/useCurrencies.ts`) starts from `DEFAULT_FALLBACK_CURRENCIES` (static list). The new page does not use it; it calls `currenciesApi.getCurrencies` directly.
- `employeesApi.checkEmployeeCodeAvailability` returns `false` on any error, so a network failure shows "already used".
- `EmployeeAssignmentForm.tsx:874–890`: payment methods are a hardcoded array; they can now read the `PAYMENT_METHOD` lookup.
- `EmployeeAccountService.AddLoginAsync`: role, `ForcePasswordChange` and `IsActive` are fixed in code. They become request options (G22) with the current values as defaults.

## 7. Step 1 result (2026-10-14)

- **SQL** (`docs/db_changes.sql`, "ADD EMPLOYEE PAGE — Step 1", applied locally, re-run safe): `employees` + `avatar`, `idTypeId`, `idIssueCountryId` (nvarchar(50), like `countries.id`), `idIssueDate`, `idExpiryDate`; new `id_types`, `lookup_values`, `employee_contacts`, `employee_locations`, `employee_drafts`; menu `menu-mst-employee-lookups` + permissions `employee.lookups.view/.manage`; audit types added to the EMPLOYEES switch. Back-fill: existing emergency contacts → `employee_contacts`; current location → `employee_locations` (primary).
- **Entities / EF**: `IdType`, `LookupValue` (+ `LookupCategories`), `EmployeeContact` (+ `EmployeeContactTypes`), `EmployeeLocation`, `EmployeeDraft`; company + access filters on contacts / locations / drafts.
- **Blob + config**: `BlobPaths.Profile`, `BlobKeyBuilder.ProfilePhoto` / `IsProfilePhotoOf`, `ProfilePhotoSettings` (appsettings `ProfilePhoto`, no fallbacks).
- **Create API**: `CreateEmployeeRequest` + `IdTypeId`, `IdIssueCountryId`, `IdIssueDate`, `IdExpiryDate`, `AdditionalLocationIds`, `Contacts`, `Notes`, `ShiftEffectiveFrom`; these are already **saved** (so Step 8 only adds the login options). Validator: salary optional (G15), ID dates, shift start, contacts; pick lists checked on Add Employee only (`CheckPickListsAsync`).
- **D6 sync**: `EmployeeContactSync` used by create, Edit Employee, My Profile and the update upload.
- **Masters → Employee Lookups** screen (`modules/employee-lookups`), `GET /api/employees/form-lookups` (+ salutations, marital statuses), `GET /api/employees/id-types`.
- **Static values removed**: salutation / marital `<option>`s in `CreateEmployeeModal.tsx` and the arrays in `EmployeeEditTab.tsx` now come from `lookup_values`.
- **Checks**: `dotnet build` clean, 513 tests pass (new `AddEmployeeBaseTests`, photo key test), `tsc --noEmit` clean, new files lint clean.
- **Not done here**: employee detail / My Profile do not return the new fields yet (Steps 2 and 4); relation lists for contacts stay free text (`RELATIONSHIPS` suggestions in `EmployeeEditTab.tsx` are still hardcoded — candidate for a third lookup list); API smoke test against the live DB was not possible (the documented superadmin / rakesh passwords do not work locally).

## 8. Step 2 result (2026-10-14)

- **No SQL.** `employees.avatar` (Step 1) holds the blob key; the API only returns `/api/employees/{id}/photo?v={file}`.
- **Backend** `EmployeeService.Photo.cs`: `UploadStagedPhotoAsync` (`BlobKeyBuilder.Staging`), `AddStagedPhotoAsync` on create (copies to `BlobKeyBuilder.ProfilePhoto`, refuses a key that is not this user's staging), `UploadMyPhotoAsync` (own record only), `OpenPhotoAsync`, `IsOwnRecordAsync`. Limits from `ProfilePhotoSettings` (appsettings `ProfilePhoto`) plus `ContentMatches` (JPEG / PNG first bytes).
- **API**: `GET /api/employees/photo-rules`, `POST /api/employees/upload-photo` (`employees.create`, returns `key`), `GET /api/employees/{id}/photo` (self or `employees.view` + scope), `POST /api/me/photo`. `CreateEmployeeRequest.PhotoKey`; `EmployeeDetailDto.AvatarUrl`; My Profile `hero.avatarUrl`.
- **Audit**: `EMPLOYEE_PHOTO_STAGED`, `EMPLOYEE_PHOTO_UPLOADED`, `EMPLOYEE_PHOTO_REPLACED` (old → new key).
- **UI**: `/portal/me` header camera button now uploads the photo (it used to open Edit Personal; that drawer is still reached from the cards). Shown with `SecureImage`; hint and pre-check from `photo-rules`; errors under the photo. `employeesApi.uploadProfilePhoto` / `getPhotoRules` are ready for the Add Employee page (Step 4).
- **Checks**: `dotnet build` clean, 515 tests pass (+2: content sniff, URL never shows the key), `tsc --noEmit` clean, changed files lint clean (2 old unused-icon warnings in `ProfileHeroCard.tsx`).
- **Not done here**: no photo on the old 9-step drawer (the new page gets it in Step 4); staged files are not cleaned up (same as certificates); the employee directory list still returns an empty `avatarUrl` (one request per row would be needed).

## 9. Step 3 result (2026-10-14)

- **SQL** (`docs/db_changes.sql`, "ADD EMPLOYEE PAGE — Step 3", applied locally, re-run safe): `menu-emp-add` made visible; inserted (sort 4, `perm-emp-02` = `employees.create`) when the database never had the seed row — the local DB did not have it.
- **Route** `app/(dashboard)/employees/add/page.tsx` behind `PermissionGate employees.create`; `AddEmployeeView` exported from `modules/employees`.
- **Module** `modules/employees/add-employee/`: `AddEmployeeView.tsx` (header, stepper, error banner, step body, footer), `useAddEmployeeForm.ts` (steps list, one `form` object + `setField`, `validateStep` per step — empty for now, `next` / `back` / `goTo`, scroll to top), `components/AddEmployeeFooter.tsx` (sticky: Cancel · Back · Next Step / Complete Onboarding, Complete disabled until `onComplete` is passed in Step 8), `components/FormSection.tsx` (section card, §4), `StepPersonalContact` / `StepEmploymentOrg` / `StepPayDocsAccess` / `StepReviewConfirm` (section cards only).
- **Shared UI**: `components/ui/Stepper.tsx` (copied from `UserWizardStepper`; `/user/create` now uses it and the old file is deleted), `components/ui/SearchSelect.tsx` (client-side filter, keyboard, clearable). `docs/ui.md` §12 Stepper (+ §12.3 SearchSelect).
- **Checks**: `tsc --noEmit` clean; new files lint clean. `CreateUserWizardView.tsx` keeps its 2 old lint errors (set-state-in-effect line 94, `any` line 187) and 1 unused-import warning — not touched.
- **Not done here**: no fields, no masters loaded, Complete disabled; the sidebar may need a fresh login to show the new menu item.

## 10. Step 4 result (2026-10-14)

- **No SQL, no backend change.** Uses the Step 1–2 APIs: `GET /api/employees/form-lookups`, `GET /api/employees/id-types?countryId=`, `GET /api/employees/photo-rules`, `POST /api/employees/upload-photo`, `GET /api/org/countries`, `GET /api/companies/{id}`.
- **Frontend** `modules/employees/add-employee/`: `usePersonalMasters.ts` (countries, form lookups, photo rules, company → `homeCountry`; failed lists shown in an amber banner), `components/StepPersonalContact.tsx` (4 cards: Basic, Identification, Contact, Address), `components/PhotoUpload.tsx` (pre-check size / type from photo-rules, stages the file, local preview, Remove), `components/PhoneInput.tsx` (dial code from `countries.dialCode`, one entry per code).
- **Form hook**: `AddEmployeeForm` = payload + screen-only values `phoneDialCode`, `alternatePhoneDialCode`, `postalSameAsResidential`, `photoPreviewUrl`, `photoFileName`, `idTypeHasExpiry`, `idTypesAvailable`. `validateStep` now returns field errors; Next shows the banner "Please fix N fields marked below" and a message under each field; `setField` clears that field's error. `joinPhone(dial, number)` and `todayIso()` exported for Step 8 (phone is saved as `"+254 712…"`, the drawer's format).
- **Step 1 rules** (mirror `EmployeeCreateValidator`): first / last name, gender, date of birth (past), nationality, country of issue, ID number, phone (7–15 digits), personal email required; ID type required when the country has ID types; expiry required when `id_types.hasExpiry`; issue not in the future; expiry after issue; work email / alternate phone checked when filled.
- **Defaults**: dial codes, nationality and country of issue = the company's country (matched by name, as in the drawer). Changing the country of issue clears the ID type.
- **Types / API**: `CreateEmployeePayload` + `idTypeId`, `idIssueCountryId`, `idIssueDate`, `idExpiryDate`; `IdTypeOption`; `employeesApi.getIdTypes`.
- **Checks**: `tsc --noEmit` clean, `add-employee/` lint clean. Browser checks 4.1–4.16 in the artifact; not run here.
- **Not done here**: duplicate national ID / email is only caught on save (Step 8); no minimum age rule (no master for it); salutation is optional (the API does not require it).

## 11. Step 5 result (2026-10-14)

- **No SQL, no backend change.** Uses the existing master APIs (employment types, departments, locations, job titles, grades, cost centres by company; `/api/shifts`, `/api/shifts/schedules`; `/employees/managers`, `/employees/hr-staff`, `/employees/hrbp-staff`) and the Step 1 request fields `additionalLocationIds` + `shiftEffectiveFrom`.
- **Frontend** `modules/employees/add-employee/`: `useEmploymentMasters.ts` (all Step 2 lists, active only, locations of the active company; failed lists join the amber banner), `components/StepEmploymentOrg.tsx` (4 cards), `components/LocationPicker.tsx` (tick boxes + search, chips, star = primary → `locationId`, rest → `additionalLocationIds`; removing the primary promotes the next), `components/FormField.tsx` (the `Field` helper moved out of `StepPersonalContact` so both steps share it). `usePersonalMasters` now also returns `companyName`.
- **Fields**: Date of Joining; Employment Type (dropdown; fills probation / notice from `employment_types.defaultProbationDays / defaultNoticeDays`); Contract Type (dropdown from form lookups, no default — §6 `"PERMANENT"` not carried over); Contract End Date; Probation / Notice days; Expected Confirmation Date read-only = joining + probation (`addDaysIso`, stored in `confirmationDate`); Company read-only; Work Station(s); Department / Designation / Job Grade / Cost Centre as `SearchSelect`; Reporting Manager L1 → Direct HR → HRBP via `PersonnelAutocomplete` (its `search` prop filters the company-scoped lists the validator accepts); Shift `SearchSelect` + read-only weekly hours (`shifts.weeklyHours`, hours per day + times as help); Work Schedule dropdown; Shift Effective From (defaults to and follows the joining date, `min` = joining date).
- **Step 2 rules** (`employmentErrors`, mirror `EmployeeCreateValidator`): joining date required and after date of birth; employment type, contract type, primary location, department, designation, grade, cost centre required; contract end ≥ joining; probation / notice whole days ≥ 0; shift effective from ≥ joining. Contract type is required on the page (the drawer always sent one).
- **Types**: `CreateEmployeePayload` + `additionalLocationIds`, `shiftEffectiveFrom`.
- **Removed**: the empty "Role" card from Step 3's shell — there is no job-role field; the login's user role is Step 7 (G22).
- **Checks**: `tsc --noEmit` clean, `add-employee/` lint clean. Browser checks 5.1–5.17 in the artifact; not run here.
- **Not done here**: `PersonnelAutocomplete` keeps its own (purple, `rounded-lg`) look — restyling it to `docs/ui.md` touches the Assignment screens; reporting manager is not filtered by the chosen department; contract end date is not required per contract type (no master flag for it).

## 12. Step 6 result (2026-10-14)

- **SQL** (`docs/db_changes.sql`, "ADD EMPLOYEE PAGE — Step 6", applied locally, re-run safe): `CK_lv_category` widened; `lookup_values` + `PAYMENT_METHOD` (3) and `BANK_ACCOUNT_TYPE` (6) with the codes already stored in `bank_details`; `countries` + `socialSecurityLabel`, `healthInsuranceLabel` (Kenya: NSSF Number / SHIF Number). **Run it before deploying the API**: the countries list returns empty when a column is missing.
- **Backend**: `Country.SocialSecurityLabel / HealthInsuranceLabel` (DTO, save, list); `LookupCategories.PaymentMethod / BankAccountType` (so Masters → Employee Lookups can manage them); `GET /api/employees/form-lookups` + `paymentMethods`, `bankAccountTypes`; `CheckPickListsAsync` refuses a payment method / account type not in the list.
- **Frontend** `modules/employees/add-employee/`: `usePayMasters.ts` (active company currencies, active banks; `useBankBranches(bankId)`), `components/StepPayDocsAccess.tsx` (Salary & Banking with the CTC tile, Statutory & Compliance), `components/EmployeeCodeCard.tsx` (series preview + Refresh, manual code with a debounced availability check). `usePersonalMasters` returns `companyCurrency`; the view defaults the currency to it. Masters → Countries drawer has the two new label fields; Employee Lookups has the two new tabs.
- **Form hook**: screen-only `codeMode`, `codeAvailable`; `payErrors` for step 3 (basic salary ≥ 0, salary → currency, bank ↔ account number, manual code filled / available).
- **Checks**: `dotnet build` clean, 515 tests pass, `tsc --noEmit` clean, `add-employee/` + `employee-lookups/` lint clean (`CountryDrawer.tsx` keeps its 2 old errors, lines 16 and 46). Browser checks 6.1–6.16 in the artifact; not run here.
- **Not done here**: mobile money number / provider are not on the page (the drawer did not show them either); IBAN is not checked against the prefix; bank list is not filtered by the company's country (the drawer showed all banks too, with the country code as hint).
