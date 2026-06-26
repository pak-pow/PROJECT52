---
date: 2026-06-26
project: Project 52
week: 27
module: Portfolio v2 (Full Stack Upgrade)
topic: Day 1 - Frontend Implementation & Bug Fixes
author: Paul Vincent Aguirre
status: Complete
Tags:
  - "[[Flask]]"
  - "[[SQLite]]"
  - "[[JavaScript]]"
  - "[[Dev Log]]"
  - "[[REST API]]"
  - "[[Admin Dashboard]]"
---

# DEV LOG: WEEK 27 - DAY 1

## 1. Goal of the Day
The focus for Day 1 was to build the complete frontend for Portfolio v2 and wire it to the Flask backend that was already in place. The goal was to have a fully working multi-page site — home, about, and admin panel — all talking to a live SQLite database through a REST API.

---

## 2. Pages Built
We constructed three fully functional HTML pages from scratch using semantic markup and ES module JavaScript:

* **Home Page (`index.html`):** Features an animated hero section with a blue glow radial gradient, an about preview, a dynamic projects grid populated live from the database, and a full AJAX contact form with loading state and toast notifications.
* **About Page (`about.html`):** Shows an updated biography reflecting Phase 2 progress, categorised skill badges (Backend, Frontend, Data & Tools), and a three-card Phase 1–3 progress tracker for Project 52.
* **Admin Panel (`admin.html`):** A login-gated dashboard with two tabs — a Contact Inbox showing all submitted messages with mark-read/delete controls, and a Projects Manager for full CRUD operations (add, edit, delete) without touching the database directly.

---

## 3. JavaScript Architecture
We built a clean, modular JS system using native ES modules — no frameworks, no bundler:

* **`api.js` (API Client):** A single centralised module that handles all `fetch()` calls to the backend. It automatically injects the `Authorization: Bearer <token>` header on protected routes by reading from `localStorage`. This keeps authentication logic in one place and out of every other file.
* **`contact.js` (Public Page Controller):** Handles the contact form submission with full loading/error states, renders the projects grid from the API with tech tags and status badges, and runs the scroll-reveal IntersectionObserver.
* **`admin.js` (Dashboard Controller):** Manages login/logout flow, tab switching, full message CRUD (list, toggle-read, delete), and full project CRUD (create form, edit form, delete) — all wired to `api.js`.
* **`reveal.js` (Shared Utility):** A lightweight scroll-reveal module created specifically for pages that do not load `contact.js`, so they still get the same animated entrance effect.

---

## 4. CSS Design System
We built a complete design system in a single `style.css` file that upgrades the v1 look while keeping the same core identity:

* **Preserved Identity:** The same `#0a0a0a` near-black background and `#4196f3` electric blue accent from the Week 1 portfolio were kept as the foundation.
* **Premium Upgrades:** Switched from Courier New to Outfit (Google Fonts) for headings, added glassmorphism cards with `backdrop-filter: blur(12px)`, blue glow box-shadows on hover, and a pulsing radial gradient behind the hero.
* **Full Component Set:** Buttons (primary/outline/danger/small), form inputs with focus glow, skeleton loading cards with shimmer animation, toast notifications, status badges, admin tab buttons, message cards with unread indicator dot, and responsive breakpoints at 768px.

---

## 5. Backend Rename — `app.py` → `run.py`
We discovered a naming conflict that was silently breaking the test suite:

* **The Problem:** `app.py` (the entry point) and `app/` (the Flask package directory) sat at the same level. Python couldn't distinguish between them during imports, requiring an ugly `importlib` workaround in the test config.
* **The Fix:** Renamed `app.py` to `run.py`, which is the conventional Flask entry point name when using the application factory pattern. The `from run import create_app` import in `conftest.py` is now clean and obvious.
* **Rule:** The entry point should never share a name with its own package.

---

## 6. Bugs Found & Fixed
Six bugs were caught and resolved during first-load testing:

* **Projects Not Loading:** `contact.js` had a second `import` statement buried mid-file. ES module spec requires all imports at the top — the browser silently ignored the second one, so `getProjects` was never defined. Fixed by merging both imports to the top.
* **All Sections Invisible:** The scroll-reveal `.section-reveal` class sets `opacity: 0` by default. The `IntersectionObserver` only fires when elements scroll into view — not for elements already on screen when the page loads. Fixed by pre-checking `getBoundingClientRect()` and immediately adding `.revealed` to anything already visible.
* **About Page Blank:** `about.html` had no `<script>` tags at all, so `.section-reveal` kept the entire page permanently invisible. Fixed by creating `reveal.js` and adding a CSS `opacity: 1 !important` fallback on `#about-page`.
* **Admin Card Squeezed:** The login card max-width was 420px with minimal padding, making it feel cramped. Widened to 500px, increased padding to `3rem 3.5rem`, and added proper spacing between every element.
* **Redundant Script Tags:** `api.js` was being loaded both as a standalone `<script>` in the HTML and as an ES module import inside `contact.js`/`admin.js`, causing it to execute twice. Removed the standalone tags from both pages.
* **Test DB Collision:** SQLite `:memory:` databases are per-connection — each HTTP request during tests got a fresh empty database, so tables set up in the fixture vanished by the time the request handler ran. Switched to `tempfile.mkstemp()` so all connections in a test share the same physical file.

---

## 7. Test Results
All 37 automated tests pass after today's fixes:

* **Admin Auth (7 tests):** Login with correct/wrong creds, logout with valid token, token invalidation after logout
* **Admin Messages (8 tests):** Auth guard, list messages, submit and retrieve, unread default state, toggle read, delete, 404 on missing IDs
* **Contact Form (8 tests):** Valid submission, each missing field, invalid email format, empty body, non-JSON body
* **Projects Public (3 tests):** Returns 200, returns seeded v1 data, all required fields present
* **Projects Admin (11 tests):** Auth guards, create/update/delete success, missing required fields, 404 on nonexistent IDs

---

## 8. Key Lessons Learned

* **ES module imports are not flexible about position.** Placing an import mid-file is a spec violation. The browser does not throw an error — it simply ignores it, which makes the bug nearly invisible.
* **IntersectionObserver is scroll-only.** It will never fire for elements already inside the viewport at page load time. Always pair it with an initial viewport check.
* **SQLite `:memory:` is per-connection, not per-process.** In a web server context, every request is a new connection and gets a brand new empty database. Always use a temp file in tests that make HTTP requests.
* **Naming entry points the same as their package is a Python antipattern.** `run.py` is the Flask standard for a reason.

---
