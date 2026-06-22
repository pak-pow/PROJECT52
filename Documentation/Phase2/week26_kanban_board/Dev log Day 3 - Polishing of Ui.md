---
date: 2026-06-22
project: Project 52
week: 26
module: Kanban Board (Trello Clone)
topic: Day 3 - Instant UI Updates, Column Dragging & visual Polish
Tags:
  - "[[Frontend Architecture]]"
  - "[[Drag and Drop]]"
  - "[[Optimistic UI]]"
  - "[[CSS Variables]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 26 - DAY 3 
## 1. Goal of the Sprint
The focus for Day 3 was making the workspace feel much smoother, adding the ability to edit boards from the main menu, allowing columns to be dragged left and right, and building a fail-safe system that handles server connection drops gracefully without disrupting the user.

## 2. Advanced Drag-and-Drop (Columns & Tasks)
* **Moving Columns:** Upgraded our dragging system so you can now slide entire columns left and right to rearrange the board layout.
* **Smart Drag Handles:** Set up special mouse settings so that a column is only draggable when you grab its top title bar. This prevents columns from accidentally sliding around when you are just trying to move individual tasks inside them.

## 3. Instant Screen Updates & Going Back on Errors
* **The Problem:** Most websites make you wait for the server to reply "success" before moving items on the screen, which creates lag and makes the app feel slow.
* **The Solution:** We set up the screen to update instantly. When you drop a card or a column, it moves immediately. The save message is sent to the server in the background.
* **The Fail-Safe Rollback:** Built a memory tracker into the layout. If the server is offline or fails to save the move, the interface catches the error, cancels the move, and automatically snaps the card or column back to its original slot without needing a page reload.

## 4. UI Polish & Smart Styling
* **Board Editing:** Built a pop-up form on the dashboard triggered by a pencil icon. You can now rename boards, write descriptions, and choose new color swatches, which save instantly to the database.
* **Toast Notification System:** Got rid of annoying browser pop-up alerts. Built a custom notification banner that slides in from the top-right of the screen to show quick, non-blocking success messages or error warnings.
* **Dynamic Hover Borders:** Solved a styling problem on the dashboard. Instead of writing complex tracking code to change colors when you hover over a board card, we injected the board's custom color directly into the card's inline style tag. The styling sheet reads this variable automatically, allowing the highlight border to match the board's color instantly when hovered.

**Result:** The application builds successfully without any errors. The core Kanban board is fully operational with active database saving, color-changing themes, and smart dragging mechanics. We are now ready to proceed to Day 4: adding dashboard board reordering and dynamic search filters!
