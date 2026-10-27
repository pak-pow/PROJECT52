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
|        +---> NLP (Text String)      ---> [TextTokenizer]       ---> 1D/2D TF|
|        |                                                                    |
|        +---> Vision (Base64/Bytes)  ---> [ImageTensorPipeline] ---> Tensor  |
|        |                                                                    |
|        +---> Tabular (JSON Record)  ---> [TabularScaler]       ---> Vector  |
|                                                                             |
|                                   |                                         |
|                                   v                                         |
|  [BaseModel Concrete Execution]                                             |
|        |                                                                    |
|        +---> NLPClassifier          ---> Logits -> Softmax     ---> Prob/Top|
|        |                                                                    |
|        +---> VisionClassifier       ---> Projections -> Softmax---> Scores  |
|        |                                                                    |
|        +---> FraudDetector          ---> Logits -> Sigmoid     ---> RiskTier|
+-----------------------------------------------------------------------------+
```

---

## 2. Preprocessor Pipeline Architecture

### 2.1 TextTokenizer
- Normalizes incoming text via case folding and punctuation stripping.
- Splits text into discrete token lists and builds frequency-ranked vocabularies.
- Transforms single text entries or batches into normalized term frequency vectors:
  $$TF(t) = \frac{\text{count}(t)}{\sum_{i} \text{count}(i)}$$
- Gracefully handles unknown or out-of-vocabulary tokens with zero-vector fallbacks.

### 2.2 ImageTensorPipeline
- Decodes inputs across raw PIL images, byte streams, plain base64 strings, and `data:image/png;base64,...` URI schemes.
- Handles malformed or non-image inputs via neutral gray canvas fallback generation.
- Resizes inputs to target spatial dimensions using bilinear interpolation.
- Normalizes pixel values into standardized channel distributions:
  $$X_{\text{norm}} = \frac{X / 255.0 - \mu}{\sigma}$$
  where default mean $\mu = [0.485, 0.456, 0.406]$ and standard deviation $\sigma = [0.229, 0.224, 0.225]$.
- Supports both flattened 1D feature vectors and channel-first $(C, H, W)$ tensor shapes.

### 2.3 TabularScaler
- Extracts specified feature subsets while preserving column ordering.
- Imputes missing, null, or non-numeric inputs using configured column defaults or population means.
- Computes z-score normalization across continuous features:
  $$z = \frac{x - \mu}{\sigma_{\text{safe}}}$$
  with zero-variance safeguards ($\sigma_{\text{safe}} = 1.0$ when $|\sigma| < 10^{-7}$).

---

## 3. Domain Inference Models

### 3.1 BaseModel Foundation
All domain models inherit from `BaseModel`, which enforces:
- `predict(input_data)`: Single inference execution returning standard dictionary schemas.
- `predict_batch(batch_data)`: Vectorized array execution across multiple payload items.
- `get_schema()`: Metadata reflection exposing input and output contractual types.
- `load_from_file(filepath)`: Deserialization directly from JSON artifact paths in the models vault.
- Numerically stable mathematical activation functions:
  $$\text{Softmax}(z)_i = \frac{e^{z_i - \max(z)}}{\sum_j e^{z_j - \max(z)}}$$
  $$\sigma(z) = \frac{1}{1 + e^{-\text{clip}(z, -500, 500)}}$$

### 3.2 NLPClassifier
- Implements linear projection scoring from tokenized vocabulary weights.
- Applies baseline category biases (`bias = [0.1, 0.5, 0.1]`).
- Maps positive sentiment scores to class 2, negative scores to class 0, and neutral scores to class 1.
- Computes top class name, confidence float, and full class probability distributions.

### 3.3 VisionClassifier
- Integrates `ImageTensorPipeline` for automated feature extraction.
- Maps flattened or summary-pooled spatial representations through projection weight matrices.
- Returns top predicted visual category, prediction confidence, and probability scores across classes (`landscape`, `document`, `portrait`, `product`).

### 3.4 FraudDetector
- Implements calibrated logistic regression for financial transaction records.
- Computes decision boundary logit $z$:
  $$z = b_0 + \sum_{i=1}^{D} w_i x_i$$
- Applies logistic sigmoid activation to generate probability risk score $p \in [0.0, 1.0]$.
- Categorizes transactions into operational tiers:
  - `HIGH`: $p \ge 0.75$
  - `MEDIUM`: $0.40 \le p < 0.75$
  - `LOW`: $p < 0.40$
- Compares risk score against configurable decision threshold ($0.50$) to evaluate `is_flagged`.

---

## 4. Quality Gates and Test Results

### 4.1 Pytest Execution & Coverage
All 38 unit tests passed in 1.07 seconds with 98% overall test coverage:
```
tests/test_db.py ......................... [ 15%]
tests/test_health.py ........              [ 23%]
tests/test_models.py .........             [ 31%]
tests/test_models_ml.py .................  [ 89%]
tests/test_seed.py ...                     [ 92%]
tests/test_storage.py ........             [100%]

Name                              Stmts   Miss  Cover
-----------------------------------------------------
app\ml\__init__.py                    3      0   100%
app\ml\models\__init__.py             5      0   100%
app\ml\models\base.py                47      3    94%
app\ml\models\nlp_model.py           58      0   100%
app\ml\models\tabular_model.py       69      0   100%
app\ml\models\vision_model.py        69      0   100%
app\ml\preprocessors.py             108      0   100%
app\models\model_entity.py           39      1    97%
app\models\model_version.py          57      1    98%
app\models\prediction_log.py         38      0   100%
-----------------------------------------------------
TOTAL                               721     12    98%
```

### 4.2 Quality Gate Summary
- **Gate 1 (Flake8):** Clean run with 0 errors across all 88-character rules.
- **Gate 2 (Black):** Formatted and verified clean across all modules.
- **Gate 3 (isort):** All import hierarchies validated under `--profile black`.
- **Gate 4 (Bandit):** Security analysis completed with 0 vulnerabilities detected across 1,354 lines of code.
- **Gate 5 (Pytest):** 38/38 tests passing (98% coverage >= 90% threshold).

---

## 5. Next Steps for Day 3

With preprocessors and inference models verified, Day 3 will establish the inference execution gateway:
1. **Dynamic Micro-Batcher (`DynamicBatcher`):** Request queuing with adaptive batch timeouts and maximum batch sizing.
2. **Worker Thread Pool:** Thread-safe execution avoiding GIL contention on compute pipelines.
3. **Inference Service Integration:** End-to-end routing from HTTP request to preprocessor, model inference, and structured response.
