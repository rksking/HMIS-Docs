# UI Design Standard

Every new or redesigned page and drawer follows this file. All classes live in `frontend/src/app/globals.css` under **"global scope styling"**. Reference page: `/employeedirectory` & `/employees/separation-tracker`
Drawer: `/portal/attendance` > `New Request`
Top status boxes design: `/portal/attendance`
Search by employee code, name or designation: `/assignment`
NOTE: the entire UI under each tab and drawer should be similar to '/portal/attendance' (My Requests and New Request) and '/employeedirectory' for your reference work

Rules:
- Use these `standered-*` classes instead of re-typing Tailwind colour, size or radius utilities for the same thing.
- Tailwind is still fine for layout (`flex`, `grid`, `gap-*`, `p-*`, `w-*`) and for one-off tweaks. Utilities can override the component classes because those classes sit in `@layer components`.
- Never hardcode a new colour for a button, tab, badge or input. If a new kind is needed, add it to `globals.css` and to this file first.
- Spelling `standered` is intentional. Keep it the same for every new class.

---

## 1. Corner radius — always 4px

| Class | Use on |
| :-- | :-- |
| `standered-form-radius` | form / section cards, page panels, the stepper strip |
| `standered-card-radius` | stand-alone cards: metric, summary, preview, list / result cards |
| `standered-box-radius` | inner boxes, info banners, dropdown panels, icon tiles |
| `standered-input-radius` | `input`, `select`, `textarea`, search boxes |
| `standered-button-radius` | every button. The button radius is set only by this class |

- Do not use `rounded-sm/md/lg/xl/2xl/3xl`.
- Keep `rounded-full` only for true circles and pills (avatars, step numbers, radio dots, status badges).
- For a single side, use `rounded-t-[4px]` etc.

## 2. Buttons

A button is always **base + colour variant + radius**, and optionally a size:

```tsx
<button className="standered-btn standered-btn-add standered-button-radius">
  <UserPlus /> Add Employee
</button>
```

- **Base `standered-btn`:** height 38px, padding 0 16px, font 13px semibold, gap 8px. Icons are sized to 16px automatically.
- **Sizes:**
  - `standered-btn-sm`: 32px, for table rows, cards and compact toolbars.
  - `standered-btn-lg`: 44px, for a hero call to action only.

### 2.1 Light buttons (white or tinted background, coloured border and text)
Use these for page header and toolbar actions.

| Variant | Colour | Use for |
| :-- | :-- | :-- |
| `standered-btn-neutral` | white / slate | Back |
| `standered-btn-cancel` | slate tint | Cancel |
| `standered-btn-refresh` | cyan tint | Refresh, Reload |
| `standered-btn-export` | teal tint | Export, Export CSV |
| `standered-btn-download` | indigo tint | Download |
| `standered-btn-filter` | orange tint | Filters |
| `standered-btn-add` | green tint | Add Employee, Add \*, New \* |
| `standered-btn-upload` | purple tint | Bulk Upload, Import CSV |
| `standered-btn-info` | blue tint | Bulk Documents, View details |
| `standered-btn-guide` | sky tint | Workflow Guide, Help |
| `standered-btn-warning` | amber tint | Data Quality, Review issues |
| `standered-btn-danger-light` | rose tint | Remove, Revoke (non-final) |

### 2.2 Filled buttons (solid background, white text)
Use a filled button for the single main action of a form, step or drawer.

| Variant | Colour | Use for |
| :-- | :-- | :-- |
| `standered-btn-primary` | green | Next Step, Continue, Save, Apply |
| `standered-btn-submit` | indigo | Submit, Create & Grant, Send for approval (final commit) |
| `standered-btn-danger` | rose | Delete, Reject (final, destructive) |

### 2.3 Other buttons
- `standered-btn-ghost`: a text-only tertiary action, e.g. Cancel next to Submit.
- `standered-icon-btn`: a 32px icon-only button (close, kebab, row actions). Add `standered-button-radius`.

### 2.4 Order and placement
- **Page header:** actions go on the right, light buttons first and Add \* last, e.g. `Data Quality · Export · Bulk Upload · Bulk Documents · Add Employee`.
- **Forms, steps and drawer footers:** Cancel / Back go on the left or before the main action. The single filled button is always last (far right).
- **One filled button per view.** Everything else is light, ghost or neutral.
- **Loading:** show `<Loader2 className="animate-spin" />` in place of the icon, keep the label, and set `disabled`.

## 3. Page header

```tsx
<div className="standered-page-header">
  <div className="flex items-center gap-4">
    <div className="standered-page-icon"><Users /></div>
    <div>
      <h1 className="standered-page-title">Employee Directory</h1>
      <p className="standered-page-subtitle">Manage your workforce…</p>
    </div>
  </div>
  <div className="standered-page-actions">…buttons…</div>
</div>
```

- Page root stays `<div className="space-y-6">` (follow.md rule 6).
- **Title:** 26px bold. **Subtitle:** one sentence, 13px slate.
- No marketing tags or engine names above the title (e.g. "Stored Procedure Engine").

## 4. Section / card header
- Use `standered-section-title` (15px bold) and `standered-section-subtitle` (12px).
- The card itself is `bg-white border border-slate-200 standered-form-radius p-6`.
- Put an optional icon tile in front: `standered-box-radius` 40px with a tinted background.

## 5. Tabs

| Kind | Classes | Use |
| :-- | :-- | :-- |
| Underline tabs | container `standered-tabs`, item `standered-tab` + `is-active` | page-level sections (Profile / Job / Documents…) |
| Count chip | `standered-tab-count` inside the tab | number of items in that tab |
| Segment | container `standered-segment`, item `standered-segment-item` + `is-active` | small switches inside a card or drawer (All / Global / Company), view toggles |

- The active tab is green, with a 2px green underline.
- Tabs scroll horizontally on small screens. Never wrap them onto two lines.

## 6. Side drawer
- Always use `SideDrawer.tsx` (follow.md rule 7). Its inner parts follow these classes:

| Part | Class |
| :-- | :-- |
| Header row | `standered-drawer-header` |
| Icon tile | `standered-drawer-icon` |
| Title | `standered-drawer-title` (16px bold) |
| Subtitle | `standered-drawer-subtitle` (12px, one line) |
| Body | `standered-drawer-body` (24px padding, 20px gap between sections) |
| Footer | `standered-drawer-footer`: right-aligned, Cancel (`cancel`/`ghost`) then one filled button |

- **Title wording:** an action plus an object, e.g. "New Leave Request" or "Edit Shift".
- **Subtitle:** what the drawer is for, or who it is for.
- **Footer:** keep it visible (sticky). Never put the Save button only at the end of a long body.

### 6.1 Drawer design reference: "New Request" (`/portal/attendance`)
Every new or redesigned drawer copies the look of the **New Request** drawer. The Employee Dossier (`/employeedirectory`) is the reference for a read-only drawer.

| Part | Use |
| :-- | :-- |
| Shell | `PortalTabbedDrawer` (`modules/portal/components/PortalTabbedDrawer.tsx`). White header with tabs fixed under it, a `bg-slate-50/40` body that scrolls, and an optional sticky `footer`. With one tab, no tab strip is shown. |
| Width | `width="50vw"` (default) for a form. `width="75vw"` for content-heavy views such as a dossier, tracker or list with details. |
| Header | Icon tile (`icon`), or `leading` for a person (the `Initials` avatar). `titleExtra` puts status badges inline after the title. The subtitle is one line joined with ` · `. **Never put a separate name/hero card in the body**; the header already does that job. |
| Tabs | Plain labels with an icon, **never numbered** ("1. Overview" reads as a stepper). Add a `count` for lists. Keep to 6 tabs or fewer, and use a `standered-segment` switch inside a tab for related views (e.g. Attendance: Shift / Punch History / Attendance List). |
| Cards | `FormCard` (white, 1px slate-200, 4px radius, p-4) with an `IconTile` (sm, tinted): emerald = job/primary, blue = contact/info, violet = personal/people, amber = attention, indigo = records, slate = notes. Title `standered-section-title`, optional `standered-section-subtitle`. |
| Read-only rows | `InfoList` (`modules/employees/components/dossier/DossierParts.tsx`). Label 12px slate-500, value 13px semibold slate-900, a hairline between rows, and "—" when empty. Use `columns={2}` in a wide card. |
| Forms | `FieldLabel` + `kitInput` / `IconField` (`RequestFormKit.tsx`), inside `standered-form-grid`. |
| Notes | `InfoNote` (sky) for guidance. Error and success messages are an inline banner (`bg-rose-50` / `bg-emerald-50` + icon), **never `alert()` / `confirm()`**. Confirm a delete inside the row ("Delete? Yes / No"). |
| Footer | `FormFooter` for a form (Cancel + one filled button). For a view: Close (`standered-btn-neutral`) on the left and one filled action on the right. |
| Text | Body text is slate: 13px for values and 12px for labels and help. No 10px text and no uppercase tracking-wider headings. |

## 7. Forms

```tsx
<div className="standered-form-grid">
  <div className="standered-field">
    <label className="standered-label">First Name<span className="standered-required">*</span></label>
    <input className="standered-input standered-input-radius" />
    <span className="standered-help">As on national ID</span>
  </div>
</div>
```

- `standered-form-grid` is an auto-fit grid with 220px minimum columns and a 16px gap.
- `standered-input` is used for input, select and textarea: 38px high, 13px text, and a green focus ring. Add `is-invalid` for errors and show `standered-error` text under the field.
- Required fields are marked with `standered-required` (red `*`).
- Group long forms into section cards (§4), not one long card.
- Errors:
  - Show a single error banner at the top of the form (`bg-rose-50 border-rose-200 standered-box-radius`).
  - Show field-level text under each field.

## 8. Status badges
- Base class `standered-badge` (a pill), plus one of:
  - `-success`: Active, Approved, Confirmed
  - `-warning`: Pending, Probation
  - `-danger`: Rejected, Terminated, Inactive
  - `-info`: In progress, Draft submitted
  - `-neutral`: Draft, Closed
- Map the badge from the status value. Never invent a colour per page.

### 8.1 Record codes and name icons
- A record code (department, grade, bank code…) is a mono chip: `standered-code` + one of `standered-code-0` … `-7`. Pick the number from the code itself (hash), so a code always has the same colour. Helper: `CodeText` in `modules/org-masters/components/master-table-kit.tsx`.
- The name cell may lead with a 28px tinted icon tile (`standered-box-radius`) showing what the record is (e.g. layers = department, map pin = facility, stethoscope = clinical job title). Use the same tone as the record's code chip.
- Code colours carry no meaning; never use them for status (§8).

## 9. Tables (data lists)

**Reference: `/employeedirectory` → "Employee List"** (`modules/employee-directory/EmployeeDirectoryView.tsx` + `components/directoryColumns.tsx`). Every list screen copies this layout and behaviour.

### 9.1 Layout (top to bottom)
1. **Table card:** `bg-white border border-slate-200 standered-card-radius`.
2. **Toolbar row** (`px-4 py-3`, title left, tools right, wraps on small screens):
   - **Title block:** a 36px icon tile, **always purple**: `bg-purple-50 text-purple-700 standered-box-radius`. Next to it go the title (16px bold, e.g. "Employee List") and a count line ("N total employees", 12px slate).
   - **Tools, in this order:**
     1. **Export:** a split button. The main part downloads Excel; the ▾ menu offers Excel (.xlsx) / CSV (.csv). It exports the **visible columns** with the current filters and sort. Show it only with the `<module>.export` permission.
     2. **Columns:** a checkbox picker grouped by column group. Choices are saved per user in `localStorage`. It is disabled in grid view.
     3. **Full screen:** a toggle between `Maximize2` and `Minimize2`. The card becomes `fixed inset-0 z-50 bg-white overflow-auto`.
     4. **Sort by:** a select with labelled options (e.g. "Name A–Z", "Newest joiners").
     5. **List / Grid:** an icon toggle (`LayoutList` / `LayoutGrid`). The active one is filled green.
     6. **Per page:** a select. Options come from **Masters → Company Setup → Display Settings** (`useListDisplaySettings`). Never hardcode them; use a fallback only until the settings load.
   - Toolbar buttons use `standered-btn standered-btn-neutral standered-btn-sm standered-button-radius`.
3. **Inline error** under the toolbar (rose box) for export or action failures.
4. **Body:** the table, or the grid of cards.
5. **Pagination footer**, described in 9.4.

### 9.2 Table
- **Lines:** a boxed grid, with horizontal and vertical lines (`border-r` between cells, `border-b` between rows).
- **Header:** `bg-slate-50`, 11px bold uppercase, slate-600, `whitespace-nowrap`.
- **Cells:** `px-3 py-2.5`, 12–13px. Use `—` (slate-400) for empty values.
- **First column `#`:** a running row number, `(page-1)*pageSize + index + 1`.
- **Sortable headers:** clickable, with a ▲/▼ icon. Clicking toggles asc/desc and resets to page 1.
- **Columns:** defined once as a column config array (`key, label, group, sort?, defaultVisible, nowrap?, render`). The same `key` drives the Columns picker and Export.
- **Last column "Actions": icons only.** Never a "View" text label and never a `⋯` menu. View is a green `Eye` icon button that opens the record's info in a drawer (e.g. the Employee Dossier). Every other row action (Edit `Pencil`, Delete `Trash2` in rose, Add Branch `Plus`…) is its own 28px icon button with a tooltip (`title`) and `aria-label`. Keep it to about 3 icons per row.
- **Name cell:** avatar (or coloured initials) + name + code underneath. Clicking the name also opens the View drawer.
- **Status:** shown with `standered-badge-*` (§8).
- **Row hover:** `hover:bg-slate-50/80`.

### 9.3 Grid view
- **Grid:** cards (`standered-card-radius`), `grid-cols-1 sm:2 lg:3 xl:4`, gap 12px.
- **Card content:** avatar + name + code, a status dot, 4–5 key facts with small icons, and a footer with a date on the left and "View →" on the right.

### 9.4 Pagination footer
- **Left:** "Showing a to b of n <items>".
- **Right:**
  - ‹ prev, `1 … 4 5 6 … 20`, next ›. The active page is filled green; 32px square buttons.
  - "Go to page [ ] Go".
- Changing filters, sort or page size always returns to page 1.

### 9.5 States
- **Loading:** a centred spinner plus "Loading <items>…". In future, prefer skeleton rows.
- **Error:** a rose icon, the message and a **Retry** button.
- **Empty:** a large faded icon, "No <items> match your search criteria", and a hint to clear filters.

### 9.6 Suggested improvements (pending your approval)
Build these into the shared table (§9.7), so every list gets them at once:
- **Sticky header** while scrolling, plus a **sticky first column** (name) when scrolling sideways.
- **Row selection:** checkbox column, select all on the page or across all results, and a **bulk action bar** ("3 selected · Export · Send email · Clear").
- **Density toggle:** Comfortable / Compact row height.
- **Column order and width:** drag to reorder, drag the edge to resize; saved per user with the Columns choice.
- **Saved views:** name and save filters + columns + sort ("My team", "Probation ending"), and switch between them.
- **Active-filter chips** above the table, each removable, plus "Clear all".
- **URL-synced state:** page, sort, filters and search kept in the query string, so a list can be shared or bookmarked and Back works.
- **Skeleton loading rows** instead of a blank spinner. Keep the old rows visible (dimmed) while the next page loads.
- **Keyboard:** ↑/↓ moves between rows and Enter opens View.
- **Summary row** (totals or averages) for numeric tables such as payroll and attendance.
- **Copy cell** on hover for codes, emails and phones.
- **Remembered per user:** page size, density, view mode and columns.

### 9.7 Shared component `src/components/ui/DataTable.tsx`
- **Status:** built.
- **Reference use:** `/employeedirectory` (`EmployeeDirectoryView.tsx`).
- **All list screens use `<DataTable>`.** New lists must use it, and existing lists move over to it when they are next touched.
- **Every feature is OFF by default.** Turn one on by passing its flag.
- **Pagination is automatic:**
  - **Server mode:** pass `pagination={{ total, page, onPageChange, onPageSizeChange }}`. The table owns rows per page and reports it through `onPageSizeChange` (on first render and on every change). The page keeps `pageSize` as `null` and fetches only once it is reported.
  - **Client mode:** leave `pagination` out and pass all `rows`. The table sorts and pages them itself.
  - **Page size:** `pageSize` sets the starting rows per page (default **10** when not passed). The per-page choices come from Masters → Display Settings unless `pageSizeOptions` is passed.

**Data**

| Prop | Purpose |
| :-- | :-- |
| `columns` | `{ key, label, group?, sortKey?, defaultVisible?, nowrap?, align?, render?, value? }`. `key` is also the export column key; `value` is used for client-side sorting |
| `rows`, `rowKey` | the data and a unique key per row |
| `title`, `subtitle`, `icon`, `itemLabel` | the toolbar title block (purple icon) and the "Showing … of n `itemLabel`" text |
| `toolbarExtra` | page-specific tools, placed before the built-in ones |
| `filterBar` | page-specific search / filter row, shown under the toolbar (e.g. `/task`) |

**Feature flags**

| Flag / prop | Feature |
| :-- | :-- |
| `isExport` + `onExport(format, visibleKeys)` (+ `exportFormats`) | Export split button: Excel / CSV of the visible columns |
| `isColumns` (+ `storageKey`) | Columns picker, remembered per browser; first visit shows columns with `defaultVisible` ≠ false; "Reset to default" restores that |
| `isFullSized` | Full screen toggle |
| `sorting` (+ `sortOptions`) | clickable sortable headers, plus the "Sort by" select when `sortOptions` is given. Use `sort` + `onSortChange` for server sorting, or `defaultSort` for client sorting |
| `viewType="list" \| "grid" \| "list\|grid"` (+ `renderCard`) | List only (default), Grid only, or both with a toggle |
| `gridClassName` | grid view columns (default 1/2/3/4 at sm/lg/xl); `/task` uses `grid-cols-1 md:grid-cols-2 xl:grid-cols-3` |
| `isRowNumber` | `#` column |
| `isView` + `onView(row)` | Actions → **View** eye icon (no label). Open the record in a drawer |
| `rowActions(row)` | one icon button per action: `{ label, icon, onClick, danger?, hidden? }` (`label` is the tooltip; always pass an `icon`) |
| `isSelectable` + `bulkActions(selected, clear)` | checkbox column and the "N selected" bulk bar |
| `isRowSelectable(row)` | with `isSelectable`: rows that return false get a disabled checkbox. Rows no longer listed drop out of the selection |
| `isStickyHeader` | header stays visible (body scrolls inside 70vh) |
| `isDensity` | Comfortable / Compact row toggle |
| `isGoToPage` | "Go to page" box in the footer |
| `pageSize` (default 10) | starting rows per page, e.g. `pageSize={50}` |
| `loading`, `error` + `onRetry`, `emptyTitle`, `emptyHint` | loading, error and empty states |
| `storageKey` | remembers columns, density and view, e.g. `"employeeDirectory"` |

**Example (server paging)**

```tsx
<DataTable<EmployeeListItem>
  title="Employee List" subtitle={`${total} total employees`} icon={Users} itemLabel="employees"
  columns={columns} rows={employees} rowKey={(e) => e.id}
  isExport onExport={(format, keys) => api.export(filter, keys, format)}
  isColumns isFullSized isRowNumber isGoToPage storageKey="employeeDirectory" pageSize={50}
  sorting sortOptions={SORT_OPTIONS} sort={sortBy} onSortChange={setSortBy}
  viewType="list|grid" renderCard={renderEmployeeCard}
  isView onView={openDossier}
  pagination={{ total, page, onPageChange: setPage, onPageSizeChange: setPageSize }}
  loading={isLoading} error={loadError} onRetry={reload}
/>
```

**Example (client paging, minimal)**

```tsx
<DataTable title="Shifts" columns={cols} rows={shifts} rowKey={(s) => s.id} sorting isView onView={openShift} />
```

**Not built yet** (from §9.6): saved views, active-filter chips, page and filter state kept in the URL, drag to reorder and resize columns, keyboard row navigation, a totals row, copy-cell, skeleton loading rows.

## 10. Colour meaning
- **Green:** primary, success, active.
- **Indigo:** final submit.
- **Rose:** danger.
- **Amber:** warning.
- **Blue / sky:** info and help.
- **Purple:** import / upload.
- Use slate for all text and borders.

## 11. Checklist before finishing a page
- [ ] No `rounded-lg/xl/2xl` left. All radius comes from `standered-*-radius`.
- [ ] Every button uses `standered-btn` + a variant + `standered-button-radius`.
- [ ] There is one filled button per view, and it is last.
- [ ] The header uses the page header classes, and drawers use the drawer classes.
- [ ] Inputs use `standered-input`, and labels use `standered-label`.
- [ ] Badges use `standered-badge-*`.
- [ ] Lists follow §9 (toolbar order, purple title icon, icon-only actions, pagination footer, loading/error/empty states).
- [ ] Layout works at phone width (no horizontal page scroll).

## 12. Stepper (multi-step pages)

**Reference:** `/user/create` (`CreateUserWizardView.tsx`) and `/employees/add` (`modules/employees/add-employee/AddEmployeeView.tsx`).

### 12.1 Shared component `src/components/ui/Stepper.tsx`
- One white strip (`standered-form-radius`, slate border) with the steps side by side, split by thin chevrons.
- **Done step:** green circle with a check. **Active step:** green circle with its number, green title and a 3px green underline. **Next steps:** grey circle with the number.
- Each step shows a name and one short sub-text line (e.g. "Personal & Contact" / "Basic information").
- Clicking a **done** step goes back to it. Later steps are never clickable; the user moves forward only with **Next Step**, so each step's checks run.
- On narrow screens the strip scrolls sideways (min width 760px). It never wraps.

```tsx
const STEPS: StepperStep[] = [
  { id: 1, name: "Personal & Contact", subtext: "Basic information" },
  { id: 2, name: "Employment & Organisation", subtext: "Job, department & location" },
];
<Stepper steps={STEPS} currentStep={step} onStepClick={goTo} />
```

| Prop | Purpose |
| :-- | :-- |
| `steps` | `{ id, name, subtext }[]`, ids 1..n in order |
| `currentStep` | id of the active step |
| `onStepClick(id)` | optional; called only for done steps |

### 12.2 Page layout
1. Page header (§3). The subtitle may change on the review step.
2. `<Stepper>`.
3. One error banner (§7) for the current step.
4. The step body: section cards (§4), one per group of fields.
5. **Footer bar**, sticky at the bottom: **Cancel** (`standered-btn-cancel`) on the left, followed by **Save as Draft** (`standered-btn-neutral`, `Save` icon) when the wizard keeps drafts; **Back** (`standered-btn-neutral`, hidden on step 1) and the single filled button on the right: **Next Step** (`standered-btn-primary`) or, on the last step, the final action (`standered-btn-submit`, e.g. "Complete Onboarding").
- Keep all values in one form hook above the steps (e.g. `useAddEmployeeForm.ts`), so Back, a stepper click or a review **Edit** link never loses typed values.
- **Next Step** checks only the current step; the first problem shows in the banner and under the field.
- Changing step scrolls to the top.

### 12.3 Searchable dropdown `src/components/ui/SearchSelect.tsx`
- Use it instead of a `<select>` when a master list is long (departments, designations, grades, cost centres, shifts). Short lists (employment type, contract type, gender) stay a plain `standered-input` select.
- It filters the already-loaded list on the client by label and `hint` (e.g. a code). For people, use `PersonnelAutocomplete` (server search).
- Keyboard: ↓/↑ move, Enter picks, Esc closes. `clearable` shows an × to empty the value. `invalid` adds the red border.

```tsx
<SearchSelect
  options={departments.map((d) => ({ value: d.id, label: d.name, hint: d.code }))}
  value={form.departmentId ?? ""}
  onChange={(v) => setField("departmentId", v)}
  placeholder="Search department"
  clearable
/>
```
