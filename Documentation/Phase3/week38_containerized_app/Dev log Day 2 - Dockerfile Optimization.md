---
date: 2026-09-14
project: Containerized App with Docker
topic: Day 2 - Multi-Stage Dockerfile & Container Optimization
Tags:
  - "[[DevOps]]"
  - "[[Docker]]"
  - "[[Security]]"
  - "[[Linux]]"
  - "[[Testing]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 38 - DAY 2

**Core Objective:** Package our backend into a secure, multi-stage production Docker image (`python:3.10-slim`), drop container privileges to an unprivileged non-root user (`appuser`), configure native container health checks, and optimize build context with `.dockerignore`.

---

## 1. The Big Picture and Simple Explanation

When building container images for production, naive approaches place all build compilers, development headers, and temporary files into a single monolithic layer. This results in bloated multi-gigabyte images, slow deployment times, and an expanded attack surface.

In Day 2, we applied modern container packaging principles using multi-stage builds:
1. **Stage 1: The Builder Environment:** Acts as an isolated compilation sandbox. It downloads C build tools (`build-essential`, `libpq-dev`), compiles Python dependencies into standalone `.whl` binaries, and stores them in `/build/wheels`.
2. **Stage 2: The Minimal Runtime:** Discards all compilers and temporary tools. Starting from a pristine `python:3.10-slim` base, it copies only the pre-compiled wheel packages and installs them with zero residual build overhead.
3. **Unprivileged System User (`appuser`, UID 1001):** By default, Docker containers run as `root`. If a container escape occurs, root container processes can compromise host kernel space. We created a dedicated system user and dropped root privileges completely via `USER appuser`.
4. **Native Container Health Probes:** Defined a Docker `HEALTHCHECK` instruction querying `/api/v1/health` every 30 seconds. If the application deadlocks or crashes, container engines mark it unhealthy and initiate remediation.

```
+-----------------------------------------------------------------------------+
|                     Multi-Stage Dockerfile Compilation Flow                 |
|                                                                             |
|      +---------------------------------------------------------------+      |
|      | STAGE 1: BUILDER SANDBOX (python:3.10-slim)                   |      |
|      | 1. Install build tools: gcc, build-essential, libpq-dev       |      |
|      | 2. Compile requirements.txt -> standalone .whl packages       |      |
|      | 3. Output directory: /build/wheels                            |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      | (Transfer ONLY Compiled Wheels)      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | STAGE 2: PRODUCTION RUNTIME (python:3.10-slim)                |      |
|      | 1. Install runtime dependencies only: libpq5, curl            |      |
|      | 2. Create unprivileged system user: appuser (UID 1001)        |      |
|      | 3. Install wheels from /build/wheels & purge directory        |      |
|      | 4. Copy backend source code with chown=appuser:appuser        |      |
|      | 5. Drop root privileges: USER appuser                         |      |
|      | 6. Declare HEALTHCHECK probe on /api/v1/health                |      |
|      | 7. Launch Gunicorn WSGI server (2 workers, 4 threads, :5000)  |      |
|      +---------------------------------------------------------------+      |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Multi-Stage Production Dockerfile (`backend/Dockerfile`)
- Stage separation: Declared `FROM python:3.10-slim AS builder` and `FROM python:3.10-slim AS runtime`.
- Non-root user setup: Executed `useradd --create-home --uid 1001 appuser` and switched execution context using `USER appuser`.
- Web server configuration: Configured Gunicorn WSGI with 2 worker processes and 4 threads bound to `0.0.0.0:5000`.
- Native healthcheck: Baked in `HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 CMD curl -f http://localhost:5000/api/v1/health || exit 1`.

```dockerfile
# Stage 1: Build dependencies
FROM python:3.10-slim AS builder
WORKDIR /build
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip && \
    pip wheel --no-cache-dir --wheel-dir=/build/wheels -r requirements.txt

# Stage 2: Minimal runtime
FROM python:3.10-slim AS runtime
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 curl && rm -rf /var/lib/apt/lists/*
RUN useradd -u 1001 -m -s /bin/bash appuser
COPY --from=builder /build/wheels /wheels
RUN pip install --no-cache-dir /wheels/* && rm -rf /wheels
COPY --chown=appuser:appuser . .
USER appuser
EXPOSE 5000
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:5000/api/v1/health || exit 1
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "--threads", "4", "app:create_app()"]
```

### Build Context Optimization (`backend/.dockerignore`)
- Filtered unnecessary files from the build context: `.git/`, `.venv/`, `__pycache__/`, `.pytest_cache/`, `htmlcov/`, test databases, and local `.env` configuration files.
- Accelerates Docker build times and prevents credential leakage into image layers.

### Automated Docker Standards Test Suite (`tests/test_dockerfile_standards.py`)
- Created 6 unit tests asserting Dockerfile directives programmatically:
  - Multi-stage pattern presence (`AS builder` and `AS runtime`).
  - `USER appuser` directive and unprivileged execution.
  - Active `HEALTHCHECK` instruction and `EXPOSE 5000` declaration.
  - Exclusion list verification across `.dockerignore`.
  - Configuration schema validation in `.env.example`.

---

## 3. Verification & Quality Gates Report

The container specifications were validated using automated static checks and live container execution:

| Test Suite Module | Target Scope | Tests Passed | Status |
| :--- | :--- | :---: | :---: |
| `tests/test_dockerfile_standards.py` | Multi-stage syntax, non-root user, health checks | 6 | Passed |
| `tests/test_db_and_cache.py` | Dual database adapters, query normalization | 12 | Passed |
| `tests/test_health_and_version.py` | Container health and readiness endpoints | 6 | Passed |
| `tests/test_task_routes.py` | Task management APIs and benchmarks | 8 | Passed |

### Quality Gates Execution Summary
- **Total Tests:** 32 automated tests passing in 0.58 seconds.
- **Branch Coverage:** 95.99% total branch coverage.
- **Image Audit:** Image verified running with UID 1001, returning HTTP 200 on health probes.
- **Linters:** Flake8, Black, isort, and Bandit reports 100% clean.

---

## 4. Key Engineering Insights & Decisions

- **Multi-Stage Build Size Reductions:** By excluding build utilities like `gcc` and `libpq-dev` from the final stage, image footprints are cut by more than 60%, significantly improving image pull and deployment speeds.
- **Enforcing Non-Root Execution:** Running containers as root introduces significant security risks in multi-tenant environments. Dropping privileges to UID 1001 ensures container compromise cannot directly manipulate host system resources.
- **Layer Caching Optimization:** Ordering Dockerfile instructions by volatility (installing requirements wheels before copying application code) ensures code modifications reuse cached dependency layers.

---

## 5. Next Steps

- Design a multi-service orchestration topology using Docker Compose (`docker-compose.yml`).
- Integrate Nginx reverse proxy gateway, Flask REST API, PostgreSQL 16 database, and Redis 7 cache.
