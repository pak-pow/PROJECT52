---
date: 2026-05-30
project: Project 52 - Week 23
module: Expense Tracker with Charts
topic: Enterprise Blueprint & Database Schema Architecture
Tags:
  - "[[Software Engineering]]"
  - "[[System Architecture]]"
  - "[[Application Factory]]"
  - "[[Database Design]]"
  - "[[Dev Log]]"
---

# DEV LOG: WEEK 23 - DAY 1 

## 1. Executive Summary
Week 23 marks a significant paradigm shift in Project 52. Moving away from the content-heavy CMS of Week 22, this week requires a heavily mathematical, data-driven backend to power an Expense Tracker. The primary objective for Day 1 was abandoning monolithic file structures (`app.py` holding everything) in favor of an industry-standard, deeply decoupled enterprise blueprint. 

## 2. Enterprise Architecture Scaffold (The "Why")
To build a scalable application, I instituted a strict directory hierarchy. This prevents "spaghetti code" and adheres to the **Separation of Concerns (SoC)** principle.

* **`app/routes/` (The Traffic Cops):** Files here are only allowed to handle HTTP requests (GET, POST), extract JSON payloads, and return HTTP status codes. They do *no* business logic.
* **`app/models/` (The Bouncers):** Files here hold the exact SQL queries. They are the only files legally allowed to talk to the SQLite database. 
* **`app/utils/` (The Mechanics):** Shared helper functions, such as database connection lifecycles and authentication wrappers, ensuring DRY (Don't Repeat Yourself) code.
* **The "Routing -> Service -> Model" Pipeline:** By forcing data through this strict pipeline, if a database bug occurs, I know exactly which folder to look in without reading through hundreds of lines of routing logic.

## 3. Financial Database Schema Design
Financial applications are notoriously fragile if the schema is not designed with absolute precision. I designed the `schema.sql` to include an `expenses` table linked via a Foreign Key to the `users` table.

* **The Floating-Point Trap:** The most critical architectural decision today was using the `DECIMAL(10, 2)` data type for the `amount` column instead of a standard `FLOAT`. 
* **The Rationale:** Standard floats suffer from IEEE 754 precision errors (e.g., where `0.1 + 0.2` equals `0.30000000000000004`). If an expense tracker uses floats, financial totals will eventually drift, destroying data integrity. `DECIMAL` forces exact math.

## 4. The Application Factory Pattern
Instead of declaring a global `app = Flask(__name__)`, I implemented the Application Factory pattern via `def create_app():` inside `__init__.py`. 
* **Why it matters:** This delays the creation of the application until the server explicitly runs it. This is a mandatory pattern for professional testing (like the `pytest` suite we built last week), as it allows us to spin up multiple isolated "dummy" apps in memory without them colliding.