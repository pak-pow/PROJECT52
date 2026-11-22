---
date: 2026-10-05
project: GraphQL API Server
topic: Day 1 - Architecture Scaffolding and Directory Setup
Tags:
  - "[[GraphQL]]"
  - "[[Backend]]"
  - "[[Schema-Design]]"
  - "[[Architecture]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 41 - DAY 1

**Core Objective:** Kick off Week 41 (GraphQL API Server - PulseGraph) by defining domain models, planning the 7-day implementation roadmap, and scaffolding the complete backend and interactive explorer directory tree with 31 clean baseline files.

---

## 1. The Big Picture and Simple Explanation

In traditional REST APIs, client applications frequently encounter two fundamental inefficiencies:
1. **Over-Fetching:** The backend returns an extensive 50-field JSON document when a mobile client only requires an author's name and avatar URL, wasting network bandwidth.
2. **Under-Fetching:** Displaying a single project page requires four consecutive round-trips (`/api/projects/1`, `/api/users/5`, `/api/projects/1/reviews`, `/api/projects/1/technologies`), multiplying request latency over cellular networks.

**GraphQL** resolves both issues by establishing a single endpoint (`/graphql`) backed by a strongly typed schema. Clients submit declarative queries requesting the exact fields and nested relationships required, and the server returns that exact shape in a single payload.

In Day 1, we scaffolded **PulseGraph**, a production-grade GraphQL API server for developer portfolios, projects, and peer reviews:
- **Strongly Typed Schema:** Defines explicit Object Types, Enums, Interfaces, and operations.
- **Relational Resolvers & DataLoaders:** Traverses nested entities while batching SQL lookups to eliminate the N+1 query problem.
- **Query Protection:** Enforces AST-based depth and complexity guards to block denial-of-service query attacks.
- **Interactive Workbench:** An in-browser developer client and performance profiling dashboard.

```
+-----------------------------------------------------------------------------+
|                     PulseGraph GraphQL Foundation Architecture              |
|                                                                             |
|      +---------------------------------------------------------------+      |
|      |             Client Browser / Interactive Explorer UI          |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v [POST /graphql (Query Payload)]      |
|      +---------------------------------------------------------------+      |
|      | Flask GraphQL Gateway Route (app/routes/graphql_routes.py)    |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | AST Query Depth & Complexity Guard (app/graphql/protection)   |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Graphene Executable Schema (app/graphql/schema.py)            |      |
|      | - Root Query Resolvers (queries.py)                           |      |
|      | - Root Mutation Resolvers (mutations.py)                      |      |
|      | - Type Definitions & Interfaces (types.py)                    |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Request-Scoped DataLoader Registry (app/graphql/dataloaders)  |      |
|      | - Batched and memoized in-memory resolution                   |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | SQLite 3 Relational Database (pulsegraph.db)                  |      |
|      | - users, technologies, projects, project_techs, reviews       |      |
|      +---------------------------------------------------------------+      |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Directory Tree Scaffolding
Scaffolded 31 modular files across 11 packages:

```text
week41_graphql_server/
+-- backend/
|   +-- app/
|   |   +-- config/
|   |   |   +-- __init__.py
|   |   |   +-- settings.py
|   |   +-- graphql/
|   |   |   +-- __init__.py
|   |   |   +-- dataloaders.py
|   |   |   +-- mutations.py
|   |   |   +-- queries.py
|   |   |   +-- schema.py
|   |   |   +-- types.py
|   |   +-- models/
|   |   |   +-- __init__.py
|   |   |   +-- project_model.py
|   |   |   +-- user_model.py
|   |   +-- routes/
|   |   |   +-- __init__.py
|   |   |   +-- graphql_routes.py
|   |   |   +-- health_routes.py
|   |   +-- __init__.py
|   |   +-- db.py
|   +-- data/
|   |   +-- schema.sql
|   |   +-- seed.py
|   +-- scripts/
|   |   +-- run_quality_checks.py
|   +-- tests/
|   |   +-- __init__.py
|   |   +-- conftest.py
|   |   +-- test_graphql_mutations.py
|   |   +-- test_graphql_queries.py
|   |   +-- test_health_routes.py
|   +-- requirements.txt
|   +-- run.py
+-- frontend/
    +-- public/
    |   +-- index.html
    +-- src/
        +-- assets/
        |   +-- base.css
        |   +-- explorer.css
        +-- utils/
        |   +-- helpers.js
        +-- explorer.js
```

### Module Responsibilities
- **GraphQL Engine (`backend/app/graphql/`):** Houses schema definition (`schema.py`), type representations (`types.py`), root query resolvers (`queries.py`), mutation handlers (`mutations.py`), and batch loaders (`dataloaders.py`).
- **Data Access Models (`backend/app/models/`):** Manages relational queries and data transformations for developer profiles and showcase projects.
- **HTTP Routing Layer (`backend/app/routes/`):** Dispatches requests to the `/graphql` execution pipeline and provides system health telemetry (`/health`).
- **Frontend Explorer Workbench (`frontend/`):** Provides a zero-dependency interactive client with query templates, synchronized line-numbered editor, and telemetry visualization.

---

## 3. Verification & Quality Gates Report

The baseline scaffolding was verified through initial test executions and static analysis:

| Test Suite Module | Target Scope | Tests Passed | Status |
| :--- | :--- | :---: | :---: |
| `tests/test_health_routes.py` | Liveness, readiness, and version probes | 3 | Passed |

### Quality Gates Execution Summary
- **Flake8:** 0 lint errors (88-character line limit enforced).
- **Black:** 100% compliant formatting.
- **isort:** Import sorting verified cleanly.
- **Bandit:** 0 security vulnerabilities identified.
- **Pytest:** Baseline suite passing with zero failures.

---

## 4. Key Engineering Insights & Decisions

- **Scaffolding DataLoaders from Day 1:** In GraphQL applications, failing to architect DataLoaders early makes it difficult to retrofit batching into nested resolvers later.
- **Separation of Types from Resolvers:** Defining types in `types.py` independently of query and mutation resolvers prevents circular import issues as schema complexity grows.
- **Zero-Dependency Explorer:** Building a custom in-browser workbench ensures fast offline access without relying on large third-party bundles like GraphiQL or Apollo Studio.

---

## 5. Next Steps

- Design and implement the GraphQL Type System (Object Types, Enums, Interfaces, Input Types).
- Author the SQLite relational schema and seeder for users, projects, technologies, and peer reviews.
