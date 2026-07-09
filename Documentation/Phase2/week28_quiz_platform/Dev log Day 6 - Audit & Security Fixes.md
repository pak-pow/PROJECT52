---
date: 2026-07-09
project: Project 52
week: 28
module: Quiz Platform with Leaderboards
topic: Day 6 - Codebase Audit & Security Fixes
author: Paul Vincent Aguirre
status: Complete
Tags:
  - "[[Security]]"
  - "[[Database]]"
  - "[[Optimization]]"
  - "[[Bug Fixes]]"
  - "[[Dev Log]]"
---
# 📝 DEV LOG: WEEK 28 - DAY 6 

## 1. Goal of the Day
Perform a comprehensive audit of the backend and frontend codebase, identify potential resource leaks, security flaws, style bugs, and structural issues, and apply clean, targeted fixes for all identified points.

---

## 2. Issues Audited & Fixed

### 🔒 Backend Security & Input Validation
- **Problem:** If a client sent a payload containing non-integer answer indices (e.g. `answers: ["0", null, true]`), the python backend would crash with a `TypeError` when evaluating comparisons during grading.
- **Fix:** Added element-level type checking in `validate_submission` inside `quiz_service.py` to ensure that every answer index in the list is strictly an integer.
- **Commit:** `fix(week28): validate that all elements in answers are integers to prevent TypeErrors in grading`

### 🗄️ SQLite Connection Resource Leaks
- **Problem:** Database helpers in `quiz_model.py` called `conn.close()` manually at the end of each method, but did not guard against exceptions. If a database query crashed or threw an exception, the connection would leak, eventually locking the SQLite database.
- **Fix:** Refactored all 5 model functions using `try...finally` blocks to guarantee `conn.close()` is executed under all circumstances.
- **Commit:** `fix(week28): ensure SQLite connections are closed in try...finally blocks to prevent connection leaks on database exceptions`

### 🎨 Frontend UI Polish: SVG Gradient Countdown Timer
- **Problem:** In splitting style templates in Day 5, the CSS reference `stroke: url(#timer-gradient)` became broken because the linear gradient definition markup was missing from the HTML/SVG tag structure.
- **Fix:** Injected a `<defs>` node containing a `<linearGradient id="timer-gradient">` structure directly inside the SVG element in `quiz.js`'s template.
- **Commit:** `fix(week28): move imports to top of quiz.js and add gradient definitions to the SVG timer ring`

### ⚡ Submitting State Lock
- **Problem:** The `state.isSubmitting` flag used to prevent duplicate submissions was never reset to `false` on a successful submit request, which could cause locks during subsequent quiz sessions.
- **Fix:** Added `state.isSubmitting = false;` in the successful transaction path in `main.js`.
- **Commit:** `fix(week28): reset state.isSubmitting to false on successful quiz submission`

---

## 3. General Repository & Code Cleanup

### 🧹 Dead Code Removal
- **Action:** Deleted `frontend/src/assets/style.css` (855 lines of obsolete styles replaced by Day 5's 6 split CSS files).
- **Commit:** `clean(week28): remove old monolithic style.css file`

### 🛡️ Gitignore Configurations
- **Action:** Added SQLite system temporary files (`*.db-journal`, `*.db-wal`, `*.db-shm`, `*.sqlite*`) to the root `.gitignore` to prevent database locks or environment cache data from accidentally being pushed to the repository.
- **Commit:** `fix(week28): add SQLite journal, WAL, and temporary database files to .gitignore`
