# Site Ledger — Project Notes (for AI / developer handoff)

This document exists so that a human or an AI assistant with **no memory of
how this project was built** can understand it fully from this file alone,
without re-reading the original conversation. It explains not just *what*
the code does, but *why* it's shaped the way it is — several design
decisions here are not obvious from the code and exist because of real
mistakes made and corrected during development.

## What this project is

A single-file HTML web app for a civil engineer in India to track money and
materials across multiple client construction projects. It replaced an
earlier Excel/VBA-based version — VBA was abandoned because this sandbox
environment (Linux + LibreOffice, no real Excel) could not reliably produce
a working `vbaProject.bin`, verified by testing: injecting a macro and
exporting to `.xlsm` silently dropped it entirely. The web app avoids that
problem completely.

**Stack:** vanilla HTML/CSS/JavaScript (no framework, no build step),
Firebase Authentication (Google Sign-In) + Firestore (database), SheetJS
(`xlsx` library, via CDN) for Excel export. Everything lives in one file:
`Site_Ledger_App.html`.

## Data model

```js
state = {
  accounts: {
    labour:   [{ id, name }],   // e.g. Mason Labour, Centring, Concrete...
    material: [{ id, name }],   // e.g. Cement, Steel, M.Sand...
    supplier: [{ id, name }],   // e.g. Lucky Cement, Royal Electric...
    client:   [{ id, name }]    // e.g. Sampath, Ramesh...  (see naming below)
  },
  transactions: [{
    id, date, section,          // section: 'labour'|'material'|'supplier'|'client'|'engineer'
    accountId,                  // null for 'engineer' (it has no sub-accounts)
    subLedger,                  // only meaningful when section==='client': 'project' | 'client'
    type,                       // 'Credit' | 'Debit'
    amount, notes,
    linkGroupId,                // shared id across a primary entry + any auto-created twins
    auto                        // true if this record was auto-created by a link, not typed directly
  }],
  profile: { name, business, phone, address }  // used as the print letterhead
}
```

Stored as a single Firestore document per user: `siteLedgers/{uid}`.
Security rules restrict each document to its own owner (`request.auth.uid
== userId`). One `db.collection('siteLedgers').doc(uid).set(state)` call
per save — the whole state is overwritten each time (fine at this scale;
see "Known limitations" below).

## The client account naming convention (important, non-obvious)

Every client actually has **two separate ledgers**, but the engineer wanted
them to appear as two normal-looking dropdown entries rather than one
account with a ledger-switcher. The convention, exactly as requested:

- Plain name (e.g. **"Sampath"**) → `subLedger: 'project'` → **Engineer
  Selavu** — tracks project *cost* (what the engineer spent on this
  client's project). Mostly filled in automatically via linking, see below.
- Name + " 2" (e.g. **"Sampath 2"**) → `subLedger: 'client'` → **Client
  Selavu** — tracks direct money exchanged with the client (payments
  received, refunds).

This is implemented via `clientFlatName(clientId, subLedger)` and
`flattenedClientList()`, which generates both dropdown options per client
account. There is no separate UI concept of "the client" — everywhere a
client is selectable (sidebar, Daily Entry dropdown), both flattened names
appear as independent-looking options. Adding a new client via the sidebar's
"+" input creates one account but both flattened names become available
immediately (no extra step).

## The linking system ("double-entry", engineer's term)

The engineer asked for something they called double-entry bookkeeping. What
was actually built, after several rounds of correction, is a simpler
**linked-entry mirroring system**, not formal accounting double-entry:

1. Any Labour, Material, or Supplier entry **always** auto-creates a second
   transaction in the Engineer account, same date/type/amount (`auto:
   true`) — because that money physically came out of the engineer's own
   pocket. This is unconditional, not optional.
2. On the Daily Entry page, there is **one client dropdown at the top of
   the whole page** (not one per section). If a client is selected there,
   *any* Labour/Material/Supplier blocks filled in during that same save
   automatically also mirror into that client's **project** ledger
   (`subLedger: 'project'`), each keeping its own amount — they are not
   forced to match each other's amounts.
   - Earlier version of this feature put a separate "link to client"
     dropdown inside each of the Labour/Material/Supplier blocks
     individually. The engineer explicitly corrected this: **there should
     be one client selection for the whole page, not one per section.**
     Don't reintroduce per-block client pickers.
3. Material entries additionally have their own **optional** "link to a
   supplier" dropdown (separate axis from the client link — e.g. "this
   cement purchase is owed to Lucky Cement" — independent of which client
   it's for).
4. All records created together (the original typed entry + any auto
   mirrors) share one `linkGroupId`. Deleting one asks whether to delete
   the whole linked group or just that one record
   (`deleteTransaction(id)`).

### The double-counting bug (already fixed — don't reintroduce it)

Early on, `overallTotals()` (used for the header stats and the Summary
grand-total row) summed **every** transaction record, including the
auto-created mirrors. A single ₹12,000 material purchase linked to both a
client and a supplier created 4 total records (primary + Engineer mirror +
client mirror + supplier mirror), and the overall total showed ₹48,000
instead of ₹12,000. The fix: `overallTotals()` only sums records where
`auto !== true`. Every individual account's own page still sums *all* its
own records (correctly, since that page needs its full picture) — only the
whole-business total excludes auto records. The same principle applies to
the "All Entries" master log (`renderAllEntries`) and its CSV/print
exports: they show **one row per real transaction** (`!t.auto` only), with
an "Also recorded in" / "linked to" column listing every other ledger it
touched (via `siblingLabelsFor(tx)`), rather than showing 3-4 rows for one
real event.

## Date filtering

Global `dateFilter` (string, `'YYYY-MM-DD'` or `'all'`), defaults to
`todayStr()`. Every account page and the All Entries page shows a `<select>`
of every date that has entries for that specific ledger, plus an "All
dates" option. `goTo(view)` resets `dateFilter` to today whenever navigating
to a *different* page (so each page you open defaults to "today" fresh);
`setDateFilter(value)` just re-renders the current page with the new
filter, without navigating. Running-balance totals are always computed
against the *full* unfiltered transaction list first, then rows are hidden
by date — so the running balance shown is always the true cumulative
figure, not reset per filtered day.

## View routing

`activeView` is a plain string: `'entry'`, `'summary'`, `'all'`, `'profile'`,
or `"${section}:${accountId}:${subLedger}"` for an account page (e.g.
`"client:a_xyz:project"`, `"labour:a_abc:"`). `parseView()` decodes it,
`render()` dispatches to the right render function. Sidebar sections
(Labour/Material/Supplier/Client) use native `<details>`/`<summary>` for
expand/collapse — **not custom click handlers** — because an earlier
JS-click-based implementation couldn't be verified working in this sandbox
(no headless browser available to test against) and the engineer reported
it not working. Native `<details>` needs no JS to function correctly; the
`ontoggle="expandedSections['section']=this.open;"` inline handler only
exists to remember open/closed state across re-renders, not to drive the
actual show/hide behavior.

## Charts

Both chart types are hand-built inline SVG (no charting library), so they
render identically on screen and in print/PDF with zero extra work:

- `creditDebitChart(credit, debit, size, onDark)` — 2-slice donut using the
  stroke-dasharray-on-a-circle technique. `onDark` swaps text colors for use
  on the dark "whole business" banner on the Summary page.
- `multiSliceChart(dataObject, size)` — N-slice donut for the "where the
  money went" category breakdown. Only shown on the Engineer page and each
  client's project (Engineer Selavu) page, since those are the only ledgers
  where "which category did this come from" is a meaningful question.
  `categoryBreakdownFor(section, accountId, subLedger)` builds the
  breakdown by tracing each `auto:true` record back to its primary sibling
  via `linkGroupId` (`primarySiblingLabelFor`).

## Printing

`@media print` hides the header, sidebar, and anything marked `.no-print`,
and shows `#printArea` (populated just before `window.print()` is called by
`printReport()` / `printSummaryReport()` / `printAccountReport()`).

**A real bug was found and fixed here**: `.shell` (the sidebar+content flex
wrapper) had `min-height: calc(100vh - 68px)` which is *not* overridden in
print media — even with its children hidden, the empty container still
reserved a full page of blank space, pushing all real content onto page 2.
Fixed by adding `.shell` to the print `display:none` selector list. If a
future change reintroduces a blank first page in print/PDF, check for
exactly this class of bug: some container with a `min-height`/`height` rule
that isn't neutralized under `@media print`.

## Firebase / hosting constraints

- Google Sign-In **cannot work on a `file://` page** — it requires a real
  hosted origin. The app is designed to be hosted on GitHub Pages (free,
  no CLI needed — just upload the file and enable Pages in repo settings).
- The Firebase config object near the top of the script is a placeholder
  (`apiKey: "YOUR_API_KEY"` etc.) — `FIREBASE_CONFIGURED` checks whether
  it's been filled in and shows a setup notice if not.
- **Every time this file is rebuilt from scratch** (as opposed to edited
  in place), the real Firebase config is lost and must be manually
  re-pasted before re-hosting. This has happened repeatedly during
  development because large restructuring was done by writing a fresh
  file rather than patching the old one. If making a large structural
  change, prefer `str_replace` edits over full rewrites where possible, to
  avoid this — or at minimum, remind whoever is deploying it to re-add
  their config.
- Firestore free tier (Spark plan): 1 GB storage, 50k reads/day, 20k
  writes/day — effectively unlimited for one engineer's personal use (each
  daily-entry save is a handful of writes at most).

## Known limitations / deliberate non-goals

- Not formal double-entry accounting (no chart-of-accounts, no trial
  balance, no accrual/cash distinction) — deliberately simplified per the
  engineer's actual needs, not an oversight.
- No multi-user collaboration — one Firestore document per Google account;
  two people can't share/edit the same ledger concurrently.
- No undo beyond the linked-group delete confirmation.
- Whole-document overwrite on every save (not incremental) — fine at this
  data scale (a small business's manual ledger entries), would need
  restructuring (e.g. one Firestore document per transaction) if this ever
  needed to scale to a much larger transaction volume or multiple
  simultaneous editors.

## If extending this project

- To add a new top-level section (beyond Labour/Material/Supplier/Client/
  Engineer): follow the Labour/Material/Supplier pattern — add to
  `defaultAccounts()`, `SECTION_LABELS`, the sidebar loop, and a Daily Entry
  block. Decide deliberately whether it should auto-mirror to Engineer
  (Labour/Material/Supplier do; Client and Engineer itself don't).
- To change what "linking" means for a new case: follow the existing
  pattern of shared `linkGroupId` + `auto: true` on generated twins, and
  remember to exclude `auto` records from any "total across everything"
  calculation (`overallTotals`, `sectionGrandTotal`) to avoid reintroducing
  the double-counting bug described above.
- Before shipping any change, there's no live browser available in most
  sandboxes to test in — verify logic changes by extracting the `<script>`
  contents and running realistic scenarios directly in Node (as was done
  throughout this project's development) rather than assuming correctness.
