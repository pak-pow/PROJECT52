---
date: 2026-07-19
project: Social Media Feed
topic: Project Scaffold, Database Schema & Full Backend + Frontend Foundation
Tags:
  - "[[Flask]]"
  - "[[SQLite]]"
  - "[[Python]]"
  - "[[JavaScript]]"
  - "[[Full Stack]]"
  - "[[Dev Log]]"
---

# 📝 DEV LOG: WEEK 30 - DAY 1

**Core Objective:** Scaffold the full project structure for the Week 30 Social Media Feed — a Twitter/X-style platform — including the database schema, Flask app factory, all backend models, routes, services, and frontend skeleton.

## 1. The Initiative & Context
Week 30 marks the start of the complex project in Phase 2. The goal is to build a Twitter/X-style social feed with posts, likes, replies, reposts, follows, and infinite scroll. Day 1 was planned as scaffold-only (folder structure, schema, app factory). Due to session momentum, the equivalent of Days 1–4 was completed in a single session — the full backend was built and wired, and the frontend skeleton was established.

## 2. Architectural Decisions & Concepts
### Concept A: Five-Table Relational Schema
The database is built around five SQLite tables: `users`, `sessions`, `posts`, `likes`, and `follows`. Key design decisions include:

- `likes` uses a compound `PRIMARY KEY (user_id, post_id)` — making duplicate likes a database-level impossibility rather than something handled in code.
- `follows` uses `CHECK(follower_id != following_id)` directly in the schema to prevent self-follows at the lowest level.
- `posts` supports replies and reposts via self-referencing foreign keys (`reply_to_id`, `repost_of_id`) — meaning the same table handles all three post types cleanly.
- WAL journal mode is enabled for better concurrent read performance.

### Concept B: Flask App Factory Pattern
The Flask app is initialized through a `create_app()` factory function rather than a global `app` object. This allows test fixtures to pass isolated configuration (like a temporary SQLite database path) without touching the real database — a pattern critical for reliable testing.

### Concept C: Optimistic Like UI
The like button on the frontend updates instantly on click (toggling the heart icon and count) without waiting for the API response. If the server call fails, the UI rolls back. This is the standard UX pattern used by real social platforms to eliminate perceived latency.

## 3. Project Structure Created

```
week30_social_feed/
├── backend/
│   ├── app/
│   │   ├── __init__.py              ← Flask app factory
│   │   ├── db.py                    ← SQLite connection helper (WAL mode, Row factory)
│   │   ├── config/
│   │   │   ├── __init__.py
│   │   │   └── settings.py          ← Config class (env vars, upload limits, paths)
│   │   ├── models/
│   │   │   ├── __init__.py
│   │   │   ├── user_model.py        ← create, get, verify, update user
│   │   │   ├── session_model.py     ← 64-char UUID bearer token sessions
│   │   │   ├── post_model.py        ← home feed, explore, user posts, replies, delete
│   │   │   ├── like_model.py        ← toggle like, bulk liked-set lookup
│   │   │   └── follow_model.py      ← follow/unfollow, counts, follower/following lists
│   │   ├── routes/
│   │   │   ├── __init__.py
│   │   │   ├── auth_routes.py       ← /api/auth/*
│   │   │   ├── post_routes.py       ← /api/posts/*
│   │   │   ├── user_routes.py       ← /api/users/*
│   │   │   └── health_routes.py     ← /api/health
│   │   └── services/
│   │       ├── __init__.py
│   │       ├── auth_service.py      ← require_auth / optional_auth decorators
│   │       ├── serializers.py       ← SQLite Row → JSON dict helpers
│   │       └── image_service.py     ← Pillow resize/crop for avatars and post images
│   ├── data/
│   │   ├── schema.sql               ← Full DB schema with indexes and constraints
│   │   └── seed.py                  ← Demo users, follows, posts, replies, likes
│   ├── tests/
│   │   ├── __init__.py
│   │   ├── conftest.py              ← Isolated temp DB fixture per test
│   │   ├── test_auth_routes.py      ← 9 auth test cases
│   │   └── test_post_routes.py      ← 9 post test cases
│   ├── run.py                       ← Development server entrypoint
│   ├── wsgi.py                      ← Waitress production WSGI server
│   └── requirements.txt
└── frontend/
    ├── public/
    │   └── index.html               ← Full app shell HTML
    └── src/
        ├── api/
        │   ├── authApi.js           ← register, login, logout, me, fetchAuth guard
        │   ├── postApi.js           ← feed, explore, create, delete, like
        │   └── userApi.js           ← profile, posts, follow, update, avatar
        ├── assets/
        │   ├── base.css             ← Design system, layout, tokens, skeletons
        │   ├── auth.css             ← Login/register card and form styles
        │   ├── feed.css             ← Inline compose bar styles
        │   ├── profile.css          ← Profile header, banner, follow button
        │   └── compose.css          ← Compose modal styles
        ├── utils/
        │   └── helpers.js           ← relativeTime, escapeHtml, linkify, toast, navigate
        └── main.js                  ← Auth flow, routing, feed, compose, optimistic like
```

## 4. Debugging: Test DB Path Override
- **Issue:** All 18 pytest tests were failing with `sqlite3.OperationalError: unable to open database file`.
- **Root Cause:** `get_db()` was reading `Config.DB_PATH` as a static class attribute at import time. When pytest fixtures passed a temporary database path via `test_config`, the running code ignored it and kept trying to open the real production database path, which didn't exist in the test environment.
- **Resolution:** Modified `get_db()` to read the path from `current_app.config` at runtime inside the Flask application context, falling back to `Config.DB_PATH` only when running outside a Flask context. All 18 tests passed after the fix.

## 5. The Output & Result
- ✅ Full Flask backend — 15+ API endpoints across auth, posts, and users
- ✅ 5-table SQLite schema with indexes, compound PKs, and CHECK constraints
- ✅ 18 pytest tests — all passing in 2.34s
- ✅ Frontend shell — auth screen, sidebar navigation, home feed, compose bar, and CSS design system
- ✅ 18 git commits, each scoped to a single file or layer

---
