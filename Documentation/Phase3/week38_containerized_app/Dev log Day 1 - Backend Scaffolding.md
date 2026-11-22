---
date: 2026-09-13
project: Containerized App with Docker
topic: Day 1 - Backend Scaffolding, Dual Adapters & Baseline Tests
Tags:
  - "[[DevOps]]"
  - "[[Docker]]"
  - "[[Python]]"
  - "[[Flask]]"
  - "[[Testing]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 38 - DAY 1

**Core Objective:** Kick off Week 38 by building the core Python backend for our containerized task platform, implementing dual-adapter database and caching layers that work seamlessly in both local testing and Docker containers, and establishing a 26-test baseline suite with 95.99% branch coverage.

---

## 1. The Big Picture and Simple Explanation

Before packaging applications into containers using Dockerfiles and orchestrating multi-service clusters with Docker Compose, the underlying backend must be engineered for container execution.

In production environments, containerized microservices rely on separate network services: a PostgreSQL database container and a Redis cache container. However, during local development or automated testing pipelines where Docker may not be running, services must not crash with missing network connection errors.

In Day 1, we constructed a resilient dual-adapter architecture:
1. **Adaptive Database Layer (`app/db.py`):** Automatically detects whether it should connect to PostgreSQL (when running inside a container cluster with `DATABASE_URL`) or SQLite (during local test runs). It transparently handles placeholder syntax mapping (`%s` vs `?`) and schema creation across engines.
2. **Resilient Caching Adapter (`app/cache.py`):** Connects to Redis when available, but automatically falls back to an in-memory thread-safe dictionary with full TTL expiration when Redis is absent.
3. **Container-Aware Health & Readiness Probes (`app/routes/health_routes.py`):** Implements Kubernetes/Docker standard probes: `/api/v1/health` for liveness and `/api/v1/ready` for database and cache dependency validation.
4. **Task Management & Benchmarking API (`app/routes/task_routes.py`):** Full CRUD for task entities combined with a live `/benchmark` endpoint measuring database query latency versus Redis retrieval speed.

```
+-----------------------------------------------------------------------------+
|                     Containerized App Dual-Adapter Architecture             |
|                                                                             |
|      +---------------------------------------------------------------+      |
|      |             HTTP Client / Browser Ingress Requests            |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      |               Flask REST Application Factory                  |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|         +----------------------------+----------------------------+         |
|         |                                                         |         |
|         v                                                         v         |
|  +-------------------------------+             +-------------------------+  |
|  | Health Probes Blueprint       |             | Task & Benchmark API    |  |
|  | GET /api/v1/health (Liveness) |             | GET / POST /tasks       |  |
|  | GET /api/v1/ready (Readiness) |             | GET /benchmark          |  |
|  +---------------+---------------+             +------------+------------+  |
|                  |                                          |               |
|                  +--------------------+---------------------+               |
|                                       |                                     |
|                                       v                                     |
|      +---------------------------------------------------------------+      |
|      | Dual-Adapter Service Abstractions                             |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                     +----------------+----------------+                     |
|                     |                                 |                     |
|                     v                                 v                     |
|      +-------------------------------+    +--------------------------+      |
|      | Database Adapter (app/db.py)  |    | Cache Adapter            |      |
|      | Postgres (Prod) | SQLite (Dev)|    | Redis (Prod) | Memory (Dev)      |
|      +---------------+---------------+    +-------------+------------+      |
|                      |                                  |                   |
|                      v                                  v                   |
|      +-------------------------------+    +--------------------------+      |
|      | Relational Storage Layer      |    | Key-Value In-Memory Tier |      |
|      +-------------------------------+    +--------------------------+      |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Multi-Environment Configuration (`app/config/settings.py`)
- Constructed inheritance profiles: `Config` -> `DevelopmentConfig`, `TestingConfig`, `ProductionConfig`.
- Automated Git commit hash extraction (`git rev-parse --short HEAD`) with containerized `GIT_COMMIT` environment fallbacks.
- Environment-aware connection strings resolving `DATABASE_URL` (PostgreSQL) and `REDIS_URL` (Redis).

### Dual-Engine Database Manager (`app/db.py` & `data/schema.sql`)
- Auto-detects target engine based on connection URI scheme (`postgresql://` vs `sqlite:///`).
- Dynamically translates parameter placeholders (`?` for SQLite, `%s` for psycopg2/PostgreSQL).
- Normalizes DDL dialect differences, converting SQLite `AUTOINCREMENT` into PostgreSQL `SERIAL PRIMARY KEY`.
- Features an active connection latency ping reporting database round-trip times in milliseconds.

```python
class DatabaseManager:
    def __init__(self, uri):
        self.uri = uri
        self.is_postgres = uri.startswith("postgresql://")

    def execute(self, query, params=None):
        params = params or ()
        normalized_query = self._normalize_query(query)
        with self.get_connection() as conn:
            with conn.cursor() as cur:
                cur.execute(normalized_query, params)
                if query.strip().upper().startswith("SELECT"):
                    return cur.fetchall()
                conn.commit()

    def _normalize_query(self, query):
        if self.is_postgres:
            return query.replace("?", "%s")
        return query
```

### Resilient Cache Client (`app/cache.py`)
- Connects to Redis instances with configurable socket timeouts and connection pooling.
- Features an automatic fallback to an in-memory thread-safe dictionary with TTL tracking when Redis is disconnected.
- Records real-time performance telemetry: cache hit ratio, active key count, and access latencies.

### Health, Version & Readiness Probes (`app/routes/health_routes.py`)
- `GET /api/v1/health`: Liveness probe reporting process uptime, container hostname, and system environment.
- `GET /api/v1/version`: Returns semantic version tag (`v1.0.0`) and Git commit hash.
- `GET /api/v1/ready`: Readiness probe verifying connectivity to both database and caching tiers. Returns HTTP 200 when operational, or HTTP 503 if any dependency fails.

### Task Management & Comparative Benchmark (`app/routes/task_routes.py`)
- RESTful CRUD endpoints (`GET`, `POST`, `PUT`, `DELETE /api/v1/tasks`) with automatic cache eviction on updates.
- `GET /api/v1/benchmark`: Measures and contrasts raw database retrieval times against Redis in-memory cache reads.

---

## 3. Verification & Quality Gates Report

The dual-adapter backend was verified using an isolated Pytest suite validating both SQLite/memory paths and mock connection adapters:

| Test Suite Module | Target Scope | Tests Passed | Status |
| :--- | :--- | :---: | :---: |
| `tests/test_db_and_cache.py` | Dual database adapters, query normalization, TTL cache | 12 | Passed |
| `tests/test_health_and_version.py` | Liveness, readiness probes, version metadata | 6 | Passed |
| `tests/test_task_routes.py` | Task CRUD lifecycle, cache invalidation, benchmarks | 8 | Passed |

### Quality Gates Execution Summary
- **Flake8:** 0 lint errors (88-character limit enforced).
- **Black:** 100% compliant formatting.
- **isort:** Clean import order validated.
- **Bandit:** 0 security vulnerabilities or insecure query patterns.
- **Pytest:** 26 tests passed in 0.42 seconds with 95.99% branch coverage.

---

## 4. Key Engineering Insights & Decisions

- **Adapter Pattern for Containerization:** Hardcoding database drivers directly inside application routes binds the code strictly to external services. Wrapping storage in an adapter layer allows the service to run anywhere: in local tests, ephemeral CI environments, or production Kubernetes pods.
- **Separation of Liveness and Readiness:** Liveness probes (`/health`) indicate whether the process is alive, while readiness probes (`/ready`) verify whether dependencies are ready to accept customer traffic. Decoupling them prevents unnecessary container restarts during database cold starts.
- **Automatic Cache Invalidation:** Writing tasks directly invalidates corresponding cache keys, ensuring strong eventual consistency while preserving read latency benefits.

---

## 5. Next Steps

- Author a multi-stage production `Dockerfile` with separate builder and runtime stages.
- Configure an unprivileged non-root user (`appuser`, UID 1001) and container `HEALTHCHECK` directives.
