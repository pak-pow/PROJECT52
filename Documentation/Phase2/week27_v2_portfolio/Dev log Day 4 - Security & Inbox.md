---
date: 2026-06-29
project: Project 52
week: 27
module: Portfolio v2 (Full Stack Upgrade)
topic: Day 4 - Security Hardening & Inbox Filtering
author: Paul Vincent Aguirre
status: Complete
Tags:
  - "[[Flask]]"
  - "[[Security]]"
  - "[[Dev Log]]"
  - "[[User Experience]]"
---

# 📝 DEV LOG: WEEK 27 - DAY 4 

## 1. Goal of the Day
The focus for Day 4 was to address key security concerns discovered during our project audit (Option A) and add administrative searching and filtering to the messages inbox (Option B). 

---

## 2. Option A: Security & Session Expiration
To harden the authentication flow, we implemented safe password hashing and token expiration rules:

* **Dynamic Database Schema Migrations:** Added an `expires_at` timestamp column to the `admin_sessions` table in `schema.sql`. Added a migration handler inside `app/db.py` executing `ALTER TABLE admin_sessions ADD COLUMN expires_at TIMESTAMP;` in a try-except block to upgrade existing databases dynamically.
* **Secure Password Hashing:** Configured `config.py` to import `generate_password_hash` from `werkzeug.security` and define `ADMIN_PASSWORD_HASH` (falling back to the hashed default password). Updated `admin_routes.py` to verify passwords using `check_password_hash` instead of plaintext matching.
* **Token Expiration Guards:** Updated `admin_routes.py` to record `expires_at` (now + 2 hours) on login. Upgraded the `admin_required` middleware to check `expires_at` and automatically purge expired tokens from the database.
* **Flexible Parser Compatibility:** Coded the parsing block to read both string timestamps (normalizing spaces to `T` for parser compatibility) and native python `datetime` objects.

---

## 3. Option B: Inbox Search & Filtering
To improve the usability of the administrative dashboard, we added a highly responsive search and filtering capability to the messages inbox:

* **Frontend HTML Elements:** Added a glassmorphic `#msg-search-input` search bar and `#msg-filter-pills` (All / Unread / Read) to the Messages tab.
* **Styling & Layout:** Configured the controls to align side-by-side using CSS flex layouts, and added styling rules inside the media query to stack them vertically on mobile devices.
* **Search and Filter Logic:** Added input and click event listeners in `admin.js` to track `currentMsgSearch` and `currentMsgFilter` states. Updated `renderMessages()` to filter messages in real-time, showing dynamic fallback states ("🔍 No matching messages found.") if zero results are returned.

---

## 4. Verification & Testing
* **Test Suite Upgrades:** Added the test case `test_expired_token_returns_401` inside `tests/test_admin_routes.py` to verify token revocation.
* **All Tests Passing:** Ran pytest, verifying all 40 out of 40 tests pass cleanly.
* **Git Commits:** Committed Option A changes and each step of Option B cleanly to the repository.