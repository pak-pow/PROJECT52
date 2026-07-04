---
date: 2026-07-04
project: Project 52
week: 28
module: Quiz Platform with Leaderboards
Tags:
  - "[[Flask]]"
  - "[[SQLite]]"
  - "[[Python]]"
  - "[[Dev Log]]"
  - "[[Backend]]"
  - "[[Database]]"
---
# DEV LOG: WEEK 28 - DAY 1

## 1. Goal of the Day
Set up the entire backend boilerplate for the Quiz Platform from scratch. This includes the project config layer, the SQLite database schema, seed data for two quizzes, a database client helper, the Flask application factory, and the server entrypoint.

---

## 2. Project Folder Structure
The project follows the standard Phase 2 full-stack template:

```
week28_quiz_platform/
├── .gitignore
├── README.md
├── backend/
│   ├── .env
│   ├── requirements.txt
│   ├── run.py                    # Server entrypoint
│   ├── data/
│   │   └── schema.sql            # Tables + seed data
│   └── app/
│       ├── __init__.py           # Flask app factory
│       ├── db.py                 # Database connection helpers
│       ├── config/
│       │   └── settings.py       # Config loader from env
│       ├── routes/
│       ├── services/
│       ├── models/
│       ├── middlewares/
│       └── utils/
└── frontend/
    ├── package.json
    ├── public/
    │   └── index.html
    └── src/
        ├── main.js
        ├── pages/
        ├── components/
        ├── api/
        ├── assets/
        │   └── style.css
        └── utils/
```

---

## 3. Config Layer (`backend/app/config/settings.py`)
Created a `Config` class that reads environment variables and provides fallback defaults for local development:

- `SECRET_KEY` — Flask secret key (defaults to a dev placeholder)
- `DATABASE_PATH` — Path to SQLite file (defaults to `data/quiz.db`)
- `CORS_ORIGIN` — Allowed frontend origin (defaults to `http://localhost:5500`)
- `DEBUG` — Enabled when `FLASK_ENV=development`

**Commit:** `feat(week28): add backend config settings loader`

---

## 4. Database Schema (`backend/data/schema.sql`)
Designed three relational tables:

### `quizzes`
Stores quiz metadata (title, description, category, and countdown timer in seconds).

### `questions`
Stores individual questions linked to a quiz. Options are serialized as a JSON string (e.g. `'["Paris", "London", "Berlin"]'`). The `correct_option_index` is a 0-based integer. **The correct index is never sent to the client** — only used server-side during grading.

### `leaderboard`
Stores all-time submission records per quiz (username, score out of total questions, time taken in seconds, and timestamp).

Foreign keys on `questions` and `leaderboard` both use `ON DELETE CASCADE` so deleting a quiz automatically cleans up all related rows.

### Seed Data
Pre-loaded two quizzes with 5 questions each:
- 🖥️ **Web Development Basics** — HTML, CSS, JavaScript (60-second timer)
- 🐍 **Python Programming Trivia** — Python syntax and types (90-second timer)

> **Bug Fixed:** The initial multi-row `INSERT` syntax caused a `sqlite3.OperationalError: near ")": syntax error` when run through Python's `executescript()`. The fix was to split each row into its own individual `INSERT OR IGNORE` statement, which `executescript` handles reliably.

**Commits:**
- `feat(week28): add database schema and seed data`
- `fix(week28): fix schema.sql to use individual INSERT statements`

---

## 5. Database Client (`backend/app/db.py`)
Wrote two helper functions:

- `get_db()` — Returns a configured SQLite connection with `row_factory = sqlite3.Row` (allows column access by name) and enables foreign key enforcement via `PRAGMA foreign_keys = ON`.
- `init_db()` — Reads `schema.sql` from disk and runs it via `executescript()`. Called automatically on app startup.

**Commit:** `feat(week28): add database connection helper and init function`

---

## 6. Flask Application Factory (`backend/app/__init__.py`)
Created a `create_app()` factory function that:

1. Instantiates the Flask app and loads config from `Config`.
2. Applies CORS via `Flask-Cors`, limiting API access to the configured frontend origin.
3. Calls `init_db()` within the app context to ensure the database and tables exist before serving requests.
4. Registers a `/api/health` route returning `{ "status": "ok", "project": "week28_quiz_platform" }`.

**Commit:** `feat(week28): add Flask app factory with CORS and health route`

---

## 7. Server Entrypoint (`backend/run.py`)
Simple entrypoint that calls `create_app()` and starts the Flask development server on `0.0.0.0:5000`.

**Commit:** `feat(week28): add backend server entrypoint`

---

## 8. Verification

- Ran `python run.py` from the `backend/` directory.
- Flask started in debug mode on `http://127.0.0.1:5000`.
- Thunder Client confirmed `GET /api/health` returned `200 OK`:

```json
{
  "project": "week28_quiz_platform",
  "status": "ok"
}
```

- `data/quiz.db` was created automatically with 2 quizzes and 10 questions seeded.

---

## 9. Next Steps (Day 2)
Build the core API routes inside `backend/app/routes/`:

- `GET /api/quizzes` — List all quizzes
- `GET /api/quizzes/<id>` — Fetch quiz + questions (correct answers stripped)
- `POST /api/quizzes/<id>/submit` — Grade answers and save to leaderboard
- `GET /api/quizzes/<id>/leaderboard` — Fetch top 10 scores
