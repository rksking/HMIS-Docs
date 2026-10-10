# /letters clean-up — todo (planned, not started)

> Raised 2026-10-10 from Separation Step 7 (`docs/sepration.md` §10, Step 7 "Code correction and static values").
> The System path (`LettersService.IssueSystemLetterAsync`, used by the Experience cum Relieving Letter) is already
> clean; the **manual /letters path** is not. Verify each line in code first, then present the plan (follow.md #9).
> Order: after the blob-path clean-up (`docs/blob_paths.md`), because issued letters must be stored as PDFs under the
> new layout.

## Issues found

- [ ] **L1. Sample fallbacks in `RenderLetterAsync` / `IssueLetterAsync`** — when data is missing they fill in
      "Sarah Omolo", "Director of Human Capital", "Consultant Physician", "Internal Medicine", "EMP-00101",
      "HR Operations Manager". Must refuse with a message naming the missing field (as the System path does).
- [ ] **L2. Letter reference** — random `LTR-HRMS-yyyy-####` can repeat. Use a per-company sequence
      (like `SEP-{year}-{0001}`), unique in the DB.
- [ ] **L3. `PdfUrl` points at a non-existent endpoint.** Add a real download endpoint (permission-checked) or remove it.
- [ ] **L4. Manual letters have no file** — the body is served as HTML; 10 of 21 `employee_documents` rows have an
      empty `storageKey`. Render every issued letter to PDF (`Pdf/LetterPdf.cs`), store it, attach with the key; back-fill
      the empty rows on a clone first.
- [ ] **L5. Template preview** uses a sample `probation_end_date` = joining + 6 months — use the employee's
      `employment_details.probationEndDate` (days-based, Step 3a).
- [ ] **L6. "Employment Letters" document type is created in code** when missing (`AttachToEmployeeDossierInternal`)
      — seed it in `docs/db_changes.sql` per company and refuse if absent.
- [ ] **L7. Seed data that looks real** — default signatory "Sarah Omolo" and letterhead contacts
      (`hr@hospital.co.ke`) in `LettersSeedData`. Seed blank / inactive, and show a readiness banner on /letters
      ("no active authorised signatory / letterhead") like Separation Config.
- [ ] **L8. Template wording** — "Extension of probation" template says "three (3) months"; use a placeholder for the
      extension days (HR text — ask before changing).

## Done when

`tsc --noEmit` and `dotnet build` clean; clone E2E as HR (issue, preview, download, refusals for missing data /
signatory) and employee (own letters only); DB changes in `docs/db_changes.sql`; `docs/policies.md` section; memory.
