---
date: 2026-10-27
project: PulseInfer ML Model Serving API
topic: Day 2 - Vectorized Preprocessors and Domain Inference Models
Tags:
  - "[[ML-Model-Serving]]"
  - "[[Feature-Preprocessing]]"
  - "[[Vectorization]]"
  - "[[Inference-Engines]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 44 - DAY 2

**Core Objective:** Design and implement vectorized data preprocessing pipelines (`TextTokenizer`, `ImageTensorPipeline`, `TabularScaler`) and domain-specific inference models (`NLPClassifier`, `VisionClassifier`, `FraudDetector`) unified under the `BaseModel` abstraction, validated by 38 passing unit tests with 98% total coverage and complete quality gate verification.

---

## 1. The Big Picture and Simple Explanation

A production inference gateway cannot execute machine learning models directly on raw, unformatted network payloads. Clients transmit disparate data representations:
- Plain text sentences or customer feedback reviews for NLP.
- Base64-encoded binary images or data URIs for Computer Vision.
- Heterogeneous JSON key-value dictionaries with varying numerical magnitudes and missing values for Tabular risk scoring.

In Day 2, we engineered the model execution core for **PulseInfer**:
1. **Vectorized Feature Preprocessors:** Specialized transformation pipelines that translate unstructured or semi-structured raw payloads into contiguous, standardized NumPy numerical matrices.
2. **Abstract Model Foundation (`BaseModel`):** A shared contract establishing mandatory single-item (`predict`) and batch (`predict_batch`) inference interfaces, schema introspection, and numerically stable activation functions (Softmax and Sigmoid).
3. **Domain-Specific Inference Models:**
   - **`NLPClassifier`:** Tokenizes raw text strings, computes sentiment term scores, applies class biases, and evaluates multi-class probabilities via Softmax.
   - **`VisionClassifier`:** Decodes base64 strings or byte buffers, applies bilinear interpolation to target spatial dimensions (32x32), normalizes RGB channels, and projects spatial tensor representations into visual class scores.
   - **`FraudDetector`:** Validates tabular records, imputes missing values, computes calibrated linear logits ($z = X \beta + b_0$), and activates probabilities through the logistic sigmoid function to classify transaction risk tiers.

```
+-----------------------------------------------------------------------------+
|                     PulseInfer Inference Architecture                       |
|                                                                             |
|  [Raw Client Payload]                                                       |
|        |                                                                    |
|        +--> NLP (Text String)      --> [TextTokenizer]       --> 1D/2D TF   |
|        |                                                                    |
|        +--> Vision (Base64/Bytes)  --> [ImageTensorPipeline] --> Tensor      |
|        |                                                                    |
|        +--> Tabular (JSON Record)  --> [TabularScaler]       --> Vector      |
|                                                                             |
|                                   |                                         |
|                                   v                                         |
|  [BaseModel Concrete Execution]                                             |
|        |                                                                    |
|        +--> NLPClassifier          --> Logits -> Softmax     --> Prob/Top    |
|        |                                                                    |
|        +--> VisionClassifier       --> Projections -> Softmax--> Scores     |
|        |                                                                    |
|        +--> FraudDetector          --> Logits -> Sigmoid     --> RiskTier   |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Vectorized Feature Preprocessor Pipeline

**TextTokenizer:**
- Normalizes incoming text via case folding and punctuation stripping.
- Splits text into discrete token lists and builds frequency-ranked vocabularies.
- Transforms single text entries or batches into normalized term frequency vectors: $TF(t) = \frac{\text{count}(t)}{\sum_{i} \text{count}(i)}$.
- Gracefully handles unknown or out-of-vocabulary tokens with zero-vector fallbacks.

**ImageTensorPipeline:**
- Decodes inputs across raw PIL images, byte streams, plain base64 strings, and `data:image/png;base64,...` URI schemes.
- Handles malformed or non-image inputs via neutral gray canvas fallback generation.
- Resizes inputs to target spatial dimensions using bilinear interpolation and normalizes pixel values: $X_{\text{norm}} = \frac{X / 255.0 - \mu}{\sigma}$ where default mean $\mu = [0.485, 0.456, 0.406]$ and standard deviation $\sigma = [0.229, 0.224, 0.225]$.
- Supports both flattened 1D feature vectors and channel-first $(C, H, W)$ tensor shapes.

**TabularScaler:**
- Extracts specified feature subsets while preserving column ordering.
- Imputes missing, null, or non-numeric inputs using configured column defaults or population means.
- Computes z-score normalization: $z = \frac{x - \mu}{\sigma_{\text{safe}}}$ with zero-variance safeguards ($\sigma_{\text{safe}} = 1.0$ when $|\sigma| < 10^{-7}$).

### Domain Inference Models with Shared BaseModel Foundation

**BaseModel Contract:**
- `predict(input_data)`: Single inference execution returning standard dictionary schemas.
- `predict_batch(batch_data)`: Vectorized array execution across multiple payload items.
- `get_schema()`: Metadata reflection exposing input and output contractual types.
- `load_from_file(filepath)`: Deserialization directly from JSON artifact paths in the models vault.
- Numerically stable activation functions: $\text{Softmax}(z)_i = \frac{e^{z_i - \max(z)}}{\sum_j e^{z_j - \max(z)}}$ and $\sigma(z) = \frac{1}{1 + e^{-\text{clip}(z, -500, 500)}}$.

**NLPClassifier:** Linear projection scoring from tokenized vocabulary weights, baseline category biases, and top class/confidence/probability distribution outputs.

**VisionClassifier:** Integrates `ImageTensorPipeline` for automated feature extraction, maps flattened or summary-pooled spatial representations through projection weight matrices, and returns top predicted visual category with confidence scores across classes (`landscape`, `document`, `portrait`, `product`).

**FraudDetector:** Calibrated logistic regression computing decision boundary logit $z = b_0 + \sum_{i=1}^{D} w_i x_i$, applying logistic sigmoid activation to generate probability risk score $p \in [0.0, 1.0]$, and categorizing transactions into tiers: `HIGH` ($p \ge 0.75$), `MEDIUM` ($0.40 \le p < 0.75$), and `LOW` ($p < 0.40$).

---

## 3. Verification & Quality Gates Report

The test suite verified preprocessing pipelines and inference model execution across all quality gates:

| Quality Gate | Tool | Target / Threshold | Result | Status |
| :--- | :--- | :--- | :--- | :--- |
| Linting | Flake8 | 0 violations (88-char limit) | 0 warnings / 0 errors | PASSED |
| Code Formatting | Black | PEP 8 / 88 columns | 100% compliant | PASSED |
| Import Sorting | isort | Black profile compatibility | 100% compliant | PASSED |
| Security Scanning | Bandit | Zero medium/high vulnerabilities | 0 issues found | PASSED |
| Test Coverage | Pytest | >= 90% statement coverage | 38/38 passed (98% coverage) | PASSED |

### Key Test Module Coverage
| Module | Statements | Miss | Coverage |
| :--- | :--- | :--- | :--- |
| `app/ml/preprocessors.py` | 108 | 0 | 100% |
| `app/ml/models/nlp_model.py` | 58 | 0 | 100% |
| `app/ml/models/vision_model.py` | 69 | 0 | 100% |
| `app/ml/models/tabular_model.py` | 69 | 0 | 100% |
| `app/ml/models/base.py` | 47 | 3 | 94% |

---

## 4. Key Engineering Insights & Decisions

- **Numerically Stable Softmax via Max-Subtraction:** Subtracting $\max(z)$ from each logit before exponentiation prevents overflow when logit magnitudes exceed $\pm 700$ (the float64 `exp` threshold), ensuring stable probability distributions across all input scales.
- **Gray Canvas Fallback for Malformed Vision Inputs:** Rather than rejecting corrupted or non-decodable image payloads outright, synthesizing a neutral gray canvas preserves batch execution continuity, allowing batch inference to complete with degraded but non-crashing results.
- **Z-Score Safe Division Guard:** Setting $\sigma_{\text{safe}} = 1.0$ when measured standard deviation is below $10^{-7}$ prevents division-by-zero explosions in features with zero variance (e.g., constant-value columns in tabular data).

---

## 5. Next Steps

- Build `DynamicBatcher` with request queuing, adaptive batch timeouts, and maximum batch sizing.
- Implement thread-safe worker pool for compute pipeline execution.
- Construct end-to-end inference service routing from HTTP request through preprocessor and model to structured response.
