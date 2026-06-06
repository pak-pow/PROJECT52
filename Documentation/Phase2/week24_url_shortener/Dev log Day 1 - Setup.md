---
date: 2026-06-05
project: Project 52
week: 24
module: URL Shortener Service (Backend)
topic: Day 1 - System Architecture, Database Schema, and Base62 Encoding
author: Paul Vincent Aguirre
status: In Progress
Tags:
  - "[[System Design]]"
  - "[[Backend Architecture]]"
  - "[[Database Indexing]]"
  - "[[Algorithms]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 24 - DAY 1 

## 1. Goal of the Day
Week 24 is a pure backend engineering masterclass. The objective is to build a high-performance URL Shortener service (similar to Bit.ly). Day 1 focused entirely on establishing a strict MVC architecture, designing an optimized database schema, and writing the mathematical core of the application: the Base62 Encoder.

---

## 2. Architecture & Environment Setup
We successfully mapped the new project into our established enterprise backend boilerplate. 

* **Strict MVC Flow:** The logic is completely separated. Requests will flow from `routes` → `services` (business logic) → `models` (database queries) + `utils` (math engine).
* **SQLite & `.gitignore` Hardening:** Configured SQLite to run in WAL (Write-Ahead Logging) mode for better concurrency. Crucially, we updated the `.gitignore` to explicitly block `*.db`, `*.db-shm`, and `*.db-wal` files. This prevents local runtime memory files from polluting the GitHub repository.
* **Pathing Fix:** Identified and resolved a directory traversal bug where the database connection script was targeting the wrong root folder, ensuring `database.db` generates exactly where it belongs: inside `backend/data/`.

---

## 3. Database Design & Indexing
A URL shortener is a highly read-heavy application. The database needs to find a short code and redirect the user in milliseconds.

* **The `urls` Table:** Created `schema.sql` with columns for `id`, `original_url`, `short_code`, `clicks`, and `created_at`.
* **The Performance Key (Indexing):** We explicitly wrote `CREATE INDEX IF NOT EXISTS idx_short_code ON urls(short_code);`. Without this index, the database would have to scan every single row one-by-one to find a URL. With this index, lookups are nearly instantaneous, even with millions of rows.

---

## 4. The Engine: Base62 Encoding
To generate short codes, we avoided standard hashes (like MD5 or SHA256) because they produce strings that are too long and look ugly in URLs. 

* **The Strategy:** We built a custom Base62 mathematical algorithm in `app/utils/base62.py`. It uses 62 characters (`a-z`, `A-Z`, `0-9`). 
* **How it Works:** When a new long URL is saved to the database, it receives a unique integer ID (e.g., `10000`). The Base62 engine takes that integer, divides it by 62 repeatedly, and maps the remainders to our character set, transforming `10000` into a short, unique string like `2Bi`. Because every database ID is unique, every short code is guaranteed to be 100% unique—zero collisions.
