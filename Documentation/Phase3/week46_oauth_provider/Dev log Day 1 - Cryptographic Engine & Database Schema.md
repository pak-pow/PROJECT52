---
date: 2026-11-07
project: PulseAuth Enterprise OAuth 2.0 & OpenID Connect Provider
topic: Day 1 - Cryptographic Engine & Database Schema
Tags:
  - "[[OAuth2]]"
  - "[[OpenID-Connect]]"
  - "[[RSA-Keypair]]"
  - "[[JWKS]]"
  - "[[SQLite-WAL]]"
  - "[[Identity-Architecture]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 46 - DAY 1

**Core Objective:** Scaffold the production repository architecture for PulseAuth, construct the asymmetric 2048-bit RSA cryptographic keypair manager (`RSAKeyManager`) with RFC 7517 JWKS export and RFC 8414 OpenID Discovery, and establish the high-concurrency SQLite WAL relational schema for users, registered clients, authorization codes, tokens, consents, and audit logs backed by 20 automated tests passing with 97.49% branch coverage and 0 linter/security warnings.

---

## 1. The Big Picture and Simple Explanation

Modern distributed software ecosystems require delegated access without sharing user credentials. In a legacy architecture, applications often collected raw usernames and passwords, storing them or proxying them across multiple internal and third-party services. This model suffers from catastrophic single-point failures: if a downstream consumer is breached, all user credentials are compromised.

The OAuth 2.0 (RFC 6749) and OpenID Connect (OIDC Core 1.0) protocols resolve this by decoupling authentication (proving *who* you are) from authorization (specifying *what* an application is allowed to do on your behalf):

1. **Centralized Identity & Delegated Authority:** The Authorization Server (**PulseAuth**) acts as the single source of truth. Users authenticate exclusively with PulseAuth; client applications obtain scoped, time-bound cryptographic tokens rather than user credentials.
2. **Asymmetric Cryptography & Decoupled Verification:** By utilizing 2048-bit RSA keys and RS256 digital signatures, PulseAuth can sign JSON Web Tokens (JWTs) using its private key, while downstream resource servers verify token authenticity using only the public JSON Web Key Set (JWKS, RFC 7517). This eliminates network lookups on every API request.
3. **Public Discovery Standards (RFC 8414):** Standards-compliant relying parties discover token endpoints, supported scopes, signing algorithms, and public keys automatically via `/.well-known/openid-configuration`.
4. **Client Heterogeneity & Trust Boundaries:** Applications vary significantly in security capability. *Confidential clients* (e.g., traditional backend servers) can safely hold a `client_secret`, whereas *Public clients* (e.g., Single-Page Applications or Mobile Apps) cannot protect secrets and must enforce Proof Key for Code Exchange (PKCE).
5. **High-Concurrency Relational Persistence:** Authorization servers encounter high-frequency read/write spikes during login surges. SQLite configured in Write-Ahead Logging (WAL) mode provides lock-free concurrent readers alongside isolated serial writes.

```
+-----------------------------------------------------------------------------------------+
|                               PulseAuth Day 1 Core Architecture                         |
|                                                                                         |
|        +-----------------------------------------------------------------------+        |
|        |                  OpenID Discovery & Metadata Layer                    |        |
|        |                                                                       |        |
|        |    GET /.well-known/openid-configuration   GET /.well-known/jwks.json |        |
|        +-----------------------------------+-----------------------------------+        |
|                                            |                                            |
|                                            v                                            |
|        +-----------------------------------------------------------------------+        |
|        |                RSAKeyManager (app/crypto/rsa_manager.py)              |        |
|        |  * 2048-bit RSA Keypair Generation (PKCS8 / SPKI)                     |        |
|        |  * Deterministic 'kid' Derivation (SHA-256 Fingerprint)               |        |
|        |  * RFC 7518 Base64URL UInt Formatter ("AQAB", Modulus n)              |        |
|        |  * RS256 JWT Signing & Signature Verification Engine                  |        |
|        +-----------------------------------+-----------------------------------+        |
|                                            |                                            |
|                                            v                                            |
|        +-----------------------------------------------------------------------+        |
|        |                    SQLite WAL Relational Database                     |        |
|        |  +--------------------+  +--------------------+  +------------------+ |        |
|        |  |       users        |  |      clients       |  |   audit_logs     | |        |
|        |  |  scrypt passwords  |  |  public/confid.    |  |  security events | |        |
|        |  +--------------------+  +--------------------+  +------------------+ |        |
|        |  | authorization_codes|  |       tokens       |  |     consents     | |        |
|        |  |  PKCE S256 / TTL   |  | SHA-256 hashed val |  | user scope grants| |        |
|        |  +--------------------+  +--------------------+  +------------------+ |        |
|        +-----------------------------------------------------------------------+        |
+-----------------------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Asymmetric RSA Keypair Manager & JWKS Engine (`app/crypto/rsa_manager.py`)
To enable standards-compliant RS256 JWT signing and public key distribution, `RSAKeyManager` wraps Python's `cryptography.hazmat` primitives:
- **Key Generation & Export:** Generates 2048-bit RSA keys with public exponent $e = 65537$. Exports unencrypted private keys in PKCS8 PEM format and public keys in SubjectPublicKeyInfo (SPKI) PEM format.
- **Deterministic Key ID (`kid`):** Generates a 16-character hexadecimal fingerprint from the SHA-256 digest of the public key's PEM bytes, ensuring cache-busting consistency across rotating keys.
- **RFC 7518 Base64urlUInt Encoding:** Converted the RSA modulus $n$ and exponent $e$ into unpadded URL-safe Base64 strings. For $e = 65537$, the exponent string evaluates strictly to `"AQAB"`:

```python
def _int_to_base64url_uint(val: int) -> str:
    """Converts a positive integer to an RFC 7518 Base64urlUInt string."""
    byte_length = (val.bit_length() + 7) // 8
    val_bytes = val.to_bytes(byte_length, byteorder="big")
    return base64.urlsafe_b64encode(val_bytes).decode("ascii").rstrip("=")
```

- **RFC 7517 JWKS Structure:** Exports active keys under `GET /.well-known/jwks.json` with keys tagged for `kty: "RSA"`, `use: "sig"`, `alg: "RS256"`, and the computed `kid`.

### Standards-Compliant Discovery Endpoints (`app/routes/discovery_routes.py`)
Published the canonical OpenID Connect Discovery configuration per RFC 8414:
- `GET /.well-known/openid-configuration`: Advertises issuer identifier, authorization and token endpoint URIs, supported scopes (`openid`, `profile`, `email`, `offline_access`, `read:documents`, `write:documents`), grant types (`authorization_code`, `client_credentials`, `refresh_token`), PKCE code challenge methods (`S256`, `plain`), and token signing algorithms (`RS256`).
- `GET /.well-known/jwks.json`: Distributes public keys directly to relying parties.
- `GET /api/v1/health`: Vitality probe returning issuer status and active `kid`.

### High-Concurrency SQLite WAL Schema (`data/schema.sql` & `app/db/connection.py`)
Constructed the relational schema supporting strict relational integrity, foreign keys, and zero reader-writer lock contention:

```sql
PRAGMA foreign_keys = ON;
PRAGMA journal_mode = WAL;
PRAGMA busy_timeout = 30000;

CREATE TABLE IF NOT EXISTS users (
    id TEXT PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    full_name TEXT NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS clients (
    id TEXT PRIMARY KEY,
    client_id TEXT UNIQUE NOT NULL,
    client_secret_hash TEXT,
    client_name TEXT NOT NULL,
    client_type TEXT NOT NULL CHECK(client_type IN ('public', 'confidential')),
    redirect_uris TEXT NOT NULL,
    allowed_grant_types TEXT NOT NULL,
    allowed_scopes TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### Client & User Entity Models (`app/models/`)
- **`ClientModel`**: Enforces strict exact-match validation for `redirect_uri` against pre-registered whitelists to prevent open-redirect vulnerabilities. Distinguishes public clients (which require zero secrets) from confidential clients (which enforce scrypt password hashing).
- **`UserModel`**: Handles identity registration and verification using scrypt hashing (`hashlib.scrypt`) with salt lengths of 16 bytes, $n=16384$, $r=8$, and $p=1$.

### Database Seeding Utility (`data/seed.py`)
Initialized baseline identities and applications for hermetic test execution and interactive validation:
- Users: `developer@pulseauth.dev`, `admin@pulseauth.dev`, and `user@pulseauth.dev`.
- Clients:
  - `pulse-web-app`: Confidential client with secret `pulse_web_secret_2026`.
  - `pulse-mobile-app`: Public client requiring PKCE with zero client secret.
  - `pulse-m2m-service`: Machine-to-machine service client for `client_credentials` grant.

---

## 3. Verification & Quality Gates

All 5 automated enterprise quality gates passed with zero warnings or errors:

1. **Pytest Test Suite:** 20/20 tests passing in 2.33 seconds.
2. **Branch Coverage:** 97.49% branch coverage across all backend modules.
3. **Flake8 Linter:** 0 errors/warnings at 88-character maximum line length.
4. **Black Formatter:** Code formatted and verified across all modules.
5. **isort:** All module and third-party imports sorted and separated.
6. **Bandit Security Audit:** 0 security vulnerabilities identified.

```text
======================================================================
  PULSEAUTH BACKEND: 5-STAGE AUTOMATED QUALITY GATES
======================================================================
  * FLAKE8 LINTER        : [OK] PASSED  (1.18s)
  * BLACK FORMATTER      : [OK] PASSED  (0.94s)
  * ISORT IMPORTS        : [OK] PASSED  (0.29s)
  * BANDIT SECURITY      : [OK] PASSED  (0.49s)
  * PYTEST COVERAGE      : [OK] PASSED  (3.43s)
======================================================================
>> ALL QUALITY GATES PASSED! Safe to commit and deploy.

============================= test session starts =============================
collected 20 items

tests/test_crypto.py::test_rsa_key_generation PASSED                     [  5%]
tests/test_crypto.py::test_kid_deterministic_derivation PASSED           [ 10%]
tests/test_crypto.py::test_base64url_uint_conversion PASSED              [ 15%]
tests/test_crypto.py::test_jwks_export_structure PASSED                  [ 20%]
tests/test_crypto.py::test_jwt_sign_and_verify PASSED                    [ 25%]
tests/test_crypto.py::test_jwt_tampered_signature_rejected PASSED        [ 30%]
tests/test_discovery.py::test_openid_configuration_endpoint PASSED       [ 35%]
tests/test_discovery.py::test_jwks_endpoint PASSED                       [ 40%]
tests/test_discovery.py::test_health_endpoint PASSED                     [ 45%]
tests/test_db.py::test_connection_wal_mode PASSED                        [ 50%]
tests/test_db.py::test_foreign_keys_enforced PASSED                      [ 55%]
tests/test_models.py::test_user_registration_and_auth PASSED             [ 60%]
tests/test_models.py::test_user_duplicate_email_rejected PASSED          [ 65%]
tests/test_models.py::test_client_confidential_validation PASSED         [ 70%]
tests/test_models.py::test_client_public_validation PASSED               [ 75%]
tests/test_models.py::test_client_redirect_uri_exact_match PASSED        [ 80%]
tests/test_seed.py::test_seed_creates_default_users PASSED               [ 85%]
tests/test_seed.py::test_seed_creates_default_clients PASSED             [ 90%]
tests/test_seed.py::test_seed_idempotency PASSED                          [ 95%]
tests/test_crypto.py::test_key_reload_from_pem PASSED                    [100%]

=============================== tests coverage ================================
Name                             Stmts   Miss Branch BrPart  Cover
------------------------------------------------------------------
app/config/settings.py              45      0      4      0   100%
app/crypto/rsa_manager.py           87      1     18      2    97%
app/db/connection.py                37      2     10      2    91%
app/models/client_model.py          71      1     22      1    98%
app/models/user_model.py            40      0      8      0   100%
app/routes/discovery_routes.py      17      0      0      0   100%
------------------------------------------------------------------
TOTAL                              297      4     62      5    97%
============================= 20 passed in 2.33s ==============================
```

---

## 4. Key Engineering Insights & Decisions

1. **RS256 vs. Symmetric HS256 in Microservice Topologies:**
   Symmetric signing (HS256) requires every microservice verifying incoming access tokens to possess the identical shared secret. If a single resource server is compromised, the attacker can forge administrative tokens across the entire fleet. By deploying asymmetric RS256, downstream resource servers require only the public JWKS (`GET /.well-known/jwks.json`), completely isolating the private signing key inside PulseAuth.
2. **RFC 7518 Big-Endian Byte Representation:**
   The RSA exponent $e = 65537$ in binary is `0x010001` (3 bytes: `[1, 0, 1]`). Standard Base64 encoding transforms this into `"AQAB"`. Stripping trailing padding (`=`) is strictly required by RFC 7518 Section 2; omitting this produces cryptographic syntax rejections in standard client libraries like Node's `jose` or Go's `golang-jwt`.
3. **Deterministic `kid` Derivation:**
   Rather than storing arbitrary database sequences or UUIDs for Key IDs, deriving the `kid` from `sha256(public_pem)[:16]` guarantees that any instance loading the same keypair deterministically emits the exact same Key ID across reboots.

---

## 5. Next Steps

With Day 1 cryptographic scaffolding and database foundations established, **Day 2** will focus on:
- Authorization Code Grant Flow (`GET` and `POST /oauth/authorize`).
- RFC 7636 Proof Key for Code Exchange (PKCE) verification engine supporting `S256` and `plain`.
- Ephemeral 60-second Authorization Code lifecycle with atomic single-use invalidation.
- Interactive user scope consent screen with high-contrast responsive styling.
