---
date: 2026-07-14
project: Project 52
week: 29
module: File Upload & Storage System
topic: Day 3 - Client Validation, Image Testing & Seeding
author: Paul Vincent Aguirre
status: Complete
Tags:
  - "[[Validation]]"
  - "[[Seeding]]"
  - "[[Testing]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 29 - DAY 3 

## 1. Goal of the Day
Implement client-side constraints in the upload dropzone to instantly validate file sizes and types, add interactive loading animations to submission forms, expand backend unit testing with Pillow image thumbnail verification cases, and write a database seeder that populates the repository with physical mock file attachments on initialization.

---

## 2. New Modules & Features

### 🛡️ Client-side Size and MIME Whitelist Filters
- **Goal:** Instantly reject invalid uploads (e.g. files >10 MB or unsupported extensions) in the browser, saving user bandwidth.
- **Implementation:** Added `ALLOWED_MIME_TYPES` and `MAX_FILE_SIZE` definitions inside `dropzone.js`. Separated files in drop queues into valid and invalid lists. Visualized invalid elements with warning icons and descriptive error text.
- **Commit:** `feat(week29): add client-side size and MIME validation to dropzone`

### 🔄 Interactive Form Button Indicators
- **Goal:** Prevent multiple form submissions when logging in or registering.
- **Implementation:** Updated the `handleAuth` controller in `main.js` to change button text (e.g., "Logging In…" or "Registering…") and disable clicks until api responses resolve, reverting them within a `finally` handler.
- **Commit:** `feat(week29): add interactive loading states to login and register buttons`

### 🌱 Database & Storage Mockup Seeder
- **Goal:** Bootstrap the application on start with sample database rows and physical file attachments.
- **Implementation:** Developed `backend/data/seed.py`. Wrote script rules that draw a welcome poster PNG via Pillow, write simple text documents, and place mock MP3 files inside the workspace upload directory. Integrated database row indexing and thumbnail creation.
- **Commit:** `feat(week29): add relational and physical mockup seeder script`

### 🖼️ Image Upload & Resizing Unit Tests
- **Goal:** Expand automated test coverage to verify image processing pipelines.
- **Implementation:** Added `test_upload_image_generates_thumbnail` to `tests/test_file_routes.py`. The test case constructs a tiny PNG in memory, posts it to the uploads endpoint, and verifies that a thumbnail is generated and served correctly.
- **Commit:** `test(week29): add unit test for image upload and thumbnail generation`

---
