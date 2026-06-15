---
date: 2026-06-15
project: Project 52
week: 25
module: Recipe Sharing Platform (Full Stack)
topic: Day 3 - Multipart Form Data & File System Integration
Tags:
  - "[[Flask Blueprints]]"
  - "[[Werkzeug]]"
  - "[[File Uploads]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 25 - DAY 3

## 1. Goal of the Day
The objective was to build the API routing layer to handle `multipart/form-data`, securely process physical file uploads, and save the corresponding text data to the SQLite database.

## 2. File Upload Security Protocol
* **Werkzeug Sanitization:** Implemented `secure_filename()` from Werkzeug to strip malicious directory traversal characters (e.g., `../`) from user-uploaded filenames.
* **UUID Collision Prevention:** Engineered a unique filename generator utilizing `uuid.uuid4().hex`. Prepending a cryptographic string to the filename ensures that if multiple users upload a file named `image.png`, the server will not overwrite existing files in the `uploads/` directory.

## 3. Database Architecture Integration
* Built `recipe_model.py` using parameterized SQLite queries to prevent SQL injection.
* **Decoupled Storage:** Successfully decoupled the file storage from the database. The physical image file is saved directly to the server's filesystem (`backend/uploads/`), while only the sanitized `image_filename` string is stored in the `recipes` database table. This dramatically improves database query performance and read speeds.

## 4. API Testing & Debugging
* **JSON vs. Form-Data Friction:** Encountered a classic file-handling block when testing via Thunder Client. Attempting to send physical files over a standard `application/json` payload resulted in a `400 Bad Request` because JSON cannot handle binary streams.
* **Resolution:** Switched the HTTP request to `multipart/form-data` via the Form tab, which successfully separated the text fields from the binary image stream. This resulted in a `201 Created` response and the physical file correctly saving to the local directory.
