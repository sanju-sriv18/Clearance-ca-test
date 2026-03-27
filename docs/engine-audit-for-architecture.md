# Intelligence Engine Audit for Target Architecture

> Audit date: 2026-02-27 (updated) | Original: 2026-02-09

## Purpose

This document provides a comprehensive audit of all intelligence/analytics capabilities in the current codebase, covering:
1. Formal engines (`engines/`)
2. Intelligence services (`services/`)
3. Business logic embedded in simulation actors (`simulation/actors/`)
4. API-layer orchestration and analytics endpoints

The goal is to inform the analytics layer design in the target event-driven architecture.

---

## 1. Formal Engine Inventory (engines/)

The codebase has **8 engines** (E0-E7) inheriting from `BaseEngine` with a standardized `EngineOutput` envelope (status, data, processing_time_ms, errors, warnings, metadata).

### Engine Summary Table

| ID | Name | Purpose | LLM | Qdrant | Deterministic Fallback | Streaming |
|----|------|---------|-----|--------|----------------------|-----------|
| E0 | Pre-Clearance | Agentic tool-calling loop for booking screening | Yes | No | Yes (deterministic pipeline) | No |
| E1 | Classification | Three-step HS code classification (enrichment → data gathering → GRI reasoning) | Yes | No | Yes (keyword + product map) | Yes (SSE) |
| E2 | Tariff & Landed Cost | Jurisdiction-specific duty/tax/fee computation | No | No | N/A (fully deterministic) | No |
| E3 | Compliance Screening | PGA + DPS + UFLPA orchestrated concurrently | No | No | N/A (fully deterministic) | No |
| E4 | FTA Qualification | USMCA rules-of-origin + RVC calculation | No | No | N/A (fully deterministic) | No |
| E5 | Exception Analysis | CROSS rulings search + LLM response drafting | Yes | Yes | Yes (keyword search + template) | Yes (SSE) |
| E6 | Regulatory Intelligence | Signal feed, scenario modeling, portfolio impact | No | No | N/A (fully deterministic) | No |
| E7 | Document Intelligence | Document requirements lookup (delegates to routes) | No | No | N/A (fully deterministic) | No |

### Detailed Engine Profiles

#### E0 — Pre-Clearance Intelligence
- **File**: `engines/e0_preclearance/engine.py`
- **Inputs**: entity_name, origin_country, destination_country, hs_code, product_description, declared_value, transport_mode
- **Outputs**: `{clearance_decision: CLEAR|FLAG|HOLD, events: [...], communication: str}`
- **Trigger**: API route `/api/preclearance`, pipeline orchestration, simulation preclearance actor
- **Mechanism**: Claude tool-calling loop (max 10 iterations) with 5 tools: screen_entity, validate_hs_code, check_value_reasonableness, identify_required_documents, assess_origin_risk
- **State**: Event timeline with tool execution details
- **Notes**: Generates regulatory-cited hold letters; deterministic fallback runs all tools sequentially without LLM

#### E1 — Classification Intelligence
- **File**: `engines/e1_classification/engine.py`
- **Inputs**: product_description, origin_country
- **Outputs**: `{hs_code, subheading, chapter, confidence_score (0-100), confidence_level: LOW|MEDIUM|HIGH, reasoning, external_predictions, relevant_rulings, enrichment_profile}`
- **Trigger**: `/api/classify/stream` (SSE), `/api/classify` (JSON), analysis pipeline
- **Mechanism**: Three-step pipeline:
  1. **Enrichment + Chapter ID** — web-search-powered agentic call enriches vague descriptions into structured `ProductProfile` (material, construction, intended use, tariff attributes). Identifies 2-digit chapter.
  2. **Concurrent Data Gathering (4 parallel)** — RAG chapter reference, FedEx HS API prediction, CROSS rulings semantic search, classification guidance search
  3. **Final Classification** — LLM applies General Rules of Interpretation (GRI) with all gathered evidence
  4. **Merge & Score** — reconciles LLM classification with external predictions; agreement boosts confidence
- **Sub-modules**: `enrichment.py` (ProductProfile), `confidence.py` (scoring), `chapter99.py` (Chapter 99 overrides), `validator.py` (HS format validation), `prompts.py`
- **Notes**: Two interfaces — `execute()` (batch) and `stream_classify()` (SSE iterator)

#### E2 — Global Tariff & Landed Cost
- **File**: `engines/e2_tariff/engine.py`
- **Inputs**: hs_code, origin_country, destination_country, declared_value, currency, entry_type
- **Outputs**: `{landed_cost, line_items[], effective_rate, mfn_rate, applicable_programs, warnings, citations}`
- **Trigger**: Direct API `/api/tariff`, E6 scenario modeling, analysis pipeline
- **Mechanism**: Dispatches to 6 `TaxRegimeEngine` subclasses (US, EU-27, CN, BR, IN, GB). US adds Section 301/232, AD/CVD, MPF, HMF. Brazil uses cascading IPI/ICMS. All regimes return legal citations.
- **State**: Regime instance cache; hardcoded MFN rate tables per jurisdiction
- **Sub-modules**: `tax_regime.py`, `landed_cost.py`, `regimes/{us,eu,cn,br,india,uk}.py`

#### E3 — Compliance Screening
- **File**: `engines/e3_compliance/engine.py`
- **Inputs**: hs_code, origin_country, entity_name, product_description, destination_country
- **Outputs**: `{overall_status: HOLD|REVIEW|PGA_REQUIRED|CLEAR, pga_flags[], dps_result{matches, risk_level}, uflpa_risk{risk_level, risk_score, factors, matched_sectors}}`
- **Trigger**: Analysis pipeline, standalone `/api/compliance`
- **Mechanism**: Runs 3 sub-engines concurrently via `asyncio.gather(return_exceptions=True)`:
  - **PGA Screener** (`pga.py`): HS prefix → 20+ agencies (FDA, EPA, USDA, CPSC, NHTSA, etc.)
  - **DPS Screener** (`dps.py`): 3-stage fuzzy match (exact → token-sort → partial-ratio via rapidfuzz) against 14 US federal restricted party lists plus 2 international lists. Lists are loaded from the `compliance.restricted_parties` table (populated by the CSL ingestion pipeline from trade.gov) with seed data fallback. **Lists (16 total):** OFAC SDN, BIS Entity List, BIS Denied Persons, BIS Unverified, BIS Military End-User, AECA Debarred (State/DDTC), ISN Nonproliferation (State), OFAC Foreign Sanctions Evaders, OFAC Sectoral Sanctions, OFAC CAPTA, OFAC NS-MBS, OFAC NS-CMIC, OFAC PLC, UFLPA Entity List (DHS), EU Consolidated Sanctions, UK OFSI. Jurisdiction-aware list selection.
  - **UFLPA Screener** (`uflpa.py`): 4-factor scoring (origin +2, sector +3, entity +5, region +4) across 6 high-risk sectors (cotton, polysilicon, tomato, hair, PVC, electronics)
- **State**: DPS in-memory index of restricted parties with aliases

#### E4 — FTA Qualification
- **File**: `engines/e4_fta/engine.py`
- **Inputs**: hs_code, origin_country, destination_country, declared_value, components (optional BOM)
- **Outputs**: `{eligible: bool|null, program, rule_type: TARIFF_SHIFT|RVC|HYBRID, rvc_threshold, qualification_assessment, documentation_required, savings_estimate}`
- **Trigger**: `/api/fta`, analysis pipeline
- **Mechanism**: USMCA-only (US/CA/MX corridor). 15 product-specific rules + DEFAULT. RVC calculation from BOM when provided. Savings estimate via E2 MFN rate lookup.
- **State**: None
- **Limitation**: Only USMCA; no EU FTAs, RCEP, CPTPP, etc.

#### E5 — Exception Analysis
- **File**: `engines/e5_exception/engine.py`
- **Inputs**: exception_description, hs_code (optional), entry_number (optional)
- **Outputs**: `{relevant_rulings[], draft_response{subject, body, cited_rulings, recommended_actions, invalid_citations_removed}, ruling_citations_validated}`
- **Trigger**: `/api/resolution`, SSE streaming
- **Mechanism**: Qdrant semantic search for CROSS rulings (with keyword fallback) → LLM drafts response letter → citation validation strips hallucinated ruling numbers
- **State**: 10 reference rulings hardcoded; valid ruling number set
- **Notes**: Only engine using Qdrant

#### E6 — Regulatory Change Intelligence
- **File**: `engines/e6_regulatory/engine.py`
- **Inputs**: Varies by sub-capability (signal filters, signal_id + HS codes, portfolio list)
- **Outputs**:
  - **6A Signal Feed**: `{signals[], total, confirmed, proposed, discussed}`
  - **6B Scenario Model**: `{comparisons[{before, after, delta}], total_cost_delta, annualized_impact, recommendation}`
  - **6C Portfolio Impact**: `{affected_products, affected_value, exposure_pct, recommendation}`
- **Trigger**: `/api/regulatory/signals|scenario|portfolio`
- **Mechanism**: 10 reference signals (Section 301 increase, EU CBAM, Section 321 elimination, IEEPA reduction, India retaliation, Brazil CET, EUDR, Section 232 expansion, UFLPA expansion, USMCA RVC tightening). Scenario modeling calls E2 for before/after comparison.
- **State**: Attempts DB load, falls back to hardcoded signals

#### E7 — Document Intelligence
- **File**: `engines/e7_documents/engine.py`
- **Inputs**: hs_code, origin_country, destination_country, compliance_flags
- **Outputs**: `{requirements[{document, required, reason, citation}], document_count, required_count, conditional_count}`
- **Trigger**: `/api/documents`, pipeline orchestration
- **Mechanism**: Thin wrapper delegating to `app.api.routes.documents.get_document_requirements()`. Universal docs (3461, 7501, CI, PL) + conditional (ISF, bond, PGA filings, USMCA CoO).
- **State**: None

---

## 2. Intelligence Services (services/)

### Service Summary Table

| Service | Type | LLM | Qdrant | Redis | DB | Scheduling | Key Capability |
|---------|------|-----|--------|-------|----|------------|----------------|
| Regulatory Monitor | Intelligence | Yes | No | Yes | Yes | APScheduler | Real-time regulatory feed monitoring + signal extraction |
| Analysis Watchdog | Intelligence | No | No | Yes | Yes | APScheduler | Staleness detection + auto-reanalysis (E1/E2/E3) |
| Broker Intelligence | Intelligence | No | No | No | No | On-demand | Work prioritization, complexity scoring, financial impact |
| Shipment Analysis | Orchestrator | No | No | No | Yes | On-demand | E1→E2→E3 pipeline orchestrator |
| Description Quality | Analytics | Optional | No | No | No | On-demand | Pre-classification quality gate with risk assessment |
| Dashboard | Analytics | No | No | Optional | Yes | On-demand | Company KPI aggregation |
| LLM Service | Infrastructure | Yes | No | No | No | On-demand | Claude/OpenAI with auto-fallback |
| Embedding Service | Infrastructure | No | Yes | No | No | Admin | Qdrant CROSS rulings search |
| Database Service | Infrastructure | No | No | No | Yes | Startup | Async/sync session management |
| Cache Service | Infrastructure | No | No | Yes | No | On-demand | Redis with fail-open semantics |
| Currency Service | Infrastructure | No | No | No | Optional | On-demand | Multi-currency conversion |
| Jurisdiction Service | Infrastructure | No | No | No | No | Static | Country→jurisdiction mapping |
| CSL Ingestion Pipeline | Integration | No | No | No | Yes | APScheduler (6h) | Consolidated Screening List fetch + parse + DB upsert (14 federal lists) |
| Feed Parsers | Integration | No | No | No | No | On-call | Federal Register, CBP CSMS, USTR, Google News |

### Key Intelligence Services Detail

#### Regulatory Monitor (`regulatory_monitor.py`)
- **Trigger**: APScheduler background job (configurable interval)
- **Pipeline**: Fetch 4 feeds → LLM batch extraction (5 articles/call) → 3-tier dedup (URL → hash → semantic LLM) → store `RegulatorySignal` in DB
- **Produces**: Structured signals with title, status, affected HS codes, countries, impact level, category, rate changes
- **Coordination**: Redis distributed lock prevents multi-worker conflicts

#### Analysis Watchdog (`analysis_watchdog.py`)
- **Trigger**: APScheduler background job
- **Pipeline**: Find stale shipments (updated_at > analyzed_at) → batch E1/E2/E3 reanalysis → update JSONB
- **Coordination**: Redis per-shipment locks (300s TTL)
- **Produces**: Updated shipment.analysis, shipment.codes, shipment.financials

#### CSL Ingestion Pipeline (`clearance_platform/shared/ingestion/csl.py`)
- **Trigger**: APScheduler background job (every 6 hours, configurable via `CSL_INGESTION_INTERVAL_HOURS`)
- **Pipeline**: Fetch Consolidated Screening List from `api.trade.gov/consolidated_screening_list/v2/search` → parse JSON → normalize to `RestrictedParty` records → upsert into `compliance.restricted_parties` (ON CONFLICT by `source_list_id, list_source`) → delete stale records → emit `ScreeningListUpdatedEvent`
- **Lists ingested (14 US federal)**: OFAC SDN, BIS Entity/Denied/Unverified/Military End-User, AECA Debarred, ISN Nonproliferation, OFAC FSE/Sectoral/CAPTA/NS-MBS/NS-CMIC/PLC, UFLPA Entity List
- **Data flow**: trade.gov → CSLPipeline.parse() → CSLPipeline.upsert() → PostgreSQL → DPSScreener loads on first screen() call
- **Configuration**: `CSL_INGESTION_ENABLED` (default: true), `CSL_INGESTION_INTERVAL_HOURS` (default: 6.0)
- **Produces**: Up-to-date denied party data for E3 Compliance Screening

#### Broker Intelligence (`broker_intelligence.py`)
- **Trigger**: Synchronous from broker dashboard/queue endpoints
- **Pure computation** — no external service calls
- **Capabilities**:
  - Time estimation for 30+ action types
  - Complexity scoring (0-1) based on shipment attributes
  - Financial impact (storage + duty exposure)
  - Hold resolution paths (step-by-step with timelines)
  - Deadline pressure (exponential decay)
  - Dependency chain analysis
  - Work sequencing: `deadline_pressure*3 + unblocks*2 + value_norm*1.5 + speed_bonus*0.5`
  - Daily capacity estimation
- **Sub-component**: `AIPreworkManager` — cached CF-28 drafts/doc requests via Redis (1h TTL)

#### Description Quality (`description_quality.py`)
- **Trigger**: `/api/description-quality` (rate-limited 10/min)
- **Outputs**: quality level (HIGH/MEDIUM/LOW), score (0-1), detected/missing attributes, follow-up questions, risk assessment (misclassification, duty overpayment, CBP inquiry)
- **Fallback**: Word count + keyword heuristics when LLM unavailable

---

## 3. Business Intelligence Embedded in Simulation Actors

The simulation layer (`simulation/actors/`) contains **20 actor files** with substantial business intelligence that is currently trapped inside simulation orchestration. This section catalogs the intelligence that should be extracted into platform services.

### Actor Intelligence Summary Table

| Actor | Business Intelligence Embedded | Priority to Extract |
|-------|-------------------------------|-------------------|
| **customs.py** | STP eligibility, risk scoring, manual review decision tree, entity screening, AD/CVD case logic, HS reclassification, queue backlog management | CRITICAL |
| **cbp_authority.py** | Entry risk score calculation (origin + value + HS chapter + ADD/CVD), response probability adjustment, CF-28/CF-29 templates, exam type selection | CRITICAL |
| **compliance_engine.py** | Classification confidence, tariff/MPF calculation, duty program allocation, compliance status assessment | CRITICAL |
| **broker_sim.py** | Broker specialization matching, priority classification (high-value, origin-risk, mode-urgency), port-of-entry mapping | HIGH |
| **pga.py** | PGA agency determination, FDA prior notice, APHIS ISPM-15, TTB permit requirements, agency-specific approval rates | HIGH |
| **financial.py** | MPF/HMF calculation, bond type selection, payment method, exam fee determination, broker fee tiering | HIGH |
| **cage.py** | Facility assignment, storage rate lookup, GO deadline management (15-day fixed), GO warning thresholds, dwell/cost calculation, exam scheduling | MEDIUM |
| **resolution.py** | Hold type profiles (review days + resolve rates), resolution decision logic, transit hold resolution | MEDIUM |
| **documents.py** | Document discrepancy detection (invoice qty/value, description, weight, origin cert, USMCA), CF-28 trigger logic | MEDIUM |
| **consolidator.py** | Consolidation grouping (mode + origin + destination + carrier), master number generation (MAWB/MBL/manifest) | MEDIUM |
| **isf.py** | ISF late filing detection (24h window, $5k penalty), data mismatch types, amendment workflow | MEDIUM |
| **terminal.py** | Berth delay logic, chassis shortage modeling, port congestion factors | LOW |
| **shipper.py** | Corridor selection weights, tariff calculation (duplicated from customs.py), Poisson arrival process, multi-product order creation | LOW |
| **carrier.py** | Transit duration by carrier, intermediate event generation (delegates to external) | LOW |
| **exceptions.py** | Exception type probabilities (damage/shortage/overage/misdeclared), AI description generation, product category detection | LOW |
| **disruption.py** | Disruption generation (delegates to `DisruptionGenerator`), port selection logic | LOW |
| **shipper_response.py** | Response timing by channel/urgency, document quality distribution, response generation templates | LOW |
| **preclearance.py** | Hold type derivation from agent results (adapter pattern — correct design) | LOW |

### Critical Extractions Required

**1. Customs Risk Scoring** (customs.py + cbp_authority.py)
- Currently: Implicit in cascading probability checks; cbp_authority has explicit `_calculate_risk_score()` with origin risk (+0.15 for CN/VN/IN/BD/PK), value thresholds, HS chapter flags, ADD/CVD flag
- Target: First-class `CustomsRiskScoringEngine` producing explainable scores (0-1) based on origin, value, HS code, entity status, importer history, PGA jurisdiction count

**2. Tariff Calculation** (customs.py + shipper.py — duplicated)
- Currently: Both actors independently compute duty using `US_CHAPTER_RATES` + `SECTION_301_RATES` from reference_data
- Target: Single `TariffCalculationService` consumed by both simulation and platform (already partially exists in E2, but actors don't use it)

**3. PGA Compliance Orchestration** (pga.py)
- Currently: Full FDA/EPA/CPSC/APHIS/TTB logic with keyword-based product detection, filing requirements, review timing, approval rates
- Target: `PGAComplianceOrchestrator` with agency-specific services, proper product classification integration, risk-based approval

**4. Fee Calculation** (financial.py)
- Currently: MPF (rate × value, capped), HMF (ocean only), bond type selection (random), exam fees (parsed from events), broker fee tiering
- Target: `FeeCalculationService` with configurable rates from regulatory database

---

## 4. API-Layer Orchestration & Analytics Endpoints

### Pipeline Orchestration

| Endpoint | Pattern | Engines Used |
|----------|---------|-------------|
| `POST /api/analyze/stream` | Sequential SSE pipeline | E1 → E2 → E3 → E4 → E5 → E6 |
| `POST /api/trade-lanes` | Parallel multi-destination | E1 (optional) + parallel (E2 + E3) × N destinations |
| `POST /api/resolution/lookup` | Composite analysis | E1 + E3 + issue synthesis |
| `POST /api/resolution/suggest` | LLM-driven resolution | E5 + LLM step generation |
| Shipment analysis (internal) | Batch pipeline | E1 + E2 + E3 |
| Order analysis (internal) | Per-line-item pipeline | E1 + E2 + E3 per line item, then aggregation |

### Analytics Endpoints

| Endpoint | Type | Data Produced |
|----------|------|--------------|
| `GET /api/dashboard/{company_id}` | KPI aggregation | Total entries, duty paid, effective rate, FTA savings, compliance score, top HS codes/origins |
| `GET /api/platform/compliance-dashboard` | Compliance monitoring | Hold rate, compliance rate, by-status/by-jurisdiction breakdowns |
| `POST /api/trade-lanes` | Cost optimization | Multi-destination landed-cost comparison (sorted by cost) |
| `GET /api/broker/dashboard` | Workload intelligence | Pipeline stages, queue counts, time estimates, risk flags |
| `GET /api/broker/briefing` | Morning intelligence | Priority cases, alerts, recommended actions |
| `GET /api/regulatory/signals` | Regulatory monitoring | Active signals filtered by category/status/country |
| `POST /api/regulatory/scenario` | What-if analysis | Before/after cost comparison for regulatory changes |
| `POST /api/regulatory/portfolio` | Exposure analysis | Portfolio-wide impact of regulatory change |

### Trade Corridor Capabilities

**Static corridors** (10 defined in `shipments.py`): CN→US, DE→US, JP→US, TW→US, KR→US, VN→US, IN→US, IT→US, MX→US, CA→US

**Dynamic trade lane comparison** (`trade_lanes.py`): Accepts 1-10 destinations, parallelizes E2+E3, sorts by landed cost

---

## 5. Architecture Classification Recommendations

### First-Class Domains (Own Bounded Context)

These engines operate on distinct data, have independent lifecycles, and represent core business capabilities:

| Engine/Capability | Recommended Domain | Rationale |
|-------------------|-------------------|-----------|
| E1 Classification | **Classification Domain** | Core IP; versioned independently; consumed by all other domains |
| E2 Tariff + Financial fees | **Tariff & Duty Domain** | Complex jurisdiction-specific logic; rate tables versioned independently; regulatory-driven |
| E3 Compliance (PGA + DPS + UFLPA) | **Compliance Domain** | Regulatory-mandated; list updates independent of platform; real-time screening SLA |
| E4 FTA Qualification | **Trade Programs Domain** (or sub-domain of Tariff) | Rules-of-origin logic is complex and FTA-specific; could be sub-domain of Tariff |
| E6 Regulatory Intelligence + Monitor | **Regulatory Intelligence Domain** | External feed ingestion + signal management is a distinct lifecycle |

### Services Within Domains

These provide capabilities within a parent domain:

| Service | Parent Domain | Role |
|---------|--------------|------|
| Description Quality | Classification | Pre-classification quality gate |
| Embedding Service (Qdrant) | Classification / Exception | Vector search for CROSS rulings |
| DPS Screener | Compliance | Sub-engine for denied-party screening |
| UFLPA Screener | Compliance | Sub-engine for forced labor risk |
| PGA Screener | Compliance | Sub-engine for agency requirements |
| Currency Service | Tariff & Duty | Cross-currency conversion |
| Jurisdiction Service | Platform (shared kernel) | Country→jurisdiction mapping |

### Analytics Layer (Cross-Cutting)

These consume events from multiple domains and produce derived intelligence:

| Capability | Current Location | Analytics Layer Role |
|------------|-----------------|---------------------|
| Dashboard aggregation | `dashboard.py` | **Consume**: shipment.analyzed, entry.filed, entry.cleared events → **Produce**: KPI snapshots |
| Compliance dashboard | `platform.py` | **Consume**: entry.filed, shipment.held, shipment.cleared → **Produce**: hold_rate, compliance_rate |
| Trade lane comparison | `trade_lanes.py` | **Consume**: tariff.calculated, compliance.screened → **Produce**: lane cost rankings |
| Broker work intelligence | `broker_intelligence.py` | **Consume**: entry.claimed, checklist.updated, hold.created → **Produce**: priority scores, capacity |
| Scenario modeling | E6 sub-capability | **Consume**: regulatory.signal_created, tariff.calculated → **Produce**: cost impact projections |
| Portfolio impact | E6 sub-capability | **Consume**: regulatory.signal_created, product.cataloged → **Produce**: exposure analysis |
| Analysis watchdog | `analysis_watchdog.py` | **Consume**: shipment.updated → **Produce**: shipment.reanalyzed (trigger, not analytics) |

---

## 6. Event-Driven Architecture Recommendations

### Event Production by Domain

| Domain | Events Produced | Consumers |
|--------|----------------|-----------|
| **Classification** | `product.classified`, `classification.confidence_changed` | Tariff, Compliance, Analytics |
| **Tariff & Duty** | `tariff.calculated`, `landed_cost.computed`, `duty.program_applied` | Analytics, Broker Intelligence |
| **Compliance** | `compliance.screened`, `entity.flagged`, `uflpa.risk_assessed`, `pga.requirement_identified` | Broker Intelligence, Analytics, Entry Filing |
| **Trade Programs** | `fta.qualification_assessed`, `fta.savings_estimated` | Tariff, Analytics |
| **Regulatory Intelligence** | `regulatory.signal_created`, `regulatory.signal_confirmed`, `regulatory.scenario_modeled` | Analytics, Tariff (rate updates) |
| **Entry Filing** | `entry.filed`, `entry.accepted`, `entry.held`, `entry.cleared`, `entry.protested` | Analytics, Broker Intelligence |
| **Shipment Lifecycle** | `shipment.booked`, `shipment.in_transit`, `shipment.at_customs`, `shipment.cleared`, `shipment.delivered` | Analytics, Broker Intelligence, Compliance |

### Analytics Layer as Event Consumer

The analytics layer should be a pure **event consumer** with no direct API dependencies on engines:

```
Events (from domains)
    │
    ├──→ KPI Aggregator       → Dashboard snapshots (materialized views)
    ├──→ Compliance Monitor    → Hold rate, compliance trends
    ├──→ Trade Lane Optimizer  → Corridor cost rankings
    ├──→ Broker Intelligence   → Work priority, capacity planning
    ├──→ Portfolio Analyzer    → Exposure tracking across signals
    └──→ Anomaly Detector      → Tariff spikes, compliance pattern changes
```

### Migration Path for Embedded Actor Intelligence

1. **Phase 1**: Extract critical services (risk scoring, tariff calculation, PGA compliance) into `services/` with clean interfaces
2. **Phase 2**: Have actors call extracted services instead of inline logic; add event emission
3. **Phase 3**: Wire analytics layer to consume events instead of querying DB directly
4. **Phase 4**: Decouple simulation from platform — actors become thin event producers, platform services are the source of truth

---

## 7. Capabilities NOT in Architecture Document

The following capabilities exist in the current codebase but may not be represented in architecture planning:

| Capability | Location | Notes |
|------------|----------|-------|
| **Broker work sequencing algorithm** | `broker_intelligence.py` | Composite scoring with deadline pressure, dependency chains, financial impact weighting |
| **AI pre-work caching** | `broker_intelligence.py:AIPreworkManager` | Redis-cached CF-28 draft preparation |
| **Trade lane parallel comparison** | `trade_lanes.py` | Multi-destination asyncio.gather with sorted cost output |
| **Order multi-line analysis** | `orders.py` | Per-line-item E1+E2+E3 with aggregation |
| **Regulatory feed parsing** | `feed_parsers.py` | 4-source integration (Federal Register API, CBP CSMS, USTR, Google News) |
| **LLM-powered regulatory signal extraction** | `regulatory_monitor.py` | Batch extraction + 3-tier dedup (URL, hash, semantic) |
| **Disruption event generation** | `disruption.py` + `disruptions.py` | Geographically/seasonally aware with AI descriptions |
| **ISF 10+2 filing compliance** | `isf.py` | Late filing detection, mismatch types, penalty calculation |
| **Cargo exception modeling** | `exceptions.py` | Mode/product-aware damage, shortage, overage, weight variance |
| **Terminal operations** | `terminal.py` | Berth delays, chassis shortage, port congestion factors |
| **Document discrepancy detection** | `documents.py` (actor) | Invoice/description/weight/origin cert variance detection |
| **APHIS ISPM-15 wood packaging** | `pga.py` (actor) | Wood packaging detection + fumigation requirements |
| **TTB alcohol/tobacco permits** | `pga.py` (actor) | Permit type assignment (BWP, DSP) |
| **Payment method distribution** | `financial.py` (actor) | ACH/wire/PMS/check selection logic |
| **Exam fee determination** | `financial.py` (actor) | VACIS/intensive/tailgate fee assignment |
| **Consolidation grouping** | `consolidator.py` | Mode+origin+destination+carrier grouping with MAWB/MBL generation |
| **Shipper response quality modeling** | `shipper_response.py` | Channel/urgency-based response timing + document quality distribution |

---

## 8. Technology Dependency Map

```
                    ┌─────────────┐
                    │   Claude /  │
                    │   OpenAI    │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
         ┌────▼────┐  ┌───▼───┐  ┌────▼────┐
         │   E0    │  │  E1   │  │   E5    │
         │Pre-Clear│  │Classify│  │Exception│
         └─────────┘  └───────┘  └────┬────┘
                                      │
                                ┌─────▼─────┐
                                │  Qdrant   │
                                │(CROSS DB) │
                                └───────────┘

         ┌─────────┐  ┌───────┐  ┌─────────┐
         │   E2    │  │  E3   │  │   E4    │
         │ Tariff  │  │Comply │  │  FTA    │
         └─────────┘  └───────┘  └─────────┘
              (deterministic — no external deps)

         ┌─────────┐  ┌───────┐
         │   E6    │  │  E7   │
         │Reg Intel│  │  Docs │
         └────┬────┘  └───────┘
              │
         ┌────▼────────────────┐
         │  Feed Parsers       │
         │(Fed Reg, CBP, USTR, │
         │ Google News)        │
         └─────────────────────┘

Background Services:
  Regulatory Monitor  →  LLM + Redis + PostgreSQL + Feed Parsers
  Analysis Watchdog   →  Redis + PostgreSQL + E1/E2/E3
```

---

## 9. Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Hardcoded reference data (restricted parties, rate tables, rulings) | Data staleness; regulatory non-compliance | Move to versioned database with admin update workflow |
| Duplicated tariff logic (E2 vs. actor code) | Inconsistency between platform and simulation | Single service consumed by both |
| Actor-embedded risk scoring | Not available to real users; not auditable | Extract to platform service with audit trail |
| E4 limited to USMCA | No EU FTAs, RCEP, CPTPP support | Extend with additional trade program modules |
| Keyword-based PGA detection in actors | Unreliable product classification | Integrate with E1 classification output |
| E5 relies on 10 hardcoded rulings | Limited CROSS coverage | Expand Qdrant corpus; implement ruling ingestion pipeline |
| No explicit customs risk score | Decisions are implicit in probability cascades | Create explainable risk scoring engine |
