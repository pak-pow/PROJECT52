---
date: 2026-06-30
project: Project 52
week: 27
module: Portfolio v2 (Full Stack Upgrade)
topic: Day 5 - Project Reordering & Validation Polish
author: Paul Vincent Aguirre
status: Complete
Tags:
  - "[[Flask]]"
  - "[[JavaScript]]"
  - "[[Dev Log]]"
  - "[[User Experience]]"
  - "[[Form Validation]]"
  - "[[Project Ordering]]"
---

# DEV LOG: WEEK 27 - DAY 5 

## 1. Goal of the Day
The focus for Day 5 was to implement interactive sorting order controls for projects inside the admin dashboard, and add real-time visual validation feedback to public and admin forms.

---

## 2. Project Reordering Endpoint
We added backend capabilities to let administrators swap sorting orders dynamically:

* **Reorder Endpoint:** Added a `POST /api/projects/<id>/reorder` endpoint inside `projects_routes.py` that receives a direction (`up` or `down`), finds the preceding or succeeding project in the list, and swaps their database `sort_order` properties.
* **Commits:** Staged and committed after modifications: `feat(week27): add project sorting reorder endpoint to projects_routes.py`.

---

## 3. Styling & Reordering UI
We designed the buttons and layout properties for project reordering:

* **Visual Arrow Buttons:** In `admin.js`, added Up (▲) and Down (▼) arrow buttons to each rendered project row.
* **Styling Arrows:** Styled the arrow icons with hover states and glow indicators, and added stacking rules in the media query to keep the buttons aligned on tablet and mobile viewports.
* **Live Reordering Hook:** Wired click events on the arrow buttons to call the reorder endpoint, automatically reload the project list, and show a toast notification ("Order updated").
* **Commits:** Staged and committed CSS: `style(week27): style project reordering controls and form input validation glows`.
* **Commits:** Staged and committed JS: `feat(week27): implement admin form validations and up/down project sorting logic in admin.js`.

---

## 4. Live Form Input Validation
We upgraded form inputs on the contact page, admin login page, and project creation form with interactive feedback:

* **Interactive Glows:** Styled inputs to glow green (`.is-valid`) when they meet HTML5 validation constraints, and red (`.is-invalid`) if there are errors (e.g. empty fields, malformed URLs, or invalid emails).
* **Real-time Listeners:** Added listeners to trigger validation checks on `blur` (leaving the input field) and during typing (`input` events) if a field has already been marked valid/invalid.
* **Contact Page Integration:** Wired validation checks to the public contact page, preventing form submissions if inputs are incorrect.
* **Commits:** Staged and committed: `feat(week27): implement live form validation feedback for contact form in contact.js`.

---

## 5. Verification & Testing
* **Test Suite Expansion:** Added automated integration tests inside `test_projects_routes.py` (including `test_reorder_swaps_sort_orders_returns_200`, checking that swapping orders swaps sorting indices correctly in the database).
* **All Tests Passing:** Successfully ran pytest, with all 44 out of 44 tests passing.
* **Commits:** Staged and committed: `test(week27): add unit tests for project sorting reorder endpoint`.


