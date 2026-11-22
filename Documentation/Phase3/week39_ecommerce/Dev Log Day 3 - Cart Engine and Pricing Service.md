---
date: 2026-09-23
project: Production E-Commerce Platform v1
topic: Day 3 - Cart Engine and Pricing Service
Tags:
  - "[[E-Commerce]]"
  - "[[Backend]]"
  - "[[Pricing]]"
  - "[[Cart]]"
  - "[[Testing]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 39 - DAY 3

**Core Objective:** Implement the session-based Cart Engine and authoritative Pricing Service for ShopPulse, featuring cart token persistence, integer cent math, tiered promo code validation, dynamic sales tax, and free shipping threshold calculations.

---

## 1. The Big Picture and Simple Explanation

In e-commerce architectures, a shopping cart is a financial calculation engine. If pricing arithmetic runs in the client browser, malicious users can manipulate item costs or discounts before checkout. Furthermore, relying on floating-point arithmetic introduces fractional cent rounding errors that cause ledger mismatches.

In Day 3, we built an authoritative server-side cart and pricing system:
1. **Server-Side Financial Truth:** The client application never dictates product prices or discount amounts. The browser only submits product IDs and quantities; the backend computes line subtotals, validates coupons, and calculates taxes.
2. **Session Cart Token Tracking:** Shoppers are identified via a persistent `X-Cart-Token` UUID header. This grants guests a persistent cart across browser refreshes without requiring account registration.
3. **Integer Cent Currency Engine:** All monetary values—unit prices, item subtotals, discount deductions, tax amounts, and shipping fees—are computed and stored as integer cents.
4. **Deterministic Pricing Policies:** Implemented validated promo codes (`DISCOUNT10` for 10%, `SAVE20` for 20%, `WELCOME5` for $5 off), automated 8.25% sales tax calculation, and a $50.00 free shipping threshold ($10 standard shipping applied otherwise).

```
+-----------------------------------------------------------------------------+
|                     Authoritative Cart & Pricing Architecture               |
|                                                                             |
|      +---------------------------------------------------------------+      |
|      |             Client HTTP Request (GET / POST /api/v1/cart)     |      |
|      |             Header: X-Cart-Token: <UUIDv4>                    |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Flask Cart Blueprint (app/routes/cart_routes.py)              |      |
|      | - Resolves or generates cart session token                    |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Cart Model Layer (app/models/cart_model.py)                   |      |
|      | - Reads cart_items and joins products                         |      |
|      | - Validates available inventory stock before item addition    |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Authoritative Pricing Service (app/services/pricing_service)  |      |
|      | - Subtotal: sum(price_cents * quantity)                       |      |
|      | - Promo: apply DISCOUNT10 / SAVE20 / WELCOME5                 |      |
|      | - Shipping: $0 if subtotal >= 5000 cents, else $10.00 (1000)  |      |
|      | - Tax: round(subtotal_after_discount * 0.0825)                |      |
|      | - Total: subtotal_after_discount + shipping + tax             |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Emitted JSON Response with Item Breakdown & Authoritative Math|      |
|      +---------------------------------------------------------------+      |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Session Cart Management (`app/models/cart_model.py`)
- Persistent token tracking: Generated UUIDv4 cart tokens stored in the `carts` table with session expiration timestamps.
- Line item mutations: Built functions to add products, adjust quantities, delete items, and purge active carts.
- Real-time stock validation: Enforced inventory checks during cart additions so customers cannot reserve more units than are currently available in stock.

### Authoritative Pricing Service (`app/services/pricing_service.py`)
- Standardized all currency calculations into pure deterministic functions operating exclusively on integer cents.
- Promotional discount engine:
  - `DISCOUNT10`: 10% percentage discount on items subtotal.
  - `SAVE20`: 20% percentage discount on items subtotal.
  - `WELCOME5`: Fixed $5.00 (500 cents) discount capped at subtotal amount.
- Sales tax calculation: Computes estimated state tax at a flat 8.25% applied to the discounted subtotal.
- Shipping rule engine: Awards free shipping when subtotal reaches $50.00 (5,000 cents), or assesses standard $10.00 (1,000 cents) shipping for orders below the threshold.

```python
def calculate_cart_pricing(items, promo_code=None):
    subtotal_cents = sum(item["price_cents"] * item["quantity"] for item in items)
    discount_cents = calculate_discount(subtotal_cents, promo_code)
    taxable_amount = max(0, subtotal_cents - discount_cents)
    shipping_cents = 0 if subtotal_cents >= 5000 else 1000
    tax_cents = int(round(taxable_amount * 0.0825))
    total_cents = taxable_amount + shipping_cents + tax_cents

    return {
        "subtotal_cents": subtotal_cents,
        "discount_cents": discount_cents,
        "shipping_cents": shipping_cents,
        "tax_cents": tax_cents,
        "total_cents": total_cents,
        "promo_code": promo_code
    }
```

### Cart API Endpoints (`app/routes/cart_routes.py`)
- `GET /api/v1/cart`: Returns active cart line items accompanied by the full pricing summary.
- `POST /api/v1/cart/items`: Adds a product ID and requested quantity to the active session.
- `PATCH /api/v1/cart/items/<id>`: Updates the quantity of a specific cart line item.
- `DELETE /api/v1/cart/items/<id>`: Removes a single line item from the cart.
- `DELETE /api/v1/cart`: Empties all items from the current cart session.
- `POST /api/v1/cart/promo` & `DELETE /api/v1/cart/promo`: Applies or removes promotional coupons.

---

## 3. Verification & Quality Gates Report

The cart engine and pricing service were verified using 13 automated unit tests:

| Test Suite Module | Target Scope | Tests Passed | Status |
| :--- | :--- | :---: | :---: |
| `tests/test_cart.py` | Cart mutations, promo validation, tax and shipping math | 13 | Passed |
| `tests/test_products.py` | Catalog listing, search, filtering, and detail APIs | 12 | Passed |
| `tests/test_health_and_version.py` | Liveness, readiness, and version probes | 4 | Passed |
| `tests/test_coverage_boost.py` | Error handlers and edge conditions | 6 | Passed |

### Quality Gates Execution Summary
- **Total Tests:** 35 automated tests passing in 0.64 seconds.
- **Branch Coverage:** 97.8% total branch coverage.
- **Linters:** Flake8, Black, isort, and Bandit reported 0 errors.

---

## 4. Key Engineering Insights & Decisions

- **Client Price Trust is a Security Flaw:** Never accept price fields submitted by frontend clients. Computing line item costs strictly from authoritative database rows prevents cart tampering and zero-cost checkout exploits.
- **Header-Based Cart Sessions:** Using custom headers (`X-Cart-Token`) rather than cookies avoids cross-site cookie restrictions (SameSite) and enables seamless testing from mobile clients and REST API tools.
- **Integer Math Guarantees Financial Consistency:** Calculating sales taxes and discounts using integer cents eliminates floating-point representation anomalies in accounting records.

---

## 5. Next Steps

- Engineer the Order State Machine and atomic inventory reservation engine.
- Integrate the Stripe payment gateway adapter and construct HMAC-verified webhook ingestion with idempotency tracking.
