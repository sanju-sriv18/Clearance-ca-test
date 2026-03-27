# E1: Classification Intelligence -- Implementation Guide

This document describes the implementation details of Engine 1, the Classification
Intelligence engine. E1 classifies products to specific HS (Harmonized System)
codes using a three-phase pipeline with concurrent execution: Phase 1 identifies
the 2-digit HS chapter, then Phase 2 (LLM classification with keyword fallbacks)
and Phase 3 (external prediction via the FedEx HS Code API) run **concurrently**.
Results are merged with multi-signal confidence scoring and HS code validation.

---

## File Locations

```
backend/app/engines/e1_classification/
  __init__.py       # classify() wrapper, _build_llm_service(), _build_fedex_client()
  engine.py         # ClassificationEngine class, CHAPTER_REFERENCE, keyword maps,
                    #   _compute_agreement(), _fetch_external_predictions()
  prompts.py        # 4 prompt templates (2 system + 2 user)
  confidence.py     # ConfidenceLevel enum, compute_confidence()
  validator.py      # HSCodeValidator, ValidationResult, KNOWN_VALID_CODES

backend/clearance_platform/adapters/fedex/
  __init__.py       # Package docstring, future phases listed
  config.py         # FedExAdapterConfig (pydantic-settings, env vars)
  models.py         # FedExPredictRequest, FedExPrediction, ExternalHSPrediction
  hs_api_client.py  # FedExHSApiClient (OAuth2 token mgmt, HTTP, normalization)

backend/clearance_platform/shared/circuit_breaker.py
  # fedex_hs_breaker instance (failure_threshold=3, recovery_timeout=120s)
```

---

## ClassificationEngine Three-Phase Pipeline (Concurrent)

After Phase 1 identifies the chapter, Phase 2 and Phase 3 fan out
concurrently via ``asyncio.gather()``.  The chapter hint from Phase 1 is
passed to FedEx as ``chapter_codes`` so both systems classify within the
same tariff chapter.

```
Phase 1: Chapter ID (sequential -- keyword or LLM)
         │
         ├──► Phase 2: LLM full classify (~2-3 s)    ┐
         │                                            ├─► merge + confidence
         └──► Phase 3: FedEx HS API (~0.5-1 s)        ┘

engine_id = "E1"
engine_name = "Classification Intelligence"
```

### Constructor

```python
def __init__(
    self,
    llm_service: Any = None,
    db_session: Any = None,
    fedex_client: Any = None,
) -> None
```

- `llm_service`: optional LLM client. When `None`, the engine uses keyword
  fallbacks exclusively.
- `db_session`: optional async database session for live HS code validation.
- `fedex_client`: optional FedEx HS API client (e.g.
  `FedExHSApiClient`). When `None`, Phase 3 prediction is skipped.

### Phase 1: Chapter Identification (2-digit)

**Method:** `_identify_chapter(product_description, origin_country)`

Two paths depending on LLM availability:

**Fast path (no LLM):**
1. `_keyword_chapter_detect(description)` is called.
2. Priority 1: scan `_PRODUCT_CODE_MAP` for multi-keyword matches. If all keywords
   in a tuple are found in the description (case-insensitive), derive the chapter
   from the mapped HS code. Returns HIGH confidence.
3. Priority 2: score each chapter in `_KEYWORD_CHAPTER_MAP` by counting keyword
   hits. The chapter with the most hits wins. Confidence: >= 2 hits = HIGH,
   1 hit = MEDIUM, 0 = LOW.

**Slow path (with LLM):**
1. Call `llm.complete()` with `CHAPTER_IDENTIFICATION_SYSTEM` (system) and
   `CHAPTER_IDENTIFICATION_USER` (user, with `{product_description}` and
   `{origin_country}` interpolated).
2. Parse the response as JSON. On parse failure, extract the first JSON object via
   regex `\{.*\}` (DOTALL). If still failing, fall back to keyword detection.
3. Expected JSON output:
   ```json
   {
     "chapter": "85",
     "chapter_title": "Electrical machinery...",
     "section": "Section XVI",
     "reasoning": "...",
     "alternative_chapters": ["84"],
     "confidence": "HIGH"
   }
   ```

### Phase 2: Full Classification (subheading)

**Method:** `_full_classify(product_description, origin_country, chapter_num, chapter_info, retry=False)`

**With LLM:**
1. Build chapter reference text via `_build_chapter_text()`: chapter title, notes,
   and key headings from `CHAPTER_REFERENCE`.
2. Format `FULL_CLASSIFICATION_SYSTEM` with `{chapter_text}` interpolated.
3. If `retry=True`, append an extra instruction urging the model to use valid codes.
4. Call `llm.complete()` with the system prompt and `FULL_CLASSIFICATION_USER`
   (with `{product_description}`, `{origin_country}`, `{chapter_number}`,
   `{chapter_title}`).
5. Parse the JSON response (same regex fallback pattern as Phase 1).
6. Expected JSON output:
   ```json
   {
     "steps": [
       {"step": 1, "action": "Identify product characteristics", "reasoning": "..."},
       {"step": 2, "action": "Apply GRI Rule 1", "reasoning": "..."},
       ...
     ],
     "hs_code": "8518.21",
     "hs_description": "Single loudspeakers, mounted in enclosures",
     "confidence": "HIGH",
     "confidence_reasoning": "...",
     "alternative_codes": [{"code": "8519.81", "reason": "..."}]
   }
   ```

**Without LLM:**
`_keyword_classify(description, chapter)` scans `_PRODUCT_CODE_MAP` for matching
keywords and returns a result with synthetic GRI reasoning steps. Falls back to
a generic `{chapter}00.00` code with LOW confidence.

### Phase 3: External Prediction (FedEx HS API) -- runs concurrently with Phase 2

**Method:** `_fetch_external_predictions(product_description, dest_country, chapter_codes=None)`

Phase 3 runs **concurrently** with Phase 2 via ``asyncio.gather()``.  It
receives the ``chapter_codes=[chapter_num]`` hint from Phase 1, which
constrains FedEx's ML model to the identified chapter for more accurate
and specific predictions.

This phase is optional and gated by `FEDEX_HS_API_ENABLED` in `app/config.py`
(default `False`). When disabled or when no prediction client is configured,
Phase 3 returns an empty list and classification behaves identically to the
original two-phase pipeline.

**Execution flow:**
1. `_fetch_external_predictions()` calls `self.fedex.predict(description,
   dest_country, chapter_codes=chapter_codes)`.
2. The `FedExHSApiClient.predict()` method wraps the call in the
   `fedex_hs_breaker` circuit breaker.
3. Inside the breaker, `_predict_inner()` ensures a valid OAuth2 token
   (checking Redis cache first, acquiring a fresh one if needed), then sends
   `POST /v2/hscode` to the FedEx HS API.
4. The raw FedEx predictions are normalised to `ExternalHSPrediction` models
   with dotted HS code format.
5. Any exception in Phase 3 is caught, logged as a warning, and an empty list
   is returned -- external prediction never blocks the classification pipeline.

**Concurrency note:** Because ``_fetch_external_predictions`` catches all exceptions
internally and returns ``[]``, it is safe to pass to ``asyncio.gather()``
without ``return_exceptions=True``.  A FedEx failure never cancels Phase 2.

**Agreement scoring:**

`_compute_agreement(hs_code, external_predictions)` compares the LLM's code
against the top-ranked FedEx prediction (digits only, dots stripped):

| Agreement Level | Condition                  |
|:----------------|:---------------------------|
| `"exact"`       | Full code match            |
| `"heading"`     | First 4 digits match       |
| `"chapter"`     | First 2 digits match       |
| `"none"`        | No match or no external data|

**Specificity upgrade (heading agreement):**

When agreement is `"heading"` and FedEx's code has more digits than the LLM's
(e.g. LLM returns HS6 `6912.00`, FedEx returns HS8 `6912.00.48`), the engine
calls `_maybe_upgrade_specificity()` to adopt the more specific code:

1. Verify FedEx code is strictly longer (more digits).
2. Run FedEx code through `HSCodeValidator.validate()` -- must pass format check.
3. If valid, replace the output code with FedEx's subheading.
4. Append a "Subheading refinement" reasoning step to the LLM chain, preserving
   full auditability.
5. Set `agreement = "exact"` (since the adopted code now matches FedEx).
6. Set `specificity_upgraded = True` and `original_llm_code` in the output.

This is safe because heading-level agreement means both systems concur on the
4-digit heading.  FedEx merely provides deeper subheading specificity, which
determines the actual duty rate in the HTSUS.

**Disagreement handling (chapter or no match):**

When agreement is `"none"` or `"chapter"`, the engine:

1. Logs a structured `classification_disagreement` warning with both codes,
   FedEx score, agreement level, and truncated product description.
2. Defaults to the LLM code (because it has auditable GRI reasoning).
3. Surfaces FedEx predictions in `alternative_codes` as a "second opinion".
4. Over time, these logs build a training dataset for a future data-driven
   arbiter.

**Alternative code merging:**

When agreement is not `"exact"` (after any upgrade) and external predictions
exist, the top 3 FedEx predictions are appended to `alternative_codes` with
source attribution:
```json
{"code": "6912.00.50", "reason": "FedEx HS API (rank 2, score 0.820)"}
```

**Output external prediction block:**

When external predictions are available, the engine output includes an
`external_prediction` field plus specificity metadata:
```json
{
  "hs_code": "6912.00.48",
  "specificity_upgraded": true,
  "original_llm_code": "6912.00",
  "external_prediction": {
    "source": "fedex_hs_api",
    "agreement": "exact",
    "predictions": [
      {"hs_code": "6912.00.48", "score": 0.95, "rank": 1},
      {"hs_code": "6912.00.50", "score": 0.82, "rank": 2}
    ]
  }
}
```

---

## The Decision: Who Wins?

After both experts return their answers, the system compares them. Here are the four possible outcomes:

| Scenario | Example | What Happens | Which Code Is Used | Confidence Impact |
|:---|:---|:---|:---|:---|
| **A. Exact match** ✅ | AI says `6912.00.48`, FedEx says `6912.00.48` | Both agree — strong signal the code is correct | Either (same answer) | Boosted to HIGH |
| **B. Same product type, FedEx more specific** ⬆️ | AI says `6912.00`, FedEx says `6912.00.48` | System adopts FedEx's more detailed code while keeping the AI's reasoning on file | FedEx's code | Boosted (heading agreement) |
| **C. They disagree** ⚠️ | AI says `8518.21` (speakers), FedEx says `9503.00` (toys) | AI wins because it has auditable legal reasoning; FedEx's answer is shown as a "second opinion" | AI's code | No boost; logged for review |
| **D. FedEx unavailable** 🔇 | FedEx service is down or turned off | System uses the AI's answer alone — nothing breaks | AI's code | No change |

### What "specificity upgrade" means (Scenario B)

Both experts agree on the *type* of product (the first four digits of the code match). FedEx simply provides a more precise sub-category — and that extra detail determines the actual tax rate. So the system takes FedEx's more specific code while keeping the AI's step-by-step reasoning as the official record.

---

## LLM Prompt Structure

### Chapter Identification

**CHAPTER_IDENTIFICATION_SYSTEM:**
- Role: expert customs classification specialist.
- Task: identify the 2-digit HS chapter.
- Reasoning chain: (1) what the product IS, (2) which HS Section, (3) which Chapter.
- Output: JSON with chapter, title, section, reasoning, alternatives, confidence.

**CHAPTER_IDENTIFICATION_USER:**
- Template: "Classify this product to its HS chapter: Product: {product_description}, Origin Country: {origin_country}"

### Full Classification

**FULL_CLASSIFICATION_SYSTEM:**
- Role: expert classification specialist.
- Input: chapter headings, notes, and key headings via `{chapter_text}`.
- GRI rules 1, 2(a), 2(b), 3, and 6 are explicitly listed.
- Requires step-by-step reasoning referencing specific headings.
- Output: JSON with steps array, hs_code, description, confidence, alternatives.
- Uses double-brace `{{` escaping for JSON template within Python string.

**FULL_CLASSIFICATION_USER:**
- Template: product description, origin country, identified chapter number and title.

---

## Streaming Implementation

`stream_classify()` yields typed SSE event dictionaries.  Phase 2 (LLM) and
Phase 3 (FedEx) are dispatched concurrently via ``asyncio.gather()`` after
Phase 1 completes.  The FedEx result typically arrives before the LLM
finishes, so from the user's perspective the prediction step appears
immediately after the reasoning steps with no additional wait.

### Event Sequence

1. **Step 0** (`classification_step`): "Analyzing product description" --
   acknowledges receipt.
2. **Phase 1 result** (`classification_step`): "Chapter identified" -- chapter
   number, title, and reasoning.
3. **Step 2** (`classification_step`): "Applying General Rules of Interpretation
   (+ external prediction in parallel)" -- announces the concurrent fan-out.
4. **Reasoning steps** (`classification_step`): one event per LLM reasoning step
   (step numbers 3+).  These are yielded after both concurrent tasks resolve.
5. **External prediction** (conditional, step 50): if FedEx HS API returns
   predictions, yields an event with the top prediction code, score, and
   agreement level.
6. **Validation check** (conditional, step 90): if the code fails validation
   entirely (neither `known_code` nor `known_heading`), the engine retries
   classification once and reports the re-classification attempt.
7. **Final result** (`classification_complete`): HS code, description, confidence,
   reasoning steps, chapter, alternatives.

### Error Events

`classification_error` is yielded if Phase 1 raises an exception, or if
*both* Phase 2 and Phase 3 fail (since they share the ``asyncio.gather``
call).  An isolated Phase 3 failure returns ``[]`` and does not trigger an
error event.  The generator returns immediately after yielding the error.

---

## Confidence Scoring Algorithm

**File:** `confidence.py`

### ConfidenceLevel Enum

```python
class ConfidenceLevel(str, Enum):
    HIGH = "HIGH"
    MEDIUM = "MEDIUM"
    LOW = "LOW"
```

### compute_confidence() Signal Scoring

```python
def compute_confidence(
    *,
    num_alternatives: int,                    # Number of alternative HS codes LLM proposed
    description_specificity: str,              # "HIGH", "MEDIUM", or "LOW"
    code_validated: bool,                      # Exact match in KNOWN_VALID_CODES
    heading_validated: bool,                   # Prefix match at heading level
    llm_confidence: str,                       # LLM's self-assessed confidence
    external_prediction_agreement: str = "none",  # Agreement with FedEx HS API
) -> ConfidenceLevel:
```

**Scoring table:**

| Signal                          | Points |
|:--------------------------------|:-------|
| LLM says HIGH                   | +3     |
| LLM says MEDIUM                 | +2     |
| LLM says LOW                    | +1     |
| 0 alternatives                  | +2     |
| 1 alternative                   | +1     |
| 3+ alternatives                 | −1     |
| Code validated (exact)          | +2     |
| Heading validated               | +1     |
| Neither validated               | −2     |
| Description HIGH                | +1     |
| Description LOW                 | −1     |
| External agreement: exact       | +3     |
| External agreement: heading     | +2     |
| External agreement: chapter     | +1     |
| External agreement: none        | +0     |

**Tier mapping:** score >= 6 = HIGH, score >= 3 = MEDIUM, otherwise LOW.

The external agreement signal is purely additive -- when no external data exists
the score is identical to the original formula. When FedEx agrees at the exact
code level (+3), it often pushes a MEDIUM classification to HIGH.

### Description Specificity Determination

Applied in `execute()` and `stream_classify()`:
```python
word_count = len(product_description.split())
description_specificity = "HIGH" if word_count > 5 else "MEDIUM" if word_count > 2 else "LOW"
```

---

## HS Code Validation Rules

**File:** `validator.py`

### HSCodeValidator

**Format checking:**
- Dotted pattern: `^\d{4}\.\d{2}(?:\.\d{2,4})?$` (e.g., `6912.00.48`).
- Bare pattern: `^\d{6,10}$` (e.g., `69120048`).

**Normalization:**
- Strips spaces, inserts dots at positions 4 and 6: `69120048` becomes `6912.00.48`.

**Validation levels (in order):**
1. **Format check** -- if the code does not match either pattern, return
   `valid_format=False` with `confidence_adjustment=-2`.
2. **Exact match** -- if `normalized` is in `_valid_codes`, return
   `known_code=True, confidence_adjustment=0`.
3. **Heading-level prefix** -- match first 7 characters against any known code
   prefix. Returns `known_heading=True, confidence_adjustment=0`.
4. **Chapter-level prefix** -- match first 4 characters. Returns
   `known_heading=True, confidence_adjustment=-1`.
5. **Unknown code** -- logs a warning, returns `confidence_adjustment=-2`.

### KNOWN_VALID_CODES

A `frozenset` of approximately 50 validated HS codes covering the golden-path demo
products: ceramics (6912), audio (8518), computers (8471), apparel (6104, 6109,
6110), automobiles (8703), cosmetics (3304), toys (9503), steel fasteners (7318),
auto parts (8708), televisions (8528), coffee (0901), wine (2204), cables (8544),
batteries (8507), plastics (3926), bags (4202).

In production, this set is augmented by loading from the `htsus_headings` database
table via `load_codes()`.

---

## CHAPTER_REFERENCE Keyword Matching

### CHAPTER_REFERENCE Dict

`CHAPTER_REFERENCE` in `engine.py` contains structured data for approximately 30
commonly encountered HS chapters. Each entry includes:
- `title`: chapter title.
- `notes`: classification guidance notes.
- `key_headings`: dict mapping 4-digit heading codes to descriptions.

Used by both the keyword fallback (chapter detection) and the LLM path (provides
context in the full classification system prompt).

### _KEYWORD_CHAPTER_MAP

Maps 2-digit chapter strings to lists of keywords. Example:
```python
"85": ["speaker", "headphone", "microphone", "bluetooth", "wireless", ...]
```

Scoring: count keyword hits in the lowercased description. Highest-scoring chapter
wins.

### _PRODUCT_CODE_MAP

List of `(keywords_tuple, result_dict)` entries for direct product-to-code mapping.
Each entry maps a tuple of required keywords to a specific HS code and description.
Example:
```python
(("bluetooth", "speaker"), {"code": "8518.21", "desc": "Single loudspeakers, mounted in enclosures"})
```

All keywords in the tuple must appear in the description for a match.

---

## How to Modify Classification Logic

### Adding a New Product Mapping

1. Add a tuple to `_PRODUCT_CODE_MAP` in `engine.py`:
   ```python
   (("new", "product", "keywords"), {"code": "XXXX.XX", "desc": "Description"}),
   ```

2. Add the HS code to `KNOWN_VALID_CODES` in `validator.py`:
   ```python
   "XXXX.XX",
   "XXXX.XX.XX",
   ```

3. If the chapter is new, add keywords to `_KEYWORD_CHAPTER_MAP` and chapter data
   to `CHAPTER_REFERENCE`.

### Adding a New Chapter

1. Add an entry to `CHAPTER_REFERENCE` with `title`, `notes`, and `key_headings`.
2. Add keywords to `_KEYWORD_CHAPTER_MAP` for keyword-based detection.
3. Add relevant product mappings to `_PRODUCT_CODE_MAP` as needed.

### Modifying Confidence Thresholds

Edit the scoring constants and tier boundaries in `compute_confidence()` in
`confidence.py`. The current thresholds (>= 6 for HIGH, >= 3 for MEDIUM) are
intentionally conservative.

### Extending Validation

- Add codes to `KNOWN_VALID_CODES` for immediate recognition.
- Implement `validate_against_db()` for live database validation (currently a
  TODO stub awaiting the `htsus_headings` table query).

---

## FedEx HS API Adapter

The adapter lives in `clearance_platform/adapters/fedex/` -- outside the core
`app/` package, following the "adapter, not modification" architecture described
in `docs/fedex-integration-architecture.md`.

### Configuration

All FedEx-specific config is in `FedExAdapterConfig` (pydantic-settings), loaded
from environment variables. The core `app/config.py` carries only a minimal
feature flag (`FEDEX_HS_API_ENABLED: bool = True`) -- it never sees FedEx
credentials.

| Variable                       | Purpose                   | Default          |
|:-------------------------------|:--------------------------|:-----------------|
| `FEDEX_HS_API_ENABLED`         | Feature flag (app/config)  | `True`          |
| `FEDEX_HS_API_BASE_URL`        | API base URL              | `""`             |
| `FEDEX_HS_API_TOKEN_URL`       | OAuth2 token endpoint     | `""`             |
| `CLIP_API_CLIENT_ID`           | OAuth2 client ID          | `""`             |
| `CLIP_API_CLIENT_SECRET`       | OAuth2 client secret      | `""`             |
| `FEDEX_HS_API_SCOPE`           | OAuth2 scope              | `"Custom_Scope"` |
| `FEDEX_HS_API_TIMEOUT_SECONDS` | Per-request timeout       | `5.0`            |

### Pydantic Models

- `FedExPredictRequest`: request body for `POST /v2/hscode` (item_description,
  dest_country_code, chapter_codes, level, date).
- `FedExPrediction`: single prediction from the API (rank, hscode, score,
  tariffDescriptions aliased to tariff_descriptions).
- `FedExErrorResponse`: error envelope (errorMessage).
- `ExternalHSPrediction`: platform-internal model consumed by the engine
  (rank, hs_code in dotted format, score, tariff_description, source).

### FedExHSApiClient

**Constructor:**
```python
def __init__(self, config: FedExAdapterConfig, *, redis: Any | None = None) -> None
```

- `config`: adapter configuration with URLs and credentials.
- `redis`: optional Redis client for OAuth2 token caching.

**HTTP pattern:** Each request opens a fresh `httpx.AsyncClient` via
`async with` -- no persistent connections.

**OAuth2 token management:**
1. `_ensure_token()` checks Redis for a cached token (`fedex_hs_api:token` key).
2. If no cached token exists, `_acquire_token()` calls the token URL with
   `client_credentials` grant.
3. The new token is cached in Redis with `ex=expires_in - 120` (120-second safety
   margin, minimum 60s).
4. No in-memory token state is persisted between requests -- Redis is the sole
   source of truth. If Redis is down, a fresh token is acquired every request.
5. No `asyncio.Lock` is used -- redundant token acquisitions are harmless since
   `client_credentials` grants are idempotent.

**Circuit breaker:** All predict calls are wrapped in `fedex_hs_breaker`
(failure_threshold=3, recovery_timeout=120s). When the breaker is open,
`CircuitOpenError` is raised, which `_fetch_external_predictions()` catches and logs.

**Response normalization:** FedEx returns bare-digit codes (`"69120048"`). The
adapter normalizes them to dotted format via `_normalise_hs_code()`:
- `"69120048"` → `"6912.00.48"`
- `"691200"` → `"6912.00"`
- `"6912.00.48"` → `"6912.00.48"` (no-op)

### Wiring: classify() → engine

The shared factory lives in `clearance_platform/adapters/fedex/__init__.py`:

```python
from clearance_platform.adapters.fedex import build_fedex_client

client = build_fedex_client(redis=redis)
```

Both the `classify()` wrapper in `__init__.py` and the resolution `lookup()`
route delegate to this factory. The client is injected into
`ClassificationEngine(fedex_client=client)`, keeping the engine fully
testable with no import-time coupling to the adapter.
