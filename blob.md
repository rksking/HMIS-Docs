# Blob paths and file security — ✅ done 2026-10-10 (Separation Step 9)

> Raised with Separation Step 7 ("correct the paths with the improved way"). Layout approved by the user 2026-10-10:
> company first, `final/` holds only the Experience cum Relieving Letter, hired candidates' files move to the person's
> `recruitment/` folder, **nothing is ever deleted by the system** (old files stay; the user removes them by hand).
> Security: "100% secure" — every file needs a login and an ownership check; line managers get no team documents;
> signatures for HR / superadmin / developer only; every download audited, switchable (Access Control → Audit
> Configuration, developer only). policies.md §96.

## 1. Inventory (re-checked against the code)

| # | Writer | Key before | DB column | Rows on dev |
|---|---|---|---|---|
| 1 | `DocumentsService` upload, `EmployeeDocumentBulkUploadService` | `employee-documents/{owner}/{company}/{type}/…` | `employee_documents.storageKey` | 12 (+10 empty) |
| 2 | `DocumentsService` repository | `company-documents/…` **and `org/{companyId}/{folderId}/…` (340 seed rows — missing in the first inventory)** | `company_documents.storageKey` | 536 |
| 3 | `EmployeeDocumentRepositoryService` education | `education/{owner}/…` | `education_details.certificateUrl` | 3 (name only) |
| 4 | `EmployeeService.UploadCertificateAsync` (**missed**) | `certificates/{companyId}/{guid}` wrapped in `/api/employees/certificate-file?key=` | `education_details.certificateUrl` | 0 |
| 5 | `RecruitmentService`, `EmailService` (offer letter **.html**) | `candidate/{company}/{vacancy}/{cand}/…` | `candidate_documents.storageKey` | 1 |
| 6 | `OnboardingService` | `candidate/…`, older `onboarding/{cand}/…` (**the same file is also in `employee_documents`**) | `candidate_onboarding_documents.storageKey` | 9 |
| 7 | `AuthorizedSignatoriesService` | `signatures/{companyId}/…` wrapped in `/api/authorized-signatories/signature-file?key=`; 5 rows are inline `data:image` | `authorized_signatories.signatureImageUrl` | 1 key |
| 8 | Letters issue (**missed**) | copy of the signatory value | `issued_letters.signatorySignatureUrl` | 0 keys |
| 9 | `LeaveProofStorage` | `leave-proofs/{co}/{yyyy-MM}/{guid}` | `leave_requests.proofDocumentUrl` | 0 |
| 10 | `ClockEvidenceService` | `clock-selfies/{co}/{yyyy-MM}/{guid}` | `attendance_clock_events.selfieKey` | 0 |
| 11 | `SeparationTaskService` | `employee/{co}/separation/{caseId}/{CAT}/{guid}` | `separation_attachments.fileUrl` | 2 |
| 12 | `SeparationReleaseService` | `separation/{empNo_name}/final/…` + `final/documents/…` copies | `separation_attachments.fileUrl`, `employee_documents.storageKey` | 0 |
| 13 | `LettersService` | `employee-documents/…` (Step 10) | `employee_documents.storageKey` | 1 |
| — | `DocumentsService.UploadDocumentAsync` (**removed**) | `docs/{co}/{emp}/{guid}.pdf` — a row with **no file**, fake 1 MB size | — | 0 |
| — | New-employee form documents (**fixed**) | never uploaded — rows with empty key and an invented document number | — | — |
| — | Photos / logos | `data:` / URLs in the DB, not files | — | — |

## 2. Final layout (appsettings `BlobPaths` — every folder name; a blank one stops the API)

```
{companyCode}/people/{employeeNumber_name}/
    recruitment/{vacancyNumber}/{category}/{file}_{ticks}.{ext}     hired person's CV, offer, onboarding documents
    documents/{docTypeCode}/{file}_{ticks}.{ext}                    employee documents, bulk upload
    documents/education/{file}_{ticks}.{ext}                        education certificates
    letters/{yyyy}/{referenceNumber}.pdf                            every issued letter
    leave/{yyyy}/{file}_{ticks}.{ext}                               leave proofs
    attendance/{yyyy-MM}/{in|out}_{yyyyMMdd-HHmmss}_{ticks}.{ext}   clock selfies
    separation/{caseReference}/{category}/{file}_{ticks}.{ext}      handover, F&F statement, payment evidence
    separation/{caseReference}/final/{file}.pdf                     Experience cum Relieving Letter only
{companyCode}/candidates/{vacancyNumber}/{candidateId_name}/{category}/…   not hired
{companyCode}/company/repository/{folderName}/…    {companyCode}/company/signatures/{name}_{ticks}.{ext}
{companyCode}/uploads/{userId}/…    certificate picked on the new-employee form; copied into the person's folder on save
```

`BlobKeyBuilder` (Application/Common) is the only place keys are built; `IBlobPathResolver` gives the person folder /
company code (refuses a company without a code). Every segment is lower-case `a-z 0-9 -`; `IsSafeKey` refuses `..`,
`/…`, `\`.

## 3. Security (what makes a file reachable)

- **No key from the browser, no key to the browser.** Downloads go by record id; DTOs carry an API URL. The only keys a
  browser ever holds are its own fresh uploads (signature → must be under `{co}/company/signatures/`; certificate →
  the person's `documents/education/` or the uploader's own `uploads/{userId}/`; onboarding → the candidate's folder).
  Anything else is refused (400).
- **Removed:** `GET api/authorized-signatories/signature-file?key=` and `GET api/employees/certificate-file?key=`
  (both anonymous, any key; the second also read any server file and invented a "VERIFIED CREDENTIAL" PDF).
- **`IPersonFileAccess`:** the employee themselves, holders of the permission whose accessible companies include the
  employee's company, developer / superadmin. Used by employee documents (`documents/{id}/file`, the repository for
  Me / Assignment / Employee), education, letters (`letters.view`). Line managers: none (ems lost `documents.view`).
- **Signatures:** `GET api/authorized-signatories/{id}/signature` — `letters.signatories.view` + company in the caller's
  workspace (HR, superadmin, developer); DTOs hide the image from everyone else.
- Education endpoints now need `employees.view` (read) / `employees.edit` (write); certificate upload `employees.edit`
  or `employees.create`.
- Local storage refuses unsafe keys and any path outside its folder; the disk fallback in EmployeeService is gone.
- Frontend: `openSecureFile` (lib/api.ts) and `SecureImage` (components/ui) fetch with the sign-in token and show a
  `blob:` address that lives only in that tab.
- Every download writes `audit_log` (`FileDownload` / `DOWNLOAD`, user, IP, browser) unless the developer switches
  "File downloads" off on `/audit/config`.

## 4. Migration (done on a clone; dev — run by the user)

`POST api/admin/blob-paths/migrate?dryRun=true|false` (`admin.blob_paths.migrate`, Developer role). Per file: copy →
byte-for-byte check → re-point every row that shares the file (priority: separation 1, hired candidate 2, employee
files 3, company 4, candidate 5) → `blob_path_migrations` row (old key, new key, status). A missing file is reported
and its rows left alone (logged once). **Never deletes.** Re-runnable.

Clone result (2026-10-10, `HRMSCore_BlobTest` + local copy of the 213 files in Azure): 564 rows, 553 files —
**213 moved, 340 missing, 0 failed**; re-run: 0 moved, 221 already current. Missing = 336 seeded `org/` repository rows
with no file, `BSc/Diploma/MSc_Certificate.pdf` (name-only seed rows), 1 signature never uploaded.
Security E2E: 58/58 (+1 expectation corrected) — anonymous 401, owner 200, line manager / other employee / other-company
HR 403, HR 200, forged keys 400, audit on/off.

Manual clean-up later: `SELECT oldKey FROM blob_path_migrations WHERE status = 'MOVED'` lists the old files that can be
deleted once you are satisfied.

## 5. Open items

- New hires: on conversion, candidate files stay under `candidates/` (the migration moved existing ones); move them to
  `people/{emp}/recruitment/` in the onboarding conversion — follow-up.
- Static values seen, not changed: `PRESET_SIGNATURES` (fake SVG signatures, letters SignatureMasterDrawer),
  `EmailService` SMTP defaults (localhost, 587, "Talent Acquisition", localhost:4000), Letters signatory department
  default "Executive Leadership", `CompanyLetterhead.LogoInitials = "AH"`, certificate upload limits in
  `EmployeesController` (5 MB, pdf/jpg/png) and signature limits (2 MB, png/jpg) hardcoded.
- Company HR has no `letters.view` (letter PDFs) — decide whether to grant.
- Many modules write no audit rows yet (leave approvals, letters, documents upload, users / roles) — follow.md #20 asks
  for them; each new audit type must be added to an area in `audit_settings.entityTypes` to be switchable.
