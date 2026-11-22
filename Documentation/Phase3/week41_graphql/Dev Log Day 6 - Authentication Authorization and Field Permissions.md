---
date: 2026-10-10
project: GraphQL API Server
topic: Day 6 - Authentication, Authorization, and Field Permissions
Tags:
  - "[[GraphQL]]"
  - "[[Authentication]]"
  - "[[Authorization]]"
  - "[[RBAC]]"
  - "[[Security]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 41 - DAY 6

**Core Objective:** Implement a stateless HMAC-SHA256 authentication engine, Role-Based Access Control (RBAC) directives, field-level permissions for PII email redaction, and ownership guardrails on project mutations passing all quality gates.

---

## 1. The Big Picture and Simple Explanation

In traditional REST architectures, authentication and authorization are typically enforced at the endpoint URL level (such as `/api/admin/*` or `/api/user/settings`). In GraphQL, however, the entire API is accessed through a single endpoint (`/graphql`). Clients can request any combination of fields, queries, and nested relations in a single HTTP POST request.

This introduces unique security requirements:
1. **Single Entry Point Authentication:** The API gateway decodes credentials (`Authorization: Bearer <token>` or `X-User-Id`) and injects the resolved `current_user` into the request-scoped GraphQL execution context.
2. **Operation-Level Authorization (RBAC):** Root query fields and mutations verify that the requester possesses required roles (`DEVELOPER`, `MAINTAINER`, `ADMIN`) or resource ownership before executing operations.
3. **Field-Level Permissions & PII Protection:** Sensitive entity attributes (such as developer email addresses) must not be exposed to arbitrary third parties. The type system evaluates permissions at the individual field resolver level, redacting or returning full values dynamically based on requester identity.

```
+-----------------------------------------------------------------------------+
|                     GraphQL Authentication & Authorization Pipeline         |
|                                                                             |
|      +---------------------------------------------------------------+      |
|      |             Client HTTP Request (POST /graphql)               |      |
|      |             Header: Authorization: Bearer <HMAC_Token>        |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Gateway Authentication (app/graphql/auth.py)                  |      |
|      | - Verifies HMAC-SHA256 signature in constant time             |      |
|      | - Resolves active developer record from database              |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | GraphQL Execution Context: { current_user, dataloaders }      |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|         +----------------------------+----------------------------+         |
|         |                            |                            |         |
|         v                            v                            v         |
|  +--------------+             +--------------+             +--------------+ |
|  | Operation    |             | Ownership    |             | Field-Level  | |
|  | RBAC Check   |             | Verification |             | PII Masking  | |
|  | @require_role|             | owner_id ==  |             | UserType     | |
|  | ADMIN only   |             | current_user |             | resolve_email| |
|  +-------+------+             +-------+------+             +-------+------+ |
|          |                            |                            |        |
|          v                            v                            v        |
|  +--------------+             +--------------+             +--------------+ |
|  | adminUsers   |             | updateProject|             | Public: a**x | |
|  | Query Guard  |             | deleteProject|             | Self/Admin:  | |
|  |              |             |              |             | Full Email   | |
|  +--------------+             +--------------+             +--------------+ |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Stateless HMAC-SHA256 Token Engine (`backend/app/graphql/auth.py`)
- Token generation (`generate_auth_token`): Creates signed tokens containing user ID (`sub`), `username`, `role`, issue timestamp (`iat`), and expiration (`exp`).
- Token validation (`decode_auth_token`): Validates signatures using constant-time comparison (`hmac.compare_digest`) and verifies expiration timestamps.
- Permission decorators:
  - `@require_auth`: Enforces active authenticated session in context.
  - `@require_role(allowed_roles)`: Validates role membership before invoking resolvers.
- Email redaction helper (`redact_email`): Obfuscates local parts for unauthenticated queries (`alex@example.com` becomes `a***x@example.com`).

```python
def redact_email(email):
    if not email or "@" not in email:
        return email
    local, domain = email.split("@", 1)
    if len(local) <= 2:
        return f"{local[0]}***@{domain}"
    return f"{local[0]}***{local[-1]}@{domain}"
```

### Field-Level Permissions on `UserType.resolve_email` (`types.py`)
- Evaluates requester identity dynamically:
  - If the requester is the profile owner (`current_user.id == self.id`) or an `ADMIN`, the raw email is returned.
  - For unauthenticated requests or other developers, the email is automatically redacted.

### RBAC Queries and Ownership Verification
- `viewer` query: Returns the currently authenticated developer profile from context.
- `adminUsers` query: Restricted strictly to clients with the `ADMIN` role; unauthorized calls return structured `GraphQLError`.
- Mutation ownership guardrails: Restricts `updateProject` and `deleteProject` to project owners or system administrators.

---

## 3. Verification & Quality Gates Report

The authentication engine, RBAC rules, and field permissions were validated across automated tests:

| Test Suite Module | Target Scope | Tests Passed | Status |
| :--- | :--- | :---: | :---: |
| `tests/test_graphql_auth.py` | HMAC tokens, RBAC roles, field redaction, ownership | 11 | Passed |
| `tests/test_graphql_dataloaders.py` | DataLoader batching, memoization, depth/complexity | 8 | Passed |
| `tests/test_graphql_mutations.py` | Mutation executions, relational traversals, errors | 22 | Passed |
| `tests/test_graphql_queries.py` | Query resolvers, filters, SDL generation | 11 | Passed |
| `tests/test_health_routes.py` | Liveness, readiness, and version probes | 3 | Passed |

### Quality Gates Execution Summary
- **Total Tests:** 55 automated tests passing in 1.15 seconds.
- **Branch Coverage:** 94.8% total branch coverage.
- **Linters:** Flake8, Black, isort, and Bandit reported 0 errors.

---

## 4. Key Engineering Insights & Decisions

- **Field-Level Resolvers Protect PII Dynamically:** Enforcing permissions at the field resolver layer rather than the endpoint level ensures that email addresses cannot be leaked through nested relational queries.
- **Constant-Time HMAC Comparisons:** Using `hmac.compare_digest` protects against timing attacks that attempt to brute-force signature bytes.
- **Stateless Tokens Simplify Scaling:** Embedding role and user ID directly in signed HMAC tokens removes database lookups during token validation.

---

## 5. Next Steps

- Implement the comprehensive end-to-end integration and security test suite (`test_graphql_e2e.py`).
- Complete technical documentation (`week41_graphql_server/README.md`) and finalize handover.
