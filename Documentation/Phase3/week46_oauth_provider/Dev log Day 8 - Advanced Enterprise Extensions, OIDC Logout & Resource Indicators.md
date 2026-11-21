---
date: 2026-11-14
project: PulseAuth Enterprise OAuth 2.0 & OpenID Connect Provider
topic: Day 8 - Advanced Enterprise Extensions, OIDC Logout & Resource Indicators
Tags:
  - "[[OAuth2]]"
  - "[[OIDC-Logout]]"
  - "[[RFC-8707]]"
  - "[[Resource-Indicators]]"
  - "[[Enterprise-Extensions]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 46 - DAY 8

**Core Objective:** Design, implement, and deploy two advanced enterprise protocol extensions for the PulseAuth Identity Provider: OpenID Connect RP-Initiated Logout 1.0 (`/oauth/logout`) with cryptographic ID token hint validation, open redirect prevention, and user session token revocation, alongside RFC 8707 Resource Indicators binding access tokens to explicit target audience URIs to defend against unauthorized token forwarding in microservice architectures, backed by 161 automated tests passing with 96.00% branch coverage and 0 linter/security warnings.

---

## 1. The Big Picture and Simple Explanation

In enterprise identity federations, initial authentication and token issuance are only part of the lifecycle. As architectures scale across multiple independent relying parties (RPs) and microservices, two critical security challenges arise:

1. **Synchronized Session Termination (OIDC RP-Initiated Logout):** When a user logs out of one relying party application (e.g. an internal dashboard), their session must be safely terminated at the Identity Provider level, and all active access/refresh tokens issued to that client must be revoked. Without standardized RP-initiated logout, stale tokens remain active, allowing unauthorized reuse. Furthermore, post-logout redirects must strictly validate target URIs to prevent open redirect phishing attacks.
2. **Audience Restriction & Token Forwarding Defense (RFC 8707 Resource Indicators):** In distributed microservice ecosystems, a client may obtain a broad access token with multiple scopes and present it to Microservice A. If Microservice A is compromised or malicious, it could replay that same token against Microservice B (token forwarding). RFC 8707 introduces the `resource` parameter, binding each issued access token to an explicit target resource URI (audience). Downstream resource servers reject any token whose bound resource does not match their own canonical URI.

```
+-----------------------------------------------------------------------------------------+
|                  PulseAuth Advanced Enterprise Extensions Architecture                  |
|                                                                                         |
|   1. OpenID Connect RP-Initiated Logout 1.0                                             |
|                                                                                         |
|   Relying Party                  PulseAuth IdP                   End User               |
|         |                              |                             |                  |
|         |  GET /oauth/logout           |                             |                  |
|         |  ?id_token_hint=...          |                             |                  |
|         |  &post_logout_redirect_uri   |                             |                  |
|         |----------------------------->|                             |                  |
|         |                              |  Validate RS256 signature   |                  |
|         |                              |  Validate redirect URI      |                  |
|         |                              |  Render confirm screen      |                  |
|         |                              |---------------------------->|                  |
|         |                              |                             |                  |
|         |                              |  Confirm Session Logout     |                  |
|         |                              |<----------------------------|                  |
|         |                              |                                                |
|         |                              |  Revoke active user tokens                     |
|         |  HTTP 302 Redirect (Clean)   |  Emit audit: "user_logged_out"                 |
|         |<-----------------------------|                                                |
|                                                                                         |
|   2. RFC 8707 Resource Indicators for OAuth 2.0                                         |
|                                                                                         |
|   Client App                     PulseAuth                     Resource Server A        |
|         |                             |                                |                |
|         |  POST /oauth/token          |                                |                |
|         |  resource=https://api.a.com |                                |                |
|         |---------------------------->|                                |                |
|         |                             | Bind "aud": https://api.a.com  |                |
|         |  Issue Targeted RS256 JWT   |                                |                |
|         |<----------------------------|                                |                |
|         |                                                              |                |
|         |  GET /documents (Bearer Token)                               |                |
|         |------------------------------------------------------------->|                |
|         |                                                              | TokenGuard:    |
|         |                                                              | Check "aud" == |
|         |                                                              | https://api.a  |
|         |  HTTP 200 OK (Audience Verified)                             |                |
|         |<-------------------------------------------------------------|                |
+-----------------------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### OpenID Connect RP-Initiated Logout 1.0 (`logout_routes.py` & `logout.html`)
- **Route Controller (`/oauth/logout`):**
  - Supports both `GET` and `POST` methods.
  - Parameter extraction: `id_token_hint`, `client_id`, `post_logout_redirect_uri`, `state`, and `confirm`.
  - Cryptographic validation: When `id_token_hint` is provided, verifies the RS256 signature using `RSAKeyManager`. Extracts `sub` (user identity) and `aud` (client identifier). If signature verification fails or the token is expired, returns HTTP 400 `invalid_request`.
  - Client validation: Confirms that `client_id` (either explicitly passed or derived from the token hint) corresponds to an active client registered in the database.
  - Post-logout redirect URI validation: Validates that the requested `post_logout_redirect_uri` matches the client's registered redirect URI list, defending against open redirect exploits.
  - User confirmation screen: When invoked via `GET` without instant redirect or when user confirmation is required, renders a high-contrast dark confirmation screen (`app/templates/logout.html`).
  - Session termination & token revocation: Upon confirmation, invokes `TokenModel.revoke_user_tokens(conn, user_id, client_id)`, revoking all active access and refresh tokens belonging to the user.
  - Redirect handling: If `post_logout_redirect_uri` is valid, issues an HTTP 302 redirect with preserved `state`. If absent, renders a session terminated status screen.

### RFC 8707: Resource Indicators for OAuth 2.0
- **Syntax & Structural Validation (`AuthCodeService.validate_resource`):**
  - Enforces RFC 8707 rules: the `resource` parameter MUST be an absolute URI and MUST NOT contain a URI fragment (`#`).
  - Rejects malformed or fragmented URIs with standard error code `invalid_target`.
- **Database Schema & Model Persistence:**
  - Added `resource TEXT` column to `authorization_codes` and `tokens` tables in `backend/data/schema.sql`.
  - Updated `AuthCodeModel.create()` and `TokenModel.create()` to persist target resource URIs.
- **Grant Flows Integration:**
  - Authorization Code Grant: `resource` is validated in `auth_routes.py`, stored on the authorization code, displayed on `consent.html`, and transferred to the issued RS256 access token.
  - Client Credentials Grant: Machine-to-Machine clients specify `resource` during `POST /oauth/token` exchange; the target is verified and bound to the issued service token.
  - Refresh Token Rotation (RTR): `RefreshService.rotate_token()` preserves the original `resource` claim across refresh token rotations.
- **Resource Server Enforcement (`TokenGuard`):**
  - Extended `@require_oauth_token(expected_resource=...)` to enforce audience matching.
  - Confirms that the presented access token's `aud` or `resource` claim matches the protected resource server's URI, preventing cross-service token substitution.

### OpenID Discovery Metadata Update
Updated `backend/app/routes/discovery_routes.py` to publish the `end_session_endpoint` in accordance with the OpenID Connect RP-Initiated Logout specification:
```json
{
  "end_session_endpoint": "http://127.0.0.1:5000/oauth/logout"
}
```

### Interactive Frontend Console Integration
- **`frontend/public/index.html`:**
  - Added `Resource Indicator (RFC 8707)` configuration field in Tab 1 (OAuth Playground).
  - Added `[POST] /oauth/logout (RP LOGOUT)` action trigger button in the Resource Server action runner.
- **`frontend/src/js/sandbox_playground.js`:**
  - Propagates `resource` parameter in authorization URL generator, fast simulation helper, and token exchange routines.
  - Wired RP logout button to invoke `POST /oauth/logout` with `id_token_hint`, clearing local tokens upon successful session termination.

---

## 3. Automated Test Suite Expansion

Two new dedicated test suites were implemented:
1. **`backend/tests/test_logout_routes.py` (10 tests):**
   - `test_logout_get_prompt_with_valid_id_token_hint`: Validates rendering of confirmation screen with user profile.
   - `test_logout_get_direct_with_redirect_uri`: Validates direct redirection when confirmation is pre-authorized.
   - `test_logout_post_confirm_revokes_tokens_and_redirects`: Confirms database token revocation and HTTP 302 redirect.
   - `test_logout_post_confirm_without_redirect_renders_logged_out`: Confirms session termination view without redirect URI.
   - `test_logout_invalid_id_token_hint_rejected`: Validates rejection of corrupted ID token hints.
   - `test_logout_unregistered_redirect_uri_rejected`: Validates rejection of unapproved redirect URIs.
   - `test_logout_unknown_client_id_rejected`: Validates rejection of unknown client IDs.
   - `test_logout_without_id_token_hint_handles_cleanly`: Confirms generic session logout without token hints.
   - `test_logout_preserves_state_parameter`: Confirms state parameter preservation across redirects.
   - `test_openid_configuration_advertises_end_session_endpoint`: Verifies discovery document metadata.

2. **`backend/tests/test_resource_indicators.py` (7 tests):**
   - `test_validate_resource_syntax_rules`: Validates RFC 8707 absolute URI and fragment-free constraints.
   - `test_authorize_request_with_valid_resource_indicator`: Verifies resource persistence on authorization code.
   - `test_authorize_request_with_invalid_resource_rejected`: Verifies HTTP 302 error redirection with `error=invalid_target`.
   - `test_token_exchange_binds_resource_indicator`: Verifies audience claim binding on issued access tokens.
   - `test_client_credentials_with_resource_indicator`: Verifies resource binding on M2M tokens.
   - `test_refresh_token_preserves_resource_indicator`: Verifies resource preservation across refresh rotations.
   - `test_token_guard_enforces_expected_resource_audience`: Verifies resource server protection and rejection on audience mismatch.

---

## 4. Verification & Quality Gates Report

All 5 automated quality gates passed with zero warnings:

```text
======================================================================
  PULSEAUTH BACKEND: 5-STAGE AUTOMATED QUALITY GATES
======================================================================
  * FLAKE8 LINTER        : [OK] PASSED  (1.24s)
  * BLACK FORMATTER      : [OK] PASSED  (0.43s)
  * ISORT IMPORTS        : [OK] PASSED  (0.44s)
  * BANDIT SECURITY      : [OK] PASSED  (0.80s)
  * PYTEST COVERAGE      : [OK] PASSED  (67.29s)
======================================================================
>> ALL QUALITY GATES PASSED! Safe to commit and deploy.
```

### Test Suite Metrics
- Total Tests: **161 passed** (0 failed, 0 skipped).
- Branch Coverage: **96.00%** (1522 statements, 39 missed; 476 branches, 39 partial). Exceeds the 90.0% project requirement.
- Zero-Emoji Compliance: 100% verified across all code, documentation, CSS, and commit messages.

---

## 5. Key Engineering Insights & Decisions

1. **ID Token Hint Verification During Logout:**
   Accepting arbitrary logout requests without an `id_token_hint` allows malicious third-party sites to perform Denial-of-Service logouts (Cross-Site Scripting or CSRF logout attacks). Verifying the RS256 signature on the presented `id_token_hint` establishes that the caller legitimately received an authenticated session token for that specific subject.
2. **Fragment-Free URI Validation (RFC 8707):**
   RFC 8707 explicitly forbids URI fragment components (`#section`) in the `resource` parameter. Fragments are client-side only and never transmitted in HTTP request lines; accepting them leads to parser desynchronization between the authorization server and the resource server.
3. **Open Redirect Defenses on Post-Logout Redirects:**
   Attackers often craft malicious links targeting the logout endpoint with `post_logout_redirect_uri=https://evil-phishing.com`. By strictly validating the requested URI against the registered client whitelist, PulseAuth neutralizes open redirect attack vectors.

---

## 6. Final Project Handover & Curriculum Status

With the completion of Day 8, PulseAuth possesses:
- RFC 6749 (The OAuth 2.0 Authorization Framework)
- RFC 7636 (Proof Key for Code Exchange by OAuth Public Clients)
- OpenID Connect Core 1.0 (ID Tokens & UserInfo Endpoint)
- OpenID Connect RP-Initiated Logout 1.0 (End-Session Endpoint & Token Revocation)
- RFC 8707 (Resource Indicators for OAuth 2.0)
- RFC 7517 (JSON Web Key / JWKS)
- RFC 7519 (JSON Web Token / JWT)
- RFC 7009 (OAuth 2.0 Token Revocation)
- RFC 7662 (OAuth 2.0 Token Introspection)
- RFC 8414 (OAuth 2.0 Authorization Server Metadata Discovery)

All documentation, tests, and source files are committed and pushed to `main`.
