---
date: 2026-11-27
project: PulsePress Full-Featured Blog Platform
topic: Day 5 - REST API Gateway and Author Endpoints
Tags:
  - "[[REST-API]]"
  - "[[JWT-Authentication]]"
  - "[[RBAC]]"
  - "[[Flask-Blueprints]]"
  - "[[Integration-Testing]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 48 - DAY 5

**Core Objective:** Expose the complete PulsePress backend through a unified REST API Gateway. This includes implementing `AuthService` with HMAC-SHA256 JWT tokens and role-based access control (`require_auth`), building six Flask route blueprints (`auth_routes`, `post_routes`, `search_routes`, `tag_routes`, `comment_routes`, `analytics_routes`), registering all nine blueprints in the Flask application factory, and validating end-to-end API flows with integration tests in `tests/test_api_routes.py`. Verify all 5 quality gates with 67 passing tests at 94% statement coverage, and push every file individually one-by-one.

---

## 1. The Big Picture and Simple Explanation

Days 1 through 4 built PulsePress's internal engines: SQLite WAL persistence, Markdown rendering, BM25 full-text search, SEO metadata/feeds, tag taxonomy, and threaded comment moderation. However, none of those capabilities were reachable over HTTP by a frontend or external client.

Without a disciplined API gateway, web backends tend to fracture in three ways:
1. **Unprotected Author & Admin Mutations:** Allowing unauthenticated or non-privileged requests to publish posts, moderate comments, or view analytics compromises editorial integrity.
2. **Stale Search Indexes on Write:** Publishing or editing a post in the database without synchronously updating the inverted search index causes readers to miss new articles in search results.
3. **Leaked Database Connections Under Concurrency:** Leaving raw SQLite connections open across request handlers exhausts file handles and causes file-locking errors on Windows.

On Day 5, **PulsePress** unifies all domain services behind a clean JSON REST API:
- **Authentication & RBAC (`AuthService` + `auth_routes.py`):** Issues signed HS256 JWT Bearer tokens, enforces role-based guards (`admin`, `author`), and provides `/api/auth/register`, `/api/auth/login`, and `/api/auth/me`.
- **Editorial & Reader Endpoints (`post_routes.py`, `search_routes.py`, `tag_routes.py`, `comment_routes.py`, `analytics_routes.py`):** Connects HTTP requests to Markdown rendering, BM25 search/autocomplete, tag cloud/archive queries, threaded comments/moderation, and visitor analytics.

```
+-----------------------------------------------------------------------------+
|                    Day 5 REST API Gateway Architecture                      |
|                                                                             |
|  HTTP Request                                                               |
|       |                                                                     |
|       v                                                                     |
|  [ Flask App Factory (app/__init__.py) ] -> 9 Registered Blueprints         |
|       |                                                                     |
|       +--> /api/auth/*      -> AuthService (HS256 JWT + Password Hashing)   |
|       +--> /api/posts/*     -> PostDAO + MarkdownEngine + SearchEngine      |
|       +--> /api/search/*    -> SearchEngine (BM25 + Prefix Autocomplete)    |
|       +--> /api/tags/*      -> TagService (Cloud + Paginated Tag Archive)   |
|       +--> /api/comments/*  -> CommentService (Tree + Rate Limit + Mod)     |
|       +--> /api/analytics/* -> AnalyticsDAO (Views, Visitors, Top Posts)    |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### JWT Authentication & Role-Based Access Control

The authentication layer (`app/services/auth_service.py` and `app/routes/auth_routes.py`) provides stateless request authentication:
- **HS256 JWT Token Lifecycle:** `AuthService` signs and verifies compact JWTs using HMAC-SHA256 with configurable expiration (`exp`), user ID (`sub`), and role claims (`role`), without requiring external JWT dependencies.
- **`require_auth()` Decorator:** Extracts the `Bearer` token from the `Authorization` header, validates signature and expiry, verifies the user record via `UserDAO`, and enforces optional role restrictions (e.g., `roles=["admin"]` for moderation and analytics, `roles=["admin", "author"]` for post creation and publishing).
- **Auth Endpoints:** Exposes `POST /api/auth/register`, `POST /api/auth/login`, and `GET /api/auth/me` with consistent JSON error envelopes.

### Post CRUD, Reader View Tracking & Synchronous Search Indexing

The post blueprint (`app/routes/post_routes.py`) bridges editorial authoring and public reading:
- **Author Workflow:** Supports creating drafts (`POST /api/posts`), updating content (`PUT /api/posts/<id>`), publishing (`POST /api/posts/<id>/publish`), and deleting posts (`DELETE /api/posts/<id>`), enforcing ownership or admin privileges.
- **Markdown Rendering & Search Sync:** Renders Markdown to HTML, TOC, and reading time via `MarkdownEngine(wpm=...)`, and updates the BM25 inverted index via `_index_post_record()` whenever a published post is created, updated, or published.
- **Public Reader Endpoint:** `GET /api/posts/<slug>` serves published articles enriched with tags, `TagService.find_related_posts()` recommendations, and `SEOEngine` metadata, while recording page views in `AnalyticsDAO`.

### Search, Tags, Comments, and Analytics Blueprints

- **Search (`app/routes/search_routes.py`):** `GET /api/search` executes ranked BM25 queries with snippet highlighting; `GET /api/search/suggest` provides prefix autocomplete suggestions.
- **Tags (`app/routes/tag_routes.py`):** `GET /api/tags` returns the normalized weighted tag cloud; `GET /api/tags/<slug>/posts` returns paginated posts for a tag.
- **Comments (`app/routes/comment_routes.py`):** Public `GET /api/posts/<id>/comments` and `POST /api/posts/<id>/comments` (with IP rate limiting and XSS escaping), plus admin-only moderation queue (`GET /api/admin/comments`), single moderation (`PATCH /api/admin/comments/<id>`), and bulk moderation (`POST /api/admin/comments/bulk`).
- **Analytics (`app/routes/analytics_routes.py`):** Admin-protected `/api/analytics/overview` reporting total views, unique visitors, and top-performing posts.

---

## 3. Verification & Quality Gates Report

All five quality gates passed with zero violations, zero Bandit findings, and 94% overall statement coverage across 67 tests:

| Quality Gate | Tool | Target / Threshold | Result | Status |
| :--- | :--- | :--- | :--- | :--- |
| Linting | Flake8 | 0 violations (88-char limit) | 0 warnings / 0 errors | PASSED |
| Code Formatting | Black | PEP 8 / 88 columns | 100% compliant (50 files checked) | PASSED |
| Import Sorting | isort | Black profile compatibility | 100% compliant | PASSED |
| Security Scanning | Bandit | Zero medium/high vulnerabilities | 0 issues across 3,696 LOC (9 `#nosec`) | PASSED |
| Test Coverage | Pytest | >= 90% statement coverage | 67/67 passed (94% coverage) | PASSED |

### Day 5 Module Coverage Breakdown
- `app/routes/auth_routes.py`: **100% coverage**
- `app/routes/__init__.py`: **100% coverage**
- `app/__init__.py`: **95% coverage**
- `app/routes/analytics_routes.py`: **92% coverage**
- `app/routes/tag_routes.py`: **87% coverage**
- `app/routes/search_routes.py`: **86% coverage**
- `app/routes/post_routes.py`: **83% coverage**
- `app/services/auth_service.py`: **83% coverage**
- `app/routes/comment_routes.py`: **81% coverage**

---

## 4. Key Engineering Insights & Decisions

- **Explicit `closing()` Context Managers for SQLite Connections:** `get_db_connection()` returns a raw `sqlite3.Connection` without an `__exit__` auto-close hook. Using `with get_db_connection(...)` left file descriptors open on Windows, triggering `PermissionError: [WinError 32]` during temporary test database teardown. Wrapping connection lifetimes with `contextlib.closing()` across route handlers and the `require_auth` decorator guarantees deterministic connection closure.
- **Aligning Route Call Sites with Service Signatures:** Integration testing caught mismatches between `post_routes.py` and the Day 2 service APIs (`MarkdownEngine(wpm=...)` instead of `words_per_minute`, `SearchEngine.index_post(conn, post_id, title, body_markdown, tag_names)` instead of passing a post dictionary, and unpacking the `(posts, total_count)` tuple from `PostDAO.list_posts()`). Introducing `_index_post_record()` normalized the post-to-search indexing contract.
- **Avoiding Hardcoded Secret Defaults for Bandit (B107):** Making `secret_key` a required parameter in `AuthService` eliminated Bandit B107 warnings about hardcoded password/secret defaults.

---

## 5. Next Steps

- Build the Day 6 frontend UI modules (`blog_reader.js`, `post_list.js`, `search_view.js`, `comment_widget.js`, and `author_dashboard.js`) wired to the Day 5 REST API Gateway.
- Commit and push each frontend file individually one-by-one.
- Proceed to Day 7 concurrency benchmarks, final quality gates, and project handover.
