---
date: 2026-06-13
project: Project 52
week: 25
module: Recipe Sharing Platform (Full Stack)
topic: Day 1 - Scaffold & Upload Architecture
Tags:
  - "[[Backend Architecture]]"
  - "[[File Uploads]]"
  - "[[Security]]"
  - "[[Dev Log]]"
---

# 📝 DEV LOG: WEEK 25 - DAY 1

## 1. Goal of the Day
Scaffold the Project 52 blueprint structure for a Recipe Sharing application and establish the backend architectural rules for securely handling user-generated image uploads (`multipart/form-data`).

## 2. Security & Upload Configurations
* **Directory Isolation:** Created a dedicated `backend/uploads/` directory, keeping dynamic user media strictly separated from application logic and database files.
* **Payload Limits (`MAX_CONTENT_LENGTH`):** Configured the Flask application to mathematically reject any request payload exceeding 5MB, mitigating the risk of Denial of Service (DoS) via massive file uploads.
* **Extension Whitelisting:** Established a strict `ALLOWED_EXTENSIONS` set (`png`, `jpg`, `jpeg`, `webp`). This prevents executable scripts (`.py`, `.sh`, `.exe`) from bypassing the image upload router.

## 3. Database Architecture (`schema.sql`)
* Drafted the `recipes` table to handle complex user content.
* **Storage Strategy:** Instead of saving raw binary blobs into the SQLite database (which degrades performance), the schema is designed to save only the sanitized `image_filename`. The actual file resides in the local filesystem, allowing the web server to route it efficiently.
