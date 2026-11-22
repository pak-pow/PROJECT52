---
date: 2026-09-15
project: Containerized App with Docker
topic: Day 3 - Multi-Container Docker Compose Orchestration
Tags:
  - "[[DevOps]]"
  - "[[Docker]]"
  - "[[Docker Compose]]"
  - "[[Nginx]]"
  - "[[PostgreSQL]]"
  - "[[Redis]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 38 - DAY 3

**Core Objective:** Wire up a complete 4-tier microservices stack using Docker Compose—orchestrating an Nginx reverse proxy gateway, Flask REST API, PostgreSQL 16 relational database, and Redis 7 cache with health-dependent startup order, named volume persistence, and an isolated bridge network.

---

## 1. The Big Picture and Simple Explanation

In modern production deployments, microservices operate as interconnected fleets of specialized containers rather than single monolithic servers. Coordinating multiple containers manually with separate commands leads to brittle, order-dependent failures.

In Day 3, we defined and orchestrated our entire multi-tier system using Docker Compose:
1. **Declarative Infrastructure (`docker-compose.yml`):** All 4 services, their networks, named storage volumes, port bindings, and environment variables are captured in a single version-controlled configuration.
2. **Deterministic Startup Ordering (`condition: service_healthy`):** If the Flask API starts before PostgreSQL is ready to accept socket connections, the API will crash. Using Docker health probes with `condition: service_healthy`, Docker Compose guarantees that PostgreSQL and Redis are fully operational before bootstrapping Flask, and waits for Flask before starting Nginx.
3. **Isolated Network Perimeter (`docker_pulse_network`):** Only the Nginx ingress gateway exposes a host port (`8080`). PostgreSQL, Redis, and Flask communicate strictly through an internal bridge network using container service names as DNS hostnames (`db`, `cache`, `api`).
4. **Persistent Named Volumes:** Database records and cache logs survive container termination and recreation through mounted host volumes (`docker_pulse_postgres_data` and `docker_pulse_redis_data`).

```
+-----------------------------------------------------------------------------+
|                     Multi-Container Docker Compose Topology                 |
|                                                                             |
|      +---------------------------------------------------------------+      |
|      |             Web Browser / Client Traffic (Host Port 8080)     |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Nginx Reverse Proxy Gateway (web:80 -> host:8080)             |      |
|      | - Serves static frontend assets (HTML/CSS/JS)                 |      |
|      | - Gzip compression enabled                                    |      |
|      | - Liveness probe: GET /healthz                                |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      | [Proxy Pass: /api/v1/*]              |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Internal Bridge Network: docker_pulse_network                 |      |
|      |                                                               |      |
|      |   +-------------------------------------------------------+   |      |
|      |   | Flask REST Application (api:5000)                     |   |      |
|      |   | Unprivileged appuser (UID 1001)                       |   |      |
|      |   +-------------------+-------------------+---------------+   |      |
|      |                       |                   |                   |      |
|      |       +---------------+                   +---------------+   |      |
|      |       v                                                   v   |      |
|      |   +---------------------------+       +-------------------+   |      |
|      |   | PostgreSQL 16 (db:5432)   |       | Redis 7 (cache)   |   |      |
|      |   | Parameterized SQL Storage |       | Key-Value Cache   |   |      |
|      |   +-------------+-------------+       +---------+---------+   |      |
|      +-----------------|-------------------------------|-------------+      |
|                        |                               |                    |
|                        v                               v                    |
|      +-------------------------------+   +---------------------------+      |
|      | Named Volume: postgres_data   |   | Named Volume: redis_data  |      |
|      | Persistent Database Storage   |   | Append-Only File (AOF)    |      |
|      +-------------------------------+   +---------------------------+      |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Nginx Gateway Configuration (`nginx/nginx.conf`)
- Ingress reverse proxy: Listens on port 80 (mapped to host 8080) and proxies `/api/v1/` to the upstream Flask backend (`http://api:5000`).
- Header forwarding: Transmits client IP addresses using `X-Real-IP` and `X-Forwarded-For`.
- Gzip compression: Automatically compresses `application/json`, `text/css`, `text/html`, and `image/svg+xml`.
- Gateway health endpoint: Exposes `/healthz` returning HTTP 200 without routing traffic to downstream backend instances.

### Orchestration Definition (`docker-compose.yml`)
- Declared services: `web`, `api`, `db`, and `cache` within `docker_pulse_network`.
- Health check configurations:
  - `db`: `pg_isready -U postgres -d pulse_db` every 10s.
  - `cache`: `redis-cli ping` every 10s.
  - `api`: `curl -f http://localhost:5000/api/v1/health` every 15s.
- Explicit startup order: `api` depends on `db` and `cache` with `condition: service_healthy`; `web` depends on `api` with `condition: service_healthy`.

### Database Bootstrapping Resilience (`backend/app/db.py`)
- Added retry loops (`retries=5, delay=1.0s`) to `init_db` to absorb container startup delays while the PostgreSQL daemon initializes tables.

### Automated Compose and Gateway Tests (`tests/test_compose_and_nginx.py`)
- Verified `docker-compose.yml` schema: validated all 4 services, volume declarations, and health check definitions.
- Verified Nginx configuration: confirmed reverse proxy paths, upstream blocks, and gzip directives.
- Validated retry handling: tested database connection retries under simulated initial socket failures.

---

## 3. Verification & Quality Gates Report

The multi-container configuration was verified through automated tests and container inspection:

| Test Suite Module | Target Scope | Tests Passed | Status |
| :--- | :--- | :---: | :---: |
| `tests/test_compose_and_nginx.py` | Compose schema, Nginx configs, DB retry loops | 4 | Passed |
| `tests/test_dockerfile_standards.py` | Multi-stage builder/runtime standards | 6 | Passed |
| `tests/test_db_and_cache.py` | Database and cache adapter operations | 12 | Passed |
| `tests/test_health_and_version.py` | Container health and readiness endpoints | 6 | Passed |
| `tests/test_task_routes.py` | Task management APIs and benchmarks | 8 | Passed |

### Quality Gates Execution Summary
- **Total Tests:** 36 automated tests passing in 0.94 seconds.
- **Branch Coverage:** 95.46% total branch coverage.
- **Service Verification:** All 4 containers booted successfully and passed health checks.
- **Linters & Security:** Flake8, Black, isort, and Bandit passed with 0 violations.

---

## 4. Key Engineering Insights & Decisions

- **Health-Checked Dependency Gating:** Using simple `depends_on` only waits for a container to be started, not for its internal processes to be ready. Enforcing `condition: service_healthy` ensures dependent services never boot against unready databases.
- **Perimeter Security via Internal Networks:** Preventing PostgreSQL and Redis ports from binding directly to host interfaces eliminates external access vectors, enforcing that all interactions traverse the reverse proxy and authentication layers.
- **Stateless Reverse Proxy Health Probing:** The `/healthz` endpoint on Nginx allows ingress load balancers to monitor proxy availability without loading backend database connections.

---

## 5. Next Steps

- Implement development hot-reloading overlays (`docker-compose.override.yml`) and hardened production configurations (`docker-compose.prod.yml`).
- Configure named volume data durability verification and Docker Secrets management.
