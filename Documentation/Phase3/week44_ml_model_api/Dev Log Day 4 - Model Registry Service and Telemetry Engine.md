---
date: 2026-10-29
project: PulseInfer ML Model Serving API
topic: Day 4 - Model Registry Service and Telemetry Engine
Tags:
  - "[[ML-Model-Serving]]"
  - "[[Model-Registry]]"
  - "[[Hot-Reloading]]"
  - "[[Telemetry]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 44 - DAY 4

**Core Objective:** Design and implement the production `ModelRegistry` (lifecycle state machine, zero-downtime atomic hot-swapping between model versions, and schema validation) and `TelemetryEngine` (sliding-window histogram calculating p50/p90/p95/p99 latency percentiles, error rates, and throughput), validated by 71 passing unit tests with 97% coverage and complete quality gate verification.

---

## 1. The Big Picture and Simple Explanation

In production machine learning systems, deploying updated model artifacts or tuning decision thresholds cannot incur downtime or drop active client connections. Furthermore, operating an inference gateway requires comprehensive real-time latency observability:
1. **Zero-Downtime Hot-Swapping:** When transitioning a model from `v1` to `v2`, the gateway must load new weights into memory, run warm-up dry runs, spin up a new batcher, and atomically swap the runtime pointers while in-flight queries continue executing without interruption.
2. **Strict Contract Validation:** Payloads must be validated against declared input schemas before reaching numerical pipelines to prevent malformed matrices or unhandled exceptions.
3. **Statistical Percentile Telemetry:** Average latency hides critical tail spikes. Computing exact percentiles (p50, p90, p95, p99) over sliding observation windows provides actionable visibility into system performance under varying load.

```
+-----------------------------------------------------------------------------+
|                     PulseInfer ModelRegistry Orchestration                  |
|                                                                             |
|  [Client Request] ---> [validate_input]                                     |
|                               |                                             |
|                               v                                             |
|                     [InferenceCache Lookup]                                 |
|                         /            \                                      |
|            Cache Hit   /              \   Cache Miss                        |
|                       v                v                                    |
|              [Return Result]    [Model Batcher / Direct]                    |
|              (< 0.1ms p99)             |                                    |
|                                        v                                    |
|                             [model.instance.predict]                        |
|                                        |                                    |
|                                        +--> [Store in Cache]               |
|                                        +--> [Record in TelemetryEngine]    |
|                                        +--> [Persist PredictionLogDAO]     |
|                                                                             |
|  -------------------------------------------------------------------------  |
|  [Hot-Swap Workflow: v1 -> v2]                                              |
|                                                                             |
|  1. State: HOT_RELOADING (v1 continues serving in-flight traffic)           |
|  2. Load v2 weights from disk artifact into fresh BaseModel instance        |
|  3. Run dry-run warmup inference on v2                                      |
|  4. Spin up and start v2 DynamicBatcher                                     |
|  5. Atomic pointer swap: model.instance = v2, model.batcher = v2_batcher    |
|  6. Invalidate InferenceCache for model                                     |
|  7. Update active_version in SQLite database (ModelDAO)                     |
|  8. Stop retired v1 batcher (allow in-flight requests to complete)          |
|  9. State: ACTIVE (v2 now serving all new traffic)                          |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Model Lifecycle State Machine

The `ModelRegistry` maintains each model through a strict lifecycle:
- **LOADING:** Instantiating the domain model class and deserializing weight artifacts from the vault.
- **WARMING_UP:** Executing synthetic dry-run inference to prime code paths and JIT caches.
- **ACTIVE:** Fully operational and serving live traffic through the `DynamicBatcher`.
- **HOT_RELOADING:** Transitioning between versions while in-flight requests on the old version continue executing.
- **UNLOADED:** Model removed from the active runtime.

### TelemetryEngine: Sliding-Window Statistical Aggregation

The `TelemetryEngine` maintains a bounded sliding window of latency observations:
- Percentiles are computed by sorting the latency buffer and indexing directly: $p_k = \mathbf{L}[\lfloor k \cdot |\mathbf{L}| \rfloor]$ for p50, p90, p95, and p99.
- Throughput (requests per second) is estimated using observation timestamps: $\text{RPS} = \frac{|\mathbf{L}|}{t_{\text{newest}} - t_{\text{oldest}}}$.
- Cache hit ratio and error rate are tracked continuously: $\text{Hit Ratio} = \frac{\text{Cache Hits}}{\text{Total Requests}}$ and $\text{Error Rate} = \frac{\text{Failed Requests}}{\text{Total Requests}}$.

### Zero-Downtime Hot-Swapping Procedure

When `hot_swap_version(model_id, new_version, conn)` is triggered:
1. Validates that the target version is cataloged in `model_versions`.
2. Marks the existing model container state as `HOT_RELOADING`.
3. Loads the replacement version artifact from disk into a completely new `BaseModel` instance.
4. Executes a synthetic warmup payload matching the model's domain.
5. Allocates and starts a fresh `DynamicBatcher`.
6. Acquires the model's re-entrant lock and atomically updates the runtime pointers (`instance`, `batcher`, `active_version`, `input_schema`, `output_schema`, `metrics`).
7. Purges all stale cache entries for this model (`InferenceCache.invalidate_model`).
8. Updates the database record via `ModelDAO.update_active_version`.
9. Safely stops the retired batcher (`old_batcher.stop(timeout=2.0)`), allowing any in-flight requests to complete cleanly.

---

## 3. Verification & Quality Gates Report

The test suite verified registry lifecycle management, telemetry aggregation, and hot-swap mechanics across all quality gates:

| Quality Gate | Tool | Target / Threshold | Result | Status |
| :--- | :--- | :--- | :--- | :--- |
| Linting | Flake8 | 0 violations (88-char limit) | 0 warnings / 0 errors | PASSED |
| Code Formatting | Black | PEP 8 / 88 columns | 100% compliant | PASSED |
| Import Sorting | isort | Black profile compatibility | 100% compliant | PASSED |
| Security Scanning | Bandit | Zero medium/high vulnerabilities | 0 issues found | PASSED |
| Test Coverage | Pytest | >= 90% statement coverage | 71/71 passed (97% coverage) | PASSED |

### Key Module Coverage
| Module | Statements | Miss | Coverage |
| :--- | :--- | :--- | :--- |
| `app/services/registry_service.py` | 254 | 24 | 91% |
| `app/services/telemetry_service.py` | 87 | 0 | 100% |
| `app/services/cache_service.py` | 100 | 0 | 100% |
| `app/services/batcher_service.py` | 124 | 2 | 98% |

---

## 4. Key Engineering Insights & Decisions

- **Atomic Pointer Swap Under Re-Entrant Lock:** The hot-swap procedure acquires a per-model `RLock` before updating `instance`, `batcher`, and `active_version` references in a single critical section. Requests arriving before the swap execute on v1; requests arriving after execute on v2. No requests are blocked or dropped.
- **Schema Validation as Early Rejection Gate:** Validating payloads against declared input schemas before compute allocation ensures that malformed inputs fail with descriptive errors at zero compute cost (< 0.01 ms), preventing wasted matrix operations or cryptic numerical exceptions.
- **Dual Telemetry Persistence Strategy:** In-memory sliding-window percentiles serve instant dashboard queries (< 0.1 ms lookup), while persistent SQLite `prediction_logs` provide historical auditability and compliance for model governance.

---

## 5. Next Steps

- Expose external REST API endpoints for inference, model management, and telemetry.
- Implement `POST /api/v1/models/<model_id>/predict` with schema validation, cache lookup, and batcher dispatch.
- Build `POST /api/v1/models/<model_id>/swap-version` for zero-downtime hot-swap triggers.
- Construct `GET /api/v1/telemetry` for global and per-model latency percentiles.
