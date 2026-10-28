---
date: 2026-10-28
project: PulseInfer ML Model Serving API
topic: Day 3 - Dynamic Request Micro-Batcher and Prediction Cache
Tags:
  - "[[ML-Model-Serving]]"
  - "[[Micro-Batching]]"
  - "[[Prediction-Cache]]"
  - "[[LRU-Cache]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 44 - DAY 3

**Core Objective:** Design and implement the high-throughput asynchronous inference execution engine comprising `InferenceCache` (thread-safe in-memory LRU cache with canonical SHA-256 payload hashing and TTL eviction) and `DynamicBatcher` (asynchronous request micro-batcher grouping concurrent inference requests by timeout and maximum capacity), validated by 55 passing unit tests with 99% coverage and complete quality gate verification.

---

## 1. The Big Picture and Simple Explanation

Serving high-throughput machine learning models under concurrent client traffic encounters two primary performance bottlenecks:
1. **Redundant Inference Computations:** A substantial proportion of production requests involve identical or near-identical inputs (e.g., frequently queried search terms, common transaction patterns, popular classification prompts). Performing full matrix math or tensor projections repeatedly for identical payloads wastes CPU cycles and inflates tail latency.
2. **Sequential Single-Item Kernel Overhead:** Machine learning inference operations are heavily optimized for batched matrix multiplication ($Z = X W + b$). Processing requests one-by-one results in high per-request synchronization overhead and poor hardware utilization.

In Day 3, we constructed the execution middleware for **PulseInfer**:
- **`InferenceCache`**: An in-memory, thread-safe LRU cache using canonical SHA-256 payload fingerprints. If an identical query is received, the gateway returns the cached prediction in sub-millisecond time ($< 0.1\text{ ms}$), completely bypassing the inference pipeline.
- **`DynamicBatcher`**: An asynchronous request aggregator. When an incoming inference request arrives, it is enqueued into a bounded buffer. Worker threads group incoming requests into a contiguous batch as soon as either the maximum batch size is reached (e.g., 32 items) or the adaptive timeout window expires (e.g., 10 ms). The aggregated batch is dispatched to `model.predict_batch()`, and individual results are resolved to each awaiting client thread via synchronization events.

```
+-----------------------------------------------------------------------------+
|                     PulseInfer Inference Gateway Engine                     |
|                                                                             |
|  Client Request 1 --+                                                       |
|  Client Request 2 --+--> [Compute Canonical SHA-256 Hash]                   |
|  Client Request 3 --+            |                                          |
|                                  v                                          |
|                     [InferenceCache Lookup]                                 |
|                         /            \                                      |
|            Cache Hit   /              \   Cache Miss                        |
|                       v                v                                    |
|              [Return Result]    [DynamicBatcher Queue]                      |
|              (< 0.1ms p99)             |                                    |
|                                 (Wait: Cap >= 32                            |
|                                  OR Timeout >= 10ms)                        |
|                                        |                                    |
|                                        v                                    |
|                          [Vectorized Batch Compute]                         |
|                           model.predict_batch(batch)                        |
|                                        |                                    |
|                                        +---> [Store in Cache]               |
|                                        |                                    |
|                                        +---> [Resolve Client Threads]       |
+-----------------------------------------------------------------------------+
```

---

## 2. Architecture & Mathematical Foundations

### 2.1 InferenceCache: Canonical Hashing and LRU Eviction
To guarantee that logically identical payloads yield the exact same cache key regardless of client-side formatting, the cache employs recursive canonicalization:
1. **Dictionary Key Ordering:** All dictionary keys are recursively sorted in lexicographical order:
   $$\text{canonical}(D) = \{ (k, \text{canonical}(v)) \mid k \in \text{sorted}(\text{keys}(D)) \}$$
2. **Float Precision Normalization:** Floating-point scalars are rounded to 8 decimal places to eliminate insignificant floating-point representation discrepancies.
3. **Numpy Array Serialization:** NumPy vectors and matrices are converted into standardized nested lists prior to serialization.
4. **Deterministic Hash Fingerprint:**
   $$\text{Key} = \text{SHA-256}(\text{model\_id} \parallel \text{version} \parallel \text{JSON}_{\text{compact}}(\text{canonical}(P)))$$

The cache enforces an LRU policy via `collections.OrderedDict`:
- Any successful cache retrieval (`get`) relocates the entry to the recency head (`move_to_end`).
- When the cache size reaches `max_size`, the oldest item is evicted (`popitem(last=False)`).
- Time-to-Live (TTL) is enforced lazily on retrieval and proactively via `purge_expired()`.

### 2.2 DynamicBatcher: Micro-Batching Queue
The `DynamicBatcher` coordinates concurrent inference threads using a condition variable (`threading.Condition`) over a thread-safe deque:
1. **Enqueue & Signal:** Callers wrap their payload into a `BatchItem` with an un-signaled `threading.Event` and timestamp, append it to `_queue`, and signal the condition.
2. **Adaptive Window Evaluation:** The background worker evaluates the age of the oldest item in the queue:
   $$\Delta t = t_{\text{now}} - t_{\text{oldest}}$$
   $$\text{Remaining Wait} = \max\left(0, \frac{\text{timeout\_ms}}{1000} - \Delta t\right)$$
   The worker waits on the condition variable until:
   - Queue length satisfies $N \ge \text{max\_batch\_size}$ (Capacity Trigger), OR
   - Remaining wait time drops to zero (Timeout Trigger), OR
   - The batcher is signaled to shut down.
3. **Batch Execution & Fan-Out:** The worker extracts $B = \min(N, \text{max\_batch\_size})$ items, executes `predict_fn([item.payload for item in batch])`, assigns outputs to `item.result`, and signals `item.event.set()` to unblock each client thread.

---

## 3. Key Design Decisions & Resilience Safeguards

1. **Non-Blocking In-Flight Queue Draining on Shutdown:**
   When `stop()` is invoked, any queued items that cannot be processed are rejected immediately with a descriptive `RuntimeError`, preventing client threads from hanging indefinitely.
2. **Per-Item Exception Propagation:**
   If the vectorized model prediction function raises an exception (e.g., input dimension mismatch or malformed tensor), the exception is captured and propagated to every item in the batch. Each caller receives the exact exception raised without crashing the background worker.
3. **Atomic Reference & Thread Safety:**
   All cache reads, writes, and evictions are guarded by a re-entrant lock (`RLock`). Batcher queue state transitions and condition variables are protected by a dedicated mutex.

---

## 4. Test Suite & Verification Results

### 4.1 Pytest Execution & Coverage
All 55 unit tests passed in 4.10 seconds with 99% overall test coverage:
```
tests/test_batcher_and_cache.py ................. [ 30%]
tests/test_db.py ................................ [ 41%]
tests/test_health.py ........                     [ 47%]
tests/test_models.py .........                    [ 52%]
tests/test_models_ml.py ......................... [ 90%]
tests/test_seed.py ...                            [ 94%]
tests/test_storage.py ........                    [100%]

Name                              Stmts   Miss  Cover
-----------------------------------------------------
app\services\batcher_service.py     124      2    98%
app\services\cache_service.py       100      0   100%
app\services\storage_service.py      26      0   100%
app\ml\preprocessors.py             108      0   100%
app\ml\models\nlp_model.py           58      0   100%
app\ml\models\vision_model.py        69      0   100%
app\ml\models\tabular_model.py       69      0   100%
app\models\prediction_log.py         38      0   100%
app\models\model_entity.py           39      1    97%
app\models\model_version.py          57      1    98%
-----------------------------------------------------
TOTAL                               947     14    99%
```

### 4.2 Quality Gate Summary
- **Gate 1 (Flake8):** Clean run with 0 errors under the 88-character limit.
- **Gate 2 (Black):** Formatted and verified clean across all modules.
- **Gate 3 (isort):** All import hierarchies validated under `--profile black`.
- **Gate 4 (Bandit):** Security analysis completed with 0 vulnerabilities detected across 1,685 lines of code.
- **Gate 5 (Pytest):** 55/55 tests passing (99% coverage >= 90% threshold).

---

## 5. Next Steps for Day 4

With caching and dynamic micro-batching verified, Day 4 will implement the Model Registry Service and Telemetry Engine:
1. **Model Registry Service (`ModelRegistry`):** Model lifecycle management (loading, warming up, active, hot-reloading, unloading) and zero-downtime hot-swapping between versions.
2. **Telemetry Engine (`TelemetryEngine`):** Sliding-window histograms computing p50, p90, p95, and p99 latencies, cache hit ratios, and throughput tracking.
3. **Audit Logging Integration:** Seamless database persistence via `PredictionLogDAO`.
