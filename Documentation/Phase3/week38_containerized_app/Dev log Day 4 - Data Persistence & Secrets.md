---
date: 2026-09-16
project: Containerized App with Docker
topic: Day 4 - Data Persistence, Environment Profiles & Secrets Management
Tags:
  - "[[DevOps]]"
  - "[[Docker]]"
  - "[[Docker Compose]]"
  - "[[Data Persistence]]"
  - "[[Security]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 38 - DAY 4

**Core Objective:** Guarantee zero data loss across container teardowns using named volumes, implement environment profile overlays (`docker-compose.override.yml` for rapid local hot-reloading vs. `docker-compose.prod.yml` for hardened resource-constrained production), build automated schema migration & seeding utilities, and engineer secure secret resolution supporting Docker Secrets and `*_FILE` paths.

---

## 1. The Big Picture and Simple Explanation

In containerized architectures, container instances are ephemeral: they can be destroyed, rescheduled, or upgraded at any moment. If persistent data is written to a container's writable layer, destroying the container destroys all user data.

In Day 4, we engineered complete storage durability, configuration overlays, and enterprise secret injection:
1. **Host-Managed Named Volumes:** Decoupled database and cache lifecycles from container lifecycles. Even when executing `docker compose down`, named volumes (`docker_pulse_postgres_data` and `docker_pulse_redis_data`) remain intact on host storage. Recreating containers mounts existing volumes without data loss.
2. **Environment Profile Overlays:**
   - **Development (`docker-compose.override.yml`):** Automatically merged during local execution. Mounts the `./backend` source code directly into containers for instant hot-reloading and exposes database/cache ports for local debugging.
   - **Production (`docker-compose.prod.yml`):** Enforces strict CPU and RAM limits (`limits: memory: 512M`), automatic restart policies (`restart: always`), log rotation to protect disks, and zero hardcoded credentials.
3. **Enterprise Secrets Resolution (`get_secret`):** Hardcoding credentials in source code or `.env` files exposes systems to accidental disclosure. Our resolver checks Docker Secrets (`/run/secrets/<name>`) and secret files (`*_FILE`), safely falling back to environment variables during local testing.

```
+-----------------------------------------------------------------------------+
|                     Persistence, Overlays & Secrets Architecture            |
|                                                                             |
|      +---------------------------------------------------------------+      |
|      | Configuration Overlays (docker-compose)                       |      |
|      | Base: docker-compose.yml (Core Services)                      |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                     +----------------+----------------+                     |
|                     |                                 |                     |
|                     v                                 v                     |
|      +-------------------------------+    +--------------------------+      |
|      | Local Dev: override.yml       |    | Production: prod.yml     |      |
|      | - Live source mount: ./backend|    | - Resource quotas (512MB)|      |
|      | - Ports: 5000, 5432, 6379     |    | - Log rotation (10MB)    |      |
|      | - FLASK_ENV=development       |    | - restart: always        |      |
|      +-------------------------------+    +--------------------------+      |
|                                                                             |
|      +---------------------------------------------------------------+      |
|      | Enterprise Secrets Resolver (get_secret)                      |      |
|      | 1. Check <KEY>_FILE environment variable                      |      |
|      | 2. Check /run/secrets/<key> (Docker Secrets)                  |      |
|      | 3. Fallback to os.getenv(key, default)                        |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Storage Durability (Named Volumes)                            |      |
|      | Containers: Ephemeral | Destroyed on compose down             |      |
|      | Volumes: Persistent   | Preserved on host disk                |      |
|      | - docker_pulse_postgres_data (Postgres ACID Storage)          |      |
|      | - docker_pulse_redis_data (Redis AOF Cache Storage)           |      |
|      +---------------------------------------------------------------+      |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Environment Overlays (`docker-compose.override.yml` & `docker-compose.prod.yml`)
- `docker-compose.override.yml`: Merged by default during `docker compose up`. Mounts local source directories into `/app` to enable hot-reloading, exposes ports `5000`, `5432`, and `6379`, and activates development profiling.
- `docker-compose.prod.yml`: Applied in production environments (`-f docker-compose.yml -f docker-compose.prod.yml`). Implements CPU and memory constraints (Postgres: 1.0 CPU / 512MB RAM; API: 1.0 CPU / 512MB RAM; Redis: 0.5 CPU / 256MB RAM), `json-file` log rotation with 10MB maximum file size, and Docker Secrets mounts.

### Enterprise Secrets Resolution (`backend/app/config/settings.py`)
- Created a prioritized resolution function `get_secret(key, default=None)`:
  1. Resolves path referenced by `<KEY>_FILE`.
  2. Resolves Docker Secret mounted at `/run/secrets/<key>`.
  3. Falls back to standard environment variable `os.getenv(key)`.

```python
def get_secret(key, default=None):
    file_env_var = f"{key}_FILE"
    file_path = os.getenv(file_env_var)
    if file_path and os.path.exists(file_path):
        with open(file_path, "r", encoding="utf-8") as fp:
            return fp.read().strip()

    docker_secret_path = Path("/run/secrets") / key.lower()
    if docker_secret_path.exists():
        with open(docker_secret_path, "r", encoding="utf-8") as fp:
            return fp.read().strip()

    return os.getenv(key, default)
```

### Database Migration & Seeding CLI (`backend/scripts/init_db.py`)
- Idempotently creates the `schema_migrations` audit table.
- Executes base schema creation (`CREATE TABLE IF NOT EXISTS`).
- Seeds initial task entries (`Verify Multi-Container Orchestration`, `Configure Named Volume Persistence`, `Enforce Enterprise Quality Gates`) only if the table is empty.

### Durability Verification Canary (`backend/scripts/verify_persistence.py`)
- CLI utility writing unique UUID canary markers to PostgreSQL and Redis.
- Allows operators to verify data survival across container restarts or teardowns.

---

## 3. Verification & Quality Gates Report

The durability engine and secrets resolution pipelines were validated using automated tests:

| Test Suite Module | Target Scope | Tests Passed | Status |
| :--- | :--- | :---: | :---: |
| `tests/test_persistence_and_secrets.py` | Secret resolution (`_FILE`, Docker Secrets), canaries | 10 | Passed |
| `tests/test_compose_and_nginx.py` | Compose schema, Nginx configs, DB retry loops | 4 | Passed |
| `tests/test_dockerfile_standards.py` | Multi-stage builder/runtime standards | 6 | Passed |
| `tests/test_db_and_cache.py` | Database and cache adapter operations | 12 | Passed |
| `tests/test_health_and_version.py` | Container health and readiness endpoints | 6 | Passed |
| `tests/test_task_routes.py` | Task management APIs and benchmarks | 8 | Passed |

### Quality Gates Execution Summary
- **Total Tests:** 46 automated tests passing in 1.18 seconds.
- **Branch Coverage:** 94.78% total branch coverage.
- **Linters:** Flake8, Black, isort, and Bandit reported 0 errors.

---

## 4. Key Engineering Insights & Decisions

- **Container State vs. Volume State:** Treating containers as disposable compute nodes enforces clean architecture where persistence is entirely decoupled into storage volumes.
- **Secrets via File Paths:** Using `<KEY>_FILE` environment variables provides broad compatibility across Kubernetes secrets, Docker Swarm secrets, and local secret files without code changes.
- **Container Resource Quotas in Production:** Unconstrained containers can monopolize host CPU and memory during query surges. Hard caps in production overlays ensure predictable cluster stability.

---

## 5. Next Steps

- Build a responsive vanilla ES6 frontend operations dashboard (**Docker Pulse**) connecting to Nginx and Flask endpoints.
- Implement live 5-second health polling, comparative PostgreSQL vs. Redis latency benchmarking widgets, and interactive task controls.
