# UFLPA Forced Labor Screening — Engine Analysis & Gap Assessment

## 1. Overview

The platform identifies forced labor risk through the `UFLPAScreener` (`app/engines/e3_compliance/uflpa.py`), which implements a **5-factor scoring model** under the **Uyghur Forced Labor Prevention Act** (19 USC §6901 et seq.).

The screener runs as one of three parallel sub-engines inside the E3 Compliance Engine:

```
ComplianceEngine.execute()
    ├── PGAScreener.screen()         ← Partner Government Agency flags
    ├── DPSScreener.screen()         ← Denied party lists
    └── UFLPAScreener.assess()       ← Forced labor risk assessment
```

All three execute concurrently via `asyncio.gather()`.

Legal basis: 19 USC §6901 et seq.; UFLPA Strategy (DHS, June 2022).

---

## 2. The 5 Risk Factors

### Factor 1: China Origin Gate

The first check is a hard gate. If `origin_country != "CN"`, the screener returns immediately with `CLEAR` and score 0. All subsequent factors only fire for China-origin goods.

This reflects UFLPA's rebuttable presumption, which specifically targets goods mined, produced, or manufactured wholly or in part in the Xinjiang Uyghur Autonomous Region (XUAR).

- **Score contribution**: +2 (baseline for any CN-origin product)
- **Severity**: MODERATE

### Factor 2: High-Risk Sector Matching

Matches the HS code (dot-stripped) against prefix patterns for 6 UFLPA enforcement priority sectors.

| Sector ID | HS Prefixes | Description | Risk Factor |
|-----------|------------|-------------|-------------|
| `COTTON_TEXTILES` | 52, 5208–5212, 6104, 6106, 6109, 6110, 6203, 6204 | Cotton and cotton textile products | Xinjiang produces ~85% of China's cotton (~20% of global supply) |
| `POLYSILICON_SOLAR` | 2804, 3818, 8541, 8501 | Polysilicon and solar panel components | Xinjiang produces ~35% of global polysilicon supply |
| `TOMATO_PRODUCTS` | 0702, 2002, 2103 | Tomatoes and tomato-based products | Xinjiang is a major tomato processing region |
| `HUMAN_HAIR` | 6703, 6704 | Human hair products and wigs | CBP has detained multiple hair product shipments linked to Xinjiang |
| `PVC_CHEMICALS` | 3904, 2815 | PVC and caustic soda | Xinjiang Zhongtai Chemical is a UFLPA entity |
| `ELECTRONICS_COMPONENTS` | 8542, 8541 | Semiconductor and electronic components | Supply chain links to forced labor in component manufacturing |

- **Score contribution**: +3 per matched sector
- **Severity**: HIGH
- One match per sector; breaks after first HS prefix hit within each sector

### Factor 3: UFLPA Entity List Matching

Substring match (uppercased) against supplier name and product description. 14 hardcoded entities:

```
HOSHINE SILICON, XPCC, BINGTUAN, XINJIANG JUNGGAR,
LOPTOP COUNTY, ZHONGTAI CHEMICAL, NONGFU SPRING,
HETIAN HAOLIN, KASHGAR HUAFU, XINJIANG PRODUCTION,
AKSU HUAFU, CHANGJI ESQUEL, HEFEI BITLAND, HEFEI MEILING
```

- **Score contribution**: +5
- **Severity**: CRITICAL
- Breaks after first match (one hit is sufficient)

### Factor 4: Xinjiang Region Reference Detection

Checks supplier name and product description (uppercased) for 17 region keywords:

- **Place names**: XINJIANG, URUMQI, KASHGAR, HOTAN, AKSU, TURPAN, HAMI, SHIHEZI, YINING, KORLA, KARAMAY
- **Organizations**: XPCC, BINGTUAN, XUAR, UYGHUR
- **Chinese characters**: 新疆, 维吾尔

- **Score contribution**: +4
- **Severity**: HIGH
- Breaks after first match

### Factor 5: Withhold Release Order (WRO) Matching

Dual match requirement — both HS prefix AND entity name must match for a WRO to trigger.

| WRO | Product | Entity | HS Prefixes |
|-----|---------|--------|------------|
| WRO-2020-01 | Cotton products | XINJIANG PRODUCTION AND CONSTRUCTION CORPS | 52, 6104, 6109 |
| WRO-2021-01 | Silica-based products | HOSHINE SILICON INDUSTRY | 2804, 3818 |
| WRO-2021-02 | Polysilicon and downstream products | XINJIANG DAQO NEW ENERGY | 2804, 3818, 8541 |
| WRO-2022-01 | Hair products | LOPTOP COUNTY | 6703, 6704 |
| WRO-2022-02 | PVC and caustic soda | XINJIANG ZHONGTAI CHEMICAL | 3904, 2815 |

- **Score contribution**: +5
- **Severity**: CRITICAL
- Returns immediately after first WRO match

---

## 3. Scoring and Risk Levels

| Score | Risk Level | Enforcement Priority | Recommendation |
|-------|-----------|---------------------|----------------|
| 0–1 | `CLEAR` | NONE | No UFLPA risk factors identified |
| 2–4 | `LOW` | MONITOR | China origin triggers baseline monitoring. No immediate enforcement concern but maintain records. |
| 5–7 | `MEDIUM` | POSSIBLE_REVIEW | Product in high-risk sector from China. Maintain supply chain traceability documentation. CBP may request additional evidence. |
| **8+** | **HIGH** | **LIKELY_DETENTION** | HIGH RISK — Likely to be detained under UFLPA. Prepare forced labor due diligence documentation. Consider alternative sourcing. |

### Example Score Combinations

- CN origin only → score 2 → LOW
- CN origin + cotton textiles (sector match) → score 5 → MEDIUM
- CN origin + cotton textiles + Xinjiang region reference → score 9 → HIGH
- CN origin + entity match (e.g. "Hoshine Silicon") → score 7 → MEDIUM
- CN origin + sector + entity match → score 10 → HIGH
- CN origin + WRO match → score 7+ → MEDIUM/HIGH
- Non-CN origin → score 0 → CLEAR (immediate return)

---

## 4. Output Schema

The screener returns a dict consumed by the `ComplianceResult` schema:

```python
# Backend: shared/schemas/analysis.py
class UFLPAResult(BaseModel):
    risk_level: ConfidenceLevel   # HIGH | MEDIUM | LOW | CLEAR
    flags: list[str]              # Specific concern flags
    recommendation: str           # Action recommendation

class ComplianceResult(EngineResult):
    engine_id: str = "e3_compliance"
    pga_flags: list[PGAFlag]
    dps_results: list[DPSResult]
    uflpa_risk: UFLPAResult       # ← UFLPA assessment lives here
    overall_risk: ConfidenceLevel
```

The raw screener output includes additional fields (`risk_score`, `factors[]`, `matched_sectors[]`, `enforcement_priority`, `citation`) that are available in the engine output's `data` dict.

### Frontend Type

```typescript
// types.ts
uflpaRisk: {
  level: "HIGH" | "MEDIUM" | "LOW" | "NONE";
  factors: string[];
}
```

### Persistence

Stored in the compliance domain model (`compliance/models.py`) as a `uflpa_risk` JSONB column.

---

## 5. Integration Points

### Where Results Are Used

| Component | Location | How |
|-----------|----------|-----|
| Compliance Engine | `app/engines/e3_compliance/engine.py` | Runs UFLPAScreener.assess() in parallel with PGA + DPS |
| Preclearance Engine | `app/engines/e0_preclearance/` | References UFLPA risk in preclearance assessment |
| Streaming API | `shared/streaming.py` | Emits `compliance_uflpa` SSE event during analysis stream |
| Compliance domain | `domains/compliance/models.py` | Persists `uflpa_risk` JSONB on compliance screening records |
| Shipper Product Detail | `shipper/screens/ShipperProductDetail.tsx` | Surfaces UFLPA_HIGH as a compliance flag |
| Broker Intelligence | `app/services/broker_intelligence.py` | Incorporates UFLPA risk into broker briefing |
| Pipeline Store | `frontend/src/store/pipelineStore.ts` | UFLPA detentions appear in clearance pipeline demo data |
| Simulation actors | `simulation/actors/compliance_engine.py` | Simulation compliance actor includes UFLPA assessment |

### Data Flow

```
Product description + HS code + origin + entity name
    │
    ▼
UFLPAScreener.assess()
    │
    ├── Factor 1: CN gate check
    ├── Factor 2: UFLPA_HIGH_RISK_SECTORS (HS prefix match)
    ├── Factor 3: UFLPA_ENTITIES (substring match)
    ├── Factor 4: UFLPA_HIGH_RISK_REGIONS (substring match)
    └── Factor 5: WRO_FINDINGS (HS + entity dual match)
    │
    ▼
ComplianceEngine.execute() aggregates into ComplianceResult
    │
    ▼
Stored in compliance domain (JSONB) + streamed to frontend (SSE)
```

---

## 6. Ingestion Pipelines (Exist but Not Connected)

### UFLPAEntityPipeline

**Location**: `clearance_platform/shared/ingestion/uflpa_entities.py`

Designed to ingest the DHS UFLPA Entity List from `https://www.dhs.gov/uflpa-entity-list` on a 24-hour schedule (stale after 72 hours). Parses JSON or CSV format with fields:

- `entity_name`, `source_list`, `entity_type`, `country`, `region`
- `sector`, `added_date`, `basis` (legal basis for listing)
- `related_entities` (subsidiaries/affiliated entities)

**Status**: The `upsert()` method **only logs** — does not write to the database or feed into the screener. The 14 entities in `UFLPA_ENTITIES` are hardcoded, not sourced from this pipeline.

### WROPipeline

**Location**: `clearance_platform/shared/ingestion/uflpa_entities.py`

Designed to ingest CBP Withhold Release Order findings.

**Status**: Same as above — `upsert()` logs only.

---

## 7. Gaps and Limitations

### Entity List Coverage

- **14 hardcoded entities** vs the actual DHS UFLPA Entity List, which contains 70+ entities and grows regularly
- The ingestion pipeline exists (`UFLPAEntityPipeline`) but isn't wired to the screener
- No mechanism to update the entity list without a code deployment

### Matching Quality

- **Substring matching only** — `"HOSHINE SILICON"` will not match `"Hoshine Silicon Industry Co., Ltd."` unless the exact substring appears
- No fuzzy matching, phonetic matching, or transliteration handling
- No alias/variant name support (e.g., Chinese company names may appear in multiple romanizations)
- Related entities / subsidiaries are captured in the ingestion pipeline schema but not used in screening

### Supply Chain Depth

- Checks **direct supplier name only**, not sub-tier suppliers
- No supply chain mapping or traceability graph
- UFLPA enforcement increasingly targets downstream products (e.g., solar panels containing Xinjiang polysilicon, garments containing Xinjiang cotton) — the screener only catches direct entity references

### WRO Coverage

- **5 hardcoded WRO findings** — CBP issues new WROs periodically; the list is not automatically updated
- The `WROPipeline` exists but has the same logging-only `upsert()`

### Due Diligence Documentation

- The screener recommends "prepare forced labor due diligence documentation" for HIGH risk
- No workflow to capture or manage due diligence documentation:
  - Supply chain maps
  - Third-party audit reports
  - Traceability evidence (cotton DNA testing, polysilicon isotope analysis, etc.)
  - Importer compliance program documentation
- CBP requires "clear and convincing evidence" to rebut the UFLPA presumption — no support for preparing or organizing this evidence

### Sector Coverage

- 6 sectors covered; DHS enforcement strategy also highlights:
  - Aluminum and steel (emerging concern)
  - Seafood/fishing (ILO forced labor indicators)
  - Lithium and rare earth minerals
- No mechanism to add new sectors without code changes

### Regional Specificity

- All of China is gated the same way (score +2); the screener doesn't differentiate between Xinjiang-origin and other Chinese provinces until Factor 4 (region reference)
- Source location data on the Product model (`source_locations`) has `country_code` but no province/region granularity

---

## 8. Potential Improvements

| Improvement | Impact | Complexity |
|------------|--------|------------|
| Wire `UFLPAEntityPipeline` to screener (DB-backed entity list) | Keeps entity list current with DHS updates | Medium |
| Wire `WROPipeline` to screener | Keeps WRO findings current | Medium |
| Fuzzy/phonetic entity matching (Levenshtein, Soundex, or embedding-based) | Catches entity name variants and transliterations | Medium |
| Sub-tier supplier screening via `related_entities` | Catches subsidiaries and affiliates of listed entities | Low-Medium |
| Province-level origin granularity | Distinguishes Xinjiang-origin from other CN provinces | Low (schema) + Medium (UI) |
| Due diligence document workflow | Helps importers prepare rebuttal evidence packages | High |
| Supply chain traceability integration | Maps multi-tier supply chains to identify Xinjiang exposure | High |
| Dynamic sector configuration | Add/remove sectors without code deployment | Low |
