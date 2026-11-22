---
date: 2026-09-08
project: CI/CD Pipeline Setup
topic: Day 1 - Production Target App & Baseline Tests
Tags:
  - "[[DevOps]]"
  - "[[CICD]]"
  - "[[Python]]"
  - "[[Flask]]"
  - "[[Testing]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 37 - DAY 1

**Core Objective:** Engineer the production target application that will serve as the core service for our CI/CD pipeline. Build a modular Flask REST API with multi-environment configuration profiles, SQLite WAL database persistence for deployment audit logging, health and version telemetry endpoints, and a comprehensive baseline unit test suite passing with zero lint warnings.

---

## 1. The Big Picture and Simple Explanation

Before engineering an automated Continuous Integration and Continuous Deployment (CI/CD) pipeline, you need a resilient, self-aware production target service. If the underlying application lacks structured telemetry, environment configurability, or health probes, the deployment pipeline cannot objectively determine if a rollout succeeded or failed.

In Day 1, we constructed the core foundation for our CI/CD target service:
1. **Multi-Profile Configuration:** The service dynamically configures itself depending on whether it runs in local development (`development`), continuous integration (`testing`), an ephemeral staging cluster (`staging`), or live production (`production`).
2. **Persistent Audit Telemetry:** Every deployment and automated pipeline execution is logged in SQLite operating in Write-Ahead Logging (WAL) mode, capturing Git commit hashes, environment identifiers, execution durations, and exit statuses.
3. **Automated Smoke Probes:** Exposes dedicated telemetry endpoints (`/api/v1/health` and `/api/v1/version`) enabling upstream orchestration runners and load balancers to interrogate server health, database connectivity, and exact software versions.

```
+-----------------------------------------------------------------------------+
|                     CI/CD Production Target Architecture                    |
|                                                                             |
|   +-----------------------+              +------------------------------+   |
|   |  HTTP Client / Tester |              |   Environment Configurator   |   |
|   +-----------+-----------+              |   (FLASK_ENV Profiles)       |   |
|               |                          +--------------+---------------+   |
|               v                                         |                   |
|   +-----------------------------------------------------+---------------+   |
|   |                  Flask Application Factory (create_app)             |   |
|   +-------------------------------------+-------------------------------+   |
|                                         |                                   |
|               +-------------------------+-------------------------+         |
|               v                                                   v         |
|   +-------------------------------+           +-------------------------+   |
|   |  Health & Telemetry Blueprint |           |   Deployment Blueprint  |   |
|   |  GET /api/v1/health           |           |   GET/POST /deployments |   |
|   |  GET /api/v1/version          |           |   GET/POST /pipeline    |   |
|   +---------------+---------------+           +------------+------------+   |
|                   |                                        |                |
|                   +-------------------+--------------------+                |
|                                       v                                     |
|                       +-------------------------------+                     |
|                       |  Data Access Layer (app/db.py)|                     |
|                       +---------------+---------------+                     |
|                                       |                                     |
|                                       v                                     |
|                       +-------------------------------+                     |
|                       |   SQLite 3 WAL Database       |                     |
|                       |   - deployments               |                     |
|                       |   - pipeline_runs             |                     |
|                       +-------------------------------+                     |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Multi-Environment Configuration (`app/config/settings.py`)
Configured an object-oriented settings hierarchy rooted in an abstract `Config` base class, extended by `DevelopmentConfig`, `TestingConfig`, `StagingConfig`, and `ProductionConfig`:
- Dynamic runtime environment resolution driven by the `FLASK_ENV` environment variable.
- Automated Git commit hash resolution executing `git rev-parse --short HEAD` with fallback to a containerized `GIT_COMMIT` environment variable.
- Configurable SQLite connection parameters and strict database path scoping.

```python
class Config:
    TESTING = False
    DEBUG = False
    DB_PATH = os.getenv("DB_PATH", "data/pipeline.db")
    VERSION = "v1.0.0"

    @classmethod
    def get_commit_hash(cls):
        try:
            return subprocess.check_output(
                ["git", "rev-parse", "--short", "HEAD"]
            ).decode("utf-8").strip()
        except Exception:
            return os.getenv("GIT_COMMIT", "unknown")
```

### Database & Audit Storage Layer (`data/schema.sql`, `app/db.py`)
- SQLite 3 operating with `PRAGMA journal_mode = WAL` and `PRAGMA synchronous = NORMAL`, granting concurrent read access without blocking write transactions.
- Table `deployments`: Records historical deployments, target environments (`development`, `staging`, `production`), release version strings, Git commit hashes, and deployment status flags.
- Table `pipeline_runs`: Captures end-to-end pipeline executions, branch metadata, stage statuses, and timing durations.

### Telemetry & Management REST APIs (`app/routes/`)
- `GET /api/v1/health`: Returns overall service health, system timestamp, and active database read probe validation.
- `GET /api/v1/version`: Returns semantic version tag (`v1.0.0`), short commit hash, and current active environment profile.
- `GET /api/v1/deployments` & `POST /api/v1/deployments`: Queries and persists deployment audit records with strict payload validation.
- `GET /api/v1/pipeline-runs` & `POST /api/v1/pipeline-runs`: Records automated pipeline runs triggered by local or CI runners.

---

## 3. Verification & Quality Gates Report

The baseline target service was verified using an isolated Pytest suite leveraging temporary in-memory database fixtures in `tests/conftest.py`:

| Test Suite Module | Target Scope | Tests Passed | Status |
| :--- | :--- | :---: | :---: |
| `tests/test_health_and_version.py` | Health probes, version metadata, profile switches | 3 | Passed |
| `tests/test_deployment_routes.py` | Deployment creation, validations, pipeline audits | 6 | Passed |

### Quality Gates Execution Summary
- **Flake8:** 0 style, whitespace, or PEP 8 violations detected.
- **Black:** 100% compliant formatting.
- **isort:** Clean import ordering enforced.
- **Bandit:** 0 security vulnerabilities or shell execution vectors.
- **Pytest:** 9 tests passed in 0.28 seconds with 92.1% code coverage.

---

## 4. Key Engineering Insights & Decisions

- **Self-Aware Applications Simplify Pipelines:** Rather than hardcoding version numbers inside CI/CD shell scripts, embedding Git metadata directly into the application binary/runtime enables declarative post-deployment verification.
- **WAL-Mode Concurrency in SQLite:** Standard SQLite locks the entire database file during writes. Enabling WAL mode permits health checks and telemetry reads to proceed uninterrupted while deployment records are actively being written.
- **Fixture Isolation in Conftest:** Connecting test runs to temporary SQLite instances guarantees zero pollution of local development state and eliminates order-dependent test failures.

---

## 5. Next Steps

- Construct automated multi-stage quality gates enforcing Flake8, Black, isort, Bandit, and strict 90%+ branch test coverage enforcement.
- Build a unified local quality runner CLI (`run_quality_checks.py`) to execute all quality checks in a single cross-platform command.
