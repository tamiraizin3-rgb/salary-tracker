# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

A single-file Hebrew RTL web app (`index.html`) for Israeli hourly workers to track shifts, calculate wages, and export to Webtime (cloud.webtime.co.il). All logic, CSS, and HTML live in one file. The `client/` and `server.js` are an abandoned earlier attempt using React + SQLite — **ignore them**. The active codebase is `index.html` only.

## Running locally

```bash
node serve.js        # serves index.html at http://localhost:4321
```

No build step. Edit `index.html`, refresh browser. Changes go live on push to `main` via GitHub Actions → GitHub Pages at `https://tamiraizin3-rgb.github.io/salary-tracker/`.

## Deployment

Push to `main`. GitHub Actions deploys the whole repo root as a GitHub Pages site (workflow: `.github/workflows/deploy.yml`, build type: `workflow` not `legacy`).

## Architecture of index.html

All code is inside a single `<script type="module">` at the bottom of the file.

**Storage layer** — two modes selected at first run, persisted in `localStorage('st_mode')`:
- `useFirebase = true`: Firestore via Firebase JS SDK (lazy-imported from CDN). Project: `my-salary-app-8eba4`.
- `useFirebase = false`: `localStorage` key `st_entries` / `st_settings`.

**Key data functions:**
- `loadEntries(month)` / `saveEntryDB(id, data)` / `deleteEntryDB(id)` — CRUD, both backends
- `loadSettings()` / `saveSettingsDB(data)`
- `getOpBonusOffset(month)` — async, queries ALL months before `month` to count cumulative מבצעי shifts (needed for cross-month bonus calculation)

**Wage calculation (pure functions, no side effects):**
- `calcEntryGross(e, s)` — per-entry gross. Driving hours deducted from overtime tiers highest-first (200→175→150→125→regular). Does NOT include op_bonus (monthly-only).
- `entryAshal(e, s)` — meal allowance; driving hours excluded from hour count; deducts ₪50 if `has_aziza`.
- `entryTotalHours(e)` — work hours only, excludes `driving_hours`.
- `calcSummary(entries, s, opBonusOffset)` — full monthly summary. Returns `{ totals, pay, gross, base_pay, hourly_pay, breakDays, deductions, net, employer, total_employer, opBonusOffset, op_bonus_cumulative, weekendMivtziShifts }`.

**Bonus systems:**
- **מבצעי**: checkbox per entry. Cumulative across all time. Shifts 1–9 = no bonus; shift 10+ = ₪500 each. `getOpBonusOffset(month)` provides the count from before the current month.
- **בונוס שישי-שבת**: entries marked מבצעי AND on Friday(dow=5)/Saturday(dow=6). First shift free, ₪200 each after. Calculated in `calcSummary` as `Math.max(0, weekendMivtziShifts - 1) * 200`.

**Auto-detection** (`autoCalcHours`): given date + start/end times, fetches Hebcal API to detect Shabbat/holidays, splits total hours into regular/OT tiers/shabbat/holiday buckets. Night shifts (overlapping 22:00–06:00) use 7h legal workday instead of 8h.

**Pages:** dashboard, log (entry form), monthly, paystub, settings — each a `<div class="page">` shown/hidden via `navigate(page)`.

**Webtime export** (`exportWebtime()`): generates a table + an auto-fill JavaScript snippet. The snippet format uses `time_start_HH_{day}`, `time_start_MM_{day}`, `time_end_HH_{day}`, `time_end_MM_{day}` where `{day}` = day-of-month integer (e.g. row 10 = 10th of the month).

## Important conventions

- **RTL layout**: first child in a flex row = rightmost element visually. Month nav: right arrow (›) = `monthShift(-1)` (go back), left arrow (‹) = `monthShift(1)` (go forward).
- **Date storage**: `YYYY-MM-DD` in Firestore/localStorage. Display: `dd/mm/yyyy` via `dayLabel()`.
- **`base_pay`** (used for pension/קרן השתלמות): regular hours pay only, after break deduction. Excludes overtime, driving, bonuses.
- **Break deduction**: −0.5h × base rate for shifts where `entryTotalHours(e) > 6`. Driving hours do NOT count toward this threshold.
- Pension, קרן השתלמות (employee + employer) calculated on `base_pay` only.
- `settings` object uses `DEFAULTS` as fallback — always spread: `{ ...DEFAULTS, ...loaded }`.
