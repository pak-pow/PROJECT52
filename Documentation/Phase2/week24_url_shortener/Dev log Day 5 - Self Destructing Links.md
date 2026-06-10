---
date: 2026-06-10
project: Project 52
week: 24
module: URL Shortener Service (Backend)
topic: Day 5 - Time-To-Live (TTL) & Self-Destructing Links
Tags:
  - "[[Backend Architecture]]"
  - "[[API Design]]"
  - "[[Database Schema]]"
  - "[[Testing]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 24 - DAY 5

## 1. Goal of the Day
The objective was to implement a Time-To-Live (TTL) feature, allowing users to generate short codes that automatically expire after a set number of hours. This ensures sensitive links or limited-time promotional URLs are mathematically destroyed without requiring manual deletion.

---

## 2. Schema Evolution & Timezone Logic
* **Database Schema (`schema.sql`):** Altered the `urls` table to include an `expires_at` column, storing the calculated death clock as an ISO-formatted string.
* **Service Layer (`url_service.py`):** Implemented datetime logic using `timezone.utc` to calculate the exact future expiration time during the `POST /api/shorten` request.
* **The Naive Timezone Bug:** Overcame a silent logical failure where SQLite returned a "naive" timestamp string, preventing accurate comparison against the current aware UTC clock. Resolved by explicitly coercing the parsed `expires_at` string back into a `timezone.utc` aware object before evaluation.

---

## 3. API Routing & The 410 Status
* Extracted the optional `expires_in_hours` parameter from the JSON payload.
* Leveraged the explicit `410 Gone` HTTP status code during the `GET /<short_code>` resolution phase. If the death clock has passed, the service intentionally interrupts the redirect, raises an `ExpiredURLError`, and the routing layer returns the 410 status to explicitly inform the client that the resource was permanently destroyed.

---

## 4. Integration Testing
* Wrote `test_self_destructing_link_returns_410` to inject a negative integer (`-1` hours) into the creation payload, generating a link that was born already expired.
* Verified that attempting to resolve this short code successfully bypasses the `301 Redirect` and correctly returns the `410 Gone` response.

**Result:** Total backend test passing at 15/15.
