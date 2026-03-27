# Event-Driven Architecture Design: State-Transfer Pattern

**Author:** event-architect
**Status:** FINAL
**Date:** 2026-02-08
**Reviewers:** design-challenger, codebase-analyst
**Feasibility Check:** codebase-analyst (19 actors analyzed — 12 clean, 5 adaptation, 2 significant rework)

---

## 1. Executive Summary

The Clearance Vibe platform has a fundamental architectural problem: 19 simulation actors and all API endpoints share a single PostgreSQL connection pool, with actors using unbounded `SELECT ... FOR UPDATE SKIP LOCKED` queries that cause connection starvation (7-60s API response times) and silent processing skips (shipments missed by actors due to non-deterministic locking).

This document designs an **event-driven architecture** using the **state-transfer pattern** to solve both the performance and correctness problems. The architecture:

- **Eliminates lock contention** by replacing poll-and-lock with event-driven actor coordination
- **Guarantees processing completeness** — every shipment is seen by every actor that needs it (no silent SKIP LOCKED drops)
- **Decouples reads from writes** via Redis read model projections, eliminating API-actor connection competition
- **Makes the correct pattern the default** — new actors subscribe to events; contention bugs can't happen by construction

### Design Principles

1. **Correctness first**: Every event is tracked, acknowledged, or retried. No silent drops.
2. **State-transfer, not event-sourcing**: Events carry complete state snapshots. Consumers never need to query the database to process an event.
3. **Contention prevention by construction**: The architecture makes it impossible to introduce the poll-and-lock anti-pattern.
4. **Proactive evaluation, not sequential processing**: Proactive evaluators (broker, compliance, financial, documents) act at the earliest moment their data prerequisites are met. Reactive responders (customs authority, CBP) adjudicate explicit submissions. Two event types: information signals (consumed by proactive evaluators) and commands (consumed by reactive responders). (See Section 5.3, 5.5.)
5. **Testability**: Per-shipment business logic is pure functions. Event wiring is separately testable.
6. **Extensibility**: New jurisdictions, actors, and UI surfaces are added by subscribing to existing events — no contention analysis required.
7. **No Kafka**: Redis Streams provides ordered, persistent, consumer-group-based messaging suitable for our single-instance topology.
8. **Hybrid where appropriate**: Time-dependent actors use event-triggered start + sim-clock polling for completion, respecting the simulation clock's dynamic time ratio.

---

## 2. Bounded Contexts / Business Domains

The bounded contexts below are derived from the **full business domain** of a customs clearance platform, informed by the comprehensive business capability map (`docs/architecture-business-capabilities.md`). Each context is defined by a coherent business capability with its own entities, lifecycle, stakeholders, and data ownership.

**Reconciliation note:** The capability map identified 14 business capabilities and proposed 11 bounded contexts. After architectural analysis, we settled on **10 bounded contexts**. Key decisions:
- **Products split from Classification** — Products are pure CRUD master data; Classification/Tariff is a stateless computation engine. Different lifecycles.
- **Broker Operations elevated to core domain** — 30+ API endpoints, richest UI, own state and lifecycle. Too large and distinct to be a sub-domain.
- **Exception Management extracted** — 5 actors, cross-cutting concern, own resolution lifecycle.
- **Regulatory Intelligence separated from Compliance** — autonomous monitoring service with no shipment lifecycle dependency.
- **Regulatory Authority NOT separated** — CBP/PGA are simulated actors within Customs & Declarations, not a separate business domain. In production, they'd be external systems.
- **Reporting NOT a separate context** — it's a read projection pattern (dashboard projector), not a bounded context in the DDD sense.

### 2.1 Product Catalog

**Business capability:** Maintain the master catalog of products that can be shipped internationally. Each product has trade-relevant attributes (HS codes, country of origin, descriptions, compliance flags).

| Dimension | Details |
|-----------|---------|
| **Core entities** | `Product` |
| **Tables owned** | `products` |
| **API routes** | `products.py` (CRUD: list, detail, create, update, delete, seed) |
| **UI surfaces** | Buyer StoreFront/ProductPage, Shipper Catalog/AddProduct, Platform Product Analysis |
| **Actors** | None (reference data — products are created by users and seed data, not by simulation actors) |
| **External integrations** | Qdrant (semantic product search) |

**Why this is a domain:** Products are the **starting point of the entire value chain**: Product → Order → Shipment → Clearance → Settlement. A product has its own master data lifecycle (created, classified, updated), its own API surface (6 endpoints), and its own UI across 3 surfaces. Products exist independently of any order or shipment — the buyer browses products without any logistics context.

**Separation from Trade Intelligence:** The capability map correctly identifies that Products are pure CRUD master data with no lifecycle states and no simulation actors. Classification engines (HS code determination, description quality scoring) are stateless computation that *operates on* products but doesn't *own* them. Products produce `ProductCreated`, `ProductUpdated` events; classification is a synchronous service call.

### 2.2 Trade Intelligence (Classification + Tariff)

**Business capability:** Determine the correct HS code for products, calculate applicable duties/taxes across multiple jurisdictions, apply trade programs and special tariff regimes. This is a **shared kernel** — a stateless computation engine consumed by multiple domains.

| Dimension | Details |
|-----------|---------|
| **Core entities** | `HTSUSChapter`, `HTSUSHeading`, `CrossRuling`, `Section301List`, `Section232Scope`, `IEEPARate`, `ADCVDOrder`, `TaxRate`, `FeeSchedule`, `ExchangeRate`, `TaxRegimeTemplate`, `FTARule` |
| **Tables owned** | `htsus_chapters`, `htsus_headings`, `cross_rulings`, `section_301_lists`, `section_232_scopes`, `ieepa_rates`, `adcvd_orders`, `tax_rates`, `fee_schedules`, `exchange_rates`, `tax_regime_templates`, `fta_rules` |
| **API routes** | `classify.py` (AI classification), `description_quality.py` (description scoring), `tariff.py` (duty calculation), `fta.py` (FTA eligibility), `trade_lanes.py` (corridor comparison), `jurisdiction.py` (regulatory requirements) |
| **UI surfaces** | Buyer CheckoutPrice/DutiesBadge, Shipper DutyBreakdownCard, Platform Product Analysis/TradeLanes |
| **Engines** | `e1_classification` (HS code + confidence), `e2_tariff/engine` (duty orchestrator), `e2_tariff/regimes/*` (US/EU/UK/IN/BR/CN), `e4_fta` (FTA eligibility), `e0_preclearance` (risk assessment) |
| **Actors** | `preclearance` (runs classification validation as part of pre-clearance) |

**Why this is a domain:** Classification and Tariff are computation-heavy, stateless operations. Given a product description, they determine the HS code. Given an HS code, origin, and value, they compute the full duty burden (base rates, Section 301/232, AD/CVD, IEEPA, FTA savings, MPF, HMF). This is pure computation over reference data — no operational state. The 6 jurisdiction-specific tariff regime engines (US, EU, UK, India, Brazil, China) are the platform's core analytical capability.

**Event relevance:** Tariff calculations are synchronous (API request/response). No real-time events needed. The `financial` actor uses tariff engines to compute results and publishes `FeesCalculated`. Classification results are cached on the product (`cached_analysis`).

### 2.3 Commercial (Orders & Valuation)

**Business capability:** Managing purchase orders, line-item valuation, document requirements, and the commercial intent that precedes logistics execution.

| Dimension | Details |
|-----------|---------|
| **Core entities** | `Order`, `OrderLineItem`, `Document` (commercial docs: invoice, packing list) |
| **Tables owned** | `orders`, `order_line_items`, `documents` (shared — see note below) |
| **API routes** | `orders.py` (CRUD, analysis, ship_order), `documents.py` (upload/status for order documents) |
| **UI surfaces** | Shipper CreateOrder/OrderDetail/OrderList, Platform PlatformOrders/PlatformOrderDetail |
| **Orchestration** | `pipeline.py` (classification → tariff → compliance pipeline), `parallel.py` (parallel stage execution) |
| **Actors** | `shipper` actor creates orders and shipments (simulated) |

**Why this is a domain:** Orders represent commercial intent — "I want to ship these goods from A to B." They have a short, simple lifecycle (`draft → confirmed → shipped → delivered`), owned by the shipper. An order is NOT a shipment — it can exist before any shipment (draft orders awaiting docs) and a shipment can exist without an order (simulation-generated). Different stakeholders: shippers care about orders, brokers care about shipments.

**Handoff to Transport:** `POST /orders/{id}/ship` creates a Shipment from an Order — this is the clear boundary. After that, the order tracks shipment progress via `shipment_id` FK, but the order's commercial data (line items, valuation, documents) stays in this context.

**Cross-context note on `documents` table:** The `documents` table has FKs to both `orders` (commercial docs: invoice, packing list) and `shipments` (clearance docs: BOL, customs forms). Ownership is split: Commercial creates order-level documents, while the `documents` actor in Broker Operations generates and manages shipment-level clearance documents. Both contexts write to the same table — this is a deliberate shared-kernel pattern, not an ownership conflict.

### 2.4 Transport & Logistics (Shipments, Routing & Consolidation)

**Business capability:** Physical movement of goods — shipment creation, carrier assignment, multi-modal routing, consolidation, intermediate transit events, waypoint tracking, and delivery.

| Dimension | Details |
|-----------|---------|
| **Core entities** | `Shipment` (core lifecycle entity), `Consolidation`, `RouteLeg`, `RegulatoryTouchpoint` |
| **Tables owned** | `shipments` (lifecycle fields: status, origin, destination, carrier, transport_mode, waypoints, tracking_number, consolidation_id, routing, intermediate_events), `consolidations` |
| **API routes** | `shipments.py` (list/detail/events), `consolidations.py` (list/detail/siblings), `trade_lanes.py` (corridor comparison) |
| **UI surfaces** | Shipper ShipmentHistory/ShipmentDetail, Platform Active Shipments/Trade Lanes/ControlTower |
| **Actors** | `shipper` (creates), `preclearance` (pre-arrival processing), `carrier` (transit/delivery — **most complex actor**), `consolidator` (grouping), `terminal` (port ops), `disruption` (supply chain events), `isf` (ISF filing for ocean) |

**Why this is a domain:** Transport & Logistics owns physical movement. A shipment's core lifecycle (booked → in_transit → at_customs → cleared → delivered) is independent of how customs processes it. The carrier doesn't care about HS codes or duty rates — it cares about transit time, multi-leg routing, intermediate regulatory touchpoints, and delivery confirmation. Consolidation is a purely logistical concept (grouping goods for efficient transport).

**Mode-specific richness:** The carrier produces **mode-specific operational events** — air freight has HAWB/MAWB/ULD/flight references (13 event types), ocean has HBL/MBL/container/vessel/voyage (13 types), ground has Pro#/BOL/PAPS (9 types). This mode-polymorphic event vocabulary is a platform differentiator. See Section 3.4a for the full mode-specific event catalog.

**Event production:** This context is the **primary event producer** — `ShipmentCreated`, `ShipmentInTransit`, `ShipmentArrivedAtCustoms`, `ShipmentCleared`, `ShipmentDelivered`, all intermediate transit events (12 types), consolidation lifecycle events (5 types), and mode-specific operational events (35+ types).

**The god object problem:** The `shipments` table currently holds 8 JSONB columns from 6+ domains (`classification_result`, `compliance_result`, `tariff_result`, `fta_result`, `events`, `intermediate_events`, `routing`, `documents`). The event-driven architecture naturally decomposes this — each domain publishes events and maintains its own state rather than writing to shared JSONB columns. Transport & Logistics owns `routing`, `intermediate_events`, `waypoints`; other JSONB columns are written by other contexts (Compliance writes `compliance_result`, Trade Intelligence writes `classification_result`/`tariff_result`/`fta_result`).

### 2.5 Customs & Declarations

**Business capability:** Government-facing customs processing — customs screening, entry routing, authority interaction (CBP/PGA), examination, jurisdiction-specific clearance workflows, and holds.

| Dimension | Details |
|-----------|---------|
| **Core entities** | `EntryFiling`, shipment customs fields (`hold_type`, `entry_number`, `cage_status`) |
| **Tables owned** | `entry_filings` |
| **API routes** | `exception_status.py` (exception status tracking), `jurisdiction.py` (jurisdiction-specific rules) |
| **UI surfaces** | Platform Exception Resolution |
| **Actors** | `customs` (screening/routing), `pga` (partner agency review), `cbp_authority` (entry decisions), `cage` (physical cargo management — customs exam facility) |

**Why this is a domain:** Customs processing has 5 jurisdiction-specific state machines (US 12 states, EU 10, BR 8, IN 9, CN 9) each reflecting real customs authority workflows. It has its own entities (entry filings with checklist state, authority responses), its own regulatory framework, and its own stakeholders (CBP, PGA agencies, foreign customs authorities). This is fundamentally different from Logistics — a shipment can be physically delivered but customs-unresolved.

**Event consumption:** Triggered by `DeclarationSubmitted` from Broker Operations (normal path) or `ShipmentArrivedWithoutClearance` from Transport (exception path). The customs authority is a **reactive responder** — it adjudicates broker-filed declarations, it does not proactively seek work. Produces all jurisdiction-specific customs events (Section 3.3): US entry lifecycle (CF28/29, exam, protest), EU UCC lifecycle (lodged/accepted/under_control), BR Siscomex (registered/parameterized), IN ICEGATE (filed/assessed/out_of_charge), CN Single Window (declared/inspected/released).

**Separation from Broker Operations:** The capability map identified that Broker Operations (30+ endpoints, richest UI, own lifecycle) deserves first-class status. Customs & Declarations focuses on the **government-facing** regulatory processing — what CBP/PGA/foreign authorities do. Broker Operations (Section 2.6) focuses on the **broker-facing** filing workflow — what the broker does. The interaction is via events: Customs produces `CF28Issued`, Broker consumes it and produces `CF28Responded`.

### 2.6 Broker Operations

**Business capability:** Customs brokers manage the clearance process — claim shipments, verify documentation, prepare and submit entry filings, respond to CBP requests (CF-28/CF-29), communicate with shippers, and track broker-specific financials.

| Dimension | Details |
|-----------|---------|
| **Core entities** | `BrokerAssignment`, `EntryFiling` (shared with Customs), `BrokerMessage`, broker-specific `checklist_state` |
| **Tables owned** | `brokers`, `broker_assignments`, `broker_messages` (plus shared write access to `entry_filings`) |
| **API routes** | `broker.py` (**30+ endpoints**: queue management, claim/release, checklist verification, entry submission, CF-28/CF-29 responses, broker insights (LLM), fee calculation, communication, document management) |
| **UI surfaces** | Broker Dashboard, Broker Queue, EntryDetail, CBPResponses, Communications, BrokerDocumentChecklist, RegulatoryIntelligence |
| **Services** | `broker_intelligence` (LLM-powered: classification suggestions, document analysis, risk insights) |
| **Actors** | `broker_sim` (simulates full broker workflow: claim → verify → prepare → submit → respond — **most complex actor after carrier**) |

**Why this is a core domain:** Broker Operations is the **most user-facing** domain — it has the richest UI (7 screens), the most API endpoints (30+), and one of the most complex actors. Brokers have their own state (assignments, filings, messages, checklist items), their own lifecycle (claim → verify → prepare → submit → respond), and their own users. The evidence is overwhelming: this is a parallel core domain, not a sub-domain of Customs.

**Interaction with Customs:** Broker Operations *consumes* events from Customs (e.g., `CF28Issued`, `ExamScheduled`) and *produces* events that Customs consumes (e.g., `EntryFiled`, `CF28Responded`, `CF29Accepted`). The broker prepares and submits; customs authorities review and decide. This is a natural bounded context interaction — not the same context.

**Event patterns:** `BrokerAssigned`, `EntryFiled`, `CF28Responded`, `CF29Accepted`, `ProtestFiled`, `CommunicationSent`, `ChecklistItemVerified`. The broker_sim actor decomposition (Section 5.8) splits into BrokerClaimHandler, BrokerProgressHandler, and BrokerCF28Handler.

### 2.7 Compliance & Screening

**Business capability:** Screen shipments, entities, and products against regulatory requirements — denied party lists (DPS, SDN, Entity List), UFLPA, PGA agency requirements, restricted materials.

| Dimension | Details |
|-----------|---------|
| **Core entities** | `RestrictedParty`, `PGAMapping` |
| **Tables owned** | `restricted_parties`, `pga_mappings` |
| **API routes** | `screening.py` (entity screening), `compliance.py` (compliance check), `analysis.py` (AI analysis) |
| **UI surfaces** | Platform Entity Screening, Platform Compliance Dashboard |
| **Engines** | `e3_compliance/engine` (orchestrator), `e3_compliance/dps` (denied party), `e3_compliance/uflpa` (UFLPA), `e3_compliance/pga` (PGA requirements), `e3_compliance/fuzzy_match` (fuzzy matching) |
| **Actors** | `compliance_engine` (generates analysis), `pga` (agency review) |
| **Background services** | `AnalysisWatchdog` |

**Why this is a domain:** Compliance is a **cross-cutting concern** — it's invoked during order analysis, pre-clearance, broker review, and regulatory signal changes. It operates on a fundamentally different data model (restricted party lists, PGA mappings) and serves different stakeholders (compliance officers, regulators). A compliance check is performed AGAINST a shipment but isn't PART of the shipment's lifecycle.

**Event consumption:** Triggered by `ShipmentCreated`, `ShipmentArrivedAtCustoms`, and `ICS2Filed`/`ACASFiled` (transit screening). Produces `AnalysisGenerated`, `PGAReviewComplete`, `ISFValidated`.

### 2.8 Exception Management

**Business capability:** Handle exceptions, holds, and problems that arise during clearance — from tariff disputes to documentation deficiencies to supply chain disruptions. Includes hold resolution and shipper notification/response.

| Dimension | Details |
|-----------|---------|
| **Core entities** | Exception events in `shipment.events` JSONB, resolution state |
| **Tables owned** | None exclusively (operates on shipment event data and broker messages) |
| **API routes** | `resolution.py` (hold resolution: lookup, upload-document, suggest — LLM, rate-limited), `exception.py` (exception analysis — LLM), `exception_status.py` (status tracking) |
| **UI surfaces** | Shipper ResolutionCenter, Platform ExceptionResolution |
| **Engines** | `e5_exception` (exception analysis and resolution suggestion — LLM-powered) |
| **Actors** | `exceptions` (generates exceptions — random + rule-based), `resolution` (processes hold resolutions — time-based + document-based), `shipper_response` (simulates shipper responding to hold notifications), `disruption` (supply chain disruptions), `terminal` (terminal-related delays) |

**Why this is a domain:** Exception Management is reactive and cross-cutting — 5 actors that independently handle holds, delays, disruptions, and resolutions. It has its own lifecycle (exception raised → investigation → resolution/escalation), its own UI surfaces (ResolutionCenter, ExceptionResolution), and its own LLM-powered analysis engine. The current implementation has these 5 actors all independently scanning and mutating shipments — the event-driven model turns them into event consumers reacting to `ShipmentHeld`, `TransitHeld`, `CageIntake`, and `DisruptionGenerated`.

**Event patterns:** Consumes `ShipmentHeld`, `TransitHeld`, `CageIntake`. Produces `HoldResolved`, `TransitHoldResolved`, `HoldEscalated`, `ExceptionInjected`, `DisruptionGenerated`, `BerthDelayReported`.

### 2.9 Financial Settlement

**Business capability:** Track all financial obligations — duties, tariffs, fees, bonds, demurrage, storage costs, and cage dwell charges. Produce a unified financial picture for each shipment.

| Dimension | Details |
|-----------|---------|
| **Core entities** | `shipment.financials` JSONB, `shipment.cage_status` JSONB (storage costs), fee/rate reference data |
| **Tables owned** | Operates on `shipments.financials` and `shipments.cage_status` (cross-context writes to Transport entity) |
| **API routes** | `broker.py/calculate_fees`, shipment detail financial breakdowns |
| **UI surfaces** | Shipper ShipmentFinancials, Buyer DutiesBadge/CheckoutPrice, Broker fee calculations |
| **Actors** | `financial` (duty/fee computation via tariff engines), `demurrage` (D&D charges — lazy computation), `cage` (storage cost accrual — lazy computation) |

**Why this is a domain:** Financial settlement has a **clear trigger point**: it happens after clearance (`ShipmentCleared` → finalize duties), during delays (`ShipmentHeld` → start accruing demurrage), and during cage dwell (`CageIntake` → storage costs). It aggregates costs from multiple sources into a unified financial picture. The buyer cares about total landed cost, the shipper about duty liability, the broker about fee entitlement.

**Event relevance:** Consumes `ShipmentCleared`, `ShipmentDelivered`, `CageReleased`, `DutyAssessed` (IN). Produces `FeesCalculated`. Demurrage and cage use lazy computation (Section 11.3) — charges computed on demand, not accrued per-tick.

### 2.10 Regulatory Intelligence

**Business capability:** Monitor and disseminate regulatory changes — tariff modifications, new trade restrictions, policy changes — that affect clearance operations.

| Dimension | Details |
|-----------|---------|
| **Core entities** | `RegulatorySignal` |
| **Tables owned** | `regulatory_signals` |
| **API routes** | `regulatory.py` (signals CRUD, scenario analysis, refresh, monitor status) |
| **UI surfaces** | Platform Regulatory Intel, Broker RegulatoryIntelligence |
| **Engines** | `e6_regulatory/engine` (signal analysis), `e6_regulatory/db` (persistence) |
| **Services** | `feed_parsers` (parse Federal Register, CBP bulletins, etc.) |
| **Background services** | `RegulatoryMonitor` (periodic scanning for regulatory changes) |
| **Actors** | None (runs as background service, not a simulation actor) |

**Why this is a domain:** Regulatory Intelligence is an **autonomous supporting service** — it has no dependency on the shipment state machine. It runs on its own schedule (configurable monitoring interval), produces its own data (regulatory signals), and serves its own UI surfaces. Other domains subscribe to `RegulatorySignalDetected` events to update their behavior (e.g., tariff engine updates rates, compliance engine updates screening lists).

### 2.11 Context Map — How Domains Interact

```
Product Catalog ──description──> Trade Intelligence ──HS/tariff──> Financial Settlement
      │                               │                                    ▲
      │ product_id                     │ classification, tariff calc       │ duty/fee results
      ▼                                ▼                                   │
Commercial ──ship_order──> Transport & ──arrived──> Customs & ──entry events──> Broker
 (Orders)                  Logistics       │        Declarations      │        Operations
                              │            │             │            │            │
                              │ transit     │ hold        │ CF28/29   │ filing     │
                              ▼ events      ▼             ▼           ▼            ▼
                        Intermediate   Exception    Compliance    CBP/PGA    EntryFiled
                        Transit Events Management   & Screening   decisions  CF28Responded
                              │                          │
                              │ ICS2/ACAS               │ analysis results
                              ▼                          ▼
                        Regulatory Intelligence ──signals──> All subscribing contexts
```

**Key interaction points:**
1. **Commercial → Transport:** `ship_order` creates a Shipment from an Order (synchronous API call)
2. **Broker → Customs:** `DeclarationSubmitted` event triggers customs adjudication (proactive filing during transit)
3. **Transport → Customs:** `ShipmentArrivedWithoutClearance` event triggers exception hold (if no pre-filed declaration)
4. **Customs → Broker:** `CF28Issued`, `ExamScheduled`, `DeclarationRejected` events trigger broker action
5. **Broker → Customs:** `EntryFiled`, `CF28Responded` events return to customs authority
6. **Customs → Exception:** `ShipmentHeld` event triggers exception resolution workflow
7. **Customs → Financial:** `ShipmentCleared` event triggers duty finalization
8. **Transport → Exception:** `TransitHeld` event triggers transit hold resolution
9. **Transport → Compliance:** `ICS2Filed`, `ACASFiled` events trigger transit screening
10. **Regulatory Intelligence → All:** `RegulatorySignalDetected` events inform rate/rule updates
11. **Product → Trade Intelligence:** Product descriptions feed classification engine

### 2.12 Actor-to-Context Mapping

| Context | Actors | Read Stores | API Surfaces |
|---------|--------|-------------|-------------|
| Product Catalog | — | — | products |
| Trade Intelligence | preclearance | — | classify, description_quality, tariff, fta, trade_lanes, jurisdiction |
| Commercial | shipper (simulated) | — | orders, documents |
| Transport & Logistics | shipper, carrier, consolidator, preclearance, terminal, disruption, isf | Shipment List, Shipment Detail | shipments, consolidations |
| Customs & Declarations | customs, pga, cbp_authority, cage | — | exception_status, jurisdiction |
| Broker Operations | broker_sim, documents | Broker Queue | broker (30+ endpoints), documents |
| Compliance & Screening | compliance_engine, pga | — | screening, compliance, analysis |
| Exception Management | exceptions, resolution, shipper_response, disruption, terminal | — | resolution, exception, exception_status |
| Financial Settlement | financial, demurrage, cage | Dashboard financials | (embedded in broker + shipment detail) |
| Regulatory Intelligence | — (background service) | — | regulatory |
| **Cross-cutting projections** | — | Dashboard (all contexts) | platform, dashboard, simulation, chat, session |

**Note:** Some actors operate across context boundaries. The `pga` actor appears in both Customs and Compliance because PGA reviews are a compliance check that feeds into the customs decision. The `cage` actor appears in both Customs (physical cargo management) and Financial (storage cost tracking). The `disruption` and `terminal` actors appear in both Transport and Exception Management. This cross-context participation is natural — it would become explicit service-to-service calls in a microservices architecture.

---

## 3. Event Catalog

### 3.1 Event Schema

All events follow this envelope schema:

```json
{
  "event_id": "uuid-v4",
  "event_type": "ShipmentStatusChanged",
  "sim_timestamp": "2026-02-08T14:30:00Z",
  "wall_timestamp": "2026-02-08T02:30:00Z",
  "actor": "customs",
  "jurisdiction": "US",
  "version": 1,
  "correlation_id": "shipment-uuid",
  "payload": {
    // Full state snapshot of the affected entity
  }
}
```

**Key design decisions:**
- **`sim_timestamp`**: The simulation clock time when the event occurred. Used for business logic ordering.
- **`wall_timestamp`**: Real wall-clock time. Used for operational monitoring and debugging.
- **`jurisdiction`**: The regulatory jurisdiction governing this event (e.g., `"US"`, `"EU"`, `"UK"`, `"BR"`, `"IN"`, `"CN"`). Derived from destination country code via `get_jurisdiction()` (see `services/jurisdiction.py`). Required for all customs processing events. Optional for logistics-only events (use `null`). Enables jurisdiction-aware consumers to filter events. Note: UK uses EU transition graph but carries `jurisdiction: "UK"` — see Section 3.3.6 principle #6.
- **Fat-reference payload** (state-transfer pattern): Payloads carry sufficient state for consumers to process without DB queries, but exclude large JSONB fields (`events` history, `analysis`) that would bloat event sizes. See Section 3.9 for the payload schema.
- **`correlation_id`**: Always the shipment ID. Enables end-to-end tracing of a shipment's journey through all events.

### 3.2 Shipment Lifecycle Events

> **Proactive evaluation principle (Section 5.5):** Events are **information signals**, not step commands. A service subscribes to every event that could satisfy one of its readiness prerequisites. Receiving `ShipmentCreated` doesn't mean "start customs processing" — it means "new data is available; re-evaluate if I have everything I need." Multiple events may contribute to the same service's readiness: broker assignment needs product, origin, and value — all present in `ShipmentCreated`, but a later `ProductUpdated` or `ClassificationCompleted` event could fill a gap.

| Event Type | Producer | Payload (state snapshot) | Consumers |
|---|---|---|---|
| `ShipmentCreated` | shipper | Full shipment record | compliance, preclearance, documents, isf, **broker_sim** (readiness eval — may begin prep before arrival), **financial** (readiness eval — calculates fees when tariff data available), exceptions, dashboard-projector |
| `ClassificationCompleted` | preclearance, compliance | Shipment + HS code + risk flags | **broker** (readiness eval — pre-files declaration when prerequisites met), **pga** (readiness eval — PGA determination requires only HS code), **financial** (readiness eval), **documents** (readiness eval — doc requirements depend on HS code), dashboard-projector |
| `ShipmentInTransit` | preclearance, carrier | Shipment + carrier, waypoints, references | carrier (transit monitoring), consolidator (grouping), exceptions, disruption, dashboard-projector |
| `ShipmentArrivedAtCustoms` | carrier | Shipment + entry references, consolidation refs | **broker** (final readiness check — if declaration not yet filed, files now), customs_authority (reactive — adjudicates `DeclarationSubmitted` or holds if no declaration filed), **cage** (intake trigger), **terminal**, exceptions, dashboard-projector |
| `ShipmentCleared` | customs_authority, cbp_authority, resolution | Shipment + clearance path (stp/manual/resolution) | carrier (for delivery), cage (for release), financial, dashboard-projector |
| `ShipmentDelivered` | carrier | Shipment + delivery timestamp | financial, dashboard-projector |

**Key shift from the previous design:** **Proactive evaluators** like `broker` (declaration filing), `pga`, `financial`, and `documents` now subscribe to **earlier** events (`ShipmentCreated`, `ClassificationCompleted`) instead of waiting for `ShipmentArrivedAtCustoms`. They evaluate their readiness criteria on each event and act at the earliest possible moment. **Reactive responders** like `customs_authority` and `cbp_authority` subscribe to command events (`DeclarationSubmitted`, `ShipmentArrivedWithoutClearance`) and adjudicate what they receive. See Section 5.5 for the readiness accumulator pattern and Section 5.3 for the proactive/reactive classification.

### 3.3 Customs Processing Events — Jurisdiction-Specific

The platform supports 5 jurisdiction-specific customs state machines, each with unique intermediate states that reflect how that country's customs authority actually processes imports. **Every jurisdiction-specific state transition is a first-class event** — the event catalog does NOT abstract these into generic events.

**Broker ↔ Customs Authority interaction events** (cross-jurisdiction):

These events model the separation between the **broker** (proactive evaluator — files declarations) and the **customs authority** (reactive responder — adjudicates submissions). This distinction is fundamental to the architecture (see Section 5.2, 5.3).

| Event Type | Producer | Payload | Consumers |
|---|---|---|---|
| `DeclarationSubmitted` | broker (declaration handler) | Shipment + declaration data + filing_time | customs_authority (adjudicates), dashboard-projector |
| `DeclarationAccepted` | customs_authority | Shipment + acceptance details + clearance path | broker (confirmation), carrier (pre-clearance flag), cage, financial, dashboard-projector |
| `DeclarationRejected` | customs_authority | Shipment + rejection reason + required corrections | broker (re-file), dashboard-projector |
| `ShipmentArrivedWithoutClearance` | carrier (when no pre-filed declaration accepted) | Shipment + arrival details | customs_authority (holds by default), broker (emergency filing), cage, dashboard-projector |

**Cross-jurisdiction events** (present in ALL jurisdictions):

| Event Type | Producer | Transition | Payload | Consumers |
|---|---|---|---|---|
| `ShipmentHeld` | customs_authority, pga, cbp_authority | any → held | Shipment + hold_type + reason + jurisdiction | resolution, cage, broker_sim, shipper_response, dashboard-projector |
| `HoldResolved` | resolution | held → cleared/at_customs/in_transit | Shipment + resolution outcome + jurisdiction | carrier, cage, dashboard-projector |
| `BrokerAssigned` | broker_sim | (assignment, no status change) | Assignment + broker details + jurisdiction | dashboard-projector |
| `CommunicationSent` | customs_authority, cbp_authority, broker_sim | (no status change) | Communication details + shipment_id + response_required | shipper_response, dashboard-projector |

#### 3.3.1 US — CBP Entry Lifecycle (11 states)

Maps to `state_machines/us.py`. The US has the richest customs lifecycle: entry filing → CF-28/CF-29 information requests → examination → protest → clearance.

| Event Type | Producer | Transition | Payload | Consumers |
|---|---|---|---|---|
| `ShipmentInspection` | customs | at_customs → inspection | Shipment + inspection type | cage (exam scheduling), dashboard-projector |
| `InspectionCompleted` | customs | inspection → cleared/held | Shipment + outcome | carrier/resolution, cage, dashboard-projector |
| `EntryFiled` | broker_sim | at_customs → entry_filed | Shipment + EntryFiling snapshot | cbp_authority, dashboard-projector |
| `CF28Issued` | cbp_authority | entry_filed → cf28_pending | EntryFiling + CF28 details (information request) | broker_sim, shipper_response, dashboard-projector |
| `CF28Responded` | broker_sim | cf28_pending → entry_filed | EntryFiling + response | cbp_authority, dashboard-projector |
| `CF29Issued` | cbp_authority | entry_filed → cf29_pending | EntryFiling + CF29 details (proposed action) | broker_sim, dashboard-projector |
| `CF29Accepted` | broker_sim | cf29_pending → cleared | EntryFiling + acceptance | carrier, cage, dashboard-projector |
| `ProtestFiled` | broker_sim | cf29_pending → protest_filed | EntryFiling + protest details (19 USC 1514) | cbp_authority, dashboard-projector |
| `ProtestResolved` | cbp_authority | protest_filed → cleared/held | EntryFiling + resolution | carrier/resolution, cage, dashboard-projector |
| `ExamScheduled` | cbp_authority | entry_filed → exam_scheduled | EntryFiling + exam type (VACIS/tailgate/intensive) | cage, dashboard-projector |
| `EntryAccepted` | cbp_authority | entry_filed/exam_scheduled → cleared | EntryFiling + release info | carrier, cage, dashboard-projector |

#### 3.3.2 EU — UCC Customs Lifecycle (8 states)

Maps to `state_machines/eu.py`. EU customs follows the Union Customs Code (UCC): declaration lodged → accepted → under control (physical/documentary check) → released → cleared.

| Event Type | Producer | Transition | Payload | Consumers |
|---|---|---|---|---|
| `DeclarationLodged` | broker | at_customs → lodged | Shipment + declaration reference (MRN) | customs, dashboard-projector |
| `DeclarationAccepted` | customs | lodged → accepted | Shipment + acceptance details | broker_sim, dashboard-projector |
| `DeclarationRejected` | customs | lodged → rejected | Shipment + rejection reason + code | broker_sim, dashboard-projector |
| `UnderControl` | customs | accepted → under_control | Shipment + control type (documentary/physical) | cage, broker_sim, dashboard-projector |
| `CustomsReleased` | customs | under_control/accepted → released | Shipment + release reference | carrier, cage, dashboard-projector |
| `ReleasedToCleared` | customs | released → cleared | Shipment + final clearance | carrier, dashboard-projector |

**EU-specific metadata in payload:** `mrn` (Movement Reference Number), `customs_office_code`, `control_type` (documentary, physical, sample), `t1_transit_doc` (for transit movements).

#### 3.3.3 BR — Siscomex Lifecycle (7 states)

Maps to `state_machines/br.py`. Brazilian customs uses the Siscomex system with parametrization channels: green (auto-release), yellow (documentary check), red (physical inspection), grey (special investigation).

| Event Type | Producer | Transition | Payload | Consumers |
|---|---|---|---|---|
| `DIRegistered` | broker | at_customs → registered | Shipment + DI number (Declaração de Importação) | customs, dashboard-projector |
| `Parameterized` | customs | registered → parameterized | Shipment + channel (green/yellow/red/grey) + implications | broker_sim, cage, dashboard-projector |
| `ParameterizationCleared` | customs | parameterized → cleared | Shipment + channel resolution | carrier, cage, dashboard-projector |
| `DIRejected` | customs | registered → rejected | Shipment + rejection details | broker_sim, dashboard-projector |

**BR-specific metadata in payload:** `di_number` (DI registration), `parametrization_channel` (green/yellow/red/grey), `siscomex_reference`, `lpco_required` (import license needed).

#### 3.3.4 IN — ICEGATE Lifecycle (8 states)

Maps to `state_machines/in_.py`. Indian customs uses ICEGATE with a Bill of Entry lifecycle: filing → duty assessment (first check / second check) → out of charge → cleared.

| Event Type | Producer | Transition | Payload | Consumers |
|---|---|---|---|---|
| `BillOfEntryFiled` | broker | at_customs → filed | Shipment + BOE number + ICEGATE reference | customs, dashboard-projector |
| `DutyAssessed` | customs | filed → assessed | Shipment + duty assessment + first/second check result | broker_sim, financial, dashboard-projector |
| `OutOfCharge` | customs | assessed → out_of_charge | Shipment + OOC order + examination details | carrier, cage, dashboard-projector |
| `OOCToCleared` | customs | out_of_charge → cleared | Shipment + final clearance | carrier, dashboard-projector |
| `BOERejected` | customs | filed → rejected | Shipment + rejection reason | broker_sim, dashboard-projector |

**IN-specific metadata in payload:** `boe_number` (Bill of Entry), `icegate_reference`, `appraisal_type` (first_check/second_check), `duty_amount`, `cess_amount`, `igst_amount`.

#### 3.3.5 CN — GACC Single Window Lifecycle (8 states)

Maps to `state_machines/cn.py`. Chinese customs uses the GACC Single Window with H2010 risk assessment: declaration → CIQ inspection (quarantine) → released → cleared.

| Event Type | Producer | Transition | Payload | Consumers |
|---|---|---|---|---|
| `CustomsDeclared` | broker | at_customs → declared | Shipment + declaration ID + Single Window reference | customs, dashboard-projector |
| `CIQInspected` | customs | declared → inspected | Shipment + CIQ inspection type + quarantine status | cage, broker_sim, dashboard-projector |
| `GACCReleased` | customs | declared/inspected → released | Shipment + release reference + duty payment status | carrier, cage, dashboard-projector |
| `ReleasedToCleared_CN` | customs | released → cleared | Shipment + final clearance | carrier, dashboard-projector |
| `DeclarationRejected_CN` | customs | declared → rejected | Shipment + rejection reason | broker_sim, dashboard-projector |

**CN-specific metadata in payload:** `declaration_id`, `single_window_reference`, `h2010_risk_level`, `ciq_inspection_type` (quarantine/quality/weight), `duty_payment_reference`.

#### 3.3.6 Jurisdiction Event Design Principles

1. **No generic `CustomsProcessed` event** — each jurisdiction's intermediate states are first-class events with domain-specific names.
2. **Jurisdiction field in envelope** — all customs events carry `jurisdiction` in the event envelope, enabling consumers to filter by jurisdiction.
3. **Jurisdiction-specific metadata** — each jurisdiction carries its own metadata fields (MRN for EU, DI number for BR, BOE number for IN, declaration ID for CN).
4. **Common status events remain shared** — `ShipmentHeld`, `HoldResolved`, `ShipmentCleared`, `ShipmentDelivered` are cross-jurisdiction because the business semantics are identical (held is held, cleared is cleared, regardless of jurisdiction).
5. **Consumer routing** — jurisdiction-aware consumers subscribe to the `customs-processing` stream and filter by `jurisdiction` field. Jurisdiction-agnostic consumers (like `dashboard-projector`) process all events.
6. **UK jurisdiction handling** — The UK uses the same transition graph as the EU (`EU_TRANSITIONS` from `state_machines/eu.py`) but carries `jurisdiction: "UK"` in the event envelope. This is derived from `JURISDICTION_MAP["GB"] = "UK"` in `services/jurisdiction.py`. UK-specific tariff regimes (post-Brexit UKGT rates) are handled by the Trade Intelligence context, not by a separate UK state machine. Consumers that need UK-specific behavior filter on `jurisdiction == "UK"`.

### 3.4 Intermediate Transit Events

These events model the regulatory touchpoints that occur while a shipment is in transit — between departure and arrival at the destination. They are generated by the routing engine based on the shipment's waypoints, regulatory touchpoints, and transport mode. Each event carries `territory` metadata identifying WHERE in the transit journey it occurred.

**Why these matter:** International shipments don't teleport from origin to destination. An air shipment from Shanghai to Chicago might transit through Leipzig (EU regulatory screening, ICS2 filing) and be subject to ACAS pre-arrival screening. An ocean shipment from Mumbai to Rotterdam transships at Singapore (TradeNet filing, port authority inspection). These intermediate events are **the platform's differentiator** — they provide visibility into regulatory processing that happens between origin and destination.

#### Filing Events

| Event Type | Producer | Territory Examples | Payload | Consumers |
|---|---|---|---|---|
| `ExportClearanceObtained` | customs | Origin country | Shipment + export clearance reference | dashboard-projector |
| `ICS2Filed` | platform | EU territories (DE, FR, NL, BE) | Shipment + ICS2/ENS reference + house number | dashboard-projector, compliance |
| `PLACIFiled` | platform | EU territories | Shipment + PLACI pre-loading data reference | dashboard-projector |
| `TradeNetFiled` | platform | SG | Shipment + TradeNet declaration reference | dashboard-projector |
| `TransitDeclarationFiled` | platform | Any transit territory | Shipment + transit declaration reference | dashboard-projector |
| `ACASFiled` | platform | US (pre-arrival) | Shipment + ACAS reference + house number | dashboard-projector, compliance |

#### Screening & Clearance Events

| Event Type | Producer | Territory Examples | Payload | Consumers |
|---|---|---|---|---|
| `TransitScreening` | customs | EU, SG (sanctions/security) | Shipment + screening type + territory | dashboard-projector, compliance |
| `TransitHeld` | customs | Any transit territory | Shipment + territory + hold reason (sanctions, dual-use, port inspection) + `triggers_hold=true` | resolution, dashboard-projector |
| `TransitCleared` | customs | Any transit territory | Shipment + territory + clearance reference | dashboard-projector |
| `TransitHoldResolved` | resolution | Any transit territory | Shipment + territory + outcome (resolved/escalated) | carrier, dashboard-projector |

#### Hub & Transshipment Events

| Event Type | Producer | Transport Mode | Payload | Consumers |
|---|---|---|---|---|
| `HubSort` | carrier | Air only | Shipment + hub location + carrier name + sort facility | dashboard-projector |
| `TransshipmentArrived` | carrier | Ocean only | Shipment + port + territory + container number | dashboard-projector |
| `TransshipmentDeparted` | carrier | Ocean only | Shipment + port + territory + container number + next destination | dashboard-projector |

**Transit event design principles:**
1. **Territory-aware** — every transit event carries the `territory` where it occurred, distinct from `origin` and `destination`.
2. **Filing ↔ screening ↔ clearance chain** — transit events follow the same filing → screening → clearance/hold pattern as destination customs, but at a lighter weight appropriate for transit processing.
3. **Hold triggers** — `TransitHeld` events set `triggers_hold=true` in the payload, signaling that the shipment status should transition to `held` with transit hold markers (see resolution actor transit hold logic).
4. **Transport-mode gating** — `HubSort` only occurs for air shipments at integrator hubs; `TransshipmentArrived/Departed` only for ocean at transit ports. Consumers can filter by transport mode.
5. **Chronological ordering** — intermediate events are distributed proportionally across the transit window and sorted by timestamp before publishing.

### 3.5 Mode-Specific Transport Events

The platform tracks shipments with **mode-specific event vocabularies** — different transport modes generate different event types with different terminology and document references. This is NOT generic "shipment moved" tracking. Mode-specific events are the operational heartbeat of the Transport & Logistics context.

#### Air Freight Events (13 types)

| Event Type | Producer | Payload References | Consumers |
|---|---|---|---|
| `HAWBIssued` | shipper | house_number (HAWB) | carrier, dashboard-projector |
| `ACASFiled` | platform | house_number, acas_reference | compliance, dashboard-projector |
| `CargoTendered` | carrier | house_number, pieces, weight_kg | dashboard-projector |
| `MAWBConsolidated` | carrier | house_number, master_number (MAWB) | dashboard-projector |
| `ULDBuildUp` | carrier | master_number, uld_type, uld_id | dashboard-projector |
| `FlightDeparted` | carrier | master_number, flight_number, origin | dashboard-projector |
| `FlightArrived` | carrier | master_number, flight_number, destination | dashboard-projector |
| `ULDBreakdown` | carrier | master_number, uld_id, terminal | dashboard-projector |
| `AirEntryFiled` | platform | house_number, entry_number | customs, dashboard-projector |
| `AirCustomsCleared` | customs | house_number, duty_assessed | carrier, financial, dashboard-projector |
| `HAWBReleased` | carrier | house_number | dashboard-projector |
| `AirDelivered` | carrier | house_number, pod_signature, pod_time | financial, dashboard-projector |

#### Ocean Freight Events (13 types)

| Event Type | Producer | Payload References | Consumers |
|---|---|---|---|
| `HBLIssued` | shipper | house_number (HBL), booking_reference | carrier, dashboard-projector |
| `ISFFiled` | platform | house_number, isf_reference | compliance, dashboard-projector |
| `CargoReceivedCFS` | carrier | house_number, cfs_location | dashboard-projector |
| `ContainerStuffed` | carrier | house_number, container_number, seal_number, master_number (MBL) | dashboard-projector |
| `VesselDeparted` | carrier | master_number, vessel, voyage, origin_port | dashboard-projector |
| `VesselArrived` | carrier | master_number, vessel, voyage, destination_port | dashboard-projector |
| `ContainerDischarged` | carrier | container_number, terminal | dashboard-projector |
| `OceanEntryFiled` | platform | house_number, entry_number | customs, dashboard-projector |
| `OceanCustomsCleared` | customs | house_number, duty_assessed | carrier, financial, dashboard-projector |
| `Deconsolidated` | carrier | house_number, container_number, cfs_location | dashboard-projector |
| `HBLReleased` | carrier | house_number | dashboard-projector |
| `OceanDelivered` | carrier | house_number, pod_signature, pod_time | financial, dashboard-projector |

#### Ground/Truck Events (9 types)

| Event Type | Producer | Payload References | Consumers |
|---|---|---|---|
| `BOLIssued` | shipper | pro_number, bol_number | carrier, dashboard-projector |
| `PAPSFiled` | platform | pro_number, paps_reference | compliance, dashboard-projector |
| `CargoPickedUp` | carrier | pro_number, origin_location | dashboard-projector |
| `BorderArrived` | carrier | pro_number, border_crossing | dashboard-projector |
| `GroundEntryFiled` | platform | pro_number, entry_number | customs, dashboard-projector |
| `GroundCustomsCleared` | customs | pro_number, duty_assessed | carrier, financial, dashboard-projector |
| `ProReleased` | carrier | pro_number | dashboard-projector |
| `GroundDelivered` | carrier | pro_number, pod_signature, pod_time | financial, dashboard-projector |

**Mode-specific event design principles:**
1. **Mode-polymorphic document references** — air events carry `house_number`/`master_number`/`flight_number`/`uld_id`; ocean carries `container_number`/`seal_number`/`vessel`/`voyage`; ground carries `pro_number`/`bol_number`. The event schema accommodates this via the `references` dict in `ShipmentEventPayload`.
2. **Not a flat generic schema** — `FlightDeparted` is NOT the same event as `VesselDeparted`. They carry different references, have different timing characteristics, and serve different operational workflows. Mode-specific events preserve this domain richness.
3. **Shared lifecycle events remain generic** — `ShipmentCreated`, `ShipmentInTransit`, `ShipmentArrivedAtCustoms`, `ShipmentCleared`, `ShipmentDelivered` are mode-independent lifecycle events. Mode-specific events are **operational detail within** the lifecycle.
4. **Dashboard projector aggregation** — the dashboard projector normalizes mode-specific events into unified timeline entries for display, but preserves the mode-specific details in the event metadata.

### 3.6 Exception, Cage & Resolution Events

Hold resolution events (`HoldResolved`, `TransitHoldResolved`) are defined in Sections 3.3 (cross-jurisdiction) and 3.4 (transit). This section covers the remaining exception and cage lifecycle events.

| Event Type | Producer | Payload | Consumers |
|---|---|---|---|
| `HoldEscalated` | resolution | Shipment + escalation details + jurisdiction | shipper_response, dashboard-projector |
| `CageIntake` | cage | Shipment + cage_status snapshot (location, intake_time) | demurrage, dashboard-projector |
| `CageExamScheduled` | cage | Shipment + exam type + scheduled time | broker_sim, dashboard-projector |
| `CageReleased` | cage | Shipment + dwell/cost summary (days, total_cost) | financial, dashboard-projector |
| `GODeadlineWarning` | cage | Shipment + days_remaining + deadline date | broker_sim, dashboard-projector |

### 3.7 Compliance & Enrichment Events

| Event Type | Producer | Payload | Consumers |
|---|---|---|---|
| `AnalysisGenerated` | compliance_engine | Shipment summary + full analysis JSONB | dashboard-projector |
| `PGAReviewComplete` | pga | Shipment summary + agency + outcome | dashboard-projector |
| `ISFValidated` | isf | Shipment summary + ISF status | dashboard-projector |
| `FeesCalculated` | financial | Shipment summary + financials breakdown | dashboard-projector |
| `DocumentDiscrepancyFound` | documents | Shipment summary + discrepancy details | dashboard-projector |
| `BerthDelayReported` | terminal | Shipment summary + delay details | dashboard-projector |
| `ExceptionInjected` | exceptions | Shipment summary + exception type + details | dashboard-projector |
| `DisruptionGenerated` | disruption | Port + severity + affected shipment IDs | dashboard-projector |

### 3.8 Consolidation Lifecycle Events

| Event Type | Producer | Transition | Payload | Consumers |
|---|---|---|---|---|
| `ConsolidationClosed` | consolidator | booked -> closed | Full consolidation + member shipments | carrier, dashboard-projector |
| `ConsolidationInTransit` | carrier | closed -> in_transit | Consolidation + carrier details | dashboard-projector |
| `ConsolidationArrived` | carrier | in_transit -> arrived | Consolidation + arrival details | dashboard-projector |
| `ConsolidationDeconsolidating` | carrier | arrived -> deconsolidating | Consolidation + member shipments | dashboard-projector |
| `ConsolidationDeconsolidated` | carrier | deconsolidating -> deconsolidated | Consolidation summary | dashboard-projector |

### 3.9 Event Payload Schema — Fat Reference

Event payloads use a **fat-reference** approach: sufficient state for consumers to process without DB queries, but excluding large JSONB fields that would bloat events.

**Critical design principle for single-writer model:** In the event-sourced architecture, actors work entirely from event payloads — they do NOT read from the database. The fat-reference payload MUST carry enough state for downstream actors to make business decisions. The `ShipmentEventPayload` is the primary data structure flowing through the system.

```python
from app.services.jurisdiction import get_jurisdiction  # Maps "DE"→"EU", "GB"→"UK", etc.

@dataclass
class ShipmentEventPayload:
    """Standard payload for shipment-related events.

    This is the PRIMARY data structure in the event-driven architecture.
    Actors receive this in events and make decisions from it — no DB reads.
    The Shipment Aggregator (Section 5.4) projects events into the DB record.

    Includes all fields consumers need for processing decisions.
    Excludes: events history (large append-only array), analysis (large JSONB blob).
    """
    id: str
    status: str
    previous_status: str | None
    product: str
    origin: str
    destination: str
    carrier: str | None
    transport_mode: str
    declared_value: float
    company_name: str
    hs_code: str | None
    hold_type: str | None
    entry_number: str | None
    consolidation_id: str | None
    cage_status: dict | None          # Included — small, needed by cage/demurrage
    financials: dict | None            # Included — small, needed by financial/dashboard
    references: dict | None            # Included — small, needed by many actors
    codes: dict | None                 # Included — small, needed by customs screening
    waypoints: list[dict] | None       # Included — moderate, needed by carrier
    jurisdiction: str | None             # Destination jurisdiction via get_jurisdiction() (US/EU/UK/BR/IN/CN/JP/KR/AU/CA/MX)
    territory: str | None               # Transit territory (for intermediate events)
    created_at: str                    # ISO timestamp
    updated_at: str                    # ISO timestamp

    # NOT included:
    # - events: list[dict]            # Timeline lives in event stream; JSONB column is a projection
    # - analysis: dict                # Large LLM-generated blob, emitted as AnalysisGenerated event
    # - cargo_detail: dict            # Large, only needed by demurrage (can fetch on demand)

    @classmethod
    def from_shipment(cls, shipment: Shipment) -> "ShipmentEventPayload":
        """Create payload from ORM Shipment object.

        Used at event ORIGIN POINTS — where an API endpoint or the shipper actor
        creates the initial event from a new/existing shipment ORM object.
        After this point, downstream actors work from event payloads, not ORM objects.
        """
        return cls(
            id=str(shipment.id),
            status=shipment.status,
            previous_status=None,  # Set by caller
            product=shipment.product,
            origin=shipment.origin,
            destination=shipment.destination,
            carrier=shipment.carrier,
            transport_mode=shipment.transport_mode,
            declared_value=float(shipment.declared_value or 0),
            company_name=shipment.company_name or "",
            hs_code=shipment.hs_code,
            hold_type=shipment.hold_type,
            entry_number=shipment.entry_number,
            consolidation_id=str(shipment.consolidation_id) if shipment.consolidation_id else None,
            cage_status=dict(shipment.cage_status) if shipment.cage_status else None,
            financials=dict(shipment.financials) if shipment.financials else None,
            references=dict(shipment.references) if shipment.references else None,
            codes=dict(shipment.codes) if shipment.codes else None,
            waypoints=list(shipment.waypoints) if shipment.waypoints else None,
            jurisdiction=get_jurisdiction(shipment.destination),  # Maps ISO-2 country code → jurisdiction key
            territory=None,  # Set by caller for transit events
            created_at=shipment.created_at.isoformat(),
            updated_at=shipment.updated_at.isoformat() if shipment.updated_at else shipment.created_at.isoformat(),
        )

    @classmethod
    def from_event(cls, event_payload: dict) -> "ShipmentEventPayload":
        """Create payload from an upstream event's payload dict.

        Used by downstream actors that receive events and need to emit
        new events with the same shipment context. This is the PRIMARY
        construction method in the event-driven architecture — actors
        receive event dicts, not ORM objects.
        """
        return cls(
            id=event_payload["id"],
            status=event_payload["status"],
            previous_status=event_payload.get("previous_status"),
            product=event_payload["product"],
            origin=event_payload["origin"],
            destination=event_payload["destination"],
            carrier=event_payload.get("carrier"),
            transport_mode=event_payload.get("transport_mode", ""),
            declared_value=float(event_payload.get("declared_value", 0)),
            company_name=event_payload.get("company_name", ""),
            hs_code=event_payload.get("hs_code"),
            hold_type=event_payload.get("hold_type"),
            entry_number=event_payload.get("entry_number"),
            consolidation_id=event_payload.get("consolidation_id"),
            cage_status=event_payload.get("cage_status"),
            financials=event_payload.get("financials"),
            references=event_payload.get("references"),
            codes=event_payload.get("codes"),
            waypoints=event_payload.get("waypoints"),
            jurisdiction=event_payload.get("jurisdiction"),
            territory=event_payload.get("territory"),
            created_at=event_payload.get("created_at", ""),
            updated_at=event_payload.get("updated_at", ""),
        )
```

**Two construction paths:**
1. **`from_shipment()`** — Used at **event origin points**: API endpoints that create shipments, the `shipper` actor that generates simulation shipments. These are the only places where ORM objects are available.
2. **`from_event()`** — Used by **all downstream actors**: customs, pga, broker_sim, carrier, etc. They receive an event payload dict and construct a `ShipmentEventPayload` from it to carry forward into their own emitted events.

**Size estimate:** ~1-2 KB per event (vs 5-20 KB with full events history + analysis). At peak volume (~200 shipments/hour), this is ~400 KB/hour in Redis Streams — negligible.

**When a consumer needs excluded fields:** Rare, but possible. The consumer can fetch the shipment by primary key (`session.get(Shipment, id)`) — a single-row indexed read, ~1ms. The Aggregator keeps the PostgreSQL record current, so this read is always consistent with the latest projected state.

---

## 4. Event Bus: Redis Streams Design

### 4.1 Why Redis Streams (Not Pub/Sub)

The current `EventBus` uses Redis Pub/Sub, which is **fire-and-forget**. 38 event publishes across 15 actors, zero subscribers. Events are lost if no consumer is listening. Redis Streams provide:

- **Persistence**: Events survive consumer restarts
- **Consumer groups**: Multiple consumers each get their own copy of every event (fan-out), or share workload within a group
- **Acknowledgment**: Events aren't removed until explicitly acknowledged — no silent drops
- **Replay**: On startup, consumers can process any events they missed while down
- **Backpressure**: If a consumer falls behind, events queue up rather than being lost

### 4.2 Stream Topology

```
Redis Streams
+-- stream:shipment-lifecycle      # ShipmentCreated, InTransit, ArrivedAtCustoms, Cleared, Delivered
+-- stream:customs-processing      # ALL jurisdiction-specific customs events:
|                                  #   US: EntryFiled, CF28/29, ExamScheduled, Inspection, Accepted
|                                  #   EU: DeclarationLodged/Accepted/Rejected, UnderControl, Released
|                                  #   BR: DIRegistered, Parameterized
|                                  #   IN: BillOfEntryFiled, DutyAssessed, OutOfCharge
|                                  #   CN: CustomsDeclared, CIQInspected, GACCReleased
|                                  #   Cross-jurisdiction: ShipmentHeld, BrokerAssigned, CommunicationSent
+-- stream:transport-ops           # Mode-specific operational events (35+ types):
|                                  #   Air: HAWBIssued, CargoTendered, MAWBConsolidated, ULDBuildUp,
|                                  #        FlightDeparted/Arrived, ULDBreakdown, AirEntryFiled, etc.
|                                  #   Ocean: HBLIssued, ContainerStuffed, VesselDeparted/Arrived,
|                                  #          ContainerDischarged, Deconsolidated, etc.
|                                  #   Ground: BOLIssued, PAPSFiled, BorderArrived, etc.
+-- stream:transit-events          # Intermediate regulatory: ExportClearance, ICS2Filed, PLACIFiled,
|                                  #   TradeNetFiled, TransitDeclaration, TransitScreening,
|                                  #   TransitHeld/Cleared, HubSort, Transshipment, ACASFiled
+-- stream:exception-resolution    # HoldResolved, TransitHoldResolved, Escalated, CageIntake/Released
+-- stream:compliance              # AnalysisGenerated, PGAReviewComplete, ISFValidated
```

**Design rationale:** One stream per bounded context / concern. Related events stay ordered within their context (important for processing sequence), while different contexts consume independently. The 6 streams reflect the domain:
- `shipment-lifecycle`: Status transitions (the core state machine events)
- `customs-processing`: Jurisdiction-specific customs authority events (5 jurisdictions)
- `transport-ops`: Mode-specific operational detail (air/ocean/ground — highest volume)
- `transit-events`: Intermediate regulatory events at transit territory touchpoints
- `exception-resolution`: Hold resolution, cage lifecycle, escalation
- `compliance`: Compliance screening, analysis, ISF validation

**Note:** The dashboard projector subscribes to all 6 streams via consumer groups — it doesn't need its own stream.

### 4.3 Consumer Groups

Each actor or service reads from the streams it depends on via consumer groups. **Critical: each consumer group gets its own independent copy of every event.** This is how customs AND pga AND broker_sim all see every `ShipmentArrivedAtCustoms` event — each has its own consumer group.

```
stream:shipment-lifecycle
+-- cg:customs-actor         # customs receives ShipmentArrivedAtCustoms
+-- cg:pga-actor             # pga receives ShipmentArrivedAtCustoms
+-- cg:broker-sim-actor      # broker_sim receives ShipmentArrivedAtCustoms
+-- cg:compliance-actor      # compliance_engine receives all lifecycle events
+-- cg:consolidator-actor    # consolidator receives ShipmentInTransit (for grouping candidates)
+-- cg:preclearance-actor    # preclearance receives ShipmentCreated
+-- cg:documents-actor       # documents receives ShipmentCreated
+-- cg:isf-actor             # isf receives ShipmentCreated
+-- cg:terminal-actor        # terminal receives ShipmentArrivedAtCustoms
+-- cg:carrier-actor         # carrier receives ShipmentInTransit (for time monitoring)
+-- cg:cage-actor            # cage receives ShipmentArrivedAtCustoms (for intake readiness)
+-- cg:financial-actor       # financial receives ShipmentCleared, ShipmentDelivered
+-- cg:demurrage-actor       # demurrage receives ShipmentArrivedAtCustoms (ocean filter, lazy computation)
+-- cg:dashboard-projector   # read model updater

stream:customs-processing
+-- cg:resolution-actor      # resolution receives ShipmentHeld (all jurisdictions)
+-- cg:cage-actor            # cage receives ShipmentHeld, ShipmentInspection, UnderControl, CIQInspected
+-- cg:carrier-actor         # carrier receives ShipmentCleared (for delivery, all jurisdictions)
+-- cg:broker-sim-actor      # broker_sim receives CF28/29 (US), DeclarationRejected (EU), etc.
+-- cg:shipper-response      # shipper_response receives CF28Issued, CommunicationSent
+-- cg:cbp-authority-actor   # cbp_authority receives EntryFiled (US)
+-- cg:financial-actor       # financial receives DutyAssessed (IN) for fee calculation
+-- cg:dashboard-projector   # read model updater — ALL jurisdiction events

stream:transport-ops
+-- cg:customs-actor         # customs receives AirEntryFiled/OceanEntryFiled/GroundEntryFiled
+-- cg:financial-actor       # financial receives AirCustomsCleared/OceanCustomsCleared/GroundCustomsCleared
+-- cg:dashboard-projector   # read model updater — ALL mode-specific operational events

stream:transit-events
+-- cg:compliance-actor      # compliance receives ICS2Filed, ACASFiled, TransitScreening
+-- cg:resolution-actor      # resolution receives TransitHeld (for transit hold resolution)
+-- cg:carrier-actor         # carrier receives TransitCleared (resume tracking)
+-- cg:dashboard-projector   # read model updater — ALL transit visibility events

stream:exception-resolution
+-- cg:carrier-actor         # carrier receives HoldResolved, TransitHoldResolved
+-- cg:cage-actor            # cage receives HoldResolved (for release)
+-- cg:financial-actor       # financial receives CageReleased
+-- cg:shipper-response      # shipper_response receives HoldEscalated
+-- cg:dashboard-projector   # read model updater

stream:compliance
+-- cg:dashboard-projector   # read model updater
```

### 4.4 Stream Configuration

```python
STREAM_CONFIG = {
    "stream:shipment-lifecycle": {
        "maxlen": 10_000,        # ~2 days of events at peak volume
    },
    "stream:customs-processing": {
        "maxlen": 10_000,        # Includes all 5 jurisdictions' customs events
    },
    "stream:transport-ops": {
        "maxlen": 30_000,        # Highest volume: ~10 mode-specific events per shipment
    },
    "stream:transit-events": {
        "maxlen": 20_000,        # High volume: multiple events per shipment per transit leg
    },
    "stream:exception-resolution": {
        "maxlen": 5_000,
    },
    "stream:compliance": {
        "maxlen": 5_000,
    },
}
```

Streams are trimmed approximately (`XTRIM ... MAXLEN ~`) by the coordinator on each dashboard cycle, reusing the existing periodic loop. No separate trimming task needed.

---

## 5. Actor Redesign: Poll-and-Lock -> Event-Driven

### 5.1 Current Pattern (Anti-pattern)

Every actor currently follows this pattern:

```python
async def tick(self):
    async with self.get_session() as session:
        async with session.begin():
            result = await session.execute(
                select(Shipment)
                .where(Shipment.status == "at_customs")
                .with_for_update(skip_locked=True)
            )
            shipments = list(result.scalars().all())  # Unbounded
            for shipment in shipments:
                await self._process(session, shipment)
```

**Problems:**
1. **Connection hoarding**: Each actor holds a DB connection for entire tick (processing N shipments)
2. **Unbounded locking**: `FOR UPDATE SKIP LOCKED` locks ALL matching rows
3. **Silent processing gaps**: When customs locks 50 rows, PGA skips all 50. No visibility.
4. **Contention scales with actors**: Adding a new actor that polls `at_customs` worsens all existing actors
5. **Pool starvation**: 19 actors + API endpoints share 30 connections

### 5.2 New Pattern: Event-Sourced Actors with Single-Writer Aggregation

**Fundamental principle:** Actors consume events, run business logic, and **emit outcome events**. They do NOT write to the shipment record. The shipment record is a **projection** maintained by a single Shipment Aggregator (Section 5.4). The event stream is the source of truth.

**Proactive evaluation principle (Section 5.5):** Actors subscribe to every event that could satisfy one of their readiness prerequisites, not just the "expected" trigger event. They evaluate whether they have everything they need before acting, and they act at the earliest possible moment — not at a specific status milestone.

```
Event arrives → Actor evaluates READINESS CRITERIA on event payload →
  If NOT ready: accumulate partial state, wait for more events →
  If READY: run pure business logic → emit OUTCOME EVENT (NO DB write) →
Shipment Aggregator consumes ALL events → Aggregator updates shipment record
```

Business logic is extracted into **pure functions** that return a `ProcessingOutcome` dataclass — a description of what should happen, not a side effect:

```python
@dataclass
class ProcessingOutcome:
    """Result of processing a shipment — returned by pure business logic functions.

    Describes the actor's decision. The actor publishes this as an event.
    The Shipment Aggregator (Section 5.4) applies it to the DB.
    Actors NEVER write to the shipment record directly.
    """
    new_status: str | None = None          # None = no status change
    hold_type: str | None = None
    description: str = ""
    event_type: str = ""                   # e.g., "ShipmentCleared", "ShipmentHeld"
    stream: str = ""                       # Target stream for the event
    references_update: dict | None = None  # Key-value pairs to merge into event payload
    extra_data: dict | None = None         # Additional domain-specific data for the event

class BrokerDeclarationHandler(EventDrivenActor):
    """PROACTIVE EVALUATOR: the broker service evaluates readiness to file declarations.

    The broker — NOT the customs authority — decides when to file. It subscribes to
    every event that could satisfy a data prerequisite and files the declaration at
    the earliest possible moment (typically while the shipment is still in_transit).

    The customs authority (see CustomsAuthorityActor below) then RECEIVES and
    adjudicates the submitted declaration. It does not go looking for work.

    Emits outcome events — does NOT write to the shipment record.
    The Shipment Aggregator (Section 5.4) projects events into DB state.
    """

    # Readiness criteria: HS code + origin + destination + declared_value + shipper info
    # These can be satisfied by ShipmentCreated, ClassificationCompleted, etc.
    READINESS_FIELDS = {"hs_code", "origin", "destination", "declared_value", "company_name"}

    subscriptions = [
        ("stream:shipment-lifecycle", "cg:broker-declaration", [
            "ShipmentCreated",            # May already have all prerequisites
            "ClassificationCompleted",    # HS code now available — re-evaluate
        ]),
    ]

    # In-memory readiness accumulator per shipment (see Section 5.5)
    _readiness: dict[str, dict] = {}

    async def handle_event(self, event: dict) -> None:
        """Evaluate readiness and file declaration at the earliest possible moment."""
        payload = event["payload"]
        shipment_id = payload["id"]

        # Accumulate state from each event
        state = self._readiness.setdefault(shipment_id, {})
        for field in self.READINESS_FIELDS:
            if field in payload and payload[field] is not None:
                state[field] = payload[field]

        # Already filed? Nothing to do.
        if state.get("filed"):
            return

        # Check if all data prerequisites are met
        if not all(state.get(f) for f in self.READINESS_FIELDS):
            return  # Not ready yet — wait for more events

        # All prerequisites met → FILE NOW (even if shipment is still in_transit)
        state["filed"] = True
        outcome = self._prepare_declaration(state, self.config, self.clock.now())

        await self.event_bus.publish("stream:customs-processing", "DeclarationSubmitted", {
            "shipment_id": shipment_id,
            "entry_number": outcome.extra_data["entry_number"],
            "hs_code": state["hs_code"],
            "origin": state["origin"],
            "destination": state["destination"],
            "declared_value": state["declared_value"],
            "company_name": state["company_name"],
            "actor": "broker",
            "sim_timestamp": self.clock.now().isoformat(),
        })

        # Clean up readiness accumulator
        del self._readiness[shipment_id]


class CustomsAuthorityActor(EventDrivenActor):
    """REACTIVE RESPONDER: the customs authority adjudicates submitted declarations.

    The customs authority does NOT proactively evaluate readiness. It RESPONDS to:
    1. DeclarationSubmitted — broker filed a declaration → analyze and decide
    2. ShipmentArrivedWithoutClearance — shipment arrived at port with no
       pre-clearance → hold by default (this is a FAILURE of the proactive system)

    This mirrors real-world CBP/customs behavior: the authority receives filings
    and adjudicates them. It does not go looking for work to do.
    """

    subscriptions = [
        ("stream:customs-processing", "cg:customs-authority", [
            "DeclarationSubmitted",            # Broker filed → analyze and decide
            "ShipmentArrivedWithoutClearance",  # Exception path → hold by default
        ]),
    ]

    async def handle_event(self, event: dict) -> None:
        """Adjudicate a submitted declaration or handle uncleared arrival."""
        payload = event["payload"]

        if event["event_type"] == "DeclarationSubmitted":
            # Normal path: broker pre-filed → customs analyzes
            queue_depth = await self._get_queue_depth()
            outcome = self._screen_and_route(payload, self.config, self.clock.now(), queue_depth)

            validate_transition(
                from_status=payload.get("status", "entry_filed"),
                to_status=outcome.new_status,
                actor=self.actor_id,
                jurisdiction=payload.get("jurisdiction", "US"),
            )

            await self.event_bus.publish(outcome.stream, outcome.event_type, {
                "shipment_id": payload["shipment_id"],
                "previous_status": payload.get("status"),
                "new_status": outcome.new_status,
                "hold_type": outcome.hold_type,
                "description": outcome.description,
                "actor": self.actor_id,
                "jurisdiction": payload.get("jurisdiction"),
                "sim_timestamp": self.clock.now().isoformat(),
                "references_update": outcome.references_update,
                **(outcome.extra_data or {}),
            })

        elif event["event_type"] == "ShipmentArrivedWithoutClearance":
            # Exception path: shipment arrived at port without pre-clearance.
            # This SHOULD be rare — it means the broker didn't file in time.
            # Default action: hold for manual review.
            await self.event_bus.publish("stream:customs-processing", "ShipmentHeld", {
                "shipment_id": payload["shipment_id"],
                "new_status": "held",
                "hold_type": "no_preclearance",
                "description": "Shipment arrived without pre-clearance — held for manual review",
                "actor": self.actor_id,
                "sim_timestamp": self.clock.now().isoformat(),
            })

    @staticmethod
    def _screen_and_route(
        state: dict, config, sim_now: datetime, queue_depth: int
    ) -> ProcessingOutcome:
        """Pure function: decide customs screening outcome.

        Takes declaration payload + config, returns a ProcessingOutcome.
        No DB session, no ORM objects, no side effects.
        """
        # ... existing screening logic, refactored to operate on dicts ...
        if stp_eligible:
            return ProcessingOutcome(
                new_status="cleared",
                description=f"STP clearance in {minutes}min (queue depth: {queue_depth})",
                event_type="ShipmentCleared",
                stream="stream:customs-processing",
            )
        # ... etc
```

**Two actor categories illustrated above:**

1. **Proactive Evaluator** (`BrokerDeclarationHandler`): Subscribes to data-readiness events, accumulates prerequisites, acts at the earliest moment all criteria are met. The broker files declarations — the customs authority does not.
2. **Reactive Responder** (`CustomsAuthorityActor`): Subscribes to command/exception events (`DeclarationSubmitted`, `ShipmentArrivedWithoutClearance`). Adjudicates incoming filings. Does not go looking for work.

**Why this is fundamentally better than event-triggered CRUD:**
1. **No polling**: Actors sleep until there's actual work
2. **No DB access**: Actors work entirely from event payloads — zero DB connections consumed
3. **Single writer**: Only the Shipment Aggregator writes to the shipment record — eliminates all concurrency bugs by construction (no JSONB race conditions, no multi-writer contention)
4. **Proactive evaluation**: Services act at the earliest possible moment. Declarations are pre-filed while cargo is still in transit. Screening happens instantly on arrival instead of after a polling cycle.
5. **Guaranteed delivery**: Every event is delivered to every subscribed consumer group. Nothing silently skipped.
6. **Contention-free by construction**: Adding a new actor = adding a new consumer group. Zero impact on existing actors.
7. **Event stream is source of truth**: "Why is this shipment in this state?" is answered by replaying the event stream, not guessing which of 15+ actors last touched the record
8. **Pure testability**: Business logic functions take dicts and return dataclasses — no mocking DB sessions, no transaction management in tests

### 5.3 Actor Classification and Migration Strategy

Actors fall into four categories based on their processing model:

#### Category A: Proactive Evaluators (data-prerequisite-driven)

These actors subscribe to **every event that could satisfy a readiness prerequisite**, maintain a readiness accumulator (Section 5.5), and act at the **earliest possible moment** — regardless of shipment status. They do not wait for a specific status milestone.

| Actor | Current Trigger | Readiness Criteria | Subscribes To (proactive) | Earliest Action |
|---|---|---|---|---|
| `broker_sim` (declaration) | Polls `at_customs` + `EntryFiling` | HS code + origin + dest + value + shipper | `ShipmentCreated`, `ClassificationCompleted` | **Pre-files declaration while shipment is in_transit** |
| `broker_sim` (assignment) | Polls `at_customs` | Shipment data for complexity assessment | `ShipmentCreated`, `ClassificationCompleted` | **Assigns broker and begins prep during transit** |
| `pga` | Polls `at_customs` | HS code + product category → PGA mapping | `ClassificationCompleted` | **Determines PGA requirements as soon as HS code known** |
| `compliance` | Polls `analysis IS NULL` | Product + shipper entity info | `ShipmentCreated`, `ClassificationCompleted`, `RegulatorySignalDetected` | **Re-screens on regulatory changes**, not just at creation |
| `financial` | Polls `cleared`/`delivered` | Tariff result + declared value | `ClassificationCompleted`, `ShipmentCleared`, `CageReleased` | **Pre-calculates fees as soon as tariff data available** |
| `documents` | Polls shipments needing docs | Documents exist for shipment | `ShipmentCreated`, `ClassificationCompleted`, `DocumentUploaded` | **Validates docs as soon as they exist** (pre-departure) |
| `isf` | Polls `booked`/`in_transit` | Ocean shipment with shipper data | `ShipmentCreated` | Already proactive — files ISF pre-departure |
| `preclearance` | Polls `booked` | Shipment exists with product info | `ShipmentCreated` | Already proactive — runs immediately after creation |

#### Category A2: Reactive Responders (command/exception-driven)

These actors do NOT proactively evaluate readiness. They **respond to commands or exceptions** — events that represent explicit submissions or failure conditions. This mirrors real-world behavior: customs authorities receive filings and adjudicate them; they do not go looking for work.

| Actor | Current Trigger | Responds To | Subscribes To | Why Reactive |
|---|---|---|---|---|
| `customs` (authority) | Polls `at_customs` | `DeclarationSubmitted` (broker filed → analyze/decide), `ShipmentArrivedWithoutClearance` (exception → hold) | stream:customs-processing | **Customs authority adjudicates submitted declarations** — it does not proactively seek work. The broker decides when to file. |
| `cbp_authority` | Polls `EntryFiling` statuses | `EntryFiled`, `CF28Responded`, `ProtestFiled` | stream:customs-processing | Government authority responds to submissions |
| `cage` (intake) | Polls `held`/`inspection` | `ShipmentHeld`, `ShipmentInspection` | stream:customs-processing | Physically-triggered — cargo must be held first |
| `terminal` | Polls `at_customs` | `ShipmentArrivedAtCustoms` | stream:shipment-lifecycle | Terminal ops require physical arrival |
| `shipper_response` | Timer + full table scan | `CommunicationSent`, `CF28Issued`, `HoldEscalated` | stream:customs-processing, stream:exception-resolution | Responds to communications directed at shipper |
| `consolidator` | Polls `in_transit` | `ShipmentInTransit` (buffered consumer — see Section 11.2) | stream:shipment-lifecycle | Groups shipments already in transit |

> **Key distinction — Broker vs Customs Authority:**
> - The **broker** is the PROACTIVE EVALUATOR — "Do I have HS code + origin + value + shipper? → File declaration NOW, even if shipment is still in transit."
> - The **customs authority** is the REACTIVE RESPONDER — "A declaration was submitted to me → Analyze and decide (accept/reject/CF-28/exam)." OR "A shipment arrived without clearance → Hold by default."
> - A shipment arriving at port without pre-clearance is a FAILURE of the proactive system. It means the broker didn't file in time. The customs authority's hold-by-default response is the exception path, not the happy path.

> **Compatibility note — `preclearance` actor identity:** The `preclearance` actor (`actor_id = "preclearance"`) currently records state transitions with `actor="platform"` rather than `"preclearance"` in its `transition()` calls. This means event consumers filtering by `actor` field will see `"platform"` for preclearance-triggered transitions. The event-driven migration should either: (a) normalize the actor identity to `"preclearance"` in the transition calls (recommended — fixes the mismatch at source), or (b) map `"platform"` → `"preclearance"` in the event envelope when the preclearance actor publishes events. This is a data quality fix, not an architectural concern.

#### Category B: Hybrid — Event-Triggered Start + Sim-Clock Polling for Completion

These actors have time-dependent logic: "process after N simulated days." They receive an initial event trigger but use the simulation clock to check elapsed time. This hybrid approach respects the `SimulationClock`'s dynamic time ratio and pause behavior.

| Actor | Event Trigger | Time-Dependent Check | Why Hybrid |
|---|---|---|---|
| `carrier` | `ShipmentInTransit` starts tracking | `clock.now() - created_at >= transit_days` on periodic check | Transit time depends on sim clock speed which can change |
| `resolution` | `ShipmentHeld` starts tracking | `clock.now() - hold_time >= review_days` on periodic check | Review period depends on sim clock |
| `pga` (completion) | `ShipmentArrivedAtCustoms` starts review | `clock.now() - review_start >= review_days` on periodic check | Review window is time-dependent |
| `cage` (dwell/release) | `CageIntake` starts tracking | Lazy computation for dwell_days/storage_cost (see Section 11.3). Only GO deadline warnings need periodic check. | Eliminates per-tick dwell updates |
| `demurrage` | `ShipmentArrivedAtCustoms` (ocean only) | Lazy computation — stores arrival time + carrier policy, computes charges on demand (see Section 11.3) | Replaces timer-based accumulation with lazy eval |

**Implementation pattern for hybrid actors:**

```python
class CarrierActor(EventDrivenActor):
    """Hybrid: event-triggered start + sim-clock polling for transit completion."""

    subscriptions = [
        ("stream:shipment-lifecycle", "cg:carrier-actor", ["ShipmentInTransit"]),
        ("stream:customs-processing", "cg:carrier-actor", ["ShipmentCleared", "EntryAccepted"]),
        ("stream:exception-resolution", "cg:carrier-actor", ["HoldResolved", "TransitHoldResolved"]),
    ]

    def __init__(self, ...):
        super().__init__(...)
        # In-memory tracking cache — rebuilt from DB on startup (see below)
        self._tracking: dict[str, dict] = {}

    async def startup(self) -> None:
        """Rebuild tracking cache from DB on startup.

        The _tracking dict is an in-memory cache, NOT durable state.
        On restart, we rebuild it by querying shipments in 'in_transit' status
        that have persisted transit_days in their references.
        """
        async with self.get_session() as session:
            result = await session.execute(
                select(Shipment).where(Shipment.status == "in_transit")
            )
            for shipment in result.scalars():
                refs = dict(shipment.references or {})
                if "transit_days" in refs:
                    self._tracking[str(shipment.id)] = {
                        "state": ShipmentEventPayload.from_shipment(shipment),
                        "transit_days": refs["transit_days"],  # Persisted on first eval
                        "started_at": shipment.created_at.isoformat(),
                    }

    async def handle_event(self, event: dict) -> None:
        if event["event_type"] == "ShipmentInTransit":
            shipment = event["payload"]
            # Calculate transit time ONCE and persist it (see Section 11.4)
            transit_days = self._calculate_transit_time(shipment)

            self._tracking[shipment["id"]] = {
                "state": shipment,
                "transit_days": transit_days,
                "started_at": event["sim_timestamp"],
            }

            # Persist the computed duration to the shipment record
            async with self.get_session() as session:
                async with session.begin():
                    s = await session.get(Shipment, shipment["id"])
                    if s:
                        refs = dict(s.references or {})
                        refs["transit_days"] = transit_days
                        s.references = refs
                        flag_modified(s, "references")

        elif event["event_type"] in ("ShipmentCleared", "EntryAccepted"):
            await self._process_delivery(event["payload"])
        elif event["event_type"] in ("HoldResolved", "TransitHoldResolved"):
            await self._resume_transit(event["payload"])

    async def tick(self) -> None:
        """Periodic check for transit completion — uses sim clock."""
        sim_now = self.clock.now()
        completed = []
        for sid, tracking in self._tracking.items():
            elapsed = sim_now - datetime.fromisoformat(tracking["started_at"])
            if elapsed >= timedelta(days=tracking["transit_days"]):
                await self._arrive_at_customs(tracking["state"])
                completed.append(sid)
        for sid in completed:
            del self._tracking[sid]
```

**`_tracking` persistence semantics:**
- `_tracking` is an **in-memory cache**, NOT durable state
- On restart, `startup()` rebuilds it by querying DB for `in_transit` shipments with persisted `transit_days` in `references`
- Computed durations (transit_days, review_days) are persisted to the shipment record on first evaluation — this ensures deterministic behavior across restarts (see Section 11.4)
- The cache avoids per-tick DB queries for the common case (checking elapsed time)

**Why this works with the simulation clock:**
- `clock.now()` is called fresh on every tick — automatically handles ratio changes and pause/resume
- No real-time sorted sets or scheduled timers that break when the clock changes
- The event subscription ensures we don't miss shipments (no SKIP LOCKED gaps)
- The tick interval only affects time-dependent completion checks, not event reception

#### Category C: Timer-Driven Producers (Minimal Change)

These actors create new data rather than reacting to status changes. They remain timer-driven but publish events and, where applicable, switch to event-based shipment discovery to eliminate `FOR UPDATE` contention.

| Actor | Behavior | Change |
|---|---|---|
| `shipper` | Creates shipments on Poisson timer | **Publishes** `ShipmentCreated` event after DB insert |
| `exceptions` | Injects exceptions on `in_transit`/`at_customs` shipments | **Reclassified to hybrid**: subscribes to `ShipmentInTransit` + `ShipmentArrivedAtCustoms` to build local tracking set, then probabilistic injection on timer. Eliminates `FOR UPDATE SKIP LOCKED` on `in_transit`/`at_customs` rows (was competing with 7 other actors). Publishes `ExceptionInjected`. |
| `disruption` | Generates port disruption events | **Reclassified to hybrid**: subscribes to `ShipmentInTransit` + `ShipmentArrivedAtCustoms` to maintain port-to-shipment map in memory, then timer-based disruption generation uses the map instead of querying DB. Eliminates 2 `FOR UPDATE` queries. Publishes `DisruptionGenerated`. |

### 5.4 Shipment Aggregator — Single Writer to the Shipment Record

**The Shipment Aggregator is the architectural cornerstone of the single-writer model.** It is the ONLY component that writes to the `shipments` table. All actors and services emit events; the Aggregator projects those events into the shipment's current state.

**Design principle:** The event stream (Redis Streams) is the source of truth. The PostgreSQL `shipments` table is a **read projection** — a materialized view of the event stream optimized for queries. If the shipment record is lost, it can be reconstructed by replaying the event stream.

```python
class ShipmentAggregator:
    """Single writer to the shipments table.

    Consumes ALL domain events from ALL 6 streams and projects them
    into the shipment record. No other component writes to shipments.

    This eliminates:
    - JSONB events append race conditions (Section 11.1 — now impossible)
    - Multi-writer contention (15+ actors no longer compete)
    - "Who last touched this record?" debugging mystery
    """

    subscriptions = [
        ("stream:shipment-lifecycle",   "cg:shipment-aggregator", ["*"]),
        ("stream:customs-processing",   "cg:shipment-aggregator", ["*"]),
        ("stream:transport-ops",        "cg:shipment-aggregator", ["*"]),
        ("stream:transit-events",       "cg:shipment-aggregator", ["*"]),
        ("stream:exception-resolution", "cg:shipment-aggregator", ["*"]),
        ("stream:compliance",           "cg:shipment-aggregator", ["*"]),
    ]

    # Event → projection handlers
    _handlers: dict[str, Callable] = {
        # Lifecycle
        "ShipmentCreated":          "_project_creation",
        "ShipmentInTransit":        "_project_status_change",
        "ShipmentArrivedAtCustoms": "_project_status_change",
        "ShipmentCleared":          "_project_status_change",
        "ShipmentDelivered":        "_project_status_change",
        "ShipmentHeld":             "_project_hold",

        # Customs processing (all jurisdictions)
        "EntryFiled":               "_project_entry_filed",
        "CF28Issued":               "_project_status_change",
        "CF28Responded":            "_project_status_change",
        "EntryAccepted":            "_project_status_change",
        "DeclarationLodged":        "_project_status_change",
        "DeclarationAccepted":      "_project_status_change",
        "UnderControl":             "_project_status_change",
        "CustomsReleased":          "_project_status_change",
        # ... all other jurisdiction-specific events follow same pattern

        # Domain-specific projections
        "AnalysisGenerated":        "_project_analysis",
        "FeesCalculated":           "_project_financials",
        "CageIntake":               "_project_cage_status",
        "CageReleased":             "_project_cage_released",
        "HoldResolved":             "_project_hold_resolved",
        "TransitHoldResolved":      "_project_transit_hold_resolved",

        # Transport ops — update references with mode-specific data
        "HAWBIssued":               "_project_transport_reference",
        "HBLIssued":                "_project_transport_reference",
        "BOLIssued":                "_project_transport_reference",
        "FlightDeparted":           "_project_transport_reference",
        "VesselDeparted":           "_project_transport_reference",
        # ... all mode-specific events update references dict
    }

    async def handle_event(self, event: dict) -> None:
        """Project an event into the shipment record."""
        event_type = event["event_type"]
        handler_name = self._handlers.get(event_type)
        if not handler_name:
            return  # Unknown event type — ack and skip

        shipment_id = event.get("correlation_id") or event["payload"].get("shipment_id")
        if not shipment_id:
            return

        async with self.get_session() as session:
            async with session.begin():
                shipment = await session.get(Shipment, shipment_id)
                if shipment is None:
                    logger.warning("aggregator_shipment_not_found",
                                   shipment_id=shipment_id, event_type=event_type)
                    return

                # Apply the projection
                handler = getattr(self, handler_name)
                handler(shipment, event)

                # Append to the timeline (single writer — no race condition)
                timeline_entry = {
                    "timestamp": event.get("sim_timestamp", event.get("wall_timestamp")),
                    "event_type": event_type,
                    "description": event.get("payload", {}).get("description", ""),
                    "actor": event.get("actor", ""),
                }
                events = list(shipment.events or [])
                events.append(timeline_entry)
                shipment.events = events
                flag_modified(shipment, "events")

            # Connection released — held for ~5ms (single-row update by PK)

    def _project_status_change(self, shipment: Shipment, event: dict) -> None:
        """Apply a status transition from an event."""
        payload = event.get("payload", {})
        new_status = payload.get("new_status")
        if new_status and new_status != shipment.status:
            # Validate the transition is legal
            validate_transition(
                from_status=shipment.status,
                to_status=new_status,
                actor=event.get("actor", "aggregator"),
                jurisdiction=payload.get("jurisdiction", "US"),
            )
            shipment.status = new_status

    def _project_hold(self, shipment: Shipment, event: dict) -> None:
        """Apply hold status + hold_type from a ShipmentHeld event."""
        payload = event.get("payload", {})
        shipment.status = "held"
        shipment.hold_type = payload.get("hold_type")
        # Merge references if transit hold markers present
        refs_update = payload.get("references_update")
        if refs_update:
            refs = dict(shipment.references or {})
            refs.update(refs_update)
            shipment.references = refs
            flag_modified(shipment, "references")

    def _project_entry_filed(self, shipment: Shipment, event: dict) -> None:
        """Apply entry number from an EntryFiled event."""
        payload = event.get("payload", {})
        shipment.entry_number = payload.get("entry_number")
        self._project_status_change(shipment, event)

    def _project_analysis(self, shipment: Shipment, event: dict) -> None:
        """Apply analysis results from compliance engine."""
        payload = event.get("payload", {})
        shipment.analysis = payload.get("analysis")
        flag_modified(shipment, "analysis")

    def _project_financials(self, shipment: Shipment, event: dict) -> None:
        """Apply financial calculations."""
        payload = event.get("payload", {})
        shipment.financials = payload.get("financials")
        flag_modified(shipment, "financials")

    def _project_cage_status(self, shipment: Shipment, event: dict) -> None:
        """Apply cage intake status."""
        payload = event.get("payload", {})
        shipment.cage_status = payload.get("cage_status")
        flag_modified(shipment, "cage_status")

    def _project_transport_reference(self, shipment: Shipment, event: dict) -> None:
        """Merge mode-specific transport references (HAWB, HBL, BOL, flight, vessel, etc.)."""
        payload = event.get("payload", {})
        refs_update = payload.get("references_update")
        if refs_update:
            refs = dict(shipment.references or {})
            refs.update(refs_update)
            shipment.references = refs
            flag_modified(shipment, "references")
```

**Operational characteristics:**

| Aspect | Detail |
|--------|--------|
| **Throughput** | Processes ~200-500 events/second (single-row PK update per event, ~5ms each) |
| **Ordering** | Events within each stream are processed in order. Cross-stream ordering is not required — each event carries sufficient data to apply its projection independently |
| **Recovery** | On crash, resumes from last acknowledged event in each consumer group. Redis Streams PEL (Pending Entries List) ensures no events are lost. |
| **Bottleneck risk** | The Aggregator is a single point of processing for shipment state. If it falls behind, shipment records become stale. Mitigation: (a) Redis Streams buffer events during downtime, (b) the Aggregator catches up at full speed on recovery, (c) read stores (dashboard, shipment list) are updated by their own independent projectors and remain current |
| **Monitoring** | Track consumer lag per stream via `XPENDING`. Alert if any stream's lag exceeds 1000 events or 30 seconds wall-clock time. |

**What the Aggregator does NOT do:**
- It does NOT update Redis read stores (dashboard, shipment list). Those have their own projectors (Section 6.2.1, 6.2.2).
- It does NOT validate business rules. Actors validate before emitting events. The Aggregator trusts the event stream.
- It does NOT re-emit events. It is a pure sink — it consumes events and writes to PostgreSQL.

### 5.5 Proactive Evaluator Pattern

> **Stakeholder principle:** "Every service must be designed not to follow a process, but to always evaluate if the requirements are met for it to do its job."

The previous design (event-triggered CRUD) replaced polling with event subscriptions but kept the **sequential processing anti-pattern**: each actor subscribed to the "expected" status-change event (e.g., customs subscribed to `ShipmentArrivedAtCustoms`). This is still process-following — the actor waits for a specific step in the sequence rather than evaluating whether it has everything it needs.

The **Proactive Evaluator Pattern** eliminates this anti-pattern:

#### 5.5.1 Core Mechanism

Each actor defines:
1. **Readiness criteria** — the minimum data prerequisites it needs to do its job
2. **Subscriptions** — every event type that could provide one or more of those prerequisites
3. **Readiness accumulator** — in-memory state that tracks which prerequisites have been satisfied for each shipment
4. **Evaluation function** — a pure function that checks: "do I have everything I need?"

```python
class ProactiveEvaluatorMixin:
    """Mixin for actors that evaluate readiness proactively.

    Instead of subscribing to a single "trigger" event, the actor subscribes to
    every event that could satisfy a prerequisite. On each event, it accumulates
    state and evaluates whether it can act.
    """

    # Subclass defines these
    READINESS_FIELDS: set[str] = set()  # e.g., {"hs_code", "origin", "destination", "declared_value"}

    def __init__(self, ...):
        super().__init__(...)
        self._readiness: dict[str, dict] = {}  # shipment_id → accumulated state

    def _accumulate(self, shipment_id: str, payload: dict) -> dict:
        """Merge new event data into the readiness accumulator."""
        state = self._readiness.setdefault(shipment_id, {})
        for field in self.READINESS_FIELDS:
            if field in payload and payload[field] is not None:
                state[field] = payload[field]
        return state

    def _is_ready(self, state: dict) -> bool:
        """Check if all readiness criteria are met."""
        return all(state.get(f) is not None for f in self.READINESS_FIELDS)

    def _cleanup(self, shipment_id: str) -> None:
        """Remove shipment from readiness accumulator after acting."""
        self._readiness.pop(shipment_id, None)
```

#### 5.5.2 Service Readiness Criteria

**Proactive Evaluators** — data-prerequisite-driven services that file/calculate/validate at the earliest moment:

| Service | Readiness Criteria | Subscribes To | Earliest Possible Action | Current (Sequential) Trigger |
|---------|-------------------|---------------|------------------------|---------------------------|
| **Declaration filing** (broker) | HS code + origin + dest + value + shipper info | `ShipmentCreated`, `ClassificationCompleted` | **Pre-file while `in_transit`** — broker decides when to file, not customs authority | `at_customs` status |
| **Broker assignment** (broker_sim) | Product + origin + dest + value + complexity assessment | `ShipmentCreated`, `ClassificationCompleted` | Assign and begin prep during `booked` or `in_transit` | `at_customs` status |
| **PGA determination** (pga) | HS code + product category → PGA mapping | `ClassificationCompleted` | Determine as soon as HS code is known | `at_customs` status |
| **Document validation** (documents) | At least one document exists for shipment | `ShipmentCreated`, `DocumentUploaded`, `ClassificationCompleted` | Validate as soon as docs uploaded (pre-departure) | `at_customs` status |
| **Fee calculation** (financial) | Tariff result + declared value | `ClassificationCompleted`, `ShipmentCleared` | Pre-calculate when tariff data available | `at_customs` or `cleared` |
| **Compliance re-screening** (compliance) | HS code + parties + origin + regulatory signal | `ShipmentCreated`, `ClassificationCompleted`, `RegulatorySignalDetected` | Re-screen on regulatory changes (not one-shot) | Order analysis time (once) |

**Reactive Responders** — command/exception-driven services that adjudicate or respond:

| Service | Responds To | Why Reactive |
|---------|------------|-------------|
| **Customs authority** (customs) | `DeclarationSubmitted` (adjudicate), `ShipmentArrivedWithoutClearance` (exception hold) | Government authority receives and adjudicates filings — does NOT proactively seek work |
| **CBP authority** (cbp_authority) | `EntryFiled`, `CF28Responded`, `ProtestFiled` | Government authority responds to submissions |
| **Cage intake** (cage) | `ShipmentHeld`, `ShipmentInspection` | Physically-triggered — cargo must be held first |

#### 5.5.3 State Machine Implications

The proactive model changes the state machine from a required sequence to a set of **possible transitions**:

```
SEQUENTIAL MODEL (current):
  booked → in_transit → at_customs → entry_filed → cleared → delivered
  (each step must happen before the next can begin)

PROACTIVE MODEL (target):
  booked → in_transit ────────────────────────────────→ cleared → delivered
                │ (while in transit, in parallel:)            ▲
                ├── Classification completed                  │
                ├── Broker assigned, prep work begins         │
                ├── PGA requirements determined               │
                ├── Declaration pre-filed                     │
                ├── Documents validated                       │
                ├── Fees pre-calculated                       │
                └── If ALL pre-clearance passed ──────────────┘
                    (on arrival: instant clearance, skip at_customs)
```

This requires adding a new state machine transition: **`in_transit → cleared`** (allowed for `customs_authority` when a pre-filed declaration has been accepted before physical arrival). The `at_customs` state becomes **optional** — it only occurs when the broker didn't file in time (exception path) or when physical inspection is required.

**Who triggers what:**
- The **broker** pre-files the declaration (emits `DeclarationSubmitted`) while shipment is `in_transit`
- The **customs authority** adjudicates and accepts (emits `DeclarationAccepted` or `ShipmentCleared`)
- When the **carrier** reports physical arrival, if the declaration was already accepted → `in_transit → cleared` (skip `at_customs`)
- If the declaration was NOT yet accepted → `in_transit → at_customs` (exception path, customs holds)

#### 5.5.4 Two Event Types: Information Signals vs Commands

The architecture distinguishes two fundamentally different event semantics:

**Information events** — consumed by **proactive evaluators**:
- `ShipmentCreated`, `ClassificationCompleted`, `DocumentUploaded`, `ShipmentArrivedAtCustoms`
- Semantic: "new data is available — re-evaluate your readiness"
- Multiple events can satisfy the same prerequisite differently (e.g., HS code from `ClassificationCompleted` or `ProductUpdated`)
- The broker doesn't care which event provides the HS code — it just checks "do I have an HS code?"
- Idempotent re-evaluation: receiving the same event twice reaches the same conclusion (or does nothing if already acted)

**Command events** — consumed by **reactive responders**:
- `DeclarationSubmitted`, `ShipmentArrivedWithoutClearance`, `EntryFiled`, `CF28Responded`
- Semantic: "an action was taken — process this submission"
- Each command represents an explicit request that requires a response (accept/reject/query)
- The customs authority receives `DeclarationSubmitted` and must adjudicate it — this is a command, not a signal to re-evaluate readiness

**Example — the old vs new mental model:**
- **Old:** `ShipmentArrivedAtCustoms` → "start customs processing" (treated as command to customs actor)
- **New:** `ShipmentArrivedAtCustoms` → "physical arrival data is now available" (information signal). The broker re-evaluates readiness; if it hasn't filed yet, it files now (`DeclarationSubmitted` — a command to customs authority). Customs authority only acts when it receives the command.

#### 5.5.5 Readiness Accumulator Persistence

The `_readiness` dict is an **in-memory cache**, not durable state. On restart:
- Actors rebuild their readiness state by replaying recent events from Redis Streams (using the consumer group's last-acknowledged offset)
- For shipments that were partially ready before restart, the replayed events re-populate the accumulator
- Actors that had already acted (and ACKed the event) will not re-process those shipments

This is safe because:
1. Events are immutable and ordered — replay always produces the same state
2. Actor actions are idempotent — even if an actor re-evaluates, it either acts (if it hadn't yet) or no-ops
3. The Aggregator deduplicates events by `event_id` — duplicate emissions are harmless

### 5.6 EventDrivenActor Base Class

```python
class EventDrivenActor(BaseActor):
    """Base class for event-driven actors.

    Replaces tick-based poll loop with event consumption.
    Actors that also need periodic time checks override tick() AND handle_event().
    """

    subscriptions: list[tuple[str, str, list[str]]] = []
    # (stream, consumer_group, [event_types])

    async def _run_loop(self) -> None:
        """Main loop: consume events, optionally run periodic ticks."""
        if not self.subscriptions:
            return await super()._run_loop()

        try:
            while self._running:
                if self.clock.paused:
                    await asyncio.sleep(0.5)
                    continue

                # Process pending events from all subscribed streams
                for stream, group, event_types in self.subscriptions:
                    events = await self.event_bus.read_stream(
                        stream, group, self.actor_id,
                        count=10,
                        block_ms=500,  # Short block to allow tick() checks
                    )

                    for event_id, event in events:
                        if event_types and event["event_type"] not in event_types:
                            # Not interested in this event type — ack and skip
                            await self.event_bus.ack(stream, group, event_id)
                            continue

                        try:
                            await self.handle_event(event)
                            await self.event_bus.ack(stream, group, event_id)
                        except Exception as exc:
                            await self._handle_processing_error(
                                stream, group, event_id, event, exc
                            )

                # Run periodic tick for hybrid actors (time-dependent checks)
                if hasattr(self, 'tick') and self.tick is not BaseActor.tick:
                    await self.tick()
                    await self.clock.sleep_sim_minutes(self.config.tick_minutes)

        except asyncio.CancelledError:
            pass

    async def handle_event(self, event: dict) -> None:
        """Override to process events. Must be idempotent."""
        raise NotImplementedError

    async def _handle_processing_error(
        self, stream: str, group: str, event_id: str, event: dict, exc: Exception
    ) -> None:
        """Log the error. Event stays unacknowledged for automatic redelivery."""
        logger.error(
            "event_processing_failed",
            actor=self.actor_id,
            event_type=event.get("event_type"),
            event_id=event.get("event_id"),
            correlation_id=event.get("correlation_id"),
            stream=stream,
            error=str(exc),
            exc_info=True,
        )
        # Don't ack — Redis Streams will redeliver after idle timeout
```

### 5.7 Idempotency

All event handlers must be idempotent because Redis Streams provides at-least-once delivery. Three complementary mechanisms:

1. **Actor-side: `validate_transition()` guards**: Before emitting an outcome event, actors call `validate_transition()` on the event payload's current status. If the transition is invalid (e.g., the payload says `status: "at_customs"` but the actor tries to emit a transition to a state that's not allowed from `at_customs`), the actor raises `TransitionError` and the event is retried or dead-lettered. This catches logic errors at the source.

2. **Aggregator-side: State machine guards**: The Shipment Aggregator calls `validate_transition()` when projecting status changes. If a shipment has already moved past the event's expected state (e.g., an `ShipmentCleared` event arrives but the shipment is already `delivered`), the Aggregator logs a warning and acknowledges the event as a safe no-op. This handles duplicate deliveries and out-of-order processing.

3. **Event deduplication via `event_id`**: Each event carries a `uuid-v4` event ID. The Aggregator can optionally check if it has already processed an event by tracking recent event IDs in a Redis set with TTL. This is a belt-and-suspenders mechanism — in practice, the state machine guards handle 99% of duplicates without explicit deduplication.

```python
# Aggregator idempotency — state machine guard
def _project_status_change(self, shipment: Shipment, event: dict) -> None:
    payload = event.get("payload", {})
    new_status = payload.get("new_status")
    if new_status and new_status != shipment.status:
        try:
            validate_transition(
                from_status=shipment.status,
                to_status=new_status,
                actor=event.get("actor", "aggregator"),
                jurisdiction=payload.get("jurisdiction", "US"),
            )
            shipment.status = new_status
        except TransitionError:
            # Shipment has already moved past this state — idempotent skip
            logger.info("aggregator_transition_skipped",
                shipment_id=str(shipment.id),
                current_status=shipment.status,
                attempted_status=new_status,
                event_id=event.get("event_id"),
            )
```

**Note:** Actors no longer read from the database, so the old "optimistic concurrency on write" pattern (check DB state before writing) is no longer needed. Actors work purely from event payloads. The Aggregator is the single point of DB state validation.

### 5.8 BrokerSimActor Decomposition

The `BrokerSimActor` is the most complex actor (5 methods, 4-5 tables, self-referential pipeline). It must be decomposed into 3 focused event handlers.

**Broker Operations and the single-writer model:** Broker Operations owns its own tables (`brokers`, `broker_assignments`, `broker_messages`, and shared write access to `entry_filings`). These are NOT the `shipments` table, so BrokerClaimHandler writes to broker-owned tables directly. For shipment state changes (e.g., `entry_number` assignment), the handler emits events that the Shipment Aggregator projects into the shipment record.

#### Handler 1: BrokerClaimHandler (Proactive Evaluator)

**Consumes:** `ShipmentCreated` (proactive — assigns broker at booking, not at customs arrival)
**Produces:** `BrokerAssigned`, `EntryDraftCreated` (internal event)
**DB writes:** `broker_assignments` and `entry_filings` (broker-owned tables). Does NOT write to `shipments` — the `entry_number` is applied by the Shipment Aggregator when it consumes the `BrokerAssigned` event.

```python
class BrokerClaimHandler(EventDrivenActor):
    """PROACTIVE EVALUATOR: Claims shipments for brokers at booking time.

    Assigns broker and creates draft entry filings as early as possible.
    Writes to broker-owned tables (broker_assignments, entry_filings).
    Emits events for shipment state changes — Aggregator applies them.
    """
    subscriptions = [
        ("stream:shipment-lifecycle", "cg:broker-claim", ["ShipmentCreated"]),
    ]

    async def handle_event(self, event: dict) -> None:
        payload = event["payload"]
        # Pure function: decide broker assignment
        assignment = self._select_broker(payload, self.config)

        # Write to BROKER-OWNED tables (not shipments)
        async with self.get_session() as session:
            async with session.begin():
                # Idempotency: check if already claimed via broker_assignments
                existing = await session.execute(
                    select(BrokerAssignment).where(
                        BrokerAssignment.shipment_id == payload["id"]
                    )
                )
                if existing.scalar_one_or_none():
                    return  # Already claimed — idempotent skip

                # Create BrokerAssignment + EntryFiling in broker-owned tables
                entry_number = generate_entry_number(...)
                session.add(BrokerAssignment(...))
                session.add(EntryFiling(entry_number=entry_number, ...))

        # Emit events — Aggregator will project entry_number onto shipment
        await self.event_bus.publish("stream:customs-processing", "BrokerAssigned", {
            "shipment_id": payload["id"],
            "entry_number": entry_number,
            "broker_id": assignment.broker_id,
            "actor": "broker_sim",
            ...
        })
```

#### Handler 2: BrokerProgressHandler (Proactive Evaluator)

**Consumes:** `EntryDraftCreated` (internal), `ClassificationCompleted` (readiness signal — may unblock filing)
**Produces:** `EntryFiled` / `DeclarationSubmitted` (same event — jurisdiction-specific name in Section 3.3, generic name in Section 5.2)
**DB writes:** `entry_filings` (broker-owned table — checklist progression, status updates)
**Behavior:** Progresses checklist items, submits entries when ready. Files declarations proactively during transit if all prerequisites met. For protected (human-playable) brokers, creates entries in draft status but does NOT auto-progress — the human player drives the workflow.

#### Handler 3: BrokerCF28Handler

**Consumes:** `CF28Issued`
**Produces:** `CF28Responded`
**DB writes:** `entry_filings` (response data), `broker_messages` (communication log)
**Behavior:** Hybrid — event-triggered start + sim-clock polling for response delay. Persists the randomly-generated response delay on first evaluation (see Section 11.4).

**Self-referential pipeline:** The current single-tick sequential pipeline (`_auto_claim → _progress → _submit`) becomes an event chain: `ShipmentCreated → claim → EntryDraftCreated → progress → EntryFiled → DeclarationSubmitted`. The broker proactively files declarations during transit. This is explicit and traceable, unlike the implicit method ordering in the current tick.

**Single-writer boundary clarification:** Broker handlers write to broker-owned tables (`broker_assignments`, `entry_filings`, `broker_messages`) directly — these are within the Broker Operations bounded context. They emit events for any shipment state changes, which the Shipment Aggregator projects. This respects bounded context ownership: Broker Operations owns broker data, Transport & Logistics (via the Aggregator) owns shipment data.

---

## 6. Read Model Projections

### 6.1 Current Problem

The `aggregate_dashboard()` function runs 12-15 SQL queries per invocation:
- `COUNT(*)` on shipments
- `GROUP BY status` with counts
- `SUM(declared_value)`
- JSONB extraction for `financials.predicted_duty` with `SUM`
- `GROUP BY origin, destination` with per-corridor sub-queries for hold rates
- CN-origin held count (UFLPA proxy)
- Cage metrics: 3 separate JSONB extraction queries (`dwell_days`, `go_days_remaining`, `storage_cost_accrued`)

This runs every 5 seconds (SSE cycle), holding a DB connection for the entire sequence. Even with pool separation, this hammers PostgreSQL.

### 6.2 Read Store Design

#### 6.2.1 Platform Dashboard Read Store (Redis Hash)

**Key:** `read:dashboard:platform`
**Updated by:** Dashboard Projector (subscribes to all 6 streams)
**Serves:** Control Tower, Active Shipments overview, Compliance Dashboard
**Maintenance:** Incremental updates — each event adjusts counters, no full recompute

```python
class DashboardProjector:
    """Maintains the dashboard read model from events.

    Replaces the 12-15 SQL query aggregate_dashboard() with
    incremental counter updates on each event.
    """

    async def handle_event(self, event: dict) -> None:
        event_type = event["event_type"]
        payload = event["payload"]
        pipe = self.redis.pipeline()

        if event_type == "ShipmentCreated":
            pipe.hincrby("read:dashboard:platform", "totalEntriesMonth", 1)
            pipe.hincrbyfloat("read:dashboard:platform", "totalValueMonth",
                              float(payload.get("declared_value", 0)))
            # Update corridor volume
            corridor = f"{payload['origin']}->{payload['destination']}"
            pipe.hincrby(f"read:dashboard:corridors:{corridor}", "volume", 1)

        elif event_type == "ShipmentCleared":
            pipe.hincrby("read:dashboard:platform", "clearedCount", 1)

        elif event_type == "ShipmentHeld":
            pipe.hincrby("read:dashboard:platform", "heldCount", 1)
            if payload.get("origin") == "CN":
                pipe.hincrby("read:dashboard:platform", "uflpaDetentions", 1)
            corridor = f"{payload['origin']}->{payload['destination']}"
            pipe.hincrby(f"read:dashboard:corridors:{corridor}", "heldCount", 1)

        elif event_type == "CageIntake":
            pipe.hincrby("read:dashboard:platform", "totalCaged", 1)

        elif event_type == "CageReleased":
            pipe.hincrby("read:dashboard:platform", "totalCaged", -1)
            cage = payload.get("cage_status", {})
            pipe.hincrbyfloat("read:dashboard:platform", "totalAccruedStorage",
                              float(cage.get("storage_cost_accrued", 0)))

        await pipe.execute()
```

**Periodic reconciliation:** A background task runs the full SQL aggregation every 5 minutes and overwrites the Redis hash. This catches any drift from missed events or counter errors. This is NOT the primary read path — it's a safety net.

**API serving:**

```python
@router.get("/api/platform/aggregates")
async def get_aggregates(request: Request):
    redis = request.app.state.redis
    data = await redis.hgetall("read:dashboard:platform")
    if not data:
        # Initial startup: compute from DB and seed the read store
        return await _compute_from_database(request)
    return _format_dashboard(data)
```

Zero SQL queries. Zero DB connections. Sub-millisecond response.

#### 6.2.2 Shipment List Read Store (Redis Sorted Set + Hash)

**Keys:**
- `read:shipments:by_status:{status}` — Sorted sets, scored by `created_at` timestamp
- `read:shipment:{id}:summary` — Hash per shipment with summary fields

**Updated by:** Shipment Projector (subscribes to all streams)
**Serves:** Shipment list endpoints, broker queue, search

```python
async def project_status_change(event: dict) -> None:
    payload = event["payload"]
    sid = payload["id"]
    new_status = payload["status"]
    prev_status = payload.get("previous_status")
    timestamp = datetime.fromisoformat(payload["created_at"]).timestamp()

    pipe = redis.pipeline()

    # Move between status sorted sets
    if prev_status:
        pipe.zrem(f"read:shipments:by_status:{prev_status}", sid)
    pipe.zadd(f"read:shipments:by_status:{new_status}", {sid: timestamp})

    # Update summary hash
    pipe.hset(f"read:shipment:{sid}:summary", mapping={
        "id": sid,
        "product": payload["product"],
        "origin": payload["origin"],
        "destination": payload["destination"],
        "status": new_status,
        "carrier": payload.get("carrier", ""),
        "declared_value": str(payload.get("declared_value", 0)),
        "company_name": payload.get("company_name", ""),
        "transport_mode": payload.get("transport_mode", ""),
        "hold_type": payload.get("hold_type", ""),
        "created_at": payload["created_at"],
    })

    await pipe.execute()
```

**TTL:** Shipment summary hashes expire 7 days after reaching terminal status (`delivered`, `rejected`).

#### 6.2.3 Shipment Detail (Write-Through Cache)

For full shipment detail (events timeline, waypoints, cargo detail):

- **Primary store:** PostgreSQL `shipments` table (unchanged)
- **Cache:** `read:shipment:{id}:detail` — JSON string, TTL 30s
- **Invalidation:** Any event with `correlation_id == shipment_id` deletes the cache key

This is the simplest approach for detail views — infrequent access (one shipment at a time), complex nested data. Cache miss falls through to PostgreSQL.

### 6.3 Startup Seeding

When the application starts, read stores may be empty (Redis restart, first deploy). The startup sequence:

1. Check if `read:dashboard:platform` exists
2. If not, run `aggregate_dashboard()` once to seed the dashboard read store
3. Scan shipments table to seed the shipment list sorted sets
4. Begin event consumption — all future updates are incremental

This runs once, takes ~2-5 seconds, and only happens when Redis read stores are empty.

---

## 7. API Layer Changes

### 7.1 Endpoint Migration Matrix

| Endpoint | Current Source | New Source | Priority |
|---|---|---|---|
| `GET /api/platform/aggregates` | `aggregate_dashboard()` 12-15 SQL queries | Redis hash `read:dashboard:platform` | **P0** |
| SSE `/api/simulation/events` | Dashboard aggregator -> PostgreSQL | Dashboard projector -> Redis | **P0** |
| `GET /api/shipments` | PostgreSQL SELECT | Redis sorted sets + hashes | **P0** |
| `GET /api/shipments/{id}` | PostgreSQL SELECT | Redis cache -> PostgreSQL on miss | **P1** |
| `GET /api/broker/queue` | PostgreSQL JOIN | Redis sorted set + hashes (filtered by broker) | **P1** |
| `POST /api/shipments` | PostgreSQL INSERT | PostgreSQL INSERT + publish `ShipmentCreated` | **P1** |
| `GET /health` | PostgreSQL SELECT 1 | Unchanged (infrastructure check) | N/A |
| LLM endpoints | Various | Unchanged (LLM-bound, not DB-bound) | N/A |

### 7.2 Write Path — Events First, DB Projection Second

**Fundamental principle:** The event stream is the source of truth. Actors publish events to Redis Streams. The Shipment Aggregator (Section 5.4) consumes events and projects them into the PostgreSQL shipment record. Actors NEVER write to the `shipments` table.

```
Actor decision → Publish event to Redis Stream → Shipment Aggregator consumes →
Aggregator projects into PostgreSQL shipment record + appends to events timeline
```

**Two categories of writes:**

#### 1. Shipment state changes (via Aggregator)

Actors emit events describing outcomes. The Aggregator applies them to the shipment record.

```python
# Actor write path — emit event only
outcome = self._screen_and_route(payload, self.config, self.clock.now())
await self.event_bus.publish(outcome.stream, outcome.event_type, {
    "shipment_id": payload["id"],
    "new_status": outcome.new_status,
    "description": outcome.description,
    "actor": self.actor_id,
    "jurisdiction": payload.get("jurisdiction"),
    "sim_timestamp": self.clock.now().isoformat(),
})
# No DB write — the Shipment Aggregator handles it
```

#### 2. Domain-owned table writes (direct)

Actors that own their own tables (not `shipments`) write directly. For example:
- **BrokerClaimHandler** writes to `broker_assignments` and `entry_filings` (broker-owned)
- **ComplianceEngine** writes to `restricted_parties` scan results (compliance-owned)

These direct writes are within bounded context ownership boundaries. Cross-context state changes (e.g., "this shipment now has entry_number X") are communicated via events.

**Event durability:** Redis Streams with AOF persistence ensures events survive Redis restarts. If Redis is temporarily unavailable, actors retry event publishing with exponential backoff. The Aggregator's consumer group tracks its position — on recovery, it processes all missed events from the last acknowledged position.

**No outbox pattern needed.** In the single-writer model, the event is the primary record. If event publishing fails, the actor retries. There's no split-brain risk because actors don't write to PostgreSQL — only the Aggregator does, and it only writes what the event stream tells it to.

**Periodic reconciliation (safety net):** A background task runs every 5 minutes, scanning recently-updated shipments to verify their PostgreSQL state matches the last events in the stream. Any drift is corrected. This catches edge cases (e.g., Aggregator crashed between consuming an event and committing the projection).

---

## 8. Consistency Model

### 8.1 Source of Truth

**The event stream (Redis Streams) is the source of truth.** All state in the system is derived from the event stream:

| Store | Role | Updated By | Relationship to Event Stream |
|-------|------|------------|------------------------------|
| **Redis Streams** | **Source of truth** | Actors (event emitters) | IS the source of truth |
| PostgreSQL `shipments` | Projection (queryable state) | Shipment Aggregator (Section 5.4) | Derived from event stream |
| Redis read stores | Projection (fast API serving) | Dashboard Projector, Shipment Projector | Derived from event stream |
| PostgreSQL `broker_assignments`, `entry_filings`, etc. | Domain-owned state | Respective domain actors (direct writes) | Domain-internal, not event-sourced |

**Why PostgreSQL `shipments` is a projection, not the source of truth:**
- If the shipment record is corrupted or lost, it can be **reconstructed by replaying the event stream** from the Aggregator's consumer group offset
- "Why is this shipment in this state?" is answered by reading the event stream history, not by guessing which actor last touched the DB record
- The Aggregator writes to PostgreSQL purely for queryability and UI serving — the authoritative record is the stream

**Durability of the event stream:**
- **Redis AOF persistence** (Append-Only File) ensures events survive Redis restarts. AOF fsync policy should be `everysec` (default) — at most 1 second of events lost on crash.
- **Stream maxlen trimming** (Section 4.4) means old events are eventually pruned. For permanent audit trail, the Shipment Aggregator maintains the `shipments.events` JSONB column as a permanent timeline. Events older than `maxlen` can be reconstructed from this timeline.
- **Consumer group offsets** are persisted in Redis. On recovery, each consumer (including the Aggregator) resumes from its last acknowledged position.

### 8.2 Eventual Consistency Budget

| Projection | Expected Lag | Max Acceptable Lag | Recovery Mechanism |
|---|---|---|---|
| PostgreSQL shipment record | < 200ms | 5 seconds | Aggregator catch-up + periodic reconciliation (5 min) |
| Dashboard aggregates | < 100ms | 5 seconds | Dashboard Projector catch-up + periodic reconciliation (5 min) |
| Shipment list (Redis) | < 100ms | 2 seconds | Shipment Projector catch-up + startup seeding |
| Shipment detail cache | < 200ms | 30s (TTL expiry) | Cache invalidation on event + miss-through to PostgreSQL |

**Key difference from previous design:** The PostgreSQL shipment record now has non-zero lag (< 200ms typical) because it's updated by the Aggregator asynchronously. In the old multi-writer model, the DB was updated synchronously by each actor. The 200ms lag is invisible to users — the SSE dashboard pushes updates every 2-5 seconds, and the Aggregator typically processes events within milliseconds.

### 8.3 Handling the UI

The UI already handles eventual consistency gracefully because the SSE dashboard stream pushes updates to the frontend:

```python
# Existing pattern in coordinator.py — _dashboard_loop
# Instead of running aggregate_dashboard() SQL queries, read from Redis
async def _dashboard_loop(self):
    while self._running:
        data = await self.redis.hgetall("read:dashboard:platform")
        dashboard = format_dashboard(data, self.clock)
        for subscriber in self._sse_subscribers:
            await subscriber.put(dashboard)
        await asyncio.sleep(self._dashboard_interval)
```

The frontend receives dashboard updates via SSE every 2-5 seconds. With event-driven read stores, the data is fresher (updated on every event, not every 5-second SQL cycle) and the SSE push is cheaper (Redis read, not 12-15 SQL queries).

No version vectors, no optimistic UI, no push invalidation needed. The existing SSE pattern is sufficient.

### 8.4 Aggregator Failure and Recovery

The Shipment Aggregator is a critical component. If it fails:

1. **Events continue flowing** — actors emit events normally. Events queue in Redis Streams.
2. **Redis read stores remain current** — Dashboard Projector and Shipment Projector are independent consumers, unaffected by Aggregator failure.
3. **PostgreSQL shipment records become stale** — they reflect the state as of the last Aggregator processing. UI may show slightly outdated data for shipment detail views (which read from PostgreSQL via cache miss).
4. **Recovery is automatic** — when the Aggregator restarts, it resumes from its last acknowledged event in each consumer group. It processes the backlog at full speed (200-500 events/second). Typical catch-up time for a 1-minute outage: ~10-30 seconds.

**Monitoring:** Alert if any Aggregator consumer group lag exceeds 1000 events or 30 seconds wall-clock time (see Section 10 observability).

---

## 9. Error Handling

### 9.1 Failed Event Processing

When an actor fails to process an event:

1. **No acknowledgment**: The event stays in the consumer group's Pending Entries List (PEL)
2. **Automatic redelivery**: Redis Streams redelivers unacknowledged events to the same consumer after the idle timeout (configurable, default 30s)
3. **Dead letter after 3 retries**: Events that fail 3+ times are moved to a dead letter stream and acknowledged (freeing the PEL). This prevents PEL accumulation from poison events.
4. **Structured logging**: Every failure is logged with full context (actor, event_type, event_id, correlation_id, error, delivery_count)
5. **State machine safety net**: The Shipment Aggregator's `validate_transition()` guard prevents invalid state changes, so retried events that arrive after the shipment has already moved forward are safe no-ops (see Section 5.7 Idempotency)

```python
async def _handle_processing_error(self, stream, group, event_id, event, exc):
    # Check delivery count via XPENDING
    pending_info = await self.event_bus.get_pending_info(stream, group, event_id)
    delivery_count = pending_info.get("delivery_count", 1) if pending_info else 1

    if delivery_count >= 3:
        # Move to dead letter stream — prevents PEL accumulation
        await self.event_bus.redis.xadd(f"{stream}:dead-letters", {
            "original_event_id": event.get("event_id", ""),
            "event_type": event.get("event_type", ""),
            "correlation_id": event.get("correlation_id", ""),
            "actor": self.actor_id,
            "error": str(exc),
            "delivery_count": str(delivery_count),
            "data": json.dumps(event),
        })
        await self.event_bus.ack(stream, group, event_id)
        logger.error(
            "event_dead_lettered",
            actor=self.actor_id,
            event_type=event.get("event_type"),
            correlation_id=event.get("correlation_id"),
            delivery_count=delivery_count,
            error=str(exc),
        )
    else:
        logger.warning(
            "event_processing_failed",
            actor=self.actor_id,
            event_type=event.get("event_type"),
            event_id=event.get("event_id"),
            correlation_id=event.get("correlation_id"),
            stream=stream,
            delivery_count=delivery_count,
            error=str(exc),
        )
        # Don't ack — Redis Streams will redeliver after idle timeout
        # State machine guards prevent duplicate processing on retry
```

Dead letter streams are trimmed to 1000 entries. The `/health` endpoint reports dead letter counts (see Section 9.2).

### 9.2 Monitoring Unprocessed Events

Operational visibility via a simple health-check extension:

```python
@app.get("/health")
async def health_check(request: Request):
    status = {"api": "ok", ...}

    # Event processing health — pending events per consumer group
    for stream in STREAM_CONFIG:
        for group in await redis.xinfo_groups(stream):
            pending = group["pending"]
            if pending > 50:
                status[f"stream:{stream}:{group['name']}"] = f"behind:{pending}"
            else:
                status[f"stream:{stream}:{group['name']}"] = "ok"

        # Dead letter count
        try:
            dl_len = await redis.xlen(f"{stream}:dead-letters")
            if dl_len > 0:
                status[f"{stream}:dead-letters"] = dl_len
        except Exception:
            pass  # Stream may not exist yet

    return status
```

### 9.3 Redis Unavailability

Redis is already a critical dependency (EventBus, dashboard caching, SSE). If Redis is down:
- The simulation stops (actors can't publish events)
- API reads fall through to PostgreSQL (detail cache miss)
- The health endpoint reports Redis as unavailable

This is the same behavior as today. The event-driven architecture does not make Redis MORE critical than it already is.

---

## 10. Connection Pool Separation (Phase 0 — Stepping Stone)

### 10.1 Dedicated Pools

Split the single 30-connection pool into isolated pools as the first step toward event-driven:

```python
# Actor pool — used by simulation actors for writes
actor_engine = create_async_engine(
    DATABASE_URL,
    pool_size=10,
    max_overflow=5,
    pool_timeout=10,    # Actors can queue
)

# API pool — used by HTTP endpoints
api_engine = create_async_engine(
    DATABASE_URL,
    pool_size=15,
    max_overflow=5,
    pool_timeout=3,     # API fails fast
)
```

**Why this is a stepping stone, not a standalone fix:** Pool separation solves connection starvation but NOT the correctness problems (silent SKIP LOCKED gaps, JSONB mutation races, unbounded batch locking). These are solved by the event-driven migration in subsequent phases.

### 10.2 Connection Usage in Event-Driven Model

After full migration, actor DB usage drops dramatically:
- **Before**: 19 actors each hold a connection for their entire tick (15-60s)
- **After**: Event-driven actors hold connections for ~5ms per single-row update

The actor pool effectively becomes a burst-capacity pool for time-dependent actors (carrier, resolution, pga, cage, demurrage) that still do periodic checks.

---

## 11. Systemic Issues and Solutions

The feasibility check (`architecture-feasibility-check.md`) identified four systemic issues that affect multiple actors. Each requires an architectural solution, not a per-actor workaround.

### 11.1 JSONB `events` Append — Eliminated by Single-Writer Model

**Severity: RESOLVED — no longer applicable**

The original concern: In the event-triggered CRUD model, a single `ShipmentArrivedAtCustoms` event would go to customs, PGA, financial, documents, terminal, and broker_sim via separate consumer groups. If they all processed concurrently and each did read-modify-write on `shipment.events`, last-write-wins — events would get lost.

**This problem cannot exist in the single-writer model.** The Shipment Aggregator (Section 5.4) is the ONLY component that writes to `shipment.events`. It consumes events from all 6 streams and appends timeline entries sequentially. There is no concurrent write, no race condition, no need for atomic JSONB append.

**Migration note:** During the transition period (Phase 0 through Phase 2a), some actors may still call the legacy `transition()` function which appends to `shipment.events`. During this period, use the atomic PostgreSQL array append (`events || $1::jsonb`) as a safety net:

```python
# TRANSITIONAL ONLY — used during Phase 0-2a migration
# Remove once all actors are event-driven and the Aggregator is live
from sqlalchemy import text
await session.execute(
    text("""
        UPDATE shipments
        SET events = COALESCE(events, '[]'::jsonb) || :new_event::jsonb
        WHERE id = :id
    """),
    {"id": str(shipment.id), "new_event": json.dumps([new_event])}
)
```

**Post-migration (Phase 2b+):** The atomic append is removed. The `transition()` function is retired (replaced by `validate_transition()` which is a pure function with no side effects). The Aggregator is the sole writer to `shipment.events`.

**Long-term improvement:** The `events` JSONB column could be replaced with a dedicated `shipment_events` table with `(shipment_id, timestamp, event_type, actor, data)` for better queryability and to avoid growing JSONB array performance degradation. This is straightforward because the Aggregator is the single writer — only one component needs to change.

### 11.2 Buffered Consumer Pattern (Consolidator)

The consolidator groups N shipments by `(transport_mode, origin, destination, carrier)` and creates a consolidation when the group threshold is met. In event-driven mode, it receives one `ShipmentInTransit` event at a time — it can't group in a single handler invocation.

**Solution: Event accumulation + periodic flush**

```python
class ConsolidatorActor(EventDrivenActor):
    """Buffered consumer: accumulates candidates, flushes when group threshold met."""

    subscriptions = [
        ("stream:shipment-lifecycle", "cg:consolidator-actor", ["ShipmentInTransit"]),
    ]

    def __init__(self, ...):
        super().__init__(...)
        # Accumulation buffer: group_key -> [shipment_states]
        self._candidates: dict[str, list[dict]] = defaultdict(list)

    async def handle_event(self, event: dict) -> None:
        """Accumulate candidates by group key."""
        shipment = event["payload"]
        if shipment.get("consolidation_id"):
            return  # Already consolidated

        key = f"{shipment['transport_mode']}:{shipment['origin']}:{shipment['destination']}:{shipment.get('carrier', '')}"
        self._candidates[key].append(shipment)

    async def tick(self) -> None:
        """Periodic check: flush groups that have reached threshold."""
        for key, candidates in list(self._candidates.items()):
            if len(candidates) >= self.config.group_threshold:
                await self._create_consolidation(candidates)
                del self._candidates[key]

    async def _create_consolidation(self, candidates: list[dict]) -> None:
        """Multi-table atomic write: Consolidation + N × Shipment."""
        async with self.get_session() as session:
            async with session.begin():
                consolidation = Consolidation(...)
                session.add(consolidation)
                await session.flush()  # Get ID
                for c in candidates:
                    shipment = await session.get(Shipment, c["id"])
                    if shipment and not shipment.consolidation_id:
                        shipment.consolidation_id = consolidation.id
                        # Atomic JSONB append for references and events
                        ...

        await self.event_bus.publish("stream:shipment-lifecycle",
                                      "ConsolidationClosed", ...)
```

**Key behaviors:**
- Events are accumulated in memory (the buffer)
- `tick()` checks groups periodically (e.g., every 30s)
- Multi-table atomicity is preserved within `_create_consolidation`
- On restart, the buffer is lost — but the tick would find ungrouped `in_transit` shipments naturally from incoming events

### 11.3 Lazy Computation (Cage Dwell & Demurrage)

Cage dwell tracking and demurrage accrual currently update every tick — `_process_dwell` recalculates for ALL caged shipments, and demurrage recalculates charges for ALL ocean shipments at customs. This is inherently poll-based and expensive.

**Solution: Store parameters at event time, compute on demand**

```python
# ON INTAKE (event handler):
async def handle_cage_intake(self, event: dict) -> None:
    shipment = event["payload"]
    async with self.get_session() as session:
        async with session.begin():
            s = await session.get(Shipment, shipment["id"])
            # Store computation parameters — NOT the computed values
            s.cage_status = {
                "status": "caged",
                "intake_time": self.clock.now().isoformat(),
                "storage_rate_per_day": self._get_rate(shipment),
                "go_days": 30,  # General Order deadline
                "free_days": self._get_free_days(shipment),
            }

# ON READ (API endpoint or dashboard projector):
def compute_cage_metrics(cage_status: dict, sim_now: datetime) -> dict:
    """Pure function: compute current dwell/cost from stored parameters."""
    intake = datetime.fromisoformat(cage_status["intake_time"])
    dwell_days = (sim_now - intake).total_seconds() / 86400
    free_days = cage_status.get("free_days", 4)
    rate = cage_status.get("storage_rate_per_day", 150)

    billable_days = max(0, dwell_days - free_days)
    storage_cost = billable_days * rate
    go_days_remaining = cage_status.get("go_days", 30) - dwell_days

    return {
        "dwell_days": round(dwell_days, 2),
        "storage_cost_accrued": round(storage_cost, 2),
        "go_days_remaining": round(max(0, go_days_remaining), 1),
    }
```

**Benefits:**
- Eliminates per-tick dwell/demurrage updates for ALL caged/ocean shipments
- Values are always perfectly up-to-date (computed at read time, not stale from last tick)
- Cage actor only needs a timer for GO deadline warnings (`go_days_remaining < 5`)
- Demurrage actor only handles `ShipmentArrivedAtCustoms` (stores parameters) — no timer needed

**Dashboard projector:** Calls `compute_cage_metrics()` when projecting cage-related events, using `sim_now` from the event's `sim_timestamp`.

### 11.4 Persisted Duration Calculations

**Severity: MEDIUM — affects carrier, resolution, PGA, broker_sim CF-28 response**

Actors with time-dependent logic use `self.rng.random()` to generate durations (transit days, review days, response delays). On re-evaluation (next tick, or after restart), the actor gets a **different random number**, causing non-deterministic behavior.

**Solution: Compute once, persist to shipment record**

When a time-dependent actor first processes a shipment, it:
1. Generates the random duration
2. Persists it to `shipment.references` (e.g., `references.transit_days`, `references.review_days`)
3. On subsequent ticks, reads the persisted value instead of re-generating

```python
# In carrier handle_event for ShipmentInTransit:
transit_days = self._calculate_transit_time(shipment_state)  # Uses rng

# Persist to references (atomic JSONB merge)
await session.execute(
    text("""
        UPDATE shipments
        SET references = COALESCE(references, '{}'::jsonb) || :update::jsonb
        WHERE id = :id
    """),
    {"id": shipment_state["id"], "update": json.dumps({"transit_days": transit_days})}
)

# In carrier tick():
for sid, tracking in self._tracking.items():
    transit_days = tracking["transit_days"]  # Persisted value, NOT re-rolled
    ...
```

**Affected actors and their persisted fields:**

| Actor | Persisted Field | Where Stored |
|-------|----------------|-------------|
| carrier | `transit_days`, `inland_days` | `references.transit_days`, `references.inland_days` |
| resolution | `review_days`, `outcome_roll` | `references.review_days`, `references.resolution_roll` |
| pga | `review_days` (per agency) | `references.pga_review_days` |
| broker_sim (CF-28) | `response_delay_days` | `entry_filings.references.cf28_response_delay` |

### 11.5 Customs Backlog Detection via Read Model

The customs actor currently counts `at_customs` shipments (`SELECT count(*) WHERE status='at_customs'`) to detect backlogs and adjust STP processing times. In event-driven mode, each event is processed independently — the actor doesn't have queue visibility.

**Solution: Read from the dashboard read model**

```python
async def _get_queue_depth(self) -> int:
    """Get current at_customs queue depth from read model."""
    count = await self.redis.zscore("read:shipments:by_status:at_customs", "count")
    if count is None:
        # Fallback to sorted set cardinality
        count = await self.redis.zcard("read:shipments:by_status:at_customs")
    return int(count or 0)
```

This is a Redis read (sub-millisecond), not a SQL COUNT query. The read model is updated by the Shipment Projector on every status change event. The value may be slightly stale (eventually consistent), but backlog detection is cosmetic — it only affects STP description timing, not correctness.

### 11.6 Multi-Table Atomicity

Three actors write to multiple tables in a single transaction: `broker_sim` (4-5 tables), `cbp_authority` (2 tables), `consolidator` (2 tables).

**Approach: Keep transactions, separate event publish**

Event-driven handlers still use database transactions for multi-table writes. The pattern:

```python
async def handle_event(self, event: dict) -> None:
    # Multi-table atomic write
    async with self.get_session() as session:
        async with session.begin():
            shipment = await session.get(Shipment, event["payload"]["id"])
            entry = EntryFiling(...)
            assignment = BrokerAssignment(...)
            session.add(entry)
            session.add(assignment)
            shipment.entry_number = entry.filing_number
            # All committed atomically

    # Event publish AFTER commit (not inside transaction)
    await self.event_bus.publish(...)
```

If the event publish fails after a successful commit, the DB state is correct and the periodic reconciliation catches up the read stores. This is acceptable for a simulation platform — no outbox pattern needed.

### 11.7 Enrichment Actor Output Events

Actors that enrich shipments without changing status (compliance_engine, financial, documents, terminal) currently produce no status events. In event-driven mode, they should publish domain-specific events so the dashboard projector can update read stores:

| Actor | Output Event | Trigger |
|-------|-------------|---------|
| compliance_engine | `AnalysisGenerated` | After writing `analysis` JSONB |
| financial | `FeesCalculated` | After writing `financials` JSONB |
| documents | `DocumentDiscrepancyFound` | After detecting discrepancy |
| terminal | `BerthDelayReported` | After injecting berth delay event |

These events flow to `stream:compliance` and are consumed only by the dashboard projector. They don't trigger other actors — they're informational.

### 11.8 Sequential Processing Anti-Pattern — Resolved by Proactive Evaluation

**Severity: ARCHITECTURAL — fundamental shift in how services are triggered**

The codebase audit (see `architecture-current-state.md` Section 10) revealed that 5 of the 19 actors follow a **sequential processing anti-pattern**: they wait for a specific status milestone (`at_customs`) before acting, even though their actual data prerequisites are available much earlier.

**Impact of the anti-pattern:**
- Declarations are filed **after** physical arrival instead of during transit → adds hours/days to clearance time
- Brokers aren't assigned until cargo arrives → no prep work, reactive scramble
- PGA requirements discovered only at the border → surprise holds that could have been anticipated
- Documents validated only at customs → discrepancies caught too late to fix without delay
- Fees calculated only after clearance → no pre-payment opportunity

**Solution: Proactive Evaluator Pattern (Section 5.5)**

Note the distinction between **proactive evaluators** and **reactive responders** (Section 5.3). The broker is the proactive evaluator that decides when to file declarations; the customs authority is a reactive responder that adjudicates submitted declarations. The anti-pattern is the broker waiting until `at_customs` to file — not customs waiting for submissions (which is correct behavior for a government authority).

Each proactive evaluator defines readiness criteria based on **data prerequisites**, not status:

```python
# ANTI-PATTERN (sequential):
select(Shipment).where(Shipment.status == "at_customs")  # Wait for physical arrival

# CORRECT (proactive):
# Subscribe to ClassificationCompleted, ShipmentCreated, etc.
# Evaluate: do I have HS code? ✅ Origin? ✅ Value? ✅ → Act NOW
def _is_ready(self, state: dict) -> bool:
    return all(state.get(f) for f in self.READINESS_FIELDS)
```

The state machine's `at_customs` state becomes **optional** — a shipment that has been fully pre-cleared can transition directly from `in_transit` → `cleared` when the carrier reports arrival. This new transition is added in the proactive model.

**Quantified benefit:** For a shipment with all data available at booking time (common for repeat shippers with known products), the proactive model enables:
- Declaration pre-filed **during transit** (days of lead time)
- Broker assigned and prep work started **at booking** (not at arrival)
- PGA requirements known **before departure** (no arrival surprises)
- Documents validated **pre-departure** (time to fix discrepancies)
- On arrival: **instant clearance** if pre-clearance passed — skip `at_customs` entirely

---

## 12. Testability

### 12.1 Testing Strategy

The architecture preserves the existing test suite (197 passing tests) while enabling clean testing of event wiring.

**Layer 1 — Business logic (existing tests, unchanged):**

```python
# These tests call per-shipment processing methods directly
# They don't know about events, streams, or consumer groups
async def test_customs_screens_shipment():
    actor = CustomsActor(...)
    shipment = create_test_shipment(status="at_customs")
    outcome = actor._screen_and_route_from_state(shipment_to_dict(shipment), config, sim_now)
    assert outcome.new_status in ("cleared", "held", "inspection")
```

**Layer 2 — Event handling (new tests):**

```python
async def test_customs_handles_arrived_event():
    actor = CustomsActor(...)
    event = make_event("ShipmentArrivedAtCustoms", payload={...})

    with mock_db_session() as session:
        await actor.handle_event(event)

    # Verify single-row update was called
    session.get.assert_called_once_with(Shipment, event["payload"]["id"])
    # Verify outcome event was published
    actor.event_bus.publish.assert_called_once()
```

**Layer 3 — Integration (new tests):**

```python
async def test_shipment_flows_through_customs_pipeline():
    """End-to-end: ShipmentCreated -> InTransit -> ArrivedAtCustoms -> Cleared."""
    bus = TestEventBus()  # In-memory implementation of EventStreamBus
    customs = CustomsActor(event_bus=bus, ...)
    carrier = CarrierActor(event_bus=bus, ...)

    # Publish initial event
    await bus.publish("stream:shipment-lifecycle", "ShipmentCreated", {...})

    # Process through the pipeline
    await carrier.process_pending_events()
    await customs.process_pending_events()

    # Verify final state
    assert bus.published_events[-1]["event_type"] == "ShipmentCleared"
```

### 12.2 TestEventBus

A test implementation of the event bus that operates in-memory:

```python
class TestEventBus:
    """In-memory event bus for unit/integration tests. No Redis needed."""

    def __init__(self):
        self.streams: dict[str, list] = defaultdict(list)
        self.consumer_positions: dict[str, int] = defaultdict(int)

    async def publish(self, stream, event_type, payload):
        event = {"event_type": event_type, "payload": payload, ...}
        self.streams[stream].append(event)

    async def read_stream(self, stream, group, consumer, count=10, block_ms=0):
        pos = self.consumer_positions[f"{stream}:{group}"]
        events = self.streams[stream][pos:pos+count]
        return [(str(i), e) for i, e in enumerate(events, start=pos)]

    async def ack(self, stream, group, event_id):
        self.consumer_positions[f"{stream}:{group}"] = int(event_id) + 1
```

---

## 13. Extensibility

### 13.1 Adding a New Jurisdiction

When a new jurisdiction (e.g., AU — Australia) is added:

**Current architecture (poll-and-lock):**
1. Add AU state machine transitions
2. Modify customs actor to handle AU-specific screening
3. **Risk**: New AU screening logic runs inside the same `at_customs` poll loop, adding to contention
4. Must test for contention with all existing actors

**Event-driven architecture:**
1. Add AU state machine transitions (new file: `state_machines/au.py`)
2. Define AU-specific events in the event catalog (Section 3.3.x) — e.g., `ABFDeclarationLodged`, `ABFAssessed`, `ABFReleased`
3. Add AU-specific metadata fields to event payloads (e.g., `abf_reference`, `aqis_status`)
4. Add AU-specific screening logic to customs `handle_event()` (or create a dedicated AU customs actor)
5. If a new actor: create consumer group, subscribe to `ShipmentArrivedAtCustoms`
6. **Zero contention risk**: Consumer groups are independent
7. No need to test for contention — it can't happen

### 13.2 Adding a New Actor

**Current:** Must choose which statuses to poll, add FOR UPDATE queries, coordinate with existing actors, tune LIMIT.
**Event-driven:** Subscribe to relevant events. Done. The consumer group guarantees delivery. No coordination needed.

### 13.3 Adding a New UI Surface

**Current:** Add API endpoints that query PostgreSQL, competing with actors for connections.
**Event-driven:** Add API endpoints that read from Redis. Add a projector if the existing read stores don't cover the new surface's data needs.

---

## 14. Observability

### 14.1 Structured Logging

All event processing includes correlation IDs:

```python
logger.info("event_processed",
    actor=self.actor_id,
    event_type=event["event_type"],
    event_id=event["event_id"],
    correlation_id=event["correlation_id"],
    processing_ms=elapsed_ms,
)
```

**Debugging "why didn't this shipment clear?":**
1. `grep correlation_id=<shipment-id>` in structured logs → see full event chain
2. If event is stuck: `XPENDING stream:customs-processing cg:customs-actor` → shows unacknowledged events with delivery count and idle time
3. Two steps. Clear chain of custody.

### 14.2 Metrics (Minimal Set)

| Metric | Source | Alert Threshold |
|---|---|---|
| `event_processing_latency_ms` | Actor handler timing | p99 > 500ms |
| `stream_pending_count` | `XPENDING` per group | > 50 events |
| `dead_letter_count` | `XLEN` on `*:dead-letters` | > 0 (any dead letter is worth investigating) |
| `db_pool_active_connections` | SQLAlchemy pool stats | > 80% capacity |
| `redis_memory_usage_mb` | Redis INFO | > 200MB |
| `read_store_last_updated` | Redis hash field | > 30s ago |

---

## 15. Implementation Phases

### Phase 0: Foundation + Systemic Fixes (2-3 days)

**Goal:** Separate connection pools, prepare event infrastructure, fix systemic issues that affect all actors.

- Split single engine into `actor_engine` + `api_engine` in `main.py`
- Implement `EventStreamBus` class (Redis Streams wrapper with publish, read_stream, ack, create_group, dead letter support)
- Create consumer groups for all streams
- Wire `EventStreamBus` into coordinator alongside existing `EventBus`
- **Transitional JSONB append fix** (Section 11.1): Update `transition()` and all manual appenders to use atomic `events || $1::jsonb` — this is a safety net during migration while some actors still use the legacy pattern. Removed in Phase 3 when the Aggregator becomes the sole writer.
- Implement `ProcessingOutcome` dataclass (Section 5.2)
- Implement `ProactiveEvaluatorMixin` with readiness accumulator (Section 5.5)
- Implement `ShipmentEventPayload` with `from_shipment()` and `from_event()` constructors (Section 3.9)
- Split `transition()` into: `validate_transition()` (pure function, no side effects) + legacy `transition()` (backward-compatible wrapper for unmigrated actors)
- Add `ClassificationCompleted` event to preclearance actor output (enables proactive downstream evaluation)
- **Add state machine transition: `in_transit → cleared`** (allowed for `customs` when pre-clearance succeeds)
- **Deploy Shipment Aggregator in shadow mode** (Section 5.4) — consumes events and validates projections against DB state, but does NOT write to the live `shipments` table yet
- All existing behavior unchanged — actors still poll, but infrastructure + patterns are ready

### Phase 1: Highest-Value Actor Migration (3-4 days)

**Goal:** Migrate the highest-contention actors + cleanest conversions. Ordered by feasibility check priority.

**Phase 1a — Proactive evaluators + reactive responders, highest contention (2 days):**
- Implement `EventDrivenActor` base class with dead letter handling
- Implement `ProactiveEvaluatorMixin` with readiness accumulator pattern (Section 5.5.1)
- Migrate `broker_sim` declaration handler — **proactive evaluator** subscribing to `ShipmentCreated`, `ClassificationCompleted`. Pre-files declarations during transit. This is the key architectural shift: the broker decides when to file. Eliminates #1 contention hotspot. (Section 5.5)
- Migrate `customs` → `customs_authority` — **reactive responder** subscribing to `DeclarationSubmitted` (adjudicates broker-filed declarations) and `ShipmentArrivedWithoutClearance` (exception path — holds by default). Does NOT proactively seek work. (Section 5.2, 5.3)
- Migrate `compliance_engine` — **proactive evaluator** subscribing to `ShipmentCreated`, `ClassificationCompleted`, `RegulatorySignalDetected`. Re-screens on regulatory changes (not just at creation). Fixes existing race condition.
- Migrate `financial` — **proactive evaluator** subscribing to `ClassificationCompleted` (pre-calculates fees as soon as tariff data available), `ShipmentCleared`, `CageReleased`
- Migrate `documents` — **proactive evaluator** subscribing to `ShipmentCreated`, `DocumentUploaded`, `ClassificationCompleted`. Validates pre-departure.
- Migrate `shipper_response` — event-reactive (eliminates full table scan)
- Add `ShipmentCreated` event publishing to `shipper` actor

**Phase 1b — Moderate adaptation actors (2 days):**
- Migrate `pga` — **proactive evaluator** subscribing to `ClassificationCompleted` (determines PGA requirements as soon as HS code known), hybrid with persisted review_days (Section 11.4)
- Migrate `broker_sim` claim handler — **proactive evaluator** subscribing to `ShipmentCreated` (assigns broker and begins prep during transit, not at customs arrival)
- Migrate `resolution` — hybrid actor with persisted review_days
- Migrate `cage` — lazy dwell computation (Section 11.3), periodic GO deadline check
- Migrate `terminal`, `preclearance`, `isf` — clean conversions

### Phase 2: Complex Actors + Read Models (3-4 days)

**Goal:** Rework the 2 hardest actors, build read model projections.

**Phase 2a — Complex actor rework (2 days):**
- Decompose `broker_sim` into 3 handlers (Section 5.8): claim, progress, CF-28 response
- Migrate `cbp_authority` — multi-table writes, 4 subscription types
- Migrate `carrier` — hybrid with persisted transit_days, consolidation side-effects extracted
- Implement buffered consumer for `consolidator` (Section 11.2)

**Phase 2b — Read model projections (2 days):**
- Implement `DashboardProjector` consuming all 6 streams (with lazy cage/demurrage computation)
- Implement `ShipmentProjector` maintaining sorted sets + hashes
- Startup seeding logic
- Periodic reconciliation (5-minute full SQL as safety net)
- Migrate `GET /api/platform/aggregates` and SSE dashboard to Redis reads
- Migrate `GET /api/shipments` to Redis reads

### Phase 3: Cleanup + Polish (1-2 days)

**Goal:** Convert remaining actors, full test suite, decommission old patterns.

- Convert `demurrage` to lazy computation model (event-triggered parameter storage, no timer)
- Reclassify `exceptions` and `disruption` to hybrid (event-based shipment discovery, timer-based injection)
- `TestEventBus` implementation for integration tests
- Decommission old `EventBus` (Redis Pub/Sub)
- Remove all `FOR UPDATE SKIP LOCKED` from migrated actors
- Verify all 197 existing tests still pass
- Add event-driven integration tests (Layer 2 and Layer 3)

### Total: 11-12 days

**Risk-adjusted estimate.** Phase 0 and Phase 1a can be delivered in the first sprint week. Phase 1b through Phase 3 in the second week. The carrier actor (Phase 2a) is the highest-risk item — intermediate regulatory events and consolidation advancement require careful extraction. Budget 1-2 days for carrier alone.

---

## 16. Appendix: Data Flow Diagrams

### Current Flow (Synchronous Poll-and-Lock)

```
Shipper --> [INSERT shipments] --> PostgreSQL
                                       ^  FOR UPDATE SKIP LOCKED (every tick)
                                       |  (19 actors, 35 queries, unbounded locks)
Carrier <--> PostgreSQL <--> Customs <--> PGA <--> Broker <--> Resolution
                                       ^  SELECT (every request)
                                       |  (competing for same 30-connection pool)
API Endpoints <--> PostgreSQL <-- Dashboard Aggregator (12-15 queries/5s)
```

### Target Flow (Event-Driven with Proactive Evaluation)

```
Shipper --> [INSERT + publish ShipmentCreated] --> PostgreSQL + Redis Stream
                                                          |
                                         Redis Streams --> Consumer Groups
                                                          |
              ┌───────────────────────────────────────────┤
              │                                           │
     ShipmentCreated consumed by:                ClassificationCompleted consumed by:
     • Preclearance (immediate)                  • Broker (readiness eval → pre-file declaration)
     • Broker (readiness eval → assign)          • PGA (readiness eval → determine reqs)
     • Compliance (readiness eval → screen)      • Financial (readiness eval → pre-calc)
     • Documents (readiness eval → validate)     • Documents (readiness eval → validate)
              │                                           │
              └────────── Proactive evaluators ──────────┘
                          accumulate readiness state
                                      │
                    Broker files declaration (DeclarationSubmitted)
                                      │
                    Customs Authority adjudicates (reactive responder)
                    • Accepts → DeclarationAccepted
                    • Rejects → DeclarationRejected
                                      │
                          ShipmentArrivedAtCustoms:
                          • Declaration accepted? → in_transit → cleared (skip at_customs)
                          • No declaration? → ShipmentArrivedWithoutClearance → hold
                                      │
                          [outcome events → Shipment Aggregator → DB projection]
                                      │
                          Dashboard Projector --> Redis Read Stores
                                                        |
                                                   API Endpoints
                                                   (zero SQL queries)
```

**Key differences:**
1. **Proactive evaluation + reactive response**: The broker (proactive evaluator) pre-files declarations during transit; the customs authority (reactive responder) adjudicates submissions. Brokers assigned at booking, not at customs arrival
2. Actors emit outcome events — the **Shipment Aggregator is the sole writer** to the `shipments` table
3. Actors process one event at a time — no batch locking, no SKIP LOCKED, no contention
4. Every event is tracked: published, pending, acknowledged, or failed (full visibility)
5. New actors subscribe to events — contention is impossible by construction
6. Dashboard serves from pre-computed Redis data — zero SQL queries per request
7. The `at_customs` state is **optional** — pre-cleared shipments skip it entirely
