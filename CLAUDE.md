# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A personal triathlon and endurance sports training hub for **Danish Eddie**. Single-file vanilla HTML/CSS/JS SPA — no build system, no framework, no dependencies beyond Google Fonts.

- **Live URL:** https://danisheddie.github.io/training-hub/
- **GitHub repo:** https://github.com/danisheddie/training-hub
- **Primary file:** `Training Hub v2.html` — this is the active app
- `Training Hub.html` and `Training Plan.html` are legacy/archived versions, do not edit them
- `index.html` is a redirect stub to `Training Hub v2.html`

## Deploying changes

```bash
cd "/Users/danisheddie/Desktop/Traning Hub"
git add "Training Hub v2.html"
git commit -m "Description of change"
git push
```

GitHub Pages auto-deploys from `main`. Live in ~60 seconds. Check with:
```bash
until [ "$(curl -s -o /dev/null -w '%{http_code}' 'https://danisheddie.github.io/training-hub/')" = "200" ]; do sleep 5; done && echo "LIVE"
```

`gh` CLI is installed at `~/bin/gh`.

## Sync rule — critical

**Whenever `Training Hub v2.html` is edited, also update the corresponding markdown files in the same session without waiting to be asked.** The markdown files mirror the training plan content:

- `Ironman 70.3 Kenting - Training Plan.md` — full 26-week plan
- `Weekly Training Log.md` — week-by-week log template
- `Indoor Trainer Workouts.md` — Tuesday trainer session scripts

## Architecture

### Single-file SPA pattern

All CSS, HTML, and JS live in one `<script>` block at the bottom of `Training Hub v2.html`. No modules, no imports. The file is ~2400 lines.

**View switching** is done by showing/hiding `#view-hub` and `#view-race` divs. Tab switching within a race shows/hides `#tab-today`, `#tab-plan`, `#tab-log`, `#tab-gear`, `#tab-overview`.

### Global state

```js
let ST   = { view, raceId, tab }          // current navigation state
let DATA = { races: Race[] }               // all race data (also in localStorage)
let _wiz = { data: {}, step: 0 }          // wizard state
let _editTarget = { raceId, weekNum, day } // session edit modal state
let _profileEditRaceId                     // profile edit modal state
let _timers = {}                           // countdown interval IDs per race
```

### Data persistence — localStorage keys

| Key | Contents |
|-----|----------|
| `thubV2` | Full `DATA` object — all races and plans |
| `thubUser` | `{ name, year }` — sidebar user profile |
| `thubTheme` | Active theme name (`dark`, `ironman`, `slate`, `light`) |
| `log_{raceId}_w{weekNum}` | Log data for a week: `{ [day]: { status, done, note }, _refl: { note }, _custom_{day}: [{id, text, status}] }` |
| `gear_{raceId}_{groupIdx}_{itemIdx}` | Boolean — gear item checked state |

### Race object shape

```js
{
  id, name, type, date, goal, goalNote,
  splits: [{ label, dist, time, pace, cls }],
  profile: { [key]: string },        // editable key-value pairs
  phases: [{
    id, name, dates, desc, note,
    weeks: [{
      weekNum, dates, sessions: { Mon?, Tue?, Wed?, Thu?, Fri?, Sat?, Sun? },
      note, isRecovery, brick, phase
    }]
  }],
  gear: [{ group, color, items: string[] }],
  zones: [{ z, name, pct, rpe, feel, color }],
  notes, log: {}
}
```

**Critical:** `sessions` must only contain day-keyed string values. Non-session keys (`brick`, `rec`, `note`) must be destructured out before assigning to `sessions` or they corrupt `countTotal()`. The `push()` function in `buildIronman703` handles this via `const {note, rec, brick:isBrick, ...days} = r`.

### Session status system

Three states stored as `dayData.status`: `'done'` | `'skipped'` | `null`. The `done` boolean is also written for backward compatibility. Use `getStatus(dayData)` to read — never read `.done` directly.

Custom workouts are stored under `_custom_{day}` in the log data as `[{id, text, status}]` arrays, separate from the planned sessions.

### Current week detection

`getCurrentWeekNum(race)` calculates the current week by diffing today against a hard-coded plan start of `2026-05-11`. This date is the same for both pre-built races. New wizard-generated plans use the next Monday from the wizard completion date.

### Day ordering

`ALL_DAYS = ['Mon','Tue','Wed','Thu','Fri','Sat','Sun']` is a module-level constant used everywhere days need ordering. Never hardcode a day array inside a function.

### Session colour classification

`sessColor(text)` returns `swim | bike | run | brick | gym | rest`. It uses `includes()` checks on lowercased text. The `brick` keyword check must come before `bike` check.

## Athlete context (Danish Eddie)

**Races in plan:**
- Ironman 70.3 Kenting — 1 November 2026, goal: Sub-6:00
- Garmin Half Marathon — 25 October 2026, goal: Sub-1:55 (7 days before Ironman — flagged as conflict)

**Weekly swim schedule (important — was corrected mid-project):**
- Monday AM: own swim session (+ afternoon run — dual session, WFH day)
- Friday AM: own swim session (WFH day, before coaching kids at 5–6pm)
- Sunday: short easy swim poolside (during/around kids coaching 9am–12pm)
- Friday 5–6pm and Sunday 9am–12pm are **kids coaching blocks** — NOT own training

**Bike:** Tuesday indoor trainer only (structured sessions named BASE-1 through TAPER-3, documented in `Indoor Trainer Workouts.md`)

## Wizard (6 steps)

1. **Basics** — race name, type (run/tri/hyrox), date
2. **Goal** — options vary by race type via `getGoalOpts()`
3. **Fitness** — varies by type via `getFitnessFields()`; hyrox gets run + gym fields
4. **Schedule** — weekly hours, fixed coached sessions
5. **Health** — injuries
6. **Lifestyle** — chip toggles (strength, shiftwork, travel, kids, etc.) + free text; each chip maps to a specific recommendation in `CHIP_NOTES` that gets appended to the plan's `notes` field

Every option field has an "Other" fallback that shows a free-text input. Use `resolveOther(val, fallback, freeText)` in `generatePlan` — never read `d.someField` directly.

## Modals

There are two shared modals:
- **Edit session modal** (`#edit-overlay`) — used for both single-day session edits and full athlete profile edits. The `_profileEditRaceId` variable determines which save path runs; `closeEdit()` clears both `_editTarget` and `_profileEditRaceId`.
- **Wizard modal** (`#wiz-overlay`) — 6-step race creation flow.

## Themes

Four CSS themes via `[data-theme]` attribute on `<html>`: `dark` (default), `ironman`, `slate`, `light`. All colour values are CSS custom properties in `:root`. Add new theme styles as `[data-theme="name"]{ --bg:...; }` blocks.

## Mobile

Breakpoint at `768px`. On mobile:
- Sidebar becomes a slide-in drawer; `#mob-bar` top bar replaces it
- Swipe right from left edge (≤40px) opens the drawer; swipe left closes it
- Wizard and edit modal become bottom sheets (`border-radius: 18px 18px 0 0`)
- Edit buttons in plan tab are always visible on mobile (not hover-only)

## Icons

All icons are inline SVG strings in the `IC` object. Render with `` IC.iconName `` directly in template literals. The `ic(name, size)` helper exists but is rarely used.
