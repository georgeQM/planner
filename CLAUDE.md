# day.plan — Planner App

## Project Overview
A self-contained, single-file daily planner and task management web app (`index.html`). No build process, no framework, no dependencies beyond Google APIs.

## Architecture
- **Single file**: All HTML, CSS, and JS lives in `index.html`
- **Vanilla JS**: ES6+ with no frameworks — direct DOM manipulation
- **Data**: localStorage (primary) + Google Drive JSON sync (optional cloud backup)
- **Branch**: active development on `ai-dev`, stable on `main`

## Data Structures

```js
weeklyPlan = {
  0: [], 1: [], ..., 6: []  // routine templates per day-of-week (0=Sun)
  // each entry: { id, name, start, end, type, recurringDays: [0-6], createdAt }
}

events = {
  "YYYY-MM-DD": [{ id, name, start, end, type, done, fromTemplate, templateId }]
}

poolTasks = [{ id, name }]
```

## Event Types
- **routine** (orange) — recurring templates from `weeklyPlan`, auto-instantiated per day
- **task** (blue) — manually scheduled events
- **unforeseen** (red) — urgent unplanned tasks, added via FAB button

## Key Constants
- `HOUR = 120` — px height of one hour on the grid
- `SNAP = 20` — px snap increment (~10 minutes)
- `TIME_STEP = 20` — px resize increment
- `WP_HEADER_H = 28` — sticky day-header height in the week planner grid; all event tops are offset by this

## Key Functions
- `getEvents(d)` — gets events for a date; auto-creates routine instances from `weeklyPlan` (filters by `recurringDays`)
- `renderGrid()` / `renderEvents()` / `renderPool()` — main render functions
- `renderWeekPlan()` — renders the 7-column weekly template editor grid
- `onDragMove()` / `onResizeDown()` / `onPoolDown()` — drag-and-drop handlers
- `openModal(isUnforeseen)` — event creation dialog for calendar day
- `openTemplateModal(preDay)` — reuses modal in template mode (day selector, no type selector)
- `confirmModal()` / `closeModal()` — modal confirm and close (closeModal resets template-mode fields)
- `openSettings()` / `closeSettings()` — settings overlay navigation
- `openWeekPlanner()` / `closeWeekPlanner()` — week planner overlay navigation
- `saveFile()` / `loadFile()` / `handleAuth()` — Google Drive integration
- `scheduleSave()` — debounced save to localStorage always; Drive only when accessToken exists

## Google Drive Integration
- OAuth 2.0 via `https://accounts.google.com/gsi/client`
- Scope: `https://www.googleapis.com/auth/drive.file`
- Stores data in a "Planner" folder as `planner-data.json`
- For `file://` protocol: use `null` as Authorized JavaScript Origin in Google Cloud Console

## Development Notes
- No build step — open `index.html` directly in a browser
- Changes to `index.html` are immediately testable
- localStorage is the fallback when Drive is unavailable
- The app auto-scrolls to the current hour on load for today's view


## UI Panels
- **Settings panel** (`#settings-panel`, z-index 300) — gear icon (⚙) in topbar opens it
- **Week planner panel** (`#week-planner-panel`, z-index 400) — opened from Settings → "Plan Your Week"
- Both panels use `history.pushState` for mobile back-gesture support
- `popstate` handler: closes week planner first (re-pushes settings state), then settings on second back

## Current State
- v1 in progress, branch: ai-dev
- `weeklyPlan` is the single source of truth for routine templates (replaces old `routineTemplates`)
- Week planner editor: add/edit/delete templates per day-of-week, persisted via `scheduleSave()`
- Template modal: name, start, end, type (routine/task), recurring day checkboxes (Sun–Sat); defaults all days checked
- `getEvents(d)`: auto-fills new days from `weeklyPlan` templates; guard is `events[k] === undefined` (explicit, not truthiness-based); never calls renderAll()
- Deleted calendar instances stay deleted — deletion uses `filter` (keeps key as `[]`), never `delete events[k]`
- All calendar event types stored with `-ev` suffix (`'routine-ev'`, `'task-ev'`, `'unforeseen-ev'`); weeklyPlan templates store without suffix
- Instance edit: tap event title → `openEditModal(ev)` → edits only that `events[dateStr]` entry; uses 5px pointer threshold to distinguish tap from drag
- Template edit: week planner ✎ → `openTemplateModal(null, editId)` → edits `weeklyPlan` only; shows "Changes apply to future days only" note
- Pool drop y-offset uses `cal-scroll` scrollTop (not `events-col` which doesn't scroll)
- Do not introduce frameworks, bundlers, or external dependencies
- Do not split into multiple files yet (v2 decision)

## Rules
- Mobile-first always
- Match existing CSS variables and visual style exactly
- localStorage saves always, Drive only when accessToken exists
- Never regenerate already-created day instances
- Deleted calendar instances stay deleted

## Claude Code Rules
- Always use pointer events, never mouse events
- weeklyPlan keys are 0-6 integers, not strings
- Never call renderAll() inside getEvents()
- Single file only — no splitting until v2