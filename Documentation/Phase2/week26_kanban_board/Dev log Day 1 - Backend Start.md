---
date: 2026-06-20
project: Project 52
week: 26
module: Kanban Board (Trello Clone)
topic: Day 1 - Database & Backend Server Setup
author: Paul Vincent Aguirre
status: Complete
Tags:
  - "[[System Architecture]]"
  - "[[API Design]]"
  - "[[Unit Testing]]"
  - "[[Dev Log]]"
---

# 📝 DEV LOG: WEEK 26 - DAY 1 (KANBAN BACKEND ENGINE)

## 1. Goal of the Sprint
The goal for Day 1 was to build a rock-solid foundation for our Kanban board. Because Kanban boards rely on deeply connected data (Cards live inside Columns, and Columns live inside Boards), we focused on making sure our database connects everything correctly, rejects bad data, and is fully prepared to handle complex actions like dragging and dropping cards. 

## 2. Database & Server Setup
* **Organized Project Structure:** Set up a clean folder system that separates our data handlers, our validation rules, and our web addresses (routes). We also locked in the necessary Python packages.
* **Smart Data Cleanup:** Engineered the database so that everything is strictly linked. If a user deletes a Board, the system automatically wipes out all the Columns and Cards attached to it, preventing leftover "orphan" data.
* **Connection & Security Management:** 
    * Ensured the database safely closes its connection after every request to prevent memory issues.
    * Added a tracking system that logs every request in the terminal (showing what was asked for, and how many milliseconds it took to reply).
    * Updated our security settings so that our upcoming web frontend can smoothly talk to this backend server without getting blocked.

## 3. The Backend Layers
* **Data Handlers:** Built dedicated Python files to handle saving and retrieving Boards, Columns, and Cards. When we create or update an item, the system now returns the fully updated record, including its exact position and the time it was created.
* **Rules & Validation:** Created a "bouncer" layer to check all incoming requests before they reach the database. 
    * It blocks invalid actions, like trying to create a board without a name or with a fake color code.
    * It safely handles missing information (fixing a bug where empty task descriptions would crash the server).
    * It automatically bundles data together—so when you ask for a Board, it grabs all the matching Columns and Cards and packages them into one clean response.

## 4. Web Routes & Drag-and-Drop Prep
* Built 17 total web addresses (endpoints) to handle all aspects of creating, reading, updating, and deleting data.
* Ensured we can fetch specific items on their own (like getting all columns for a specific board, or pulling up a single card's details).
* Added strict "404 Not Found" guards everywhere, preventing the server from trying to update or delete records that don't exist.
* Created specialized routes designed specifically for our upcoming frontend drag-and-drop interface. These endpoints (`/reorder` and `/move`) allow the system to instantly update the exact order of cards as they are dragged between different columns.

## 5. Automated Testing & Dummy Data
* **Safe Dummy Data:** Created a script to generate sample boards and tasks so we have something to look at while building the frontend. The script is smart enough to check if data already exists, so it doesn't accidentally create duplicates if run twice.
* **Massive Test Expansion:** Expanded our automated testing system from 4 tests to **89 tests**. We set up a dedicated testing environment that spins up a fresh, temporary database for every test. This proves that all 17 of our routes actively reject bad data, handle missing items properly, enforce automatic deletions, and successfully process drag-and-drop position updates.
