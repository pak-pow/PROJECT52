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
- **`InferenceCache`**: An in-memory, thread-safe LRU cache using canonical SHA-256 payload fingerprints. If an identical query is received, the gateway returns the cached prediction in sub-millisecond time (< 0.1 ms), completely bypassing the inference pipeline.
- **`DynamicBatcher`**: An asynchronous request aggregator grouping incoming requests into contiguous batches dispatched to `model.predict_batch()`, resolving individual results to each awaiting client thread via synchronization events.

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
|                                        +--> [Store in Cache]               |
|                                        |                                    |
|                                        +--> [Resolve Client Threads]       |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### InferenceCache: Canonical Hashing and LRU Eviction

To guarantee that logically identical payloads yield the exact same cache key regardless of client-side formatting, the cache employs recursive canonicalization:
1. **Dictionary Key Ordering:** All dictionary keys are recursively sorted in lexicographical order: $\text{canonical}(D) = \{ (k, \text{canonical}(v)) \mid k \in \text{sorted}(\text{keys}(D)) \}$.
2. **Float Precision Normalization:** Floating-point scalars are rounded to 8 decimal places to eliminate insignificant floating-point representation discrepancies.
3. **Numpy Array Serialization:** NumPy vectors and matrices are converted into standardized nested lists prior to serialization.
4. **Deterministic Hash Fingerprint:** $\text{Key} = \text{SHA-256}(\text{model\_id} \parallel \text{version} \parallel \text{JSON}_{\text{compact}}(\text{canonical}(P)))$.

The cache enforces an LRU policy via `collections.OrderedDict`: successful retrieval relocates entries to the recency head (`move_to_end`), capacity overflow evicts the oldest item (`popitem(last=False)`), and TTL is enforced lazily on retrieval and proactively via `purge_expired()`.

### DynamicBatcher: Micro-Batching Queue with Adaptive Timeout

The `DynamicBatcher` coordinates concurrent inference threads using a condition variable (`threading.Condition`) over a thread-safe deque:
1. **Enqueue & Signal:** Callers wrap their payload into a `BatchItem` with an un-signaled `threading.Event` and timestamp, append it to `_queue`, and signal the condition.
2. **Adaptive Window Evaluation:** The background worker evaluates the age of the oldest item: $\Delta t = t_{\text{now}} - t_{\text{oldest}}$ and computes $\text{Remaining Wait} = \max(0, \frac{\text{timeout\_ms}}{1000} - \Delta t)$. The worker waits until queue length satisfies $N \ge \text{max\_batch\_size}$ (Capacity Trigger) or remaining wait drops to zero (Timeout Trigger).
3. **Batch Execution & Fan-Out:** The worker extracts $B = \min(N, \text{max\_batch\_size})$ items, executes `predict_fn([item.payload for item in batch])`, assigns outputs to `item.result`, and signals `item.event.set()` to unblock each client thread.

### Resilience Safeguards

- **Non-Blocking In-Flight Queue Draining on Shutdown:** When `stop()` is invoked, any queued items that cannot be processed are rejected immediately with a descriptive `RuntimeError`, preventing client threads from hanging indefinitely.
- **Per-Item Exception Propagation:** If the vectorized model prediction raises an exception (e.g., input dimension mismatch), the exception is captured and propagated to every item in the batch without crashing the background worker.
- **Atomic Reference & Thread Safety:** All cache reads, writes, and evictions are guarded by a re-entrant lock (`RLock`). Batcher queue state transitions are protected by a dedicated mutex.

---

## 3. Verification & Quality Gates Report

The test suite verified cache operations, batcher concurrency, and thread safety across all quality gates:

| Quality Gate | Tool | Target / Threshold | Result | Status |
| :--- | :--- | :--- | :--- | :--- |
| Linting | Flake8 | 0 violations (88-char limit) | 0 warnings / 0 errors | PASSED |
| Code Formatting | Black | PEP 8 / 88 columns | 100% compliant | PASSED |
| Import Sorting | isort | Black profile compatibility | 100% compliant | PASSED |
| Security Scanning | Bandit | Zero medium/high vulnerabilities | 0 issues found | PASSED |
| Test Coverage | Pytest | >= 90% statement coverage | 55/55 passed (99% coverage) | PASSED |

### Key Module Coverage
| Module | Statements | Miss | Coverage |
| :--- | :--- | :--- | :--- |
| `app/services/cache_service.py` | 100 | 0 | 100% |
| `app/services/batcher_service.py` | 124 | 2 | 98% |
| `app/ml/preprocessors.py` | 108 | 0 | 100% |

---

## 4. Key Engineering Insights & Decisions

- **Canonical SHA-256 Hashing vs. Naive String Keys:** Recursive dictionary key sorting and float rounding guarantee that `{"b": 1.0, "a": 2.0}` and `{"a": 2.0, "b": 1.0}` produce identical cache keys, eliminating duplicate model execution for semantically identical payloads.
- **Condition Variable vs. Polling for Batch Coalescing:** Using `threading.Condition.wait(timeout=remaining)` instead of busy-loop polling eliminates wasted CPU cycles during idle periods while still triggering immediate batch dispatch when capacity thresholds are met.
- **LRU Over FIFO Eviction:** Least-recently-used eviction prioritizes retention of hot query patterns (e.g., the same sentiment query submitted repeatedly), maximizing cache hit ratios under realistic production traffic distributions.

---

## 5. Next Steps

- Implement Model Registry Service (`ModelRegistry`) with model lifecycle management (loading, warming up, active, hot-reloading, unloading) and zero-downtime hot-swapping between versions.
- Build Telemetry Engine (`TelemetryEngine`) with sliding-window histograms computing p50, p90, p95, and p99 latencies, cache hit ratios, and throughput tracking.
- Integrate audit logging persistence via `PredictionLogDAO`.
