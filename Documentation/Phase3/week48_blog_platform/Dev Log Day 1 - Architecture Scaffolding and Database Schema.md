---
date: 2026-11-23
project: PulsePress Full-Featured Blog Platform
topic: Day 1 - Architecture Scaffolding and Database Schema
Tags:
  - "[[Blog-Engine]]"
  - "[[SQLite-WAL]]"
  - "[[Schema-Design]]"
  - "[[Slug-Generation]]"
  - "[[Architecture]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 48 - DAY 1

**Core Objective:** Kick off Week 48 (PulsePress Blog Platform) by scaffolding the directory structure, designing the SQLite WAL schema for users, posts, tags, comments, search index, and views, building strong Data Access Objects (DAOs), implementing a collision-safe unicode slug generator, establishing the test suite with 26 passing tests at 96% coverage, and verifying all five quality gates.

---

## 1. The Big Picture and Simple Explanation

Modern content publishing platforms must deliver more than just basic article rendering. Today's web standards demand sub-millisecond retrieval speeds, rock-solid relational data integrity, crawlable metadata for search engines, and real-time interactive search across thousands of posts.

In traditional blogging systems, content architectures often suffer from:
1. **Unindexed Text Blobs:** Searching full text using naive SQL `LIKE %term%` queries degrades rapidly as post counts grow, locking database tables and spiking latency.
2. **Brittle URL Slugs:** Creating URLs without unicode transliteration or collision handling causes broken links, duplicate route collisions, and poor international SEO.
3. **Flat or Unmoderated Comments:** Disconnected comment schemas without parent-child nesting cannot construct conversation threads without expensive recursive queries.

To solve this systematically from Day 1, **PulsePress** establishes a robust foundation:
- A normalized relational schema in SQLite with Write-Ahead Logging (WAL) mode enabled, supporting concurrent readers alongside background search indexing.
- Specialized Data Access Objects (`UserDAO`, `PostDAO`, `TagDAO`, `CommentDAO`, `SearchIndexDAO`, `AnalyticsDAO`) providing clean separation between database queries and application logic.
- A deterministic slug generator handling unicode transliteration, punctuation stripping, length clamping, and database collision counter resolution.
- A modular Flask factory structure with security headers, CORS support, and static asset streaming.

```
+-----------------------------------------------------------------------------+
|                         PulsePress Architecture - Day 1                     |
|                                                                             |
|  [ Frontend Workbench Shell ]              [ Flask Application Factory ]    |
|  - index.html (SPA Entrypoint)             - create_app()                   |
|  - base.css (Tokens & Themes)              - config (Settings & Envs)       |
|  - typography.css (Reading Layout)         - routes/health (/api/health)    |
|  - code-theme.css (Pygments Syntax)        - routes/static_routes           |
|  - app_shell.js (Theme & Catalog)          - services/slug_service          |
|                                                          |                  |
|                                                          v                  |
|  [ Data Access Layer (DAOs) ]              [ SQLite WAL Database ]          |
|  - UserDAO (Auth & Roles)                  - users (Admins, Authors)        |
|  - PostDAO (Lifecycle & Filtering)        - posts (Markdown & HTML)        |
|  - TagDAO (Taxonomy & Counts)              - tags & post_tags (Junction)    |
|  - CommentDAO (Threaded Trees)             - comments (Parent-Child FK)     |
|  - SearchIndexDAO (Inverted Index)         - search_index (Terms & TF)      |
|  - AnalyticsDAO (Views & Visitors)         - post_views (Visitor Hashes)    |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Directory Scaffolding and Module Topology

The baseline directory layout and placeholder files were scaffolded cleanly across backend and frontend:

```
week48_blog_platform/
├── backend/
│   ├── app/
│   │   ├── config/
│   │   │   ├── __init__.py
│   │   │   └── settings.py
│   │   ├── content/
│   │   │   └── __init__.py
│   │   ├── db/
│   │   │   ├── __init__.py
│   │   │   └── connection.py
│   │   ├── models/
│   │   │   ├── __init__.py
│   │   │   ├── analytics.py
│   │   │   ├── comment.py
│   │   │   ├── post.py
│   │   │   ├── search_entry.py
│   │   │   ├── tag.py
│   │   │   └── user.py
│   │   ├── routes/
│   │   │   ├── __init__.py
│   │   │   ├── health.py
│   │   │   └── static_routes.py
│   │   ├── seo/
│   │   │   └── __init__.py
│   │   ├── services/
│   │   │   ├── __init__.py
│   │   │   └── slug_service.py
│   │   └── __init__.py
│   ├── data/
│   │   ├── schema.sql
│   │   └── seed.py
│   ├── tests/
│   │   ├── __init__.py
│   │   ├── conftest.py
│   │   ├── test_connection.py
│   │   ├── test_health.py
│   │   ├── test_models.py
│   │   ├── test_schema.py
│   │   ├── test_seed.py
│   │   └── test_slug_service.py
│   ├── .flake8
│   ├── requirements.txt
│   └── run.py
└── frontend/
    ├── public/
    │   └── index.html
    └── src/
        ├── assets/
        │   ├── base.css
        │   ├── code-theme.css
        │   └── typography.css
        ├── components/
        └── ui/
            └── app_shell.js
```

### Relational Database Schema and SQLite WAL Architecture

The persistence layer operates under SQLite WAL (Write-Ahead Logging) mode with strict foreign key enforcement. Seven relational tables manage the blog lifecycle:

1. **`users`:** Manages identity, email addresses, securely hashed credentials (`werkzeug.security`), display names, biographies, avatar links, and role constraints (`admin`, `author`, `reader`).
2. **`posts`:** Stores canonical articles with author references, unique URL slugs, raw Markdown text, pre-rendered HTML, excerpts, featured image URLs, publishing status (`draft`, `published`, `archived`), calculated reading times, structured table of contents JSON, and timestamp telemetry.
3. **`tags`:** Stores article taxonomy categories with unique name and slug pairs, descriptions, and denormalized published post counts for fast tag cloud rendering.
4. **`post_tags`:** Composite primary key junction table (`post_id`, `tag_id`) establishing many-to-many associations between posts and tags with cascading deletions.
5. **`comments`:** Self-referencing hierarchical table with nullable `parent_id` foreign keys, visitor credentials, hashed IP addresses, moderation statuses (`pending`, `approved`, `rejected`, `flagged`), and timestamps.
6. **`search_index`:** Dedicated inverted index staging table mapping individual term tokens to parent posts, indexed fields (`title`, `body`, `tag`), term frequencies, and document lengths for Day 2 BM25 ranking.
7. **`post_views`:** High-speed audit log recording visitor fingerprint hashes against post identifiers for unique view deduplication and engagement metrics.

### Core Data Access Objects (DAOs)

- **`UserDAO` (`app/models/user.py`):** CRUD operations for user accounts, password hashing and verification, role gatekeeping, and user listings.
- **`PostDAO` (`app/models/post.py`):** Full post lifecycle management, publication transitions, slug lookups, multi-factor filtering (by status, author, tag slug), and paginated catalog queries with attached tags, view counts, and approved comment counts.
- **`TagDAO` (`app/models/tag.py`):** Taxonomy management, automated slug assignment, post association synchronization (`set_tags_for_post`), and transactional post count recalculation.
- **`CommentDAO` (`app/models/comment.py`):** Comment persistence, status transitions, IP-based submission rate limiting checks, and in-memory hierarchical tree assembly (`get_threaded_tree`) that resolves nested replies without recursive SQL overhead.
- **`SearchIndexDAO` (`app/models/search_entry.py`):** Term indexing, document length aggregation, autocomplete term lookups, and index cleanups.
- **`AnalyticsDAO` (`app/models/analytics.py`):** Page view tracking, unique visitor counting, and top-read post rankings.

### Collision-Safe Unicode Slug Generator Service

The slug service (`app/services/slug_service.py`) guarantees robust URL generation:
- **Unicode Transliteration:** Decomposes accented characters into base ASCII representations using `unicodedata.normalize('NFKD', ...)`.
- **Character Cleaning:** Strips non-alphanumeric symbols and collapses whitespace and multiple hyphens into single `-` delimiters.
- **Boundary Clamping:** Clamps slugs to a configurable maximum length (default 80 characters) at hyphen word boundaries.
- **Database Collision Resolution:** Queries existing database records with exact and pattern matching (`slug = ? OR slug LIKE ?`), iteratively appending incrementing numeric suffixes (`-1`, `-2`, `-3`) to guarantee uniqueness.

### Application Factory and Static Asset Delivery

- **Flask App Factory (`app/__init__.py`):** Initializes application settings, automatically executes schema migrations and seeds demo data on first startup, and registers modular HTTP blueprints.
- **Health Check (`app/routes/health.py`):** Verifies active SQLite connectivity (`SELECT 1`) and returns service health status.
- **Static Assets (`app/routes/static_routes.py`):** Streams public HTML and modular CSS design tokens directly to browser clients.
- **Security Headers:** Injects `X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN`, and permissive CORS headers on all HTTP responses.

---

## 3. Verification & Quality Gates Report

All five quality gates passed cleanly with zero violations and 96% test coverage:

| Quality Gate | Tool | Target / Threshold | Result | Status |
| :--- | :--- | :--- | :--- | :--- |
| Linting | Flake8 | 0 violations (88-char limit) | 0 warnings / 0 errors | PASSED |
| Code Formatting | Black | PEP 8 / 88 columns | 100% compliant (29 files checked) | PASSED |
| Import Sorting | isort | Black profile compatibility | 100% compliant | PASSED |
| Security Scanning | Bandit | Zero medium/high vulnerabilities | 0 issues across 1,068 LOC | PASSED |
| Test Coverage | Pytest | >= 90% statement coverage | 26/26 passed (96% coverage) | PASSED |

### Test Suite Highlights across Modules
- `test_connection.py`: Verified WAL journal mode, foreign key enforcement, busy timeout (5000ms), and in-memory database execution.
- `test_schema.py`: Verified all 7 core tables and indexes, check constraints for user roles and comment statuses, and cascading foreign key deletions.
- `test_seed.py`: Verified baseline data population (2 users, 6 tags, 3 posts, 3 comments), idempotent repeated executions, and secure password hash verification.
- `test_slug_service.py`: Verified basic slug generation, accented unicode transliteration, empty/symbol fallback to untitled, max-length boundary truncation, and database collision counter resolution.
- `test_models.py`: Verified full CRUD lifecycles for UserDAO, TagDAO, PostDAO, CommentDAO (including hierarchical reply trees and rate limit counters), SearchIndexDAO, and AnalyticsDAO.
- `test_health.py`: Verified `/api/health` 200 OK responses, static index and CSS streaming, and security/CORS response headers.

---

## 4. Key Engineering Insights & Decisions

- **SQLite WAL Mode for High Read/Write Concurrency:** Standard SQLite locks the entire database file during writes. Enabling Write-Ahead Logging (`PRAGMA journal_mode = WAL`) allows readers to query articles and tags concurrently while background processes index search terms or record visitor analytics.
- **Denormalized Tag Post Counts with Recalculation Triggers:** Displaying post counts next to tag badges in tag clouds is an extremely frequent read operation. Storing a denormalized `post_count` column in `tags` and updating it upon publication or tag reassignment avoids expensive joins across every catalog request.
- **In-Memory Threaded Tree Assembly over Recursive SQL:** While SQLite supports recursive Common Table Expressions (CTEs), building comment reply trees in Python memory from a single indexed query (`ORDER BY created_at ASC`) proved faster, simpler to unit test, and fully compatible across testing fixtures.
- **Deterministic Slug Disambiguation:** By querying both exact matches and pattern prefixes (`base_slug-%`), the slug generator avoids trial-and-error database write collisions, deriving the next available integer suffix in a single query pass.

---

## 5. Next Steps

- Implement the Markdown content engine (`app/content/markdown_engine.py`) with CommonMark rendering, Pygments code syntax highlighting, and heading anchor ID generation.
- Construct the Table of Contents (TOC) JSON extractor and reading time calculator.
- Build the full-text search engine (`app/services/search_engine.py`) featuring Porter stemming, stop-word filtering, inverted index generation, and BM25 probabilistic relevance ranking.
- Expand test coverage to validate Markdown rendering, syntax tokenization, and BM25 search scoring accuracy.
