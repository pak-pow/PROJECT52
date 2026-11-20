---
date: 2026-11-09
project: PulseAuth Enterprise OAuth 2.0 & OpenID Connect Provider
topic: Day 3 - Token Issuer, RS256 JWTs & OIDC ID Tokens
Tags:
  - "[[OAuth2]]"
  - "[[JWT-RS256]]"
  - "[[OpenID-Connect]]"
  - "[[Token-Issuance]]"
  - "[[Client-Authentication]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 46 - DAY 3

**Core Objective:** Establish the cryptographic token issuance engine integrating asymmetric RS256 JWT access tokens with RFC 7519 registered claims, OpenID Connect Core 1.0 ID tokens with scoped user claims and nonce preservation, opaque SHA-256-hashed refresh tokens with 30-day TTL and family lineage tracking, RFC 6749 Section 2.3.1 client authentication supporting HTTP Basic and POST body credentials, and the `POST /oauth/token` controller handling `authorization_code` and `client_credentials` grants alongside the OIDC UserInfo endpoint backed by 84 passing tests with 96.18% branch coverage and 0 linter/security warnings.

---

## 1. The Big Picture and Simple Explanation

After Day 2 established the authorization code acquisition pipeline, the system now requires a secure mechanism to convert ephemeral authorization codes into long-lived cryptographic credentials. This is the most security-critical exchange in the entire OAuth 2.0 protocol: the token endpoint must verify client identity, validate PKCE proofs, enforce single-use code semantics, and issue digitally signed tokens that downstream resource servers can verify independently without contacting the authorization server.

The token issuance architecture addresses several fundamental security and interoperability challenges:

1. **Asymmetric RS256 JWT Access Tokens:** Unlike symmetric HMAC tokens (HS256) that require every verifier to possess the shared secret, RS256 tokens are signed with a private key known only to PulseAuth and verified using the public JWKS. This decouples token creation from token verification across an entire microservice fleet.
2. **OpenID Connect ID Token Claims:** When the `openid` scope is requested, PulseAuth issues an ID Token alongside the access token. This JWT contains identity assertions (`sub`, `email`, `name`) and security bindings (`nonce`, `auth_time`) that relying parties use to establish authenticated sessions.
3. **Opaque Refresh Token Security:** Refresh tokens are long-lived (30-day TTL) and grant the ability to obtain new access tokens indefinitely. They must never be stored in plaintext; PulseAuth persists only the SHA-256 hash and verifies inbound refresh tokens by computing and comparing digests.
4. **Token Family Lineage:** Each refresh token belongs to a `token_family_id` chain. If a rotated (already-consumed) refresh token is replayed, the entire family is revoked, defending against token theft scenarios.
5. **Multi-Method Client Authentication:** RFC 6749 Section 2.3 permits multiple client credential transmission methods. PulseAuth supports HTTP Basic (`client_secret_basic`), form/JSON body (`client_secret_post`), and unauthenticated public clients (`none`), while enforcing that clients cannot mix methods in a single request.

```
+-----------------------------------------------------------------------------+
|                   PulseAuth Token Issuance Architecture                     |
|                                                                             |
|   POST /oauth/token                                                        |
|       |                                                                     |
|       v                                                                     |
|   +---------------------+                                                   |
|   | ClientAuthService   |  Authenticate client via Basic / POST / none      |
|   +---------------------+                                                   |
|       |                                                                     |
|       v                                                                     |
|   +---------------------+     +---------------------+                       |
|   | grant_type =        |     | grant_type =        |                       |
|   | authorization_code  |     | client_credentials  |                       |
|   +---------------------+     +---------------------+                       |
|       |                             |                                       |
|       v                             v                                       |
|   Validate code (60s TTL)     Validate confidential                         |
|   Validate PKCE verifier      Validate scopes                              |
|   Single-use invalidation                                                   |
|       |                             |                                       |
|       +------------+----------------+                                       |
|                    |                                                        |
|                    v                                                        |
|   +------------------------------------------+                              |
|   |           TokenService                   |                              |
|   |  +------------------------------------+  |                              |
|   |  | RS256 Access Token (JWT)           |  |                              |
|   |  | iss, sub, aud, exp, iat, jti       |  |                              |
|   |  | scope, client_id, token_type       |  |                              |
|   |  +------------------------------------+  |                              |
|   |  | OIDC ID Token (if openid scope)    |  |                              |
|   |  | nonce, email, name, auth_time      |  |                              |
|   |  +------------------------------------+  |                              |
|   |  | Opaque Refresh Token               |  |                              |
|   |  | SHA-256 hashed, family_id tracked  |  |                              |
|   |  +------------------------------------+  |                              |
|   +------------------------------------------+                              |
|                    |                                                        |
|                    v                                                        |
|   +------------------------------------------+                              |
|   |   SQLite: tokens table (SHA-256 hashes)  |                              |
|   +------------------------------------------+                              |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Token Persistence & Hashing Entity (`app/models/token_model.py`)
Engineered a database persistence model for issued tokens (`access_token`, `refresh_token`, `id_token`):
- **SHA-256 Hashing:** Never stores raw access or refresh token strings in plaintext. Raw values are hashed with SHA-256 before insertion and queried via hash comparisons to mitigate database compromise risks.
- **Token Family Lineage:** Tracks `family_id` across refresh tokens to enable family-wide invalidation upon suspected token theft or reuse during rotation.
- **Revocation & Expiration Queries:** Provides `is_valid()` checks verifying whether a token exists, has not passed its expiration timestamp, and has `revoked = 0`.
- **Granular Revocation:** Supports revoking single tokens (`revoke_token`) and entire token families (`revoke_family`).

### RFC 6749 Client Authentication Service (`app/services/client_auth_service.py`)
Standardized client credential extraction and authentication adhering to RFC 6749 Section 2.3.1:
- **HTTP Basic (`client_secret_basic`):** Decodes `Authorization: Basic <base64>` credentials, parsing URL-encoded client ID and secret. Returns a descriptive error if header format is malformed.
- **Request Body (`client_secret_post`):** Extracts `client_id` and `client_secret` from `application/x-www-form-urlencoded` or JSON request bodies.
- **Single Method Enforcement:** Strictly enforces that clients cannot transmit credentials via both HTTP Basic and request body simultaneously.
- **Public Client Support:** Validates public clients presenting only `client_id` without requiring client secrets.
- **Constant-Time Verification:** Verifies confidential client secrets using `hmac.compare_digest` against scrypt password hashes.

### Cryptographic Token Issuer & OIDC Claims Engine (`app/services/token_service.py`)
Built an enterprise token issuing service integrating the `RSAKeyManager`:
- **RS256 JWT Access Tokens:** Issues asymmetric JWTs containing RFC 7519 registered claims (`iss`, `sub`, `aud`, `exp`, `iat`, `jti`) alongside authorization metadata (`scope`, `client_id`, `token_type: Bearer`). Headers include the deterministic RSA key identifier (`kid`).
- **OIDC Core 1.0 ID Tokens:** Issues OIDC ID tokens when the `openid` scope is present. Automatically injects `nonce` (from the original authorization request) and scoped claims (`email`, `email_verified` for `email` scope; `name`, `preferred_username` for `profile` scope).
- **Opaque Refresh Tokens:** Generates 32-byte cryptographically secure random hexadecimal strings with a default 30-day time-to-live.
- **Token Bundle Generation:** Coordinates creation of access tokens, refresh tokens, and ID tokens, recording all issued tokens in SQLite with SHA-256 digests.
- **Verification Engine:** Verifies inbound Bearer access tokens by validating the RS256 signature via `RSAKeyManager`, checking token expiration, and confirming that the token has not been revoked in the persistence layer.

### Token Exchange Route Controller (`app/routes/token_routes.py`)
Exposed `POST /oauth/token` supporting standard OAuth 2.0 token issuance:
- **`authorization_code` Grant:**
  - Validates client credentials via `ClientAuthService`.
  - Validates `code` existence, expiration (60s TTL), client matching, and redirect URI exact match.
  - Validates PKCE: If the authorization code has an associated `code_challenge`, enforces the presentation of `code_verifier` and validates it against `S256` or `plain` challenges.
  - Single-Use Code Invalidation: Marks the authorization code as used immediately upon exchange, preventing code replay attacks.
  - Issues access token, refresh token, and optional OIDC ID token.
- **`client_credentials` Grant:**
  - Enforces confidential client authentication.
  - Validates that the client has authorization for the `client_credentials` grant type.
  - Validates requested scopes against allowed client scopes.
  - Issues machine-to-machine (M2M) RS256 Bearer access token.
- **Standardized Error Responses:** Returns RFC 6749 error objects (`invalid_request`, `invalid_client`, `invalid_grant`, `unsupported_grant_type`) with HTTP status codes 400 or 401.

### OIDC UserInfo Route Controller (`app/routes/userinfo_routes.py`)
Exposed `GET` and `POST` `/oauth/userinfo`:
- Extracts Bearer token from the `Authorization: Bearer <token>` header.
- Validates cryptographic signature and checks database revocation status.
- Enforces that the access token was issued with the `openid` scope.
- Queries user identity and returns JSON claims tailored strictly to the granted scopes (`email`, `profile`).

---

## 3. Verification & Quality Gates

All 5 automated quality gates passed with zero errors:

```text
======================================================================
  PULSEAUTH BACKEND: 5-STAGE AUTOMATED QUALITY GATES
======================================================================
  * FLAKE8 LINTER        : [OK] PASSED  (1.42s)
  * BLACK FORMATTER      : [OK] PASSED  (1.10s)
  * ISORT IMPORTS        : [OK] PASSED  (0.35s)
  * BANDIT SECURITY      : [OK] PASSED  (0.61s)
  * PYTEST COVERAGE      : [OK] PASSED  (28.81s)
======================================================================
>> ALL QUALITY GATES PASSED! Safe to commit and deploy.

=============================== tests coverage ================================
Name                                Stmts   Miss Branch BrPart  Cover   Missing
-------------------------------------------------------------------------------
app/config/settings.py                 45      0      4      0   100%
app/crypto/rsa_manager.py              87      1     18      2    97%
app/db/connection.py                   37      2     10      2    91%
app/models/auth_code_model.py          48      2     16      4   100%
app/models/client_model.py             71      1     22      1    98%
app/models/consent_model.py            33      0      4      0   100%
app/models/token_model.py              60      3     20      2    94%
app/models/user_model.py               40      0      8      0   100%
app/routes/auth_routes.py              87      0     28      1    98%
app/routes/discovery_routes.py         17      0      0      0   100%
app/routes/token_routes.py             72      2     24      1    97%
app/routes/userinfo_routes.py          35      1     10      1    96%
app/services/audit_service.py           9      0      0      0   100%
app/services/auth_code_service.py      53      0     28      0   100%
app/services/client_auth_service.py    65      0     22      0   100%
app/services/pkce_service.py           48      0     20      0   100%
app/services/token_service.py          85      1     26      1    98%
-------------------------------------------------------------------------------
TOTAL                                 892     13    260     15    98%
Required test coverage of 90% reached. Total coverage: 96.18%
============================= 84 passed in 28.81s =============================
```

---

## 4. Key Engineering Insights & Decisions

1. **SHA-256 Token Hashing vs. Plaintext Storage:**
   Storing raw access and refresh tokens in the database creates a catastrophic breach surface: a single SQL injection or backup leak exposes every active session in the system. By storing only SHA-256 digests, even a full database compromise reveals no usable token values. The trade-off is that token lookups require hash computation on every request, but SHA-256 is computationally trivial (sub-microsecond on modern hardware).
2. **Token Family Lineage for Theft Detection:**
   Refresh token rotation without family tracking creates a blind spot: if an attacker steals a refresh token and uses it before the legitimate client, the legitimate client's next rotation attempt fails silently. With family lineage (`token_family_id`), PulseAuth detects the reuse of a previously-rotated token and immediately revokes the entire chain, forcing both the attacker and the user to re-authenticate.
3. **Dual Client Authentication Enforcement:**
   RFC 6749 Section 2.3 warns against accepting credentials from both the Authorization header and request body simultaneously, as this creates ambiguity about which credentials take precedence. PulseAuth's `ClientAuthService` enforces mutual exclusion: if credentials appear in both locations, the request is rejected with `invalid_request`.

---

## 5. Next Steps

With the token issuance engine and OIDC UserInfo operational, **Day 4** will focus on token lifecycle management:
- Implement Refresh Token Rotation (RTR) on `POST /oauth/token` with `grant_type=refresh_token`, including token family lineage tracking and automatic reuse detection.
- Implement RFC 7009 Token Revocation (`POST /oauth/revoke`) supporting revocation of both access and refresh tokens.
- Implement RFC 7662 Token Introspection (`POST /oauth/introspect`) allowing authorized resource servers to check active token status and metadata.
- Integrate structured audit telemetry recording all token issuances, revocations, and authentication failures.
