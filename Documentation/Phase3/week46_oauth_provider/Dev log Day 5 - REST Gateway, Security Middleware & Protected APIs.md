---
date: 2026-11-11
project: PulseAuth Enterprise OAuth 2.0 & OpenID Connect Provider
topic: Day 5 - REST Gateway, Security Middleware & Protected APIs
Tags:
  - "[[OAuth2]]"
  - "[[Bearer-Token]]"
  - "[[Security-Middleware]]"
  - "[[Resource-Server]]"
  - "[[Client-Management]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 46 - DAY 5

**Core Objective:** Build the resource server security middleware decorator (`@require_oauth_token`) enforcing RFC 6750 Bearer token parsing with RS256 public key verification, real-time database revocation checks, and dual-mode scope authorization policies, construct protected resource endpoints modeling real-world API delegation, implement developer client management APIs with dynamic registration and secret rotation per RFC 7591, and build user authentication and registration APIs with scrypt password hashing backed by 126 passing tests with 96.07% branch coverage and 0 linter/security warnings.

---

## 1. The Big Picture and Simple Explanation

An OAuth 2.0 authorization server is only half the security architecture. The other half is the resource server: the downstream APIs that accept Bearer tokens, verify their authenticity, and enforce scope-based access control. Without a robust security middleware layer, every protected endpoint must independently parse authorization headers, validate cryptographic signatures, check revocation status, and evaluate scope requirements. This duplication creates inconsistency, missed checks, and security gaps.

Day 5 addresses the resource server protection model and developer self-service administration:

1. **Reusable Security Middleware:** A single `@require_oauth_token` decorator encapsulates the entire Bearer token verification pipeline. Route handlers declare their required scopes declaratively and receive a pre-verified, pre-hydrated request context with the authenticated user identity.
2. **Multi-Layer Token Validation:** Each incoming Bearer token undergoes four sequential checks: (a) RFC 6750 header format extraction, (b) RS256 cryptographic signature verification against the public JWKS, (c) claim validation (issuer, expiration), and (d) real-time database revocation lookup. Failure at any layer immediately short-circuits with the appropriate HTTP status and `WWW-Authenticate` challenge.
3. **Dual Scope Enforcement:** Some endpoints require all listed scopes (`required_scopes`), while others require any one of several scopes (`any_of_scopes`). The middleware supports both modes, returning HTTP 403 `insufficient_scope` for unauthorized calls.
4. **Developer Self-Service:** Modern identity platforms allow developers to register and manage their own OAuth client applications without manual administrator intervention. Dynamic client registration, secret rotation, and deletion APIs enable this self-service model.

```
+-----------------------------------------------------------------------------+
|                   PulseAuth Resource Server Security Model                  |
|                                                                             |
|   Incoming Request                                                          |
|   Authorization: Bearer <RS256 JWT>                                         |
|       |                                                                     |
|       v                                                                     |
|   +---------------------------------------+                                 |
|   |   @require_oauth_token Middleware     |                                 |
|   |                                       |                                 |
|   |   1. Extract Bearer token from header |                                 |
|   |      (RFC 6750 format validation)     |                                 |
|   |                                       |                                 |
|   |   2. RS256 signature verification     |                                 |
|   |      (RSAKeyManager + public JWKS)    |                                 |
|   |                                       |                                 |
|   |   3. Claim validation                 |                                 |
|   |      (iss, exp, iat checks)           |                                 |
|   |                                       |                                 |
|   |   4. Real-time revocation check       |                                 |
|   |      (SHA-256 hash -> tokens table)   |                                 |
|   |                                       |                                 |
|   |   5. Scope authorization              |                                 |
|   |      required_scopes: ALL must match  |                                 |
|   |      any_of_scopes: ONE must match    |                                 |
|   +---------------------------------------+                                 |
|       |                          |                                          |
|       v                          v                                          |
|   HTTP 401/403               request.token_payload = {...}                  |
|   WWW-Authenticate           request.token_sub = "usr_..."                  |
|                              request.token_scopes = {"openid", "profile"}   |
|                              request.db_user = {full user record}           |
|                                  |                                          |
|                                  v                                          |
|                          Route Handler Executes                             |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Security Middleware Decorator (`app/services/token_guard.py`)
Engineered a reusable, standards-compliant `@require_oauth_token` route decorator:
- **RFC 6750 Bearer Parsing:** Extracts and validates the `Authorization: Bearer <token>` header, returning HTTP 401 with standard `WWW-Authenticate` challenge headers on format or credential failures.
- **Asymmetric Signature & Issuer Verification:** Decodes and verifies RS256 JWT tokens using the active `RSAKeyManager` and configured issuer URI.
- **Real-Time Revocation Gating:** Queries SQLite database to ensure the access token has not been marked as revoked via RFC 7009 or refresh family invalidation.
- **Dual Scope Evaluation:** Supports both `required_scopes` (subset requirement where all listed scopes must be present) and `any_of_scopes` (at least one listed scope must be present). Rejects unauthorized calls with HTTP 403 `insufficient_scope`.
- **Context Injection:** Attaches decoded token payload, subject (`sub`), client ID, granted scope set, and resolved database user record directly to Flask's `request` object.

### Protected Resource Server APIs (`app/routes/resource_routes.py`)
Implemented demo resource server endpoints simulating real-world OAuth-protected services:
- `GET /api/v1/resource/profile`: Requires `profile`, `read:profile`, or `openid` scope; returns caller profile data.
- `GET /api/v1/resource/documents`: Requires `read:documents` scope; returns mock confidential security architecture documents.
- `POST /api/v1/resource/documents`: Requires `write:documents` scope; creates and returns new document metadata linked to the caller's subject ID.

### Developer Client Management & Secret Rotation (`app/routes/client_routes.py`)
Implemented developer portal client administration:
- `POST /api/v1/clients`: Dynamically registers new applications, generating unique client identifiers and scrypt-hashed secrets for confidential clients.
- `GET /api/v1/clients`: Lists all registered clients with secret hashes omitted.
- `GET /api/v1/clients/<client_id>`: Fetches individual client configuration details.
- `POST /api/v1/clients/<client_id>/rotate-secret`: Rotates credentials for confidential clients and returns the new plaintext secret; rejects rotation requests on public clients.
- `DELETE /api/v1/clients/<client_id>`: Removes client registrations from the system.

### User Authentication & Registration APIs (`app/routes/user_auth_routes.py`)
Built JSON API endpoints for user identity management:
- `POST /api/v1/auth/login`: Authenticates credentials with constant-time scrypt hash verification.
- `POST /api/v1/auth/register`: Enforces email validation, password length checks (>=8 characters), duplicate prevention, and scrypt hash generation.
- `GET /api/v1/auth/users`: Lists registered user profiles while stripping sensitive password hashes.

### Entity Model Enhancements (`app/models/`)
- Extended `ClientModel` with `rotate_secret()` and `delete()`.
- Extended `UserModel` with `list_all()`.

---

## 3. Verification & Quality Gates

All 5 automated quality gates passed with zero warnings or errors:

```text
======================================================================
  PULSEAUTH BACKEND: 5-STAGE AUTOMATED QUALITY GATES
======================================================================
  * FLAKE8 LINTER        : [OK] PASSED  (1.16s)
  * BLACK FORMATTER      : [OK] PASSED  (0.31s)
  * ISORT IMPORTS        : [OK] PASSED  (0.35s)
  * BANDIT SECURITY      : [OK] PASSED  (1.12s)
  * PYTEST COVERAGE      : [OK] PASSED  (50.60s)
======================================================================
>> ALL QUALITY GATES PASSED! Safe to commit and deploy.

=============================== tests coverage ================================
Name                                Stmts   Miss Branch BrPart  Cover   Missing
-------------------------------------------------------------------------------
app/config/settings.py                 45      0      4      0   100%
app/crypto/rsa_manager.py              87      1     18      2    97%
app/db/connection.py                   37      2     10      2    91%
app/models/auth_code_model.py          48      2     16      4    91%
app/models/client_model.py             71      2     22      2    96%
app/models/consent_model.py            33      0      4      0   100%
app/models/token_model.py              60      5     20      3    90%
app/models/user_model.py               40      0      8      0   100%
app/routes/auth_routes.py              87      0     28      1    99%
app/routes/client_routes.py            55      4     18      3    91%
app/routes/discovery_routes.py         17      0      0      0   100%
app/routes/introspection_routes.py     42      0     14      1    99%
app/routes/resource_routes.py          40      1     12      1    97%
app/routes/revocation_routes.py        38      0     12      0   100%
app/routes/token_routes.py             72      4     24      3    92%
app/routes/user_auth_routes.py         45      0     14      0   100%
app/routes/userinfo_routes.py          35      1     10      1    96%
app/services/audit_service.py           9      0      0      0   100%
app/services/auth_code_service.py      53      0     28      0   100%
app/services/client_auth_service.py    65      0     22      0   100%
app/services/pkce_service.py           48      1     20      1    97%
app/services/refresh_service.py        55      0     18      0   100%
app/services/token_guard.py            60      1     20      1    97%
app/services/token_service.py          85      3     26      2    94%
-------------------------------------------------------------------------------
TOTAL                                1187     27    378     27    96%
Required test coverage of 90% reached. Total coverage: 96.07%
============================= 126 passed in 49.03s ============================
```

---

## 4. Key Engineering Insights & Decisions

1. **Decorator-Based Middleware vs. Flask Before-Request Hooks:**
   Flask's `@app.before_request` applies globally to all routes, requiring opt-out mechanisms for public endpoints. A decorator-based approach (`@require_oauth_token`) inverts this: routes are unprotected by default and explicitly opt-in to token verification with specific scope requirements. This prevents accidental exposure of sensitive endpoints while keeping health checks and discovery endpoints freely accessible.
2. **Real-Time Revocation vs. Token Expiration Alone:**
   Many lightweight implementations rely solely on JWT expiration (`exp` claim) and skip database revocation checks, arguing that short-lived tokens (15 minutes) limit exposure. However, in security-critical environments (banking, healthcare), even 15 minutes of unauthorized access after a revocation event is unacceptable. PulseAuth performs a database lookup on every protected request, trading a few milliseconds of latency for immediate revocation enforcement.
3. **Secret Rotation vs. Client Re-Registration:**
   When a client secret is compromised, the naive response is to delete and re-register the application. This disrupts all existing authorization grants, consent records, and token families. Secret rotation preserves the `client_id` and all associated records while invalidating only the compromised credential, minimizing service disruption.

---

## 5. Next Steps

With the resource server middleware, protected APIs, and developer self-service portal operational, **Day 6** will focus on the interactive frontend:
- Construct the responsive mobile-first web application shell with dark/light theme modes and tabbed navigation.
- Build the Interactive In-Browser OAuth 2.0 & OIDC Playground Console with client-side PKCE generation, authorization code acquisition, token exchange, and RS256 JWT claim inspection.
- Build the Developer Client Management Portal UI for registering applications and rotating secrets.
- Build the User Accounts & Identity Directory UI.
- Build the OpenID Discovery & Cryptographic Keyset Explorer UI.
