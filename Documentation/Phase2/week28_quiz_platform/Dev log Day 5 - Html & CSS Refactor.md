---
date: 2026-07-08
project: Project 52
week: 28
module: Quiz Platform with Leaderboards
topic: Day 5 - CSS & HTML Modularity refactoring
author: Paul Vincent Aguirre
status: Complete
Tags:
  - "[[HTML Split]]"
  - "[[CSS Modularization]]"
  - "[[Live Server]]"
  - "[[Dev Log]]"
---
# 📝 DEV LOG: WEEK 28 - DAY 5 

## 1. Goal of the Day
Clean up the front-end layout structure by splitting the massive monolithic `style.css` and `index.html` files into modular, maintainable pages and style sheets. Fix user submission page reload issues.

---

## 2. Refactoring Actions

### ✂️ HTML Split
- Moved the static HTML templates for all pages out of `index.html` and turned them into exported string constants inside their respective page JS files (`catalog.js`, `quiz.js`, `results.js`, `leaderboard.js`, `modal.js`).
- Reduced `index.html` to a 40-line minimal layout container shell. `main.js` injects the templates dynamically on boot.

### ✂️ CSS Split
- Split `style.css` (856 lines) into 6 modular, scoped files linked directly in `index.html`:
  * **`base.css`**: Variables, shared buttons, view system, and responsive utilities.
  * **`catalog.css`**: Banner header, catalog cards, and loading skeletons.
  * **`quiz.css`**: sticky bar, timer ring SVG, and option buttons.
  * **`results.css`**: Score rings, correct/incorrect rows.
  * **`leaderboard.css`**: Table layout, gold/silver/bronze tints.
  * **`modal.css`**: Blur overlay, cards, and input styling.

---

## 3. Live Server Reload Fix
- **Problem:** Database writes to `quiz.db` on submission triggered a workspace change, causing Live Server to refresh the browser tab, clearing frontend state.
- **Fix:** Added a custom websocket blocker utility script at the top of `index.html` to intercept and drop Live Server's websocket reload signals.
