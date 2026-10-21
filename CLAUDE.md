# Flex Audit App

## What it is

A weekly compliance-audit tracker for the Flex Onsite Team (I-9s, Workday Core HR data, headcount
accuracy, terminations, absenteeism, etc). Replaces a manually-copied spreadsheet that
assigned the same audit to the same person every week.

## Who it's for

The Flex Onsite Team (YES staffing). Non-technical end users — built for zero ambiguity,
large tappable chips, no jargon.

## Tech stack & files

- `flex-audit-tracker.html` — the whole app. Single-file vanilla HTML/CSS/JS, no build
  step, no dependencies, no network requests. Edit it directly. Two lines hold large base64
  images (the CSS `body::before` background tile and `const I9_IMAGE`) — never read those
  lines whole; use `cut -c1-300`.
- `admin-handoff.md` — integration notes for the admin/dev folding this into the Flex
  Master Dashboard (state schema, globals, storage, PIN, recommended path). Keep it in sync
  with the data model below.
- `Form-I-9---Quick-Reference-Guide_ENGLISH.pdf` — source material for the I-9 guide; not
  used by the app.
- Script is wrapped in an IIFE (`"use strict"`) — no globals. No inline event handlers:
  every control uses `data-action="..."`, dispatched by one delegated click listener to the
  `actions` map. Keyboard focus survives re-renders via `data-focus-key` attributes.

## Storage

`window.localStorage` (per-browser/per-device — see Deferred Decisions):
- `audit-tracker-state` — the app state (schema below).
- `audit-tracker-user-name` — "Recording as" name for this browser.
- `audit-tracker-last-backup` — when a backup was last downloaded here.
- Safety copies, never read by the app: `audit-tracker-state-before-restore-<ts>`,
  `-damaged-copy-<ts>`, `-unreadable-copy-<ts>`.

## Visual direction

"Console" style — cool white/slate palette, cobalt accent (`#2454d9`), card-based rows
with avatar-initial circles and pill chips. Chosen to match the Flex Master Dashboard's
look. System fonts only. Colors were darkened 2026-10-07 to pass WCAG AA (muted `#555c67`,
pass `#157a3f`, fail `#b42323`, control borders `#8a93a3`) — don't lighten them.
Minimum 14px text; chips 14px, audit names 15px.

## Terminology (user-facing)

- **Team role** — A, C, B, D, Day Admin, Night Admin (never real names).
- **Rotating audit** — the 6 weekly rotation slots.
- **Audit Legend** — short "quick summaries" at the bottom of the tracker (`auditDetails`).
- **How-To Guide** — the full step-by-step page per item in the left menu (`sopDetails`).
  Code still calls these SOP; the UI never says "SOP" or "walkthrough".
- "Failed" = a Fail result. "Repeat Flags" = a Bold ID seen more than once. Don't mix them.
- **Avionte Bold** for the system (not "Avionte" or "Bold" alone); "Bold ID" for the number.
- **Flex Master Dashboard** — same thing as "Flex Dashboard"; always use the full name.

## Data model

```
state = {
  version: 3,
  results:       { [week]: { [person]: { status: "Pass"|"Fail"|null, verified: bool,
                                         note: string, boldIds: string[], by: string, at: ISO,
                                         correctedBy: string, correctedAt: ISO|"" } } },
  weekSnapshots: { [week]: [{ person, audit, fixed }] },
  reassignments: { [week]: { [person]: { to: string, reason: string, by: string, at: ISO } } },
  auditDetails:  { [itemName]: string },   // Legend quick summaries
  sopDetails:    { [itemName]: string },   // How-To Guides
  config: { title, subtitle, sidebarTitle, passNote, weekOneStart: "YYYY-MM-DD",
            rotatingPeople[], displayOrder[], audits[], fixedPerson, fixedAudit,
            legendItems: [{ name, image }] }
}
```
- All maps are null-prototype (`dict()`) and read with `own()` so typed names like
  "constructor" or "__proto__" are plain data. Keep that pattern for any new map.
- `sanitizeState()` is the single entry point for saved data AND imported backups. It drops
  bad weeks/entries one at a time (keeping a raw copy) instead of resetting everything.
- An entry is deleted when status, verified, note and boldIds are all empty
  (`isEmptyResult`); a week with no entries is deleted.
- `boldIds` are normalized by `parseBoldIds`: split on commas/spaces/semicolons,
  uppercased, de-duplicated.
- `by`/`at` are stamped on every change (`by` = this browser's "Recording as" name, may
  be blank).
- `correctedAt`/`correctedBy` only exist meaningfully on a Fail: `sanitizeResult` drops
  them otherwise and `setResult` clears them when the status leaves Fail. `""` = not
  corrected yet.

## Business rules

- **Nothing user-facing is hardcoded.** The constants (`ROTATING_PEOPLE`, `DISPLAY_ORDER`,
  `DEFAULT_AUDITS`, `FIXED_PERSON`/`FIXED_AUDIT`, `LEGEND_ITEMS`) are only starting
  defaults for a brand-new tracker or pre-setup saved data. Changing them does nothing to
  an existing browser's data — the live values are in `state.config`.
- **Roster is role codes, not real names**, so turnover never needs a change.
- **Rotation**: for week N, `offset = (N-1) % audits.length`; `rotatingPeople[i]` gets
  `audits[(i + offset) % audits.length]` (`computeAssignments`). `assignmentsForWeek`
  returns `weekSnapshots[N]` instead when it exists. Never reorder existing
  `rotatingPeople` entries — index drives the rotation. The Setup page only reorders
  `displayOrder`; new roles are appended to `rotatingPeople`.
- **Fixed assignment**: `fixedPerson` (default "Everyone") does `fixedAudit` (default
  "I-9's & W4 & DD") every week, outside the rotation.
- **Weeks are real dates**: Week 1 starts on `config.weekOneStart`; each week is 7 days.
  The tracker opens on the week containing today; future weeks can't be opened or
  recorded. Viewed week is per-session UI state, not saved. The Week 1 date can't be in
  the future, before 2020, or late enough that an already-recorded week would become a
  future week. Older saved data (only a `currentWeek` counter) is anchored so the latest
  week in use = this week (Monday start).
- **History protection**: on every setup save, every week with results is frozen into
  `weekSnapshots`. Renames are corrections and are applied to results keys, snapshots,
  `auditDetails` and `sopDetails`. A role may not be renamed onto any name that already
  has saved results/snapshots unless that name is being freed by another rename in the
  same save (swaps are fine) — this prevents silently merging two roles' history.
- **Edit Mode**: PIN `9377` (`EDIT_PIN`) — **never change the PIN; Nicholas wants it 9377
  permanently (2026-10-08)**. In-memory only (reload re-locks). Client-side
  deterrent only — anyone can read the source. Without the PIN: Pass/Fail/Verified, notes,
  Bold IDs, name, print. With the PIN: Legend Edit buttons, Edit How-To Guide, Edit Setup
  (roles, audits, fixed audit, legend/guide items, calendar start date, page text,
  Backup & Restore).
- **Legacy migrations** (`NAME_MIGRATIONS` real names → codes, `LEGEND_NAME_MIGRATIONS`)
  run only on data with no `config`, so a role later renamed to e.g. "Alma" is untouched.
  `LEGEND_CONTENT_REFRESH` / `SOP_CONTENT_REFRESH` run every load but only replace text
  that still exactly matches an old default (W4 exempt update; 2026-10-08 DD no-paper-check
  and Core HR report wording).
- Default Legend/Guide text is seeded only once, so text deliberately cleared stays clear.
- **I-9 image**: `I9_IMAGE` (WebP, Form I-9 edition 01/20/25, expires 05/31/2027 —
  replace when USCIS updates the form). A legend item points to it via
  `image: "<default item name>"`, which survives renames.
- **E-Verify audit retired (2026-10-08)**: replaced by **Audit Core HR Report** (legend/guide
  item) and the rotation slot "Bold Status & EAD's & Core HR Report". It checks the Workday
  Core HR - Contingent Worker Details report for Light Industrial code, work area, primary
  org, agency, work shift, wage, pay frequency, supervisor, markup and Contingent Kronos ID
  area. Supervisor/shift/department are checked against the Req Tracker / Flex Q (Flex Q is
  imported from the Req Tracker); wage, pay frequency, markup and the rest are standard
  account values. Staff correct errors directly in Workday. E-Verify is intentionally no
  longer part of the weekly audits (Nicholas, 2026-10-08). `migrateEverifyToCoreHr` runs once on any save/backup with `version < 3`: it freezes
  every recorded week into `weekSnapshots` first (so history still says Everify), then
  renames the slot (only if it still has the exact old name) and swaps the legend item in
  place. Don't rename it via Edit Setup instead — setup renames rewrite history snapshots.
- **Audit Prep List "no answer" step** (still update contact count + Last Contacted date when
  unreachable) — confirmed by Nicholas 2026-10-09, keep it.
- **EAD workflow** (2026-10-09): staff check the ID on each associate's Avionte Certificates
  tab (not the I-9), upload the new ID there and edit the certificate. The "Complete
  Supplement B" step stays in the guide — Nicholas confirmed (federal reverification
  requirement). The guide also covers the Expiring Certificates dashboard widget.
- **File locations vary by site** (2026-10-21): Flex/Axiom may use Box, SharePoint, OneDrive,
  or the extranet in Teams depending on the site and its current process — staff use whichever
  is current. Guides that open shared files (New Hire, Terminations, Audit Prep List,
  Absenteeism/Payroll Tracker) carry this note; don't hardcode "Box" as the only location.
- **Fully termed** (Nicholas, 2026-10-21): assignment End Date = last day worked, a Logged
  Activity with type Termination (picked from the list, then notes typed), and status not On
  Assignment. Terminations guide step 4 fixes whatever is missing.
- **I-9 documents** (Nicholas, 2026-10-21): an ID expired at the time the I-9 was signed
  can't be accepted. A List A document that expires later must be replaced if the associate
  isn't a U.S. citizen (EAD guide; its permanent-resident exception still stands — Nicholas confirmed 2026-10-21). SSN is
  required on the I-9 because YES uses E-Verify for new hires (E-Verify is only gone from the
  weekly audits). Box 4 "N/A" is valid only when the work permit has no end date.
- **W4 exempt**: last year's exempt W4 stays valid Jan 1 – Feb 15; after that a new one is needed.
- **DD**: paper checks are emergency only (HR confirmed direct deposit/Rapid Card policy).
- The "Pass = 80% or better" note vs the 75% Pass Rates cutoff stays as is (Nicholas, 2026-10-21).
- **EAD policy**: the 3-day-after-expiration removal rule in the EAD summary and guide is
  Nicholas's decision (confirmed 2026-10-07) — keep it as written.
- **Repeat Flags**: any Bold ID recorded 2+ times across all weeks/rows shows in a red
  panel with jump links. Phase two is built: the note window warns live when a typed Bold
  ID was recorded in another week/row.
- **Overdue Audits**: past weeks (from the first week with any data up to last week) with
  rows lacking Pass/Fail show in an amber panel, newest first, max 8 listed.
- **Failed audit correction**: a Fail row gets a "Mark Corrected" chip that records who/when
  it was corrected. Un-marking it, or changing the result away from Fail, asks for
  confirmation first (`confirmRemoveCorrection`) since it erases the correction record. Shown on the row, the Failed stat ("N not corrected
  yet"), Past Weeks and print. In-app there is deliberately **no** uncorrected-fail alert
  panel: once integrated, the Flex Master Dashboard's existing Alerts section shows every
  Fail with no `correctedAt` until it is corrected (`isUncorrectedFail`).
- **Pass Rates by Audit**: table below the weekly rows covering the last 8 weeks
  (`PASS_RATE_WEEKS`) up to this week, grouped by audit name, lowest pass rate first,
  rates under 75% in red (cutoff confirmed by Nicholas 2026-10-09), plus a Not Corrected count. Hidden when no Pass/Fail exists.
- **Past Weeks** shows the 4 most recent weeks with data; "Show All" expands into
  collapsible calendar-year sections (year of the week's start date; newest year open).
  Weeks never run out — numbering keeps counting (cap `MAX_WEEK` 5200); nothing is archived
  or deleted.
- **Reassign** (2026-10-21, Edit Mode only): a manager can hand any rotating row (not the
  fixed "Everyone" row) for the viewed week — this week or a past week — to another team role
  or a typed name, with an optional reason ("Out sick"). Stored in `reassignments`; the
  result stays keyed by the original role, so rotation, history, Pass Rates and Alerts are
  unchanged, and it affects that one week only. Shown on the row ("Covered by D · Out sick"),
  print, Past Weeks, Overdue and the note window. Undo Reassign removes it. Setup renames
  carry reassignment keys and `to` values; weeks with reassignments are frozen into
  `weekSnapshots` on setup save.
- **Print This Week** builds `#printArea` (table + manager signature line) and calls
  `window.print()`; a `beforeprint` listener also builds it so Ctrl+P / the browser menu
  print the week being viewed. Print CSS hides everything else.
- **Backup & Restore** (Edit Setup): Download Backup writes
  `{ app: "flex-audit-tracker", version: 3, exportedAt, data: state }`. Restore runs the
  file through `sanitizeState`, confirms, and saves a safety copy first (only the newest
  `-before-restore-` copy is kept).
- Errors go to the persistent red error bar (`showError`, role=alert, Dismiss button);
  successes go to the 4-second toast. Any action that throws shows the error bar. If the
  app can't render at all, a fallback screen offers "Download a Copy of My Data".
- `saveBlocked`: if damaged/unreadable saved data can't be safety-copied, every save is
  refused (so the only original isn't overwritten) and the error bar shows "Download a Copy
  of My Data". A successful restore clears it. Imports over 5MB (`MAX_IMPORT_BYTES`) and
  week numbers over `MAX_WEEK` (5200) are rejected; an invalid saved Week 1 date is
  re-anchored like legacy data.
- Weekly rows put the chips under the text at ≤1100px wide (they squashed the text before).

## Deferred decisions — do not build without re-confirming with Nicholas

- **Shared/live multi-user backend.** Nicholas wants everyone to access one shared live
  instance via the company SharePoint. `localStorage` can't do this (each browser/device
  has its own private copy). Options discussed: Supabase (rejected — "I don't want to use
  supabase") and SharePoint Lists via Microsoft Graph API + Azure AD app registration
  (tentatively preferred, similar pattern to the Built Bar Dashboard). Being handed to
  admin for integration into the Flex Master Dashboard — see `admin-handoff.md`. Note:
  SharePoint Online won't host the .html file directly (downloads it instead).
- **Real access control** on top of the client-side PIN — needs the backend. The PIN itself
  stays 9377.

## Audit history

- 2026-10-07: full audit (logic, UI/accessibility, handoff readiness, compliance text).
  All findings fixed, and added: real-date weeks, backup/restore, multiple Bold IDs,
  who/when stamps, overdue panel, live repeat-ID warning, print view, W4 exempt check.
  Images recompressed (file 553KB → 325KB). An independent regression review afterwards
  found 4 more issues, all fixed. Verified with a 70-check Playwright suite.
- 2026-10-08: added failed-audit correction tracking and Pass Rates table; DD guide says
  paper checks are not an option; Headcount/New Hire name the Workday report "Core HR -
  Contingent Worker Details". 70-check suite still passes plus 32 new checks.
- 2026-10-08: E-Verify audit replaced by Audit Core HR Report (state version 3 migration).
  Verified with a 22-check migration suite (fresh, v2, pre-setup legacy, re-load, v2 backup
  import) plus the existing suites.
- 2026-10-09: guide refinements — EAD rewritten around the Avionte Certificates tab plus
  the Expiring Certificates widget; Audit Prep List adds the Mini Orientation check (add
  them to the Mini Orientation tab in Flex Q); Headcount Accuracy requires the Flex
  Dashboard to match the Core HR report. Old default text is in the *_CONTENT_REFRESH
  tables so unedited browsers update. All suites pass.
- 2026-10-21: file-location note (Box/SharePoint/OneDrive/extranet in Teams) on guides that
  open shared files; Terminations "fully termed" definition + fix-it step; pre-handoff sweep
  (security, logic, content, browser regression) — fixed narrow-window row layout, save
  overwrite after a failed safety copy, uncounted damaged entries, invalid Week 1 anchor,
  import size/week caps, Corrected chip label; I-9/W4/EAD/DD compliance wording and
  naming standardized; admin-handoff.md updated (separate-origin sandboxed iframe, server-
  side who/when stamps, MSAL PKCE, content refresh, v3 migration).
- 2026-10-21: added Reassign (Edit Mode) and year-grouped Past Weeks. 32-check browser
  suite for the feature plus the existing 32 + 17 checks all pass.

## Note on a rejected earlier version

An earlier prototype (built separately via claude.ai's Artifacts interface) used real
names instead of role codes and `window.storage` for persistence instead of
`localStorage`. `window.storage` only exists inside Claude.ai's Artifacts runtime — it
would not work as a standalone file. That version was not used.

## Status

Feature-complete for handoff to admin (internal YES tool). Pending: admin integration
into the Flex Master Dashboard with a shared backend.
