---
date: 2026-09-10
project: CI/CD Pipeline Setup
topic: Day 5 - Interactive Web Dashboard & Real-Time Pipeline Telemetry
Tags:
  - "[[DevOps]]"
  - "[[CICD]]"
  - "[[Frontend]]"
  - "[[Dashboard]]"
  - "[[Testing]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 37 - DAY 5

**Core Objective:** Build a production-grade, zero-dependency interactive web dashboard (`frontend/`) connecting directly to our Flask CI/CD backend (`deployment_routes.py`), rendering real-time 5-stage pipeline steppers, an ANSI-styled terminal execution log, interactive status badges, deployment audit tables, and seamless offline demo simulation.

---

## 1. The Big Picture and Simple Explanation

While terminal CLIs and workflow files power automated background execution, engineering teams need a centralized operational dashboard to observe build health, monitor deployments, inspect logs, and trigger pipeline runs.

In Day 5, we engineered a dedicated mission control interface:
1. **Interactive Pipeline Stepper:** A visual 5-stage progress track (`Lint` -> `Security` -> `Test & Coverage` -> `Package Bundle` -> `Deploy & Smoke`) with animated state transitions, duration counters, and pass/fail indicators.
2. **Streaming Execution Terminal:** A real-time developer console displaying stdout/stderr execution logs with stage-filtering tabs (`All`, `Lint`, `Security`, etc.), per-line timestamps, and auto-scrolling.
3. **Live SVG Badge Gallery:** Dynamic previews of generated status badges (`build: passing`, `coverage: 95.0%`, `deploy: staging`) with one-click copyable Markdown, HTML, and raw URL snippets.
4. **Dual-Mode Connectivity (Live API + Offline Demo Simulator):** When connected to the Flask backend on `127.0.0.1:5000`, the dashboard triggers real subprocess runs and fetches live SQLite data. If opened standalone or during network outages, it gracefully switches to an offline simulation mode with a status banner.

```
+-----------------------------------------------------------------------------+
|                     CI/CD Operational Dashboard Architecture                |
|                                                                             |
|      +---------------------------------------------------------------+      |
|      |               Web Browser / Operator Interface                |      |
|      |             frontend/public/index.html (Vanilla JS)           |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      |              API Connectivity Check (GET /health)             |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                     +----------------+----------------+                     |
|                     | [Online: Port 5000]             | [Offline Fallback]  |
|                     v                                 v                     |
|      +-------------------------------+    +--------------------------+      |
|      | Flask REST Backend            |    | Offline Demo Simulator   |      |
|      | app/routes/deployment_routes  |    | Client-Side Progression  |      |
|      +---------------+---------------+    +-------------+------------+      |
|                      |                                  |                   |
|         +------------+------------+                     |                   |
|         |                         |                     |                   |
|         v                         v                     v                   |
|  +--------------+          +--------------+      +--------------+           |
|  | Subprocess   |          | Static Files |      | Simulated    |           |
|  | Pipeline Run |          | Badges/JSON  |      | Terminal Log |           |
|  +-------+------+          +-------+------+      +-------+------+           |
|          |                         |                     |                  |
|          +------------+------------+---------------------+                  |
|                       |                                                     |
|                       v                                                     |
|      +---------------------------------------------------------------+      |
|      | UI Component Renderers:                                       |      |
|      | - 5-Stage Stepper Track (pipelineStepper.js)                  |      |
|      | - Monospace Terminal Log Viewer (terminalViewer.js)           |      |
|      | - SVG Badge Gallery with Copy Actions (badgeViewer.js)        |      |
|      | - Historical Deployment Audit Log (deploymentTable.js)        |      |
|      +---------------------------------------------------------------+      |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Backend Integration Routes (`app/routes/deployment_routes.py`)
- `GET /api/v1/pipeline-report`: Reads the latest structured telemetry from `reports/pipeline_report.json` and returns it as JSON. If no execution has run, returns a default schema.
- `GET /api/v1/badges/<name>`: Serves generated SVG badges with `Content-Type: image/svg+xml; charset=utf-8` and `Cache-Control: no-cache` headers, preventing stale browser caching.
- `POST /api/v1/pipeline/trigger`: Executes local pipeline runs via Python subprocess with stage filtering (`all`, `lint`, `security`, `test`, `build`, `deploy`) and target environment configuration (`staging`, `production`).

### Frontend Architecture (`frontend/src/`)
- `assets/base.css` & `assets/pipeline.css`: Modern CSS styling featuring design tokens, glassmorphic card headers, responsive metric displays, and theme variables.
- `utils/theme.js` & `utils/helpers.js`: Client-side theme persistence via `localStorage`, toast notification manager, and human-readable timestamp parsers.
- `api/pipelineApi.js`: Resilient HTTP client with automated fallback to demo simulation if the Flask backend is unreachable.
- `components/pipelineStepper.js`: Interactive 5-stage progress component displaying real-time execution states and stage durations.
- `components/terminalViewer.js`: Monospace terminal console supporting stage filtering, autoscroll behavior, and log export.
- `components/badgeViewer.js`: Interactive badge gallery with one-click copyable embed snippets.
- `components/deploymentTable.js`: Historical deployment log with environment tags, Git commit links, and status chips.

---

## 3. Verification & Quality Gates Report

The frontend and backend integration was verified through end-to-end API testing and browser rendering checks:

| Subsystem Component | Verification Target | Status |
| :--- | :--- | :---: |
| REST Integration Routes | `/pipeline-report`, `/badges/<name>`, `/pipeline/trigger` | Passed |
| Connectivity Probe | Automated fallback between live API and demo mode | Passed |
| Badge Geometry | SVG text-width calculations across various percentages | Passed |
| Terminal Console | Log streaming, stage filtering, and buffer management | Passed |
| Theme Engine | Dark/light theme persistence across browser reloads | Passed |

### Challenge Resolution Summary
- **Progress Track Line Alignment:** The stepper line initially extended beyond the bounds of the 5th stage. Resolved by constraining `.pipeline-track-line` with absolute positioning bounded between 10% and 90% widths.
- **Badge Parser Artifacts:** Inline SVG fallback strings with unescaped attributes caused HTML parsing warnings. Resolved by adopting programmatic DOM SVG generation.
- **Timestamp Standardization:** SQLite default UTC timestamps were inconsistent with ISO format expectations. Implemented a safe parsing helper standardizing display across date strings.

---

## 4. Key Engineering Insights & Decisions

- **Graceful Offline Degradation:** Designing web consoles with client-side simulation fallbacks ensures that portfolios, documentation sites, and demos remain fully interactive even when backend APIs are stopped.
- **Cache-Control for Dynamic SVGs:** SVG badges served directly from disk must include explicit `no-cache` and `no-store` headers, ensuring browsers immediately reflect updated build numbers and coverage statistics.
- **Decoupled Component Architecture:** Separating the UI into distinct vanilla JavaScript component modules (`pipelineStepper.js`, `terminalViewer.js`, `badgeViewer.js`) kept the code maintainable without framework overhead.

---

## 5. Next Steps

- Implement security hardening against directory traversal attacks, parameter injection, and concurrency race conditions in `deployment_routes.py`.
- Construct an automated security and edge-case test suite (`tests/test_security_and_edge_cases.py`) to raise branch coverage above 97%.
