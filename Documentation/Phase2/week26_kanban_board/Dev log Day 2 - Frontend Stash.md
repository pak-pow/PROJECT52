---
date: 2026-06-21
project: Project 52
week: 26
module: Kanban Board (Trello Clone)
topic: Day 2 - Frontend SPA, Drag & Drop, & State Sync
Tags:
  - "[[Frontend Architecture]]"
  - "[[Drag and Drop]]"
  - "[[UI/UX]]"
  - "[[State Management]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 26 - DAY 2 

## 1. Goal of the Sprint
The objective for Day 2 was to build a responsive, single-page application (SPA) frontend that consumes the Flask API built on Day 1. The primary technical challenge was engineering a native HTML5 Drag-and-Drop interface capable of moving cards between columns, visually updating the DOM, and synchronizing those positional changes with the SQLite database.

## 2. Design System & CSS Architecture
* **Dark Mode Tokens:** Established a premium dark-theme design system in `variables.css`, defining background palettes (zinc/slate), typography scales, soft shadows, and fast transition curves.
* **Dynamic Theming:** Engineered the Board Canvas view to dynamically intercept the board's `accent_color` from the API and override the global `--accent` CSS variable. This allows the UI (buttons, borders, highlights) to change colors based on the specific board the user opens.
* **Layouts & Glassmorphism:** Built horizontal flex-scroll containers in `layout.css` to allow columns to scroll left-to-right infinitely. Styled modal overlays with translucent backgrounds and center-stage structural grids.

## 3. The API Bridge & Security
* **The Fetch Wrapper (`kanban_api.js`):** Built a comprehensive Javascript client to interface with the Flask backend. It handles all CRUD operations (Boards, Columns, Cards) and specifically intercepts `204 No Content` responses to prevent JSON parsing errors on deletion.
* **XSS Sanitization (`dom.js`):** Implemented an `escapeHtml` utility. Every piece of user-generated data (board titles, card descriptions) is scrubbed through this function before being injected into the DOM to prevent Cross-Site Scripting attacks.

## 4. SPA Architecture & Componentization
* **The Hash Router:** Avoided heavy frameworks by engineering a lightweight `#` hash router to seamlessly toggle the user between the Dashboard (`dashboard.js`) and the active Kanban Canvas (`board.js`) without reloading the page.
* **Modular UI Components:** Separated the HTML generation logic into discrete files:
    * `boardCard.js`: Handles the dashboard workspace tiles, description clipping, and delete handlers.
    * `column.js`: Renders the vertical task lists, complete with inline title editing and "Add Card" triggers.
    * `card.js`: Generates the individual task blocks, wiring up the edit/delete buttons and the `draggable="true"` HTML5 attribute.

## 5. The HTML5 Drag & Drop Engine
* **The Drag Controller (`drag.js`):** Built a custom drag-and-drop engine without relying on external libraries (like `interact.js`). 
    * **Proximity Math:** Implemented `getDragAfterElement` to calculate mouse Y-coordinates against sibling elements. This creates the "parting the sea" visual effect, where cards slide out of the way as you drag a hovering card over them.
    * **State Synchronization:** Engineered the `drop` event to calculate the new positional index of the card, determine if it crossed into a new column, and immediately fire `PATCH` requests to the API (`reorderCards` or `moveCard`) to permanently save the new layout to the database.

**Result:** The application successfully compiles (`npm run build`) with zero errors. The Trello clone is fully operational. Users can create custom-colored boards, add columns, generate tasks, and drag them seamlessly across the canvas with persistent database synchronization. Day 2 is complete.