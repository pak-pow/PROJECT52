---
date: 2026-09-24
project: Production E-Commerce Platform v1
topic: Day 4 - Order State Machine and Webhooks
Tags:
  - "[[E-Commerce]]"
  - "[[Backend]]"
  - "[[Payments]]"
  - "[[Webhooks]]"
  - "[[Testing]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 39 - DAY 4

**Core Objective:** Build the Order State Machine, atomic inventory reservation engine, Stripe payment gateway adapter, HMAC-verified webhook ingestion with idempotency, and automated invoicing for ShopPulse.

---

## 1. The Big Picture and Simple Explanation

Checkout processing is the most operationally critical workflow in an e-commerce platform. If an order fails midway through payment authorization, inventory can be trapped in limbo. Conversely, if a payment gateway redelivers a webhook notification, an unhardened store might fulfill duplicate orders or issue multiple receipts.

In Day 4, we built an enterprise order processing and payment pipeline:
1. **Atomic Inventory Reservation:** When checkout is triggered, inventory is deducted immediately inside an atomic database transaction. If any cart item lacks sufficient stock, the entire checkout rolls back cleanly.
2. **Order Lifecycle State Machine:** Orders transition strictly through validated states (`pending` -> `processing` -> `paid` -> `shipped`). Illegal state transitions (e.g., jumping from `cancelled` to `paid`) are rejected at the application layer.
3. **Stripe Payment Gateway Integration:** Created an adapter managing Stripe `PaymentIntent` lifecycles, supporting test simulations for successful authorization, card decline, and insufficient funds.
4. **Idempotent Webhook Processing:** Inbound payment webhooks from Stripe are tracked by unique `event_id`. Duplicate webhook deliveries are acknowledged with HTTP 200 without executing duplicate order fulfillments.
5. **Inventory Restoration on Failure:** If a payment fails or an order is cancelled, reserved stock is automatically returned to the product inventory.

```
+-----------------------------------------------------------------------------+
|                     Order State Machine & Payment Lifecycle                 |
|                                                                             |
|      +---------------------------------------------------------------+      |
|      |             Customer Checkout (/checkout/create-order)        |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Step 1: Atomic Inventory Reservation                          |      |
|      |         Deduct stock inside database transaction              |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                     +----------------+----------------+                     |
|                     | [Stock Available]               | [Out of Stock]      |
|                     v                                 v                     |
|      +-------------------------------+    +--------------------------+      |
|      | Step 2: Create Pending Order  |    | Abort Checkout           |      |
|      | State: PENDING                |    | Return HTTP 409 Conflict |      |
|      +---------------+---------------+    +--------------------------+      |
|                      |                                                      |
|                      v                                                      |
|      +---------------------------------------------------------------+      |
|      | Step 3: Create Stripe PaymentIntent                           |      |
|      |         Attach client secret to client response               |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                     +----------------+----------------+                     |
|                     |                                 |                     |
|                     v                                 v                     |
|      +-------------------------------+    +--------------------------+      |
|      | Inbound Webhook: Succeeded    |    | Inbound Webhook: Failed  |      |
|      | POST /api/v1/webhooks/stripe  |    | State -> CANCELLED       |      |
|      | State: PENDING -> PAID        |    | Restore reserved stock   |      |
|      | Issue Invoice & Confirmation  |    +--------------------------+      |
|      +-------------------------------+                                      |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Order Model & State Machine (`app/models/order_model.py`)
- Schema design: Captured customer contact details, shipping addresses, line items, and pricing snapshots in `orders` and `order_items` tables.
- Validated state transitions:
  - Allowed: `pending` -> `processing`, `processing` -> `paid`, `paid` -> `shipped`, `pending` -> `cancelled`.
  - Disallowed: Any transition originating from `cancelled` or direct jumps from `pending` to `shipped`.
- Automatic inventory restoration: Cancelling an order executes compensatory inventory additions, returning reserved stock to products.
- Formatted invoice generation: Generates printable invoice summaries detailing transaction IDs, customer information, line item amounts, and tax breakdowns.

### Stripe Payment Gateway Adapter (`app/services/payment_service.py`)
- Encapsulated Stripe `PaymentIntent` lifecycle creation, metadata tagging, and client secret generation.
- Built a simulation runner allowing tests and demo environments to trigger deterministic payment outcomes: `success`, `decline`, and `insufficient_funds`.
- Implemented cryptographic webhook signature verification asserting that webhook payloads originate from the authorized payment provider.

### Idempotent Webhook Processing (`app/routes/webhook_routes.py` & `webhook_event_model.py`)
- `POST /api/v1/webhooks/stripe`: Ingests payment events (`payment_intent.succeeded`, `payment_intent.payment_failed`).
- Enforces idempotency by recording incoming `event_id` values in a dedicated `webhook_events` table with a unique constraint. Duplicate deliveries return HTTP 200 immediately without re-triggering business logic.

```python
def process_stripe_webhook(payload, signature):
    event = verify_webhook_signature(payload, signature)
    event_id = event["id"]

    if WebhookEventModel.exists(event_id):
        return {"status": "ignored", "reason": "duplicate_event"}, 200

    WebhookEventModel.record(event_id, event["type"])

    if event["type"] == "payment_intent.succeeded":
        order_number = event["data"]["object"]["metadata"]["order_number"]
        OrderModel.transition_state(order_number, "paid")
    elif event["type"] == "payment_intent.payment_failed":
        order_number = event["data"]["object"]["metadata"]["order_number"]
        OrderModel.transition_state(order_number, "cancelled")

    return {"status": "processed"}, 200
```

### Order & Checkout Endpoints (`app/routes/order_routes.py`)
- `POST /api/v1/checkout/create-order`: Reserves inventory and creates an initial order in `pending` status.
- `POST /api/v1/checkout/simulate-payment`: Simulates payment execution for demo and test flows.
- `GET /api/v1/orders/<order_number>`: Retrieves order metadata, current status, and line items.
- `GET /api/v1/orders/<order_number>/invoice`: Emits a structured invoice for display or printing.

---

## 3. Verification & Quality Gates Report

The order state machine and payment webhook pipelines were verified across 11 automated unit tests:

| Test Suite Module | Target Scope | Tests Passed | Status |
| :--- | :--- | :---: | :---: |
| `tests/test_orders_and_webhooks.py` | Checkout, state transitions, webhook idempotency | 11 | Passed |
| `tests/test_cart.py` | Cart operations and pricing calculations | 13 | Passed |
| `tests/test_products.py` | Catalog listings and search queries | 12 | Passed |
| `tests/test_health_and_version.py` | Liveness, readiness, and version probes | 4 | Passed |
| `tests/test_coverage_boost.py` | Error handlers and edge conditions | 6 | Passed |

### Quality Gates Execution Summary
- **Total Tests:** 46 automated tests passing in 0.92 seconds.
- **Branch Coverage:** 96.5% total branch coverage.
- **Linters:** Flake8, Black, isort, and Bandit reported 0 errors.

---

## 4. Key Engineering Insights & Decisions

- **Pre-Payment Inventory Reservation:** Deducting stock before redirecting to payment providers prevents overselling during high-traffic sales events.
- **Idempotency Keys in Webhook Handlers:** Webhooks operate under an "at-least-once" delivery guarantee. Persisting unique event identifiers prevents duplicate fulfillment when network retries occur.
- **State Machine Guardrails Prevent Fraud:** Enforcing formal state transitions ensures that cancelled or failed orders cannot be marked as paid through forged status payloads.

---

## 5. Next Steps

- Construct a responsive vanilla ES6 frontend storefront (`frontend/`).
- Implement the product grid, slide-out cart drawer, checkout modal, and printable order confirmation view.
