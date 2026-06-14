---
date: 2026-06-14
project: Project 52
week: 25
module: Recipe Sharing Platform (Full Stack)
topic: Day 2 - Test-Driven Input Validation
Tags:
  - "[[TDD]]"
  - "[[Pytest]]"
  - "[[Backend Validation]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 25 - DAY 2

## 1. Goal of the Day
The objective was to build the business logic engine (`recipe_service.py`) to validate complex multipart form data *before* it interacts with the database or file system. We utilized Test-Driven Development (TDD) to prove the logic handled edge cases and rejected malicious inputs.

## 2. The Validation Engine
* **Text Payload Sanitization:** Built the `validate_recipe_data` function to intercept incoming dictionaries. It explicitly checks for the presence of `title`, `description`, `ingredients`, and `instructions`, strips trailing whitespace, and raises a custom `ValidationError` if any required field is missing or empty.
* **File Extension Whitelisting:** Implemented `allowed_file` to parse incoming filenames and mathematically verify their extensions against a strict whitelist (`png`, `jpg`, `jpeg`, `webp`), mitigating remote code execution risks from unauthorized file types.

## 3. Test-Driven Development (TDD)
* Wrote a comprehensive Pytest suite (`test_recipe_service.py`) to simulate both perfect and malformed API payloads.
* **Bug Squashing:** The test suite successfully trapped a `KeyError` caused by a singular/plural naming mismatch (`instruction` vs `instructions`) and an `AssertionError` caused by missing quotation marks in the error response strings. 
* **Result:** After patching the syntax errors, the validation engine passed 5/5 integration tests. The backend is now fully armored against bad data.
