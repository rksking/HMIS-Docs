# Blob path clean-up — task (planned, not started)

> Raised by the user on 2026-10-10 with Separation Step 7: "after some time we will look at all the folders and correct
> the paths with the improved way". This file is the task: what exists today, a proposed target, and how to migrate
> without losing a file. **Nothing here is applied yet.** The user approves the target layout first (follow.md #9).

## 1. What exists today (inventory from the code, 2026-10-10)

| # | Where it is written | Key layout today | DB column holding the key |
|---|---|---|---|
| 1 | `DocumentsService` (employee documents), `EmployeeDocumentBulkUploadService` | `employee-documents/{ownerId_ownerName}/{companyName_companyId}/{docType_docTypeId}/{name_ticks}.{ext}` (`BlobKeyBuilder.Build`) | `employee_documents.storageKey` |
| 2 | `DocumentsService` (company repository) | `company-documents/{companyName_companyId}/{folderName_folderId}/{name_ticks}.{ext}` (`BuildCompanyDocument`) | repository documents table |
| 3 | `EmployeeDocumentRepositoryService` (education) | `education/{ownerId_ownerName}/…` (`BlobKeyBuilder.Build`) | education rows |
| 4 | `RecruitmentService`, `EmailService` (offer letters) | `candidate/{companyName}/{vacancyNumber}/{candidateId_name}/{category}/{name_ticks}.{ext}` (`BuildCandidateDocument`) | `candidate_documents.storageKey` |
| 5 | `OnboardingService` | `candidate/…` (same builder) — older rows: `onboarding/{profileId_name}/…` | `candidate_onboarding_documents.storageKey`; some copied to `employee_documents` |
| 6 | `DocumentsService` line ~229 (generated PDF) | `docs/{companyId}/{employeeId}/{guid}.pdf` | `employee_documents.storageKey` |
| 7 | `EmployeeService` (certificates) | `certificates/{companyId}/{guid}{ext}` | certificate rows |
| 8 | `AuthorizedSignatoriesService` | `signatures/{companyId}/{guid}_{name}{ext}` | `authorized_signatories.signatureImageUrl` |
| 9 | `LeaveProofStorage` | `leave-proofs/…` (appsettings `LeaveProof:StorageFolder`) | `leave_requests.proofDocumentUrl` |
| 10 | `ClockEvidenceService` | `clock-selfies/…` (appsettings `SelfieStorageFolder`) | attendance punch rows |
| 11 | `SeparationTaskService` (handover, F&F statement, payment evidence) | `employee/{companyId}/separation/{separationId}/{CATEGORY}/{guid}{ext}` | `separation_attachments.fileUrl` |
| 12 | `SeparationReleaseService` (Step 7) | `separation/{employeeNumber_employeeName}/final/…` + `final/documents/{source}/NNN_{file}` (copies) | `separation_attachments.fileUrl` (EXPERIENCE_LETTER / ARCHIVE), `employee_documents.storageKey` |
| — | `LettersService` (manual letters) | **no file** — the letter body is served as HTML; 10 of 21 `employee_documents` rows had an empty `storageKey` | — |

Problems: one person's files sit under several roots (candidate, onboarding, employee-documents, education, docs,
certificates, employee/…/separation, separation/…/final); some keys use ids only (not browsable), some names only; the
company sits at different depths; manual letters have no file at all.

## 2. Proposed target (for the user to approve or change)

One root per **person**, one per **company**, and everything about a person under their folder:

```
people/{employeeNumber_employeeName}/                    (candidate before hire: people/candidate_{candidateId_name}/)
    recruitment/{vacancyNumber}/{category}/…              (CV, offer, onboarding documents)
    documents/{documentType}/…                            (employee documents, education, certificates)
    letters/{yyyy}/{reference}.pdf                        (every issued letter as a PDF)
    leave/{yyyy}/…                                        (leave proofs)
    attendance/{yyyy-MM}/…                                (clock selfies)
    separation/{caseReference}/{category}/…               (handover, F&F statement, payment evidence)
    separation/{caseReference}/final/…                    (Experience cum Relieving Letter + archive copies)
company/{companyCode_companyName}/
    repository/{folder}/…    signatures/…    letterheads/…
```

All keys built only through `BlobKeyBuilder` (one method per area), sanitised segments, the root folder names from
appsettings (no literals in services).

## 3. Migration method (when approved)

1. **Inventory script** (read-only): every table / column that stores a key → CSV of `table, id, oldKey, newKey`; check each
   old key exists in the container; report missing files.
2. **Copy** each blob to its new key (server-side copy; add `CopyBlobAsync` to `IBlobStorageService` — Azure
   `StartCopyFromUri`, local file copy). Originals untouched.
3. **Update the DB keys** in one transaction per table (script in `docs/db_changes.sql`, re-runnable: only rows still on
   the old key).
4. **Verify** on a DB clone + a copy of the container: every row's key exists; downloads work through the API for each
   area (role E2E).
5. Apply to dev / production; keep the old blobs for an agreed period, then delete with a second script.
6. Switch every writer to the new `BlobKeyBuilder` methods in the same release so no new file lands on an old path.
7. Manual letters: render and store a PDF at issue time (reuse `Pdf/LetterPdf.cs`) so every letter has a file.
