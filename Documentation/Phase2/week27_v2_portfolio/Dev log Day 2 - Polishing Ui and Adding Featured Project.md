---
date: 2026-06-27
project: Project 52
week: 27
module: Portfolio v2 (Full Stack Upgrade)
topic: Day 2 - Spotlight, Filtering & UI Polish
author: Paul Vincent Aguirre
status: Complete
Tags:
  - "[[Flask]]"
  - "[[SQLite]]"
  - "[[JavaScript]]"
  - "[[Dev Log]]"
  - "[[User Experience]]"
  - "[[Animations]]"
---

# 📝 DEV LOG: WEEK 27 - DAY 2 (SPOTLIGHT, FILTERING & UI POLISH)

## 1. Goal of the Day
The focus for Day 2 was to polish and enrich the user experience of our full-stack portfolio upgrade. We wanted to implement a prominent featured project spotlight, add interactive filtering controls for technology tags and project status, build staggered entrance animations, and provide a downloadable CV on the About page.

---

## 2. Database Upgrades & Dynamic Migrations
To support designating a project as featured, we modified the backend database structure and added a safeguard mechanism to handle the schema update seamlessly:

* **Featured Column:** Added a `featured` column (defaulting to `0`) to the `projects` table definition. Seed data was updated to default `PROJECT_PYGAME` to a featured project (`1`).
* **Dynamic Table Migrations:** Updated our database initialization script in `app/db.py` to run an `ALTER TABLE projects ADD COLUMN featured INTEGER DEFAULT 0` query. Wrapping this in a try-except block and running it before executing the schema scripts ensures that existing databases receive the new column without throwing schema mismatch exceptions during inserts.

---

## 3. Backend Routing & Testing Upgrades
We updated the REST API to handle reading, creating, and updating the new featured project status:

* **Serialization Updates:** Expanded the `_project_to_dict` helper in `projects_routes.py` to serialize and return the `featured` value.
* **CRUD Integration:** Updated the `POST /api/projects` and `PUT /api/projects/<id>` handlers to accept and sanitize the `featured` value from JSON payloads.
* **Test Suite Expansion:** Added unit test coverage checking that `featured` is returned in listing responses and can be correctly set during project creation and update requests. The backend test suite runs successfully with 39 out of 39 tests passing.

---

## 4. Frontend Interactive Polish & UX Upgrades
The frontend was updated to take full advantage of the new backend fields and provide a premium user experience:

* **Featured Project Spotlight:** Built a widescreen spotlight container above the project grid. The featured project (e.g. `PROJECT_PYGAME`) is rendered here with a custom star icon and glowing border layout, and is excluded from the general grid to prevent duplicates.
* **Dynamic Tech Filters:** Scripted a dynamic filter extraction system that reads all loaded projects, counts technology tags, and generates filter pills for the top 6 technologies (e.g. Python, Flask, OpenGL).
* **Grid & Status Filtering:** Added status dropdown and tag pill filters. Clicking on any filter hides the spotlight container and filters the grid live.
* **Staggered Animations:** Card elements are rendered with inline CSS variable delays (`animation-delay: index * 75ms`), yielding a smooth sequential entrance animation when cards render.
* **CV Download Button:** Added a professional CV download action to `about.html` pointing to a local mock PDF resume file, allowing visitors to save credentials locally.

---
