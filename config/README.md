# `config/events.json`

This is the live event/day/facility registry for `sage-tools-api`'s sync
feature, which `src/registry/SyncConfigStore.mjs` fetches and caches. It replaced
the old hardcoded `EVENTS` const in that repo's sync code — editing this file (and committing it, on `main`)
is now how you add an event, add a day, or fix a wrong sheet ID. **No
redeploy of `sage-tools-api` is needed** for a change here to take effect;
every running instance re-checks this file roughly once a minute (see
`SYNC_CONFIG_TTL_MS` in that repo's `.env.example`).

For the day-to-day instantiation checklist (choosing a template, filling in
tokens, etc.), see `_templates/CLAUDE.md` in `sage-match-control.github.io`.
This file documents the schema itself.

## Shape

```json
{
  "version": 1,
  "defaults": {
    "matchesSheetName": "CSV",
    "standingsSheetName": "STANDINGSCSV"
  },
  "events": {
    "<event-key>": {
      "type": "dual-meet",
      "archived": false,
      "title": "PNF × BUP Dual Meet",
      "attendance": "desks",
      "scoreEntry": "links",
      "days": {
        "<day-key>": {
          "label": "Day 1 · Aug 15",
          "date": "2026-08-15",
          "isLive": "auto",
          "facilities": [
            { "name": "Main", "sheetId": "1hbjqbH3H1..." }
          ]
        }
      },
      "display": {
        "divisions": { "LI": "Low Intermediate" },
        "events": { "MD": "Men's Doubles" },
        "clubs": { "PNF": "Pickle & Friends Community" }
      }
    }
  }
}
```

- `version` — must be `1`. `sage-tools-api` checks this and refuses the file
  (falling back to its last-known-good copy) if it doesn't match, rather than
  silently mis-parsing a future shape change.
- `livePush` — optional boolean, the operator's switch for live push (see
  `sage-tools-api`'s live push). `false` makes every sync, and Live/Hide,
  publish to GitHub alone; absent or `true` leaves live push to Cloud Run's
  environment (`LIVE_PUSH_URL`). Normally written by Control Center's
  **Sync method** switch rather than by hand, and takes effect within
  `SYNC_CONFIG_TTL_MS`. A value that isn't a boolean makes the whole file
  invalid.
- `defaults` — optional. Anything it omits falls back to an in-code default
  in `sage-tools-api` (the values shown above are those defaults). Only set
  this if most days need the same non-default tab names.
- `events.<event-key>` — one entry per event. The key must match this
  event's folder name under `events/` in `sage-match-control.github.io` **and**
  its folder name in this repo (`<event-key>/data/...`).
  - `type` — `"dual-meet"`, `"standard"` or `"team"`. Read only by Control Center
    (`tools/control-center.html`) — `sage-tools-api` never looks at
    it. Picks both the standings layout and how team codes split
    (`<CLUB>_<DIV><EVT>_<REST>` vs `<DIV><EVT>_<REST>`). Not validated here
    (the console shows its own visible error for a missing/unrecognized
    value rather than guessing) — see
    `sage-docs/docs/specs/.../match-control-console-spec.md`.
  - A `"team"` event takes its team names from `STANDINGSCSV` and needs no
    `display` block.
  - `attendance` — optional, `"console"` or `"desks"`. Turns on staff attendance for the event:
    `"console"` lets operators mark people in Control Center, `"desks"` also
    allows desk links (a day needs a `date` for that). Anything else fails
    validation in `sage-tools-api`. Absent means no attendance.
  - `scoreEntry` — optional, `"console"` or `"links"`. Turns on entering a match's
    score from Control Center and the API: `"console"` lets signed-in operators
    do it in Match Finder, `"links"` also allows scorer links (issued for a day until
    06:00 Manila the morning after its `date`; a day without a `date` is never
    over). Absent
    means off; anything else fails validation in `sage-tools-api`. Mission
    Control's **Scorer links** switch moves an event between `"links"` and
    `"console"` by editing this value; it never turns score entry on or off, so
    adding the setting is a commit here. Every facility workbook of the event
    must be shared with the API's service account as **Editor**, or saving a
    score fails with a message naming the account.
  - `archived` — optional, console-only. `true` hides the event from the
    console's event picker entirely. Omit or set `false` for a live event.
    **An event stays in this file for as long as any page built on the site
    engine shows it** (its Tournament Hub, schedule board, scorer page or
    desk page, which read it too): removing a finished event's entry blanks
    those pages. `archived` is how to hide a finished event from the console.
  - `title` — optional. Shown as the console's masthead label once this event
    is selected.
  - `display` — optional: `{ divisions, events, clubs }`, each a plain
    `code → label` map (e.g. `"LI": "Low Intermediate"`). Control Center and the
    event pages' Tournament Hubs read it. Without it they still work, showing
    raw codes. Order comes from each map's own key order. `clubs` has no logo
    field — a dual meet's logos are in its Hub page's `CLUB_LOGOS`.
- `events.<event-key>.days.<day-key>` — one entry per tournament day.
  - `label` — required, shown in error messages and diagnostics.
  - `date` — optional (`YYYY-MM-DD`), console-only. Lets the console order
    days chronologically and default to the current one.
  - `isLive` — optional, one of `true`, `false`, or the literal string
    `"auto"`. Any other value fails validation the same as a structural
    error (whole file rejected, last-known-good config kept serving). Absent
    is treated the same as `"auto"`. This is Control Center's
    go-live override (`POST /sync/:day/live`, secret-gated): every sync
    stamps the day's current value into its published snapshot alongside
    `label`, and the override endpoint writes here first, then immediately
    republishes that snapshot — so a later real sync can never silently
    revert an operator's override, since it re-reads this file every time.
  - `facilities` — required array of `{ name, sheetId }`. A facility with an
    empty/missing `sheetId` is treated as "not set up yet" and skipped rather
    than fetched — useful for adding a day's entry before its spreadsheet
    exists.
  - Optionally override `matchesSheetName` and/or `standingsSheetName` for
    just this day, if its spreadsheet's tabs are literally named something
    other than `CSV`/`STANDINGSCSV`. A `"team"` event's day can likewise set
    `rosterSheetName` (default `Teams`), the tab whose roster is published as
    `rosterCsv`; it must be a non-empty string.

    > **There is deliberately no GID equivalent of these fields.** Both
    > `sage-tools-api` fetch paths address tabs by name, never by numeric
    > GID. A GID is assigned per-workbook, so a value correct for one
    > event's spreadsheet can point at a completely different tab in
    > another — this actually happened: `pnf-x-bup-dual-meet`'s workbook was
    > duplicated from the archived `ppa-x-club-2600-dual-meet` sheet and
    > inherited its old tabs' GIDs, so the shared default `standingsGid`
    > (back when that field existed) silently resolved to a leftover tab
    > from the old event instead of the real `STANDINGSCSV` tab — with no
    > error, just wrong data. A tab *name* doesn't carry that risk across
    > workbooks.
- A `"_comment"` string key is allowed anywhere in the tree (on the root
  object, an event, or a day) for notes that would otherwise have no home in
  JSON. It's ignored by validation and by `sage-tools-api`.

## Validation rules

Enforced by `sage-tools-api`'s `SyncConfigStore` on every load. A file that
fails any of these is **rejected wholesale** — the service keeps serving
whatever it last loaded successfully (or its bundled fallback seed, if
nothing has ever loaded successfully) rather than partially applying a
broken commit or crashing:

- `version` must equal the supported version.
- Every event key and every day key must match `^[a-z0-9][a-z0-9-]*$` — this
  is a hard requirement, not a style preference: both keys become path
  segments/filenames in this repo, so an invalid key is refused outright
  rather than sanitized.
- Day keys must be **globally unique across every event** in the file — two
  events can never declare the same day key, since that key is also the
  route (`POST /sync/:day` on Cloud Run) and the two would otherwise race to
  publish into each other's data.
- Each day needs a non-empty `label` and a `facilities` array (it can be
  empty, e.g. before spreadsheets exist for it — but the key must be
  present).
- A day's `isLive`, if present, must be `true`, `false`, or the literal
  string `"auto"` — anything else fails validation the same as a structural
  error.
- Facility names must be unique within a day.
- The file needs at least one event, and each event needs at least one day.

If you're unsure whether an edit is valid before committing it, ask
whoever's driving `sage-tools-api` to check `GET /sync/config` (with the
`X-Sync-Secret` header) after your commit — it reports which config
revision is actually live, including whether the service fell back to its
bundled seed because your commit failed validation.

## Sheet IDs are effectively public

This repo is public (GitHub Pages requires it on a free plan), so this file
publishes the list of every facility spreadsheet's ID. That's expected —
sheet IDs aren't secrets, the sheets are already shared "anyone with the
link can view" by design, and the published match/standings snapshots
already contain everything on their SCHEDULE/COURT CONTROL tabs. But it does
make the *set* of spreadsheets enumerable. Two things follow from that:

- Only put IDs of spreadsheets that are already link-shareable and meant to
  be public.
- Don't keep private organizer notes in an extra tab of a spreadsheet that's
  registered here — anyone who finds the sheet ID can open the whole
  spreadsheet, not just the CSV/STANDINGSCSV tabs this file names.
