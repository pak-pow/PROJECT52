---
date: 2026-06-08
project: Project 52
week: 24
module: URL Shortener Service (Backend)
topic: Day 3 - Custom Aliases & Collision Handling
author: Paul Vincent Aguirre
status: Complete
Tags:
  - "[[Backend Architecture]]"
  - "[[API Design]]"
  - "[[Error Handling]]"
  - "[[TDD]]"
  - "[[Dev Log]]"
---

# DEV LOG: WEEK 24 - DAY 3 

## 1. Goal of the Day
The objective was to upgrade the URL shortening engine to support custom, user-defined aliases (e.g., `domain.com/summer-sale`) while maintaining the strict uniqueness constraints of the database. This required splitting the core business logic into two distinct paths: standard Base62 encoding and direct custom alias injection.

## 2. Business Logic Forking (`url_service.py`)
We refactored the `shorten_url` service to conditionally evaluate the `custom_alias` parameter.
* **Path A (Custom Alias):** Bypasses the auto-increment/Base62 engine. We implemented an `is_valid_alias` regex checker to restrict inputs to alphanumeric characters and hyphens (3-20 characters limit). Crucially, the service proactively checks the database for collisions and aborts the operation if the alias is taken.
* **Path B (Standard):** Falls back to the original deduplication and Base62 algorithmic encoding.

## 3. The API Contract (`url_routes.py`)
Updated the `POST /api/shorten` payload to securely accept the optional `custom_alias` key. 
* **HTTP Status Mapping:** Mapped invalid regex formats to `400 Bad Request`.
* **Conflict Resolution:** Mapped database collision exceptions to the correct HTTP status code: `409 Conflict`.

## 4. Test-Driven Development
Injected three new scenarios into the test suite:
1. `test_custom_alias_success`: Verified standard insertion.
2. `test_custom_alias_collision`: Proved the system correctly blocks duplicate alias attempts.
3. `test_custom_alias_invalid_format`: Verified the regex successfully rejects symbols and spaces.
