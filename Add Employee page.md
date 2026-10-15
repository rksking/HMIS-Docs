# Add Employee page (4-step wizard)

Branch: `add-new-emp-page` · Started 2026-10-14 · Follows `docs/follow.md` and `docs/ui.md`. Management artifact: https://claude.ai/artifact/FR3XhYGHenVQ49uk1rH9V4 (source `docs/artifacts/add-employee-page.html`; progress, how to operate, manual test checklist).

**Status: Steps 0–10 done 2026-10-15, HR API run done on a throwaway copy (§19). Open: your answer on Q7 (HR cannot upload compliance documents), then Steps 11–14 (§20). Old drawer stays until you say (Q1). See §16–§20.** The 4-step page is complete, saves the employee, their login (with the step 3 options), contacts, locations and compliance documents, and keeps unfinished forms as personal drafts.

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
| 7 | **Step 3 UI, part B:** Educational Qualifications, Compliance Documents, Emergency & Next of Kin, User Account Creation (toggle, role, password mode, activation, force-change tick box), Notes | ✅ Done 2026-10-14 (§13) |
| 8 | **Step 4:** Review cards + Edit links + confirmation; submit, upload documents, show credentials panel; backend saves locations, contacts, login options; audit | ✅ Done 2026-10-14 (§14) |
| 9 | **Save as Draft**: save / resume / delete | ✅ Done 2026-10-14 (§15) |
| 10 | **Wire-up:** directory Add Employee → page, rename the drawer (kept until you compare both, D8), end-to-end test as HR on a throwaway DB clone, docs | ✅ Done 2026-10-15 (§16); HR browser run still open |

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
- `EmployeeAccountService.AddLoginAsync`: role, `ForcePasswordChange` and `IsActive` were fixed in code. **Done Step 8:** request options (G22) with the old values as defaults.
- `CreateEmployeeModal.tsx` `handleCertificateFileChange`: certificate size fixed at **5 MB** in code. The new page uses the company's Education document type instead (same as the API). Drawer not changed (D8).
- `EmployeeEditTab.tsx`: `RELATIONSHIPS` array removed (Step 7); suggestions now come from the `RELATIONSHIP` lookup.

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

## 13. Step 7 result (2026-10-14)

- **SQL** (`docs/db_changes.sql`, "ADD EMPLOYEE PAGE — Step 7", applied locally, re-run safe): `CK_lv_category` widened; `lookup_values` + `RELATIONSHIP` (6 values, the same ones the Edit Employee screen suggested, so existing `emergencyContactRel` rows match).
- **Backend**: `LookupCategories.Relationship` (Masters → Employee Lookups manages it); `GET /api/employees/form-lookups` + `relationships`; `CheckPickListsAsync` refuses a contact relationship not in the list; `GET /api/employees/grantable-roles` (`employees.create`) → `EmployeeAccountService.GetGrantableRolesAsync` (god-mode user: every active role; others: non-god-mode roles whose active permissions they hold all of; `isDefault` = `EmployeeAccounts:DefaultRoleCode`); validator: Notes ≤ 500 (`EmployeeCreateValidator.NotesMaxLength`).
- **Frontend** `modules/employees/add-employee/`: `useDocsAccessMasters.ts` (education lookups, document types, grantable roles, password policy; `acceptOf` / `fileProblem` for document-type file rules), `components/EducationCard.tsx`, `ComplianceDocsCard.tsx`, `ContactsCard.tsx`, `UserAccountCard.tsx`, Notes in `StepPayDocsAccess.tsx`. Types: `CreateEmployeePayload` + `contacts`, `notes`; `EmployeeContactInput`, `GrantableRole`; `employeesApi.getGrantableRoles`. Employee Lookups has a Relationship tab.
- **Form hook**: screen-only `contactRows` (with dial code), `complianceDocs` (files kept for Step 8), `createLogin`, `usernameMode`, `loginRoleId`, `passwordMode`, `manualPassword`, `confirmPassword`, `activation`, `forcePasswordChange` (defaults = today's behaviour); `useAddEmployeeForm(passwordPolicy)`; `passwordRules` / `passwordProblems`; row errors keyed `contact.<localId>.<field>`; step 3 = `payErrors` + `docsAccessErrors`.
- **Checks**: `dotnet build` clean, 516 tests pass (+1 `GrantableRoles_OnlyRolesWhosePermissionsTheCreatorHolds`), `tsc --noEmit` clean, `add-employee/`, `employee-lookups/`, `EmployeeEditTab.tsx` lint clean. Browser checks 7.1–7.18 in the artifact; not run here (API smoke test not possible: local login passwords unknown).
- **Not done here (Step 8)**: `contactRows` → `contacts` (phone joined with dial code) and `notes` on submit; compliance files uploaded after create; API options `createLogin`, `roleId`, `usernameMode`, `passwordMode`, `manualPassword`, `activation`, `forcePasswordChange` (+ the same grant rule enforced on save, audit). "Activate on joining date" needs a way to switch the login on at the joining date (job or check at sign-in) — decide in Step 8. Staged certificates are not cleaned up if the page is left.

## 14. Step 8 result (2026-10-14)

- **SQL** (`docs/db_changes.sql`, "ADD EMPLOYEE PAGE — Step 8", applied locally, re-run safe): `users.activeFrom` (datetime2 NULL). **Run before deploying the API** (EF reads it on every sign-in).
- **Decision "activate on joining date"**: no job. The login is created at once with `users.activeFrom` = joining date (00:00 UTC); `AuthService.LoginAsync` refuses sign-in before it ("Your account becomes active on …"). A joining date of today or earlier = `null` (active at once). No session can exist before that date, so refresh needs no check.
- **Backend**: `CreateEmployeeRequest` + `CreateLogin`, `RoleId`, `UsernameMode`, `PasswordMode`, `ManualPassword`, `Activation`, `ForcePasswordChange` (values in `LoginChoices`; defaults = the old behaviour, so the bulk upload and onboarding are unchanged). `EmployeeLoginOptions` + `EmployeeAccountService.AddLoginAsync(employee, options)` (role, username = code or work email, typed or random password, `ActiveFrom`, forced change). `CheckLoginOptionsAsync` on save and in `ValidateNewEmployeeAsync`: unknown option values, role not in `GetGrantableRolesAsync` of the signed-in user, work email missing / already a username, manual password against `PasswordPolicy` (shared `PasswordPolicyRules`, also used by change-password). `EmployeeService.Create.cs`: `SaveNewEmployeesAsync` returns employee + optional login (no login when `CreateLogin` is off); `CreateEmployeesAsync` (bulk) unchanged in behaviour. `EmployeeCredentialDto` + `RoleName`, `PasswordSetManually`, `ForcePasswordChange`, `ActiveFrom`. A typed password is never emailed; the email mentions the activation date and only asks for a change when forced.
- **Audit**: `LOGIN_CREATED` (`User`: employee code, username, username option, role, password option, activation, active-from, forced change; never the password; Users & Roles switch) or `LOGIN_NOT_CREATED` (`Employee`, Employees switch) for every new employee, bulk upload included.
- **Frontend** `modules/employees/add-employee/`: `buildCreatePayload.ts` (phones joined with `joinPhone`, `contactRows` → `contacts`, login options mapped, screen-only fields dropped, `employeeCode` only in manual mode), `useCompleteOnboarding.ts` (POST, then compliance files via `documentsApi.uploadDocumentFile` with number / issue / expiry; failures listed; refusals as a list), `components/StepReviewConfirm.tsx` (5 cards with Edit, confirmation tick box), `components/OnboardingComplete.tsx` (result, failed uploads, `CredentialsPanel`, View Employee / Add Another). Form hook: `checkAllSteps` (all steps again before saving), `returnToReview` + footer **Back to Review** after an Edit, screen-only `idTypeName`. `FormSection` has an `action` slot.
- **Checks**: `dotnet build` clean, 518 tests pass (+2: login options refused / applied), `tsc --noEmit` clean, `add-employee/` lint clean. Browser checks 8.1–8.20 in the artifact; not run here (local login passwords unknown).
- **Not done here**: a blank account type / payment method is still saved as `SALARY` / `BANK_TRANSFER` (§6, request defaults); `activeFrom` uses UTC midnight, not the company time zone (Kenya: from 03:00 local); a login waiting for its date still shows as Active on user screens; the personal email must still be unused by any login even when no login is created (bulk validator rule); notice period left empty is saved as 30 (request default).

## 15. Step 9 result (2026-10-14)

- **No SQL.** Uses `employee_drafts` from Step 1 (`title` = name for the list; `payloadJson` = `{ step, form }`; `updatedAt`). The audit switch already has `EmployeeDraft` under EMPLOYEES.
- **Backend**: `IEmployeeDraftService` / `EmployeeDraftService` (list, get, create, update, delete), always filtered on the active company **and** `createdBy` = the signed-in user (also for developers, whose company filter is lifted). `BuildPayload` strips `manualPassword`, `confirmPassword`, `photoPreviewUrl` and every `complianceDocs[].file`. `EmployeeDraftsController` `api/employees/drafts` (`employees.create` on all): `GET`, `GET {id}`, `POST`, `PUT {id}`, `DELETE {id}`; another user's id = 404. DTOs `SaveEmployeeDraftRequest` (title, step, form), `EmployeeDraftSummaryDto`, `EmployeeDraftDto`.
- **Audit**: `EMPLOYEE_DRAFT_SAVED` (title + step; on update old → new title / step) and `EMPLOYEE_DRAFT_DELETED`, entity `EmployeeDraft` (Employees switch). The form JSON is not written to the audit log.
- **Frontend** `modules/employees/add-employee/`: `useEmployeeDrafts.ts` (list for the company, `save` = POST first then PUT the same draft, `open`, `remove`, `discardOpenDraft`; strips the same values before sending), `components/ResumeDraftList.tsx` (card above the stepper: name, step, last saved, Resume / Open badge, delete; hidden when empty), footer **Save as Draft** (`standered-btn-neutral`, left next to Cancel, no checks) + "Draft saved …". Form hook `loadDraft(form, step)`; Resume asks before replacing the page and resets the confirmation tick. `useCompleteOnboarding.complete` returns `true` on save; then the open draft is deleted (a failed delete never undoes the save).
- **Files in a draft**: compliance rows keep their details; `!file` = **Re-attach needed** badge + **Re-attach** button (same file rules as the add form); `docsAccessErrors` blocks step 3 until every row has its file or is removed. The staged photo key and staged education certificates are kept (already on blob); `PhotoUpload` shows Change / Remove when only the file name is known.
- **Checks**: `dotnet build` clean, 520 tests pass (+2 `EmployeeDraftTests`: never-stored values, creator + company only, audit rows), `tsc --noEmit` clean, `add-employee/` lint clean (`api.ts` / `types.ts` keep their old lint errors, none in the new lines). Browser checks 9.1–9.12 in the artifact; not run here.
- **Not done here**: drafts never expire and there is no limit per user; switching company in the top bar does not close an open draft (saving again then makes a new draft in the new company); staged photo / certificate files of a deleted draft are not cleaned up (same as Steps 2 and 7).

## 16. Step 10 result (2026-10-15)

- **No SQL, no backend change.**
- **Employee Directory** (`modules/employee-directory/EmployeeDirectoryView.tsx`): **Add Employee** now goes to `/employees/add` (`router.push`). A new light button **Register (old form)** opens the old drawer (D8). Both buttons show only with `employees.create` (before, Add Employee showed for everyone and the save was refused later).
- **Old drawer** (`modules/employees/components/CreateEmployeeModal.tsx`): title renamed to **"Register Employee & their user access account"**. Nothing else changed; subtitle still "9-step complete workforce onboarding wizard".
- **Not changed**: `/employees` (`EmployeesView.tsx`) still opens the old drawer from its own "+ Add Employee" button (`PermissionGate EMP_CREATE`, old styling).
- **End-to-end check (throwaway clone `HRMSCore_StepTest`, dropped afterwards; shared DB never written)**: `dotnet build` clean, 520 tests pass, `tsc --noEmit` clean, directory file lint clean (1 old `<img>` warning). On the clone, read-only: all 5 new tables, `users.activeFrom`, the 5 new `employees` columns, the 2 country label columns, lookup lists (SALUTATION 6, MARITAL_STATUS 4, PAYMENT_METHOD 3, BANK_ACCOUNT_TYPE 6, RELATIONSHIP 6), `menu-emp-add` visible, role `company_hr` has `employees.create`, 0 drafts.
- **Not done: the HR run through the API** (drafts, 4 steps, save, login options, uploads, audit rows). The HR password is not known locally and setting one on the clone was blocked by the tool's safety check. The full run is written as browser checks **10.1–10.20** in the artifact.
- **Note**: SQL Server (Docker) was stopped when this step ran; Docker Desktop and the `sqlserver` container were started for the clone and left running. Your API (:5197) and dev server (:4000) were not running and were not started.

## 17. Remaining follow-ups (from §7–§16)

Grouped; none of these block using the page.

| # | Follow-up | From |
| :-- | :-- | :-- |
| F1 | ~~HR API run~~ **done** (§19). Browser-only checks (4.x–9.x, 10.1–10.4, 10.6–10.10, 10.19–10.20) still for you | §10–§16 |
| F2 | Remove `CreateEmployeeModal.tsx` and the "Register (old form)" button **when you say so** (Q1). `/employees` already points to the page (Q2, done) | §16, §19 |
| F3 | ~~Blank account type / payment method / notice period from code~~ **done** (§21): defaults now come from the masters | §6, §14 |
| F4 | `users.activeFrom` uses UTC midnight, not the company time zone (Kenya: sign-in from 03:00 local) | §14 |
| F5 | A login waiting for its active-from date still shows as **Active** on user screens | §14 |
| F6 | Personal email must be unused by any login even when no login is created (bulk validator rule) | §14 |
| F7 | Staged photo / certificate files are never cleaned up (page left, draft deleted) | §8, §13, §15 |
| F8 | Drafts never expire, no limit per user; switching company with a draft open saves a new draft in the new company | §15 |
| F9 | Employee directory list returns an empty `avatarUrl` (photo not shown in the list) | §8 |
| F10 | No photo on the old drawer (goes away with F2) | §8 |
| F11 | Duplicate national ID / email only caught on save; no minimum-age rule (no master) | §10 |
| F12 | `PersonnelAutocomplete` keeps its old purple `rounded-lg` look (not `docs/ui.md`); reporting manager not filtered by department | §11 |
| F13 | Contract end date not required per contract type (no master flag) | §11 |
| F14 | Mobile money number / provider not on the page; IBAN not checked against prefix; bank list not filtered by company country | §12 |
| F15 | `employeesApi.checkEmployeeCodeAvailability` returns "already used" on a network error | §6 |
| F16 | `EmployeeAssignmentForm.tsx` payment methods hardcoded; can read the `PAYMENT_METHOD` lookup | §6 |
| F17 | `useCurrencies` starts from a static fallback list (`DEFAULT_FALLBACK_CURRENCIES`) | §6 |
| F18 | `/employees/add` without `employees.create` shows an empty page (the gate has no message) | §16 |
| F19 | Old lint errors left untouched: `CreateUserWizardView.tsx` (2), `CountryDrawer.tsx` (2), `employees/api.ts` / `types.ts` | §9, §12, §15 |
| F20 | Edit Employee (`AssignmentService`) still defaults payment method `BANK_TRANSFER` / notice 30 in code for new rows | §21 |
| F21 | Countries and Work Stations masters write no audit rows (incl. the new time-zone fields) | §23 |

## 18. Things you should know & decisions

### Things you should know

1. **Two ways to add an employee exist right now.** Directory → **Add Employee** = new 4-step page. Directory → **Register (old form)** and `/employees` → **+ Add Employee** = old 9-step drawer. Both save real employees.
2. **The end-to-end HR run has not been done by Claude.** Build, 520 tests and the database check passed, but nothing was saved through the API as HR. Please run artifact checks 10.1–10.20 on a throwaway copy.
3. **Run the SQL before deploying the API**: Steps 1, 3, 6, 7, 8 in `docs/db_changes.sql`. Without Step 6 the countries list comes back empty; without Step 8 (`users.activeFrom`) sign-in fails.
4. **"Activate on joining date"** creates the login at once; sign-in is refused before the joining date (00:00 UTC). No job runs.
5. **A typed (manual) password is never emailed** and never stored in drafts or audit rows. HR must hand it to the employee.
6. **Drafts are personal**: only the user who saved one sees it, in the company it was saved in (developers and superadmin too). Attached compliance files are not kept in a draft and must be re-attached.
7. **Role list on the page** = only roles whose permissions the signed-in user already holds (god-mode users see all). The same rule is checked on save.
8. **Lookup lists** (salutation, marital status, payment method, bank account type, relationship) and **ID types per country** are managed under Masters & Configuration → Employee Lookups / ID Types. Nothing is hardcoded.
9. **Emergency contact sync (D6)**: the primary emergency contact is also written to the 3 old `employees.emergencyContact*` columns, so old screens and reports keep working.
10. **Primary work station** is also written to `employment_details.locationId`, so existing reports keep working.
11. **Local DB** used by the app is `HRMSCore_Local2` (appsettings), not `HRMSCore_Local`.
12. **Local passwords**: the documented `hr` / `superadmin` / `rakesh` passwords do not work locally; repeated tries raise `failedAttempts`.

### Decisions you need to make

| # | Decision | Options |
| :-- | :-- | :-- |
| Q1 | When can the old 9-step drawer be removed (D8)? | After you run check 10.19 (same employee through both forms) |
| Q2 | `/employees` "+ Add Employee": keep the old drawer until Q1, or send it to the new page now? | Now: one way in. Later: remove together with the drawer |
| Q3 | Blank account type / payment method / notice period (F3) | Keep blank, or use a lookup value flagged as default / company setting |
| Q4 | `activeFrom` time zone (F4) | Keep 00:00 UTC, or use the company time zone |
| Q5 | Clean-up of staged photos / certificates and old drafts (F7, F8) | A daily clean-up job with an age limit, or leave as is |
| Q6 | How Claude may sign in for API end-to-end tests | You give a test password for `hr` on the throwaway copy, or allow resetting it there, or keep testing in the browser only |

### Your answers (2026-10-15)

| # | Answer | What happens |
| :-- | :-- | :-- |
| Q1 | "Once I will tell" | Old drawer stays (Directory → Register (old form)) until you say remove it |
| Q2 | New page | **Done 2026-10-15**: `/employees` "+ Add Employee" (and its empty-state buttons) open `/employees/add`; the drawer is no longer mounted there |
| Q3 | OK | Use a default from the masters instead of code. **Done in Step 11** (§21). The 15 Oct reply did not correct this reading, so it was built as written |
| Q4 | Time per country; employee time zone setting; travel changes attendance | **Step 12** (design first, needs your OK): time zone on country (and location override), employee time zone + dated travel periods, `activeFrom` and attendance worked out in that zone |
| Q5 | Claude decides | **Decided: Step 13** daily clean-up job: staged photos / certificates older than **7 days** that no draft or employee uses are deleted; drafts not saved for **90 days** are deleted (audit `EMPLOYEE_DRAFT_DELETED`, reason "expired"). Both ages in appsettings, job switch on/off |
| Q6 | Test logins given | Saved in `docs/follow.md` → "API / end-to-end test logins". Used Company HR `68032` for the run in §19 |

## 19. HR end-to-end run through the API (2026-10-15)

Throwaway copy `HRMSCore_StepTest` of `HRMSCore_Local2`, test API on :5299 (process checked: clone DB, biometric sync off), signed in as Company HR **68032** (`company_hr`, MAKL). Copy dropped afterwards; the shared DB was never written.

| Check | Result |
| :-- | :-- |
| Masters for all 4 steps (lookups, ID types, org lists, shifts, schedules, managers / HR / HRBP, roles, document types, banks, photo rules, next code) | ✅ all load for HR |
| Grantable roles | ✅ `company_hr, ess, role_md, role_emp` (no god-mode roles) |
| 10.5 Draft save → update (same id) → list once; passwords, preview and files never stored | ✅ |
| Photo staged; non-image content refused (400) | ✅ |
| Role not grantable refused / weak manual password refused | ✅ "You may not give the chosen user role…" / "…at least 8 characters…" |
| 10.11 Save A: username = work email, manual password, activate on joining date, forced change | ✅ 201; role Staff Employee, active from joining date |
| Saved rows for A (DB): ID type / country / issue date, photo, 2 work stations (1 primary), emergency + next of kin, notes | ✅ |
| 10.12 Draft deleted after save | ✅ |
| 10.14 A signs in before joining date | ✅ refused: "Your account becomes active on 25 Oct 2026…" |
| 10.15 Save B: username = code, auto password, active now → B signs in, must change password | ✅ |
| 10.16 Save C with Create User Account off | ✅ no login |
| 10.18 Audit | ✅ `EMPLOYEE_DRAFT_SAVED/_DELETED`, `EMPLOYEE_PHOTO_STAGED/_UPLOADED`, `LOGIN_CREATED` (no password in it), `LOGIN_NOT_CREATED`; **`EMPLOYEE_CREATED` was missing → fixed** (below) |
| 10.8 Compliance document upload as HR | ❌ **403**: `POST /api/documents/upload-file` needs `documents.manage`; `company_hr` does not have it. Employee is still saved; the page lists the document as failed. **Q7** |
| 10.13 Employee detail API returns ID details, extra work stations, contacts, notes | ❌ not returned (saved in DB, not in `EmployeeDetailDto`). Photo is returned. **Step 14** |

**Fixed in this run**: `EmployeeService.Create.cs` → `AuditEmployeeCreated`: one `EMPLOYEE_CREATED` row (entity `Employee`, Employees switch) per new employee from the page, the old drawer and the bulk upload — code, name, joining date, employment / contract type, department, locations, job title, grade, cost centre, manager / HR / HRBP, shift, photo added, number of contacts and education rows. Salary and ID numbers are left out. `dotnet build` clean, 520 tests pass; checked on the copy (3 rows for 3 saves).

**Good to know**: the save response returns the typed password to the HR user who typed it (`credentials.temporaryPassword`), so the success panel can show it behind its eye toggle; it is never emailed or audited.

**Q7 (answered 15 Oct, done in Step 11, §21)**: HR could not upload compliance documents on Add Employee. Options: (a) give `company_hr` the `documents.manage` permission (DB script, also lets HR manage documents elsewhere); (b) let `POST /documents/upload-file` also accept `employees.create` for an employee created in the last 24 h by the same user (onboarding only); (c) hide the Compliance Documents card when the user lacks `documents.manage`.

## 20. Next steps

| Step | Work | Status |
| :-- | :-- | :-- |
| 11 | Q7 fix + Q3 defaults from masters (`lookup_values.isDefault`, notice from employment type) | ✅ 15 Oct (§21) |
| 12 | Q4 time zones: plan first (country / location / employee time zone, travel periods, `activeFrom`, attendance) — written for your review before any code | ✅ plan 15 Oct (§22); answers 15 Oct (§22.11) |
| 12a | Time-zone list, country default zone, work-station zone + country link, resolver, `activeFrom` at local midnight, Z7 script | ✅ 15 Oct (§23) |
| 12b | Employee's own time zone on Add Employee step 2 / Edit Employee / bulk upload; `activeFrom` worked out again when joining date, work station or zone changes before the login is active | ✅ 15 Oct (§24) |
| 12c | Travel periods: table, employee request + line-manager approval on /task, HR direct entry, audit | ☐ |
| 12d | Attendance in the employee's zone: SQL functions, the two procedures, web clock-in, biometric conversion, geofence rule Z5 | ☐ |
| 12e | Zone notes on ESS day drawer / register / summary + full browser checks | ☐ |
| 13 | Q5 clean-up job (staged files 7 days, drafts 90 days, appsettings, audit) | ☐ |
| 14 | Employee detail / My Profile return the new fields (ID details, extra work stations, contacts, notes) — F-item from §7 | ☐ |
| — | Remove old drawer | ☐ when you say (Q1) |

## 21. Step 11 result (2026-10-15)

**Your answers**: Q7 (a) "yes, give" + (b) "ok" + (c) "manage". Q3: the §18 reading was not corrected, so it was built as written.

| Part | What changed |
| :-- | :-- |
| Q7 (a) | `docs/db_changes.sql` block "Step 11" grants `documents.manage` to `company_hr` (re-runnable). HR must **sign in again** to get it. This also lets HR upload / verify documents on other screens |
| Q7 (b) | `POST /api/documents/upload-file` now accepts `documents.manage` **or** `employees.create` for an employee **the same user added in the last 24 h** (checked against the `EMPLOYEE_CREATED` audit row). Any other employee → 403 "You do not have permission to upload documents for this employee." (`DocumentsController`, `DocumentsService.IsNewEmployeeAddedByAsync`) |
| Q7 (c) | Read as "Claude manages it": the card is **not hidden**. With (b), everyone who can open Add Employee can upload for the employee they just saved, so hiding it would never apply |
| Q3 lookups | New column `lookup_values.isDefault` (one per list). Masters → Employee Lookups: **Default** column + "Default" tick in the drawer (only for active values; ticking one un-ticks the old one; audited). `GET /employees/form-lookups` returns `isDefault`; Add Employee step 3 pre-selects it once, only on a blank field |
| Q3 save | Blank payment method / account type → the marked default, else **blank** (was `BANK_TRANSFER` / `SALARY` in code). Blank notice → the employment type's default notice days, else 0 (was 30 in code). Same for the page, the old drawer and the bulk upload (`EmployeeService.Create.cs`) |
| Q3 data | The SQL block marks `BANK_TRANSFER` and `SALARY` as defaults (only where a list has none), so blank fields save the same as before. Change them on the screen any time |

**Checks**: `dotnet build` clean, **520/520** tests, `tsc` + eslint clean on the changed files. API run as Company HR **68032** on throwaway copy `HRMSCore_StepTest` (test API :5299, biometric sync off, files to local scratch storage, not Azure). SQL block run twice: 2nd run changed 0 rows. Copy dropped afterwards; the shared DB was not written.

| Check | Result |
| :-- | :-- |
| HR token holds `documents.manage` after the SQL | ✅ |
| `form-lookups` marks `BANK_TRANSFER` / `SALARY` as default | ✅ |
| Save with blank notice / payment / account type (employment type Intern) | ✅ saved 14 days (Intern default) / `BANK_TRANSFER` / `SALARY` |
| Save with typed values (45 / CHEQUE / SAVINGS) | ✅ kept as typed |
| 10.8 Compliance document upload as HR | ✅ 200 (was 403 in §19) |
| `documents.manage` taken off HR (copy only) → upload for the employee HR just saved | ✅ 200 (rule b) |
| Same, for an old employee | ✅ 403 |
| No default marked → blank payment method / account type saved | ✅ blank |

**Good to know**
- Rule (b) needs the `EMPLOYEE_CREATED` audit row. If the Employees switch on Audit Configuration is off, only `documents.manage` can upload.
- The Edit Employee screen (`AssignmentService`) still falls back to `BANK_TRANSFER` / 30 days in code when a new bank / job row is made there. Not part of Add Employee; new follow-up **F20**.
- Browser checks for you: artifact 11.1–11.5.

**Run before deploying the API**: `docs/db_changes.sql` block "ADD EMPLOYEE PAGE — Step 11" (EF reads `isDefault` on every lookup query).


## 22. Step 12 plan: time zones (Q4) — for your review, no code yet (2026-10-15)

Your answer to Q4: "time per country; employee time zone setting; travel changes attendance". This section is the design only. Nothing is built until you answer Z1–Z9 below.

### 22.1 What exists today

| Area | Today |
| :-- | :-- |
| Company | `companies.timeZone` (IANA, e.g. `Africa/Nairobi`, default `UTC`) + `companies.timeZoneSqlId` (filled by the API). Set on Company Setup |
| Country master | No time zone (`countries`: name, ISO, dial code, currency, labels) |
| Work station | `locations` has `country` as **text** (not a link to `countries`), lat / long / geofence radius, no time zone |
| Employee | No time zone. No travel record |
| Attendance | All times = **company** local wall time: `CompanyClock` (.NET), `dbo.fn_CompanyLocalNow` (SQL) in `sp_ProcessDailyAttendance` / `sp_GetEmployeeMonthlyAttendance`. Biometric devices send their own local time; web clock-in is converted UTC → company time (policies.md §49) |
| `users.activeFrom` | Joining date **00:00 UTC** (Kenya: sign-in from 03:00 local). "Is the joining date in the future?" also uses the UTC date |

### 22.2 The rule: which time zone applies to an employee on a date

One resolver answers "employee X on date D → time zone". First match wins:

| Order | Source | When |
| :-- | :-- | :-- |
| 1 | **Travel period** (approved) | D falls inside `fromDate … toDate` |
| 2 | **Employee time zone** | Set on the employee (remote worker, lives in another zone). Blank = go on |
| 3 | **Primary work station** time zone | Set on the location. Blank = go on |
| 4 | **Country** default time zone | Country of the work station (else the company's country). Blank = go on |
| 5 | **Company** time zone | Always set (today's behaviour) |

Because every new field starts blank, **nothing changes on the day this ships**: every employee still resolves to the company zone until someone fills a field. A country with several zones (USA, Australia) gets one default; each work station there sets its own.

Time-zone names are IANA ids (`Africa/Nairobi`). The pick list comes from the server's own time-zone list (`TimeZoneInfo`), not from code. A small table `time_zones` (IANA id, SQL Server name, display name) is filled by the API at start-up, so SQL can use `AT TIME ZONE` for any zone (same idea as `companies.timeZoneSqlId`, one place instead of a column per table).

### 22.3 Travel periods

- A dated record: employee, **from date**, **to date**, destination **work station** (optional) and/or **time zone** (one of them required; a work station gives its zone), reason, status.
- Dates are whole days. The new zone applies from 00:00 local on the from date to 24:00 local on the to date.
- Two travel periods of the same employee may not overlap. A travel period may not start before the joining date or end after the last working day.
- Changes after a day is processed: those days are **re-processed** (same as the regularisation re-process).
- Audit: `EMPLOYEE_TRAVEL_CREATED / _UPDATED / _APPROVED / _REJECTED / _CANCELLED` (Employees switch).
- Approval (if Z2 = request + approve) goes to **/task** like every other approval, through the approval role mapping.

### 22.4 `users.activeFrom`

- Saved as the **UTC moment** of joining date 00:00 in the employee's zone on that date (rules 2–5; travel is ignored before joining). Kenya, joining 25 Oct → `2026-10-24 21:00 UTC`; sign-in opens at 00:00 Nairobi.
- "Joining date in the future?" (decides whether `activeFrom` is set at all) uses today **in that zone**, not the UTC date.
- Sign-in refusal message and the welcome email show the **local date** ("active on 25 Oct 2026").
- If HR later changes the joining date, work station or employee time zone before the login is active, `activeFrom` is worked out again (Edit Employee / bulk update).
- Existing rows: one data script recomputes `activeFrom` for logins not yet active (see Z7).

### 22.5 Attendance

| Part | Change |
| :-- | :-- |
| Day and times | Attendance date and clock-in / out times = wall time **in the employee's zone for that date** (resolver). For everyone on the company zone this is today's result |
| Web clock-in | `CompanyClock` → new `EmployeeClock` (resolver + UTC now). Company clock stays for company-level things (dashboard "today", reports) |
| Biometric punches | Device sends its own local time. Device zone = its work station's zone (else company). If it differs from the employee's zone that day, the punch is converted device time → UTC → employee time. Same zone = untouched (today's case) |
| SQL | New `dbo.fn_EmployeeTimeZoneOn(@EmployeeId, @Date)` (SQL Server zone name) and `dbo.fn_EmployeeLocalNow(@EmployeeId)`. The two procedures switch from `fn_CompanyLocalNow` to these. Live definitions are dumped first (the repo copies are stale). Set-based: zone per employee-day resolved once into a temp table, not a function call per row |
| Shift times | A shift 09:00–18:00 means 09:00 **where the employee is that day** (Z3) |
| Holidays | Taken from the employee's normal holiday calendar, not the destination's (Z4) |
| Geofence | During travel, the clock-in location check uses the travel work station; travel with only a time zone → see Z5 |
| Screens | ESS day drawer, register and summary show a small "Times in Asia/Dubai · Travel" note on days not in the employee's usual zone |

### 22.6 Database changes (one block in `docs/db_changes.sql`, run before the API)

| Change | Detail |
| :-- | :-- |
| `countries.defaultTimeZone` | nvarchar(64) null |
| `locations.timeZone` | nvarchar(64) null |
| `locations.countryId` | nullable FK to `countries`, filled once from the text `country` (match on name or ISO code); unmatched rows listed by the script for HR to fix. Text column kept for old screens |
| `employees.timeZone` | nvarchar(64) null |
| `time_zones` | new: `ianaId` (PK), `sqlServerId`, `displayName`; filled / refreshed by the API at start-up |
| `employee_travel_periods` | new: id, companyId, employeeId, fromDate, toDate, locationId null, timeZone null, reason, status, approvedBy / At, created / updated audit columns. Index (employeeId, fromDate, toDate) |
| SQL functions | `fn_EmployeeTimeZoneOn`, `fn_EmployeeLocalNow` |
| Procedures | `sp_ProcessDailyAttendance`, `sp_GetEmployeeMonthlyAttendance` (only the "now / zone" lines change) |
| Permissions / menus | `employees.travel.view`, `employees.travel.manage`; `portal.travel.apply` if Z2 includes employee requests; ESS menu "My Travel" under the portal menu |
| Data | `users.activeFrom` recompute (Z7) |

### 22.7 Screens (all follow `docs/ui.md`)

| Screen | Change |
| :-- | :-- |
| Masters → Countries (`CountryDrawer`) | **Default time zone** search-select |
| Masters → Locations (`LocationDrawer`) | **Country** from the master (select) + **Time zone** (blank = "Country default: Africa/Nairobi") |
| Add Employee step 2 | Under Work Station: **Time zone** read-only line "Africa/Nairobi (from work station)" with **Change** → select (sets the employee's own zone); clear = inherit again |
| Edit Employee | Same field; Travel tab (list + Add travel drawer) for HR |
| ESS portal | "My Travel" list + New Request drawer (design like `/portal/attendance` New Request) if Z2 allows |
| /task | Travel requests for approval (if Z2) |
| Attendance (ESS, register, summary) | Zone note on travel / other-zone days |

### 22.8 Risks

| # | Risk | How it is handled |
| :-- | :-- | :-- |
| R1 | Location `country` is free text; some may not match the master | Script lists unmatched rows; resolver then skips rule 4 for them (falls to company) |
| R2 | Device has no work station | Device treated as company zone (today's behaviour) |
| R3 | Change-over day: a night shift that starts the day before travel and ends after the switch | Shift belongs to the day it **starts**; that day's zone is used for the whole shift |
| R4 | Daylight saving (not in Kenya, but in other countries): 23 h / 25 h days | Hours worked from UTC difference of in/out, not wall-clock subtraction |
| R5 | Company-wide "today" (dashboard present-today, reports) mixes zones | Stays company zone in this plan (Z6) |
| R6 | Performance of per-employee zone lookups in the monthly / register procedures | Resolve once per employee-day range into a temp table |
| R7 | Live procedures differ from the repo | Dump with `OBJECT_DEFINITION` before changing; change only the zone lines |
| R8 | Server missing a zone (Linux vs Windows names) | `time_zones` built from the server's own list; unknown id → falls back to the next rule, logged |
| R9 | Changing a past zone changes past attendance | Only travel periods re-process past days; employee / location / country zone changes apply from the next processed day (no re-process) unless HR re-processes |
| R10 | Payroll attendance already locked for a month | Travel periods inside a locked payroll month are refused |

### 22.9 Decisions you need to make

| # | Question | Options (recommended first) |
| :-- | :-- | :-- |
| Z1 | Order of rules in 22.2 OK? | **Yes** / employee zone below work station |
| Z2 | Who records travel? | **Employee requests, line manager approves on /task, HR can add directly (approved)** / HR only / employee only, no approval |
| Z3 | Shift times during travel | **Local time where the employee is** / keep home-zone times (09:00 Nairobi shown as 10:00 in Dubai) |
| Z4 | Public holidays during travel | **Home calendar** / destination country's calendar |
| Z5 | Web clock-in during travel with no work station | **No location check those days** / refuse web clock-in / normal check against the home station |
| Z6 | Dashboards and reports "today" | **Company zone (unchanged)** / each employee's zone |
| Z7 | Logins already waiting with UTC `activeFrom` | **Recompute to local midnight** / leave them |
| Z8 | Do leave, separation (resignation "today") and other "today" checks move to the employee zone too? | **Not now (later step)** / yes, in this work |
| Z9 | Build order | **12a masters + resolver + `activeFrom` → 12b employee zone on Add / Edit → 12c travel periods + approvals → 12d attendance (procedures, web / biometric) → 12e screens notes + checks**, one fresh chat each / other |

### 22.11 Your answers (2026-10-15)

Your message kept the "…" placeholders; asked once, you chose **"All recommended"**.

| # | Answer |
| :-- | :-- |
| Z1 | Order as in 22.2 (travel → employee → work station → country → company) |
| Z2 | Employee requests travel, line manager approves on /task, HR can add directly (saved approved) |
| Z3 | Shift times = local time where the employee is |
| Z4 | Home holiday calendar during travel |
| Z5 | Web clock-in during travel with no work station: no location check on those days |
| Z6 | Dashboards and reports "today" stay on the company zone |
| Z7 | Logins already waiting: recompute to local midnight |
| Z8 | Leave / resignation / other "today" checks: not now (later step) |
| Z9 | Build 12a → 12e, one chat each |

### 22.10 Code correction and static values

- `Company.TimeZone` default `"UTC"` and the `CompanyTimeZoneResolver` fallback `UTC` are code values; they stay as the last safety net (every company has a zone in the DB).
- `EmployeeService.Create.cs` (line ~109) and `AuthService.cs` (line ~137) use `DateTime.UtcNow` for the joining-date / sign-in checks; `EmployeeAccountService.cs` (line ~183) prints the UTC date in the email. All three move to the resolver in 12a.
- `locations.country` free text instead of a link to the country master (R1).

## 23. Step 12a result (2026-10-15)

| Part | What changed |
| :-- | :-- |
| Time-zone list | New table `time_zones` (IANA id, SQL Server name, display name, base offset). The API fills / refreshes it at start-up from the server's own zone list (`TimeZoneCatalogSync`, 419 zones locally). `GET /api/org/time-zones` (any signed-in user) feeds the pick lists. No zone list in code |
| Countries | `countries.defaultTimeZone`. Masters → Countries drawer: **Default time zone** (search, clearable). A zone not on the list → 400 "Choose a time zone from the list …" |
| Work stations | `locations.timeZone` + `locations.countryId` (FK to `countries`). Saving a work station links the country by name / ISO code; the text column stays. Work Stations drawer: **Time zone** (helper shows the country's zone) and an amber note when the saved country text is not in the master. Copy-to-companies copies both |
| Employee | `employees.timeZone` column only (screens in 12b) |
| Resolver | `EmployeeTimeZone` (`Infrastructure/Services/EmployeeTimeZone.cs`): employee → primary work station → its country (no linked country → company's country) → company → UTC. A zone unknown to the server is skipped. `LocalMidnightUtc`, `TodayIn`, `ForEmployeeAsync`, `ForUserAsync`, `ResolveAsync` |
| `activeFrom` | Add Employee / old drawer / bulk upload with "activate on joining date": the UTC moment of 00:00 on the joining date in the new employee's zone (from the chosen primary work station). "Joining date in the future?" uses today in that zone. Credentials carry `activeFromDate` (local date); success panel, welcome email and the sign-in refusal show the local date. `LOGIN_CREATED` audit adds `ActiveFromDate` |
| Z7 script | `docs/db_changes.sql` Step 12a **Part 2**: dry-run SELECT, then UPDATE of waiting logins still at 00:00 UTC to local midnight (same rule in SQL via `time_zones`). Second run changes 0 rows |

**Checks**: `dotnet build` clean, **526/526** tests (6 new in `EmployeeTimeZoneTests`: rule order, unknown zone skipped, local midnight incl. a skipped midnight (Chile), new / saved employee and login lookups). `tsc` clean; eslint on the changed files shows only errors that were already there (`any` in error handlers, set-state in effects, `!` in api.ts). Throwaway copy `HRMSCore_StepTest` (test API :5299, biometric sync off, local file storage), dropped afterwards; the shared DB was not written.

| Check (on the copy) | Result |
| :-- | :-- |
| Part 1 run twice | ✅ 122 work stations linked, **0 unmatched**; 2nd run 0 rows |
| `time_zones` filled at API start | ✅ 419 rows; Nairobi → E. Africa Standard Time |
| `GET /org/time-zones` as Company HR 68032 | ✅ 419 |
| Work station zone as Group HR (Puneet): `Mars/Base` / `Asia/Kolkata` | ✅ 400 / 200, `countryId` = Kenya |
| Country default zone as Finance Manager (Geoffrey): `Nowhere/X` / `Africa/Nairobi` | ✅ 400 / 200 |
| Company HR edits a work station | 403 (HR has no `orgmasters.manage`; unchanged) |
| Save A (Nairobi station, joining 25 Oct) | ✅ `activeFrom` 2026-10-24 21:00 UTC, `activeFromDate` 25 Oct |
| Save B (Noida station set to Asia/Kolkata) | ✅ 2026-10-24 18:30 UTC |
| A / B sign in before the date | ✅ "Your account becomes active on 25 Oct 2026 …" |
| Save C joining today (Nairobi) | ✅ no `activeFrom`, signs in at once |
| Z7 Part 2 (A, B reset to 00:00 UTC) | ✅ → 21:00 / 18:30 (India Standard Time from the station); 2nd run 0 rows |

**Good to know**
- **The shared DB `HRMSCore_Local2` does not have the Step 11 block** (`lookup_values.isDefault`): every employee save failed on the copy until it was run there. Run Step 11, then Step 12a Part 1, before starting this API.
- Work station **Noida** (MAKL) has country text "Kenya", so it links to Kenya. If it is in India, pick India on the Work Stations screen (and its zone).
- Masters screens (Countries, Work Stations) still write no audit rows, as before this step (follow-up **F21**).
- Nothing in attendance changed yet (12d). Edit Employee does not yet re-work `activeFrom` when the joining date or work station changes (12b).
- Run order: Part 1 → deploy API → start once (fills `time_zones`) → Part 2.

## 24. Step 12b result (2026-10-15)

| Part | What changed |
| :-- | :-- |
| Add Employee step 2 | Under Work Station: **Time Zone** line "Africa/Nairobi (company zone)" / "(from work station)" / "(country default)", read from `GET /api/employees/station-time-zone?locationId=` (follows the primary station). **Change** opens the time-zone search; **Clear** goes back to the inherited zone. Review step shows it. Shared component `modules/employees/components/EmployeeTimeZoneField.tsx` |
| Save | `CreateEmployeeRequest.TimeZone` → `employees.timeZone`; checked against `time_zones` (else "Time zone '…' is not on the list"). `activeFrom` uses it (rule 2). `EMPLOYEE_CREATED` audit has `TimeZone` |
| Edit Employee (/assignment → Edit) | Same field in "Employment Record"; the GET returns `timeZone`, `inheritedTimeZone`, `inheritedTimeZoneSource`. Change audited in `EMPLOYEE_DETAILS_EDITED` ("Time zone: old → new") |
| Bulk upload | Optional column **Time Zone** after Location (Masters sheet list, export fills it). Add: saved on the employee. Update: filled cell replaces, blank keeps. Unknown zone = row not saveable (plain check, no Upload Config rule). `EMPLOYEE_BULK_UPDATE` lists the change |
| Waiting logins | New save hook `Infrastructure/Persistence/LoginActiveFromSync.cs` (runs on every `SaveChanges`, like `JobHistoryRecorder`): when the joining date, primary work station or employee zone changes and the login is still waiting (`activeFrom` in the future), `activeFrom` = 00:00 of the joining date in the employee's zone; joining date today or earlier → `null` (login opens). Audit `LOGIN_ACTIVE_FROM_CHANGED` (User): `Changed`, `From`, `To`, `ActiveFromDate`, `TimeZone`, `ZoneFrom`. Active logins are never touched |
| SQL | None |

**Checks**: `dotnet build` clean, **529/529** tests (3 new in `EmployeeTimeZoneTests`: waiting login follows station / own zone / joining date with 3 audit rows; joining date moved to the past opens the login; active login untouched). `tsc` clean; eslint on the changed files: only the errors already in `employees/api.ts`. API run as Company HR **68032** on throwaway copy `HRMSCore_StepTest` (Step 11 + Step 12a Part 1 run on it first; 0 unmatched stations), test API :5299, biometric sync off, local file storage. Copy dropped afterwards; the shared DB was not written.

| Check (on the copy, joining date = today + 10) | Result |
| :-- | :-- |
| Station zone of Arch Place | ✅ Africa/Nairobi, source Company (no station / country zone set) |
| Add with zone `Mars/Base` | ✅ 400 "not on the list" |
| Add with `Asia/Dubai`, activate on joining | ✅ saved; `activeFrom` 20:00 UTC the day before; audit has `TimeZone` |
| Edit Employee GET | ✅ own Asia/Dubai, inherited Africa/Nairobi (Company) |
| Clear the zone on Edit Employee | ✅ `activeFrom` → 21:00 UTC; `EMPLOYEE_DETAILS_EDITED` + `LOGIN_ACTIVE_FROM_CHANGED` |
| Move joining date +3 days (with confirmation date) | ✅ `activeFrom` follows (21:00 UTC the day before) |
| Assignment tab: primary station → Noida (zone Asia/Kolkata set on the copy) | ✅ 18:30 UTC |
| Bulk update `Nowhere/X` / `asia/dubai` | ✅ not saveable / saved as Asia/Dubai, 20:00 UTC, audit lists "Time zone" |
| Joining date moved to today | ✅ `activeFrom` cleared (login opens); 5 `LOGIN_ACTIVE_FROM_CHANGED` rows in total |
| Zone changed on an employee with an active login | ✅ login untouched |
| Import columns list | ✅ has "Time Zone" |

**Good to know**
- The zone only affects the login start date until Step 12d (attendance).
- The Edit Employee save sends the zone with the other fields: a page open from before this release would clear it on save (reload once after deploying).
- A bulk update cannot clear a zone (blank keeps). Use Edit Employee.
- Moving a joining date earlier than the confirmation date is refused as before ("Confirmation date cannot be before the hire date"): move both.
- Browser checks for you: artifact 12b.1–12b.7.

**Run before deploying the API**: nothing new for 12b. The shared DB `HRMSCore_Local2` still needs **Step 11**, then **Step 12a Part 1**, then (after the first start) **Part 2**.

