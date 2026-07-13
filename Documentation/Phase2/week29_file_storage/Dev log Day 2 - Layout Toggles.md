---
date: 2026-07-13
project: Project 52
week: 29
module: File Upload & Storage System
topic: Day 2 - Layout Toggles & Integration
author: Paul Vincent Aguirre
status: Complete
Tags:
  - "[[Design]]"
  - "[[Integration]]"
  - "[[State Management]]"
  - "[[Dev Log]]"
---

# 📝 DEV LOG: WEEK 29 - DAY 2 

## 1. Goal of the Day
Integrate the Flask backend server and the vanilla JS frontend client. Build a layout switching system that allows users to toggle between a visual Grid gallery and a compact List details view, persisting user view preferences.

---

## 2. New Modules & Features

### 🎛️ Grid/List Layout Switcher Controls
- **Goal:** Enable users to toggle between visual card galleries and metadata-dense text lists.
- **Implementation:** Added Grid/List switcher buttons to the toolbar in `index.html`. Added CSS selectors for `.layout-toggles` inside `base.css` to group and style the new buttons.
- **Commit:** `feat(week29): add Grid/List layout toggle buttons to toolbar`
- **Commit:** `feat(week29): style Grid/List toggle buttons in base.css`

### 📝 Compact List View CSS Layouts
- **Goal:** Design a clean, responsive row-based alternative to the card grid.
- **Implementation:** Added `.file-list` rules to `gallery.css`. Reorganized cards to align items horizontally. The container displays files in rows showing small media icons, title strings, size capsules, relative dates, and action controls side-by-side.
- **Commit:** `feat(week29): add CSS layout styles for compact list view in gallery.css`

### 🔄 Layout State Routing & LocalStorage Persistence
- **Goal:** Track active layout toggles on the client and preserve choice between page reloads.
- **Implementation:** Initialized `currentLayout` state in `main.js` from `localStorage` values. Linked button action event listeners to update local state, synchronize storage caches, and pass layout parameters into the rendering pipeline. Updated `dashboard.js` to toggle classes on the gallery wrapper.
- **Commit:** `feat(week29): wire Grid/List state toggles in main.js routing`
- **Commit:** `feat(week29): pass layout state parameter to renderDashboard container classes`

---

## 3. Verification Check
- The Flask backend is running on `http://127.0.0.1:5000`.
- All **20 unit tests** pass:
  ```bash
  ============================= 20 passed in 2.62s ==============================
  ```
