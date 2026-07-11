---
date: 2026-07-11
project: Project 52
week: 28
module: Quiz Platform with Leaderboards
topic: Day 8 - Category Filters & UX Polish
author: Paul Vincent Aguirre
status: Complete
Tags:
  - "[[Accessibility]]"
  - "[[WSGI]]"
  - "[[Documentation]]"
  - "[[Design]]"
  - "[[Dev Log]]"
---

# 📝 DEV LOG: WEEK 28 - DAY 8 (CATEGORY FILTERS & UX POLISH)

## 1. Goal of the Day
Polish the user experience by adding category filtering tabs for expanded quizzes, implement keyboard-driven accessibility shortcuts, add visually rich danger pulses to the timer ring, configure a production WSGI entrypoint, and compile final project-wide documentation.

---

## 2. Refactoring & New Features

### 📂 Catalog Category Filters
- **Problem:** As our quiz content grows (now 4 seeded quizzes), it becomes harder for users to locate specific topics on one page.
- **Fix:** Added a responsive filter button bar above the quiz catalog grid. Built logic inside `catalog.js` to handle dynamic filtering ("All", "Web Dev", "Python", "JavaScript", "Databases") with clean active tab state transitions.
- **Commit:** `feat(week28): add category filter tabs to catalog view`

### ⌨️ Keyboard-driven Speedrunning Shortcuts
- **Problem:** Quiz takers had to click option buttons and the navigation button manually, limiting speed and accessibility.
- **Fix:** Registered a global keydown handler active during quiz play. Maps numeric keys `1` through `4` to highlight options, and `Enter` to submit the current question.
- **Commit:** `feat(week28): add keyboard shortcuts (1-4, Enter) for quiz navigation`

### ⏱️ Red Shadow Pulsing Timer Ring
- **Problem:** The original danger warning state (under 10s) only toggled stroke colors, lacking visual urgency.
- **Fix:** Updated `quiz.css` danger keyframes to scale the timer number text, pulsate the circular stroke width, and cast an outer-glow red drop shadow pulse.
- **Commit:** `style(week28): add pulse animation to timer ring during danger state`

---

## 3. Production Deployment & Documentation

### 🚀 Waitress WSGI Server Entrypoint
- **Problem:** Developers running the app in production were greeted with Flask's development environment warning.
- **Fix:** Added `wsgi.py` leveraging the `waitress` package to serve the app on port 5000 securely. Added `waitress` to `requirements.txt`.
- **Commit:** `feat(week28): add production WSGI entrypoint with Waitress server`

### 📝 Comprehensive Project README.md
- **Action:** Created a root `README.md` documenting installation instructions, seeder execution commands (`seed.py --reset`), route unit tests (`pytest`), and client-side setup instructions.
- **Commit:** `docs(week28): add comprehensive project README guide`

---

## 4. Verification Check
- All **25 unit test cases** pass cleanly:
  ```bash
  tests\test_quiz_routes.py .........................                      [100%]
  ============================= 25 passed in 1.32s ==============================
  ```
  