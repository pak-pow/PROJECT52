---
date: 2026-07-01
project: Project 52
week: 27
module: Portfolio v2 (Full Stack Upgrade)
topic: Day 6 - Stacking Toasts & Project Search
author: Paul Vincent Aguirre
status: Complete
Tags:
  - "[[Flask]]"
  - "[[JavaScript]]"
  - "[[Dev Log]]"
  - "[[User Experience]]"
  - "[[Stacking Toasts]]"
  - "[[Admin Search]]"
---

# 📝 DEV LOG: WEEK 27 - DAY 6 

## 1. Goal of the Day
The focus for Day 6 was to complete our administrative dashboard and messaging layout polish. We aimed to implement a floating, stacking notification alert queue in the bottom-right corner of the page (Option 1) and add a live search bar and status filters for projects inside the admin dashboard (Option 3).

---

## 2. Stacking Toast Alerts Container
We replaced static stationary toast popups with a modern stacking alert queue:

* **Markup Updates:** Replaced `#admin-toast` in `admin.html` and `#contact-toast` in `index.html` with `<div id="toast-container" class="toast-container"></div>` elements.
* **Layout Design:** Styled `.toast-container` to float at the bottom-right corner using `position: fixed` with a vertical flex layout. Individual alert cards slide in using a subtle scaling transition.
* **Dynamic Creation:** Rewrote `showToast()` in both `admin.js` and `contact.js` to dynamically instantiate `.toast-card` alert elements with small close buttons (×). When multiple triggers are clicked, cards stack vertically and fade out after 3.5 seconds.
* **Commits:**
  - `feat(week27): replace contact toast with stacking toast container in index.html`
  - `style(week27): add stacking toast container and filter select dropdown styles`
  - `feat(week27): implement public page stacking toasts in contact.js`

---

## 3. Admin Project Search & Filter
We added dynamic searching and status filters to the projects tab of the admin panel:

* **Projects Search Bar:** Placed an input `#proj-search-input` and status select `#proj-filter-status` in the projects tab header of `admin.html`.
* **Keystroke Listening:** Configured event listeners in `admin.js` to update state variables `currentProjSearch` and `currentProjStatus` in real-time.
* **Project Row Filtering:** Updated `renderAdminProjects()` to filter list entries by matching title, tech stack, and description strings, showing custom empty fallbacks if there are no search hits.
* **Commits:**
  - `feat(week27): add project search filters and toast container to admin.html`
  - `feat(week27): implement admin project search filtering and stacking toasts in admin.js`

---

## 4. Verification
* **Test Suite Check:** Verified that all 44 backend pytest test files compile and pass successfully.
* **Manual Verification:** Checked that sorting projects updates list items cleanly, stacking alert popups dynamically as clicks are rapidly triggered, and searching/filtering operations update instantly in the dashboard view.