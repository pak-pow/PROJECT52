---
date: 2026-06-09
project: Project 52
week: 24
module: URL Shortener Service (Backend)
topic: Day 4 - Security & Rate Limiting
Tags:
  - "[[Security]]"
  - "[[API Design]]"
  - "[[Testing]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 24 - DAY 4 

## 1. Goal of the Day
With the core URL shortening engine and custom alias features fully operational, the objective shifted to infrastructure security. A public-facing URL shortener is highly vulnerable to Denial of Service (DoS) attacks and database exhaustion via automated scripts. The goal was to implement strict rate limiting across the API.

---

## 2. Infrastructure Tooling
* **Dependency Optimization:** Cleaned up the global `requirements.txt` to ensure the project footprint remains minimal and production-ready, retaining only `Flask`, `Flask-Limiter`, and `pytest`.
* **Flask-Limiter Integration:** Initialized an in-memory rate limiter globally within `app/__init__.py`, utilizing `get_remote_address` to track request origins by IP address. 

---

## 3. Route-Specific Throttling
Applied granular security protocols to the API endpoints based on their operational cost:
* **Writes (`POST /api/shorten`):** Strictly limited to `5 per minute` to prevent database bloat and abuse.
* **Reads (`GET /<short_code>`):** Generously limited to `50 per minute` to ensure legitimate viral links do not break while still capping aggressive scraping bots.
* **Analytics (`GET /api/stats`):** Limited to `5 per minute`. Because analytics endpoints typically require heavier database queries, they are classic vectors for resource exhaustion attacks and must be protected.

---

## 4. Integration Testing (TDD)
Manually engineered a new test suite (`test_url_routes.py`) utilizing Pytest's dependency injection (`client` fixture) to simulate network requests.
* Verified `201 Created` responses for standard usage.
* Verified `400 Bad Request` responses for malformed JSON payloads.
* **The Security Proof:** Wrote a loop to intentionally exhaust the rate limit and mathematically verified that the server successfully intercepts the attack and returns a `429 Too Many Requests` status code.
