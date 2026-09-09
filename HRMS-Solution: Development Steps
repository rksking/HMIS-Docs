# HRMS Enterprise Development Steps

Deliver a multi-tenant SaaS HRMS for independent business groups, using PostgreSQL, ASP.NET Core 10 and Next.js. The first production release covers the complete mockup workflow, SaaS tenant lifecycle, biometric integration, callback management and a future-improvements workspace. [CONTEXT.md](CONTEXT.md) is the governing product and architecture baseline; this document turns it into reviewable delivery gates.

The current repository contains the browser mockup and these planning documents. The production application, migrations, deployment and device adapters remain to be implemented. All gates below are initially **Not started**; documentation and source inspection do not satisfy implementation gates. Research baseline: 9 September 2026.

## 1. Delivery rules

1. Build complete vertical slices: database constraints, API permissions, UI, background work, audit, tests and operational evidence together.
2. Preserve the mockup for reference. Port validated requirements, not its storage, frontend security assumptions or unverified statutory calculations.
3. Tenant isolation is a release prerequisite for every feature. A company's scope is nested inside its tenant; group access never spans customer tenants.
4. ASP.NET Core is the sole business API and EF Core owns database migrations. Next.js has no direct HR database access or duplicate payroll engine.
5. Finish the current gate's acceptance evidence before adding dependent features. An attractive screen without its correct persistence and authorization is incomplete.
6. Integrate one real biometric device/vendor path early. A simulator verifies the HRMS protocol, not vendor compatibility.
7. Publish financial and integration contracts before dependent clients. Use generated API types and explicit versioning.
8. Use synthetic data in development and automated tests. Live employee data enters only through the approved migration process.
9. Keep plans, estimates and unresolved decisions visible. Do not declare compliance, capacity or hardware support without evidence.

The recommended launch architecture is a modular API with separate workers and shared PostgreSQL tables protected by tenant ownership/RLS. Operational automation, tests and support access are part of the product. Self-service paid signup is optional after launch; tenant onboarding, quotas, plan assignment, suspension, export and offboarding are mandatory at launch.

## 2. Planned repository structure

```text
HRMS-Solution/
├── Mockup/                         # Preserved reference
├── docs/
│   ├── CONTEXT.md
│   ├── DEVELOPMENT_STEP.md
│   ├── adr/                        # Architecture decisions
│   ├── contracts/                  # OpenAPI/event schemas and vendor mappings
│   ├── evidence/                   # Sanitized gate evidence and reconciliations
│   └── runbooks/                   # Deployment, incidents, recovery and support
├── src/
│   ├── backend/
│   │   ├── Hrms.slnx
│   │   ├── global.json
│   │   ├── Directory.Packages.props
│   │   ├── Hrms.Api/
│   │   ├── Hrms.Worker/
│   │   ├── Hrms.SharedKernel/
│   │   ├── Hrms.Migrations/         # Single migration owner/runner
│   │   └── Modules/
│   │       ├── SaaS/
│   │       ├── Identity/
│   │       ├── Organization/
│   │       ├── People/
│   │       ├── Workflow/
│   │       ├── Leave/
│   │       ├── Attendance/
│   │       ├── Payroll/
│   │       ├── Recruitment/
│   │       ├── Documents/
│   │       ├── Integrations/
│   │       ├── Reporting/
│   │       ├── Product/
│   │       └── Audit/
│   ├── web/                        # Next.js App Router, feature folders, owned UI
│   └── edge/                       # Collector plus isolated vendor adapters
├── packages/
│   └── api-client/                 # Generated TypeScript contract client
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── contracts/
│   ├── e2e/
│   ├── devices/
│   └── performance/
├── infra/                          # Container builds, local composition and IaC
└── scripts/                        # Reproducible checks and data migration tools
```

This is a target layout, not a claim these folders already exist. Avoid creating empty module frameworks just to populate it. Modules can initially use feature folders with clear Domain/Application/Infrastructure boundaries; add separate projects when dependency enforcement benefits from them. PostgreSQL schemas do not require separate databases per module.

## 3. Gate map and sequencing

| Gate | Outcome | Dependencies | Accountable roles |
|---|---|---|---|
| G0 | Mockup parity, policy/device inventory and SaaS decisions | None | Product, HR/payroll, architecture, privacy, integrations |
| G1 | Reproducible platform and delivery foundation | G0 baseline | Backend, frontend, platform |
| G2 | Tenant lifecycle, authentication, authorization and isolation | G1 | Backend, security, platform |
| G3 | Responsive accessible design system and application shell | G1; G2 for protected journeys | Design, frontend, QA |
| G4 | Organization, employee lifecycle and imports | G2, G3 | People team, HR, QA |
| G5 | Durable jobs, approvals, audit integration and notifications | G2, G3 | Backend, frontend, QA |
| G6 | Leave policies, accrual and approval | G4, G5 | HR, backend, frontend, QA |
| G7 | Rosters, raw punches, attendance and close | G4, G5; G6 for final leave reconciliation | Attendance team, HR, QA |
| G8 | Real biometric adapter and callback management | G5, G7; device/privacy work from G0 | Integrations, platform, privacy, QA |
| G9 | Reconciled payroll, payslips and exports | G4, G5, G6, G7 | Payroll, backend, frontend, QA |
| G10 | Secure documents, healthcare credentials and letters | G4, G5 | HR/compliance, backend, frontend, QA |
| G11 | Recruitment from requisition to employment | G4, G5; G10 for final document flow | Recruitment, backend, frontend, QA |
| G12 | Reports, customization and improvement/release window | Relevant domain gates | Product, design, reporting, QA |
| G13 | Migration rehearsal, recovery, security, capacity and launch | G0–G12 | Delivery owner and all business/technical owners |

The sequence permits independent team work after shared contracts are agreed; it is not an instruction to skip dependencies. Security, telemetry, file storage and migrations start in the foundation and are strengthened continuously. G10's file substrate is made available early so employee imports and attachments do not invent temporary storage mechanisms.

Do not commit a calendar delivery date until G0 resolves tenant count, policies, device access and integrations. Size work in person-weeks per epic after the first vertical slice, then forecast from observed throughput. Budget explicitly for vendor access/licensing, device hardware, independent payroll review, security assessment and restore drills. The biometric privacy lead time identified in [CONTEXT.md](CONTEXT.md) can constrain the pilot even when code is ready.

## 4. G0 — Discovery and verified acceptance baseline

**Build the evidence set.** Catalogue all 30 `Pages.*` route assignments from the assembled HTML, the 18 split source files, 54 Prisma models, 11 test suites and both CSV templates. Reconcile differences between the assembled application and `phase3-d.js`, `phase5-a.js`, `phase5-b.js`, `phase5-f.js`. Document each workflow's actors, preconditions, transitions, data scope, output and known defects. Do not mistake historical test counts for test results.

Create the first ADRs: SaaS/shared-table isolation, module boundaries, identity provider/session model, payroll versioning, job/outbox design, object storage, hosting/data location, edge adapter strategy and UI component ownership. Resolve whether identity membership spans several tenants through separate memberships; default to explicit selection with no cross-tenant aggregate views.

Create a business policy workbook/specification covering company structures, employee categories, CBAs, working calendars, accrual units, pay groups, taxes, contributions, relief eligibility, overtime, joiner/leaver treatment, bank formats, GL accounts, credential types and retention classes. Country-rule fixtures require official sources and payroll-owner interpretation. Capture the confirmed February 2026 NSSF changes and the distinction between three calendar months and 90 days of maternity leave.

Inventory every intended biometric model/firmware and vendor server. Obtain approved API/SDK documentation, licence terms, one test device or vendor sandbox, sample events, event identifiers, time-zone behavior, retention limits, offline behavior and the installation network diagram. Begin biometric DPIA/data-flow review before any live-data pilot. Use synthetic employees for technical prototyping.

**Acceptance evidence:** a traceability matrix links every mockup capability to a gate and test scenario; blocking policy questions have owners; architecture choices are recorded; missing device contracts are listed honestly; and a product owner has accepted the launch scope. Device-specific gates remain blocked on hardware evidence if equipment is unavailable, while unrelated implementation continues.

## 5. G1 — Platform foundation

Pin supported .NET 10, EF/Npgsql 10, PostgreSQL 18, Next.js 16 and Node 24 versions as described in [CONTEXT.md](CONTEXT.md). Build API, worker and UI with local container composition for PostgreSQL, private object storage, an email sink and telemetry collection. Use environment templates without secrets; separate development data from production configuration.

Implement configuration validation, health probes, structured logging, request correlation, Problem Details errors, OpenAPI generation, a generated TypeScript client and a minimal end-to-end feature. Establish the approved PostgreSQL migration runner and distinct migration/runtime database roles. No startup path creates demo data in production.

Create CI tasks for restore/install from locks, formatting, compilation, type checking, unit and PostgreSQL integration tests, API-contract drift, secret detection, dependency/container scanning and licences. Pin build inputs and produce immutable artifacts with provenance/SBOM. Infrastructure code should support staging and production separation without committing to Kubernetes unless justified by hosting needs.

**Acceptance evidence:** a clean checkout boots through documented steps; UI calls the API and persists a harmless synthetic record through PostgreSQL; worker completion survives restart; invalid configuration fails clearly; CI produces versioned deployable images; migrations run from empty and prior schemas; secrets do not appear in logs or browser bundles.

## 6. G2 — SaaS lifecycle, identity and isolation

Implement tenants/groups/companies, memberships, users, scoped roles, permissions and service principals. Keep platform control-plane permissions separate from tenant administrator permissions. Deliver tenant provisioning, initial administrator invitation, country defaults, plan/quotas, activation, suspension/reactivation and authorized tenant selection. Provide a provisioning status screen with retryable failures and audit, using idempotent jobs.

Implement OIDC and the .NET-owned secure-cookie session, MFA requirements, idle/absolute expiry, logout, revocation and access-change invalidation. Protect cookie mutations with antiforgery/origin controls. Make recovery and break-glass procedures explicit and auditable. Do not reuse mockup password hashes, demo accounts or broad `hasGroupAccess` behavior.

Add database ownership columns, composite foreign keys, EF query filters, write validation and RLS. Set context transaction-locally; use a runtime role without table ownership or bypass privileges. Reject missing tenant context. Verify reads/writes through joins, raw SQL, includes, bulk commands, jobs and exports. Prevent a tenant from referencing another tenant's department, file, employee or device even with a valid UUID.

Build administration pages for company grants, role assignments and time-limited support access. The support grant requires tenant approval and automatically expires; operator tooling defaults to metadata only. Enforce quotas independently of UI feature flags, including API request rate, import/export concurrency, files and connector intake. Access checks happen again when queued work runs.

**Required test fixture:** Tenant A has Companies A1/A2; Tenant B has B1/B2. Include duplicate employee numbers, names and emails across tenants, group HR, company HR, manager, employee, payroll preparer/approver, auditor and connector principals. Include an identity with memberships in both tenants to test deliberate switching.

**Acceptance evidence:** no cross-tenant data or existence leaks through detail/list/search/count/dashboard/export/download/job/realtime/cache routes; roles cannot escalate themselves; support grants expire; suspension stops applicable access and machine ingestion has explicit status; a high-volume tenant does not starve a second tenant; pooled connections do not retain the previous context after success, failure or cancellation. Repeat important SQL operations under the actual production runtime role.

## 7. G3 — Design system and shell

Build an owned component set for typography, buttons, form fields, date/range controls, tables, filters, tabs, dialogs, side panels, timeline, approval cards, alerts, skeletons and background-job progress. Use the screenshot's dark sidebar/teal direction, then validate spacing, contrast and readable chart labels. Implement search, breadcrumbs, company/tenant context, personal inbox and responsive navigation.

Implement permission-aware navigation, URL-backed filters, saved density/column preferences, pending/empty/error states and keyboard shortcuts with discoverability. Company/tenant switching cancels stale requests and clears query caches, selections and sensitive rendered data. Stale responses must not populate the newly selected scope. Restore only preferences compatible with the current permissions.

Build interactive prototypes for group dashboard, Employee 360, approval inbox, attendance register and payroll workspace using synthetic data, followed by real API wiring as domain gates arrive. Validate the biometric and callback detail layouts early so diagnostic information is usable without exposing secrets.

**Acceptance evidence:** desktop and mobile viewport review, keyboard-only completion of representative flows, screen-reader spot checks, contrast and automated accessibility checks. Verify full company names, multi-company filters, loading/error behavior, focus return from dialogs and non-drag alternatives for recruitment stages. Record baseline frontend performance measurements; library choice alone is not evidence of accessibility.

## 8. G4 — Organization, employees and imports

Implement company-owned divisions, departments, sites, jobs, grades and cost centres with deactivation/effective dates. Relationships must allow departments across sites and validate company ownership. Build the employee directory, Employee 360, self profile and approved edit paths. Store salary and bank details separately, with DTO projections that omit unauthorized fields entirely.

Create person/employment/assignment history, contracts, probation, status changes, manager relationships and effective-dated compensation. Prevent cycles in reporting lines and overlapping incompatible assignments. Implement transfers with source/destination checks, effective dates, history, manager reassignment and an explicit leave/payroll settlement policy. A transfer does not retag old payslips or share prior-employer documents automatically.

Implement import upload → map → validate → preview → approve/commit → reconcile. Support the supplied 36-column employee CSV, normalize UTF-8 BOM/date/decimal conventions, preserve employee/bank identifiers as strings, and require tenant/company context outside the legacy file. Resolve managers in a second pass. Propose new master-data values for review instead of silently creating departments from spelling errors.

Store import source hash, schema version, mapping, per-row errors, source IDs and outcomes. Commit through idempotent chunks with an explicit all-or-partial success policy; reports distinguish inserted/updated/skipped/rejected rows. Large files and results live in private storage. Re-running a batch must not duplicate employees or employment history.

**Acceptance evidence:** imported counts/totals match approved input; duplicate conflicts and missing managers are reviewable; cross-company references fail; concurrent edits show a conflict; effective changes appear on correct dates; a transfer preserves original historical ownership; employees cannot change protected salary/bank fields through modified requests. Deactivation revokes sessions and creates downstream integration tasks.

## 9. G5 — Workflow and durable processing

Create versioned workflow definitions, active instances, assigned steps, decisions and delegations. Build the approval inbox with filtering by assigned-to-me, overdue, delegated and completed. Decisions require current assignment, permission and valid transition. Implement maker/checker separation for payroll, bank changes and selected configuration changes.

Persist decision, business change, audit and outbox atomically. Implement durable inbox/outbox/jobs with leases, recovery, bounded retries, dead letters and idempotent consumers. Provide an authorized job status API and UI. Schedule work fairly by tenant, company and queue class. Use separate concurrency budgets for payroll, file processing, notification and connector traffic.

Build in-app notifications with per-recipient read state and an email adapter using the development email sink. Templates contain minimal sensitive information; links re-authorize on opening. Notification failures do not undo committed approvals. Material workflow definition changes apply to new instances unless an explicit migration is approved and audited.

**Acceptance evidence:** two concurrent approval attempts produce one decision; a requester cannot approve their own restricted workflow; revoked/delegated users cannot decide stale steps; no assigned approver produces a visible exception; worker crash after commit does not lose the event; duplicate delivery does not duplicate side effects; a poison message does not block other tenants.

## 10. G6 — Leave

Implement leave type/policy versions, working calendars and holiday versions, eligibility, accrual, carry-forward/expiry and opening-balance migration. Store entitlement units explicitly, including calendar months where required. Separate legal minimums, company benefits and negotiated employment terms. Avoid inferring eligibility solely from a generic demographic field when additional policy evidence is needed.

Use the ledger for accrual, pending reservations, consumption, reversals and adjustments. Build request preview with working dates, policy version, balance before/after and approval route. Support half days, cross-year requests, documents, cancellation/rejection and reasoned adjustments. Holiday/roster changes require a defined recalculation policy and cannot silently rewrite settled payroll.

**Acceptance evidence:** join-date accrual, leap years, month ends, cross-year leave, calendar-month maternity boundaries, carry-forward expiry and holiday exceptions have reviewed fixtures. Simultaneous requests/approvals cannot overspend balances or overlap disallowed periods. Duplicate accrual jobs do not accrue twice. Cancellation restores the correct reservation/consumption entries. Leave-liability reports show the configured valuation basis rather than assuming basic salary divided by 26 is universal.

## 11. G7 — Attendance and rostering

Build shift templates, effective rosters, shift instances, breaks, overtime rules and site time zones. Separate `RawPunch`, `WorkInterval`, `AttendanceSummaryVersion` and `AttendanceException`. Store source timestamp, UTC interpretation, device/collector receipt and provenance. Begin with web clocking and the legacy CSV adapter, plus a simulator for the normalized device contract.

Implement deterministic pairing and shift assignment with explicit overnight/multiple-punch rules. Recognize scheduled weekend/holiday work. Treat missing punches, unknown mapping/direction, unreasonable duration and clock drift as exceptions. Do not infer half a day from a missing clock-out. Preserve observations when leave or a holiday changes classification.

Deliver daily monitoring, monthly register, employee time view, regularization workflow and approved overtime. Establish attendance close, reopening and adjustment controls. Attendance calculation versions feed payroll snapshots. Employee web clocking is server-timestamped and authenticated; optional location evidence must be separately specified, minimized and reviewed.

**Acceptance evidence:** tests cover night shifts, split shifts, breaks, reordered/duplicate punches, missing in/out, holidays, leave overlap, location time zones, device clock changes and employment transfers. Reprocessing unchanged inputs is deterministic. Closed periods remain unchanged until an authorized correction. An out-of-order event after payroll approval creates an adjustment issue rather than changing net pay.

## 12. G8 — Biometric adapters and callbacks

### 12.1 Edge collector and first real adapter

Implement the adapter interface with capability discovery, connection testing, bounded historical retrieval, live subscription where supported, normalization, health reporting and checkpointing. Select the vendor path based on G0 evidence: vendor server API, proprietary push/listening protocol or native SDK. Isolate SDK-specific dependencies in the collector; the API consumes only the normalized contract.

Implement encrypted disk spool, bounded buffers, retry/jitter, credential storage/rotation, heartbeat, clock-skew measurement and a safe upgrade/restart procedure. The collector runs under a minimal service account and initiates authenticated HTTPS to SaaS. Device-facing ports stay inside the approved site network. Capability/firmware support must be explicit; no assumed universal TCP port or undocumented vendor endpoint.

Add device inventory, mappings, live events, sync history and exception screens. The ingestion endpoint authenticates the connector, validates registered devices, maps ownership server-side, enforces limits and persists accepted records before returning per-item outcomes. Unknown employee mappings go to quarantine. Cursor advancement and acknowledgments are designed for retry without data loss.

**Hardware acceptance evidence:** a compatibility matrix names the actual model, firmware, vendor software, licence and collector platform. Demonstrate an ordinary workday, overnight shift, duplicate burst, at least a 24-hour simulated disconnection/backfill, collector reboot, expired credential, device time adjustment, event-ID reset and full-spool alarm. Reconcile exported vendor counts against accepted + duplicates + quarantined/rejected outcomes for a known window. A synthetic load test demonstrates the longer proposed spool capacity. Mark untested commands unsupported.

### 12.2 Incoming callbacks

Create versioned vendor-specific routes with raw-body authentication verification and a durable inbox. Validate connector ownership, event schema, timestamp/replay rules and body limits before processing. Support partial failures explicitly where the vendor protocol permits them. Keep callback authentication separate from browser cookies and OIDC redirect handlers.

**Acceptance evidence:** invalid signatures, altered bodies, old timestamps, wrong tenant/device, unknown schema, oversized requests and expired keys fail safely. Repeated valid messages produce one business effect. A crash after durable intake recovers. Unauthenticated legacy protocols are accepted only by the approved local collector, never directly by the public SaaS endpoint.

### 12.3 Outgoing callbacks

Implement the event catalog, subscription creation, redacted preview, synthetic test, activation, pause/resume, key rotation, delivery history and controlled replay. Use the signing and delivery contract in [CONTEXT.md](CONTEXT.md). Implement destination validation and enforced egress controls, including DNS rebinding/IPv6/redirect cases. Recheck event permissions at dispatch.

Track message, delivery and attempt identities separately. Retry with the documented schedule, bounded `Retry-After`, circuit breakers and dead-letter handling. Render only sanitized response diagnostics. Subscription scope changes must not deliver already queued sensitive events under stale authorization.

**Acceptance evidence:** contract tests verify exact-byte signatures, key overlap, duplicate handling, retry policy, 410 pause, timeout, 429, 5xx and manual replay. SSRF tests cover loopback/private/metadata addresses, alternate IP representations, DNS changes and redirect destinations. One failing tenant's callback cannot delay another tenant's payroll job.

**Gate boundary:** a real adapter and the privacy prerequisites must pass before advertising biometric support. If hardware or licence access is pending, web/CSV attendance can be tested independently but G8 is not complete. No real messages are sent to third parties during tests without approved recipients.

## 13. G9 — Payroll

Build an isolated deterministic calculation engine using C# decimal and explicit country-rule versions. Define earning/contribution bases, brackets, caps, eligibility, reliefs, rounding and employer charges. Implement Kenya as the first country pack. Retain source documents and reviewed expected outputs; the mockup's computed output is not the financial oracle.

Add company pay groups, periods, regular/off-cycle/adjustment run types, input readiness, snapshots, computation chunks, reconciliation, assigned approvals and publication. Enforce unique active run rules and row-level ownership. Freeze employee/company/pay-component/rate/bank/template details used by an approved run so later changes cannot alter prior outputs.

Build payroll comparison screens and blocking exceptions for missing identifiers/bank data, unsupported tax treatment, unapproved overtime, incomplete attendance, negative net and conflicting employments. Explain every amount through input references and component calculations. Sensitive figures never appear in manager payloads, notifications or general audit excerpts.

Deliver own payslips, payroll register, bank exports, statutory exports and balanced GL journals with versioned schemas and checksums. An exported file records who downloaded it and which approved version generated it. Track payment batch creation, acknowledgments, rejections, partial settlement and final settlement separately; a callback cannot mark a run paid without a validated mapping and policy-authorized transition. Initial bank integration generates approved files; it does not initiate transfers.

**Financial test matrix:** tax bracket edges; NSSF thresholds before/after February 2026; SHIF floor; approved AHL treatment; resident/nonresident relief; eligible insurance/pension/mortgage/PWD cases as applicable; cash/non-cash benefits; partial months; leavers; overtime; unpaid/half-pay leave; loans/arrears; negative/zero net; off-cycle accumulation; year boundary; cross-company transfers; rounding and adjustment linkage. Unsupported scenarios block the affected run and are visible in scope documentation.

**Acceptance evidence:** independently reviewed expected amounts match every supported fixture; line/run/GL/payment controls reconcile; the same snapshot produces identical results; recomputation never edits approved runs; concurrent compute/approve attempts cannot race; a preparer cannot approve their own run; tenant/company isolation holds in exports. Complete two consecutive parallel payroll cycles, explain every variance and obtain payroll-owner sign-off before live cutover.

## 14. G10 — Documents, credentials and letters

Complete the private upload pipeline: authorized upload session, immutable object key, checksum/type/size validation, quarantine, malware scan and publish. Model document versions, verification, supersession, archive, expiry, retention and legal holds. No listed metadata-only file is presented as downloadable or verified.

Build personnel document register, My documents, role-based required checklist, compliance dashboard and renewal tracker. Clinical requirements use configurable issuer/role rules. Pending/unverified/expired mandatory credentials do not count as compliant. Schedule deduplicated reminders and policy-controlled roster warnings with clear owners.

Implement safe letter template editing, versioning, approved merge fields, preview, unresolved-placeholder validation, issuance and a permanent reference. Freeze the issued content and company/employee context. Generate printable/downloadable artifacts with accessible source HTML where feasible; render representative letters/payslips to inspect truncation, page breaks and character support.

**Acceptance evidence:** unauthorized object access and guessed keys fail; uploads with mismatched MIME/malware remain quarantined; archived versions remain authorized historical records; retention respects legal holds; expiry uses actual credential evidence; pending verification does not pass compliance. Issued letters cannot change when templates or employee details change. Missing legacy file bytes are reported and never invented.

## 15. G11 — Recruitment

Implement requisition creation, budget/headcount review and assigned approval. Vacancies inherit approved terms and track opening count/state. Separate a candidate profile from applications, so one person may apply to several vacancies without duplicate personal profiles within a tenant. Candidate sharing across companies requires explicit policy and permissions.

Deliver pipeline/list views, safe stage transitions, interview scheduling, assigned evaluations, offer drafting/approval, acceptance/decline/expiry and candidate-to-employee conversion. Record stage history and use vacancy capacity constraints. Offer terms and letters are versioned. Conversion creates the person/employment, source links and onboarding tasks in an idempotent transaction.

**Acceptance evidence:** unapproved requisitions cannot open vacancies; invalid state jumps are rejected; concurrent hires cannot exceed openings without authorized change; interviewers see only assigned evidence; offer approvals enforce separation of duties; the same accepted offer converts once; attachment permissions carry through correctly. Rejected candidates follow approved retention and reinstatement policy, not indefinite unconditional retention.

## 16. G12 — Reports, configurable UI and improvements

Implement governed headcount, movement, absenteeism, leave balance/liability, payroll cost, credential compliance and recruitment metrics. Define numerator/denominator, time window, employment status, currency and organizational ownership for every KPI. Distinguish current headcount from all records and historical company ownership from present assignment. Use approved projections/materialized views only with tenant-aware freshness and authorization.

Deliver saved filters/views, report jobs, export history, scheduled delivery and dashboard preferences. Export scope is captured and revalidated when executing/downloading. Data masking and row/field restrictions survive CSV/spreadsheet/PDF generation. Scheduled recipients and delivery destinations are explicit; do not attach unrestricted payroll to notification emails.

Build the typed custom-field registry, safe form renderer, versioned workflow/policy configuration and controlled tenant branding. Add feature assignments with prerequisites, expiry/review date, audit and rollback/kill switch. The API enforces the effective feature set. Performance or security overrides cannot be introduced through tenant-provided code.

Build **Improvements & releases**: submit problem/impact/module, review status, roadmap, release notes and linkage to shipped capabilities. Product managers can triage; ordinary users see only permitted requests. Tenant-specific requests do not become globally visible to other SaaS customers. Feature activation is a separate authorized operation from roadmap status.

**Acceptance evidence:** metrics reconcile to source records for defined periods and scopes; small-scope restricted users cannot infer salary through aggregation; saved views cannot resurrect forbidden columns; tenant/company switching removes stale data; feature disable takes effect server-side; custom-field validation and export behavior are consistent; improvement attachments/requests respect tenant privacy.

## 17. G13 — Migration, operations and production launch

### 17.1 Migration rehearsal

1. Inventory source stores and actual file bytes. The mockup uses sharded key-value storage and may be memory-only; obtain a complete authorized export while the source is available.
2. Encrypt and checksum source exports; record provenance, export time and schema version. Preserve source evidence without importing demo credentials.
3. Map source group to tenant, companies to legal employers, and employee IDs to person/employment IDs. Maintain a source-ID mapping table. Separate demo records from real records explicitly.
4. Stage all shards: employees, masters, history, leave, attendance, payroll lines/config snapshots, documents, letters, recruitment, notifications and audit. Record missing shards and metadata-only files as blockers or accepted exceptions.
5. Normalize codes, dates, decimals and time zones; validate company ownership, duplicate numbers, manager relationships, document references, totals and retention status. Preserve original values alongside approved transformations.
6. Load through migration jobs with explicit ownership. Historical payroll retains its original amounts/rule snapshot as imported evidence; do not silently recalculate old approved payroll using new rules.
7. Reconcile by tenant/company/period: employee counts/statuses, leave balances/ledger totals, attendance counts, payroll component/net totals, GL controls, document byte counts/checksums and recruitment stage counts.
8. Run at least one full rehearsal on production-like infrastructure. Time the freeze/export/load/reconciliation process and document manual exceptions.
9. At cutover, freeze source writes for the agreed window, take final export/delta, migrate, reconcile and obtain named business acceptance. Do not keep two systems independently accepting authoritative changes.
10. Retain a time-limited read-only legacy reference under the retention/access policy, and remove unnecessary migration copies after acceptance.

**Rollback boundary:** before production writes start, restore the pre-cutover environment and re-enable the source according to the runbook. After production writes start, reconcile and export those writes before any fallback; restoring an old database blindly loses valid work. Prefer forward correction where safe. Integration callbacks and bank exports stay paused until cutover reconciliation is complete.

### 17.2 Security and recovery

Complete threat-model review covering cross-tenant access, privileged support, exports, SQL injection, object downloads, CSRF/XSS, callback SSRF, credential rotation, forged/replayed device events, dependency compromise and recovery-data exposure. Conduct independent tenant isolation/security review. Record disposition and owners for findings; no unresolved critical/high issue affecting tenant boundaries or financial integrity is accepted for launch.

Exercise PITR into an isolated environment, object/key restoration and tenant-specific recovery. Include a test where only Tenant A is recovered while Tenant B's current data remains untouched. Reconcile outbox/inbox/cursors before enabling deliveries to avoid duplicate notifications or settlement processing. Prove deletion manifests and legal holds survive recovery.

Implement monitoring and runbooks for login failures, unavailable API/database, exhausted pools, oldest jobs, stuck payroll, device offline/backlog/skew, unmapped punches, failed callbacks, expired secrets, file scans and backup freshness. Test on-call alert delivery to approved channels. Define tenant-visible service incident messaging and support escalation.

### 17.3 Capacity and SaaS operations

Run the load profiles and targets in [CONTEXT.md](CONTEXT.md), recording hardware, data cardinality, index plans, concurrency, payload sizes, p50/p95/p99 latency, errors and queue depth. Include several active tenants and a deliberately noisy tenant alongside another tenant's payroll. Test backlog recovery, not only a clean steady state. Test storage growth and retention/partition maintenance.

Verify tenant provisioning/retry, initial admin recovery, subscription/plan changes, quota enforcement, suspension/reactivation, support-access expiry, data export and offboarding. Validate customer-facing terms, processing agreements, subprocessors and retention commitments with the responsible owners. Manual billing is acceptable for launch if the operational process is explicit; hidden usage assumptions are not.

### 17.4 Launch decision

Required evidence: all prior gates pass; two payroll parallel cycles reconcile; the first supported device is certified against its matrix; relevant biometric prerequisites are completed; migration reconciles; tenant isolation/security review passes; recovery targets are demonstrated; key UX journeys pass UAT/accessibility review; and operational owners accept runbooks and alerting.

Roll out to a controlled set of independent pilot tenants with feature flags, then expand after agreed observation and support criteria. Keep separate tenant-level and platform-level rollback procedures. Monitor payroll, access errors, punch reconciliation and callback backlog closely during the first operating period. A single tenant pilot does not establish multi-tenant isolation or fairness.

## 18. Cross-cutting test and evidence matrix

| Concern | Minimum evidence | Gates |
|---|---|---|
| Tenant/company isolation | Runtime-role SQL and API/browser tests across A1/A2/B1/B2; files/jobs/cache/events included | G2 onward |
| Field-level privacy | Assert forbidden fields absent in JSON/HTML/exports/logs, not simply hidden in UI | G2, G4, G9, G12 |
| SaaS lifecycle | Provisioning idempotency, quota fairness, suspension, support expiry, scoped export/deletion | G2, G13 |
| State/concurrency | Double decisions, overlapping leave, duplicate conversion, competing payroll runs | G5–G11 |
| Money | Independently reviewed fixtures, exact decimal arithmetic, invariants and parallel-run reconciliation | G9 |
| Time | Calendar boundaries, night/split shifts, time-zone changes, late/out-of-order events | G6–G8 |
| Durability | Crash before/after commit/ack, lost lease, retries, poison jobs and recoverable backlogs | G5, G8, G9 |
| Integrations | Real device matrix, authenticated intake, duplicate/replay handling and reconciliation | G8 |
| Callback security | Raw-byte signature tests, SSRF variants, secret rotation, permission revocation | G8 |
| Files | Quarantine, scan failures, MIME mismatch, unauthorized signed access and missing legacy bytes | G10 |
| UI/accessibility | Responsive review, keyboard/screen reader, focus, errors, chart alternatives and measured UX | G3 onward |
| Recovery | PITR + objects + keys + integration checkpoints + isolated tenant recovery | G13 |
| Migration | Signed reconciliation by tenant/company/period and documented exceptions | G13 |

Use real PostgreSQL for RLS, constraints, transactions and query behavior; EF's in-memory provider cannot establish these guarantees. Use unit tests for deterministic domain rules, integration tests for transactional and authorization invariants, contract tests for API/events, and Playwright for complete user journeys. Choose targeted tests that detect meaningful failures rather than duplicating implementation details.

Existing mockup test mapping: `test.js`, `test2.js`, `test3.js` → foundation/people/security; `test4.js` → persistence/recovery; `test5.js` → leave; `test6.js` → attendance; `test7.js`, `test8.js` → payroll/letters; `test9.js` → documents; `test10.js` → multi-company behavior; `test11.js` → recruitment/transfers. Preserve useful scenarios while replacing incorrect expected statutory or attendance behavior with reviewed fixtures.

## 19. Definition of done and progress tracking

A feature is done when its permitted users can complete the workflow through the real UI/API/database, forbidden users cannot access its data or actions, state/financial invariants hold under concurrency, failures are recoverable, outputs reconcile, accessibility is reviewed, and operations can diagnose it. Configuration, migration impact, API/event changes and relevant runbooks must be documented. No mocked API responses or placeholder integration success messages remain in an accepted production feature.

For each gate, maintain one record with: owner, status, dependencies, accepted scope, unresolved decisions, relevant PRs/commits, test commands/results, screenshots where useful, data reconciliation, known limitations and the date/person accepting it. Evidence must be sanitized and must not contain employee payroll, secrets or biometric data.

Suggested statuses: Not started → In progress → In review → Accepted; Blocked includes the external dependency, owner and next action. A gate remains blocked if a required real-device, financial or recovery proof is unavailable. Product scope changes update both this document and [CONTEXT.md](CONTEXT.md), with the reason and affected gate recorded in an ADR or decision log.

## 20. Research references for implementation

The full source inventory and local findings are in [CONTEXT.md](CONTEXT.md). The most consequential implementation references are:

- Microsoft: [.NET lifecycle](https://dotnet.microsoft.com/en-us/platform/support/policy), [EF query filters](https://learn.microsoft.com/en-us/ef/core/querying/filters), and [ASP.NET Core OpenAPI](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/openapi/overview?view=aspnetcore-10.0).
- PostgreSQL: [Row security](https://www.postgresql.org/docs/18/ddl-rowsecurity.html) and [PITR](https://www.postgresql.org/docs/18/continuous-archiving.html).
- Vercel: [Next.js support](https://nextjs.org/support-policy) and [data security](https://nextjs.org/docs/app/guides/data-security).
- NSSF Kenya: [2026 Year 4 employer notice](https://www.nssf.or.ke/notice-to-employers-year-4-2026-nssf-contribution-rates); KRA: [PAYE guidance](https://www.kra.go.ke/individual/filing-paying/types-of-taxes/paye).
- ODPC: [Final 2025 biometric guidance](https://www.odpc.go.ke/wp-content/uploads/2025/11/ODPC-%E2%80%93-Guidance-Note-on-Biometric-Data.pdf), especially pages 32–34. Confirm applicable governance and lead times before the live pilot.
- Integration/security: [Standard Webhooks specification](https://github.com/standard-webhooks/standard-webhooks/blob/main/spec/standard-webhooks.md), [OWASP SSRF prevention](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html), and the vendor-specific sources in the context document.
- Accessibility: [WCAG 2.2](https://www.w3.org/TR/WCAG22/).

Revalidate supported versions and time-sensitive policies at the relevant gate. Vendor marketing confirms an integration option exists; only target-version documentation and a tested device establish the implemented contract.
