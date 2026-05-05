---
date: 2026-05-05
project: Project 52 - Phase 2 (Integration)
module: Week 19 - Markdown Note-Taking App
topic: Backend File I/O, REST API Methods, and Absolute Pathing
Tags:
  - "[[Python]]"
  - "[[Flask]]"
  - "[[REST API]]"
  - "[[Debugging]]"
  - "[[Dev Log]]"
---
# 🚀 DEV LOG: WEEK 19, DAY 2

## 1. Executive Summary
Day 2 focused entirely on the backend data layer. The objective was to build a complete CRUD (Create, Read, Update, Delete) engine capable of manipulating physical `.md` files on the local hard drive. By the end of the session, the Python Flask API was fully functional and verified via direct REST Client testing, completely independent of the frontend UI.

## 2. Core Logic: Mapping CRUD to HTTP Methods
We built a standard RESTful interface to handle file operations, mapping Python `os` and `open()` commands to specific HTTP methods:

*   **READ (`GET /api/notes/<filename>`):** Utilized the `<filename>` URL parameter to locate specific files. Opened the file in `'r'` (read) mode and returned the raw string payload to the client.
*   **CREATE / UPDATE (`POST /api/notes`):** Accepted JSON payloads (`request.get_json()`) containing the desired filename and markdown content. Opened the file in `'w'` (write) mode, which natively handles both creating new files and overwriting existing ones.
*   **DELETE (`DELETE /api/notes/<filename>`):** Implemented `os.remove(filepath)` to permanently delete files from the disk, complete with defensive checks (`os.path.exists()`) to prevent crashing on non-existent files.

*Note: All file operations explicitly declared `encoding='utf-8'` to prevent system crashes when handling special characters or emojis.*

## 3. The Pathing Bug: Absolute vs. Relative Paths
During testing, a critical bug emerged: a `404 File Not Found` error, despite the file visibly existing in the VS Code file tree.
*   **The Cause:** Python relied on the Current Working Directory (CWD) of the terminal (`DATA_DIR = "data"`). Because the terminal was running from the root `PROJECT52` workspace, Python was looking for a `data` folder at the top level, not inside `/backend`.
*   **The Fix:** Migrated from relative pathing to bulletproof Absolute Pathing. 

``` python
BASE_DIR = os.path.dirname(os.path.abspath(__file__))
DATA_DIR = os.path.join(BASE_DIR, "data")
```

This forces Python to determine the exact location of `app.py` on the hard drive and securely anchor the `data/` directory relative to that specific file, rendering the terminal's execution context irrelevant.

## 4. The Pygame Bridge (Mental Model)
This backend system operates exactly like the `SaveSystem` class in the Pygame 3D engine.

- `GET` is `SaveManager.load_game()`
- `POST` is `SaveManager.write_save()`
- `DELETE` is `SaveManager.corrupt_save_wipe()`

The only difference is that instead of the Pygame game loop calling these functions directly, we are exposing them to the local network so a web browser can pull the trigger.

## 5. API Verification
The backend was rigorously tested using the VS Code REST Client (Thunder Client). We successfully verified payload transmission, file generation on the hard drive, and file deletion without writing a single line of frontend JavaScript. The backend engine is now sealed.

