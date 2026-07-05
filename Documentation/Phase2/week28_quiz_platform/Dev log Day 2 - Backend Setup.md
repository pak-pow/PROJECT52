---
date: 2026-07-05
project: Project 52
week: 28
module: Quiz Platform with Leaderboards
Tags:
  - "[[Flask]]"
  - "[[SQLite]]"
  - "[[Python]]"
  - "[[Dev Log]]"
  - "[[Backend]]"
  - "[[REST API]]"
  - "[[Testing]]"
---
# 📝 DEV LOG: WEEK 28 - DAY 2 

## 1. Goal of the Day
Build all four quiz API endpoints following a strict layered architecture: Models (raw DB queries) → Services (business logic) → Routes (HTTP interface). Also write a full automated test suite covering all endpoints.

---

## 2. Architecture Pattern Used
Day 2 follows a three-layer separation of concerns:

```
HTTP Request
    ↓
routes/quiz_routes.py      ← Handles HTTP, calls services
    ↓
services/quiz_service.py   ← Business logic (grading, validation, serialization)
    ↓
models/quiz_model.py       ← Raw DB queries only (no logic)
    ↓
app/db.py                  ← SQLite connection
```

This keeps each layer independently testable and easy to maintain.

---

## 3. Model Layer (`backend/app/models/quiz_model.py`)
Wrote five raw database query functions — no logic, just SQL:

- `get_all_quizzes()` — Joins `quizzes` with `questions` to include a live `question_count` per quiz.
- `get_quiz_by_id(quiz_id)` — Fetches a single quiz row or `None`.
- `get_questions_by_quiz(quiz_id)` — Returns all question rows for a quiz, including `correct_option_index` (used internally for grading only).
- `insert_leaderboard_entry(quiz_id, username, score, time_taken)` — Inserts a new score record.
- `get_leaderboard_by_quiz(quiz_id, limit=10)` — Returns top scores sorted by `score DESC`, then `time_taken_seconds ASC` (tie-breaking by speed).

**Commit:** `feat(week28): add quiz model with raw DB query functions`

---

## 4. Service Layer (`backend/app/services/quiz_service.py`)
Wrote the business logic layer with three categories of functions:

### Serializers
- `serialize_quiz(row)` — Converts a quiz DB row to a safe public dict.
- `serialize_question_public(row)` — Converts a question row to a public dict. **`correct_option_index` is intentionally excluded** to prevent client-side cheating.
- `serialize_leaderboard_entry(row)` — Converts a leaderboard row to a public dict.

### Validator
- `validate_submission(data, expected_count)` — Validates the POST body against three rules:
  - `username` must be present and non-empty.
  - `answers` must be a list with exactly `expected_count` items.
  - `time_taken` must be a non-negative number.
  - Returns `(True, None)` on success or `(False, error_message)` on failure.

### Grading Engine
- `grade_submission(questions, answers)` — Loops through each question, compares the submitted index against the stored `correct_option_index`, and builds a detailed result list including submitted answer text, correct answer text, and a boolean `is_correct` flag.

**Commit:** `feat(week28): add quiz service with serializers, validator, and grading logic`

---

## 5. Routes Layer (`backend/app/routes/quiz_routes.py`)
Registered a Flask Blueprint `quiz_bp` at prefix `/api/quizzes` with four endpoints:

| Method | Endpoint | Status | Description |
|---|---|---|---|
| GET | `/api/quizzes` | 200 | List all quizzes with question counts |
| GET | `/api/quizzes/<id>` | 200 / 404 | Fetch quiz + questions (no correct answers) |
| POST | `/api/quizzes/<id>/submit` | 201 / 400 / 404 | Grade answers, save to leaderboard |
| GET | `/api/quizzes/<id>/leaderboard` | 200 / 404 | Top 10 scores for a quiz |

**Commit:** `feat(week28): add quiz routes Blueprint with list, detail, submit, and leaderboard endpoints`

---

## 6. App Factory Update (`backend/app/__init__.py`)
Imported and registered `quiz_bp` inside `create_app()`. Also fixed a CORS wildcard bug — the original pattern `r"/api/"` did not match sub-paths; corrected to `r"/api/*"`.

**Commit:** `feat(week28): register quiz blueprint in app factory`

---

## 7. Tests (`backend/tests/`)

### conftest.py — Test Fixture
Wrote a `client` fixture using pytest's built-in `tmp_path` to give each test its own isolated temporary SQLite file. This ensures:
- No state bleeds between tests.
- Each test starts with a clean, fully seeded database.

### test_quiz_routes.py — 21 Test Cases
Organized into four test classes:

| Class | Tests |
|---|---|
| `TestListQuizzes` | 200 status, 2 quizzes returned, correct fields, question count = 5 |
| `TestGetQuiz` | 200/404 status, 5 questions returned, no `correct_option_index` in response, options is a list |
| `TestSubmitQuiz` | 201 on valid, score/total fields present, results length, 400 on missing username/answers/time_taken, 404 on bad quiz ID |
| `TestLeaderboard` | 200 status, empty initially, entry appears after submission, 404 on bad quiz ID, sorting (highest score → fastest time) |

**Commits:**
- `test(week28): add pytest conftest with in-memory test client fixture`
- `test(week28): add comprehensive tests for all quiz API routes`

---

## 8. Bug Fixed: SQLite In-Memory Isolation
**Problem:** Setting `DATABASE_PATH=:memory:` in tests caused all 21 tests to fail with `sqlite3.OperationalError: no such table: quizzes`.

**Root Cause:** Every call to `sqlite3.connect(":memory:")` creates a **brand new, completely empty database**. When `init_db()` initialized the schema on connection A, the route handlers opened connection B — which had no tables at all.

**Fix:**
1. Updated `db.py` `get_db()` to detect `file:` URI paths and pass `uri=True` to `sqlite3.connect()`.
2. Updated `conftest.py` to use `tmp_path` (pytest's built-in temp directory fixture) so each test gets a fresh but persistent file-based SQLite DB that all connections can share.

**Commit:** `fix(week28): fix test isolation by using tmp_path DB file and uri-aware get_db()`

---

## 9. Verification

```bash
cd backend
pytest -v
```

**Result:**
```
============================= test session starts =============================
platform win32 -- Python 3.10.0, pytest-8.2.2, pluggy-1.6.0
collected 21 items

tests/test_quiz_routes.py::TestListQuizzes::test_returns_200 PASSED
tests/test_quiz_routes.py::TestListQuizzes::test_returns_two_seeded_quizzes PASSED
tests/test_quiz_routes.py::TestListQuizzes::test_quiz_has_expected_fields PASSED
tests/test_quiz_routes.py::TestListQuizzes::test_question_counts_are_five PASSED
tests/test_quiz_routes.py::TestGetQuiz::test_returns_200_for_valid_id PASSED
tests/test_quiz_routes.py::TestGetQuiz::test_returns_404_for_invalid_id PASSED
tests/test_quiz_routes.py::TestGetQuiz::test_returns_five_questions PASSED
tests/test_quiz_routes.py::TestGetQuiz::test_correct_option_index_is_not_in_response PASSED
tests/test_quiz_routes.py::TestGetQuiz::test_options_is_a_list PASSED
tests/test_quiz_routes.py::TestSubmitQuiz::test_returns_201_on_valid_submission PASSED
tests/test_quiz_routes.py::TestSubmitQuiz::test_returns_score_and_total PASSED
tests/test_quiz_routes.py::TestSubmitQuiz::test_results_list_length_matches_questions PASSED
tests/test_quiz_routes.py::TestSubmitQuiz::test_returns_400_when_username_missing PASSED
tests/test_quiz_routes.py::TestSubmitQuiz::test_returns_400_when_answers_wrong_length PASSED
tests/test_quiz_routes.py::TestSubmitQuiz::test_returns_400_when_time_taken_missing PASSED
tests/test_quiz_routes.py::TestSubmitQuiz::test_returns_404_for_invalid_quiz PASSED
tests/test_quiz_routes.py::TestLeaderboard::test_returns_200 PASSED
tests/test_quiz_routes.py::TestLeaderboard::test_returns_empty_leaderboard_initially PASSED
tests/test_quiz_routes.py::TestLeaderboard::test_entry_appears_after_submission PASSED
tests/test_quiz_routes.py::TestLeaderboard::test_returns_404_for_invalid_quiz PASSED
tests/test_quiz_routes.py::TestLeaderboard::test_leaderboard_sorted_by_score_then_time PASSED

============================= 21 passed in 1.76s ==============================
```
