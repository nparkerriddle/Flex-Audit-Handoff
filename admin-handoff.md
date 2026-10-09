# Flex Audit Tracker — Admin Integration Notes

Prepared 2026-10-07 for integrating `flex-audit-tracker.html` into the Flex Master Dashboard.

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
- **Past Weeks:** the history of earlier weeks.
- **Print This Week:** a printable sheet with a manager signature line.
- **Audit Legend:** short summaries of each audit.
- **How-To Guides:** a step-by-step page for each audit.
- **Edit Setup:** a PIN-protected page where staff can change everything without a developer.

- One self-contained file, about 325KB, with no dependencies and no network requests.
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
- **Rotation rule:** for week N, `offset = (N-1) % audits.length`, and
  `rotatingPeople[i]` gets `audits[(i + offset) % audits.length]`.
- **Older saves are converted automatically** by `sanitizeState()` in the file:
  - they have no `config`,
  - results may be plain strings,
  - they use a single `boldId` string,
  - they use a `currentWeek` counter instead of dates.

  Reuse that function when importing into a backend.

## Suggested backend tables (SharePoint Lists or SQL)

| Table | Columns |
|---|---|
| `audit_results` | week_number, week_start_date, role, status, verified, note, recorded_by, recorded_at, corrected_by, corrected_at |
| `audit_result_bold_ids` | result_id, bold_id |
| `audit_week_assignments` | week_number, role, audit, is_fixed (the snapshots) |
| `audit_config` | single row/JSON with the `config` object |
| `audit_docs` | item_name, legend_summary, how_to_guide, image_key |

Supabase was ruled out; SharePoint Lists through Microsoft Graph was the tentative choice.

## Dashboard Alerts integration (required)

The Flex Master Dashboard's existing **Alerts** section must show every failed audit
that has not been corrected, and keep showing it until it is corrected:

- **Alert when:** `status === "Fail"` and `correctedAt` is empty (see `isUncorrectedFail`
  in the file). Include past weeks, not just the current one.
- **Alert text** should name the week (number + date range), the role, and the audit, e.g.
  "Week 3 (Oct 5 – Oct 11): Headcount Accuracy failed (A), not corrected yet."
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

## Embedding considerations

- **JavaScript:** wrapped in an IIFE with `"use strict"`, so it adds no global variables.
  It does add `document` listeners (click, keydown, input, focusout, change) and a
  `window` `beforeunload` listener (the unsaved-changes warning).
- **Fixed element IDs:**
  - Page layout: `sidebar`, `mainContent`, `toast`, `errorBar`, `errorText`, `modalRoot`,
    `modalOverlay`, `printArea`.
  - Pop-up and form fields: `noteBoldIds`, `noteText`, `boldIdMatches`, `nameInput`,
    `pinInput`, `legendText`, `guideText`, `importFile`.
  - Setup fields: `field-*`, `fixed-*`, `err-*`.
- **CSS would leak into a host page.** It includes page-wide rules (`*`, `body`,
  `body::before` tiled background, `button:focus-visible`, `button:disabled`, `@media print`),
  generic `:root` variables (`--bg`, `--ink`, `--accent`, …) and generic class names
  (`.sidebar`, `.toast`, `.modal-*`, `.card-panel`, `.btn-*`…). Namespace it, or isolate it.

## Hosting

- **SharePoint Online won't host the .html file directly.** It downloads .html files
  instead of displaying them, and the .aspx workaround depends on custom scripts, which
  Microsoft now turns off again automatically.
- **Short term:** host the file on any same-origin web server and embed it in the dashboard
  with an `<iframe>`. Don't use `sandbox`, or include `allow-scripts allow-same-origin`,
  otherwise browser storage is blocked.
- **Cross-origin iframes get separate storage.** Because of browser storage partitioning,
  a cross-origin iframe won't see data saved when the file is opened directly.
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
policy decisions in it, including the EAD 3-day removal rule. The I-9 guide is based on
`Form-I-9---Quick-Reference-Guide_ENGLISH.pdf` in this folder.
