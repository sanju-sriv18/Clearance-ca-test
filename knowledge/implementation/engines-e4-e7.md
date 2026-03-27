# Engines E4-E7: FTA, Exception, Regulatory, and Document Intelligence -- Implementation Guide

This document covers the implementation details for the four standalone engines
that operate outside the main classification-tariff-compliance pipeline.

---

## Table of Contents

1. [E4: FTA Qualification](#e4-fta-qualification)
2. [E5: Exception Analysis](#e5-exception-analysis)
3. [E6: Regulatory Change Intelligence](#e6-regulatory-change-intelligence)
4. [E7: Document Intelligence](#e7-document-intelligence)

---

## E4: FTA Qualification

**File:** `backend/app/engines/e4_fta/engine.py`

### Class: FTAEngine

```
engine_id = "E4"
engine_name = "FTA Qualification"
```

### USMCA Rules of Origin

Rules are keyed by HS heading (4-digit) or chapter (2-digit) in the `USMCA_RULES`
dict. More specific keys (heading) take precedence over less specific keys
(chapter) at lookup time.

### Three Rule Types

**YARN_FORWARD (Chapters 61-63: Textiles)**

Yarn must be formed in USMCA territory. The tariff shift requirement is a chapter
change from outside chapters 50-63.

```python
"61": {
    "rule": "YARN_FORWARD",
    "rvc_threshold": None,
    "tariff_shift": "Chapter change from outside Ch.50-63",
    "documentation": ["Certificate of origin", "Fiber/yarn source documentation"],
}
```

**RVC (Headings 8703, 8708: Automotive)**

Regional Value Content with strict thresholds under the Net Cost method.

```python
"8703": {
    "rule": "RVC",
    "rvc_threshold": 75.0,
    "rvc_method": "NET_COST",
    "tariff_shift": "Heading change + RVC",
    "documentation": [
        "USMCA certification of origin",
        "RVC calculation worksheet",
        "BOM with country of origin for each component",
        "Labor Value Content documentation",
    ],
}
```

**HYBRID (Heading 8471 Electronics, and others)**

Tariff shift OR Regional Value Content. The importer can qualify through either
path.

```python
"8471": {
    "rule": "HYBRID",
    "rvc_threshold": 50.0,
    "rvc_method": "TRANSACTION_VALUE",
    "tariff_shift": "Subheading change within Ch.84",
    "documentation": ["USMCA certification of origin", "BOM or RVC calculation"],
}
```

### Complete Rule Table

| HS Key  | Rule Type    | RVC Threshold | RVC Method        | Tariff Shift |
|---------|-------------|---------------|-------------------|--------------|
| 61      | YARN_FORWARD | --            | --                | Chapter change from outside 50-63 |
| 62      | YARN_FORWARD | --            | --                | Chapter change from outside 50-63 |
| 8703    | RVC          | 75%           | NET_COST          | Heading change + RVC |
| 8708    | RVC          | 75%           | NET_COST          | Heading change + RVC |
| 8471    | HYBRID       | 50%           | TRANSACTION_VALUE | Subheading change within Ch.84 |
| 8518    | HYBRID       | 45%           | TRANSACTION_VALUE | Subheading change within Ch.85 |
| 8528    | HYBRID       | 50%           | TRANSACTION_VALUE | Subheading change within Ch.85 |
| 69      | TARIFF_SHIFT | --            | --                | Change from any other chapter |
| 73      | HYBRID       | 50%           | TRANSACTION_VALUE | Heading change within Ch.73 |
| 09      | TARIFF_SHIFT | --            | --                | Chapter change |
| 22      | TARIFF_SHIFT | --            | --                | Change from any other chapter |
| 33      | TARIFF_SHIFT | --            | --                | Change to heading 3304 from any other |
| 95      | HYBRID       | 45%           | TRANSACTION_VALUE | Heading change within Ch.95 |
| DEFAULT | HYBRID       | 50%           | TRANSACTION_VALUE | Heading change |

### Rule Lookup Algorithm

`_find_rule(hs_code)`:
1. Strip dots from HS code.
2. Try 4-digit heading key in `USMCA_RULES`.
3. Try 2-digit chapter key.
4. Fall back to `USMCA_RULES["DEFAULT"]`.

### RVC Calculation

When a Bill of Materials (BOM) is provided (`components` list), the engine
calculates actual Regional Value Content:

```python
usmca_value = sum(c["value"] for c in components if c["origin"] in USMCA_MEMBERS)
total_value = declared_value or sum(c["value"] for c in components)
rvc = (usmca_value / total_value) * 100
```

If `rvc >= threshold`, the product QUALIFIES. The result includes the calculated
RVC percentage.

Without a BOM, the engine returns guidance-only assessment describing what
documentation is needed for a definitive determination.

### Duty Savings Estimation

`_estimate_savings(hs_code, declared_value)`:
- Imports `USTaxRegime` from E2 and looks up the MFN rate.
- If MFN rate > 0: savings = declared_value * rate / 100.
- If MFN rate = 0: no savings (already duty-free).

### Output Structure

```python
{
    "eligible": True | False | None,
    "program": "USMCA",
    "program_full_name": "United States-Mexico-Canada Agreement",
    "rule_type": "RVC",
    "rule_description": "75% RVC required under net cost method...",
    "tariff_shift_requirement": "Heading change + RVC",
    "rvc_threshold": 75.0,
    "rvc_method": "NET_COST",
    "qualification_assessment": "RVC calculated at 82.3% (meets 75% threshold)...",
    "documentation_required": [...],
    "savings_estimate": 60.0,
    "savings_description": "Potential duty savings of $60.00 if USMCA...",
    "origin": "MX",
    "destination": "US",
}
```

`eligible` is `None` when the assessment is guidance-only (no BOM provided).

---

## E5: Exception Analysis

**File:** `backend/app/engines/e5_exception/engine.py`

### Class: ExceptionEngine

```
engine_id = "E5"
engine_name = "Exception Analysis"
```

### Constructor

```python
def __init__(
    self,
    llm_service: Any | None = None,
    embedding_service: Any | None = None,
    db_session: Any | None = None,
) -> None
```

Three optional services: LLM for response drafting, embedding service for Qdrant
vector search, and database session.

### Three-Step Pipeline

**Step 1: Search -- `_search_rulings(query, hs_code)`**

Primary path: Qdrant vector search.
```python
if self.embedding is not None:
    results = await self.embedding.search_rulings(query, hs_code=hs_code, limit=5)
```

Fallback: keyword + HS-prefix matching via `_keyword_search()`.

The keyword search algorithm:
1. Iterate `REFERENCE_RULINGS` (10 curated CROSS rulings).
2. Score each ruling:
   - HS code prefix match (first 4 digits): +10 points.
   - Each query word (> 3 chars) found in ruling text: +2 points.
3. Sort by score descending, return top 5.

**Step 2: Draft -- `_generate_response(exception, hs_code, rulings, entry_number)`**

Primary path: LLM-generated response.
- System prompt: "You are a customs compliance specialist. Respond only in valid JSON."
- User prompt includes the exception description, HS code, entry number, and
  formatted ruling summaries.
- Instructs the LLM to cite only provided ruling numbers.
- Output: JSON with `subject`, `body`, `cited_rulings[]`, `recommended_actions[]`.

Fallback: `_template_response()` generates a structured response using the top 3
rulings, with generic recommended actions.

**Step 3: Validate -- Citation Stripping**

After LLM generation, cited rulings are validated against `VALID_RULING_NUMBERS`
(a frozenset of known ruling IDs). Hallucinated ruling numbers are stripped:
```python
validated = [r for r in cited if r in VALID_RULING_NUMBERS]
result["invalid_citations_removed"] = len(cited) - len(validated)
```

### Reference CROSS Rulings

10 curated rulings covering: ceramic mugs (N332456), porcelain tea sets (N318967),
Bluetooth speakers (N325781), laptops (N301234), yoga pants (N315678), building
blocks (N328901), facial moisturizer (N340123), hex bolts (N335567), automobiles
(N342890), coffee beans (N322456).

### Streaming Interface

`stream_analyze()` yields four events:
1. `exception_step` -- "Searching CROSS rulings database..."
2. `exception_step` -- "Found N relevant rulings" (includes rulings list).
3. `exception_step` -- "Generating draft response..."
4. `exception_complete` -- draft response + rulings + validation confirmation.

### Output Structure

```python
{
    "exception_description": "...",
    "relevant_rulings": [...],
    "draft_response": {
        "subject": "Response to Classification Exception -- 6912.00.48",
        "body": "...",
        "cited_rulings": ["N332456"],
        "recommended_actions": ["Submit additional product documentation..."],
    },
    "hs_code": "6912.00.48",
    "entry_number": "...",
    "ruling_citations_validated": True,
}
```

---

## E6: Regulatory Change Intelligence

**File:** `backend/app/engines/e6_regulatory/engine.py`

### Class: RegulatoryEngine

```
engine_id = "E6"
engine_name = "Regulatory Change Intelligence"
```

### Constructor

```python
def __init__(
    self,
    tariff_engine: Any | None = None,
    db_session: Any | None = None,
) -> None
```

The `tariff_engine` parameter is optional; if not provided, E6 lazily instantiates
a `TariffEngine` when scenario modelling is invoked.

### Sub-Capability 6A: Signal Feed

**Method:** `get_signals(category=None, status=None, country=None)`

Filters the `REGULATORY_SIGNALS` reference list by:
- `category`: TARIFF, COMPLIANCE, TRADE_POLICY, FTA.
- `status`: CONFIRMED, PROPOSED, DISCUSSED.
- `country`: ISO code (includes signals with "ALL" in affected countries).

Returns signal list with summary counts (confirmed, proposed, discussed).

### Reference Signals

| ID       | Title | Status | Category | Impact |
|----------|-------|--------|----------|--------|
| SIG-001  | Section 301 Rate Increase on Chinese Electronics (25% to 50%) | CONFIRMED | TARIFF | HIGH |
| SIG-002  | EU Carbon Border Adjustment Mechanism (CBAM) Phase 2 | CONFIRMED | COMPLIANCE | HIGH |
| SIG-003  | US De Minimis Reform -- Section 321 Elimination | CONFIRMED | TRADE_POLICY | HIGH |
| SIG-004  | Proposed IEEPA Rate Reduction for China (145% to 60%) | PROPOSED | TARIFF | HIGH |
| SIG-005  | India Retaliatory Tariffs on US Agricultural Products | PROPOSED | TARIFF | MEDIUM |
| SIG-006  | Brazil Mercosur CET Revision for Electronics | DISCUSSED | TARIFF | MEDIUM |
| SIG-007  | EU Deforestation Regulation (EUDR) Enforcement | CONFIRMED | COMPLIANCE | HIGH |
| SIG-008  | Section 232 Expansion to Semiconductors | DISCUSSED | TARIFF | HIGH |
| SIG-009  | UFLPA Entity List Expansion -- Solar Supply Chain | CONFIRMED | COMPLIANCE | HIGH |
| SIG-010  | USMCA Automotive Rules of Origin Tightening | CONFIRMED | FTA | MEDIUM |

Each signal includes: `id`, `title`, `status`, `description`, `affected_hs_codes`,
`affected_countries`, `effective_date`, `source`, `impact_level`, `category`, and
optional `rate_change` (with `from`, `to`, `program` fields).

### Sub-Capability 6B: Scenario Modelling

**Method:** `model_scenario(signal_id, hs_codes, declared_values, origin_country, destination_country)`

For tariff-type signals with `rate_change` data:
1. Compute current landed cost via E2 for each product.
2. Calculate delta from rate change: `delta_amount = value * (new_rate - old_rate) / 100`.
3. Project after-scenario landed cost.
4. Return per-product comparisons with before/after landed cost, rate delta,
   percentage change.

For compliance-type signals (no `rate_change`):
Return qualitative assessment with `impact: "COMPLIANCE_CHANGE"` and
`action_required: True`.

Output includes `total_cost_delta`, `annualized_impact` (delta * 12), and
a recommendation generated by `_generate_recommendation()`.

### Sub-Capability 6C: Portfolio Impact

**Method:** `analyze_portfolio_impact(signal_id, portfolio)`

1. Iterate portfolio items (each with `hs_code`, `declared_value`, optional
   `origin_country` and `volume`).
2. Match each item's HS prefix against the signal's `affected_hs_codes` list
   (prefix matching, or "ALL" for universal signals).
3. Compute affected vs. total portfolio value (weighted by volume).
4. Calculate `exposure_pct`.
5. Rough cost impact estimate: 25% of affected value as proxy.

### Recommendation Generation

`_generate_recommendation(signal, cost_impact)`:

| Status    | Impact > $100K | Impact > $10K | Otherwise |
|-----------|----------------|---------------|-----------|
| CONFIRMED | URGENT: immediate supply chain review | ACTION REQUIRED: review affected lines | MONITOR: update procedures |
| PROPOSED  | PREPARE: begin contingency planning | -- | -- |
| DISCUSSED | WATCH: monitor for developments | -- | -- |

---

## E7: Document Intelligence

**File:** `backend/app/engines/e7_documents/engine.py`

### Class: DocumentEngine

```
engine_id = "E7"
engine_name = "Document Intelligence"
```

### Architecture

E7 is a thin `BaseEngine` wrapper. The main logic resides in the API routes module
(`app.api.routes.documents`) which serves three HTTP endpoints for the three
sub-capabilities.

### Sub-Capability 7A: Requirements Lookup (Deterministic)

The `execute()` method delegates to `get_document_requirements()` from the routes
module:

```python
from app.api.schemas.documents import DocumentRequirementsRequest
from app.api.routes.documents import get_document_requirements

request_model = DocumentRequirementsRequest(
    hs_code=hs_code,
    origin_country=origin_country,
    destination_country=destination_country,
    compliance_flags=compliance_flags,
)
requirements = await get_document_requirements(request_model)
```

The requirements response is serialized via `model_dump()` and returned in the
`EngineOutput.data["requirements"]` field.

**Input:**
- `hs_code` (required): HTS / HS code of the goods.
- `origin_country` (required): ISO country code for origin.
- `destination_country` (default `"US"`): ISO destination code.
- `compliance_flags` (optional): list of compliance flag strings from E3.

**Output:**
```python
{
    "requirements": [...],       # List of document requirement dicts
    "document_count": 8,         # Total documents identified
    "required_count": 6,         # Mandatory documents
    "conditional_count": 2,      # Conditional documents
}
```

### Sub-Capability 7B: LLM Generation

Accessed via the API routes module (not through `execute()`). Drafts specific
customs documents:
- Commercial invoices.
- Packing lists.
- Certificates of origin.
- Customs declarations.

The LLM generates document content based on shipment parameters and compliance
requirements.

### Sub-Capability 7C: LLM Analysis

Also accessed via the API routes module. Analyses uploaded documents to:
- Extract key fields (dates, values, parties, HS codes).
- Validate completeness against requirements.
- Identify missing or incorrect information.
- Suggest corrections.

### Error Handling

- Missing required parameters (`hs_code`, `origin_country`): returns error status
  immediately.
- Import failures or route-level exceptions: caught broadly, logged, and returned
  as error status.
