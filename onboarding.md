# Talent Acquisition (ATS) & Employee Onboarding System Specification

## Implementation Progress Tracker

| Step # | Task / Workflow Step | Status | Description |
| :--- | :--- | :--- | :--- |
| **Step 1** | **Architecture Review, Walkthrough Screen & Master Plan** | `[Completed]` | Review all existing modules (`/recruitment`, `/task`, `/letters`, `/documents`, `/employees`), incorporate all 9 user mockup screens, create walkthrough review plan, generate `docs/ats.md` with step-by-step breakdown. |
| **Step 2** | **Database Schema & SQL Server Alignments** | `[Completed]` | Extend `offers`, `candidates`, `approval_matrix_rules` and add `candidate_onboarding_profiles`, `candidate_onboarding_educations`, `candidate_onboarding_documents`, `candidate_onboarding_experiences`, `onboarding_audit_logs`. Register permissions & menu (`/onboarding`) in SQL Server. |
| **Step 3** | **Approval Matrix & `/task` Engine for Offer Letters** | `[Completed]` | Integrate Offer Letter approval requests with dynamic `ApprovalMatrixRule` (`MatrixType = 'OFFER'`), assign approvers dynamically, update `/task` decision engine, enforce mandatory remarks on rejection, record audit history. |
| **Step 4** | **Applications → Create Offer Letter Drawer (Mockup 4)** | `[Pending]` | Implement 50% slide-over `CreateOfferLetterDrawer.tsx` with 4-step wizard, letterhead selector, template selector, authorized signatory selector, auto-filled candidate/requisition info, live preview, and document attachments. Transition candidate to `OL Creation` in `Pending Approval` (deep-orange styling). |
| **Step 5** | **Reusable Email Service & Offer Email Dispatcher** | `[Pending]` | Build backend `IEmailService` reading credentials strictly from `appsettings.json`. Attach approved OL and optional documents. Generate cryptographically secure tokenized link with configurable expiry. |
| **Step 6** | **Candidate Offer Acceptance Public Portal (Mockup 3)** | `[Pending]` | Public tokenized route `/offer/accept/[token]`. Display letterhead PDF/HTML viewer, terms table, optional remarks, Accept (green) and Decline (red with mandatory remarks) actions. Transition candidate to `Hired` (Accepted) or `Final` (Rejected). |
| **Step 7** | **ATS Kanban Dynamic Flow Alignment** | `[Pending]` | Configure default ATS pipeline stages: `Applications` → `OL Creation` → `Offer` → `Hired` → `Final`. Connect live transitions with SQL Server persistence and deep-orange `Pending Approval` badges. |
| **Step 8** | **Dedicated `/onboarding` Pipeline Dashboard (Mockup 5)** | `[Pending]` | Create dedicated frontend feature slice `src/modules/onboarding/` and backend controller/service. Implement 5 stat cards, department & search filters, and 4 dynamic columns (`Accepted`, `Pending with Candidate`, `Pending with HR`, `Activate Candidate to Employee`). |
| **Step 9** | **Initiate Onboarding Drawer (Mockup 2)** | `[Pending]` | When candidate is in `Accepted`, clicking `Initiate Onboarding` opens 50% slide-over `InitiateOnboardingDrawer.tsx` with 4-step wizard, candidate & joiner details, template selector, expiry/reminder settings, and email preview. Dispatch tokenized link and advance stage to `Pending with Candidate`. |
| **Step 10** | **Candidate Multi-Step Onboarding Form Portal (Mockup 1)** | `[Pending]` | Public tokenized route `/onboarding/form/[token]`. Implement 7-step candidate onboarding portal with circular completion progress, left step navigation, photo & signature upload, Address, Education (multi-record), Experience, Bank/Payroll, Statutory, and Documents. On submit advance stage to `Pending with HR`. |
| **Step 11** | **Pending with HR Review Drawer (Mockup 9)** | `[Pending]` | Right-side 50% drawer `ReviewOnboardingSubmissionDrawer.tsx` for HR to review submitted bio, contact, education, experience, banking, statutory, and compliance documents. Support `Send Back` (mandatory remarks, candidate email notification, status revert to `Pending with Candidate`) and `Approve` (advances to `Activate Candidate to Employee`). |
| **Step 12** | **Activate Candidate to Employee Drawer (Mockup 8)** | `[Pending]` | Final stage drawer `ActivateCandidateDrawer.tsx` with 4-step wizard. Confirm appointment details, auto-generate official employee code using existing sequence service, attach `/documents` policies & contracts, auto-generate user login credentials, and activate employee. |
| **Step 13** | **Onboarding History / Timeline Drawer (Mockup 6)** | `[Pending]` | Implement `OnboardingHistoryDrawer.tsx` showing complete audit trail from offer creation to employee activation with quick actions, key dates, status badge, and notes. |
| **Step 14** | **Onboarding Configuration & Rules (Mockup 7)** | `[Pending]` | Implement `/onboarding/configuration` settings view managing pipeline stages, stage transition rules, general onboarding settings, templates, and document checklist masters. |
| **Step 15** | **End-to-End Workflow Verification Across Roles** | `[Pending]` | Verify complete lifecycle: Requisition → Candidate → OL Creation → `/task` Approval → Candidate Acceptance → Onboarding Submission → HR Review/Send Back → Employee Activation. Validate audit log traceability across all state changes. |

---

## Complete 9-Screen Mockup Index

1. **Mockup 1: Candidate Public Onboarding Multi-Step Form Portal** (`/onboarding/form/[token]`)
   - 7-step wizard: Personal, Contact & ID, Education, Previous Employment, Bank & Payroll, Documents Upload, Review & Submit.
   - Circular progress ring, left step navigation, profile photo & signature upload, address fields, quick tips.

2. **Mockup 2: Initiate Onboarding Right-Side Drawer** (`InitiateOnboardingDrawer.tsx`)
   - 4-step wizard: Candidate Details, Onboarding Configuration, Email & Message, Confirmation.
   - Pre-filled candidate and joiner information, template selector, expiry/reminder settings, email preview with merge fields.

3. **Mockup 3: Candidate Public Offer Acceptance / Rejection Portal** (`/offer/accept/[token]`)
   - Embedded PDF/HTML letter viewer with download button.
   - Summary terms table, remarks textarea, Accept (green) and Reject (red, mandatory remarks) buttons, important notes callout.

4. **Mockup 4: Applications → Create Offer Letter Drawer** (`CreateOfferLetterDrawer.tsx`)
   - 4-step wizard: Letter Details, Letter Content, Review & Preview, Submit for Approval.
   - Reuses `/letters` letterheads, templates, authorized signatories. Live letter preview & attachments upload.

5. **Mockup 5: Employee Onboarding Pipeline Board** (`/onboarding`)
   - 5 top stat cards: Total Onboarding, Accepted, Pending with Candidate, Pending with HR, Activate to Employee.
   - Department filter, search bar, Board Settings, List/Board toggle.
   - 4 dynamic columns: `Accepted`, `Pending with Candidate`, `Pending with HR`, `Activate Candidate to Employee`.

6. **Mockup 6: Onboarding History / Timeline Drawer** (`OnboardingHistoryDrawer.tsx`)
   - Full chronological audit timeline from Offer Created to Employee Activated and Joined Organisation.
   - Quick actions (`Download Timeline (PDF)`, `Resend Communication`, `View Employee Record`), key dates summary, status card, notes.

7. **Mockup 7: Onboarding Configuration Page** (`/onboarding/configuration`)
   - Pipeline Stages table with action buttons and auto-notification toggles.
   - Stage Transition Rules table with allowed roles and conditions.
   - General Onboarding Settings (expiry days, reminders, auto-account toggle).
   - Onboarding Templates & Document Types masters.

8. **Mockup 8: Activate Candidate to Employee Drawer** (`ActivateCandidateDrawer.tsx`)
   - 4-step wizard: Employee Details, Policies & Documents, Account Setup, Review & Activate.
   - Employment information, payroll & bank details, attached documents & policy letters, user login credentials generation with password generator.

9. **Mockup 9: Review Onboarding Submission Drawer** (`ReviewOnboardingSubmissionDrawer.tsx`)
   - Tabbed review: Personal & Contact, Education, Experience, Bank & Payroll, Documents, Additional, History.
   - HR Edit capabilities, photo & signature viewer, document verification badges.
   - HR Remarks textarea with `Send Back` (amber, mandatory remarks) and `Approve` (green) actions.
