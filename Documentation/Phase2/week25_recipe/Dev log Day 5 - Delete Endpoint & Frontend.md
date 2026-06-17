---
date: 2026-06-17
project: Project 52
week: 25
module: Recipe Sharing Platform (Full Stack)
topic: Day 5 - Backend Finale & Frontend Ignition
Tags:
  - "[[API Integration]]"
  - "[[Frontend Architecture]]"
  - "[[Data Deletion]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 25 - DAY 5 

## 1. Goal of the Day
The objective was to finalize the backend architecture completely and ignite the frontend client. This required building the final CRUD endpoints (Read Single, Delete) with secure file system cleanup, scaffolding the modular HTML/CSS structure, and establishing the JavaScript fetch layer to pull and render the seeded database records.

## 2. Backend Finalization (`recipe_routes.py`)
* **Single View Endpoint:** Built `GET /api/recipes/<id>` to allow specific recipe querying.
* **Cascading Deletion Protocol:** Engineered `DELETE /api/recipes/<id>`. Implemented critical system architecture: before a database record is dropped, the `os` module executes a file system check and physically removes the associated image from the `uploads/` directory. This prevents orphan files and long-term hard drive bloat.

## 3. Frontend Ignition & Architecture
* **Strict Separation of Concerns:** Scaffolded `index.html` ensuring zero inline styling. Built a modular CSS pipeline (`variables.css`, `base.css`, `layout.css`, `components.css`) to enforce scalable design tokens and UI consistency.
* **The API Engine (`recipe_api.js`):** Built isolated asynchronous fetch wrappers to handle data extraction. Prepared the `createRecipe` function to accept `FormData` objects natively, allowing the browser to automatically handle multipart boundary headers.
* **Dynamic DOM Manipulation (`main.js`):** Executed the retrieval logic on page load. Mapped the JSON payload into dynamic HTML cards. Implemented dynamic event delegation to attach `DELETE` fetch triggers to individual generated buttons.

