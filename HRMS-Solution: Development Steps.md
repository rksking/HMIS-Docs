# HRMS Development Steps

Build a multi-tenant HRMS using PostgreSQL 18, ASP.NET Core 10 and Next.js 16.

- [CONTEXT.md](CONTEXT.md) defines product rules and architecture.
- [Frontend Architecture Guide.md](Frontend%20Architecture%20Guide.md) defines the Next.js folder structure.
- [instruction.md](../instruction.md) defines coding rules.
- Mark completed work with `-- DONE`. Do not mark a gate complete while required verification remains.

## Current status

| Gate | Status | Main result |
|---|---|---|
| G0 Discovery | In progress | Scope and architecture documented; business/vendor approvals remain |
| G1 Foundation | In progress | .NET, PostgreSQL, Next.js, migrations and CI foundation created |
| G2 Identity and tenancy | In progress | Tenant lifecycle, company scope, RLS and OIDC boundary created |
| G3 Design system and shell | In progress | Login, dashboard, navigation and feature-page previews created |
| G4–G13 | Not started | Domain and production work remains |

## Development rules

1. Complete backend, frontend and tests for each feature together.
2. Keep API controllers and Next.js routes thin.
3. Use application-layer repository interfaces and EF Core infrastructure implementations.
4. Every business record must have tenant ownership; company-owned records also require company ownership.
5. PostgreSQL RLS and server authorization are mandatory. UI hiding is not security.
6. Use synthetic data in development and tests.
7. Use generated API contracts between .NET and Next.js.
8. Do not place secrets, payroll data, employee data or biometric data in source or logs.

## G0 — Discovery and approvals

### Completed

- `-- DONE` Read the mockup, product context and frontend architecture guide.
- `-- DONE` Confirm multi-tenant SaaS for independent business groups.
- `-- DONE` Inventory mockup routes, models, tests and known defects.
- `-- DONE` Create initial architecture decisions and coding instructions.

### Remaining

- `-- TODO` Product owner approves launch scope and exclusions.
- `-- TODO` Payroll owner approves Kenya payroll rules and fixtures.
- `-- TODO` Obtain biometric vendor, model, firmware and API/SDK information.
- `-- TODO` Complete biometric privacy/DPIA review.
- `-- TODO` Assign product, payroll, privacy, integration and operations owners.

Evidence: [G0_DISCOVERY.md](evidence/G0_DISCOVERY.md)

## G1 — Platform foundation

### Backend

- `-- DONE` Create .NET 10 API, worker, migration runner and shared kernel.
- `-- DONE` Create SaaS domain, application and infrastructure projects.
- `-- DONE` Configure EF Core 10, Npgsql and PostgreSQL migrations.
- `-- DONE` Add repository-pattern tenant persistence.
- `-- DONE` Add health endpoints, Problem Details, correlation IDs and OpenTelemetry.
- `-- DONE` Separate migration owner and runtime database roles.
- `-- TODO` Add PostgreSQL integration tests to CI.
- `-- TODO` Add durable worker job storage and restart verification.

### Frontend

- `-- DONE` Create Next.js 16 and Node 24 project with locked dependencies.
- `-- DONE` Follow the View-Component-API folder structure.
- `-- DONE` Add API client wrapper and environment validation.
- `-- TODO` Generate the TypeScript client from OpenAPI.
- `-- TODO` Fail CI when the generated client differs from the API contract.

### Delivery

- `-- DONE` Add local Compose services for PostgreSQL, MinIO, Mailpit and telemetry.
- `-- DONE` Add backend, frontend and Compose CI checks.
- `-- TODO` Build versioned API, worker and web container images in CI.
- `-- TODO` Add secret, dependency, licence and container scanning.
- `-- TODO` Produce SBOM and build provenance.

Evidence: [G1_FOUNDATION.md](evidence/G1_FOUNDATION.md)

## G2 — Identity, tenancy and permissions

### Backend

- `-- DONE` Add tenant lifecycle states.
- `-- DONE` Add tenant-owned companies with tenant-unique company codes.
- `-- DONE` Add tenant-scoped repository access and EF query filters.
- `-- DONE` Add PostgreSQL RLS with transaction-local tenant context.
- `-- DONE` Block company creation for provisioning and suspended tenants.
- `-- DONE` Add provider-neutral OIDC code-flow and PKCE configuration.
- `-- DONE` Add secure cookie settings and safe return-URL validation.
- `-- TODO` Configure and test a real OIDC provider.
- `-- TODO` Add users and external identity mappings.
- `-- TODO` Add tenant memberships and explicit tenant selection.
- `-- TODO` Add roles, permissions and company grants.
- `-- TODO` Add MFA policy, logout, session revocation and access-change invalidation.
- `-- TODO` Add tenant provisioning, invitations, plans and quotas.
- `-- TODO` Add suspension, reactivation, export and offboarding workflows.
- `-- TODO` Add tenant-approved, time-limited support access.
- `-- TODO` Add security audit records for login and access changes.
- `-- TODO` Test cross-tenant access, pooled connections, raw SQL, jobs and exports.

### Frontend

- `-- DONE` Connect the login button to the API OIDC challenge endpoint.
- `-- DONE` Keep local preview separate from real sign-in.
- `-- TODO` Load the authenticated session from the API.
- `-- TODO` Build tenant selection for users with multiple memberships.
- `-- TODO` Make navigation permission-aware.
- `-- TODO` Build users, invitations, roles and permission assignments.
- `-- TODO` Build tenant provisioning and lifecycle-status screens.
- `-- TODO` Build support-access request, approval and expiry screens.
- `-- TODO` Add secure logout and session-expired handling.

Evidence: [G2_TENANT_SCOPE.md](evidence/G2_TENANT_SCOPE.md)

## G3 — Design system and application shell

### Frontend

- `-- DONE` Build responsive login and post-login layouts.
- `-- DONE` Build the dark sidebar, header, company selector and user summary.
- `-- DONE` Build the group dashboard layout and KPI cards.
- `-- DONE` Add preview routes for all mockup feature areas.
- `-- DONE` Add loading and route-level error components.
- `-- TODO` Create reusable Button, Input, Select, DatePicker, Dialog and Drawer components.
- `-- TODO` Create reusable DataTable, FilterBar, Pagination, Tabs and status components.
- `-- TODO` Implement responsive mobile navigation.
- `-- TODO` Implement working global employee search.
- `-- TODO` Make tenant/company switching clear cached and rendered data.
- `-- TODO` Add permission-aware breadcrumbs and navigation.
- `-- TODO` Add complete loading, empty, error and retry states.
- `-- TODO` Add accessible charts with text/table alternatives.
- `-- TODO` Complete keyboard, screen-reader, contrast and mobile testing.
- `-- TODO` Measure frontend performance.

### Backend

- `-- TODO` Provide session, navigation-permission and company-context APIs.
- `-- TODO` Provide narrow dashboard and search DTOs.
- `-- TODO` Reject unauthorized company switching and stale scope versions.

Evidence: [G3_UI_SHELL.md](evidence/G3_UI_SHELL.md)

## G4 — Organization and employees

### Backend

- `-- TODO` Add divisions, departments, sites, jobs, grades and cost centres.
- `-- TODO` Add employee, employment, assignment and transfer history.
- `-- TODO` Separate salary, bank, government ID and medical data.
- `-- TODO` Add employee create, update, deactivate, rehire and transfer workflows.
- `-- TODO` Add import upload, mapping, validation, duplicate review and reconciliation.
- `-- TODO` Add tenant/company constraints, RLS, authorization and audit.

### Frontend

- `-- TODO` Build organization-management screens.
- `-- TODO` Build employee directory with search, filters and pagination.
- `-- TODO` Build Employee 360 and self-profile pages.
- `-- TODO` Build employee create/edit/transfer flows.
- `-- TODO` Build import mapping, preview, error and progress screens.

## G5 — Jobs, approvals, audit and notifications

### Backend

- `-- TODO` Add PostgreSQL durable jobs, outbox and inbox.
- `-- TODO` Add retries, leases, deduplication and poison-job handling.
- `-- TODO` Add versioned approval definitions, assigned steps and decisions.
- `-- TODO` Enforce delegation, escalation and separation of duties.
- `-- TODO` Add append-only business audit storage.
- `-- TODO` Add email/in-app notification templates and delivery tracking.

### Frontend

- `-- TODO` Build approval inbox, detail, history and delegation screens.
- `-- TODO` Build background-job progress and failure recovery UI.
- `-- TODO` Build notification inbox and preferences.
- `-- TODO` Build audit search and detail views.

## G6 — Leave

### Backend

- `-- TODO` Add effective-dated leave types and policies.
- `-- TODO` Add calendars, accrual ledger, balances and carry-forward rules.
- `-- TODO` Add request, approval, cancellation and adjustment workflows.
- `-- TODO` Handle calendar-month maternity rules and policy changes.
- `-- TODO` Add overlap, concurrency, tenant and company checks.

### Frontend

- `-- TODO` Build leave balance, request and history pages.
- `-- TODO` Build team calendar and manager review pages.
- `-- TODO` Build leave policy and adjustment administration.

## G7 — Attendance and shifts

### Backend

- `-- TODO` Add shifts, rosters, holidays and employee assignments.
- `-- TODO` Store immutable raw punches and derived work intervals.
- `-- TODO` Add missing-punch and schedule exceptions.
- `-- TODO` Add attendance corrections, overtime and monthly close.
- `-- TODO` Handle overnight shifts, multiple punches and late events.

### Frontend

- `-- TODO` Build daily and monthly attendance registers.
- `-- TODO` Build shift/roster scheduling.
- `-- TODO` Build exception correction and approval screens.
- `-- TODO` Build attendance import and reconciliation screens.

## G8 — Biometric devices and callbacks

### Backend

- `-- TODO` Add device inventory, credentials, sites and employee mappings.
- `-- TODO` Build vendor-neutral punch ingestion and deduplication.
- `-- TODO` Build at least one real device/vendor adapter.
- `-- TODO` Add device health, checkpoints, replay and reconciliation.
- `-- TODO` Add callback subscriptions, signing, retries and delivery history.
- `-- TODO` Add SSRF protection, secret rotation and payload redaction.

### Frontend

- `-- DONE` Add the devices/callbacks preview route.
- `-- TODO` Build device setup, mapping, sync and health pages.
- `-- TODO` Build callback subscription and secret-rotation pages.
- `-- TODO` Build delivery logs, retry and replay controls.

## G9 — Payroll

### Backend

- `-- TODO` Add pay groups, components, employee inputs and effective-dated rules.
- `-- TODO` Implement Kenya PAYE, NSSF, SHIF and housing levy using approved fixtures.
- `-- TODO` Use decimal arithmetic and explicit rounding.
- `-- TODO` Add payroll snapshots, calculation, reconciliation and approval.
- `-- TODO` Add immutable approved revisions and adjustment runs.
- `-- TODO` Add payslips, bank files, statutory exports and balanced GL exports.

### Frontend

- `-- TODO` Build payroll run workspace and exception review.
- `-- TODO` Build reconciliation and maker/checker approval.
- `-- TODO` Build employee payslip pages.
- `-- TODO` Build payroll settings, statutory rate and export pages.

## G10 — Documents, credentials and letters

### Backend

- `-- TODO` Add private object-storage upload sessions.
- `-- TODO` Add checksum, MIME, size, quarantine and malware-scan validation.
- `-- TODO` Add file versions, verification, expiry, retention and legal holds.
- `-- TODO` Add credential requirements and compliance evaluation.
- `-- TODO` Add versioned letter templates and immutable issued letters.

### Frontend

- `-- TODO` Build employee document and credential pages.
- `-- TODO` Build upload, verification, expiry and renewal flows.
- `-- TODO` Build letter template, preview, issue and register pages.

## G11 — Recruitment

### Backend

- `-- TODO` Add requisitions, approvals, vacancies and opening limits.
- `-- TODO` Add candidate profiles and separate applications.
- `-- TODO` Add interviews, evaluations, offers and approval history.
- `-- TODO` Add accepted-offer conversion to employee.
- `-- TODO` Add candidate retention and access policies.

### Frontend

- `-- TODO` Build recruitment dashboard and pipeline.
- `-- TODO` Build requisition and vacancy pages.
- `-- TODO` Build candidate profile, application, interview and offer pages.
- `-- TODO` Provide keyboard-accessible alternatives to drag-and-drop stages.

## G12 — Reports and improvements

### Backend

- `-- TODO` Define governed metrics and authorized reporting projections.
- `-- TODO` Add report export and scheduled delivery jobs.
- `-- TODO` Add saved views and server-enforced feature flags.
- `-- TODO` Add custom-field definitions with validation and audit.
- `-- TODO` Add improvement requests, roadmap items and release notes.

### Frontend

- `-- TODO` Build headcount, movement, leave, attendance, payroll and recruitment reports.
- `-- TODO` Build saved views and report scheduling.
- `-- TODO` Build custom-field administration.
- `-- TODO` Build improvement request, roadmap and release-note pages.

## G13 — Production launch

### Backend and infrastructure

- `-- TODO` Build and rehearse data migration with signed reconciliation.
- `-- TODO` Configure staging and production infrastructure.
- `-- TODO` Configure secrets, key rotation, monitoring and alerts.
- `-- TODO` Test backup, point-in-time recovery and object restoration.
- `-- TODO` Complete security, performance, capacity and failure testing.
- `-- TODO` Complete tenant export and offboarding tests.

### Frontend and operations

- `-- TODO` Complete supported-browser and accessibility testing.
- `-- TODO` Complete user-acceptance testing for each role.
- `-- TODO` Write support, incident, recovery and payroll runbooks.
- `-- TODO` Train administrators and pilot users.
- `-- TODO` Complete parallel payroll and controlled pilot rollout.

## Next execution order

1. `-- TODO` Finish G2 users, memberships, roles and permission enforcement.
2. `-- TODO` Finish G3 reusable UI components and authenticated shell behavior.
3. `-- TODO` Build G4 organization and employee management as the first full business module.
4. `-- TODO` Build G5 durable jobs, approvals and audit before leave, attendance and payroll.
5. Continue G6 through G13 in order while biometric and payroll owners complete G0 approvals.

## Gate completion rule

A gate is complete only when its backend, frontend, database constraints, authorization, audit, automated tests and required business evidence pass. Record proof under `docs/evidence/` and then change the status to **Accepted**.
