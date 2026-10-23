# Web Clock In / Out — Guide, Settings and Work Done

Branch `web-clockInOut-ui` · work dated 22–23 Oct 2026 · technical detail in `docs/policies.md` §182–§190 · browser checklist in the management artifact (section *Clock-in Access & Check*, rows C.1–C.63).

---

## 1. Summary

Employees can clock in and out from the portal on a computer or a phone browser, if HR has given them access. Each company decides whether the punch must come from the work location, needs a photo, needs a remark, must be inside the shift's hours, and must come from the employee's registered phone. HR reviews every web punch, including refused attempts, on one page.

This round of work:

- redesigned the two HR pages to the standard look,
- closed ten gaps found in the clock-in flow (G1–G7, G9, A6, A7),
- added three company settings and one per-employee switch.

Nothing changes for a company until HR changes its settings, with one exception: the new punch-window check starts on **Warn** for every company (see §3).

---

## 2. Pages and who uses them

| Page | Who | What it is for |
|---|---|---|
| **Attendance → Clock-in Access** (`/attendance/clock-access`) | HR (needs *manage attendance*; Export needs *export attendance*) | Setup. Who may clock in from web / phone, who is not ready, registered phones, switching the location check off for one person |
| **Attendance → Clock-in Check** (`/attendance/clock-check`) | HR (same permissions) | Review. Where each web punch happened, its photo and device, and refused attempts |
| **Masters → Shift & Grace Policies → Clock In / Out tab** (`/masters/shift-grace-policies`), box *Clock In-Out Configuration* | HR (once per company) | The company rules listed in §3.1 |
| **Shifts → Roster Planner** (`/shifts/roster`) | Roster planner | Optional location per roster day (`G1 @ ANNEX`) |
| **Self Service → Attendance → clock card** | Employee | Clock In / Clock Out |

Both HR pages list only the employees the user may see: the companies of their workspace and, when location scope is switched on, their locations.

---

## 3. Settings and configuration

### 3.1 Company settings (Shift & Grace Policies → Clock In / Out → *Clock In-Out Configuration*)

All of these sit in one box. Only the punch window is on the *Hours & Overtime* tab, because daily processing uses it too; the box shows its current values and a link to it.

| Setting | Values | Default | Effect |
|---|---|---|---|
| Location check | Off / Warn / Enforce | Off | **Warn** saves the punch and tells the employee and HR they were outside the circle. **Enforce** refuses the punch outside the circle; the attempt is kept for HR |
| Circle size (radius) and GPS precision | metres | per company | Used when a location has no radius of its own |
| Photo | Off / At clock in / At clock in and out | Off | The camera opens before the punch |
| Remark required | on / off | off | Every web clock in / out needs a remark |
| **Clock-out Allowed After (Minutes)** | 0–1440, blank | blank (no wait) | Clock-out is refused until this many minutes after the clock-in. The card counts down |
| **Web Clock-in Outside Punch Window** | Block / Warn / Allow | **Warn** | See §4.4 |
| **Clock in / out from registered phone only** | on / off | off | See §4.5. Was called "Login with specific mobile" |
| Punch window before / after shift (*Punches counted from / until*, on the Hours & Overtime tab) | minutes | company value; a shift policy may override | Used for the clock-in window, the night-shift clock-out and daily processing |

### 3.2 Per employee (Clock-in Access)

| Switch | Where | Notes |
|---|---|---|
| Web clock-in access | Row switch, or tick several → *Give Access* / *Remove Access* | Without it the clock card is hidden and the server refuses. Every change is audited per employee |
| Location check off for this person | Eye → Access tab, or tick several → *Location Check Off* | A reason is required. The last day is optional and switches back on by itself. The position is still recorded |
| Registered phone | Eye → Phone tab → *Reset Phone* (Yes / No) | Clears the phone; the next phone the person clocks in from becomes theirs |

### 3.3 Master data HR must have in place

- **Locations** (Org Masters → Locations) need a map position before the location check is set to Warn or Enforce. Clock-in Access → *Not ready* lists people whose location has none.
- **Shifts** need correct times, because the clock-in window comes from them. People with no shift can clock in at any time.
- **Logins**: an employee with access but no login is listed as *Not ready*.

### 3.4 Time zones (correct clock-in time)

Every web punch is saved as the wall-clock time in the **employee's time zone**, and the clock card's "today" is the employee's own date. Biometric punches are converted from the device's zone to the employee's zone when the two differ.

| Level | Where | Used when |
|---|---|---|
| 1. Employee | Edit Employee (`/assignment`) → **Time Zone**; also Add Employee step 2 and the bulk upload *Time Zone* column | Set for this person (e.g. a remote worker). Blank = next level |
| 2. Work station | Org Masters → Locations → the location → **Time Zone** | Everyone at that site without their own zone. Also the zone of the biometric devices registered at that site |
| 3. Country | Masters → Country master → **Default time zone** | Sites with no zone of their own |
| 4. Company | Company Setup → the company → **Time Zone** (e.g. Lifecare = `Africa/Nairobi`) | Last fallback; also the zone of a device with no work station |

- **Zone missing from the list:** Masters → Employee Lookups → **Time Zones** → *Add Time Zone* switches it on (permission `employee.lookups.manage`, today developer / superadmin). A zone in use cannot be switched off.
- **From when:** a change applies from the next processed day. Days already processed keep their times; re-process them on Attendance → Register → *Process* for those dates if needed.
- **A biometric punch looks hours off:** check the raw time on Attendance → Punch Telemetry. If the device's own clock is wrong, fix it on the device; if the device sits at a site in another zone, set that zone on its work station.
- Not affected by the employee's zone: company-wide "today" on dashboards and reports, leave and resignation dates.

### 3.5 Server / IT

- Serve the portal over **HTTPS**. Phones allow the camera and location only on HTTPS.
- Photo limits (types, width, size) and the location timeout are server settings (`ClockEvidence` section).
- Run `docs/db_changes.sql` on the server. The blocks dated 2026-10-23 add the new columns; each block is safe to run twice.
- No new permissions were added, and no scheduled job is needed.

---

## 4. How it works

### 4.1 Clock in / out (employee)

1. Open Self Service → Attendance. The card shows today's real shift (or Off day / Holiday / No shift assigned), the hours clock-in is allowed today, and which location is checked.
2. Click **Clock In**. Allow location and camera if the company asks for them. Add a remark if required.
3. **One clock-in and one clock-out per day.** A second clock-in or clock-out is refused with the time already recorded. To correct a day, raise a regularisation.
4. **Clock-out wait:** if the company set one, Clock Out works only after the wait, and the card shows the time left.
5. **Night shift:** the next morning the card shows "Night shift of 22 Oct … clock out by 11:00". Clock Out closes last night's shift. This is allowed until the shift end plus the punch window after it.
6. A day already locked by payroll, or in a closed attendance period, refuses the punch.

### 4.2 Location check

- The punch is compared with the **rostered location for that day**, if a published roster names one. Otherwise it is compared with the employee's work location.
- **Warn:** the punch is saved, and the employee sees "you are N m from …; HR can see this".
- **Enforce:** the punch is refused, and the attempt is kept on Clock-in Check as *Refused*. No attendance is recorded and no photo is kept.
- If HR has switched the location check off for the person, the punch is accepted anywhere and shown as *Check off for person*.

### 4.3 Location per roster day

- In the Roster Planner, click a shift cell → *Works at* → pick a location.
- In the upload sheet, write `G1 @ ANNEX` (the location code or name). Exports use the same form, so the file uploads back unchanged.
- A shift swap moves the location with the shift. Draft rosters are ignored.

### 4.4 Clock-in window (punch window)

- **Today's window** = shift start minus the punch window before, to shift end plus the punch window after. For example, G1 08:00–17:00 with 60 / 240 minutes gives **07:00 – 21:00**.
  - This is the same span daily processing uses to read punches.
  - FLEXI shifts use their own start and end.
  - For night shifts the window ends the next morning.
  - Weekly offs and days without a shift have no window.
- **Outside the window**, the setting decides:
  - **Block:** the clock-in is refused.
  - **Warn:** the employee must confirm ("Clock in anyway?"). The punch is saved with a note HR sees on Clock-in Check and in its export.
  - **Allow:** no check (the old behaviour).
- **After midnight**, a clock-in inside last night's shift window is refused (Block and Warn) if last night has no clock-in. It would otherwise be saved on the wrong date; the employee raises a regularisation for last night instead.
- The card also says when a lone clock-in would count as a clock-out ("A clock-in after 13:00 counts as a clock-out unless you clock out later").

### 4.5 Registered phone

- **Signing in is never refused** because of the phone. Leave, payslips and the rest of the portal work on any device.
- When the company setting is on, a **clock in / out from a phone** must come from the registered phone:
  - The first phone the employee clocks in or out from is registered.
  - Any other phone is refused. The card greys out the buttons and says why, and the attempt is kept on Clock-in Check as *Refused*.
  - Computers are not checked.
- HR resets the phone on Clock-in Access.

### 4.6 HR review (Clock-in Check)

- Pick the dates (Today / Yesterday / 7 Days), In / Out, With photo, or click a result box: Inside, Outside, Imprecise GPS, Location not shared, No map point, Not checked, Refused.
- The Eye icon shows:
  - the result in plain words, the distance against the radius, a map, the photo (opening a photo is audited) and the device;
  - why a punch was refused;
  - a punch-window note, when the employee confirmed a clock-in outside the window.
- Export to Excel / CSV.

---

## 5. Step-by-step setup for a company (HR)

1. **Org Masters → Locations:** give each work location its map position, and its own radius if it needs one.
2. **Time zones** (§3.4): check the company's time zone, the country's default and each work station's zone, so web punches and biometric devices land at the right local time. Set an employee's own zone (Edit Employee → Time Zone) only for people working in another zone, e.g. remote staff. Switch a missing zone on under Masters → Employee Lookups → Time Zones.
3. **Shifts:** check that every employee who will clock in has a shift with the correct times. Review the punch window before / after (company value, or a shift policy).
4. **Shift & Grace Policies → Clock In / Out tab** (then **Save Policy**):
   - Set *Location check* to **Warn** for the first weeks, plus the photo and remark settings.
   - Set *Clock-out Allowed After*, e.g. 120, if wanted.
   - Leave *Web Clock-in Outside Punch Window* on **Warn**.
   - Tick *Clock in / out from registered phone only* if wanted.
5. **Clock-in Access:**
   - Give access to the employees who should clock in from the web.
   - Click *Not ready* and fix each person listed (create a login, add a map point).
6. **Week 1–2:** review **Clock-in Check** daily. When Warn results look right, change the location check to **Enforce** and, if wanted, the punch window to **Block**.
7. **People travelling or at outreach:** Clock-in Access → Eye → *Location check for this person* → reason and last day.
8. **New phone or lost phone:** Clock-in Access → Eye → Phone → *Reset Phone*.

---

## 6. What was done

| # | Area | Change | Policies § | Commit |
|---|---|---|---|---|
| 1 | Both HR pages | Redesigned to the standard UI (status boxes, list with export and columns, details drawer). They list only employees the user may see; *Not ready* list; every grant / removal / phone reset audited per employee | §182 | `efe4704` |
| 2 | G1 | Punches refused by Enforce are kept for HR (no attendance, no photo) | §183 | `efe4704` |
| 3 | G2 | In Warn mode the employee is told they were outside | §183 | `efe4704` |
| 4 | G3 | One clock-in and one clock-out per day; a second one is refused instead of overwriting | §183 | `efe4704` |
| 5 | G5 | The card uses the employee's own date (time zone), not the browser's UTC date | §183 | `efe4704` |
| 6 | G6 | A payroll-locked or closed day refuses the punch instead of showing success | §183 | `efe4704` |
| 7 | G9 | The card shows the real shift, not a fixed "Standard Shift 08:00–17:00" | §183 | `efe4704` |
| 8 | New | Location check off per employee, with reason and optional last day | §183 | `efe4704` |
| 9 | G4 | Night shift can be clocked out after midnight; a stray clock-out with nothing to close is refused | §184 | `efe4704` |
| 10 | G7 | Location per roster day (planner, upload, export, swaps, Not ready) | §185 | `efe4704` |
| 11 | Card | A card left open past midnight switches to the new day | §185 | `efe4704` |
| 12 | New | Clock-out wait after clock-in (company setting) | §186 | `09e4c70` |
| 13 | Fix | A web clock-in on a day already marked absent no longer shows "No biometric punch recorded" | §186 | `09e4c70` |
| 14 | New | Clock-in outside the shift's punch window: Block / Warn / Allow; the card shows today's window; after-midnight rule | §187 | `52dd07c` |
| 15 | A6 / A7 | Registered phone checked at every clock in / out instead of at sign-in; sign-in never refused for it; refused attempts kept | §188 | `52dd07c` |
| 16 | Settings page | Shift & Grace Policies redesigned to the standard UI with tabs; all clock settings gathered in one *Clock In-Out Configuration* box | §189 | not committed |
| 17 | Clock card | A day punched at a biometric terminal shows one slim, coloured row (In / Out times, status) instead of the full web-clock layout | §190 | not committed |

**Database** (`docs/db_changes.sql`, blocks dated 2026-10-23; all run on HRMSCore_Local2):

- refused-punch outcome and reason, and the location source on clock events;
- per-employee location-check switch and its last day;
- location per roster day;
- clock-out wait;
- punch-window mode (default Warn) and the punch-window note on clock events.

**Automated tests:** 658 backend tests pass (20 new for this work). The frontend type check is clean.

---

## 7. Testing

The browser checklist is in the management artifact, section *Clock-in Access & Check*:

| Rows | Covers |
|---|---|
| C.1–C.11 | Both pages, filters, grant access, export, phone width |
| C.12–C.21 | Refused punches, Warn message, one session a day, location check off per person, locked day, midnight |
| C.22–C.26 | Night-shift clock-out |
| C.27–C.34 | Roster location, card at midnight |
| C.35–C.39 | Clock-out wait, web-punch remark |
| C.40–C.47 | Punch window: Block / Warn / Allow, after midnight |
| C.48–C.54 | Registered phone at clock in / out |
| C.55–C.62 | Shift & Grace Policies page redesign and the Clock In-Out Configuration box |
| C.63 | Slim biometric clock card |

Run anything that gives access, resets phones or records punches on the test copy (`docs/scripts/stepTestClone.sh up` / `ui`, :3000), not on live data. **None of the rows has been run in a browser yet.**

---

## 8. Management notes

**Decisions taken (23 Oct 2026)**

- Refused location punches are kept for HR, but **no photo** is kept for them.
- **One** clock-in and **one** clock-out per day. Corrections go through regularisation.
- The roster location is set **per day**, not per shift or per roster.
- A clock-in outside the punch window defaults to **Warn**. A clock-in after midnight inside last night's open window is **refused**.
- The registered phone is set at the **first clock-in**, not at sign-in. Refused phone attempts are **kept** on Clock-in Check.

**Known limits**

- A phone browser in "desktop site" mode counts as a computer, so the registered-phone check does not apply to it.
- A clock-in the employee confirmed outside the window (Warn) is still read by daily processing as it decides. For example, a lone clock-in late in the evening becomes a clock-out with a half day; the employee was told this before confirming.
- Clock-out outside the window is not checked.
- A day shift's clock-in is not closed the next day. Only night shifts carry over; anything else needs a regularisation.
- Location accuracy depends on the phone's GPS. Use Warn first and review the *Imprecise GPS* results before choosing Enforce.

**Before go-live**

1. Run the 2026-10-23 database blocks on the server.
2. Serve the portal over HTTPS.
3. Give map positions to the work locations, and to any location used on the roster.
   Check the time zone of the company, country and each work station (§3.4).
4. Choose per company: location check (start on Warn), photo, remark, clock-out wait, punch-window mode, registered phone.
5. Give clock-in access, then clear the *Not ready* list.
6. Run the browser checklist C.1–C.63 on the test copy.
7. Merge `web-clockInOut-ui` into `main` (pull request not yet opened).

**Related open items outside this work**

- The Company HR role (e.g. login 68032) does not have *manage attendance*, so it cannot open these pages. Give it the permission, or use an HR Manager login.
- The attendance-policy fix steps F1–F21 (deferred on 7 Oct) are still open. F5 (biometric punches matched to the right person) and F8 (approved corrections kept on recalculation) also affect attendance accuracy.
