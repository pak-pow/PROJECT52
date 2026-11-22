---
date: 2026-10-07
project: GraphQL API Server
topic: Day 3 - GraphQL Mutations and Business Logic
Tags:
  - "[[GraphQL]]"
  - "[[Mutations]]"
  - "[[Business-Logic]]"
  - "[[Backend]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 41 - DAY 3

**Core Objective:** Implement GraphQL Mutations and business logic for PulseGraph, providing write operations for developer profiles, showcase projects, peer reviews, and stars with SQLite persistence, relational traversal resolvers, and automated test coverage passing 5/5 quality gates.

---

## 1. The Big Picture and Simple Explanation

In Day 2, we established the GraphQL read path (queries, types, enums, interfaces). However, an API that only reads data cannot support user interaction. Today, we built the write path using **GraphQL Mutations**.

In REST APIs, modifying data usually requires multiple endpoints with different HTTP verbs (`POST /api/users`, `PUT /api/projects/:id`, `DELETE /api/users/:id`, `POST /api/reviews`). In GraphQL, all state changes are orchestrated through a unified `Mutation` root type:
1. **Explicit Structured Payloads:** Every mutation receives an input object and returns a structured payload containing the modified entity, a boolean `success` flag, and a human-readable `message`. This avoids ambiguous or silent failures.
2. **Atomic Relational Operations:** Creating a project automatically attaches tagged technologies in a junction table (`project_technologies`). Submitting a review automatically triggers project rating recalculations.
3. **Deep Relational Traversals:** Because mutations return rich GraphQL types, a client can create a project and immediately query its owner, tagged technologies, and reviews in a single request without extra network round-trips.

```
+-----------------------------------------------------------------------------+
|                     GraphQL Mutation Architecture Flow                      |
|                                                                             |
|      +---------------------------------------------------------------+      |
|      | Client Request: mutation { createProject(input: {...}) {...} }|      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Root Mutation Handler (app/graphql/mutations.py)              |      |
|      | - Dispatches to matching Mutation class                       |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|         +----------------------------+----------------------------+         |
|         |                            |                            |         |
|         v                            v                            v         |
|  +--------------+             +--------------+             +--------------+ |
|  | User         |             | Project      |             | Review & Star| |
|  | Mutations    |             | Mutations    |             | Mutations    | |
|  +-------+------+             +-------+------+             +-------+------+ |
|          |                            |                            |        |
|          v                            v                            v        |
|  +--------------+             +--------------+             +--------------+ |
|  | user_model.py|             | project_model|             | project_model| |
|  +-------+------+             +-------+------+             +-------+------+ |
|          |                            |                            |        |
|          +----------------------------+----------------------------+        |
|                                       |                                     |
|                                       v                                     |
|                      +-------------------------------+                      |
|                      | SQLite 3 Relational Database  |                      |
|                      | - users, projects, reviews    |                      |
|                      +-------------------------------+                      |
|                                       |                                     |
|                                       v                                     |
|      +---------------------------------------------------------------+      |
|      | Return Structured Payload: { success, message, project {...} }|      |
|      +---------------------------------------------------------------+      |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Business Logic Models (`backend/app/models/`)
- User Model (`user_model.py`):
  - Validates non-empty usernames, emails, and system roles (`DEVELOPER`, `MAINTAINER`, `ADMIN`).
  - Parameterized SQLite queries preventing SQL injection vulnerabilities.
  - CRUD routines: `create_user`, `get_user_by_id`, `update_user`, and `delete_user` (with foreign key cascading).
- Project Model (`project_model.py`):
  - Project lifecycle: `create_project`, `get_project_by_id`, `list_projects`, `update_project`, `delete_project`, `star_project`.
  - Multi-parameter filtering in `list_projects` supporting keywords, status, owner ID, technology ID, and minimum average rating.
  - Peer reviews in `create_review`: strict 1 to 5 integer validation and existence checks.
  - Rating metrics: recalculates `average_rating` and `review_count` using SQL group functions.

### GraphQL Mutations Layer (`backend/app/graphql/mutations.py`)
- Registered 8 mutations on the root `Mutation` class:
  - `createUser`: Registers new developers with duplicate detection.
  - `updateUser`: Modifies bio and permissions.
  - `deleteUser`: Removes profile and cascades related data.
  - `createProject`: Publishes showcase projects with tagged technology IDs.
  - `updateProject`: Modifies title, status, or technology stack.
  - `deleteProject`: Deletes projects and cascades reviews.
  - `starProject`: Increments repository stars.
  - `createReview`: Posts peer reviews with rating validation.

```python
class CreateProject(graphene.Mutation):
    class Arguments:
        input = ProjectInput(required=True)

    success = graphene.Boolean(required=True)
    message = graphene.String(required=True)
    project = graphene.Field("app.graphql.types.ProjectType")

    def mutate(self, info, input):
        project = ProjectModel.create_project(input)
        return CreateProject(
            success=True,
            message="Project created successfully",
            project=project
        )
```

### Relational Field Resolvers (`backend/app/graphql/types.py`)
- `ProjectType`: `resolve_owner`, `resolve_technologies`, `resolve_reviews`.
- `UserType`: `resolve_projects`, `resolve_reviews`, `resolve_project_count`.
- `ReviewType`: `resolve_author`, `resolve_project`.

---

## 3. Verification & Quality Gates Report

The mutations and relational resolvers were verified across 22 automated test scenarios:

| Test Suite Module | Target Scope | Tests Passed | Status |
| :--- | :--- | :---: | :---: |
| `tests/test_graphql_mutations.py` | Mutation executions, relational traversals, errors | 22 | Passed |
| `tests/test_graphql_queries.py` | Query resolvers, filters, SDL generation | 11 | Passed |
| `tests/test_health_routes.py` | Liveness, readiness, and version probes | 3 | Passed |

### Quality Gates Execution Summary
- **Total Tests:** 36 automated tests passing in 0.72 seconds.
- **Branch Coverage:** 95.33% total branch coverage.
- **Linters:** Flake8, Black, isort, and Bandit reported 0 errors.

---

## 4. Key Engineering Insights & Decisions

- **Structured Payloads Prevent Ambiguity:** Returning `{ success, message, entity }` from every mutation gives frontend clients predictable error handling and facilitates optimistic UI updates.
- **Atomic Junction Table Writes:** Attaching technologies to projects inside a single database transaction guarantees data consistency and eliminates orphaned records.
- **Relational Field Resolvers Enable Single-Trip Workflows:** Returning rich GraphQL types from mutations allows clients to create an entity and inspect its resolved relationships in a single operation.

---

## 5. Next Steps

- Implement request-scoped DataLoaders to eliminate the N+1 database query problem.
- Introduce AST-based query depth and complexity safety validation to protect against Denial of Service attacks.
