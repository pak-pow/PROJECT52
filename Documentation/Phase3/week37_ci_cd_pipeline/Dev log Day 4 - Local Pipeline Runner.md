---
date: 2026-09-09
project: CI/CD Pipeline Setup
topic: Day 4 - Local Pipeline Runner CLI & Status Badges
Tags:
  - "[[DevOps]]"
  - "[[CICD]]"
  - "[[Automation]]"
  - "[[CLI]]"
  - "[[Testing]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 37 - DAY 4

**Core Objective:** Build a local pipeline orchestrator CLI (`pipeline_runner.py`) replicating our GitHub Actions stages on developer workstations with strict stage gating and JSON execution telemetry, alongside a zero-dependency SVG badge generator (`badge_generator.py`) for live build and coverage metrics.

---

## 1. The Big Picture and Simple Explanation

Relying exclusively on remote cloud pipelines creates long feedback loops. Pushing a commit, waiting 5 minutes for a cloud runner to queue and spin up, only to find a minor lint or unit test failure disrupts engineering velocity.

In Day 4, we implemented local-first pipeline parity:
1. **Local Pipeline Orchestrator (`pipeline_runner.py`):** Replicates the exact sequential stages of cloud CI/CD locally: `lint` -> `security` -> `test` -> `build` -> `deploy`. Any stage failure terminates execution immediately to preserve computing cycles and avoid invalid builds.
2. **Zero-Dependency SVG Badge Generator (`badge_generator.py`):** Produces vector-crisp, Shields.io-style status badges (`build: passing`, `coverage: 95.0%`, `deploy: staging`) locally using pure Python string formatting without third-party services.
3. **Structured Telemetry (`reports/pipeline_report.json`):** Emits structured JSON records containing stage runtimes, exit codes, and coverage metrics for ingestion by command-line utilities and future web dashboards.

```
+-----------------------------------------------------------------------------+
|                     Local Pipeline Orchestrator Architecture                |
|                                                                             |
|      +---------------------------------------------------------------+      |
|      |             Developer CLI Trigger (pipeline_runner.py)        |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Stage 1: Linting Gate (Flake8 & Black Formatting)             |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                     +----------------+----------------+                     |
|                     | [Pass]                          | [Fail: Abort]       |
|                     v                                 v                     |
|      +-------------------------------+    +--------------------------+      |
|      | Stage 2: Security Gate        |    | Terminate Early (Exit 1) |      |
|      | Bandit AST Analysis (-ll)     |    +--------------------------+      |
|      +---------------+---------------+                                      |
|                      |                                                      |
|     +----------------+----------------+                                     |
|     | [Pass]                          | [Fail: Abort]                       |
|     v                                 v                                     |
|      +-------------------------------+    +--------------------------+      |
|      | Stage 3: Testing & Coverage   |    | Terminate Early (Exit 1) |      |
|      | Pytest + Coverage >= 90%      |    +--------------------------+      |
|      +---------------+---------------+                                      |
|                      |                                                      |
|     +----------------+----------------+                                     |
|     | [Pass]                          | [Fail: Abort]                       |
|     v                                 v                                     |
|      +-------------------------------+    +--------------------------+      |
|      | Stage 4: Package Artifact     |    | Terminate Early (Exit 1) |      |
|      | app-release-bundle.tar.gz     |    +--------------------------+      |
|      +---------------+---------------+                                      |
|                      |                                                      |
|     +----------------+----------------+                                     |
|     | [Pass]                          | [Fail: Abort]                       |
|     v                                 v                                     |
|      +-------------------------------+    +--------------------------+      |
|      | Stage 5: Deploy & Record      |    | Terminate Early (Exit 1) |      |
|      | Audit Logging in SQLite WAL   |    +--------------------------+      |
|      +---------------+---------------+                                      |
|                      |                                                      |
|                      v                                                      |
|      +---------------------------------------------------------------+      |
|      | Telemetry & Badge Generation (badge_generator.py)             |      |
|      | Output: badges/*.svg and reports/pipeline_report.json         |      |
|      +---------------------------------------------------------------+      |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Local Pipeline Runner CLI (`scripts/pipeline_runner.py`)
- Sequentially executes all 5 pipeline phases while tracking millisecond-accurate stage durations.
- Flag support: Accepts `--stage <name>` for targeted single-phase execution (`lint`, `security`, `test`, `build`, `deploy`) and `--env <environment>` for target environment selection.
- Strict gating: Returns exit code `1` immediately upon any stage failure, suppressing downstream packaging and deployment operations.
- Emits execution telemetry payload to `reports/pipeline_report.json` containing timestamp, overall status, total duration, and per-stage breakdowns.

### Standalone SVG Status Badge Generator (`scripts/badge_generator.py`)
- Renders pixel-perfect SVG graphics without external libraries or network dependencies using XML templates.
- Dynamically assigns semantic colors based on thresholds:
  - Coverage $\ge 90.0\%$: Forest Green (`#4c1`).
  - Coverage $\ge 80.0\%$: Olive Yellow (`#a4a61d`).
  - Coverage $\ge 70.0\%$: Amber Orange (`#dfb317`).
  - Coverage $< 70.0\%$: Crimson Red (`#e05d44`).
- Calculates dynamic text width metrics to ensure proper right-pill geometry across variable-length percentages and status words.

```python
def generate_badge(label, value, color, output_path):
    label_width = len(label) * 7 + 12
    value_width = max(len(value) * 7.5 + 16, 44)
    total_width = label_width + value_width

    svg_content = f"""<svg xmlns="http://www.w3.org/2000/svg" width="{total_width}" height="20">
  <linearGradient id="b" x2="0" y2="100%">
    <stop offset="0" stop-color="#bbb" stop-opacity=".1"/>
    <stop offset="1" stop-opacity=".1"/>
  </linearGradient>
  <mask id="a"><rect width="{total_width}" height="20" rx="3" fill="#fff"/></mask>
  <g mask="url(#a)">
    <path fill="#555" d="M0 0h{label_width}v20H0z"/>
    <path fill="{color}" d="M{label_width} 0h{value_width}v20H{label_width}z"/>
    <path fill="url(#b)" d="M0 0h{total_width}v20H0z"/>
  </g>
  <g fill="#fff" text-anchor="middle" font-family="DejaVu Sans,Verdana,sans-serif" font-size="11">
    <text x="{label_width / 2}" y="15" fill="#010101" fill-opacity=".3">{label],
    <text x="{label_width / 2}" y="14">{label}</text>
    <text x="{label_width + value_width / 2}" y="15" fill="#010101" fill-opacity=".3">{value}</text>
    <text x="{label_width + value_width / 2}" y="14">{value}</text>
  </g>
</svg>"""
    with open(output_path, "w", encoding="utf-8") as fp:
        fp.write(svg_content)
```

### Automated Unit Tests (`tests/test_badge_generator.py` & `tests/test_pipeline_runner.py`)
- Tested SVG output validation: Confirmed valid XML tags, presence of correct color codes, and accurate text width scaling.
- Tested runner isolation: Verified CLI argument parsing, stage isolation, non-zero exit codes on failure injection, and report file emission.

---

## 3. Verification & Quality Gates Report

The local pipeline runner and badge generator were verified through dedicated automated tests:

| Test Suite Module | Target Scope | Tests Passed | Status |
| :--- | :--- | :---: | :---: |
| `tests/test_badge_generator.py` | SVG structure, color thresholds, pill width math | 5 | Passed |
| `tests/test_pipeline_runner.py` | CLI execution, stage gating, report JSON emission | 5 | Passed |
| `tests/test_workflows.py` | CI/CD GitHub Actions parity checks | 4 | Passed |
| `tests/test_quality_gates.py` | Flake8, Bandit, AST security enforcement | 6 | Passed |
| `tests/test_health_and_version.py` | Target service health and version telemetry | 3 | Passed |
| `tests/test_deployment_routes.py` | Audit database routes and pipeline records | 6 | Passed |

### Quality Gates Execution Summary
- **Total Tests:** 29 automated tests passing in 1.15 seconds.
- **Branch Coverage:** 95.05% total branch coverage.
- **Linter Status:** Flake8 and Black reports clean with 0 warnings.
- **Bandit Audit:** Clean scan with 0 issues.

---

## 4. Key Engineering Insights & Decisions

- **Fast-Feedback Development Cycles:** Running pipeline stages locally before pushing code reduces turnaround times from minutes to seconds, improving code quality before commits reach shared Git branches.
- **Offline Self-Contained Badges:** Generating SVG badges directly in Python removes reliance on external web badge APIs, allowing internal or offline enterprise pipelines to generate status badges safely.
- **Structured JSON Telemetry as an Interface:** Exporting machine-readable reports (`pipeline_report.json`) decouples pipeline execution from visualization, enabling command-line tools, scripts, and web interfaces to consume identical data.

---

## 5. Next Steps

- Build a zero-dependency interactive frontend dashboard (`frontend/`) to visualize real-time pipeline execution, streaming console logs, and deployment audit histories.
- Connect the frontend to Flask backend routes with support for an offline fallback demo mode.
