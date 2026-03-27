# Codebase Audit: Platform vs Simulation vs Engine

> **Purpose**: Map every module in the current monolith to one of three categories — Platform Service, Simulation, or Engine/Intelligence — to inform the NATS-based microservices redesign.
>
> **Key principle**: In the new architecture, **Simulation is a separate platform**. Actors use platform APIs like external humans/systems would. They cannot emit clearance-platform events directly.

---

## Table of Contents

1. [Engines (E0–E7)](#1-engines-e0e7)
2. [Orchestration Layer](#2-orchestration-layer)
3. [API Routes](#3-api-routes)
4. [Services](#4-services)
5. [Simulation](#5-simulation)
6. [Knowledge / Data Models](#6-knowledge--data-models)
7. [Infrastructure / Config](#7-infrastructure--config)
8. [Critical Findings: Misplaced Logic](#8-critical-findings-misplaced-logic)
9. [Dependency Map](#9-dependency-map)

---

## 1. Engines (E0–E7)

All engines inherit from `BaseEngine` (`engines/base.py`) and return `EngineOutput` envelopes. These are **pure intelligence/computation services** — they belong inside their functional domain microservices.

### E0 — Pre-Clearance Intelligence
| Attribute | Value |
|---|---|
| **Location** | `engines/e0_preclearance/` (engine.py, prompts.py, tools.py) |
| **Category** | **Engine/Intelligence** |
| **State managed** | None (stateless) |
| **APIs exposed** | None directly (invoked by simulation `PreClearanceAdapter`) |
| **Dependencies** | `LLMService` (optional), hardcoded screening tools |
| **What it does** | Agentic LLM tool-calling loop for pre-clearance screening. Screens entities against denied-party lists, validates HS codes, checks value reasonableness, identifies required documents, assesses origin risk. Deterministic fallback when no LLM. |
| **New architecture** | Part of **Clearance Service** (pre-clearance subdomain). Consumes `shipment.booking_received` events, emits `preclearance.screening_complete` events. |

### E1 — Classification Intelligence
| Attribute | Value |
|---|---|
| **Location** | `engines/e1_classification/` (engine.py, prompts.py, validator.py, confidence.py) |
| **Category** | **Engine/Intelligence** |
| **State managed** | None (stateless); reads from `CHAPTER_REFERENCE` and `_PRODUCT_CODE_MAP` in-memory |
| **APIs exposed** | Used by routes/classify.py SSE, routes/analysis.py SSE, pipeline, watchdog |
| **Dependencies** | `LLMService` (optional), `HSCodeValidator` |
| **What it does** | Two-phase HS classification: chapter ID (keyword or LLM) → full subheading (keyword or LLM). Streaming + non-streaming. |
| **New architecture** | Part of **Classification Service**. Consumes `product.classify_requested`, emits `product.classified`. Reference data (`CHAPTER_REFERENCE`, `_PRODUCT_CODE_MAP`) → separate reference data store. |

### E2 — Global Tariff & Landed Cost
| Attribute | Value |
|---|---|
| **Location** | `engines/e2_tariff/` (engine.py, tax_regime.py, landed_cost.py, programs/__init__.py, regimes/{us,eu,cn,br,india,uk}.py) |
| **Category** | **Engine/Intelligence** |
| **State managed** | None (stateless, fully deterministic) |
| **APIs exposed** | Used by routes/tariff.py, pipeline, watchdog, E6 scenario modelling |
| **Dependencies** | Regime-specific rate tables (hardcoded), `CurrencyService` |
| **What it does** | Dispatches to jurisdiction-specific `TaxRegimeEngine` (US, EU/27, GB, CN, BR, IN). Returns complete duty/tax/fee breakdown with legal citations. |
| **New architecture** | Part of **Tariff Service**. Consumes `product.classified` (needs HS code), emits `tariff.calculated`. Rate tables → separate reference data store. |

### E3 — Compliance Screening
| Attribute | Value |
|---|---|
| **Location** | `engines/e3_compliance/` (engine.py, pga.py, dps.py, uflpa.py, fuzzy_match.py) |
| **Category** | **Engine/Intelligence** |
| **State managed** | None (stateless) |
| **APIs exposed** | Used by routes/compliance.py, routes/screening.py, pipeline, watchdog |
| **Dependencies** | Hardcoded PGA rules, DPS entity lists, UFLPA risk factors, `fuzzy_match` |
| **What it does** | Orchestrates three sub-engines in parallel: PGA screening, Denied Party screening, UFLPA risk assessment. Returns overall compliance status (HOLD/REVIEW/PGA_REQUIRED/CLEAR). |
| **New architecture** | Part of **Compliance Service**. Consumes `shipment.requires_screening`, emits `compliance.screening_complete`. DPS lists and PGA rules → separate reference data store. |

### E4 — FTA Qualification
| Attribute | Value |
|---|---|
| **Location** | `engines/e4_fta/` (engine.py) |
| **Category** | **Engine/Intelligence** |
| **State managed** | None (stateless) |
| **APIs exposed** | Used by routes/fta.py, pipeline |
| **Dependencies** | `USMCA_RULES` (hardcoded), E2 US regime for savings estimation |
| **What it does** | USMCA rules-of-origin analysis. Tariff shift test, RVC calculation (when BOM provided), savings estimation. |
| **New architecture** | Part of **Trade Agreements Service** (or merged into Tariff Service). Consumes `tariff.calculated`, emits `fta.eligibility_assessed`. |

### E5 — Exception Analysis
| Attribute | Value |
|---|---|
| **Location** | `engines/e5_exception/` (engine.py) |
| **Category** | **Engine/Intelligence** |
| **State managed** | None (stateless); reads from `REFERENCE_RULINGS` in-memory + Qdrant |
| **APIs exposed** | Used by routes/exception.py SSE |
| **Dependencies** | `LLMService` (optional), `EmbeddingService`/Qdrant (optional) |
| **What it does** | CROSS ruling search (Qdrant semantic + keyword fallback) + LLM-powered response drafting for CBP exceptions. Citation validation against known corpus. |
| **New architecture** | Part of **Exception Resolution Service**. Consumes `entry.exception_raised`, emits `exception.response_drafted`. CROSS rulings → Qdrant + reference data store. |

### E6 — Regulatory Change Intelligence
| Attribute | Value |
|---|---|
| **Location** | `engines/e6_regulatory/` (engine.py, db.py) |
| **Category** | **Engine/Intelligence** |
| **State managed** | Reads from `regulatory_signals` table (or hardcoded fallback) |
| **APIs exposed** | Used by routes/regulatory.py |
| **Dependencies** | DB session (optional), E2 for scenario modelling |
| **What it does** | Three sub-capabilities: 6A Signal Feed (curated regulatory signals), 6B Scenario Modelling (landed-cost impact projection), 6C Portfolio Impact Analysis. |
| **New architecture** | Part of **Regulatory Intelligence Service**. Emits `regulatory.signal_detected`, consumes portfolio data for impact analysis. |

### E7 — Document Intelligence
| Attribute | Value |
|---|---|
| **Location** | `engines/e7_documents/` (engine.py) |
| **Category** | **Engine/Intelligence** |
| **State managed** | None (thin wrapper, delegates to routes/documents.py logic) |
| **APIs exposed** | Pipeline wrapper for 7A (requirements lookup) |
| **Dependencies** | `routes/documents.py` reference data |
| **What it does** | Document requirements lookup. Determines required docs based on HS code, origin, destination, compliance flags. |
| **New architecture** | Part of **Document Service**. Consumes `shipment.created`, emits `documents.requirements_determined`. |

---

## 2. Orchestration Layer

| File | Category | What it does | New Architecture |
|---|---|---|---|
| `orchestration/pipeline.py` | **Platform Service** | `AnalysisPipeline`: E1 → parallel(E2, E3, E4) → assembly. Both streaming and non-streaming. The primary end-to-end analysis entry point. | **Replaced by NATS choreography**. E1 completes → emits event → E2/E3/E4 react independently. Assembly is done by a read model/aggregator consuming all engine-complete events. |
| `orchestration/parallel.py` | **Platform Service** | `run_parallel()`: asyncio.gather with timeout. | Eliminated — NATS pub/sub handles parallel invocation natively. |
| `orchestration/error_handling.py` | **Platform Service** | Error handling utilities for pipeline. | Absorbed into each domain service's error handling + dead-letter topics. |

---

## 3. API Routes

### Platform Routes (business logic that exists in production)

| Route File | Prefix | Category | State Managed | New Architecture |
|---|---|---|---|---|
| `routes/analysis.py` | `/api/analyze/stream` | **Platform** | None (delegates to pipeline) | API Gateway → Analysis aggregate service |
| `routes/classify.py` | `/api/classify/stream` | **Platform** | None (delegates to E1) | API Gateway → Classification Service |
| `routes/tariff.py` | `/api/tariff/*` | **Platform** | None (delegates to E2) | API Gateway → Tariff Service |
| `routes/compliance.py` | `/api/compliance/*` | **Platform** | None (delegates to E3) | API Gateway → Compliance Service |
| `routes/fta.py` | `/api/fta/*` | **Platform** | None (delegates to E4) | API Gateway → Trade Agreements Service |
| `routes/exception.py` | `/api/exception/*` | **Platform** | None (delegates to E5) | API Gateway → Exception Resolution Service |
| `routes/screening.py` | `/api/screening/*` | **Platform** | None (delegates to E3 DPS) | API Gateway → Compliance Service |
| `routes/regulatory.py` | `/api/regulatory/*` | **Platform** | Reads `regulatory_signals` table | API Gateway → Regulatory Intelligence Service |
| `routes/documents.py` | `/api/documents/*` | **Platform** | Reads/writes `documents` table | API Gateway → Document Service |
| `routes/products.py` | `/api/products/*` | **Platform** | CRUD on `products` table | API Gateway → Product Catalog Service |
| `routes/orders.py` | `/api/orders/*` | **Platform** | CRUD on `orders`, `order_line_items` | API Gateway → Order Service |
| `routes/shipments.py` | `/api/shipments/*` | **Platform** | CRUD on `shipments` table, seeds demo data | API Gateway → Shipment Service |
| `routes/consolidations.py` | `/api/consolidations/*` | **Platform** | CRUD on `consolidations` table | API Gateway → Shipment Service (or Logistics Service) |
| `routes/broker.py` | `/api/broker/*` | **Platform** | CRUD on `brokers`, `broker_assignments`, `entry_filings`, `broker_messages` | API Gateway → Broker Service |
| `routes/dashboard.py` | `/api/dashboard/*` | **Platform** | Reads aggregated data | API Gateway → Dashboard Service (reads from cache/read models) |
| `routes/trade_lanes.py` | `/api/trade-lanes/*` | **Platform** | Returns trade lane reference data | API Gateway → Reference Data Service |
| `routes/chat.py` | `/api/chat/stream` | **Platform** | LLM chat (stateless per request) | API Gateway → Chat Service |
| `routes/resolution.py` | `/api/resolution/*` | **Platform** | LLM-powered resolution tools | API Gateway → Exception Resolution Service |
| `routes/description_quality.py` | `/api/description-quality` | **Platform** | None (delegates to service) | API Gateway → Classification Service |
| `routes/exception_status.py` | `/api/exception-status/*` | **Platform** | Reads shipment exception data | API Gateway → Exception Resolution Service |
| `routes/jurisdiction.py` | `/api/jurisdiction/*`, `/api/countries/*` | **Platform** | Returns jurisdiction config data | API Gateway → Reference Data Service |
| `routes/platform.py` | `/api/platform/*` | **Platform** | Control tower data | API Gateway → Dashboard Service |
| `routes/session.py` | `/api/session/*` | **Platform** | Session management (basic) | API Gateway |

### Simulation Routes

| Route File | Prefix | Category | What it does | New Architecture |
|---|---|---|---|---|
| `routes/simulation.py` | `/api/simulation/*` | **Simulation** | Start/stop/pause/reset sim, SSE dashboard stream, actor config management | Moves to **Simulation Platform** (separate deployment). Calls platform APIs via HTTP/NATS. |

---

## 4. Services

| Service File | Category | State Managed | What it does | New Architecture |
|---|---|---|---|---|
| `services/llm.py` | **Shared Infrastructure** | None (stateless) | Unified LLM client (Claude primary, OpenAI fallback). Streaming + non-streaming + tool-calling. | **Shared LLM Gateway Service** consumed by all domain services that need LLM. |
| `services/cache.py` | **Shared Infrastructure** | Redis | Redis caching with fail-open semantics. Key builders for tariff, classification, compliance lookups. | Each domain service manages its own Redis cache namespace. Shared Redis infrastructure. |
| `services/embedding.py` | **Shared Infrastructure** | Qdrant | Qdrant vector search for CROSS rulings. Manages `cross_rulings` collection, embedding generation via OpenAI. | Part of **Exception Resolution Service** (owns its Qdrant collection). |
| `services/database.py` | **Shared Infrastructure** | PostgreSQL | Database session management. | Each domain service owns its database schema. Shared PostgreSQL with schema-per-service or DB-per-service. |
| `services/currency.py` | **Platform Service** | None (in-memory rates, loadable from DB) | Currency conversion via USD intermediary. Locale-formatted output. | Part of **Reference Data Service** or shared utility. |
| `services/dashboard.py` | **Platform Service** | Reads from DB | Bridges dashboard API to SQL queries. | Part of **Dashboard Service** (reads from CQRS read models). |
| `services/description_quality.py` | **Platform Service** | None (stateless) | LLM-powered product description quality assessment with deterministic fallback. | Part of **Classification Service**. |
| `services/shipment_analysis.py` | **Platform Service** | Writes `shipment.analysis`, `shipment.codes`, `shipment.financials` | Orchestrates E1/E2/E3 on a single shipment, stores results. Used by watchdog and broker endpoints. | **Replaced by event choreography**: shipment updated → classification requested → tariff calculated → compliance screened → results written to shipment read model. |
| `services/analysis_watchdog.py` | **Platform Service** | Reads/writes `shipments` table, Redis locks | Scheduled background job. Detects stale shipments (updated_at > analyzed_at), re-runs E1/E2/E3. | **Replaced by event-driven reactivity**: any shipment mutation emits event → engines automatically re-process. No polling needed. |
| `services/broker_intelligence.py` | **Platform Service** | None (pure computation) | Work intelligence for broker queue: time estimates, complexity scoring, financial impact, resolution paths, optimal sequencing, AI pre-work caching. | Part of **Broker Service** (intelligence subdomain). |
| `services/regulatory_monitor.py` | **Platform Service** | Writes `regulatory_signals` table, Redis locks | Background regulatory feed monitor. Fetches articles, LLM-extracts signals, 3-tier dedup (URL, hash, semantic), stores to DB. | Part of **Regulatory Intelligence Service**. Runs as scheduled NATS-triggered job. |
| `services/feed_parsers.py` | **Platform Service** | None (fetches external feeds) | Feed fetchers for government/news sources. | Part of **Regulatory Intelligence Service** (data ingestion). |
| `services/jurisdiction.py` | **Platform Service** | None (in-memory config) | Maps ISO-2 country codes to jurisdiction configurations (authority names, filing workflows, currency defaults). | Part of **Reference Data Service**. |

---

## 5. Simulation

### Core Framework

| File | What it does | New Architecture |
|---|---|---|
| `simulation/coordinator.py` | Orchestrates actor lifecycle, clock, SSE broadcast. Central control point for start/stop/pause/reset. Creates 19 actors. | **Simulation Platform coordinator** — separate deployment. |
| `simulation/event_bus.py` | Redis pub/sub inter-actor communication. Channel `sim:events`. In-memory fallback. | **REPLACED**: Simulation actors communicate via platform REST/NATS APIs, not an internal event bus. Between-actor coordination uses NATS within the sim platform. |
| `simulation/clock.py` | Simulated time with configurable ratio. `sleep_sim_minutes()`. | **Stays in Simulation Platform**. |
| `simulation/config.py` | Pydantic configs for all 19 actors + top-level sim config. | **Stays in Simulation Platform**. |
| `simulation/state_machine.py` | Guarded shipment status transitions. All status changes must go through `transition()`. Multi-jurisdiction support. | **CRITICAL: THIS IS PLATFORM LOGIC.** Must move to **Shipment Service**. The state machine enforces business rules (valid transitions, actor permissions). Currently coupled to simulation but is fundamentally a platform concern. |
| `simulation/state_machines/{us,eu,cn,br,in_}.py` | Jurisdiction-specific transition maps. | **PLATFORM LOGIC**. Part of **Shipment Service** (jurisdiction-aware state machine). |
| `simulation/routing.py` | Multi-modal routing engine. Generates realistic leg sequences based on geography, carrier type, corridor. | **AMBIGUOUS**. The routing *logic* (hub networks, transshipment points, regulatory touchpoints) is **platform intelligence** that would exist in production. The *randomization/simulation* aspect is simulation. Split: routing rules → **Logistics Service**; random selection → Simulation Platform. |
| `simulation/reference_data.py` | `CarrierService`, `Corridor`, `TERRITORY_FILINGS`, `TERRITORY_RISKS` reference data. | **Platform reference data**. Part of **Reference Data Service**. |
| `simulation/reference_generators.py` | Generates realistic reference numbers (HAWB, MBL, entry numbers, etc.). | **PLATFORM LOGIC** (reference number generation exists in production). Part of **Shipment Service**. |
| `simulation/intermediate_events.py` | Generates intermediate tracking events for shipment legs. | **AMBIGUOUS**. Event *templates* (what events occur on each leg type) are platform knowledge. The *timing/randomization* is simulation. |
| `simulation/calendar.py` | Business calendar with holidays and working hours. | **Platform utility**. Part of shared utilities. |
| `simulation/disruptions.py` | Disruption scenarios (weather, labor, congestion). | **Simulation only**. |
| `simulation/dashboard_aggregator.py` | SQL aggregation queries for dashboard KPIs from shipments table. | **PLATFORM LOGIC**. Part of **Dashboard Service** (CQRS read model builder). |

### Actors (19 total)

**All actors are Simulation** — they emulate external parties. However, some contain logic that should be platform services.

| Actor | What it emulates | Logic that should be platform | New Architecture |
|---|---|---|---|
| `actors/shipper.py` | Shippers creating new bookings. Creates shipments, assigns products, generates orders. | Product selection, value calculation | **Simulation Platform**. Creates shipments via Shipment Service API. |
| `actors/consolidator.py` | Freight consolidators grouping shipments into MAWB/MBL. Creates `Consolidation` records, assigns shipments. | Consolidation business rules (min/max group size, mode matching) | **Simulation Platform** calls Logistics Service API. Consolidation rules → **Logistics Service**. |
| `actors/carrier.py` | Carriers moving shipments. Advances status through `in_transit → arrived_port → customs_hold/at_customs`. | Status transition logic (uses `state_machine.transition()`) | **Simulation Platform** calls Shipment Service API. |
| `actors/customs.py` | **CBP/customs authority processing entries.** STP clearance, manual review, holds, inspections, reclassifications. | **ALL OF THIS IS PLATFORM LOGIC.** Customs processing (STP scoring, hold determination, inspection scheduling) is what the clearance platform does. | **CRITICAL REDESIGN**: Extract to **Clearance Service** (the core platform service). Simulation actor becomes a thin wrapper calling Clearance Service APIs. |
| `actors/pga.py` | **PGA agencies (FDA, EPA, CPSC) reviewing entries.** | **PLATFORM LOGIC.** PGA review workflow (requirement checking, approval/rejection) belongs in the platform. | Extract to **Compliance Service** (PGA subdomain). |
| `actors/compliance_engine.py` | Runs compliance engines on new shipments. | Invokes E1/E2/E3. | **PLATFORM LOGIC.** This is the analysis pipeline trigger — should be event-driven in the platform. |
| `actors/preclearance.py` | Adapter that invokes E0 on new shipments. | Invokes E0 pre-clearance engine. | **PLATFORM LOGIC.** Pre-clearance screening trigger — should be event-driven. |
| `actors/resolution.py` | Resolves held shipments over time. | Hold resolution logic, escalation rules | **PLATFORM LOGIC for resolution workflows.** Extract to **Exception Resolution Service**. |
| `actors/shipper_response.py` | Shippers responding to information requests. | Response templates, document submission | **Simulation Platform**. Calls platform APIs to submit responses. |
| `actors/cage.py` | Cargo cage management. Assigns cage slots, tracks dwell time, calculates GO deadlines, storage costs. | **Cage tracking is PLATFORM LOGIC.** Physical cargo location, dwell monitoring, and GO deadline calculation exist in production. | Extract to **Cargo Management Service**. |
| `actors/terminal.py` | Port/terminal operations. Berth delays, chassis shortages, congestion. | Terminal operations data (berth status, congestion metrics) could be platform data feeds. | **Mostly Simulation**. Terminal status feeds → **Logistics Service** (data ingestion). |
| `actors/demurrage.py` | Demurrage & detention tracking. Free time calculations, fee accrual. | **D&D calculation is PLATFORM LOGIC.** Free time rules, fee schedules, appointment tracking. | Extract to **Financial Service** (D&D subdomain). |
| `actors/disruption.py` | Weather/labor disruption events. LLM-generated descriptions. | Disruption impact assessment could be platform intelligence. | **Mostly Simulation**. Impact assessment → **Regulatory Intelligence Service**. |
| `actors/exceptions.py` | Cargo damage, shortage, overage events. LLM-generated descriptions. | Exception categorization and routing. | **Simulation** (generates events). Exception management → **Exception Resolution Service**. |
| `actors/documents.py` | Document validation. Invoice/packing list/certificate checks. | **Document validation is PLATFORM LOGIC.** Mismatch detection, variance checking. | Extract to **Document Service** (validation subdomain). |
| `actors/financial.py` | Financial calculations. Bond requirements, fee schedules, payment processing. | **PLATFORM LOGIC.** Bond calculation, fee determination, payment tracking. | Extract to **Financial Service**. |
| `actors/isf.py` | ISF 10+2 filing simulation. Late filing detection, data mismatch checks. | **ISF filing workflow is PLATFORM LOGIC.** Filing deadlines, validation rules. | Extract to **Clearance Service** (ISF subdomain). |
| `actors/cbp_authority.py` | **Simulates CBP responses to entry filings.** Accept/CF-28/CF-29/exam/reject. | This is a CBP *emulation* — not platform logic. But the response *processing* logic (what to do with a CF-28) is platform. | **Simulation Platform** (CBP emulation). Response *handling* → **Clearance Service**. |
| `actors/broker_sim.py` | Automated broker actions. Claims entries, fills checklists, submits filings, responds to CF-28s. | Broker workflow intelligence (auto-claim, optimal sequencing) | **Simulation Platform**. Calls Broker Service APIs. |

---

## 6. Knowledge / Data Models

### ORM Models (`knowledge/models/`)

| Model | Table | Category | Owned By (new arch) |
|---|---|---|---|
| `RegulatorySignal` | `regulatory_signals` | **Platform** | Regulatory Intelligence Service |
| `DemoShipment` | `demo_shipments` | **Simulation** (demo data) | Simulation Platform or deprecated |
| `Product` | `products` | **Platform** | Product Catalog Service |
| `Order` | `orders` | **Platform** | Order Service |
| `OrderLineItem` | `order_line_items` | **Platform** | Order Service |
| `Consolidation` | `consolidations` | **Platform** | Logistics Service |
| `Shipment` | `shipments` | **Platform** | Shipment Service |
| `Document` | `documents` | **Platform** | Document Service |
| `Broker` | `brokers` | **Platform** | Broker Service |
| `BrokerAssignment` | `broker_assignments` | **Platform** | Broker Service |
| `EntryFiling` | `entry_filings` | **Platform** | Clearance Service |
| `BrokerMessage` | `broker_messages` | **Platform** | Broker Service |
| `CrossRuling` | `cross_rulings` | **Platform** | Exception Resolution Service |
| `TaxRate` | `tax_rates` | **Platform** | Reference Data Service |
| `FeeSchedule` | `fee_schedules` | **Platform** | Reference Data Service |
| `ExchangeRate` | `exchange_rates` | **Platform** | Reference Data Service |
| `TaxRegimeTemplate` | `tax_regime_templates` | **Platform** | Reference Data Service |

### Reference Data in Models

| Model File | Contents | Category |
|---|---|---|
| `models/base.py` | `Base`, `TimestampMixin` | Shared infrastructure |
| `models/tariff.py` | Tariff knowledge model schemas | Platform (Tariff Service) |
| `models/compliance.py` | Compliance knowledge model schemas | Platform (Compliance Service) |
| `models/reference.py` | `CrossRuling`, `TaxRate`, `FeeSchedule`, `ExchangeRate`, `TaxRegimeTemplate` | Platform (Reference Data Service) |
| `models/operational.py` | All operational models (see table above) | Platform (various services) |

### API Schemas (`api/schemas/`)

| Schema File | Category | Owned By (new arch) |
|---|---|---|
| `schemas/common.py` | Shared | Shared types |
| `schemas/analysis.py` | Platform | Analysis aggregate |
| `schemas/products.py` | Platform | Product Catalog Service |
| `schemas/orders.py` | Platform | Order Service |
| `schemas/shipments.py` | Platform | Shipment Service |
| `schemas/documents.py` | Platform | Document Service |
| `schemas/resolution.py` | Platform | Exception Resolution Service |
| `schemas/trade_lanes.py` | Platform | Reference Data Service |
| `schemas/chat.py` | Platform | Chat Service |
| `schemas/description_quality.py` | Platform | Classification Service |
| `schemas/simulation.py` | Simulation | Simulation Platform |

---

## 7. Infrastructure / Config

| File | Category | New Architecture |
|---|---|---|
| `main.py` | **Monolith entry point** | Replaced by individual service entry points + API gateway |
| `config.py` (implied) | **Shared config** | Each service has its own config. Shared infra config (DB URLs, Redis URLs) via env vars. |
| `api/streaming.py` | **Platform** | SSE handling moves to API gateway or individual services |
| `api/agent/__init__.py` | **Platform** | Chat/agent capabilities → Chat Service |

---

## 8. Critical Findings: Misplaced Logic

### Logic in Simulation that MUST become Platform Services

These represent the most important redesign items. In the current monolith, these are implemented as simulation actors but contain core business logic that exists in any production customs clearance platform:

#### 8.1 Customs Processing (`actors/customs.py`) → Clearance Service
- **STP (Simplified Trade Processing) scoring** — determines if an entry can clear automatically
- **Hold determination** — business rules for when to hold shipments (UFLPA, entity screening, inspection)
- **Reclassification logic** — detecting and flagging HS code discrepancies
- **Inspection scheduling** — deciding which entries need physical exam
- This is the **core value proposition** of the clearance platform

#### 8.2 State Machine (`simulation/state_machine.py`) → Shipment Service
- **Guarded status transitions** with actor permissions
- **Multi-jurisdiction support** (US, EU, CN, BR, IN transition maps)
- **Event logging** on each transition
- Currently used by ALL actors to change shipment status — this is platform logic

#### 8.3 Cage Management (`actors/cage.py`) → Cargo Management Service
- **Physical cargo location tracking**
- **Dwell time monitoring**
- **GO deadline calculation** (General Order — when unclaimed cargo becomes government property)
- **Storage cost accrual**

#### 8.4 Document Validation (`actors/documents.py`) → Document Service
- **Invoice/packing list cross-validation**
- **Weight/quantity variance detection**
- **Origin certificate verification**
- **USMCA certification checking**

#### 8.5 Financial Calculations (`actors/financial.py`) → Financial Service
- **Bond requirement calculation** (single entry vs continuous)
- **Fee schedule application** (MPF, HMF, broker fees)
- **Payment method processing**
- **Duty deposit calculations**

#### 8.6 D&D Tracking (`actors/demurrage.py`) → Financial Service
- **Free time calculations** (ocean import: 5 days, air: 48 hours)
- **Demurrage/detention fee accrual**
- **Appointment tracking**

#### 8.7 ISF Filing (`actors/isf.py`) → Clearance Service
- **ISF 10+2 filing deadline enforcement**
- **Data validation** (shipper, consignee, HTS matching)
- **Late filing penalty assessment**

#### 8.8 PGA Processing (`actors/pga.py`) → Compliance Service
- **Agency-specific review workflows** (FDA, EPA, CPSC)
- **Approval/rejection determination**
- **Agency-specific documentation requirements**

#### 8.9 Dashboard Aggregation (`simulation/dashboard_aggregator.py`) → Dashboard Service
- **SQL aggregation queries** for KPIs
- **Status distribution, corridor volumes, financial summaries**
- **Should be a CQRS read model** built from events, not polling queries

#### 8.10 Reference Number Generation (`simulation/reference_generators.py`) → Shipment Service
- **HAWB, MBL, entry number, tracking number generation**
- Format-correct reference numbers following real-world patterns

#### 8.11 Routing Intelligence (`simulation/routing.py`) → Logistics Service
- **Hub routing tables** (integrator networks, ocean transshipment)
- **Regulatory touchpoint derivation** (which territories need clearance)
- **Border-crossing mode determination**

---

## 9. Dependency Map

### Current Inter-Module Dependencies

```
main.py
├── All API routes (25 routers)
├── SimulationCoordinator
│   ├── SimulationClock
│   ├── EventBus (Redis pub/sub)
│   ├── 19 Actors (all depend on clock, event_bus, session_factory)
│   │   ├── Most actors → state_machine.transition()
│   │   ├── customs → state_machine + hardcoded STP/hold logic
│   │   ├── compliance_engine → E1, E2, E3
│   │   ├── preclearance → E0
│   │   └── shipper → reference_generators, routing, reference_data
│   └── dashboard_aggregator
├── RegulatoryMonitor → LLMService, feed_parsers, DB
├── AnalysisWatchdog → shipment_analysis → E1, E2, E3
├── DB Engine (SQLAlchemy async)
├── Redis
└── Qdrant

AnalysisPipeline (orchestration/)
├── E1 → LLMService (optional)
├── E2 → TaxRegimeEngines (stateless)
├── E3 → PGA + DPS + UFLPA sub-engines
└── E4 → USMCA rules + E2 for savings

Services layer:
├── LLMService → Anthropic + OpenAI clients
├── CacheService → Redis
├── EmbeddingService → Qdrant + OpenAI
├── CurrencyService → in-memory rates
├── BrokerIntelligenceService → pure computation
├── DescriptionQualityService → LLMService
└── ShipmentAnalysis → E1, E2, E3
```

### Database Tables (shared PostgreSQL)

All 17+ tables currently in a single database. In the new architecture, each domain service owns its schema:

| Service | Tables |
|---|---|
| Shipment Service | `shipments`, `consolidations` |
| Order Service | `orders`, `order_line_items` |
| Product Catalog Service | `products` |
| Broker Service | `brokers`, `broker_assignments`, `broker_messages` |
| Clearance Service | `entry_filings` |
| Document Service | `documents` |
| Regulatory Intelligence Service | `regulatory_signals` |
| Reference Data Service | `cross_rulings`, `tax_rates`, `fee_schedules`, `exchange_rates`, `tax_regime_templates` |
| Simulation (deprecated) | `demo_shipments` |

---

## Summary Statistics

| Category | Count |
|---|---|
| Engine files | 38 |
| API route files | 25 |
| Service files | 14 |
| Simulation files | 39 |
| Knowledge/model files | 8 |
| Orchestration files | 3 |
| **Total backend Python files** | **~127** |

| Classification | Modules |
|---|---|
| **Pure Platform** | Routes (24), Services (11), Orchestration (3), Models (8) = **46** |
| **Pure Simulation** | Simulation framework (5), Pure actors (4), Sim routes (1) = **10** |
| **Pure Engine** | E0-E7 engine modules = **38** |
| **Misplaced (Sim → Platform)** | 11 actors + 5 sim modules containing platform logic = **16** |
| **Shared Infrastructure** | LLM, Cache, Embedding, DB, Config = **5** |
