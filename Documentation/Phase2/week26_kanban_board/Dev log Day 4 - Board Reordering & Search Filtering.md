---
date: 2026-06-23
project: Project 52
week: 26
module: Kanban Board (Trello Clone)
topic: Day 4 - Dashboard Board Reordering & Quick Search/Filter
Tags:
  - "[[Dashboard Design]]"
  - "[[Drag and Drop]]"
  - "[[Search Filter]]"
  - "[[Dev Log]]"
---

# DEV LOG: WEEK 26 - DAY 4 (BOARD REORDERING & SEARCH FILTERING)

## 1. Goal of the Day
The focus for Day 4 was to give users more control over their workspace by letting them drag and rearrange board cards on the main dashboard. We also wanted to make it much easier to find items by adding a live search filter to both the dashboard and individual board screens.

---

## 2. Reorganizing Workspace Boards
Previously, boards were shown in the order they were created. We upgraded this to support custom layout arrangements using drag-and-drop:

* **Accent Drag Handles:** We added a small vertical grip dots icon next to the title on each board card. To prevent accidental page clicks when dragging, you can only pick up and drag a card by grabbing this handle.
* **Grid-Aware Drag Proximity:** Because dashboard boards are arranged in rows that wrap automatically to fit the screen, we updated the drag code to calculate distances. The drag engine looks at the mouse position relative to the center of nearby cards to figure out exactly where the dragged card should land.
* **Optimistic Sorting & Database Saving:** When you drop a board, the screen immediately indexes all cards to match their new positions and sends the new layout order to the server. If the server is offline, the card automatically slides back to where it was, and a red error banner pops up.

---

## 3. Quick Search and Filtering
To make it easy to manage large collections of projects and tasks, we built an instant search feature:

* **Dashboard Filtering:** A search bar in the workspace header lets you search for boards. As you type, the cards list instantly filters, leaving only boards whose titles or descriptions match your query.
* **Card Canvas Filtering:** Inside a board, a search bar in the navigation bar lets you filter tasks. Cards that do not match the query fade out and shrink, collapsing the empty space smoothly.
* **Drag-Safety Lock:** To prevent indexing bugs, we disable board, column, and card dragging while a search query is active. This keeps positions safe and consistent.

---

## 4. UI Cleanups
To keep the application looking premium:
* We removed all remaining default browser popups (`alert` popups) and replaced them with smooth success and error toast notifications.