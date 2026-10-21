# Flex Audit Tracker — Admin Integration Notes

Prepared 2026-10-07 for integrating `flex-audit-tracker.html` into the Flex Master Dashboard.
Last updated 2026-10-21.

## What it is

A weekly compliance-audit tracker for the Flex Onsite Team. Each week, every team role
(A, C, B, D, Day Admin, Night Admin) gets a different rotating audit, and "Everyone" also
does the I-9 / W4 / DD audit. For each assignment, staff record Pass/Fail, Manager
Verified, a note, and the Bold IDs involved.

The app also has:
- **Repeat Flags:** any Bold ID recorded more than once, plus a live warning while typing.
- **Overdue Audits:** past weeks that still have missing results.
- **Failed audit correction:** a failed audit can be marked Corrected, which records who and when.
- **Pass Rates by Audit:** pass/fail counts per audit over the last 8 weeks.
- **Past Weeks:** the history of earlier weeks, grouped by calendar year under Show All.
- **Reassign:** in Edit Mode, a manager can have someone else cover a role's audit for one week.
- **Print This Week:** a printable sheet with a manager signature line.
- **Audit Legend:** short summaries of each audit.
- **How-To Guides:** a step-by-step page for each audit.
- **Edit Setup:** a PIN-protected page where staff can change everything without a developer.

- One self-contained file, about 380KB, with no dependencies and no network requests.
- Opens directly in any modern browser (Chrome, Edge, Firefox, Safari).

## How staff use it (plain-language summary)

1. Open the file. It opens on **this week**, and the dates show under the week number.
2. Click **Add Your Name** once per computer, so each result shows who recorded it.
3. For each row, click **Pass** or **Fail**, plus **Manager Verified** when a manager has checked it.
   When a failed audit has been fixed, click **Mark Corrected** on that row.
4. Click **Add Note / Bold ID** to type the Bold IDs involved (separate them with commas) and a note. If an ID has come up before, a red warning appears.
5. **Print This Week** gives a paper copy with a signature line.
6. Managers click **Edit Mode** (bottom of the left menu) and enter PIN **9377** to change:
   - roles and audits,
   - the guide text,
   - the calendar,
   - backups.

## Current storage — important

- **Data lives in each browser's `localStorage`. It is not shared and not synced.**
- Clearing the browser cache deletes it.
- Each computer that has used the file has its own separate copy.
- The app stores these keys:

  | Key | Contents |
  |---|---|
  | `audit-tracker-state` | All app data (schema below) |
  | `audit-tracker-user-name` | That browser's "Recording as" name |
  | `audit-tracker-last-backup` | Timestamp of the last backup downloaded there |
  | `audit-tracker-state-before-restore-<ts>` | Safety copy, not read by the app |
  | `audit-tracker-state-damaged-copy-<ts>` | Safety copy, not read by the app |
  | `audit-tracker-state-unreadable-copy-<ts>` | Safety copy, not read by the app |

- **To collect existing data for migration:** on each computer that used it, go to
  Edit Mode → Edit Setup → Backup & Restore → **Download Backup**. This saves a JSON file.

## Backup file format

```json
{ "app": "flex-audit-tracker", "version": 3, "exportedAt": "<ISO timestamp>", "data": <state> }
```

## State schema (`data` / `localStorage["audit-tracker-state"]`)

Week keys are stringified integers ≥ 1.

```
{
  version: 3,                                // 2 = before E-Verify was replaced; sanitizeState upgrades it
  results: {
    [week]: {
      [roleName]: {
        status:   "Pass" | "Fail" | null,   // mutually exclusive
        verified: boolean,                   // Manager Verified, independent of status
        note:     string,
        boldIds:  string[],                  // uppercased, de-duplicated
        by:       string,                    // "Recording as" name, may be ""
        at:       string,                    // ISO timestamp of last change
        correctedAt: string,                 // ISO timestamp, "" = not corrected; Fail only
        correctedBy: string                  // "Recording as" name of who marked it corrected
      }
    }
  },                                         // empty entries and empty weeks are deleted
  weekSnapshots: { [week]: [ { person, audit, fixed } ] },
  reassignments: {                           // optional; added 2026-10-21
    [week]: { [roleName]: { to: string, reason: string, by: string, at: string } }
  },
  auditDetails:  { [itemName]: string },     // Audit Legend quick summaries
  sopDetails:    { [itemName]: string },     // How-To Guide text
  config: {
    title, subtitle, sidebarTitle, passNote: string,
    weekOneStart:   "YYYY-MM-DD",            // Week 1's first day; weeks are 7 days
    rotatingPeople: string[],                // ORDER DRIVES ROTATION, never reorder
    displayOrder:   string[],                // on-screen order only
    audits:         string[],                // rotating audit slots
    fixedPerson:    string,                  // default "Everyone"
    fixedAudit:     string,                  // default "I-9's & W4 & DD"
    legendItems:    [ { name: string, image: string | null } ]  // image = built-in image key
  }
}
```

- **`weekSnapshots`:** each week with recorded results is frozen into this when the setup
  changes, so editing roles never rewrites history. If `weekSnapshots[N]` exists, it is
  week N's assignment list; otherwise the rotation rule below applies.
- **`reassignments`:** a manager (Edit Mode) can have someone else cover a role's audit for
  one week, for example when that person is out sick. `to` is a team role or a typed name,
  and `reason` is optional.
  - The result stays under the original role, so the rotation, results, Pass Rates and
    Alerts are not affected.
  - It only changes what is shown: "Covered by D · Out sick" on the row, in print, in Past
    Weeks and in Overdue.
  - The fixed "Everyone" row can't be reassigned.
  - Setup renames update both the keys and the `to` values.
- **Rotation rule:** for week N, `offset = (N-1) % audits.length`, and
  `rotatingPeople[i]` gets `audits[(i + offset) % audits.length]`.
- **Older saves are converted automatically** by `sanitizeState()` in the file:
  - they have no `config`,
  - results may be plain strings,
  - they use a single `boldId` string,
  - they use a `currentWeek` counter instead of dates.

  Reuse that function when importing into a backend. It drops bad entries one at a time
  and reports how many (`dropped`); week numbers above `MAX_WEEK` (5200) are rejected.
- **Version 3 migration (E-Verify retired, 2026-10-08):** the rotation slot "Bold Status &
  EAD's & Everify" became "Bold Status & EAD's & Core HR Report", and the Everify legend
  item became **Audit Core HR Report**. `migrateEverifyToCoreHr` runs once on any data with
  `version < 3`. It freezes every recorded week into `weekSnapshots` first, so history
  still says Everify. Don't redo this rename through Edit Setup, because setup renames
  rewrite history snapshots.
- **Saving can be paused on purpose.** If damaged saved data can't be safety-copied
  (storage full), `saveBlocked` stops every save so the only original isn't overwritten.
  The red error bar then offers **Download a Copy of My Data**. A successful restore clears it.

## Guide and Legend content

- **The default text is in `LEGEND_ITEMS`:** `detail` is the Legend summary and `sop` is
  the How-To Guide.
- **When you change a default, add the old text to the refresh table.** Append the previous
  text to `LEGEND_CONTENT_REFRESH` or `SOP_CONTENT_REFRESH`. On every load, saved text that
  still exactly matches an old default is upgraded. Text a user edited is never touched.
  In a backend, run the same rule once against `audit_docs`.
- **File locations vary by site.** Flex/Axiom may use Box, SharePoint, OneDrive or the
  extranet in Teams, depending on the site and its current process. The guides tell staff
  to use their site's current location, so don't hardcode Box.
- **Naming used everywhere:**
  - "Avionte Bold" for the system, and "Bold ID" for the associate number.
  - "Flex Master Dashboard" for the dashboard (the same thing as "Flex Dashboard").
  - "Terminated" in Avionte Bold means three things: an assignment End Date (last day
    worked), a Logged Activity with type Termination, and a status other than On Assignment.

## Suggested backend tables (SharePoint Lists or SQL)

| Table | Columns |
|---|---|
| `audit_results` | week_number, week_start_date, role, status, verified, note, recorded_by, recorded_at, corrected_by, corrected_at |
| `audit_result_bold_ids` | result_id, bold_id |
| `audit_week_assignments` | week_number, role, audit, is_fixed (the snapshots) |
| `audit_reassignments` | week_number, role, covered_by, reason, reassigned_by, reassigned_at (manager-write-only) |
| `audit_config` | single row/JSON with the `config` object |
| `audit_docs` | item_name, legend_summary, how_to_guide, image_key |

Supabase was ruled out; SharePoint Lists through Microsoft Graph was the tentative choice.

**Access control and audit trail (required for the backend):**
- These are compliance records, so who and when must not be spoofable. Right now `by` and
  `correctedBy` come from a free-text "Recording as" name, and anyone can type any name.
  In the backend, set `recorded_by`, `recorded_at`, `corrected_by` and `corrected_at` on
  the server, or in a Power Automate flow, from the signed-in Entra ID user. Don't take
  them from the browser.
- Staff need write access to `audit_results` only. Keep `audit_config`, `audit_docs` and
  `audit_week_assignments` manager-write-only (a SharePoint group). Otherwise anyone could
  change setup or guides directly in SharePoint and bypass the PIN.
- **Graph auth from the browser:** use MSAL with the auth code + PKCE flow (SPA) and
  delegated or `Sites.Selected` permissions. Never put a client secret or app-only
  credentials in browser code.

## Dashboard Alerts integration (required)

The Flex Master Dashboard's existing **Alerts** section must show every failed audit
that has not been corrected, and keep showing it until it is corrected:

- **Alert when:** `status === "Fail"` and `correctedAt` is empty (see `isUncorrectedFail`
  in the file). Include past weeks, not just the current one.
- **Alert text** should name the week (number + date range), the role, and the audit, e.g.
  "Week 3 (Oct 5 – Oct 11): Headcount Accuracy failed (A), not corrected yet." If the
  row was reassigned, add the cover, e.g. "(A, covered by D)".
- **Clears when:** someone clicks Mark Corrected (sets `correctedAt`), or the result is
  changed away from Fail.
- The audit name for a week comes from `weekSnapshots[week]` if present, otherwise from the
  rotation rule above.

The tracker itself intentionally has no separate alert panel for this, so the Alerts
section is the one place staff see it.

## Security notes

- **The PIN is a UI lock only.** `EDIT_PIN = "9377"` is in plain text in the source, and
  edit mode can be bypassed from the browser console. **Keep the PIN as 9377. Do not
  change it** (owner's decision). Server-side role checks (Entra ID / SharePoint group) can
  be added on top once there is a backend.
- **User text is always escaped before it goes on the page.**
  - All typed text goes through `escapeHtml()` before it's inserted into the page.
  - Map lookups are safe against keys like `__proto__`.
  - There is no `eval` or `new Function`.
- **Content Security Policy:**
  - There are no inline event handlers; the app uses `data-action` attributes with one
    delegated listener.
  - The script and styles are inline, and images are `data:` URIs. A strict CSP would need
    `script-src 'unsafe-inline'` (or a hash), `style-src 'unsafe-inline'` and
    `img-src data:` — or move the script and style into external files.
- **Personal data:** notes are asked to contain Bold IDs, not associate names. Data sits
  unencrypted in browser storage.
- **When porting:** don't render notes or guide text with `innerHTML` /
  `dangerouslySetInnerHTML`. Render them as text, the same way `escapeHtml()` does now.
- **CSV/Excel exports:** if the dashboard or SharePoint exports these lists, start any
  note that begins with `=`, `+`, `-` or `@` with `'`. Otherwise Excel can run it as a
  formula (formula injection).
- **Backup restore limits:** files over 5MB are refused (`MAX_IMPORT_BYTES`).

## Embedding considerations

- **JavaScript:** wrapped in an IIFE with `"use strict"`, so it adds no global variables.
  It adds these listeners:
  - on `document`: click, keydown, input, focusout, change;
  - on `window`: `beforeprint` (builds the print sheet for Ctrl+P) and `beforeunload`
    (the unsaved-changes warning);
  - on `#noteBoldIds`: `input` (the live repeat-ID warning).
- **Fixed element IDs:**
  - Page layout: `sidebar`, `mainContent`, `toast`, `errorBar`, `errorText`, `modalRoot`,
    `modalOverlay`, `printArea`.
  - Pop-up and form fields: `noteBoldIds`, `noteText`, `boldIdMatches`, `nameInput`,
    `pinInput`, `legendText`, `guideText`, `importFile`, `reassignTo`, `reassignOther`,
    `reassignOtherWrap`, `reassignReason`, `reassignError`, `errorExport`.
  - Setup fields: `field-*`, `fixed-*`, `err-*`.
- **CSS would leak into a host page.** It includes page-wide rules (`*`, `body`,
  `body::before` tiled background, `button:focus-visible`, `button:disabled`, `@media print`),
  generic `:root` variables (`--bg`, `--ink`, `--accent`, …) and generic class names
  (`.sidebar`, `.toast`, `.modal-*`, `.card-panel`, `.btn-*`…). Namespace it, or isolate it.

## Hosting

- **SharePoint Online won't host the .html file directly.** It downloads .html files
  instead of displaying them, and the .aspx workaround depends on custom scripts, which
  Microsoft now turns off again automatically.
- **Short term:** host the file on its own origin, for example a separate subdomain like
  `audits.<company domain>`. Don't put it on the dashboard's own origin. Embed it with:

  ```html
  <iframe src="https://audits.example.com/flex-audit-tracker.html"
          sandbox="allow-scripts allow-same-origin allow-downloads allow-modals"></iframe>
  ```

  - **Why a separate origin:** an iframe on the same origin, or with `allow-scripts
    allow-same-origin` on the same origin, has full access to the dashboard page, its
    storage and any MSAL/Graph tokens. Anyone who could edit the hosted file could then
    take over the dashboard. On a separate origin, that sandbox pair is safe.
  - **What each permission is for:** `allow-same-origin` keeps localStorage working,
    `allow-downloads` is for Download Backup, and `allow-modals` is for printing.
- **Embedded storage is separate.** Because of browser storage partitioning, the embedded
  tracker has its own storage. It won't see data saved when the file was opened directly,
  so move existing data in with Download Backup → Restore.
- **Proper:** port it into the dashboard as a native module or component backed by the
  shared backend.
  - The logic is self-contained and ports cleanly: `computeAssignments`,
    `sanitizeState`, `parseBoldIds`, `computeBoldIdOccurrences`, `overdueWeeks`, the date
    helpers, and the setup rename/snapshot logic in `applySetup` / `validateSetupDraft`.
  - The rendering is HTML template strings and should be rebuilt in the dashboard's
    framework.

## Assets

The two images are embedded in the file:
- **I-9 reference:** WebP, 900×1164, about 149KB, stored in `const I9_IMAGE`.
  - It is the Form I-9 edition dated 01/20/25 (expires 05/31/2027) with YES annotations.
  - Replace it when USCIS issues a new form.
- **Background mascot tile:** an SVG with an embedded PNG, about 9KB, stored in
  `body::before`.
- If these move to external files, load the I-9 image only on the I-9's guide page.

## Compliance content owners should review

The How-To Guide and Legend text is editable in the app. Nicholas has confirmed the
policy decisions in it, including:
- the EAD 3-day removal rule;
- the SSN is required on the I-9, because YES uses E-Verify;
- paper checks are for emergencies only (HR confirmed);
- expired documents can't be accepted on an I-9.

The I-9 guide is based on `Form-I-9---Quick-Reference-Guide_ENGLISH.pdf` in this folder.

## Challenge question for your AI assistant

Before you start, paste the prompt below into the AI coding assistant on your machine,
opened on the **Flex Master Dashboard** codebase so it can look at the real code. Nicholas
would prefer the tracker **built into the dashboard**, but wants an honest check first. If
building it in would slow the dashboard down or make it fragile, he wants to know.

> I'm deciding whether to build the Flex Audit Tracker into the Flex Master Dashboard (this
> codebase) or keep it as its own separate app linked from the dashboard. The owner prefers
> building it in. Challenge that preference using the actual code in this repo, not general
> advice.
>
> About the tracker (`flex-audit-tracker.html` and `admin-handoff.md` are attached or in
> this folder):
> - A single vanilla HTML/JS file of about 390KB.
>   - About 210KB is two embedded base64 images: the I-9 reference image and a background
>     tile.
>   - Most of the rest is guide text. The logic is about 2,200 lines with no dependencies.
> - Data today is per-browser localStorage. It will move to the dashboard's shared backend
>   (see "Suggested backend tables" in `admin-handoff.md`).
> - Its CSS uses page-wide rules and generic class names, which would clash with the host
>   page.
> - It needs: weekly results, Repeat Flags, Overdue, Pass Rates, reassignments, print,
>   backup/restore, and a manager-only Edit Mode. Uncorrected fails must show in the
>   dashboard's existing Alerts section.
>
> Please answer:
> 1. **Performance:** if built in, how much would it add to the dashboard's initial load and
>    bundle? Can it be lazy-loaded as its own route or chunk, with the I-9 image loaded only
>    on its guide page, so the rest of the dashboard isn't slowed down?
> 2. **Fit:** how well does it fit this codebase's framework, routing, state management, auth
>    (Entra ID / MSAL) and data layer? Roughly how much has to be rewritten versus reused?
> 3. **Risk:** what could break in the dashboard (CSS collisions, global listeners, print
>    styles, shared auth tokens), and how would you isolate it?
> 4. **Alternatives:** compare (a) a native module in the dashboard, (b) a separate app on
>    its own subdomain, linked or embedded in a sandboxed iframe, and (c) anything better you
>    see. Cover load time, maintenance effort, security, and the Alerts integration.
> 5. **Recommendation:** pick one and explain why. If you recommend building it in, list
>    the conditions that keep the dashboard fast (lazy loading, image handling, CSS
>    scoping). If you recommend keeping it separate, say what would have to change for
>    building it in to become the better option.
