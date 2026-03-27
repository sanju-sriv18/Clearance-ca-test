# Regulatory Intelligence Automation — Design

**Date:** 2026-02-25
**Branch:** feature/rag-index
**Status:** Approved

---

## Overview

Automate ingestion of regulatory signals from Tier 1 and Tier 2 coverage countries, connect them to the real shipment portfolio for genuine what-if analysis, and close the extraction confidence gap through a human review layer in the platform surface.

**Tier 1:** United States, China, Mexico, Germany, Japan
**Tier 2:** Taiwan, South Korea, Vietnam, India, Thailand

---

## Problem Statement

E6 regulatory intelligence today has three disconnected layers:

1. **Signals are static** — hardcoded in `REGULATORY_SIGNALS` or manually seeded. No automated ingestion from Federal Register, USTR, MOFCOM, or any other live source.
2. **What-if is frontend-only** — `RegulatoryIntel.tsx` uses a `DEMO_SIGNALS` constant and a `SIGNAL_RATE_IMPACTS` dict of hardcoded multipliers. The backend E6B scenario engine exists and is correct but is never called. The real shipment portfolio (`shipment.shipments`, 210K+ records) is invisible to the analysis.
3. **Non-US signal extraction is imprecise** — automated LLM extraction of rate changes from Bing search results reaches ~80% confidence. The last 20% (exact rates, precise HS codes, confirmed effective dates) requires a human.

---

## Signal Lifecycle

```
DRAFT → REVIEWED → CONFIRMED → [ACCEPTED — future phase]
         ↑
    human editor in platform
```

| State | Description | What-if mode |
|---|---|---|
| `DRAFT` | Auto-ingested, LLM-extracted, unreviewed | Estimate (uses extracted rate with "estimate" flag) |
| `REVIEWED` | Analyst has edited one or more fields | Estimate → Real as fields are confirmed |
| `CONFIRMED` | Analyst signed off, all fields trusted | Real (E2-calculated, against live portfolio) |
| `ACCEPTED` | Actively applied to shipments with date bounds | Future phase |
| `ARCHIVED` | Superseded or expired | N/A |

DRAFT signals are visible in the platform and broker intelligence feeds immediately with a "Pending review" badge. CONFIRMED signals display normally.

---

## Architecture

### 1. Ingestion Pipeline

Two scheduled jobs sharing a single LLM extraction step, both running every 6 hours via APScheduler.

#### FederalRegisterPipeline (US)

Polls the Federal Register REST API filtered by trade-relevant agencies:
- USTR, CBP, Commerce (BIS, ITA), DHS (FLETF), OFAC, ITC, FDA, CPSC, FWS

Query parameters: `conditions[type][]=Rule`, `conditions[type][]=Notice`, `conditions[agencies][]=...`, `conditions[publication_date][gte]=<last_run>`.

The API returns structured metadata directly: FR citation, document number, agency, dates, abstract, and full text URL. LLM extraction focuses on rate_change and affected HS codes from the notice body. High confidence expected for US signals.

#### BingRegulatorySearchPipeline (Tier 1+2 ex-US)

Uses the existing `WebSearchService` (Azure AI Foundry + Bing grounding) with one query per jurisdiction per run:

| Country | Query pattern |
|---|---|
| China | `"China MOFCOM tariff customs regulation 2026"` |
| Mexico | `"Mexico SAT SHCP arancel importacion 2026"` |
| Germany/EU | `"EU Official Journal trade regulation import 2026"` |
| Japan | `"Japan MOF METI customs tariff notification 2026"` |
| Taiwan | `"Taiwan customs administration tariff 2026"` |
| South Korea | `"Korea customs service tariff regulation 2026"` |
| Vietnam | `"Vietnam general department customs tariff 2026"` |
| India | `"India CBIC DGFT customs notification 2026"` |
| Thailand | `"Thailand customs department tariff 2026"` |

Results are passed to the LLM extraction step. Lower confidence — these signals are always created as DRAFT.

#### LLM Extraction Step (shared)

Both pipelines pass raw text to a single extraction function that outputs a structured signal with per-field confidence scores:

```json
{
  "title": "Section 301 Rate Increase on Chinese Electronics",
  "status": "CONFIRMED",
  "description": "...",
  "affected_hs_codes": ["8541", "8542"],
  "affected_countries": ["CN"],
  "effective_date": "2026-03-01",
  "source": "USTR Federal Register Notice FR-2025-28456",
  "source_url": "https://www.federalregister.gov/d/...",
  "rate_change": {
    "from": 25.0,
    "to": 50.0,
    "program": "Section 301"
  },
  "confidence": {
    "rate_change": "HIGH",
    "affected_hs_codes": "MEDIUM",
    "effective_date": "HIGH",
    "affected_countries": "HIGH"
  }
}
```

Confidence levels: `HIGH` (directly stated in source), `MEDIUM` (inferred), `LOW` (not found, defaulted).

#### Deduplication

Before insert: compute `source_url` + title similarity hash. If a match exists:
- Status escalation (PROPOSED → CONFIRMED) → update existing record
- Rate or date change → update and add a changelog entry
- No change → skip

New signals are inserted as DRAFT.

---

### 2. DB Schema Changes

Add to `regulatory_signals` table:

```sql
ALTER TABLE regulatory_signals
  ADD COLUMN ingestion_source    VARCHAR(32),   -- 'federal_register' | 'bing_search' | 'manual'
  ADD COLUMN raw_source_text     TEXT,          -- original notice text / Bing snippet
  ADD COLUMN field_confidence    JSONB,         -- per-field confidence scores
  ADD COLUMN reviewed_by         VARCHAR(255),  -- analyst who confirmed
  ADD COLUMN reviewed_at         TIMESTAMPTZ,
  ADD COLUMN changelog           JSONB DEFAULT '[]';  -- [{ts, field, old, new, by}]
```

Update status enum to include `REVIEWED`, `ARCHIVED`.

---

### 3. Platform Signal Editor

New **Signal Review** panel in the platform Regulatory Intelligence screen.

**Layout:**
- Left panel: raw source content (Federal Register notice or Bing search result), with link to original source URL
- Right panel: editable form showing all extracted fields

**Field editing:**
- Fields with `confidence: LOW` → amber highlight, cursor focus on open
- Fields with `confidence: HIGH` → lock icon, one click to unlock for editing
- HS code field → tag input with typeahead against the HTSUS index
- Rate change → structured from/to/program subfields

**Actions:**
- **Save Draft** — saves edits, stays DRAFT
- **Confirm Signal** — moves to CONFIRMED, records `reviewed_by` + `reviewed_at`, unlocks real E6B what-if
- **Archive** — marks signal as superseded

**Signal feed indicators:**
- DRAFT: amber "Pending review" badge
- CONFIRMED: normal display
- Both states visible in platform and broker surfaces immediately

---

### 4. E6B Portfolio Wiring

Replace the current per-HS-code calculation against caller-supplied values with a real portfolio query:

**`model_scenario()` new flow:**

1. Load signal from DB
2. Query `shipment.shipments`:
   ```sql
   WHERE (codes->>'hs_code' LIKE '8541%' OR codes->>'hs_code' LIKE '8542%')
     AND origin = ANY('{CN}')
     AND status NOT IN ('delivered', 'archived')
   ```
3. Deduplicate by unique (hs_code_4digit, origin) combinations → run E2 once per combination
4. Aggregate:
   - `affected_shipment_count`
   - `total_affected_declared_value`
   - `total_duty_before` / `total_duty_after` / `total_delta`
   - `per_shipment_sample` (up to 100 entries for the UI list)
5. Flag result as `estimate: true` if signal is DRAFT, `estimate: false` if CONFIRMED

---

### 5. Frontend Wiring

**`RegulatoryIntel.tsx` (platform) and `RegulatoryIntelligence.tsx` (broker):**

- Replace `DEMO_SIGNALS` constant → `GET /api/v1/regulatory-signals` on mount, with filters for status and jurisdiction
- Replace `calculateRealScenario()` frontend function → `POST /api/v1/regulatory-signals/{id}/scenario`
- `ScenarioComparison` renders API response (real shipment count, real duty delta)
- Add "estimate" visual indicator when signal is DRAFT (amber border, footnote: "Based on unconfirmed rate data — confirm signal for precise calculation")
- Add Signal Review panel to platform surface (new route: `/platform/regulatory/review/:id`)

---

## Data Flow

```
Federal Register API ──┐
                        ├─→ LLM Extraction ─→ regulatory_signals (DRAFT)
Bing (9 jurisdictions) ─┘         │
                                   ↓
                          Platform Signal Editor
                          (human reviews, edits,
                           confirms fields)
                                   │
                                   ↓ CONFIRMED
                          E6B model_scenario()
                          queries shipment.shipments
                          calls E2 per HS/origin combo
                                   │
                                   ↓
                          RegulatoryIntel.tsx
                          ScenarioComparison
                          (real portfolio impact)
```

---

## New Files

| File | Purpose |
|---|---|
| `clearance_platform/shared/ingestion/regulatory_signals.py` | `FederalRegisterPipeline` + `BingRegulatorySearchPipeline` + shared LLM extraction |
| `alembic/versions/028_regulatory_signals_review_fields.py` | Schema migration |
| `app/api/routes/regulatory.py` | Update scenario endpoint to query real portfolio |
| `frontend/src/surfaces/platform/screens/SignalReview.tsx` | Signal editor panel |
| `frontend/src/surfaces/platform/screens/RegulatoryIntel.tsx` | Wire to API, remove DEMO_SIGNALS |
| `frontend/src/surfaces/broker/screens/RegulatoryIntelligence.tsx` | Wire to API |

---

## Modified Files

| File | Change |
|---|---|
| `app/engines/e6_regulatory/engine.py` | `model_scenario()` queries `shipment.shipments` instead of accepting declared_values |
| `clearance_platform/shared/ingestion/__init__.py` | Register new pipelines |
| `clearance_platform/platform_service/main.py` | Add APScheduler jobs for both pipelines |
| `app/api/schemas/regulatory.py` | Add `ingestion_source`, `field_confidence`, `is_estimate` fields |

---

## Out of Scope

- ACCEPTED state and application of signals to shipments (future phase)
- Per-shipment proactive alerting (e.g. "shipment CLR-001 now affected by new signal")
- Non-English source translation (Bing grounding handles this implicitly via synthesis)
- EUR-Lex or MOFCOM primary source adapters (replaceable Bing path in future)
