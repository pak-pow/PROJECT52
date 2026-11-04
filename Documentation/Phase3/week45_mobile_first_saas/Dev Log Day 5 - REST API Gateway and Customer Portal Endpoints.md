---
date: 2026-11-04
project: PulseSub Mobile-First SaaS Subscription & Entitlements Platform
topic: Day 5 - REST API Gateway and Customer Portal Endpoints
Tags:
  - "[[SaaS-Billing]]"
  - "[[REST-API]]"
  - "[[Customer-Portal]]"
  - "[[Bearer-Auth]]"
  - "[[Printable-Receipts]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 45 - DAY 5

**Core Objective:** Construct the comprehensive REST API Gateway and customer portal endpoints for authentication, plan catalog browsing, subscription provisioning, mid-cycle tier changes with proration previews, metered usage tracking, and printable HTML receipt generation, backed by 125 passing tests with 96% code coverage and 0 linter/security issues.

---

## 1. The Big Picture and Simple Explanation

In modern subscription platforms, backend business logic (FSMs, proration engines, entitlements, usage accumulators, and webhooks) must be exposed cleanly to mobile clients, desktop browsers, and third-party integrations via a resilient, intuitive REST API. Day 5 bridges our domain layer with external clients by introducing six focused API blueprints:

1. **Authentication & Session Management (`/api/v1/auth`):** Provides user registration, password verification with deterministic salt hashing, bearer token generation with HMAC-SHA256 signatures, and profile resolution (`/me`).
2. **Public Plan Catalog (`/api/v1/plans`):** Allows clients to query available subscription tiers (Free, Starter, Pro, Enterprise) along with pricing, monthly/annual intervals, numeric quotas, and boolean feature flags.
3. **Subscription Lifecycle & Proration (`/api/v1/subscriptions`):**
   - Enables users to subscribe to tiers with automated initial invoicing.
   - Provides dry-run `/change-tier` previewing (calculating exact unused credit, new plan charges, and net amount due without modifying database state).
   - Executes tier transitions, issuing prorated invoices and updating periods atomically.
   - Handles scheduled cancellations (`cancel_at_period_end`) and instant reactivations (`/resume`).
4. **Feature Entitlements & Diagnostics (`/api/v1/entitlements`):** Exposes `/my-features` for full client state hydration and `/check` for quick boolean feature checks.
5. **Metered Consumption & Quotas (`/api/v1/usage`):** Enables client apps to record billable consumption with idempotency keys while injecting transparent quota headers (`X-Quota-Limit`, `X-Quota-Used`, `X-Quota-Remaining`, `X-Quota-Alert`).
6. **Billing History & Receipts (`/api/v1/billing`):** Provides invoice history, structured receipt metadata, simulated invoice settlement with automated subscription reactivation, and a standalone, printable HTML receipt with `@media print` styling.

```
+-----------------------------------------------------------------------------+
|                      PulseSub REST API Gateway Architecture                 |
|                                                                             |
|      Mobile Client / Desktop Browser / Sandbox Console                      |
|                |                                                            |
|                | Authorization: Bearer pst_...  or  X-User-ID: usr_...       |
|                v                                                            |
|   +---------------------------------------------------------------------+   |
|   |                         Flask App Router                            |   |
|   +----+----------+------------+------------+-------------+---------+---+   |
|        |          |            |            |             |         |       |
|        v          v            v            v             v         v       |
|    /auth      /plans     /subscriptions  /entitlements  /usage   /billing   |
|   (JWT-lite) (Catalog)   (FSM+Proration)  (Gatekeeper)  (Quota)  (Invoices) |
|        |          |            |            |             |         |       |
|        +----------+------------+------------+-------------+---------+       |
|                                |                                            |
|                                v                                            |
|   +---------------------------------------------------------------------+   |
|   |                     PulseSub Core Domain Layer                      |   |
|   |   - AuthService            - SubscriptionFSM       - Proration      |   |
|   |   - InvoiceService         - EntitlementsService   - UsageService   |   |
|   +---------------------------------------------------------------------+   |
|                                |                                            |
|                                v                                            |
|   +---------------------------------------------------------------------+   |
|   |                  SQLite Database (WAL Mode Enabled)                 |   |
|   |   - users     - plans      - subscriptions   - invoices  - usage    |   |
|   +---------------------------------------------------------------------+   |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Signed Bearer Token Authentication (`auth_service.py`)
To enable frictionless mobile and web authentication without heavyweight third-party dependencies, `AuthService` issues lightweight, tamper-proof bearer tokens (`pst_<b64_payload>.<signature>`) validated in constant time:

```python
def create_auth_token(
    user_id: str,
    email: str,
    role: str = "user",
    secret_key: Optional[str] = None,
    expires_in_seconds: int = 604800,
) -> str:
    key = secret_key or Config.SECRET_KEY
    now = int(time.time())
    payload = {
        "uid": user_id,
        "eml": email,
        "rol": role,
        "iat": now,
        "exp": now + expires_in_seconds,
    }
    raw_json = json.dumps(payload, separators=(",", ":"))
    b64_payload = base64.urlsafe_b64encode(raw_json.encode("utf-8")).decode().rstrip("=")
    sig = hmac.new(
        key.encode("utf-8"),
        b64_payload.encode("utf-8"),
        hashlib.sha256,
    ).hexdigest()[:32]
    return f"pst_{b64_payload}.{sig}"
```

### Dry-Run Proration Preview & Tier Switching (`subscriptions.py`)
Upgrading or downgrading a plan requires transparency. PulseSub supports `preview_only: true` on `/change-tier`, allowing clients to preview the exact unused credit and net charge before executing the change:

```python
@subscriptions_bp.route("/change-tier", methods=["POST"])
def change_tier():
    user_id = _resolve_user_id()
    payload = request.get_json(silent=True) or {}
    new_plan_id = payload.get("new_plan_id")
    preview_only = bool(payload.get("preview_only", False))

    proration = calculate_proration(
        current_plan=current_plan,
        new_plan=new_plan,
        subscription=current_sub,
        target_interval=target_interval,
    )

    if preview_only:
        return jsonify({"preview_only": True, "proration": proration.to_dict()}), 200

    invoice = invoice_service.create_proration_invoice(
        user_id=user_id,
        subscription_id=current_sub.id,
        proration_result=proration,
        auto_pay=True,
    )
    updated_sub = SubscriptionFSM(current_sub, conn).change_tier(
        new_plan_id=new_plan.id,
        new_interval=target_interval,
    )
    return jsonify({
        "subscription": updated_sub.to_dict(),
        "proration": proration.to_dict(),
        "invoice": invoice.to_dict(),
        "message": "Subscription tier updated successfully.",
    }), 200
```

### Standalone Printable HTML Receipts (`billing.py`)
Endpoint `GET /api/v1/billing/invoices/<id>/html` renders a responsive, print-optimized HTML receipt with clean CSS variables, invoice itemization, and `@media print` rules for browser PDF export.

---

## 3. Verification & Quality Assurance

All quality gates passed with zero warnings:
1. **Pytest Test Suite:** 125 tests passing in 3.92s (expanded from 90).
2. **Code Coverage:** 96% line coverage across the entire codebase.
3. **Flake8 Linter:** 0 errors at 88-char limits across 53 files.
4. **Black & isort:** 100% compliant formatting.
5. **Bandit Security Audit:** 0 vulnerabilities across 3,582 LOC.

```text
============================= test session starts =============================
collected 125 items

tests/test_auth_routes.py::test_register_route_success PASSED            [  0%]
tests/test_auth_routes.py::test_register_route_missing_fields PASSED     [  1%]
tests/test_auth_routes.py::test_register_route_duplicate_email PASSED    [  2%]
tests/test_auth_routes.py::test_login_route_success PASSED               [  3%]
tests/test_auth_routes.py::test_login_route_invalid_credentials PASSED   [  4%]
tests/test_auth_routes.py::test_me_route_authenticated PASSED            [  4%]
tests/test_auth_routes.py::test_me_route_nonexistent_user PASSED         [  5%]
tests/test_auth_service.py::test_password_hashing_and_verification PASSED [  6%]
tests/test_auth_service.py::test_token_creation_and_verification PASSED  [  7%]
tests/test_auth_service.py::test_token_tampering_fails PASSED            [  8%]
tests/test_auth_service.py::test_token_expiration PASSED                 [  8%]
tests/test_auth_service.py::test_auth_service_registration_and_login PASSED [  9%]
tests/test_billing_routes.py::test_list_invoices_for_user PASSED         [ 10%]
tests/test_billing_routes.py::test_get_invoice_detail PASSED             [ 11%]
tests/test_billing_routes.py::test_get_invoice_receipt_json PASSED       [ 12%]
tests/test_billing_routes.py::test_get_invoice_html_receipt PASSED       [ 12%]
tests/test_billing_routes.py::test_get_nonexistent_invoice_returns_404 PASSED [ 13%]
tests/test_billing_routes.py::test_pay_invoice_and_reactivate_subscription PASSED [ 14%]
tests/test_connection.py::test_get_db_connection_in_memory PASSED        [ 15%]
tests/test_connection.py::test_get_db_connection_file_based PASSED       [ 16%]
tests/test_connection.py::test_row_factory_access PASSED                 [ 16%]
tests/test_connection.py::test_close_db_connection PASSED                [ 17%]
tests/test_connection.py::test_init_db_missing_schema_raises PASSED      [ 18%]
tests/test_decorators.py::test_require_entitlement_granted PASSED        [ 19%]
tests/test_decorators.py::test_require_entitlement_denied PASSED         [ 20%]
tests/test_decorators.py::test_require_entitlement_denied_for_enterprise PASSED [ 20%]
tests/test_decorators.py::test_check_quota_permitted_and_headers_injected PASSED [ 21%]
tests/test_decorators.py::test_check_quota_exceeded_returns_402 PASSED   [ 22%]
tests/test_entitlements_routes.py::test_get_my_features_snapshot PASSED  [ 23%]
tests/test_entitlements_routes.py::test_check_feature_access PASSED      [ 24%]
tests/test_entitlements_routes.py::test_check_feature_missing_parameter PASSED [ 24%]
tests/test_entitlements_service.py::test_get_active_subscription_demo_user PASSED [ 25%]
tests/test_entitlements_service.py::test_get_user_plan_subscribed_and_free PASSED [ 26%]
tests/test_entitlements_service.py::test_check_feature_access PASSED     [ 27%]
tests/test_entitlements_service.py::test_can_consume_metric PASSED       [ 28%]
tests/test_entitlements_service.py::test_get_user_entitlements_snapshot PASSED [ 28%]
tests/test_health.py::test_health_endpoint_healthy PASSED                [ 29%]
tests/test_health.py::test_health_degraded_when_db_fails PASSED          [ 30%]
tests/test_health.py::test_security_headers_present PASSED               [ 31%]
tests/test_health.py::test_static_index_route PASSED                     [ 32%]
tests/test_health.py::test_static_css_route PASSED                       [ 32%]
tests/test_health.py::test_static_js_route PASSED                        [ 33%]
tests/test_health.py::test_config_retrieval PASSED                       [ 34%]
tests/test_invoice_service.py::test_sequential_invoice_numbering PASSED  [ 35%]
tests/test_invoice_service.py::test_create_auto_pay_invoice PASSED       [ 36%]
tests/test_invoice_service.py::test_create_open_invoice PASSED           [ 36%]
tests/test_invoice_service.py::test_create_proration_invoice PASSED      [ 37%]
tests/test_invoice_service.py::test_mark_as_paid PASSED                  [ 38%]
tests/test_invoice_service.py::test_mark_as_failed_and_uncollectible PASSED [ 39%]
tests/test_invoice_service.py::test_void_invoice PASSED                  [ 40%]
tests/test_invoice_service.py::test_get_invoices_by_user PASSED          [ 40%]
tests/test_invoice_service.py::test_nonexistent_invoice_operations PASSED [ 41%]
tests/test_models.py::test_plan_serialization PASSED                     [ 42%]
tests/test_models.py::test_subscription_serialization PASSED             [ 43%]
tests/test_models.py::test_invoice_serialization_and_from_row PASSED     [ 44%]
tests/test_models.py::test_usage_record_serialization_and_from_row PASSED [ 45%]
tests/test_models.py::test_webhook_event_record_serialization_and_from_row PASSED [ 46%]
tests/test_plans_routes.py::test_list_plans_returns_all_active_tiers PASSED [ 46%]
tests/test_plans_routes.py::test_get_single_plan_found PASSED            [ 47%]
tests/test_plans_routes.py::test_get_single_plan_not_found PASSED        [ 48%]
tests/test_proration.py::test_proration_mid_cycle_upgrade PASSED         [ 49%]
tests/test_proration.py::test_proration_mid_cycle_downgrade PASSED       [ 50%]
tests/test_proration.py::test_proration_at_start_of_period PASSED        [ 50%]
tests/test_proration.py::test_proration_at_end_of_period PASSED          [ 51%]
tests/test_proration.py::test_proration_free_tier_transitions PASSED     [ 52%]
tests/test_proration.py::test_proration_monthly_to_annual_interval PASSED [ 53%]
tests/test_schema.py::test_schema_tables_exist PASSED                    [ 54%]
tests/test_schema.py::test_schema_indexes_exist PASSED                   [ 54%]
tests/test_users_unique_email_constraint PASSED                          [ 55%]
tests/test_plans_tier_and_price_check_constraints PASSED                 [ 56%]
tests/test_foreign_key_cascade_deletion PASSED                           [ 56%]
tests/test_seed_plans_created PASSED                                      [ 57%]
tests/test_seed_database_idempotency PASSED                               [ 58%]
tests/test_demo_user_and_subscription PASSED                              [ 59%]
tests/test_hash_password_deterministic PASSED                             [ 60%]
tests/test_subscription_fsm.py::test_fsm_initial_status PASSED           [ 60%]
tests/test_subscription_fsm.py::test_fsm_mark_past_due PASSED            [ 61%]
tests/test_subscription_fsm.py::test_fsm_activate_from_past_due PASSED   [ 62%]
tests/test_subscription_fsm.py::test_fsm_cancel_at_period_end PASSED     [ 63%]
tests/test_subscription_fsm.py::test_fsm_cancel_immediate PASSED         [ 64%]
tests/test_subscription_fsm.py::test_fsm_resume_scheduled_cancel PASSED  [ 64%]
tests/test_subscription_fsm.py::test_fsm_resume_immediate_canceled_before_expiry PASSED [ 65%]
tests/test_subscription_fsm.py::test_fsm_resume_immediate_canceled_after_expiry_raises PASSED [ 66%]
tests/test_subscription_fsm.py::test_fsm_resume_active_non_canceled_raises PASSED [ 67%]
tests/test_subscription_fsm.py::test_fsm_expire_terminal_state PASSED    [ 68%]
tests/test_subscription_fsm.py::test_fsm_renew_active PASSED             [ 68%]
tests/test_subscription_fsm.py::test_fsm_change_tier PASSED              [ 69%]
tests/test_subscription_fsm.py::test_fsm_database_persistence PASSED     [ 70%]
tests/test_subscriptions_routes.py::test_get_current_subscription_demo_user PASSED [ 71%]
tests/test_subscriptions_routes.py::test_get_current_subscription_unsubscribed_user PASSED [ 72%]
tests/test_subscriptions_routes.py::test_subscribe_success_and_duplicate_prevention PASSED [ 72%]
tests/test_subscriptions_routes.py::test_subscribe_validation_errors PASSED [ 73%]
tests/test_subscriptions_routes.py::test_change_tier_preview_and_execute PASSED [ 74%]
tests/test_subscriptions_routes.py::test_change_tier_same_plan_and_no_subscription PASSED [ 75%]
tests/test_subscriptions_routes.py::test_cancel_and_resume_subscription PASSED [ 76%]
tests/test_usage_routes.py::test_record_usage_valid PASSED               [ 76%]
tests/test_usage_routes.py::test_record_usage_idempotent_duplicate PASSED [ 77%]
tests/test_usage_routes.py::test_record_usage_validation_errors PASSED   [ 78%]
tests/test_usage_routes.py::test_get_usage_summary PASSED                [ 79%]
tests/test_usage_service.py::test_record_usage_standard PASSED           [ 80%]
tests/test_usage_service.py::test_record_usage_idempotency PASSED        [ 80%]
tests/test_usage_service.py::test_record_usage_invalid_quantity PASSED   [ 81%]
tests/test_usage_service.py::test_get_period_usage PASSED                [ 82%]
tests/test_usage_service.py::test_get_user_metric_usage_with_subscription PASSED [ 83%]
tests/test_usage_service.py::test_evaluate_quota_status_unlimited PASSED [ 84%]
tests/test_usage_service.py::test_evaluate_quota_status_threshold_alerts PASSED [ 84%]
tests/test_webhook_dispatcher.py::test_dispatch_subscription_created PASSED [ 85%]
tests/test_webhook_dispatcher.py::test_dispatch_subscription_updated_tier_and_status PASSED [ 86%]
tests/test_webhook_dispatcher.py::test_dispatch_subscription_deleted PASSED [ 87%]
tests/test_webhook_dispatcher.py::test_dispatch_invoice_paid_reactivates_past_due PASSED [ 88%]
tests/test_webhook_dispatcher.py::test_dispatch_invoice_payment_failed PASSED [ 88%]
tests/test_webhook_dispatcher.py::test_dispatch_idempotency_duplicate_events PASSED [ 89%]
tests/test_webhook_dispatcher.py::test_dispatch_unhandled_event PASSED   [ 90%]
tests/test_webhook_routes.py::test_webhook_route_missing_signature PASSED [ 91%]
tests/test_webhook_routes.py::test_webhook_route_invalid_signature PASSED [ 92%]
tests/test_webhook_routes.py::test_webhook_route_valid_dispatch_and_audit PASSED [ 92%]
tests/test_webhook_routes.py::test_webhook_route_malformed_json PASSED   [ 93%]
tests/test_webhook_routes.py::test_webhook_route_missing_event_id_or_type PASSED [ 94%]
tests/test_webhook_security.py::test_verify_valid_signature PASSED       [ 95%]
tests/test_webhook_security.py::test_verify_missing_header_raises PASSED [ 96%]
tests/test_webhook_security.py::test_verify_malformed_header_missing_parts PASSED [ 96%]
tests/test_webhook_security.py::test_verify_timestamp_out_of_tolerance PASSED [ 97%]
tests/test_webhook_security.py::test_verify_tampered_payload PASSED      [ 98%]
tests/test_webhook_security.py::test_verify_wrong_secret PASSED          [ 99%]
tests/test_webhook_security.py::test_verify_multiple_signatures_rollover PASSED [100%]

=============================== tests coverage ================================
Name                                   Stmts   Miss  Cover   Missing
--------------------------------------------------------------------
app\__init__.py                           41      2    95%   34, 39
app\billing\__init__.py                    6      0   100%
app\billing\invoice_service.py            92      3    97%   41-44
app\billing\proration.py                  70      4    94%   106, 110, 117, 136
app\billing\subscription_fsm.py          117      1    99%   192
app\billing\webhook_dispatcher.py        145     14    90%   94-108, 171-172, 186, 203, 205, 211, 223, 238
app\billing\webhook_security.py           43      1    98%   38
app\config\__init__.py                     2      0   100%
app\config\settings.py                    30      0   100%
app\db\__init__.py                         2      0   100%
app\db\connection.py                      42      2    95%   86-87
app\models\__init__.py                     2      0   100%
app\models\schemas.py                    141     10    93%   57, 60, 87-88, 94-95, 186-187, 264-265
app\routes\__init__.py                    10      0   100%
app\routes\auth.py                        59      1    98%   82
app\routes\billing.py                    102      2    98%   358, 434
app\routes\entitlements.py                30      0   100%
app\routes\health.py                      21      0   100%
app\routes\plans.py                       32      0   100%
app\routes\static_routes.py               26      3    88%   23, 44-45
app\routes\subscriptions.py              161     11    93%   41, 50-51, 229, 266, 364, 377-378, 422, 436-437
app\routes\usage.py                       48      0   100%
app\routes\webhooks.py                    49      0   100%
app\services\__init__.py                   5      0   100%
app\services\auth_service.py              92      2    98%   83-84
app\services\decorators.py                71      6    92%   23, 31, 34-36, 150
app\services\entitlements_service.py      57      1    98%   62
app\services\usage_service.py             55      0   100%
data\seed.py                              40      8    80%   244-252
--------------------------------------------------------------------
TOTAL                                   1591     71    96%
============================= 125 passed in 3.92s =============================
```

---

## 4. Next Steps

With Day 5 API endpoints fully verified and operational, **Day 6** will construct the mobile-first frontend application:
- Responsive Mobile-First App Shell with bottom navigation tab bar (Overview, Usage, Plans, Billing, Sandbox).
- Pricing Matrix Card Component with live Monthly vs. Annual toggle.
- Interactive Metered Usage Gauges with color-coded threshold alerts (80%, 90%, 100%).
- Customer Billing Portal with receipt modal and print trigger.
- Built-in Device Frame Simulator (iPhone 15, Pixel 8, iPad, Full Desktop).
- Interactive Webhook Event Simulator Console for real-time testing without live Stripe webhooks.
