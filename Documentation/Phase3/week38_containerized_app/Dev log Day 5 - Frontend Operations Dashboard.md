---
date: 2026-09-17
project: Containerized App with Docker
topic: Day 5 - Frontend Operations Dashboard & Container Monitor
Tags:
  - "[[DevOps]]"
  - "[[Docker]]"
  - "[[Frontend]]"
  - "[[Vanilla-JS]]"
  - "[[Observability]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 38 - DAY 5

**Core Objective:** Design and implement a responsive, lightweight, standalone vanilla ES6 operations dashboard (**Docker Pulse**) to monitor the multi-container stack in real time. The frontend interfaces with the Nginx reverse proxy to visualize service topology, poll database and cache latencies, execute comparative query benchmarks (PostgreSQL 16 relational storage vs. Redis 7 in-memory cache), and provide full task management controls with live cache hit/miss feedback.

---

## 1. The Big Picture and Simple Explanation

A multi-tier containerized architecture requires operational observability. While backend logs and terminal probes confirm process execution, operators benefit from an interactive visual control console to assess stack health, monitor latencies, and observe caching performance.

In Day 5, we engineered the frontend operations layer strictly adhering to the standalone vanilla architecture (pure HTML5, CSS3, modern ES6 modules, with zero external frameworks or CDN dependencies):
1. **Real-Time 5-Second Telemetry Polling:** Regularly queries `/healthz` on Nginx and `/api/v1/ready` on Flask to track ping latencies to PostgreSQL and Redis.
2. **PostgreSQL vs. Redis Comparative Benchmarking:** An interactive benchmarking utility executes rapid sequential reads comparing cold database retrieval against in-memory Redis cache hits.
3. **Task Lifecycle Management:** Real-time task creation, filtering, inline status toggling, and deletion controls with non-blocking toast notifications.
4. **Cache Feedback Inspection:** Reads the `X-Cache` response header (`HIT` vs `MISS`) to provide immediate visual confirmation of cache eviction and hydration.

```
+-----------------------------------------------------------------------------+
|                     Docker Pulse Operations Dashboard Architecture          |
|                                                                             |
|      +---------------------------------------------------------------+      |
|      |               Web Browser (http://localhost:8080)             |      |
|      |               frontend/public/index.html (Vanilla ES6)        |      |
|      |                                                               |      |
|      |  +---------------------------------------------------------+  |      |
|      |  | UI Subsystems:                                          |  |      |
|      |  | - Service Topology Grid (Nginx, API, DB, Cache)         |  |      |
|      |  | - Comparative Benchmark Widget (Postgres vs Redis)      |  |      |
|      |  | - Task Lifecycle Manager & Filter Engine                |  |      |
|      |  | - Toast Alert & Notification System                     |  |      |
|      |  +----------------------------+----------------------------+  |      |
|      +-------------------------------|-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Nginx Ingress Gateway (:8080)                                 |      |
|      | - /         -> Static HTML/CSS/ES6 UI Assets                  |      |
|      | - /healthz  -> Gateway Liveness Probe (HTTP 200)              |      |
|      | - /api/v1/* -> Reverse Proxy Pass to Flask API                |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Internal Bridge Network: Flask API (:5000)                    |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                     +----------------+----------------+                     |
|                     |                                 |                     |
|                     v                                 v                     |
|      +-------------------------------+    +--------------------------+      |
|      | PostgreSQL 16 (db:5432)       |    | Redis 7 (cache:6379)     |      |
|      | Parameterized SQL Storage     |    | In-Memory Cache Key-Val  |      |
|      +-------------------------------+    +--------------------------+      |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Modular Frontend Architecture (`week38_containerized_app/frontend`)
- `public/index.html`: Clean semantic HTML layout organizing topology cards, benchmark widgets, and task management forms.
- `src/assets/base.css` & `src/assets/pulse.css`: Design tokens for dark palette theming, responsive grid systems, status beacons, and toast notifications.
- `src/utils/helpers.js`: Pure helper functions for XSS sanitization (`escapeHtml`), timestamp parsing, and dynamic latency badge calculation.
- `src/api/opsApi.js`: Promise-based HTTP client wrapping `/healthz`, `/api/v1/health`, `/api/v1/ready`, `/api/v1/tasks`, and `/api/v1/benchmark`.
- `src/components/serviceGrid.js`: Renders the 4-tier service topology with live status beacons and ping metrics.
- `src/components/benchmarkCard.js`: Interactive benchmark runner contrasting database and cache retrieval speeds with visual comparison bars.
- `src/components/taskManager.js`: Complete task lifecycle management component handling creation, state filtering, and deletion.
- `src/pages/dashboardPage.js`: Master dashboard controller coordinating rendering, user actions, and background telemetry polling.

---

## 3. Verification & Quality Gates Report

The frontend components and backend integration routes were verified using automated tests:

| Test Suite Module | Target Scope | Tests Passed | Status |
| :--- | :--- | :---: | :---: |
| `tests/test_compose_and_nginx.py` | Modular frontend assets, Nginx proxy, static routing | 5 | Passed |
| `tests/test_persistence_and_secrets.py` | Volume persistence, secrets, canary validation | 10 | Passed |
| `tests/test_dockerfile_standards.py` | Multi-stage builder/runtime standards | 6 | Passed |
| `tests/test_db_and_cache.py` | Dual database adapters, query normalization | 12 | Passed |
| `tests/test_health_and_version.py` | Container health and readiness endpoints | 6 | Passed |
| `tests/test_task_routes.py` | Task management APIs and benchmarks | 8 | Passed |

### Quality Gates Execution Summary
- **Total Tests:** 47 automated tests passing in 1.25 seconds.
- **Branch Coverage:** 94.78% total branch coverage.
- **Linters:** Flake8, Black, isort, and Bandit reported 0 errors.

---

## 4. Key Engineering Insights & Decisions

- **Observability Enhances Reliability:** Providing visible latency metrics for both database and cache layers allows developers and operators to immediately identify performance bottlenecks without inspecting log files.
- **Pure Vanilla Architecture in DevOps Dashboards:** Avoiding heavy frontend build steps (Webpack, Vite) allows operations dashboards to be served directly from Nginx static roots with zero build overhead.
- **Clear User-Facing Cache Feedback:** Inspecting the `X-Cache` header helps verify that caching logic functions properly in production and that cache invalidation operates on writes.

---

## 5. Next Steps

- Implement security hardening on the Nginx reverse proxy (security headers, payload size caps, rate limiting).
- Build backend graceful degradation handling database drops and cache restarts with structured 503 responses.
