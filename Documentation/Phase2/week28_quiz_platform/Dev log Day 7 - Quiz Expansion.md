---
date: 2026-07-10
project: Project 52
week: 28
module: Quiz Platform with Leaderboards
topic: Day 7 - Quiz Content Expansion & Test Suite Seeding
author: Paul Vincent Aguirre
status: Complete
Tags:
  - "[[Testing]]"
  - "[[Pytest]]"
  - "[[SQL]]"
  - "[[Data Seeding]]"
  - "[[Dev Log]]"
---
# 📝 DEV LOG: WEEK 28 - DAY 7 

## 1. Goal of the Day
Expand the quiz platform's content by adding two new quizzes to the seeder, fix test database seeding isolation bugs to resolve failing test suites, and write new unit test cases to verify input validation and health endpoint status.

---

## 2. Expanded Quizzes
Two new quizzes with exactly 5 questions each were added to the `seed.py` data collection:
1. **JavaScript Deep Dive** (Category: JavaScript, Time limit: 120s): Focuses on data types, prototype scopes, array methods, DOM, and strict comparisons.
2. **Database Systems & SQL** (Category: Databases, Time limit: 75s): Focuses on joins, clauses, keys, constraints, and DDL/DML syntax.

*Now the platform serves exactly 4 quizzes, with a total of 20 questions.*
**Commit:** `feat(week28): add JavaScript and SQL quizzes to database seed script`

---

## 3. Test Database Seeding & Isolation Fixes

### 🐛 The Test Isolation Cache Bug
- **Symptom:** After cleaning `schema.sql` to structural declarations only, running `pytest` failed 16/21 tests with 404/KeyErrors because the test database was empty.
- **Root Cause:** Trying to load `data.seed` dynamically in `conftest.py` worked for the first test run, but subsequent tests referenced cached module-level configurations in `sys.modules`, pointing to the first test run's connection path. The second test case's database was left completely unseeded.
- **Fix:** Refactored `get_connection()` inside `seed.py` to evaluate `os.environ.get("DATABASE_PATH")` dynamically at runtime, ensuring isolation and clean seeding on every single temporary test database.
- **Commit:** `fix(week28): resolve DB path dynamically in get_connection to support conftest mock paths`

---

## 4. Test Suite Additions
We updated our assertions to expect 4 quizzes (up from 2) and expanded our route verification suite to **25 passed unit tests** by adding:
* `test_returns_400_when_answers_contain_non_integers`: Verifies that sending strings/other types inside the answers array is blocked by our Day 6 input validation.
* `test_returns_400_when_time_taken_negative`: Verifies negative duration values are rejected.
* `test_grades_out_of_bounds_answers_as_incorrect`: Verifies out-of-bounds options are graded incorrect rather than triggering index/coercion crashes.
* `test_health_check_returns_200`: Verifies Flask base health endpoints.

**Commit:** `test(week28): seed mock database in conftest fixture and expand test suite to 25 cases covering input validation and health endpoint`

---

## 5. Verification Check
```bash
collected 25 items
tests\test_quiz_routes.py .........................                      [100%]
============================= 25 passed in 0.80s ==============================
```
*All tests are green.*
