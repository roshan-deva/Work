# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Roshan's Impact Tracker (RIT) — a PWA task logger for Welspun Tubular Little Rock operations. Primary use: logging daily tasks from meetings, shift logs, Granola AI meeting notes, and Notion AI meeting notes. Deployed as a GitHub Pages static site and installed as a home screen app on iPhone.

## No Build System

Vanilla HTML/CSS/JS — no npm, no bundler, no transpilation. Edit `06Opstask/index.html` directly and open in a browser.

**Files:**
- `06Opstask/index.html` — entire app (HTML + CSS + JS, all inline)
- `06Opstask/manifest.json` — PWA manifest
- `06Opstask/sw.js` — service worker (cache-first, offline support). Cache name: `rit-v1`. Bump this string whenever deploying changes so stale installs refresh.

**Preview locally:**
```
python -m http.server 3000 --directory 06Opstask
```
`.claude/launch.json` is configured for this — use `preview_start` with server name `wtsl-ops`.

**Deploy:** Local branch is `deploy` (tracks `origin/main` of `https://github.com/roshan-deva/Work`). Push with:
```
git push origin deploy:main
```
GitHub Pages URL: `https://roshan-deva.github.io/Work/06Opstask/index.html`

The `Work` repo has other HTML files at its root — never push local `master` directly to `origin/main` (it would clobber them). Always use the `deploy` branch.

## Data Model

Tasks stored in `localStorage` under key `wtslops` as a JSON array:

```js
{
  id: string,          // Date.now() string
  text: string,
  source: string,      // 'Self'|'Verbal'|'Meeting'|'Email/Teams'|'Shift Log'|custom person name
  done: boolean,
  synced: boolean,     // true = pushed to Notion (shows "In Notion" status)
  notionId: string|null,
  dueDate: string|null, // 'YYYY-MM-DD'
  date: string,        // 'YYYY-M-D' via dk(), used for today-filter
  ts: number           // Date.now() timestamp
}
```

`priority` was removed — old tasks may still carry it, but it's unused. Due date coloring replaces it: red = past due, orange = 0–7 days, gray = 8+ days or null.

Custom people stored under `wtslops_people` as a JSON array of strings. Notion credentials stored under `wtslops_notion` as `{token, databaseId}`.

`date` uses `dk()` (returns `YYYY-M-D`) — today-filtering uses `t.date === dk()`.

## Architecture

All logic is a single `<script>` block at the bottom of `index.html`. Key functions:

| Function | Purpose |
|---|---|
| `render()` | Rebuilds task list, KPI cards, and source-chip counts. Applies `filt` (status tab) + `srcFilt` (source chip) + `searchQ` before rendering. Sorts via `dueSortKey`. |
| `row(t)` | Returns HTML string for one task row. Uses `data-id` / `data-act` — **no inline onclick**. |
| `taskStatus(t)` | Returns `'done'`/`'notion'`/`'overdue'`/`'pending'` — single source of truth for status display. |
| `dueColor(dueDate)` | Returns hex color for due date: red / orange / gray. |
| `dueSortKey(t)` | Sort comparator: overdue first, then by days remaining, synced/done last. |
| `addTask()` | Reads `#ti` + `#src` + `#dueInp`, unshifts to `T`, calls `sv()` + `render()`. |
| `markDone(id)` | Toggles `done`; clears `synced` when marking done. |
| `delTask(id)` | Removes task by id after `confirm()`. |
| `importGranola()` | Parses `#gp` textarea. Strips bullets/numbers/checkboxes, filters lines < 3 chars. Stamps `source` from `#impSrc` select (Granola or Notion AI). |
| `setSF(el, sf)` | Sets status filter (`filt`): `all`/`pending`/`notion`/`done`. Updates `.st` active state. |
| `setSrcF(el, sf)` | Sets source filter (`srcFilt`): `'all'`/`'Granola'`/`'Notion AI'`. Updates `.sc` active state. |
| `buildSrcOptions()` | Rebuilds `#src` select from base options + `customPeople`. Called at init and after people changes. |
| `showTab(tab)` | Switches between `tasks`/`analytics`/`granola`/`settings` panels. Calls `renderAnalytics()` when switching to analytics. |
| `openFab()` / `closeFab()` | Shows/hides the FAB overlay sheet for task entry. |
| `doVoice()` | Toggles `webkitSpeechRecognition`. Fills `#ti` on result. |
| `analyticsFor(period)` | Returns `{captured, done, synced, open, byDay, bySource}` for `'day'`/`'week'`/`'month'`. |
| `renderAnalytics(period)` | Destroys + recreates `barChartInst` (stacked bar) and `donutChartInst` (doughnut) via Chart.js. |
| `expReport()` | Copies analytics summary to clipboard (iOS-safe); falls back to `dl2()`. |
| `expCSV()` | Exports all tasks as CSV. iOS uses `dl2()` textarea fallback. |
| `expWeekly()` | Exports a plain-text weekly summary (Done / In Notion / Pending). Tries `navigator.clipboard` first. |
| `clrDone()` | Removes all `done:true` tasks after `confirm()`. |
| `notionPush(id)` | POSTs to `https://api.notion.com/v1/pages`. Sends: Name, Source, Status ('To Do'), Date Logged, Due Date. Sets `t.synced = true`, `t.notionId`. |
| `syncAllPending()` | Sequential `pushNext()` loop with 350 ms delay between tasks. Disables button during run. |
| `testNotionConn()` | GETs the configured database to verify credentials. |
| `loadNotionSettings()` | Reads `wtslops_notion` from localStorage and populates Settings form fields. Called once at init. |
| `getNotionSettings()` | Returns parsed `wtslops_notion` object `{token, databaseId}`. Internal helper used by all Notion functions. |
| `sv()` | Saves `T` to `localStorage`. |
| `toast(m, isErr)` | Shows bottom toast for 2.4 s. Pass `true` for red error variant. |
| `dl2(name, content, type)` | iOS-safe download: shows fullscreen copyable textarea on iOS, blob URL download elsewhere. |

**Init sequence** (bottom of script): `buildSrcOptions()` → `loadNotionSettings()` → `render()`

**Event delegation:** All card interactions (checkbox, delete, Notion push) handled by a single `document.addEventListener('click', ...)` reading `data-act` (`done`/`del`/`notion`) and `data-id`. Never add inline `onclick` to row HTML — ID type mismatches (`string` vs `number`) break `===` comparisons.

**Global state:**
- `T` — task array (source of truth, mirrors localStorage)
- `filt` — active status tab (`'all'`/`'pending'`/`'notion'`/`'done'`)
- `srcFilt` — active source chip (`'all'`/`'Granola'`/`'Notion AI'`)
- `searchQ` — current search string (lowercase)
- `activeTab` — active panel name (`'tasks'`/`'analytics'`/`'granola'`/`'settings'`)
- `activePeriod` — analytics period (`'day'`/`'week'`/`'month'`)
- `barChartInst` / `donutChartInst` — Chart.js instances (destroyed before re-create)
- `SR` / `isRec` — speech recognition state

## CSS

All styles inline in `<style>` in `<head>`. Design system via CSS custom properties on `:root`:

```css
--bg, --card, --surf, --bdr        /* surfaces */
--acc (#e8ff3c)                    /* yellow-green accent */
--pur, --purdim, --purbd           /* purple (log button, filter chip) */
--txt, --dim, --muted              /* text hierarchy */
--grn, --org, --red, --blu, --pnk  /* status/priority colors */
--grndim, --orddim, --reddim, --bludim, --pnkdim  /* dim variants */
```

Task row classes: `.trow` (base), `.done-row`, `.notion-row`. Status label classes: `.s-over`, `.s-notion`, `.s-done`, `.s-pend`.

Source chips (`.sc`) use `--blu` / `--bludim` for active state — filter by import source (Granola / Notion AI). Status tabs (`.st`) use `--acc` / `--adim`.

Header uses `padding-top: calc(10px + env(safe-area-inset-top))` — required for iPhone notch/Dynamic Island with `black-translucent` status bar. Bottom nav uses `env(safe-area-inset-bottom)` similarly.

## Meeting Notes Import Workflow

No direct API integration for meeting notes. Workflow:

1. Ask Claude (in Claude Code or claude.ai) to fetch action items from Granola / Notion AI meeting notes
2. Copy the resulting plain-text list
3. Open app → 🌿 Granola tab → paste into textarea → select source (🌿 Granola or 📝 Notion AI) → "Import Tasks from Paste"

`importGranola()` strips leading `- * • ·` bullets, `[ ]` checkboxes, and `1. 2)` numbered prefixes. Each surviving line becomes one task with `source` from `#impSrc`.

**Claude Code can fetch directly** via MCP tools:
- Granola: `mcp__67a16353-*` tools (`list_meetings`, `get_meeting_transcript`)
- Notion: `mcp__1c7c41a1-*` tools (`notion-search`, `notion-fetch`) — requires Notion integration connected

## Notion Sync

Settings tab stores a Notion API key and database ID under `wtslops_notion` in localStorage. `notionPush(id)` POSTs to the Notion API (v1, `2022-06-28`) and marks the task `synced: true`. Synced tasks appear under the "In Notion" status tab and show ✓ instead of the 🔗 push button.

Notion database columns expected: **Name** (title), **Source** (select), **Status** (select), **Date Logged** (date), **Due Date** (date, optional).
