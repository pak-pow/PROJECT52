---
date: 2026-07-07
project: Project 52
week: 28
module: Quiz Platform with Leaderboards
topic: Day 4 - JavaScript & API Integration
author: Paul Vincent Aguirre
status: Complete
Tags:
  - "[[JavaScript]]"
  - "[[ES Modules]]"
  - "[[Frontend]]"
  - "[[API Integration]]"
  - "[[Dev Log]]"
  - "[[Debugging]]"
---
# 📝 DEV LOG: WEEK 28 - DAY 4 

## 1. Goal of the Day
Wire the static HTML shell from Day 3 to the live backend API using JavaScript organized across the proper `src/` folder structure using ES Modules. Fix all bugs discovered during integration.

---

## 2. Architecture Decision: ES Modules over One File
Initially all JavaScript was planned as a single `main.js`. This was corrected — code was split properly across the existing folder structure using `import`/`export` (ES Modules). No bundler needed; just `type="module"` on the `<script>` tag.

```
frontend/src/
├── api/
│   └── quizApi.js          ← fetch() wrappers for all 4 endpoints
├── utils/
│   ├── helpers.js          ← pure utility functions (no DOM access)
│   └── timer.js            ← SVG countdown ring + interval engine
├── pages/
│   ├── catalog.js          ← renderCatalog()
│   ├── quiz.js             ← renderQuiz(), renderQuestion()
│   ├── results.js          ← renderResults()
│   └── leaderboard.js      ← renderLeaderboard()
├── components/
│   └── modal.js            ← openModal(), closeModal()
└── main.js                 ← state object, event listeners, app boot
```

**Commit:** `feat(week28): switch script tag to ES module type`

---

## 3. Files Built

### `api/quizApi.js`
Four async fetch wrappers — each returns parsed JSON or throws an error:
- `fetchQuizzes()` → `GET /api/quizzes`
- `fetchQuiz(id)` → `GET /api/quizzes/<id>`
- `submitQuiz(id, payload)` → `POST /api/quizzes/<id>/submit`
- `fetchLeaderboard(id)` → `GET /api/quizzes/<id>/leaderboard`

**Commit:** `feat(week28): add API client module with fetch wrappers`

---

### `utils/helpers.js`
Pure functions with no DOM access — safe to import anywhere:
- `showView(viewId)` — toggles `.active` class between the four views
- `getTimeTaken(startTime)` — calculates elapsed seconds from `Date.now()`
- `getHeadline(score, total)` — returns a result message based on percentage
- `getRankBadge(rank)` — returns medal emoji for top 3, numbered badge for rest
- `getRankClass(rank)` — returns CSS class for gold/silver/bronze row tinting

**Commit:** `feat(week28): add helpers utility module`

---

### `utils/timer.js`
Drives the SVG countdown ring:
- `startTimer(seconds, state, onExpire)` — runs `setInterval` every 1s
  - Updates `#timer-display` text
  - Updates `stroke-dashoffset` on `#timer-circle` using: `offset = circumference × (1 - timeLeft / totalTime)`
  - Adds `.danger` class to `.timer-wrapper` when `timeLeft ≤ 10` (triggers red pulse)
  - Calls `onExpire()` when `timeLeft === 0`
- `stopTimer(state)` — clears the interval handle

**Commit:** `feat(week28): add countdown timer engine module`

---

### `pages/catalog.js`
- `renderCatalog(quizzes, onQuizSelect)` — builds quiz cards from API data and attaches click listeners

**Commit:** `feat(week28): add catalog page renderer`

---

### `pages/quiz.js`
- `renderQuiz(quiz, state)` — resets state fields for a fresh quiz run, sets header title
- `renderQuestion(index, state)` — swaps in question text, rebuilds options grid, resets Next button

**Commit:** `feat(week28): add quiz runner page renderer`

---

### `pages/results.js`
- `renderResults(data, state)` — fills score hero, headline from `getHeadline()`, time taken, and builds per-question breakdown rows with correct/incorrect styling

**Commit:** `feat(week28): add results page renderer`

---

### `pages/leaderboard.js`
- `renderLeaderboard(data, state)` — builds ranked table rows with medal badges, handles empty state

**Commit:** `feat(week28): add leaderboard page renderer`

---

### `components/modal.js`
- `openModal()` — removes `.hidden`, clears input, focuses field
- `closeModal()` — adds `.hidden`

**Commit:** `feat(week28): add modal component`

---

### `main.js` — App Entry Point
Imports from all modules above. Contains:
- **`state` object** — single source of truth: `currentQuiz`, `questions`, `currentQuestionIndex`, `answers`, `timerInterval`, `timeLeft`, `startTime`, `totalTime`, `lastResult`
- **`loadCatalog()`** — fetches and renders the quiz catalog, shows error message if backend unreachable
- **`startQuiz(id)`** — fetches quiz, calls `renderQuiz()`, `showView()`, `renderQuestion()`, and `startTimer()`
- **`handleTimerExpire()`** — fills unanswered slots with `-1`, opens modal
- **All button event listeners** — wired to the correct functions

**Commit:** `feat(week28): add main entry point with state, event wiring, and app boot`

---

## 4. Bugs Fixed

### Bug 1 — CORS Blocking Response
**Symptom:** Flask logged `200 OK` for every request, but the browser showed "Could not connect to backend."

**Root Cause:** `localhost` and `127.0.0.1` are treated as **different origins** by the browser. The CORS config only allowed `http://localhost:5500`, but Live Server served from `http://127.0.0.1:5500`.

**Fix:** Updated CORS origins list in `backend/app/__init__.py` to allow both variants plus `file://` protocol:
```python
CORS(app, resources={r"/api/*": {"origins": [
    "http://localhost:5500",
    "http://127.0.0.1:5500",
    "http://localhost:3000",
    "null",
]}})
```
**Commit:** `fix(week28): expand CORS origins to cover all local dev variants`

---

### Bug 2 — `https://` typo in API Base URL
**Symptom:** All fetch calls failed immediately.

**Root Cause:** `quizApi.js` had `https://127.0.0.1:5000/api` — Flask runs on `http`, not `https`.

**Fix:** Changed to `http://127.0.0.1:5000/api`.

**Commit:** `fix(week28): fix https typo to http in API base URL`

---

### Bug 3 — Quiz Questions Repeating (11 / 35)
**Symptom:** Quiz showed 35 questions instead of 5 — looping and repeating.

**Root Cause:** `schema.sql` seed `INSERT OR IGNORE INTO questions` had no explicit `id`. Without a primary key match, `INSERT OR IGNORE` treats every run as a new unique row. After 7 Flask restarts: 5 questions × 7 = **35 duplicates** in the DB.

**Fix — Part 1:** Added explicit `id` values (1–10) to all question INSERT statements.

**Fix — Part 2 (Architectural):** Removed all seed data from `schema.sql` entirely. Moved seeds to a dedicated standalone `backend/data/seed.py` script.

**Commits:**
- `fix(week28): add explicit IDs to question seeds to prevent duplicate inserts on restart`
- `refactor(week28): remove seed data from schema.sql — structure only`
- `feat(week28): add standalone seed.py with safe-insert and --reset flag`

---

## 5. New Seeder (`backend/data/seed.py`)

`schema.sql` now defines structure only. All sample data lives in `seed.py`.

```bash
# First time / after deleting quiz.db
python data/seed.py

# Run again safely — skips existing rows, no duplicates
python data/seed.py

# Wipe all quiz + question data and re-seed
python data/seed.py --reset
```

Features:
- Checks for existing quizzes by title before inserting — idempotent
- `--reset` flag deletes all quiz and question rows (leaderboard cascades automatically)
- Runs `schema.sql` itself first so it works without Flask running

---
