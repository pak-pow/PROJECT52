---
date: 2026-09-22
project: Production E-Commerce Platform v1
topic: Day 2 - Backend Foundation and Catalog API
Tags:
  - "[[E-Commerce]]"
  - "[[Backend]]"
  - "[[Python]]"
  - "[[Flask]]"
  - "[[SQLite]]"
  - "[[Testing]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 39 - DAY 2

**Core Objective:** Implement the complete backend core and product catalog engine for ShopPulse, featuring clean relational schema definitions, realistic seed data, search and filtering with pagination, health and readiness probes, and 22 automated tests achieving 98.44% branch coverage.

---

## 1. The Big Picture and Simple Explanation

A production e-commerce catalog must deliver fast, accurate product discovery while maintaining financial and relational integrity. Storing product prices as floating-point numbers or performing unindexed database queries leads to rounding errors and degraded latency under search loads.

In Day 2, we transformed our initial scaffolding into a production catalog service:
1. **Integer Cent Pricing Architecture:** Prices are stored strictly as whole integer cents (e.g., `$129.00` is persisted as `12900`). This eliminates floating-point precision loss during subtotal, discount, and tax calculations.
2. **Predictable Query Filters:** Implemented multi-parameter search across product titles and descriptions, category slug filtering, structured sorting (`price_asc`, `price_desc`, `name_asc`, `newest`), and clamped pagination (`limit` and `offset`).
3. **Database Health Telemetry:** Engineered container-ready probes (`/api/v1/health` and `/api/v1/ready`) validating SQLite WAL connectivity and measuring database ping latencies in milliseconds.
4. **Relational Constraints:** Enforced foreign keys, unique slugs, and non-negative stock constraints at the database schema level.

```
+-----------------------------------------------------------------------------+
|                     ShopPulse Catalog Query Engine Architecture             |
|                                                                             |
|      +---------------------------------------------------------------+      |
|      |             Client HTTP Request (GET /api/v1/products)        |      |
|      |      ?category=keyboards&sort=price_asc&limit=10&page=1       |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Flask Router Blueprint (app/routes/product_routes.py)         |      |
|      | - Validate and clamp pagination (limit: 1..100)               |      |
|      | - Parse search query, category slug, and sort parameters      |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Product Model Query Engine (app/models/product_model.py)      |      |
|      | - Parameterized SQL query builder                             |      |
|      | - Joins categories ON products.category_id = categories.id    |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | SQLite 3 Database (store.db)                                  |      |
|      | - Indexed tables: categories, products                        |      |
|      | - Integer cents storage (price_cents: 12900)                  |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | JSON Serialization & Pagination Metadata                      |      |
|      | { products: [...], total: 12, page: 1, limit: 10 }            |      |
|      +---------------------------------------------------------------+      |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Database Connection & Relational Schema (`app/db.py` & `data/schema.sql`)
- Built a context-aware connection manager using Flask `g` to maintain single-connection-per-request lifecycles with automatic cleanup on teardown.
- Enforced `PRAGMA foreign_keys = ON;` and `PRAGMA busy_timeout = 5000;`.
- Structured schema with dedicated indexes on `products.category_id`, `products.slug`, and `categories.slug`.
- Added a `ping_db()` helper measuring real SQLite query latency in milliseconds.

```sql
CREATE TABLE IF NOT EXISTS categories (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS products (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    category_id INTEGER NOT NULL,
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    description TEXT,
    price_cents INTEGER NOT NULL CHECK (price_cents >= 0),
    stock_quantity INTEGER NOT NULL DEFAULT 0 CHECK (stock_quantity >= 0),
    image_url TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE RESTRICT
);
```

### Catalog Seed Dataset (`data/seed.py`)
- Seeded 4 distinct product categories: Mechanical Keyboards, Studio Audio, Desk Accessories, and Developer Apparel.
- Seeded 12 developer-focused products with realistic pricing in integer cents, inventory counts, descriptions, and placeholder image assets.
- Built idempotent `ON CONFLICT` insertion logic ensuring repeated executions never create duplicate rows.

### Product Query Engine (`app/models/product_model.py`)
- Supported multi-parameter queries: category slug filtering, case-insensitive substring searching (`LIKE %query%`), and sorting (`price_asc`, `price_desc`, `name_asc`, `newest`).
- Clamped pagination parameters between 1 and 100 to prevent denial-of-service memory exhaustion.
- Enforced atomic stock deduction helpers validating that inventory never drops below zero.

### Catalog REST Endpoints (`app/routes/product_routes.py` & `health_routes.py`)
- `GET /api/v1/products`: Returns paginated product cards with total matching count and pagination metadata.
- `GET /api/v1/products/<slug>`: Returns full product detail or structured `404 Not Found` with `PRODUCT_NOT_FOUND` error code.
- `GET /api/v1/categories`: Returns categories with active product count aggregations.
- `GET /api/v1/health` & `GET /api/v1/ready`: Returns liveness and database readiness status codes (200 vs 503).

---

## 3. Verification & Quality Gates Report

The catalog API was verified across 22 automated unit and integration tests:

| Test Suite Module | Target Scope | Tests Passed | Status |
| :--- | :--- | :---: | :---: |
| `tests/test_products.py` | Product listing, category filters, sorting, detail API | 12 | Passed |
| `tests/test_health_and_version.py` | Liveness, readiness probes, database ping | 4 | Passed |
| `tests/test_coverage_boost.py` | Error handlers (400, 404, 405, 500) and edge cases | 6 | Passed |

### Quality Gates Execution Summary
- **Total Tests:** 22 automated tests passing in 0.48 seconds.
- **Branch Coverage:** 98.44% total branch coverage.
- **Linters:** Flake8, Black, isort, and Bandit passed with 0 warnings.

---

## 4. Key Engineering Insights & Decisions

- **Integer Cents Prevent Rounding Bugs:** Floating-point numbers cannot accurately represent base-10 fractions (e.g., `0.1 + 0.2 != 0.3`). Storing prices strictly as whole cents eliminates mathematical drift across e-commerce pricing engines.
- **Clamped Pagination Protects Memory:** Unbounded database queries can easily overwhelm application memory. Enforcing strict upper bounds on `limit` protects backend workers.
- **Database Schema Constraints as Last Line of Defense:** While application validation checks stock levels, adding `CHECK (stock_quantity >= 0)` directly in SQLite schema guarantees negative inventory is impossible at the database level.

---

## 5. Next Steps

- Implement the session-based Cart Engine (`cart_model.py`) and authoritative Pricing Service (`pricing_service.py`).
- Support cart token persistence (`X-Cart-Token`), tiered promo codes, sales tax, and free shipping calculations.
