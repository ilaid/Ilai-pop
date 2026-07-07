# CLAUDE.md

Guidance for AI assistants (Claude Code and others) working in this repository.

## What this is

A single-page trade journal web app for futures traders (ES/NQ contracts). It tracks
individual trades, P&L, R:R, setup tags, economic-event context, and trading psychology
(emotions, mistakes, plan adherence), plus a calendar/dashboard view and CSV export.

## Repository layout

This repo is **one file**:

```
index.html   — the entire application (HTML + inline CSS + inline vanilla JS, ~830 lines)
```

There is no build step, no package.json, no bundler, no test suite, and no CI config.
"Development" means editing `index.html` directly and opening it in a browser (or
serving it statically). Do not introduce a build toolchain, framework, or dependency
manager unless the user explicitly asks for one — that would be a major, unsolicited
architecture change for what is intentionally a zero-build single file.

## Stack

- **Frontend**: vanilla JS, no framework. UI is built by string-templating HTML into
  `innerHTML` (no virtual DOM, no JSX). Fonts via Google Fonts (`Azeret Mono`, `Syne`),
  loaded over the network.
- **Backend**: [Supabase](https://supabase.com) (loaded from CDN as
  `@supabase/supabase-js@2`), used for both:
  - **Auth**: `sb.auth.signInWithPassword` / `sb.auth.signUp`. There's no real "email"
    concept in the UI — usernames are converted to a fake email via
    `getAuthEmail()` (`${username}@tradejournal.app`) because Supabase auth requires an
    email-shaped identifier.
  - **Data storage**: a single table, `journal_data`, with one row per user
    (`user_id`, `trades` (jsonb array), `tags` (jsonb array), `day_notes` (jsonb map),
    `updated_at`). The whole app state is loaded/saved as one JSON blob via
    `loadData()` / `saveData()` — there is no per-trade CRUD in the DB, it's whole-state
    upsert on every change (debounced ~1.2s via `scheduleSave()`).
- **Credentials**: the Supabase URL and anon/public key are hardcoded at the top of the
  script (`SUPA_URL`, `SUPA_KEY`). This is normal for Supabase's anon key (it's designed
  to be public; row-level security on the `journal_data` table is what should enforce
  per-user isolation) — but don't add any *service-role* key or other secret to this
  file.

## Code organization (within index.html)

The script is organized into sections marked by comment banners (search for these to
navigate):

- Supabase init / Constants (`CT` contract specs, `EMO` emotions, `MIS` mistakes,
  `ITAGS` default setup tags, `EVENTS` economic events)
- State (module-level `let trades`, `tags`, `dayNotes`, `currentUser`, `curTab`, etc.
  — no framework state management, just globals mutated in place then re-rendered)
- Helpers (P&L / R:R math, date formatting)
- Supabase DB (`loadData`, `saveData`, `scheduleSave`)
- Auth (login/register screens, `doLogin`, `doRegister`, `doLogout`)
- App render / tab switching (`renderApp`, `showTab`, `renderContent`)
- Calendar, Dashboard, Trade row rendering, Trades list, Stats/Analytics, Tags tab,
  Settings (CSV export, localStorage import, danger zone)
- Day popup modal, Trade add/edit modal (4-step wizard: details → psychology →
  reflection → review), Tag creation modal

**Rendering pattern**: every view is a function returning an HTML string
(`renderDash()`, `renderStats()`, etc.), assigned to `#content` or `#modal-area`
`innerHTML`. Interactivity is wired via inline `onclick="..."` attribute handlers that
call global functions — there's no addEventListener-based delegation. When editing,
follow this existing pattern rather than introducing a different event-binding style.

**Persistence pattern**: any mutation to `trades`, `tags`, or `dayNotes` should be
followed by `scheduleSave()` (debounced Supabase upsert) and a re-render
(`renderContent()` or similar). Forgetting `scheduleSave()` after a mutation is a
silent data-loss bug — changes will appear in the UI but never reach Supabase.

**Localization**: UI strings are a mix of English and Hebrew (e.g. login screen,
day-note placeholder, save-status text) — this is intentional, not mojibake, in most
of the login/status text. Match the existing language for any given piece of UI you
touch rather than translating it.

## ⚠️ Known critical issue: corrupted/broken JS in the current file

A significant portion of `index.html`'s `<script>` is **not valid JavaScript** as
currently committed, and will throw a `SyntaxError` the moment the browser parses it —
meaning most of the interactive app (adding/editing a trade, the day-note popup, the
tag-creation modal) currently does not run at all. Confirmed via `node --check` on the
extracted script content.

Two distinct corruption patterns, both apparently introduced by a copy-paste from a
chat/markdown source without stripping formatting:

1. **Stray literal ` ``` ` (triple-backtick) markdown code fences** embedded directly in
   the JS, inside template literals — e.g. around `openDayPopup`, the trade modal's
   `showModal(step===1)` branch, and the modal HTML output. These close a template
   literal early and leave dangling HTML as bare statements, cascading into further
   syntax errors.
2. **Mojibake curly quotes** (`â€œ`/`â€™`-style sequences, rendered as `â` glyphs) used
   in place of straight `'`/`"` as actual string delimiters in code — not just inside
   already-quoted strings (where they'd just be cosmetically wrong emoji/dashes) but as
   the quote characters themselves in `getElementById(...)`, object literal keys/values,
   and function bodies from roughly `openDayPopup` through the end of the file
   (`saveDayNote`, `emptyForm`, `openAdd`, `openBT`, `openEdit`, `collectFormValues`,
   `calcModalPnL`, `showModal`, `saveFromModal`, `openTagModal`).

**If you are asked to fix a bug, add a feature, or otherwise touch this file:**
- Assume the file will not run in a browser until this is fixed, and say so if the user
  seems unaware.
- Before trusting any manual edit, validate the script actually parses. A quick way:
  extract the last `<script>...</script>` block's contents and run
  `node --check <file>` on it (Node accepts modern browser JS syntax fine for this
  purpose since there's no browser-only syntax used here).
- When fixing, replace corrupted quote characters with plain ASCII `'`/`"` and delete
  the stray ` ``` ` lines — don't reformat or "improve" surrounding code while doing so,
  since the fix should be a minimal, faithful repair of the original intent (compare
  against the equivalent working patterns earlier in the file, e.g. `openEdit`/`openAdd`
  follow the same shape as later corrupted functions).
- Watch your own editor/paste path: if you copy code from a chat response into this
  file, strip markdown fences and make sure quotes land as straight ASCII quotes, not
  smart quotes — that's how this corruption likely happened in the first place.

## Testing / verification

There is no test suite. To verify a change:
1. Syntax-check the script (see `node --check` approach above) — this alone will catch
   a large class of bugs given the file's history.
2. Open `index.html` in a browser (or serve it via any static file server) and exercise
   the affected flow manually: login/register, add a trade through all 4 modal steps,
   check the calendar/dashboard/stats update, check Supabase save indicator shows
   "saved".
3. Check the browser console for runtime errors — there's no error boundary or
   reporting, errors will otherwise fail silently against the UI.

## Conventions to preserve

- No semicolons-optional style debates — the existing code uses semicolons
  consistently; match it.
- Dense, minified-looking single-line style is the existing convention (little
  whitespace, chained ternaries, arrow functions) — don't reformat unrelated code to a
  more verbose style as a side effect of a small fix.
- Colors/design tokens (hex values like `#3b82f6`, `#00c07a`, `#ef4444`) are repeated
  inline rather than centralized in CSS variables — if adding new UI, match existing
  colors for the same semantic meaning (blue = neutral/primary action, green =
  win/positive/confirm, red = loss/danger/delete, purple = backtest/psychology,
  amber = warning).
- IDs for new records use `Date.now()` (+ `Math.random()` where collisions are
  plausible within the same tick) rather than a real UUID/sequence — follow this if
  adding new record types.
