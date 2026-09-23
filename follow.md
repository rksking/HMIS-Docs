# Developer Master Instructions & Core Principles

## Core Development Guidelines
1. **100% Database-Driven:** Strictly zero hardcoded/mock data or fake fallbacks; all data must originate from SQL Server.
2. **Developer-Friendly Code:** Code must be simple, clean, readable, and human-written. Strictly avoid over-engineered, opaque, or AI-style complex code; adhere fully to Clean Code and SOLID principles.
3. **Stored Procedures for Complex Logic:** Use SQL Server stored procedures (`sp_*`) for difficult calculations, matrices, or aggregations.
4. **Dynamic Country Masters:** Resolve statutory rules, quotas, and governing acts dynamically per employee/company country master (no hardcoded country rules).
5. **Strict Modularity:** Frontend code must be cleanly isolated under `src/modules/<domain>/components/` with public exports in `index.ts`.
6. **Page Layout Standard:** Root must be `<div className="space-y-6">` with an unboxed header, inline title/subtitle, and single horizontal action bar (`flex items-center gap-2`).
7. **50% Slide-Over Drawers:** Use `SideDrawer.tsx` (right-side drawer) for forms, creation, and details; never use centered modal popups.
8. **Permissions & RBAC Registration:** For any new module or action, add permissions to the `permissions` table using standard lowercase dot-notation: `<module_name>.<action>` (e.g. `leave.view`, `leave.apply`, `leave.approve`, `leave.delete`, `leave.export`). Also add new modules to the database `menus` table under their respective parent menu.
9. **No Assumptions — Mandatory Review Plan:** Never guess or assume requirements. Always present a detailed implementation plan for user review and approval before making changes.
10. **Flag Static Values in Plans:** While inspecting or working on any component, if you encounter any static variables, mock arrays, or code violating principles, explicitly highlight them in the plan under a dedicated section: `> Code correction and static values`.
11. **Strict Typing & Build Integrity:** Ensure zero TypeScript errors (`tsc --noEmit`) and zero backend build errors (`dotnet build`).
12. **Real Process Simulation:** Verify workflows end-to-end across real user roles (Employee, Line Manager, HR) with complete database audit trails.

---

## System Credentials Directory

| Persona / Resource | Username / Identifiers | Password | Role / Scope | Details |
| :--- | :--- | :--- | :--- | :--- |
| **Docker (SQL Server)** | Host: `localhost:1433`, User: `sa` | `YourStrong!Passw0rd` | Database: `HRMSCore_Local` | Docker SQL Server Container |
| **Rakesh (Developer / God Mode)** | `rakesh` / `rksthedev@gmail.com` | `Admin@123` | `role_dev` / `DEVELOPER` | **Universal God Mode**: Unrestricted access across all business endpoints, dev tools, and diagnostics (reassignable anytime) |
| **Developer Root** | `developer` / `dev@hrms.internal` | `KingPwd!2032` | `role_dev` / `DEVELOPER` | Dedicated system developer root account |
| **Superadmin** | `superadmin` / `admin@hrms.internal` | `KingPwd!2032` | `superadmin` / `Super Admin` | IT System Administrator (governed strictly by database-assigned `role_permissions`) |
| **Amina Gitau (Employee)**| `employee` / `EMP-STAFF-005` / `amina.gitau@hospital.co.ke` | `KingPwd!2032` | Staff Employee / ESS (`comp-lch-01`) | Reports to Nafula Gitau |
| **Nafula Gitau (Line Manager)** | `manager` / `EMP-MGR-004` / `nafula.gitau@hospital.co.ke` | `KingPwd!2032` | Line Manager / EMS (`comp-lch-01`) | Stage 1 Approver (Department) |
| **Caroline Nduta (HR Manager)** | `hr` / `EMP-HR-002` / `caroline.nduta@hospital.co.ke` | `KingPwd!2032` | HR Manager (`comp-lch-01`) | Stage 2 Approver (Hospital HR) |
| **Wanjiku Muthoni (HRBP / Group HR)** | `wanjiku.muthoni` / `EMP20001` / `wanjiku.muthoni@africare.org` | `Wan@EMP2000126!` *(or `KingPwd!2032`)* | Group HR / HRBP (`comp-afri-01`) | Group-wide HR governance |
