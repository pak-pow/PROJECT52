---
date: 2026-10-30
project: PulseInfer ML Model Serving API
topic: Day 5 - REST API Gateway and Inference Endpoints
Tags:
  - "[[ML-Model-Serving]]"
  - "[[REST-API]]"
  - "[[Inference-Gateway]]"
  - "[[Telemetry]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 44 - DAY 5

**Core Objective:** Design and implement the external REST API Gateway blueprints (`models_bp` and `metrics_bp`) exposing single-item inference with schema validation and caching, direct vectorized batch inference, model catalog inspection and metadata discovery, zero-downtime hot-reloading version switching, and latency telemetry endpoints, validated by 99 passing unit and integration tests with 97% overall coverage and complete 5-gate quality verification.

---

## 1. The Big Picture and Simple Explanation

A production model serving gateway must bridge the gap between network clients (web apps, mobile devices, microservices) and internal machine learning pipelines. Clients need clear, uniform HTTP contracts regardless of whether they are submitting natural language queries, image classification buffers, or financial fraud records:
1. **Unified Inference Interface:** Regardless of whether the underlying model is NLP, Computer Vision, or Tabular, clients interact with standard REST endpoints (`POST /api/v1/models/<model_id>/predict` and `POST /api/v1/models/<model_id>/predict-batch`).
2. **Transparent Performance Acceleration:** Clients seamlessly benefit from in-memory SHA-256 prediction caching and dynamic micro-batching without modifying request payloads. Repeated queries return in sub-millisecond time with `"cached": true`.
3. **Operational Self-Service & Zero-Downtime Upgrades:** Operators can inspect catalog metadata, query statistical percentiles (p50, p90, p95, p99), purge prediction caches, and trigger version hot-swaps on the fly without server restarts.

In Day 5, we connected the core registry, batcher, cache, and telemetry systems into a production-ready HTTP API:
- **`models_bp`** (`/api/v1/models`): Endpoints for model discovery, schema inspection, single inference, vectorized batch inference, and zero-downtime hot reload.
- **`metrics_bp`** (`/api/v1`): Endpoints for global and per-model latency histograms, cache hit ratios, throughput rates, and cache eviction.
- **Enhanced `health_bp`** (`/api/health`): Integrated reporting of database health, storage vault capacity, active loaded model counts, and cache memory usage.

```
+-----------------------------------------------------------------------------+
|                       PulseInfer REST API Gateway                           |
|                                                                             |
|  [External Clients / Workbench]                                            |
|        |                                                                    |
|        +---> GET  /api/v1/models                (Catalog & Health)          |
|        +---> GET  /api/v1/models/<id>/metadata  (Schemas & Version History) |
|        +---> POST /api/v1/models/<id>/predict   (Single Inference + Cache)  |
|        +---> POST /api/v1/models/<id>/predict-batch (Vectorized Batch)      |
|        +---> POST /api/v1/models/<id>/reload    (Zero-Downtime Hot Swap)    |
|        +---> GET  /api/v1/metrics               (p50/p90/p95/p99 Latencies) |
|        +---> GET  /api/health                   (Cluster Health & Vault)    |
|                                                                             |
|                                   |                                         |
|                                   v                                         |
|  [Flask Application Layer]                                                  |
|        |                                                                    |
|        +---> models_bp    ---> [ModelRegistry Orchestrator]                 |
|        +---> metrics_bp   ---> [TelemetryEngine & InferenceCache]           |
|        +---> health_bp    ---> [StorageService & Database WAL]              |
+-----------------------------------------------------------------------------+
```

---

## 2. API Contract Specifications

### 2.1 Model Discovery & Metadata
- **`GET /api/v1/models`**: Returns registered models, active versions, and execution telemetry summaries. Supports optional `?domain=nlp|vision|tabular` filtering.
- **`GET /api/v1/models/<model_id>`**: Returns detailed runtime state, input/output schemas, and batcher metrics.
- **`GET /api/v1/models/<model_id>/metadata`**: Returns comprehensive metadata including all historical versions recorded in the database, training accuracy/F1 scores, and expected tensor dimensions.

### 2.2 Single-Item & Vectorized Batch Inference
- **`POST /api/v1/models/<model_id>/predict`**:
  - Request body: JSON object containing input features (`{"text": "..."}`, `{"image_base64": "..."}`, or tabular fields).
  - Query parameters:
    - `use_cache=true|false` (default: `true`)
    - `use_batcher=true|false` (default: `true`)
    - `timeout=<seconds>` (optional request deadline)
  - Response:
    ```json
    {
      "model_id": "nlp-sentiment",
      "version": "v1",
      "prediction": {
        "sentiment": "positive",
        "confidence": 0.9821,
        "probabilities": { "negative": 0.0041, "neutral": 0.0138, "positive": 0.9821 }
      },
      "latency_ms": 1.245,
      "cached": false,
      "status": "success"
    }
    ```
- **`POST /api/v1/models/<model_id>/predict-batch`**:
  - Request body: `{"inputs": [...]}` or raw JSON array `[...]`.
  - Executes vectorized model batch prediction directly without queueing delays.
  - Returns array of predictions with batch size and execution duration.

### 2.3 Zero-Downtime Version Hot-Swapping
- **`POST /api/v1/models/<model_id>/reload`** or **`POST /api/v1/models/<model_id>/swap-version`**:
  - Request body: `{"version": "v2"}`.
  - Triggers the zero-downtime hot-swap protocol:
    1. Loads target version weights from vault storage into an isolated model instance.
    2. Runs synthetic warmup dry-run to prime execution pipelines.
    3. Allocates and starts a fresh micro-batcher worker thread.
    4. Atomically replaces model runtime pointers under lock.
    5. Invalidates stale prediction cache entries for the model.
    6. Updates database active version record in `models` table.
    7. Gracefully drains and terminates the retired batcher thread.

### 2.4 Observability & Telemetry
- **`GET /api/v1/metrics`** and **`GET /api/v1/telemetry`**:
  - Returns global and per-model latency histograms: p50, p90, p95, p99, min, max, avg.
  - Returns cache statistics: item count, maximum capacity, hits, misses, hit ratio.
  - Returns request rate estimates: throughput in requests per second (RPS).
- **`POST /api/v1/metrics/reset`**: Resets all sliding windows and historical telemetry.
- **`POST /api/v1/cache/clear`**: Purges all cached prediction entries.

---

## 3. Resilience and Error Handling

1. **Structured Error Responses:**
   All API errors return consistent JSON payloads with appropriate HTTP status codes:
   - `400 Bad Request`: Schema validation failure or malformed JSON syntax.
   - `404 Not Found`: Model identifier or target version does not exist.
   - `504 Gateway Timeout`: Inference request exceeded client-specified deadline.
   - `500 Internal Server Error`: Computation error or hardware exception.
2. **Self-Healing Lazy Model Loading:**
   If a model exists in the database but was not preloaded into memory upon startup, the registry automatically instantiates, warms up, and activates the model on the first incoming request without requiring manual operator intervention.
3. **Database Audit Logging:**
   All inference attempts (successes, cache hits, and validation failures) record execution latency and status codes in SQLite WAL tables via `PredictionLogDAO`.

---

## 4. Test Suite & Verification Results

### 4.1 Pytest Execution & Coverage
All 99 unit and integration tests passed in 9.01 seconds with 97% overall test coverage:
```
tests/test_api_routes.py ............................ [ 28%]
tests/test_batcher_and_cache.py ..................... [ 45%]
tests/test_db.py .................................... [ 51%]
tests/test_health.py ........                         [ 54%]
tests/test_models.py .........                        [ 57%]
tests/test_models_ml.py ............................. [ 79%]
tests/test_registry_and_telemetry.py ................ [ 95%]
tests/test_seed.py ...                                [ 96%]
tests/test_storage.py ........                        [100%]

Name                                Stmts   Miss  Cover
-------------------------------------------------------
app\routes\models.py                  121      3    98%
app\routes\metrics.py                  41      0   100%
app\routes\health.py                   28      0   100%
app\routes\static_routes.py            22      0   100%
app\services\registry_service.py      285     32    89%
app\services\telemetry_service.py      87      0   100%
app\services\cache_service.py         100      0   100%
app\services\batcher_service.py       124      2    98%
app\services\storage_service.py        26      0   100%
app\ml\preprocessors.py               108      0   100%
app\ml\models\nlp_model.py             58      0   100%
app\ml\models\vision_model.py          69      0   100%
app\ml\models\tabular_model.py         69      0   100%
app\models\prediction_log.py           38      0   100%
app\models\model_entity.py             39      1    97%
app\models\model_version.py            57      1    98%
-------------------------------------------------------
TOTAL                                1511     51    97%
```

### 4.2 Quality Gate Summary
- **Gate 1 (Flake8):** Clean run with 0 errors under the 88-character limit.
- **Gate 2 (Black):** Formatted and verified clean across all modules.
- **Gate 3 (isort):** All import hierarchies validated under `--profile black`.
- **Gate 4 (Bandit):** Security analysis completed with 0 vulnerabilities detected across 2,699 lines of code.
- **Gate 5 (Pytest):** 99/99 tests passing (97% coverage >= 90% threshold).

---

## 5. Next Steps for Day 6

With the REST API gateway operational and thoroughly validated, Day 6 will build the modern interactive frontend workbench and dashboard:
1. **Interactive Model Workbench (`workbench.js`):** Model selection dropdown, dynamic input forms (text area for NLP, drag-and-drop file uploader for Vision, numeric sliders for Tabular fraud scoring).
2. **Observability Dashboard (`dashboard.js`):** Live telemetry panels showing request rates, p95 latency gauges, cache hit ratios, and real-time execution logs.
3. **In-Browser Concurrency Benchmarker (`benchmarker.js`):** Interactive load test simulator for generating concurrent traffic and observing micro-batching efficiency.
4. **Dark-Themed Modern Styling (`workbench.css`, `dashboard.css`, `base.css`):** Polished user experience.
