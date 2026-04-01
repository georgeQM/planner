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
  "YYYY-MM-DD": [{ id, name, start, end, type, done, fromTemplate, templateId, createdAt, returnedToPool? }]
}

poolTasks = [{ id, name, createdAt }]

completedTasks = [{ id, name, completedAt, duration, date }]
// completedAt: ISO string; duration: minutes; date: "YYYY-MM-DD"
```

## Event Types
- **routine** (orange) — recurring templates from `weeklyPlan`, auto-instantiated per day
- **task** (blue) — manually scheduled events
- **unforeseen** (red) — urgent unplanned tasks, added via type dropdown in the Add task modal

## Key Constants
- `HOUR = 240` — px height of one hour on the main calendar grid
- `SNAP = 40` — px snap increment (10 minutes)
- `TIME_STEP = SNAP` — px resize increment
- `WP_HEADER_H = 28` — sticky day-header height in the week planner grid; all event tops are offset by this
- `WP_HOUR = HOUR / 2 = 120` — px height of one hour in the week planner grid (half scale of main calendar)
- `.wp-time-col` / `.wp-day-col` heights hardcoded at `WP_HOUR * 24 = 2880px`

## Key Functions
- `getEvents(d)` — gets events for a date; auto-creates routine instances from `weeklyPlan` (filters by `recurringDays`)
- `renderGrid()` / `renderEvents()` / `renderPool()` — main render functions
- `renderWeekPlan()` — renders the 7-column weekly template editor grid
- `renderWeekStrip()` — renders the 7-day strip; uses `querySelectorAll('.day-btn').forEach(e=>e.remove())` (not `innerHTML=''`) to preserve static `#pool-btn`
- `onDragMove()` / `onResizeDown()` / `onPoolDown()` — drag-and-drop handlers
- `openModal()` — event creation dialog for calendar day (always opens as 'Add task'; unforeseen selectable via type dropdown)
- `togglePool()` — hides/shows the pool sidebar; state persisted in `localStorage.poolHidden`
- `autoReconnect()` — attempts silent OAuth reconnect to Drive on load using `prompt:''`; on success sets `#drive-status` to green "● Connected"
- `openTemplateModal(preDay)` — reuses modal in template mode (day selector, no type selector)
- `confirmModal()` / `closeModal()` — modal confirm and close (closeModal resets template-mode fields, `m-type.onchange`, and `recur-all` checkbox)
- `openSettings()` / `closeSettings()` — settings overlay navigation
- `openWeekPlanner()` / `closeWeekPlanner()` — week planner overlay navigation
- `openHistory()` / `closeHistory()` / `renderHistory()` — history panel navigation and rendering
- `returnUncompletedToPool()` — moves past-day uncompleted `task-ev` (non-template) events to pool; guards with `ev.returnedToPool` flag; called on load and at midnight
- `saveFile()` / `loadFile()` / `handleAuth()` — Google Drive integration
- `scheduleSave()` — debounced save to localStorage always; Drive only when accessToken exists

## Google Drive Integration
- OAuth 2.0 via `https://accounts.google.com/gsi/client`
- Scope: `https://www.googleapis.com/auth/drive.file`
- Stores data in a "Planner" folder as `planner-data.json`
- For `file://` protocol: use `null` as Authorized JavaScript Origin in Google Cloud Console
- `autoReconnect()` fires on load — uses `prompt:''` for silent token acquisition; silently no-ops if not previously authorized
- Drive connect/disconnect button is `#settings-auth-btn` inside the Settings panel (not topbar)
- Drive connection state shown in topbar via `#drive-status` span: "● Connected" (green) / "● Disconnected" (red)
- Initial state set immediately after `autoReconnect()` call: red "● Disconnected" (overwritten to green if autoReconnect succeeds)

## Development Notes
- No build step — open `index.html` directly in a browser
- Changes to `index.html` are immediately testable
- localStorage is the fallback when Drive is unavailable
- The app auto-scrolls to the current hour on load for today's view


## UI Panels
- **Settings panel** (`#settings-panel`, z-index 300) — gear icon (⚙) in topbar opens it; contains "📅 Plan Your Week", "📋 History", and "🔗 Connect Drive" rows
- **Week planner panel** (`#week-planner-panel`, z-index 400) — opened from Settings → "📅 Plan Your Week"
- **History panel** (`#history-panel`, z-index 300) — opened from Settings → "📋 History"; shows completed tasks newest-first with date, time, and duration
- All panels use `history.pushState` for mobile back-gesture support
- `popstate` handler order: history → week planner (re-pushes settings state) → settings
- **Pool sidebar** — toggled by `▤` button (`#pool-btn`) in the **week-strip** (not topbar); hides via `.pool.pool-hidden{display:none}`; state persisted in `localStorage.poolHidden`

## Topbar
`.topbar-right` contains (left to right): `#drive-status` → `#sync-status` → `#gear-btn`
No auth button in topbar — Drive auth lives in Settings.

## cal-event Structure
HTML child order inside each `.cal-event`:
1. `.ev-check` — done circle, `position:absolute; top:5px; left:5px; width:16px; height:16px`
2. `.ev-edit` — edit button (✏), `position:absolute; top:24px; left:4px; width:18px; height:18px; font-size:12px`
3. `.ev-handle` — drag handle (⠿), `position:absolute; top:3px; left:50%; transform:translateX(-50%); touch-action:none; cursor:grab`
4. `.ev-title` — event name; gets `marginTop:'18px'` when event is ≥30 min tall
5. `.ev-time` — time range
6. `.ev-actions` — top-right wrapper containing `.ev-del` only
7. `.ev-resize` — bottom resize bar

`.cal-event` CSS: `padding:5px 26px 12px 28px` (28px left clears the check/edit column); `touch-action:pan-y`; `cursor:default`

## Drag System
- **Drag initiated by**: `pointerdown` on `.ev-handle` only — touching anywhere else on the event does NOT start drag (native scroll applies)
- **`onMoveDown`**: uses `e.currentTarget.closest('.cal-event')` to get the event element (since listener is on `.ev-handle`)
- **setPointerCapture**: deferred — NOT called in `onMoveDown`; called lazily on first `onDragMove` tick via `dragState.captured` flag
- `dragState` for move: `{ type:'move', id, startY, origTop, dur, pointerId, captured:false }`
- Resize (`onResizeDown`): setPointerCapture is **immediate** (not deferred)
- Pool drag: setPointerCapture is **immediate**

## Event Editing
- **Edit modal**: opened via `.ev-edit` button (✏) — NOT by tapping the event body (tap-to-edit removed)
- **Instance edit**: `openEditModal(ev)` → edits only that `events[dateStr]` entry
- **Template edit**: week planner ✎ → `openTemplateModal(null, editId)` → edits `weeklyPlan` only; shows "Changes apply to future days only" note

## Current State — v1 (ai-dev branch)

### Completed
- Fix: pool drop y-position (removed double-counting scroll offset)
- Fix: modal z-index raised to 500 (above all panels)
- Fix: mobile scroll over calendar events (removed touch-action:none override on .cal-event)
- Fix: drag only triggers from ⠿ handle (pointerdown moved to .ev-handle, delayed setPointerCapture)
- Fix: resize handle hit area shrunk to centered 32px pill
- Feature: calendar scale — HOUR=240, SNAP=40 (10min snapping), WP grid at HOUR/2=120
- Feature: remove FAB (+ Unforeseen button gone, openModal simplified)
- Feature: pool toggle button (▤) in week strip, persisted in localStorage
- Feature: Drive auth moved to Settings panel with 🔗/⛓️‍💥 icons
- Feature: drive status indicator in topbar (● Connected / ● Disconnected)
- Feature: history panel (Settings → 📋 History) — tracks completed tasks with date/time/duration, ↩ Pool button
- Feature: midnight rollover — uncompleted task-ev events (fromTemplate:false) return to pool at midnight
- Feature: age indicator — red border on pool tasks and calendar events with createdAt > 7 days
- Feature: routine edit propagation — editing a template updates current week instances from current time forward
- Feature: All/None day toggle in template modal recur row
- Feature: task-ev type in template modal hides recur days, sets recurringDays:[]
- Feature: test environment banner (orange, bottom) on non-production hosts
- Feature: auto-reconnect Drive on load (prompt:'', silent fail)
- Feature: event overlap — full interval-overlap detection (greedy column-slot assignment); side-by-side column layout (slotOf/totalSlots), drag z-index boost to 50, reset on drop
- Fix: Drive sync on load — compare `savedAt` timestamps; keep newer of local vs Drive, push local to Drive if local is newer (keep in sync with code)
- `weeklyPlan` is the single source of truth for routine templates (replaces old `routineTemplates`)
- All calendar event types stored with `-ev` suffix (`'routine-ev'`, `'task-ev'`, `'unforeseen-ev'`); weeklyPlan templates store without suffix
- `getEvents(d)`: guard is `events[k] === undefined` (explicit, not truthiness-based); never calls renderAll()
- Deleted calendar instances stay deleted — deletion uses `filter` (keeps key as `[]`), never `delete events[k]`
- Pool drop y-offset: `e.clientY - rect.top` only — `getBoundingClientRect()` already accounts for scroll, do not add scrollTop
- Week planner grid uses `WP_HOUR=120` (half of main `HOUR=240`); event positioning uses `Math.round(timeToY()/2)` — do not use `HOUR` inside `renderWeekPlan()`
- **History**: `completedTasks` array persisted in localStorage + Drive; populated when ev-check toggled done, pruned on undone; rendered in `#history-panel` newest-first
- **Midnight rollover**: `returnUncompletedToPool()` runs on load and at midnight via `setTimeout`; uses `ev.returnedToPool = true` guard; deduplicates by name
- Test banner: checks `window.location.hostname !== 'georgeqm.github.io'`; shows orange fixed banner at bottom on non-production hosts

### Deferred to v1.1 or v2
- Past days horizontally scrollable in week strip
- Week strip dots showing which days have tasks planned
- Inline event name edit on double-tap
- Drive refresh token / persistent auth (requires backend)

### Branch status
- ai-dev: all features above committed and pushed
- main: stable drag-and-drop baseline
- Production URL: https://georgeqm.github.io/planner/
- Ready for v1 → main merge when Jorge confirms stable

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
- `renderWeekStrip()` clears day buttons via `querySelectorAll('.day-btn').forEach(e=>e.remove())` — do NOT use `innerHTML=''` (would delete the static `#pool-btn`)
- Drag starts only from `.ev-handle`; `onMoveDown` must use `e.currentTarget.closest('.cal-event')` to get the event element
- Never include Co-Authored-By or any Claude attribution in commit messages
