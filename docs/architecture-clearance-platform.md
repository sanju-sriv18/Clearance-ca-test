# Clearance Platform — Microservices Architecture

**Status:** Definitive Reference Document (v5, amended v5.1, v5.2)
**Date:** 2026-02-09
**Author:** Simon Kissler (simon.kissler@accenture.com)
**Co-Author:** Claude Code (Anthropic)
**Replaces:** All previous architecture documents (v1-v4)
**Review inputs:** Broker expert review, enterprise architect review, engine audit, jurisdiction deep-dive, final jurisdiction validation review
**v5.1 amendments (2026-02-09):** VAT/consumption tax fields on TariffCalculation, cascading duty calculation support (base_amount, computation_order, includes_prior), destination_subdivision on Shipment and TariffCalculation, jurisdiction_entry_type and clearance_channel on EntryFiling, catalog_registrations on Product, register_product on JurisdictionAdapter interface, deferred items appendix (Appendix C).
**v5.2 amendments (2026-02-09):** Export declarations as first-class filing entities. ExportDeclaration entity in Declaration Management domain with full jurisdiction-specific filing data, export checklists, and two-layer status model. Export filing workflows for all 7 target jurisdictions (US AES/EEI, EU ECS, BR DU-E, IN Shipping Bill, CN Single Window, MX pedimento A1, CA CAED). Export declaration NATS events (6 new subjects). Export adjudication events. Jurisdiction Adapter interface extended with export_filing_workflow, export_status_machine, export_document_requirements, export_checklist_generator, export_license_required, export_filing_required. Export API surface (9 endpoints). Export status machine mapping per jurisdiction. HU and Consolidation events consumed updated for export clearance flow. Appendix C.3 resolved.

---

## 1. Architecture Vision & Principles

### 1.1 Vision

The Clearance Platform is a **customs clearance intelligence system** built as **microservices organized by functional business domain**, connected by **NATS JetStream** (event backbone), with a **CQRS read/write split** (Redis for reads, domain services for writes).

A separate **Simulation Platform** sits on top, exercising the Clearance Platform through the same APIs that humans and AI agents use. The simulation is a consumer, not a component.

### 1.2 Core Principles

| # | Principle | Meaning |
|---|-----------|---------|
| 1 | **Domain-first design** | Services organized by business function. Each domain owns its state, logic, and events. Not by actor, screen, or technical concern. |
| 2 | **State-transfer events** | When a domain's key state object changes, it emits an event carrying the **full state snapshot**. Consumers never need to query the producer. |
| 3 | **Share-on-principle** | Events make data available by default. New intelligence service = subscribe to existing events. No new API needed. |
| 4 | **Actors are automation, not architecture** | Actors (simulation bots, AI agents) use the same APIs humans use. They sit on top, never inside. |
| 5 | **Intelligence as domains** | Intelligence capabilities that serve multiple consumers are **proper domains** with state, events, and lifecycle — not utility functions or floating engines. |
| 6 | **Intelligence flows via events** | Consumers receive intelligence by subscribing to events, not by calling APIs. Classification results flow to brokers via `ClassificationProduced` events, not via a classify API call. |
| 7 | **CQRS everywhere** | Writes → domain services (business logic → state change → event). Reads → Redis state cache (pre-computed). UI subscribes to NATS for real-time. |
| 8 | **APIs as MCP tools** | Every API endpoint is also an MCP tool for AI agents. |
| 9 | **Proactive clearance** | Intelligence services produce results as soon as data prerequisites are met. Brokers pre-file during transit. The happy path skips `at_customs` entirely. |

### 1.3 What This Is NOT

- **NOT** the previous Redis Streams + actor-centric design. That organized around 19 simulation actors.
- **NOT** a monolith with events bolted on. Each domain owns its data, logic, and event contracts.
- **NOT** simulation-aware. The clearance platform has zero knowledge of the simulation.
- **NOT** intelligence-as-utility. Classification, tariff, and compliance are proper domains with persistent state, not stateless computation functions.

---

## 2. Platform Topology

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SIMULATION PLATFORM                                 │
│  ┌─────────┐ ┌─────────┐ ┌──────────┐ ┌──────────┐ ┌────────────────────┐ │
│  │ Shipper │ │ Carrier │ │ Customs  │ │ Buyer    │ │ Simulation         │ │
│  │ Bot     │ │ Bot     │ │ Authority│ │ Bot      │ │ Coordinator        │ │
│  │         │ │         │ │ Bot      │ │          │ │ (clock, lifecycle) │ │
│  └────┬────┘ └────┬────┘ └────┬─────┘ └────┬─────┘ └────────┬───────────┘ │
│       │           │           │            │                 │             │
│       │      Uses platform APIs (REST/gRPC)│                 │             │
│       │    Subscribes to clearance.* (READ-ONLY)             │             │
│       │    Publishes to sim.* ONLY         │                 │             │
│       └───────────┴───────────┴────────────┴─────────────────┘             │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CLEARANCE PLATFORM                                   │
│                                                                             │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │                        API GATEWAY                                 │     │
│  │  Routing • Auth • Rate limiting • MCP tool registry • SSE proxy   │     │
│  └────────────────────────────────┬───────────────────────────────────┘     │
│                                   │                                         │
│  ┌──────────────────── FUNCTIONAL DOMAINS ────────────────────────┐        │
│  │                                                                 │        │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐   │        │
│  │  │ PRODUCT     │  │ TRADE       │  │ COMPLIANCE &        │   │        │
│  │  │ CATALOG     │  │ INTELLIGENCE│  │ SCREENING           │   │        │
│  │  │             │  │             │  │                     │   │        │
│  │  │ Products    │  │ Classific.  │  │ DPS/UFLPA/PGA      │   │        │
│  │  │ Desc Qual   │  │ Tariffs     │  │ Risk Scoring        │   │        │
│  │  │             │  │ FTA/Landed  │  │                     │   │        │
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘   │        │
│  │                                                                 │        │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐   │        │
│  │  │ ORDER       │  │ SHIPMENT    │  │ CARGO &             │   │        │
│  │  │ MANAGEMENT  │  │ LIFECYCLE   │  │ HANDLING UNITS      │   │        │
│  │  │             │  │             │  │                     │   │        │
│  │  │ Orders      │  │ Shipper     │  │ Physical units      │   │        │
│  │  │ Line Items  │  │ abstraction │  │ Customs states      │   │        │
│  │  │ Doc Reqs    │  │ HU project. │  │ Movement/Cage       │   │        │
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘   │        │
│  │                                                                 │        │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐   │        │
│  │  │ CONSOLIDATN │  │ DECLARATION │  │ CUSTOMS             │   │        │
│  │  │             │  │ MANAGEMENT  │  │ ADJUDICATION        │   │        │
│  │  │             │  │             │  │                     │   │        │
│  │  │ MAWB/MBL    │  │ Entries     │  │ Decisions           │   │        │
│  │  │ Nesting     │  │ Broker Asgn │  │ Reviews             │   │        │
│  │  │ Movement    │  │ Checklists  │  │ Exams               │   │        │
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘   │        │
│  │                                                                 │        │
│  │  ┌─────────────┐  ┌─────────────────────────────────────────┐   │        │
│  │  │ EXCEPTION   │  │ FINANCIAL SETTLEMENT                     │   │        │
│  │  │ MANAGEMENT  │  │                                          │   │        │
│  │  │             │  │ Duties/Fees • Demurrage • Bonds          │   │        │
│  │  │ Escalation  │  │ Drawback • Reconciliation                │   │        │
│  │  │ SLA Monitor │  │                                          │   │        │
│  │  │ Disruptions │  │                                          │   │        │
│  │  └─────────────┘  └─────────────────────────────────────────┘   │        │
│  │                                                                 │        │
│  │  ┌─────────────┐                                               │        │
│  │  │ PARTY       │                                               │        │
│  │  │ MANAGEMENT  │                                               │        │
│  │  │             │                                               │        │
│  │  │ Parties     │                                               │        │
│  │  │ POA / IOR   │                                               │        │
│  │  │ Trust Progs │                                               │        │
│  │  └─────────────┘                                               │        │
│  │                                                                 │        │
│  │  ┌─────────────┐  ┌─────────────────────────────────────────┐ │        │
│  │  │ REGULATORY  │  │ DOCUMENT MANAGEMENT                     │ │        │
│  │  │ INTELLIGENCE│  │ (Cross-cutting supporting domain)        │ │        │
│  │  │             │  │                                          │ │        │
│  │  │ Signals     │  │ Generation • Validation • Storage        │ │        │
│  │  │ Scenarios   │  │ Requirements • Discrepancy detection     │ │        │
│  │  └─────────────┘  └─────────────────────────────────────────┘ │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐      │
│  │                     NATS JetStream                                │      │
│  │  clearance.product.>       clearance.trade-intel.>                │      │
│  │  clearance.compliance.>   clearance.order.>                      │      │
│  │  clearance.shipment.>     clearance.handling-unit.>              │      │
│  │  clearance.consolidation.>  clearance.declaration.>              │      │
│  │  clearance.adjudication.>  clearance.exception.>                 │      │
│  │  clearance.financial.>   clearance.regulatory.>                  │      │
│  │  clearance.document.>   clearance.party.>                       │      │
│  └──────────────────────────────────────────────────────────────────┘      │
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐  ┌──────────┐        │
│  │  REDIS      │  │ POSTGRESQL  │  │    NATS      │  │ QDRANT   │        │
│  │  State Cache│  │ Per-domain  │  │  JetStream   │  │ Vector   │        │
│  │  (reads)    │  │ schemas     │  │  (events)    │  │ Intel    │        │
│  └─────────────┘  └─────────────┘  └──────────────┘  └──────────┘        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.1 Infrastructure Components

| Component | Technology | Role |
|-----------|------------|------|
| **Event Bus** | NATS JetStream | Persistent event streaming with replay, subject-based routing, consumer groups |
| **State Cache** | Redis | CQRS read side — listens to NATS events, maintains pre-computed state for fast reads |
| **Vector Store** | Qdrant | Intelligence knowledge base — embeddings for rulings, classifications, regulatory corpus |
| **Databases** | PostgreSQL (per-domain schemas) | Each domain owns its schema. Logical separation initially, physical if needed. |
| **API Gateway** | Caddy + routing | TLS, auth, rate limiting, MCP tool registry, SSE/WebSocket proxy for NATS |

---

## 3. Functional Domains

**15 domains** organized by business function:

| # | Domain | Business Purpose | Key State Object | Owner |
|---|--------|-----------------|-------------------|-------|
| 1 | Product Catalog | Master catalog of products with trade-relevant attributes | `Product` | Shipper |
| 2 | Trade Intelligence | Classification, tariff calculation, FTA, landed cost | `Classification`, `TariffCalculation`, `FTADetermination` | Platform |
| 3 | Compliance & Screening | DPS, UFLPA, PGA screening, risk assessment | `ComplianceResult` | Platform |
| 4 | Order Management | Purchase orders from creation to shipment | `Order` | Shipper |
| 5 | Shipment Lifecycle | Shipper-facing convenience abstraction — groups goods from origin to destination | `Shipment` | Platform |
| 6 | Cargo & Handling Units | Physical cargo units — packages, pallets, containers — as first-class entities | `HandlingUnit` | Platform |
| 7 | Consolidation | Grouping of handling units and consolidations for transport, with own customs states | `Consolidation` | Platform |
| 8 | Declaration Management | Broker's clearance workflow — entries, checklists, filing | `Declaration` | Broker |
| 9 | Customs Adjudication | Authority processing — receive, review, decide | `AdjudicationDecision` | Authority |
| 10 | Exception Management | Cross-domain coordination/escalation for unresolved issues spanning multiple domains, SLA monitoring | `Exception` | Platform |
| 10a | Supply Chain Disruptions | Port closures, weather, strikes, system outages — thin sub-domain tracking disruption events | `Disruption` | Platform |
| 11 | Financial Settlement | Duties, fees, demurrage, bonds, settlement | `FinancialRecord` | Platform |
| 12 | Regulatory Intelligence | Regulatory change monitoring, scenario modeling | `RegulatorySignal` | Platform |
| 13 | Document Management | Generation, validation, storage, requirements | `Document` | Cross-cutting |
| 14 | Party Management | Party identity, registrations, trust programs, power of attorney, importer of record | `Party`, `PowerOfAttorney`, `ImporterOfRecord` | Platform |

**Note on Exception Management split (v5):** Holds are managed by the originating domain (Adjudication creates holds on HU; Declaration Management/broker handles resolution). Cage/detention stays on the HU entity. Supply chain disruptions are tracked in a separate thin sub-domain (10a). Exception Management (10) becomes the coordination/escalation layer for unresolved issues spanning multiple domains and SLA monitoring — it is NOT a state-owning domain for holds.

---

### 3.1 Product Catalog Domain

**Business purpose:** Maintain the master catalog of products with trade-relevant attributes. Products are **what the shipper says their product is** — descriptions, materials, weight, price. Classification of those products belongs to Trade Intelligence.

**Key State Object: `Product`**

```
Product {
  id: UUID
  name: string
  description: string               # Shipper's own description (input to classification)
  category: string                  # electronics, automotive, textiles, industrial, pharma, food, energy, consumer
  unit_value: decimal
  currency: string                  # ISO 4217 (USD, EUR, CNY, etc.)
  weight_kg: decimal

  # Multi-jurisdiction classification (shipper-declared, NOT system-determined)
  hs_code: string?                  # Primary HS code (shipper-declared, may be overridden by Trade Intelligence)
  hs_codes: {                       # Per-jurisdiction HS codes (same product classified differently)
    "US": "8507.60.0020",
    "EU": "8507.60.00",
    "CN": "8507.6000"
  }?

  # Trade-critical product attributes (v5)
  country_of_origin: string?        # ISO 2-letter — distinct from source_country/export country
                                    # A CN product warehoused in VN: origin=CN, export=VN
  manufacturer_id: string?          # MID format: country code + city(2) + mfr(3) + address digits
  material_composition: string?     # Critical for textile/apparel classification (chapters 50-63)
  unit_of_measure: string?          # CBP-required per tariff heading (dozens, pairs, sqm, kg)

  # Sourcing (a product may come from multiple locations)
  source_locations: [{              # Where this product is manufactured/sourced
    country: string,                # ISO 2-letter
    facility: string?,              # Factory/warehouse name
    is_primary: boolean
  }]?

  # Intelligence cache (populated by Description Quality service)
  cached_analysis: {}?              # Cached analysis results for performance (includes description quality scores)

  # Jurisdiction catalog registrations (v5.1)
  catalog_registrations: {}?        # JSONB: tracks which jurisdiction product catalogs this product is registered in.
                                    # Example: {"BR_CATPROD": {"registered_at": "2026-01-15T...", "registration_id": "CAT-12345"},
                                    #           "CN_SINGLE_WINDOW": {"registered_at": "...", "registration_id": "..."}}
                                    # Brazil's DUIMP system requires products to be pre-registered in the
                                    # Catalogo de Produtos before a declaration can reference them.

  entity_state: active | deleted    # Soft delete (v5): no physical deletes, default active

  created_at: timestamp
  updated_at: timestamp
}
```

**Key distinction:** Product Catalog owns what the **shipper says** (description, declared HS codes, source locations). Trade Intelligence owns what the **system determines** (classified HS codes, confidence scores). These are different entities in different domains.

**Owned Tables:** `products`

**Intelligence Services:**
| Service | Trigger | Output |
|---------|---------|--------|
| **Description Quality** | Product created/updated | Quality score + improvement suggestions (helps shipper write better descriptions for classification) |

**Note:** Classification does NOT live here. Product Catalog owns what the shipper says; Trade Intelligence owns what the system determines.

**Jurisdiction catalog registration (v5.1):** Brazil's DUIMP (Declaracao Unica de Importacao) system requires products to be pre-registered in the Catalogo de Produtos before a declaration can reference them. The Product Catalog domain must support exporting product registrations to jurisdiction-specific systems via the `JurisdictionAdapter.register_product(product, jurisdiction)` method. For Brazil, this maps to Siscomex Catalogo de Produtos registration. Registration status is tracked in `Product.catalog_registrations` JSONB. The Declaration Management domain's pre-filing validation should verify that required catalog registrations exist before allowing filing.

**Events Produced:**
| Event | Subject | Payload |
|-------|---------|---------|
| `ProductCreated` | `clearance.product.created` | Full Product state |
| `ProductUpdated` | `clearance.product.updated` | Full Product state |
| `DescriptionQualityProduced` | `clearance.product.description-quality.produced` | Product ID + quality score + improvement suggestions |

**Events Consumed:** None — Product Catalog is a pure producer. It doesn't react to other domains.

**API Surface:**
| Method | Endpoint | Type | MCP Tool |
|--------|----------|------|----------|
| `GET` | `/products` | Read (Redis) | `list_products` |
| `GET` | `/products/{id}` | Read (Redis) | `get_product` |
| `POST` | `/products` | Write (service → event) | `create_product` |
| `PUT` | `/products/{id}` | Write (service → event) | `update_product` |
| `DELETE` | `/products/{id}` | Write (service → event) | `delete_product` |
| `POST` | `/products/{id}/description-quality` | Write (triggers intelligence) | `check_description_quality` |

---

### 3.2 Trade Intelligence Domain

**Business purpose:** Determine HS classifications, calculate tariffs/duties, assess FTA eligibility, and compute landed costs. Trade Intelligence is a **proper domain that owns state** — classification results, tariff calculations, and FTA determinations are persistent state objects, not ephemeral computation results.

**Key insight:** Trade Intelligence subscribes to `ProductCreated` events and **proactively** produces classification. It subscribes to `ShipmentCreated` events and proactively produces tariff calculations. Consumers (Declaration Management, Financial Settlement, Compliance) receive intelligence by subscribing to Trade Intelligence events — they do NOT call Trade Intelligence APIs for routine computation.

**The litmus test:** "Can I add a new consumer of classification results by just subscribing to `clearance.trade-intel.classification.produced`?" **YES.**

**Key State Objects:**

```
Classification {
  id: UUID
  product_id: UUID              # What product was classified
  hs_code: string               # Determined HS code
  hs_description: string        # HS heading text
  confidence: HIGH | MEDIUM | LOW
  reasoning_steps: [string]     # GRI application steps
  alternative_codes: [{code, description, confidence}]
  validation: {known_code, known_heading, normalized}
  classified_at: timestamp
  version: int                  # Re-classification increments version
}

TariffCalculation {
  id: UUID
  context_type: product | shipment | order
  context_id: UUID              # Product, shipment, or order ID
  hs_code: string
  origin_country: string
  destination_country: string
  destination_subdivision: string?   # ISO 3166-2 state/province code (e.g., "BR-SP", "CA-ON", "IN-MH")
                                     # Required for state/province-level tax calculation:
                                     # Brazil ICMS (varies by state), Canada HST/PST (varies by province),
                                     # India IGST vs SGST/CGST (intra-state vs inter-state).
  declared_value: decimal
  currency: string
  jurisdiction: string
  line_items: [{
    program: string,              # Tax/duty program identifier (e.g., "US_DUTY", "EU_VAT", "CN_CONSUMPTION_TAX", "BR_IPI")
    rate: decimal,
    amount: decimal,
    description: string,
    legal_citation: string?,
    vat_rate: decimal?,           # VAT/GST/IVA rate (EU member-state VAT, CN VAT, MX IVA, IN IGST, CA GST/HST)
    vat_amount: decimal?,         # Calculated VAT amount
    consumption_tax_rate: decimal?,  # Consumption tax rate (China select products, MX IEPS)
    consumption_tax_amount: decimal?,
    base_amount: decimal,         # Calculation base for this line item (may differ from declared_value due to cascading)
    computation_order: int,       # Order in which this component is calculated (critical for cascading taxes)
    includes_prior: boolean       # Whether this component's base includes prior components' amounts
  }]
  # Note on cascading tax jurisdictions: computation_order determines calculation sequence
  # and includes_prior indicates the base amount includes previously calculated components.
  # Example: Brazil II (order 1) -> IPI (order 2, base includes II) -> ICMS (order 3,
  # base includes II+IPI) -> PIS/COFINS (order 4). For non-cascading jurisdictions (US, CA),
  # all items have includes_prior=false and base_amount equals the declared/transaction value.

  # -- Summary totals (v5.1: distinguish duty vs tax vs fees) --
  duty_rate: decimal              # Primary customs duty rate
  duty_total: decimal             # Customs duties only (US duty, EU customs duty, BR II, IN BCD, etc.)
  tax_total: decimal              # All non-duty taxes (VAT, GST, IVA, IGST, consumption tax, IPI, PIS/COFINS, ICMS)
  fee_total: decimal?             # Processing fees (US MPF/HMF, MX DTA, BR AFRMM, etc.)
  total_duty: decimal             # DEPRECATED — use grand_total. Kept for backward compatibility.
  grand_total: decimal            # duty_total + tax_total + fee_total
  effective_rate: decimal         # grand_total / declared_value
  programs_applied: [string]    # Section 301, 232, IEEPA, ADCVD, etc.
  warnings: [string]
  calculated_at: timestamp
}

FTADetermination {
  id: UUID
  product_id: UUID
  hs_code: string
  origin_country: string
  destination_country: string
  eligible: boolean
  program: string               # USMCA, CAFTA-DR, KORUS, etc.
  rule_type: string             # YARN_FORWARD, RVC, HYBRID, TARIFF_SHIFT
  rvc_threshold: decimal
  rvc_actual: decimal?
  savings_estimate: decimal
  documentation_required: [string]
  determined_at: timestamp
}
```

**Owned Tables:** `classifications` (new), `tariff_calculations` (new), `fta_determinations` (new), `htsus_chapters`, `htsus_headings`, `section_301_lists`, `section_232_scope`, `ieepa_rates`, `adcvd_orders`, `cross_rulings`, `tax_rates`, `fee_schedules`, `exchange_rates`, `tax_regime_templates`, `fta_rules`

**Intelligence Services:**
| Service | Trigger Event | Output | Knowledge Store |
|---------|---------------|--------|-----------------|
| **Classification Engine** (e1) | `ProductCreated`, `ProductUpdated` | `ClassificationProduced` event | Qdrant: `collection:cross_rulings` |
| **Tariff Engine** (e2) | `ShipmentCreated`, `ClassificationProduced`, `RegulatorySignalDetected` | `TariffCalculated` event | Qdrant: `collection:tariff_rulings` |
| **FTA Engine** (e4) | `ClassificationProduced` (for products with FTA-eligible origin/dest) | `FTADetermined` event | — |
| **Landed Cost** | `TariffCalculated` + tax/fee schedule | Embedded in `TariffCalculated` event | — |

**Events Produced:**
| Event | Subject | Trigger |
|-------|---------|---------|
| `ClassificationProduced` | `clearance.trade-intel.classification.produced` | Classification engine completed HS code determination |
| `TariffCalculated` | `clearance.trade-intel.tariff.calculated` | Tariff computation completed for a context (product, shipment, order) |
| `FTADetermined` | `clearance.trade-intel.fta.determined` | FTA eligibility assessed |

**Events Consumed:**
| Event | From Domain | Action |
|-------|-------------|--------|
| `ProductCreated` | Product Catalog | **Proactively classify** the new product → emit `ClassificationProduced` |
| `ProductUpdated` | Product Catalog | **Re-classify** if description/origin changed → emit `ClassificationProduced` |
| `ShipmentCreated` | Shipment Lifecycle | **Proactively compute tariff** for shipment → emit `TariffCalculated` |
| `RegulatorySignalDetected` | Regulatory Intelligence | **Re-compute tariffs** for affected HS codes → emit `TariffCalculated` |

**API Surface (for on-demand/interactive use only — routine computation flows via events):**
| Method | Endpoint | Type | MCP Tool | When Used |
|--------|----------|------|----------|-----------|
| `GET` | `/classifications/{product_id}` | Read (Redis) | `get_classification` | UI displaying classification result |
| `GET` | `/tariff-calculations/{context_id}` | Read (Redis) | `get_tariff` | UI displaying tariff breakdown |
| `POST` | `/classify/stream` | Write (SSE streaming) | `stream_classify` | Interactive classification (shipper/platform) |
| `POST` | `/tariff/calculate` | Write (on-demand computation) | `calculate_tariff` | Interactive tariff calc (buyer checkout, trade lane comparison) |
| `POST` | `/fta/check` | Write (on-demand computation) | `check_fta` | Interactive FTA check |
| `POST` | `/trade-lanes/compare` | Write (on-demand computation) | `compare_trade_lanes` | Trade lane comparison tool |
| `GET` | `/jurisdictions` | Read (Redis) | `list_jurisdictions` | Reference data |

---

### 3.3 Compliance & Screening Domain

**Business purpose:** Screen entities, products, and shipments against regulatory requirements — denied party lists (DPS), UFLPA, PGA requirements, restricted materials. This is a regulatory obligation.

**Key State Object: `ComplianceResult`**

```
ComplianceResult {
  id: UUID
  entity_type: shipment | product | entity | order_line_item
  entity_id: UUID
  overall_status: CLEAR | REVIEW | HOLD | PGA_REQUIRED

  # -- Pre-clearance screening (E0) --
  preclearance: {
    entity_screening: {status, details},     # Entity vs denied party lists
    hs_validation: {status, details},        # HS code validity check
    value_reasonableness: {status, details}, # Declared value sanity check
    document_requirements: [string],         # Required docs identified
    origin_risk: {level, factors}            # Origin country risk assessment
  }?

  # -- PGA agency flags --
  pga_flags: [{
    agency: string,                  # FDA, EPA, CPSC, USDA, NHTSA, TTB, APHIS
    requirement: string,             # What's required (prior notice, permit, cert)
    status: pending | cleared | hold,
    details: string,
    prior_notice_hours: int?         # FDA: 8-15 hours depending on product
  }]

  # -- Denied Party Screening (DPS) --
  dps_screening: {
    entity_name: string,             # Entity screened
    matched: boolean,
    match_score: decimal,            # Fuzzy match confidence (0-1)
    matched_lists: [string],         # Which lists matched (SDN, Entity, UFLPA)
    risk_level: low | medium | high | critical
  }

  # -- UFLPA (Uyghur Forced Labor Prevention Act) --
  uflpa_risk: {
    level: low | medium | high,
    factors: [string],               # Risk factors identified
    score: decimal,
    origin_flagged: boolean,         # From Xinjiang region
    entity_flagged: boolean          # Known UFLPA entity
  }

  # -- Restricted materials --
  restricted_materials: [{material, regulation, status}]?

  screened_at: timestamp
  version: int                       # Re-screening increments version
}
```

**Key insight: compliance screening is multi-layered.** Pre-clearance (E0) runs an agentic tool-calling loop with 5 independent checks. PGA screening maps HS codes to agency requirements (FDA for food/pharma, EPA for industrial chemicals, CPSC for consumer safety, etc.). DPS uses fuzzy matching against restricted party lists. UFLPA assesses forced labor risk based on origin and entity. Each layer can independently trigger a HOLD.

**Owned Tables:** `compliance_results` (new), `restricted_parties`, `pga_mappings`

**Intelligence Services:**
| Service | Trigger Event | Output | Knowledge Store |
|---------|---------------|--------|-----------------|
| **Compliance Engine** (e3) | `ShipmentCreated`, `ClassificationProduced` | `ComplianceScreened` event | Qdrant: `collection:compliance_precedents` |
| **DPS Screening** | `ShipmentCreated` (entity name) | Included in `ComplianceScreened` | Qdrant: `collection:denied_parties` (global) |
| **UFLPA Assessment** | `ShipmentCreated` (origin + product) | Included in `ComplianceScreened` | — |
| **PGA Determination** | `ClassificationProduced` (HS code) | `PGAReviewRequired` event | — |

**Events Produced:**
| Event | Subject | Trigger |
|-------|---------|---------|
| `ComplianceScreened` | `clearance.compliance.screened` | Full compliance screening completed |
| `PGAReviewRequired` | `clearance.compliance.pga-required` | PGA agency review needed |
| `DPSMatchFound` | `clearance.compliance.dps-match` | Denied party match detected (high/critical) |
| `UFLPARiskFlagged` | `clearance.compliance.uflpa-flagged` | UFLPA risk identified |

**Events Consumed:**
| Event | From Domain | Action |
|-------|-------------|--------|
| `ShipmentCreated` | Shipment Lifecycle | **Proactively screen** new shipment → emit `ComplianceScreened` |
| `ClassificationProduced` | Trade Intelligence | **Re-screen** if classification changes PGA requirements |
| `RegulatorySignalDetected` | Regulatory Intelligence | **Re-screen** affected entities against updated lists |

**API Surface:**
| Method | Endpoint | Type | MCP Tool |
|--------|----------|------|----------|
| `GET` | `/compliance/{entity_id}` | Read (Redis) | `get_compliance_result` |
| `POST` | `/compliance/screen` | Write (on-demand screening) | `screen_compliance` |
| `POST` | `/screening/entity` | Write (on-demand DPS) | `screen_entity` |

---

### 3.4 Order Management Domain

**Business purpose:** The commercial world -- products, quantities, pricing, buyer/seller relationships. An order represents a commercial transaction. Its relationship to logistics is a reference: "this order is being fulfilled by these shipments." The order domain does not touch logistics or regulatory concerns.

**Key State Objects:**

```
Order {
  id: UUID
  status: draft | confirmed | analyzing | analyzed | shipping | shipped | cancelled
  origin: string                     # ISO 2-letter origin country
  destination: string                # ISO 2-letter destination country (default "US")
  shipment_id: UUID?                 # Currently 1:1 (future: many-to-many via join table)

  # Intelligence results (assembled from event subscriptions)
  analysis: {}?                      # JSONB: assembled from ClassificationProduced, TariffCalculated, ComplianceScreened
  document_status: {}?               # JSONB: document readiness per line item

  line_items: [OrderLineItem]        # Child line items
  documents: [Document]              # Attached documents

  created_at: timestamp
  updated_at: timestamp
}

OrderLineItem {
  id: int
  order_id: UUID                     # FK to Order
  product_id: UUID                   # FK to Product (catalog reference)
  product_name: string               # Snapshot of product name at order time
  quantity: int
  source_country: string             # ISO 2-letter — WHERE this line item is sourced from
  source_facility: string            # Factory/warehouse name

  # Per-line financial
  unit_value: decimal
  currency: string                   # ISO 4217
  line_total: decimal

  # Per-line trade intelligence (populated via events)
  hs_code: string?                   # Classified HS code for THIS line item
  duty_amount: decimal?              # Calculated duty for this line
  tariff_breakdown: {}?              # JSONB: full tariff detail per line

  # Per-line compliance
  compliance_status: string?         # CLEAR | REVIEW | HOLD | PGA_REQUIRED
}
```

Line items are independently classified and tariffed -- a single order can contain products from different origins, with different HS codes, different duty rates, and different compliance outcomes.

**Split shipments (v5):** The Order-to-Shipment relationship is many-to-many via a join table (`order_shipment_lines` mapping `order_line_item_id` to `shipment_id`). One order can spawn multiple shipments (partial shipments, split shipments). Different order parts may ship separately — on different dates, via different modes, and clear customs under different entries. The shipper sees "your order is 60% delivered, 40% still in transit."

**Owned Tables:** `orders`, `order_line_items`

**Events Produced:**
| Event | Subject | Trigger |
|-------|---------|---------|
| `OrderCreated` | `clearance.order.created` | New order placed |
| `OrderUpdated` | `clearance.order.updated` | Order modified |
| `OrderShipped` | `clearance.order.shipped` | Order converted to shipment |

**Events Consumed:**
| Event | From Domain | Action |
|-------|-------------|--------|
| `ClassificationProduced` | Trade Intelligence | Update order analysis with classification intelligence |
| `TariffCalculated` | Trade Intelligence | Update order analysis with tariff intelligence |
| `ComplianceScreened` | Compliance | Update order analysis with compliance flags |

**Note:** Order Management is an **assembler** — it receives intelligence from Trade Intelligence and Compliance via events and presents a unified analysis view to the shipper. It does not call those domains' APIs.

**API Surface:**
| Method | Endpoint | Type | MCP Tool |
|--------|----------|------|----------|
| `GET` | `/orders` | Read (Redis) | `list_orders` |
| `GET` | `/orders/{id}` | Read (Redis) | `get_order` |
| `POST` | `/orders` | Write (service → event) | `create_order` |
| `PUT` | `/orders/{id}` | Write (service → event) | `update_order` |
| `POST` | `/orders/{id}/ship` | Write (creates shipment) | `ship_order` |
| `POST` | `/orders/{id}/documents/upload` | Write (service) | `upload_order_document` |

---

### 3.5 Shipment Lifecycle Domain

**Business purpose:** Shipper-facing convenience abstraction. Groups goods traveling from the same origin to the same destination within an order. Provides the shipper a simplified view without exposing logistics/regulatory complexity. **NOT a customs entity** -- customs processes attach to physical entities (handling units and consolidations).

An order can split into multiple shipments (typically by origin warehouse/factory). The shipper attaches commercial documents at shipment level. The shipment is the interface between the commercial world and the physical/regulatory world.

**Key State Object:**

```
Shipment {
  id: UUID
  order_id: UUID?                    # FK to Order (future: many-to-many via join table)
  product: string                    # Product description snapshot
  product_id: UUID?                  # FK to Product catalog (optional)

  # -- Shipper-facing context --
  # Shipment is the shipper's unit of thinking: "send these goods from A to B"
  # It groups items leaving the same origin going to the same destination within an order.
  status: pending | booked | in_transit | held | partially_cleared | cleared | delivered | cancelled
  origin: string                     # ISO 2-letter origin country (origin warehouse/factory)
  destination: string                # ISO 2-letter destination country
  destination_subdivision: string?   # ISO 3166-2 state/province code (e.g., "BR-SP", "CA-ON", "IN-MH")
                                     # Required for: Brazil ICMS (rate varies by state, 7-25%),
                                     # Canada provincial taxes (GST/HST/PST varies by province),
                                     # India IGST vs SGST routing. Propagated to TariffCalculation requests.

  # -- Commercial --
  declared_value: decimal
  company_name: string               # Importer of record
  company_contacts: {}?              # Contact info for communications

  # -- Documents (attached by shipper at shipment level) --
  # Commercial documents (invoices, packing lists, certificates) live here
  # because this is the shipper's unit of document organization.
  # Regulatory documents are attached at HU/consolidation level.
  documents: [UUID]?                 # FK refs to Document domain

  # -- Handling unit state projection (NON-AUTHORITATIVE) --
  # Derived from HU state-transfer events. The HU domain is authoritative.
  # Enables shipper-facing UI without cross-domain queries.
  handling_unit_summary: [{          # JSONB: projection for shipper view
    handling_unit_id: UUID,
    type: string,                    # pallet, carton, crate, etc.
    status: string,                  # Current HU status (from HU events)
    location: string?,               # Current location (from movement events)
    consolidated_into: UUID?,        # Consolidation ID if consolidated
    customs_status: string?          # Simplified customs status
  }]?

  # -- Intelligence results (convenience copies, NON-AUTHORITATIVE) --
  # Populated via Trade Intelligence / Financial / Compliance events.
  # Authoritative copies live in their respective domains.
  codes: {}?                         # Classification/tariff snapshot
  financials: {}?                    # Financial summary
  analysis: {}?                      # Analysis results

  created_at: timestamp
  updated_at: timestamp
}
```

The shipment stores a **non-authoritative convenience projection** of HU states. It subscribes to HU state changes and derives aggregate status. It never writes back. If there is a discrepancy between the shipment projection and the HU domain, the HU domain wins.

**Owned Tables:** `shipments`

**Events Produced:**
| Event | Subject | Trigger |
|-------|---------|---------|
| `ShipmentCreated` | `clearance.shipment.created` | Shipment created from order |
| `ShipmentStatusChanged` | `clearance.shipment.status-changed` | Aggregate status derived from HU events |
| `ShipmentDelivered` | `clearance.shipment.delivered` | All HUs delivered |

**Events Consumed:**
| Event | From Domain | Action |
|-------|-------------|--------|
| `OrderShipped` | Order Management | Create shipment |
| `HUStatusChanged` | Cargo & Handling Units | Update handling_unit_summary projection, derive aggregate status |
| `ClassificationProduced` | Trade Intelligence | Update codes convenience copy |
| `TariffCalculated` | Trade Intelligence | Update financials convenience copy |
| `ComplianceScreened` | Compliance | Update analysis convenience copy |

**Shipment State Machine:**

The shipment's status is an **aggregate derived from its HUs**, not driven by direct carrier events. The shipment observes its handling units and computes its own status.

```
pending ──→ booked ──→ in_transit ──→ partially_cleared ──→ cleared ──→ delivered
                            │                │                  ▲
                            └──→ held ───────┘                  │
                                   │                            │
                                   └────────────────────────────┘
                                              (holds released, HUs clear)
```

Status derivation rules:
- `pending` -- shipment created, no HUs picked up yet
- `booked` -- HUs created and assigned
- `in_transit` -- at least one HU is in transit
- `held` -- ALL HUs in the shipment are held (v5: shipper needs to see this)
- `partially_cleared` -- some HUs cleared, others still in transit or at customs
- `cleared` -- all HUs cleared
- `delivered` -- all HUs delivered
- `cancelled` -- shipment cancelled

**The broker bridges both worlds:** The broker sees the shipper's shipment context (commercial documents, declared values, product descriptions) but files declarations and manages clearance at the HU/consolidation level. The shipment gives the broker the commercial context; the HU/consolidation gives customs the physical reality.

**API Surface:**
| Method | Endpoint | Type | MCP Tool |
|--------|----------|------|----------|
| `GET` | `/shipments` | Read (Redis) | `list_shipments` |
| `GET` | `/shipments/{id}` | Read (Redis) | `get_shipment` |
| `POST` | `/shipments` | Write (service → event) | `create_shipment` |

---

### 3.6 Cargo & Handling Units Domain

**Business purpose:** Physical cargo units -- the actual things that move through the supply chain and that customs authorities inspect, hold, and clear. Each handling unit is a first-class entity with its own customs states, movement history, and regulatory lifecycle. When consolidated, an HU subscribes to its parent consolidation's movement events. Customs declarations, inspections, holds, and clearance attach at this level.

**Key State Object:**

```
HandlingUnit {
  id: UUID
  shipment_id: UUID                  # FK to Shipment (commercial context)

  # -- Physical description --
  type: pallet | carton | crate | drum | envelope | tube | other
  pieces: int                        # Number of pieces
  weight_kg: decimal
  dimensions: {length, width, height, unit}?
  shipping_marks: string?
  description: string?               # Cargo description for customs purposes

  # -- Status --
  status: created | at_origin | picked_up | in_transit | at_customs | in_bond |
          inspection | exam_ordered | held | general_order | seized | refused |
          abandoned | cleared | out_for_delivery | delivered
  # v5 additions: in_bond, exam_ordered, general_order, seized, refused, abandoned
  # in_bond: cargo moving under customs bond (IT/IE/T&E) between ports of entry
  # exam_ordered: between hold and inspection — CBP orders exam, broker arranges transport to CES
  # general_order: after 15 days unclaimed — distinct legal status, cargo transferred to GO warehouse
  # seized: CBP seized merchandise (IPR, contraband, UFLPA) — terminal state
  # refused: importer refused delivery — triggers return/re-export/destruction options
  # abandoned: importer formally abandoned goods (different from GO which is involuntary)

  # -- Current position --
  current_location: {                # JSONB: where is this HU right now?
    facility_type: warehouse | airport | seaport | ground_terminal | customs_facility | bonded_warehouse | carrier_facility | delivery_address,
    facility_name: string,
    country: string,                 # ISO 2-letter
    city: string?
  }?

  # -- Consolidation relationship --
  consolidated_into: UUID?           # Immediate parent consolidation only
  # When consolidated, this HU subscribes to the parent consolidation's
  # movement events and derives its position from them.
  # When deconsolidated, reverts to tracking its own movement directly.

  # -- Customs state --
  export_clearance: not_required | pending | declared | cleared
  # v5.2: Driven by ExportDeclaration lifecycle. Set to `not_required` when
  # adapter.export_filing_required() returns false. Transitions to `declared` on
  # ExportDeclarationSubmitted, to `cleared` on ExportDeclarationCleared.
  import_clearance: not_required | pending | declared | under_review | held | cleared
  hold_type: string?                 # Active hold type if import_clearance=held
  export_declaration_id: UUID?       # FK to ExportDeclaration (v5.2, origin-country filing)
  declaration_id: UUID?              # FK to EntryFiling (import declaration in Declaration domain)

  # -- In-bond movement detail (v5) --
  in_bond_detail: {                  # JSONB: populated when status=in_bond
    bond_type: IT | IE | TE,         # IT=Immediate Transportation, IE=Immediate Exportation, TE=Transportation & Exportation
    origin_port: string,             # CBP port of arrival (e.g., USLAX)
    destination_port: string,        # CBP inland destination (e.g., USDFW)
    filing_reference: string?,       # CBP 7512 or ACE electronic in-bond filing reference
    transit_time_limit_days: int,    # 30 days for IT
    departed_at: timestamp?,
    arrival_due: timestamp?,
    arrived_at: timestamp?,
    extension_granted: boolean       # Can be extended with proper notification
  }?
  # Note: EU equivalent is T1/T2 transit procedures (NCTS).
  # India: Section 54 transshipment bond. Brazil: DTA.

  # -- Cross-references --
  tracking_number: string?           # Primary tracking number
  house_number: string?              # HAWB / HBL / Pro# — the document for THIS unit

  # -- Cage/detention (populated when physically detained) --
  cage_status: {                     # JSONB: physical detention tracking
    facility_type: carrier_cage | cfs | ces | border_facility | bonded_warehouse,
    facility_name: string,
    cage_location: string?,          # Physical slot: "Bay A, Rack 3"
    intake_time: timestamp,
    dwell_days: decimal,
    storage_rate_per_day: decimal,
    storage_cost_accrued: decimal,
    go_deadline: timestamp?,         # General Order deadline
    go_days_remaining: decimal?,
    exam_type: string?,              # vacis | tailgate | intensive
    exam_scheduled: timestamp?,
    exam_duration_hours: decimal?
  }?

  # -- Movement history (v5: extracted to separate table) --
  # Transit history is stored in a separate `hu_transit_events` table for performance.
  # JSONB arrays in PostgreSQL are inefficient for append-only growth patterns — every
  # movement event rewrites the entire column. Separate table enables proper indexing.
  last_known_location: {             # Denormalized from latest transit event for common queries
    facility_name: string,
    country: string,
    city: string?,
    timestamp: timestamp
  }?
  # Full history: SELECT * FROM cargo.hu_transit_events WHERE hu_id = ? ORDER BY timestamp
  # Redis joins last_known_location + transit events for full views.

  # -- Two-layer status model (v5) --
  jurisdiction_status: string?       # Raw status from customs authority (flexible, no constraints)
  platform_status: pending | under_review | held | cleared | released | rejected
                                     # Calculated/normalized from jurisdiction_status via jurisdiction config mapping
                                     # No code changes to add a jurisdiction — just configuration

  entity_state: active | deleted     # Soft delete (v5): no physical deletes, default active

  created_at: timestamp
  updated_at: timestamp
}
```

**FTZ / Bonded Warehouse (v5):** FTZ admission/withdrawal and bonded warehouse are special HU status paths:
- **FTZ lifecycle:** Goods enter FTZ under entry type 06/26. Can be manipulated/manufactured in zone. Withdrawn under entry type 06 (consumption) with duty paid on either admitted article or finished product (whichever has lower duty — "inverted tariff" benefit). Distinction between privileged foreign status (classified at admission) and non-privileged (classified at withdrawal, affected by tariff changes).
- **Bonded warehouse:** Goods can stay up to 5 years before withdrawal for consumption, export, or destruction. Entry type 21 (warehouse), type 31 (warehouse withdrawal for consumption).
- These are tracked via the HU's `status` field and `in_bond_detail` JSONB.

**Owned Tables:** `handling_units`, `hu_transit_events` (v5: extracted from JSONB)

**Events Produced:**
| Event | Subject | Trigger |
|-------|---------|---------|
| `HUCreated` | `clearance.handling-unit.created` | HU created for shipment |
| `HUStatusChanged` | `clearance.handling-unit.status-changed` | Any status/customs change |
| `HUConsolidated` | `clearance.handling-unit.consolidated` | Assigned to a consolidation |
| `HUDeconsolidated` | `clearance.handling-unit.deconsolidated` | Removed from consolidation |
| `HUMovement` | `clearance.handling-unit.movement` | Physical movement event |
| `HUCustomsStatusChanged` | `clearance.handling-unit.customs-status-changed` | Export/import clearance change |
| `HUCageIntake` | `clearance.handling-unit.cage.intake` | Entered detention facility |
| `HUCageReleased` | `clearance.handling-unit.cage.released` | Released from detention |
| `HUGOWarning` | `clearance.handling-unit.cage.go-warning` | General Order deadline approaching |

**Events Consumed:**
| Event | From Domain | Action |
|-------|-------------|--------|
| `ConsolidationMovement` | Consolidation | Derive own position from parent (when consolidated) |
| `ConsolidationStatusChanged` | Consolidation | Update own status to reflect parent state |
| `ConsolidationDeconsolidated` | Consolidation | Clear `consolidated_into`, resume direct tracking |
| `AdjudicationDecision` | Customs Adjudication | Update import_clearance based on authority decision |
| `ExportAdjudicationDecision` | Customs Adjudication | Update export_clearance based on origin authority decision (v5.2) |
| `ExportDeclarationCleared` | Declaration Management | Set export_clearance = cleared (v5.2) |
| `HoldReleased` | Exception Management | Clear hold_type, update import_clearance |
| `ExamScheduled` | Customs Adjudication | Update cage_status with exam details |

**API Surface:**
| Method | Endpoint | Type | MCP Tool |
|--------|----------|------|----------|
| `GET` | `/handling-units` | Read (Redis) | `list_handling_units` |
| `GET` | `/handling-units/{id}` | Read (Redis) | `get_handling_unit` |
| `GET` | `/handling-units/by-shipment/{shipment_id}` | Read (Redis) | `get_shipment_handling_units` |
| `POST` | `/handling-units` | Write (service → event) | `create_handling_unit` |
| `POST` | `/handling-units/{id}/movement` | Write (service → event) | `record_hu_movement` |

---

### 3.7 Consolidation Domain

**Business purpose:** Grouping of handling units (and other consolidations) for transport. A consolidation represents the physical transport unit that carriers and customs deal with -- a MAWB, MBL, manifest, container, or ULD. Has its own customs states independent of its contents. Supports nesting: a consolidation can contain handling units AND other consolidations. Cannot be deconsolidated if any contained entity has an active customs hold.

**Key State Object:**

```
Consolidation {
  id: UUID
  border_crossing_mode: air | ocean | ground    # Regulatory mode for customs docs
  consolidation_type: mawb | mbl | manifest | container | uld
  master_number: string              # MAWB / MBL / manifest ID
  carrier: string                    # Operating carrier
  origin_port: string                # Port/airport of consolidation
  destination_port: string           # Port/airport of deconsolidation

  # -- Status --
  status: open | closed | in_transit | arrived | at_customs | held | cleared |
          deconsolidating | partially_deconsolidated | deconsolidated
  # v5: partially_deconsolidated — some HUs removed, others remain (e.g., 197 of 200 HAWBs
  # sorted to onward routing, 3 held in cage). Consolidation reaches `deconsolidated`
  # only when ALL contained entities have been removed.

  # -- Nesting (consolidation in consolidation) --
  consolidated_into: UUID?           # Parent consolidation (null if top-level)
  # When nested, this consolidation subscribes to its parent's movement events
  # and derives position from them, just like an HU does.

  # -- Customs state (consolidation-level) --
  export_clearance: not_required | pending | declared | cleared
  # v5.2: Driven by ExportDeclaration lifecycle (same as HU).
  import_clearance: not_required | pending | declared | under_review | held | cleared
  hold_type: string?
  export_declaration_id: UUID?       # FK to ExportDeclaration (v5.2, origin-country filing)
  declaration_id: UUID?              # Consolidation-level import declaration

  # -- Deconsolidation readiness (v5: query-time computed, NOT event-driven) --
  # `can_deconsolidate` is NOT a stored boolean. It is a readiness checklist computed
  # on-demand when deconsolidation is requested, by querying Redis cache:
  #   (1) No active customs holds on contained entities
  #   (2) Required documents validated
  #   (3) Carrier release obtained (carrier's authorization to break the consolidation)
  #   (4) No outstanding authority instructions
  # This eliminates bidirectional HU↔Consolidation event coupling. The consolidation
  # does not need to subscribe to every HU customs status change to maintain a boolean.

  # -- Two-layer status model (v5) --
  jurisdiction_status: string?       # Raw status from customs authority
  platform_status: pending | under_review | held | cleared | released | rejected

  # -- Mode-specific identifiers --
  # Air
  flight_number: string?
  uld_id: string?
  uld_type: string?                  # LD3 / LD7 / LD11 / PMC
  acas_status: pending | filed | cleared | do_not_load?  # v5: ACAS is AIR ONLY, carrier-side filing
  # ACAS (Air Cargo Advance Screening) is a carrier obligation, NOT broker.
  # Different from ISF (ocean, importer/broker). Modeled here on the air Consolidation entity.

  # Ocean
  vessel_name: string?
  voyage_number: string?
  container_number: string?          # ISO 6346 format
  seal_number: string?

  # -- Current location --
  current_location: {
    facility_type: airport | seaport | ground_terminal | customs_facility | bonded_warehouse,
    facility_name: string,
    country: string,
    city: string?
  }?

  # -- Aggregates --
  total_pieces: int                  # Sum of all contained HU pieces
  total_weight_kg: decimal           # Sum of all contained HU weights

  # -- Movement events (v5: extracted to separate table) --
  # Movement events stored in `consolidation_transit_events` table.
  # Keep `last_known_location` on the consolidation for common queries.
  # Redis joins them for full views.
  last_known_location: {
    facility_name: string,
    country: string,
    city: string?,
    timestamp: timestamp
  }?

  entity_state: active | deleted     # Soft delete (v5): no physical deletes, default active

  created_at: timestamp
  updated_at: timestamp
}
```

**Key insight: `border_crossing_mode` = the regulatory mode.**
- A "FedEx Air" shipment from Shanghai to College Station has 5+ legs: ground pickup, air to CDG hub, air to Memphis hub, ground to delivery. `border_crossing_mode = "air"` because the border is crossed by air.
- A "J.B. Hunt" (ground carrier) shipment from Shanghai to Dallas crosses the border by ocean. `border_crossing_mode = "ocean"`, not "ground".
- The routing engine determines `border_crossing_mode` from the first international leg's mode, not from the carrier's primary mode.

**Key insight: document hierarchy.**
- Air: HAWB (on HU) -> MAWB (on Consolidation) -> ULD -> Flight
- Ocean: HBL (on HU) -> MBL (on Consolidation) -> Container -> Vessel/Voyage
- Ground: Pro# (on HU) -> Manifest (on Consolidation) -> Trailer
- The `house_number` (HU) -> `consolidation.master_number` (Consolidation) chain is the fundamental document hierarchy of international trade.

**Partial deconsolidation (v5):** Individual HUs can be deconsolidated independently — an HU's `consolidated_into` is cleared individually. Source of truth = querying all entities where `consolidated_into = this_consolidation_id`. The consolidation status progresses: `deconsolidating` → `partially_deconsolidated` (some removed) → `deconsolidated` (ALL contained entities removed).

**Owned Tables:** `consolidations`, `consolidation_transit_events` (v5: extracted from JSONB)

**Events Produced:**
| Event | Subject | Trigger |
|-------|---------|---------|
| `ConsolidationCreated` | `clearance.consolidation.created` | New consolidation opened |
| `ConsolidationClosed` | `clearance.consolidation.closed` | Ready for transport |
| `ConsolidationStatusChanged` | `clearance.consolidation.status-changed` | Any status change |
| `ConsolidationNested` | `clearance.consolidation.nested` | Nested into parent consolidation |
| `ConsolidationMovement` | `clearance.consolidation.movement` | Physical movement event |
| `ConsolidationCustomsStatusChanged` | `clearance.consolidation.customs-status-changed` | Export/import clearance change |
| `DeconsolidationStarted` | `clearance.consolidation.deconsolidation-started` | Deconsolidation begins |
| `DeconsolidationBlocked` | `clearance.consolidation.deconsolidation-blocked` | Blocked by customs hold |
| `ConsolidationDeconsolidated` | `clearance.consolidation.deconsolidated` | Fully broken down |

**Events Consumed:**
| Event | From Domain | Action |
|-------|-------------|--------|
| `ConsolidationMovement` (parent) | Consolidation (parent) | Derive own position from parent when nested |
| `ConsolidationStatusChanged` (parent) | Consolidation (parent) | Update own status to reflect parent |
| `AdjudicationDecision` | Customs Adjudication | Update consolidation-level import customs status |
| `ExportAdjudicationDecision` | Customs Adjudication | Update consolidation-level export customs status (v5.2) |
| `ExportDeclarationCleared` | Declaration Management | Set export_clearance = cleared (v5.2) |
Note (v5): `can_deconsolidate` is computed on-demand at query time (not event-driven), so this domain no longer subscribes to `HUCustomsStatusChanged` or child `ConsolidationCustomsStatusChanged` for deconsolidation eligibility. This eliminates the bidirectional event coupling.

**Reference Number Generation:**

The platform generates format-correct reference numbers following real-world patterns:
| Reference Type | Format | Example |
|---------------|--------|---------|
| HAWB | Carrier prefix + 8 digits | `FX12345678` |
| MAWB | Airline prefix + 8 digits | `176-12345678` |
| MBL | Carrier + voyage + sequence | `MAEU123456789` |
| Entry Number | Filer code + entry + check digit | `ABC-1234567-0` |
| Tracking Number | Mode-specific format | `1Z999AA10123456784` |
| ISF Transaction | Filer + year + sequence | `ISF-2026-000142` |

These are platform concerns (exist in production), not simulation artifacts.

**Routing Intelligence:**

The platform includes routing intelligence for determining optimal transport routes:
| Capability | Description |
|-----------|-------------|
| **Hub routing tables** | Integrator networks (FedEx Memphis, DHL Leipzig), ocean transshipment (Singapore, Algeciras) |
| **Regulatory touchpoint derivation** | Which territories require clearance based on route |
| **Border-crossing mode determination** | `border_crossing_mode` assignment based on corridor and carrier |
| **Multi-modal leg generation** | Air, ground, ocean, rail sequences |

Routing rules are platform knowledge that exists in production. The simulation uses these rules + randomization for variety.

**API Surface:**
| Method | Endpoint | Type | MCP Tool |
|--------|----------|------|----------|
| `GET` | `/consolidations` | Read (Redis) | `list_consolidations` |
| `GET` | `/consolidations/{id}` | Read (Redis) | `get_consolidation` |
| `GET` | `/consolidations/{id}/contents` | Read (Redis) | `get_consolidation_contents` |
| `GET` | `/consolidations/{id}/all-handling-units` | Read (Redis cache) | `get_all_handling_units_in_consolidation` |
| `POST` | `/consolidations` | Write (service → event) | `create_consolidation` |
| `POST` | `/consolidations/{id}/add` | Write (service → event) | `add_to_consolidation` |
| `POST` | `/consolidations/{id}/deconsolidate` | Write (service → event) | `deconsolidate` |
| `POST` | `/consolidations/{id}/movement` | Write (service → event) | `record_consolidation_movement` |

#### 3.7.1 Movement Event Cascade

Movement events happen at whatever level is physically moving. When entities are consolidated, only the outermost entity moves -- but every contained entity must reflect that movement through event propagation.

**Cascade flow:**

```
Consolidation Y (top-level, physically moving)
  +-- publishes: ConsolidationMovement {location: "CDG Airport, FR"}
       |
       +-- Consolidation X (nested inside Y)
       |    +-- subscribes to Y's movement events
       |        +-- updates own current_location
       |            +-- publishes: ConsolidationMovement {location: "CDG Airport, FR", derived_from: Y.id}
       |                 |
       |                 +-- HU-1 (inside X)
       |                 |    +-- subscribes to X's movement events
       |                 |        +-- updates own current_location
       |                 |            +-- publishes: HUMovement {derived_from: X.id}
       |                 |                 +-- Shipment (subscribes to HU-1)
       |                 |                      +-- updates handling_unit_summary projection
       |                 |
       |                 +-- HU-2 (inside X)
       |                      +-- same cascade as HU-1
       |
       +-- HU-3 (directly inside Y, not nested)
            +-- subscribes to Y's movement events
                +-- updates own current_location -> publishes -> Shipment updates
```

**Rules:**
1. Each entity only subscribes to its **immediate parent** (`consolidated_into`)
2. When consolidated, the entity stops tracking its own movement directly
3. When deconsolidated, the entity resumes direct movement tracking
4. The `derived_from` field in movement events traces the cascade origin
5. The Shipment domain subscribes to its HUs' events and maintains a **non-authoritative projection**

**Redis convenience cache:**
- `consolidation:{id}:all_hus` -- flattened set of ALL handling unit IDs inside a consolidation, regardless of nesting depth
- Maintained by a NATS subscriber that listens to consolidation/HU events
- Rebuildable via PostgreSQL recursive CTE: walk `consolidated_into` references
- Used by broker and intelligence services for quick introspection: "what is everything inside this consolidation?"

---

### 3.8 Declaration Management Domain

**Business purpose:** The broker's clearance workflow — claim shipments, verify documents, prepare entry filings, submit to authorities, respond to authority requests. This is what the **BROKER** does.

**Key distinction:** Declaration Management prepares and submits. Customs Adjudication receives and decides. These are fundamentally different domains with different owners.

**ISF Filing:** Declaration Management owns the ISF 10+2 (Importer Security Filing) workflow — deadline enforcement (24 hours before vessel loading for ocean), data validation (shipper, consignee, HTS matching against classification), late filing penalty assessment, and amendment tracking. ISF is a pre-arrival filing obligation that exists alongside the entry declaration.

**Export declarations (v5.2):** Export declarations are first-class filing entities in this domain. Every international shipment requires TWO declaration workflows: an export declaration at origin and an import declaration (EntryFiling) at destination. These are filed by different parties, to different authorities, on different timelines — but the platform manages both. The `ExportDeclaration` entity parallels `EntryFiling` for the origin-country filing. The HU/Consolidation `export_clearance` field is driven by the ExportDeclaration lifecycle.

**Key State Objects:**

```
EntryFiling {
  id: UUID
  handling_unit_id: UUID?            # FK to HandlingUnit (declaration for a physical unit)
  consolidation_id: UUID?            # FK to Consolidation (declaration for a consolidation)
  # Declarations attach to the physical entity customs cares about — a handling
  # unit or a consolidation — not to the shipment abstraction.
  broker_id: UUID?                   # FK to Broker (assigned broker)
  entry_number: string?              # CBP entry number (assigned at filing, e.g., "ABC-1234567-0")
  entry_type: string                 # v5: expanded entry type codes:
                                     # "01" (consumption), "02" (quota/visa), "03" (AD/CVD),
                                     # "06" (FTZ), "11" (informal), "21" (warehouse),
                                     # "23" (TIB — Temporary Importation Bond),
                                     # "31" (warehouse withdrawal), "46" (drawback),
                                     # "86" (Section 321 — SUSPENDED since Aug 2025)
  port_of_entry: string?             # CBP port code

  # -- Entry type determination logic (v5) --
  # Under $2,500 → informal Type 11; over $2,500 → formal Type 01
  # Section 321 de minimis ($800 threshold, Type 86): SUSPENDED since August 29, 2025.
  # All imports now require formal or informal entry regardless of value.
  # Section 321 reinstatement is configurable — the platform can re-enable Type 86
  # and de minimis exemption via jurisdiction adapter configuration.

  # -- Entry/Release vs Entry Summary split (v5) --
  # US customs operates a two-step process:
  # 1. Entry (CBP 3461) — "release me from customs" request, can be filed up to 15 days before arrival
  # 2. Entry Summary (CBP 7501) — "here's what I owe" declaration, within 10 working days of release
  # Cargo can be released and physically moving while entry summary is still being prepared.
  release_number: string?            # CBP 3461 entry/release number
  release_status: pending | filed | released  # Entry for release status
  summary_status: pending | filed | accepted | liquidated  # Entry summary status
  released_at: timestamp?            # When CBP released the cargo
  summary_due_date: timestamp?       # 10 working days from release
  summary_filed_at: timestamp?       # When entry summary was filed
  # Non-US equivalents:
  # EU: ENS (carrier-filed) → import declaration → customs released → customs debt confirmed
  # India: Prior BoE → assessed BoE (assessment is the "summary" equivalent)
  # Brazil: DI/DUIMP registration → parametrization (channel is the "release" equivalent)

  # -- Multi-jurisdiction support --
  jurisdiction: string               # "US", "EU", "IN", "BR", "CN", "MX", "CA"
  jurisdiction_entry_type: string?   # Jurisdiction-specific declaration/entry type (v5.1).
                                     # Complements entry_type (which holds US-specific codes).
                                     # Examples:
                                     #   India BoE type: "home_consumption" | "warehousing" | "ex_bond"
                                     #   Mexico pedimento clave: "A1" | "A4" | "AF" | "G1" | "IN" | "RT" | "V1"
                                     #   Canada CAD type: "B3" | "B3B" | "K84"
                                     #   Brazil DUIMP modality: "normal" | "simplified" | "express"
                                     # Validated by the jurisdiction adapter's entry_type_validator.
  jurisdiction_config: {}?           # JSONB: jurisdiction-specific filing configuration
                                     # (authority names, filing workflows, currency defaults)
                                     # v5: The checklist model + jurisdiction adapter pattern enables
                                     # adding new jurisdictions without domain model changes.
                                     # Status machines use two-layer approach. Filing workflows
                                     # are jurisdiction-configured.

  # -- Two-layer status model (v5) --
  jurisdiction_status: string?       # Raw status from customs authority (flexible, no constraints)
  platform_status: pending | under_review | held | cleared | released | rejected

  # -- Clearance channel assignment (v5.1) --
  clearance_channel: string?         # Jurisdiction-specific clearance channel assigned by customs authority.
                                     # Brazil: verde/amarelo/vermelho/cinza (green/yellow/red/grey)
                                     # India: RMS green/yellow/red
                                     # China: green/yellow/red channel
                                     # Channel determines level of documentary/physical inspection required.
                                     # Maps to platform_status via jurisdiction adapter config.
  channel_assigned_at: timestamp?    # When the authority assigned the clearance channel

  # -- Filing lifecycle --
  filing_status: draft | pending_broker_approval | submitted | accepted |
                 cf28_pending | cf29_pending | exam_scheduled | rejected | released

  # -- Dynamic checklist (varies by jurisdiction + transport mode + commodity) --
  checklist_state: {                 # JSONB: broker's filing readiness checklist
    items: [{
      key: string,                   # "classification", "tariff", "compliance", "bond", "documents"
      label: string,
      status: pending | verified | overridden | failed,
      required: boolean,
      override_reason: string?,
      verified_at: timestamp?
    }],
    bond_type: string                # "continuous" or "single_entry"
  }?

  # -- Authority interaction --
  authority_response: {}?            # JSONB: CBP/customs authority response data
                                     # (CF-28 requests, CF-29 notices, exam results)
  summary_data: {}?                  # JSONB: filing summary (value, duty, fees)
  broker_approval: {}?               # JSONB: broker sign-off data

  filed_at: timestamp?
  released_at: timestamp?
  created_at: timestamp
  updated_at: timestamp
}

ISFFiling {
  id: UUID
  handling_unit_id: UUID?            # FK to HandlingUnit
  consolidation_id: UUID?            # FK to Consolidation
  declaration_id: UUID?
  transaction_number: string
  filing_status: pending | filed | accepted | rejected | amended
  deadline: timestamp                # 24h before vessel loading (ocean)
  is_late: boolean
  data: {
    shipper, consignee, seller, buyer,
    ship_to, container_stuffing_location,
    consolidator, hs_codes: [string],
    country_of_origin, manufacturer
  }
  validation_issues: [{field, issue}]
  filed_at: timestamp?
  created_at: timestamp
}
# v5: ISF is OCEAN ONLY, importer/broker obligation.
# ACAS (Air Cargo Advance Screening) is AIR ONLY, CARRIER obligation.
# ACAS is modeled on the air Consolidation entity (see acas_status below), NOT in Declaration Management.

ENSFiling {
  # v5: EU ICS2 Release 3 — mandatory since Feb 2026 for ALL EU-bound cargo
  id: UUID
  consolidation_id: UUID             # FK to Consolidation (ENS is filed per transport document)
  carrier_id: string                 # Filed by carrier/forwarder, NOT importer
  ens_status: pending | filed | registered | do_not_load | report_required | no_action
  mrn: string?                       # EU Master Reference Number (assigned upon registration)
  filing_deadline: timestamp         # Air: before loading; Ocean: 24h before loading (deep sea)
  hs_codes: [string]                 # 6-digit HS codes for all goods
  consignor: string
  consignee: string
  filed_at: timestamp?
  created_at: timestamp
}
# ENS is a CARRIER obligation, not a broker obligation.
# The broker's checklist includes "EU entries need ENS/ICS2 filing confirmation"
# but the broker does not file ENS — they verify it was filed by the carrier.

ExportDeclaration {
  # v5.2: First-class export filing entity — parallel to EntryFiling for import.
  # Every international shipment originates from a jurisdiction that requires (or may require)
  # an export declaration. The exporter/agent files this with the origin country's customs authority
  # BEFORE departure. Without export clearance, goods cannot legally leave the country.
  id: UUID
  handling_unit_id: UUID?            # FK to HandlingUnit (export declaration for a physical unit)
  consolidation_id: UUID?            # FK to Consolidation (export declaration for a consolidation)
  # Like EntryFiling, export declarations attach to the physical entity — HU or consolidation.

  # -- Filer identity --
  exporter_id: UUID?                 # FK to Party (exporter of record / shipper)
  agent_id: UUID?                    # FK to Party (customs agent/broker handling the export)
  # In express integrator model, the carrier may file export declarations.
  # In forwarder model, the freight forwarder or customs agent files.

  # -- Jurisdiction --
  jurisdiction: string               # Origin country jurisdiction: "US", "EU", "IN", "BR", "CN", "MX", "CA"
  jurisdiction_filing_type: string?  # Jurisdiction-specific export declaration type:
                                     #   US: "AES_EEI" (Electronic Export Information via AESDirect/AES)
                                     #   EU: "ECS_EXPORT" (Export Control System declaration, types A/B/C)
                                     #   BR: "DU_E" (Declaração Única de Exportação via Portal Único)
                                     #   IN: "SHIPPING_BILL" (via ICEGATE, types: free/dutiable/drawback/ex_bond)
                                     #   CN: "EXPORT_DEC" (Customs export declaration via Single Window)
                                     #   MX: "PEDIMENTO_A1" (Pedimento de exportación, clave A1 = definitive)
                                     #   CA: "CAED" (Canadian Automated Export Declaration via CERS)

  # -- Reference numbers --
  export_reference: string?          # Filed reference number (US: ITN, EU: Export MRN, etc.)
                                     # US ITN = Internal Transaction Number returned by AES
                                     # EU MRN = Master Reference Number for export
                                     # BR: DU-E number
                                     # IN: Shipping Bill number
  export_license: string?            # If export-controlled: license number or exemption (NLR/EAR99)

  # -- Two-layer status model --
  jurisdiction_status: string?       # Raw status from origin authority (e.g., "LEO_ISSUED", "RELEASED_FOR_EXPORT")
  platform_status: pending | under_review | held | cleared | released | rejected
                                     # Normalized from jurisdiction_status via adapter config

  # -- Filing lifecycle --
  filing_status: draft | pending_review | submitted | accepted | held | cleared | rejected | amended
  # Unlike import entries, export declarations typically have fewer authority interaction rounds.
  # But they CAN be held (e.g., ITAR review, export license required, sanctions screening).

  # -- Filing data (JSONB, jurisdiction-specific) --
  filing_data: {
    # Common fields across jurisdictions:
    exporter_name: string,
    consignee_name: string,
    destination_country: string,      # ISO 2-letter
    commodity_description: string,
    hs_code: string,                  # 6-digit (international) or jurisdiction-specific
    declared_value: decimal,
    currency: string,
    weight_kg: decimal,
    quantity: int?,
    unit_of_measure: string?,

    # US-specific (AES/EEI):
    schedule_b_code: string?,         # US export classification (derived from HS, but distinct)
    eccn: string?,                    # Export Control Classification Number (BIS/EAR)
    license_type: string?,            # NLR | EAR99 | license_number
    ultimate_consignee_type: string?, # Direct consumer | Government entity | Reseller | Other
    routed_transaction: boolean?,     # If true, foreign principal party handles filing
    ftsr_exemption: string?,          # Foreign Trade Statistics Regulations exemption code

    # EU-specific (ECS):
    procedure_code: string?,          # 4-digit CPC (e.g., 1000 = normal export)
    additional_procedure: string?,    # A00, B02, etc.

    # BR-specific (DU-E):
    ncm_code: string?,               # Nomenclatura Comum do Mercosul
    cfop: string?,                    # Tax operation code
    suframa_benefit: boolean?,

    # IN-specific (Shipping Bill):
    bill_type: string?,              # "free" | "dutiable" | "drawback" | "ex_bond"
    drawback_rate: decimal?,
    igst_exemption: boolean?,

    # CN-specific:
    supervision_mode: string?,        # General trade, processing trade, etc.
    ciq_required: boolean?,           # Quality inspection required

    # MX-specific:
    pedimento_clave: string?,         # A1, A4, etc.
    fraccion_arancelaria: string?,    # Mexican tariff fraction

    # CA-specific:
    export_permit_number: string?,    # If controlled goods
  }?

  # -- Export checklist (dynamic, jurisdiction-specific) --
  checklist_state: {
    items: [{
      key: string,                   # "export_license", "sanctions_screening", "hs_classification",
                                     #  "commercial_invoice", "packing_list", "dangerous_goods",
                                     #  "dual_use_check", "origin_cert", "phytosanitary"
      label: string,
      status: pending | verified | overridden | failed | not_applicable,
      required: boolean,
      override_reason: string?,
      verified_at: timestamp?
    }]
  }?

  # -- Timestamps --
  filed_at: timestamp?
  cleared_at: timestamp?             # When origin authority cleared for export
  created_at: timestamp
  updated_at: timestamp
}
# KEY INSIGHT: Export declarations have fundamentally different filing triggers than import entries.
# Import entries are filed when cargo approaches destination (pre-arrival filing).
# Export declarations are filed when cargo is being prepared for departure at origin.
# The broker/agent has BOTH in their workflow — but the timelines and checklists differ.
#
# COORDINATION: For a CN→US shipment, the China export declaration must clear BEFORE
# the cargo departs China. The US import entry can be filed up to 15 days before arrival.
# These are independent filings to independent authorities, but the platform tracks both.

Broker {
  id: UUID
  name: string
  license_number: string             # Unique customs broker license
  license_district: string           # CBP district of license
  status: active | inactive | suspended
  specializations: {}?               # JSONB: commodity expertise, jurisdiction expertise
  workload_capacity: int             # Max concurrent entries (default 20)
  contact_info: {}?                  # JSONB: email, phone, etc.
  created_at: timestamp
  updated_at: timestamp
}

BrokerAssignment {
  id: UUID
  broker_id: UUID                    # FK to Broker
  handling_unit_id: UUID?            # FK to HandlingUnit
  consolidation_id: UUID?            # FK to Consolidation
  status: assigned | in_progress | submitted | completed
  priority: urgent | high | normal | low
  assigned_at: timestamp
  notes: string?
  created_at: timestamp
  updated_at: timestamp
}

BrokerMessage {
  id: UUID
  shipment_id: UUID?                 # FK to Shipment (optional — some messages are general)
  broker_id: UUID                    # FK to Broker
  direction: inbound | outbound
  channel: email | phone | portal | edi
  party_type: shipper | carrier | cbp | importer
  party_name: string?
  subject: string?
  body: string
  thread_id: string?                 # Conversation threading
  attachments: {}?                   # JSONB: attachment metadata
  read_at: timestamp?                # Read tracking for inbox
  created_at: timestamp
}
```

**Key insight: checklist is dynamic, not static.** The checklist_state varies by jurisdiction, transport mode, commodity type, and compliance flags. US ocean entries need ISF 10+2 (importer/broker obligation); US air entries need ACAS confirmation (carrier obligation — broker verifies it was filed, does not file it); EU entries need ENS/ICS2 filing confirmation (carrier obligation). Entries with FDA-regulated products need PGA clearance. The checklist items are populated from intelligence events (ClassificationProduced, TariffCalculated, ComplianceScreened) via state-transfer subscriptions.

**Owned Tables:** `entry_filings`, `export_declarations` (v5.2), `isf_filings` (new), `ens_filings` (v5), `broker_assignments`, `brokers`, `broker_messages`

**Intelligence Services:**
| Service | Trigger Event | Output | Knowledge Store |
|---------|---------------|--------|-----------------|
| **Broker Intelligence** | `DeclarationCreated`, `ClassificationProduced`, `ComplianceScreened` | Filing guidance, risk insights, readiness assessment | Qdrant: `collection:filing_precedents` |
| **Filing Intelligence** | `AmendmentRequested` (from Adjudication), explicit request | Draft CF-28 responses, protest arguments, communication templates | Qdrant: `collection:cbp_responses` |

**Events Produced:**
| Event | Subject | Trigger |
|-------|---------|---------|
| `DeclarationCreated` | `clearance.declaration.created` | Draft declaration created for a shipment |
| `BrokerAssigned` | `clearance.declaration.broker-assigned` | Broker claimed shipment |
| `ChecklistUpdated` | `clearance.declaration.checklist-updated` | Checklist item verified/overridden |
| `DeclarationSubmitted` | `clearance.declaration.submitted` | Entry filed with customs authority |
| `DeclarationAmended` | `clearance.declaration.amended` | Amended filing resubmitted |
| `CF28Responded` | `clearance.declaration.cf28-responded` | Broker responded to info request |
| `CF29Responded` | `clearance.declaration.cf29-responded` | Broker responded (accept/protest) |
| `ProtestFiled` | `clearance.declaration.protest-filed` | Formal protest filed |
| `ISFFiled` | `clearance.declaration.isf-filed` | ISF 10+2 submitted |
| `ISFAmended` | `clearance.declaration.isf-amended` | ISF amended |
| `ISFLateFiling` | `clearance.declaration.isf-late` | ISF filed after deadline (penalty assessment) |
| `ENSFiled` | `clearance.declaration.ens-filed` | EU ICS2 ENS filed by carrier (v5) |
| `ExportDeclarationCreated` | `clearance.declaration.export-created` | Export declaration drafted (v5.2) |
| `ExportDeclarationSubmitted` | `clearance.declaration.export-submitted` | Export declaration filed with origin authority (v5.2) |
| `ExportDeclarationCleared` | `clearance.declaration.export-cleared` | Origin authority cleared goods for export (v5.2) |
| `ExportDeclarationHeld` | `clearance.declaration.export-held` | Export held (license review, sanctions, etc.) (v5.2) |
| `ExportDeclarationAmended` | `clearance.declaration.export-amended` | Export declaration amended and resubmitted (v5.2) |
| `ExportDeclarationRejected` | `clearance.declaration.export-rejected` | Origin authority rejected export declaration (v5.2) |

**Events Consumed:**
| Event | From Domain | Action |
|-------|-------------|--------|
| `HUCreated` | Cargo & Handling Units | Create draft declaration for HU, begin readiness evaluation |
| `ClassificationProduced` | Trade Intelligence | Update declaration with classification data, advance checklist |
| `TariffCalculated` | Trade Intelligence | Update declaration with tariff data, advance checklist |
| `ComplianceScreened` | Compliance | Update declaration with compliance flags, advance checklist |
| `DocumentValidated` | Document Management | Update checklist document items |
| `AdjudicationDecision` | Customs Adjudication | Update filing status based on authority decision |
| `CF28Issued` | Customs Adjudication | Add pending CF-28 to broker workbench |
| `CF29Issued` | Customs Adjudication | Add pending CF-29 to broker workbench |
| `ExamScheduled` | Customs Adjudication | Update declaration with exam status |
| `HUStatusChanged` (at_customs) | Cargo & Handling Units | Escalate priority if declaration not yet submitted |
| `ExportAdjudicationDecision` | Customs Adjudication | Update export filing status based on origin authority decision (v5.2) |

**API Surface (import declarations + broker workflow):**
| Method | Endpoint | Type | MCP Tool |
|--------|----------|------|----------|
| `GET` | `/broker/queue` | Read (Redis) | `get_broker_queue` |
| `GET` | `/broker/dashboard` | Read (Redis) | `get_broker_dashboard` |
| `GET` | `/broker/briefing` | Read (Redis) | `get_broker_briefing` |
| `POST` | `/broker/queue/claim` | Write (service → event) | `claim_shipment` |
| `GET` | `/broker/entries/{id}` | Read (Redis) | `get_entry_detail` |
| `POST` | `/broker/entries/{id}/approve` | Write (service → event) | `approve_entry` |
| `POST` | `/broker/entries/{id}/submit` | Write (service → event) | `submit_entry` |
| `POST` | `/broker/entries/{id}/checklist/override` | Write (service → event) | `override_checklist_item` |
| `POST` | `/broker/entries/{id}/bond` | Write (service → event) | `set_bond_type` |
| `POST` | `/broker/entries/{id}/verify-classification` | Write (triggers intel) | `verify_classification` |
| `POST` | `/broker/entries/{id}/calculate-fees` | Write (triggers intel) | `calculate_entry_fees` |
| `GET` | `/broker/cbp-responses` | Read (Redis) | `list_cbp_responses` |
| `POST` | `/broker/cbp-responses/{id}/respond-cf28` | Write (service → event) | `respond_cf28` |
| `POST` | `/broker/cbp-responses/{id}/respond-cf29` | Write (service → event) | `respond_cf29` |
| `POST` | `/broker/ai/draft-cf28` | Write (triggers intel) | `draft_cf28_response` |
| `POST` | `/broker/ai/draft-protest` | Write (triggers intel) | `draft_protest` |
| `GET` | `/broker/messages` | Read (Redis) | `list_broker_messages` |
| `POST` | `/broker/messages` | Write (service → event) | `send_broker_message` |
| `POST` | `/broker/ai/draft-communication` | Write (triggers intel) | `draft_communication` |
| `POST` | `/broker/isf/{shipment_id}/file` | Write (service → event) | `file_isf` |
| `PUT` | `/broker/isf/{shipment_id}/amend` | Write (service → event) | `amend_isf` |
| `GET` | `/broker/isf/{shipment_id}` | Read (Redis) | `get_isf_status` |

**API Surface (export declarations — v5.2):**
| Method | Endpoint | Type | MCP Tool |
|--------|----------|------|----------|
| `GET` | `/export-declarations` | Read (Redis) | `list_export_declarations` |
| `GET` | `/export-declarations/{id}` | Read (Redis) | `get_export_declaration` |
| `GET` | `/export-declarations/by-hu/{hu_id}` | Read (Redis) | `get_hu_export_declaration` |
| `POST` | `/export-declarations` | Write (service → event) | `create_export_declaration` |
| `PUT` | `/export-declarations/{id}` | Write (service → event) | `update_export_declaration` |
| `POST` | `/export-declarations/{id}/submit` | Write (service → event) | `submit_export_declaration` |
| `POST` | `/export-declarations/{id}/amend` | Write (service → event) | `amend_export_declaration` |
| `POST` | `/export-declarations/{id}/checklist/verify` | Write (service → event) | `verify_export_checklist_item` |
| `POST` | `/export-declarations/{id}/checklist/override` | Write (service → event) | `override_export_checklist_item` |

**Key insight: bidirectional trade lane workflow.** For a CN→US shipment, the broker's queue contains TWO work items: (1) China export declaration (filed before departure), (2) US import entry (filed before/at arrival). The broker dashboard shows both, and the platform tracks their coordination. An HU cannot be marked `in_transit` (international leg) without export_clearance = cleared. This is enforced at the CarrierBot/ShipperBot level (simulation) or via carrier integration (production).

---

### 3.9 Customs Adjudication Domain

**Business purpose:** Receive declarations, apply authority processing logic, issue decisions (accept/reject/hold/request-info/exam). This is what the **AUTHORITY** does — fundamentally separate from what the broker does.

**Key distinction:** In the real world, this domain would be an external system (CBP, EU customs, etc.). The Clearance Platform provides the **interface** — APIs for authorities to communicate decisions. In simulation, a bot calls these APIs. In production, a real EDI/API integration replaces the bot.

**Key State Object: `AdjudicationDecision`**

```
AdjudicationDecision {
  id: UUID
  declaration_id: UUID
  handling_unit_id: UUID?            # FK to HandlingUnit (decision on a physical unit)
  consolidation_id: UUID?            # FK to Consolidation (decision on a consolidation)
  # Authority decisions apply to the physical entity presented for clearance.
  jurisdiction: string
  decision_type: accepted | rejected | held | information_requested |
                 penalty_notice | exam_required | protest_resolved
  decision_details: {
    # Jurisdiction-specific:
    # US: entry_accepted, cf28_issued, cf29_issued, exam_scheduled, protest_resolved
    # EU: declaration_accepted, under_control, customs_released
    # BR: parameterized (green/yellow/red/grey), cleared
    # IN: duty_assessed, out_of_charge
    # CN: ciq_inspected, gacc_released
  }
  hold_type: string?            # If held: uflpa, dps, classification, etc.
  response_deadline: timestamp? # If info requested: when broker must respond
  exam_type: string?            # If exam: VACIS, tailgate, intensive
  decided_at: timestamp
  decided_by: string            # Authority identifier
}
```

**Key insight: adjudication outcomes vary by corridor, not just jurisdiction.** CN-origin shipments face ~1.3x scrutiny multiplier (UFLPA, Section 301). The STP (Simplified Transaction Processing) fast-lane rate is ~87% for normal corridors but drops to ~30% for misclassified goods. Physical inspection pass rate is ~80%. These are configurable parameters, not hardcoded — the platform models them as decision factors, the simulation tunes them.

**Owned Tables:** `adjudication_decisions` (new)

**Events Produced:**
| Event | Subject | Trigger |
|-------|---------|---------|
| `AdjudicationDecision` | `clearance.adjudication.decision` | Authority made a decision on a declaration |
| `CF28Issued` | `clearance.adjudication.cf28-issued` | Information request issued |
| `CF29Issued` | `clearance.adjudication.cf29-issued` | Penalty notice issued |
| `ExamScheduled` | `clearance.adjudication.exam-scheduled` | Physical exam scheduled |
| `ProtestResolved` | `clearance.adjudication.protest-resolved` | Protest adjudicated |
| `ExportAdjudicationDecision` | `clearance.adjudication.export-decision` | Origin authority decision on export declaration (v5.2) |

**Jurisdiction-Specific Events:**
| Jurisdiction | Subject Pattern | Decision Types |
|-------------|-----------------|----------------|
| US (CBP) | `clearance.adjudication.us.*` | `entry-accepted`, `cf28-issued`, `cf29-issued`, `exam-scheduled`, `protest-resolved` |
| EU (UCC) | `clearance.adjudication.eu.*` | `declaration-accepted`, `under-control`, `customs-released`, `released-to-cleared` |
| BR (Siscomex) | `clearance.adjudication.br.*` | `parameterized`, `cleared`, `rejected` |
| IN (ICEGATE) | `clearance.adjudication.in.*` | `duty-assessed`, `out-of-charge`, `cleared`, `rejected` |
| CN (GACC) | `clearance.adjudication.cn.*` | `ciq-inspected`, `released`, `cleared`, `rejected` |

**Events Consumed:**
| Event | From Domain | Action |
|-------|-------------|--------|
| `DeclarationSubmitted` | Declaration Management | Receive declaration, begin adjudication |
| `DeclarationAmended` | Declaration Management | Receive amended declaration, re-adjudicate |
| `CF28Responded` | Declaration Management | Evaluate broker's response to info request |
| `CF29Responded` | Declaration Management | Process broker's response (accept/protest) |
| `ProtestFiled` | Declaration Management | Adjudicate protest |
| `ExportDeclarationSubmitted` | Declaration Management | Receive export declaration, begin origin authority adjudication (v5.2) |
| `ExportDeclarationAmended` | Declaration Management | Re-adjudicate amended export declaration (v5.2) |

**API Surface (for authority interaction — simulation bots or real integrations call these):**
| Method | Endpoint | Type | MCP Tool |
|--------|----------|------|----------|
| `POST` | `/adjudication/declarations/{id}/decide` | Write (authority decision → event) | `adjudicate_declaration` |
| `POST` | `/adjudication/declarations/{id}/request-info` | Write (CF-28 → event) | `request_information` |
| `POST` | `/adjudication/declarations/{id}/issue-penalty` | Write (CF-29 → event) | `issue_penalty_notice` |
| `POST` | `/adjudication/declarations/{id}/schedule-exam` | Write (exam → event) | `schedule_exam` |
| `POST` | `/adjudication/protests/{id}/resolve` | Write (protest resolution → event) | `resolve_protest` |
| `GET` | `/adjudication/decisions/{id}` | Read (Redis) | `get_adjudication_decision` |
| `POST` | `/adjudication/export-declarations/{id}/decide` | Write (origin authority decision → event) | `adjudicate_export_declaration` |

---

### 3.10 Exception Management Domain

**Business purpose:** Handle holds, cage/exam processing, cargo exceptions, supply chain disruptions, and resolutions.

**Key State Object: `Exception`**

```
Exception {
  id: UUID
  handling_unit_id: UUID?            # FK to HandlingUnit (hold/exception on a physical unit)
  consolidation_id: UUID?            # FK to Consolidation (hold/exception on a consolidation)
  # Holds apply to physical entities. Cage/detention tracking lives on the HU entity
  # but the exception record references it for workflow management.
  exception_type: hold | inspection | cage | disruption | cargo_exception
  hold_type: uflpa | dps | pga | classification | documentation |
             adcvd | no_preclearance | physical_exam | transit_hold
  status: open | investigating | pending_resolution | resolved | escalated
  severity: critical | high | medium | low

  # -- Transit hold detail (for multi-jurisdiction holds) --
  transit_hold_detail: {
    territory: string,               # "EU", "SG", "KR" — which territory issued the hold
    location: string,                # "Paris CDG, FR" — where cargo is physically held
    authority: string,               # "EU Customs", "Singapore MPA"
    hold_reason: string,             # "EU sanctions screening — entity match flagged"
    resolution_path: string,         # Different per territory (EU customs contact vs US CBP)
    required_documentation: [string],# What's needed to resolve
    estimated_resolution_days: int?
  }?

  # -- Cage/facility detail (physical cargo detention) --
  cage_status: {
    facility_type: carrier_cage | cfs | ces | border_facility | bonded_warehouse,
    facility_name: string,           # "FedEx Memphis Gateway Cage"
    cage_location: string,           # "Bay A, Rack 3"
    intake_at: timestamp,
    dwell_days: decimal,
    storage_rate_per_day: decimal,   # $40-$100/day by facility type
    storage_cost: decimal,           # Accrued storage charges
    go_deadline: timestamp,          # General Order: 15 days from arrival (US CBP)
    go_days_remaining: decimal,
    go_warning_thresholds: [10, 5, 3, 1],  # Days remaining when warnings emit
    exam_type: vacis | tailgate | intensive?,
    exam_description: string?,       # "VACIS/NII — non-intrusive x-ray scan"
    exam_scheduled_at: timestamp?,
    exam_duration_hours: decimal?,   # Expected duration (VACIS: 4-16h, intensive: 24-72h)
    released_at: timestamp?
  }?

  # -- Resolution --
  resolution: {
    method: string,                  # How the hold was resolved
    resolved_by: string,             # Who resolved it
    resolved_at: timestamp,
    notes: string
  }?

  created_at: timestamp
  updated_at: timestamp
}
```

**Key insight: transit holds are fundamentally different from destination holds.** A hold at EU CDG requires engagement with EU customs (different authority, different documentation, different timeline) than a hold at US Memphis. The architecture must model the territory context for each hold, not just a flat hold_type string. Resolution paths vary by territory:
- **US CBP hold** → existing CF-28/CF-29 workflow
- **EU transit hold** → EU customs contact, T1 documentation, sanctions license if applicable
- **Origin export hold** → origin country regulatory engagement
- **Transshipment port hold** → port authority engagement, may require rerouting

**Owned Tables:** `exceptions` (new)

**Intelligence Services:**
| Service | Trigger Event | Output | Knowledge Store |
|---------|---------------|--------|-----------------|
| **Exception Engine** (e5) | `ExceptionCreated`, explicit request | Resolution suggestions, similar case analysis, drafted responses | Qdrant: `collection:resolution_precedents` |

**Events Produced:**
| Event | Subject |
|-------|---------|
| `ExceptionCreated` | `clearance.exception.created` |
| `ExceptionUpdated` | `clearance.exception.updated` |
| `HoldReleased` | `clearance.exception.hold-released` |
| `HoldEscalated` | `clearance.exception.hold-escalated` |
| `DisruptionDetected` | `clearance.exception.disruption.detected` |

**Note:** Cage/detention events (`CageIntake`, `CageReleased`, `GODeadlineWarning`) are now published by the Cargo & Handling Units domain on `clearance.handling-unit.cage.*` subjects, since cage/detention tracking lives on the HU entity. Exception Management coordinates the workflow but does not own the cage event subjects.

**Events Consumed:**
| Event | From Domain | Action |
|-------|-------------|--------|
| `AdjudicationDecision` (held) | Customs Adjudication | Create hold exception |
| `ExamScheduled` | Customs Adjudication | Create cage intake |
| `ComplianceScreened` (HOLD) | Compliance | Create hold for compliance failure |
| `DPSMatchFound` | Compliance | Create hold for DPS match |
| `HUStatusChanged` (at_customs) | Cargo & Handling Units | Check if declaration pre-filed; if not, create `no_preclearance` hold |

**Cage Management Subprocess:**

When a handling unit requires physical examination, Exception Management runs the cage subprocess. Cage/detention tracking lives on the HU entity (`cage_status` JSONB); the exception record references the HU for workflow management.

```
ExamScheduled (from Adjudication)
    │
    ├──► CageIntake
    │    Assign facility + cage slot
    │    Record intake timestamp
    │    Calculate GO deadline (15 days from arrival for US)
    │    Start storage cost accrual
    │    → Notify Financial Settlement (CageIntake event)
    │
    ├──► CageExamScheduled
    │    Schedule specific exam type:
    │    • VACIS (non-intrusive X-ray scan)
    │    • Tailgate (open container, partial visual)
    │    • Intensive (full unload and physical)
    │
    ├──► [Exam performed — result from Adjudication]
    │    AdjudicationDecision event carries exam result
    │
    ├──► CageReleased (exam passed)
    │    │ Record release timestamp
    │    │ Stop storage cost accrual
    │    │ → Notify Financial Settlement (CageReleased event)
    │    │ → HoldReleased → HU domain updates import_clearance
    │    │
    │    OR
    │    │
    │    ├──► [Exam failed / discrepancy found]
    │    │    Exception escalated
    │    │    Additional hold created
    │    │    Dwell continues
    │    │
    │    └──► GODeadlineWarning (5 days before GO)
    │         Alert broker + exception dashboard
    │         If GO deadline passes: cargo becomes government property
    │         (General Order — unclaimed cargo, 6 months)
```

**Dwell time monitoring:** A periodic check (hourly) computes `dwell_days = NOW() - intake_at` for all caged cargo. Emits `GODeadlineWarning` when `dwell_days > (go_deadline - 5 days)`. Storage costs accrue daily at facility-specific rates.

**API Surface:**
| Method | Endpoint | Type | MCP Tool |
|--------|----------|------|----------|
| `GET` | `/exceptions` | Read (Redis) | `list_exceptions` |
| `GET` | `/exceptions/{id}` | Read (Redis) | `get_exception` |
| `POST` | `/exceptions/{id}/transition` | Write (service → event) | `transition_exception` |
| `POST` | `/resolution/lookup` | Write (triggers intel) | `lookup_resolution` |
| `POST` | `/resolution/upload-document` | Write (service) | `upload_resolution_document` |
| `POST` | `/resolution/suggest` | Write (triggers intel) | `suggest_resolution` |

---

### 3.11 Financial Settlement Domain

**Business purpose:** Calculate duties, fees, demurrage/detention, manage bonds, process payments, handle drawback claims, and reconcile post-entry adjustments. Financial Settlement tracks every dollar from estimated duty at booking through liquidated duty at final settlement.

**Key State Objects:**

```
FinancialRecord {
  id: UUID
  shipment_id: UUID
  declaration_id: UUID?
  phase: estimated | provisional | liquidated | reconciled

  # Duty & fees
  duty_total: decimal                # Sum of all duty programs
  duty_line_items: [{program, rate, amount, legal_citation}]
  adcvd_cash_deposit: decimal?       # v5: AD/CVD tracked separately — different liquidation timelines
  adcvd_deposit_rate: decimal?       # v5: Set annually by Commerce Dept administrative reviews
  adcvd_bond_id: UUID?              # v5: AD/CVD bonds are separate from customs bonds
  mpf: decimal                       # Merchandise Processing Fee — CONFIGURABLE (v5: not hardcoded)
                                     # Current: 0.3464% formal, min $31.67, max $614.35
                                     # Informal: $2/$6/$9 depending on type. Adjusted annually by CBP.
  hmf: decimal                       # Harbor Maintenance Fee — CONFIGURABLE (v5: not hardcoded)
                                     # Current: 0.125%, ocean only. Does NOT apply to FTZ withdrawals for export.
  exam_fees: decimal                 # CONFIGURABLE per facility (v5: varies by CES and port)
  broker_fees: decimal               # Broker service fees
  terminal_fees: decimal             # Appointment miss, redelivery, chassis split — configurable
  other_fees: [{type, amount, description}]

  # -- PMS payment workflow (v5) --
  pms_enrollment: boolean?           # Whether importer is enrolled in Periodic Monthly Statement
  pms_statement_period: string?      # Calendar month — all entries consolidated into single payment
  pms_payment_due: timestamp?        # 15th working day of following month

  # -- Liquidation tracking (v5) --
  liquidation_status: pending | liquidated | extended | suspended | deemed_liquidated
  liquidation_due: timestamp?        # 314 days post-entry (US)
  liquidation_extensions: [{         # v5: CBP can extend up to 3 times, 1 year each (4 years total)
    extension_number: int,
    reason: string,
    extended_until: timestamp
  }]?
  deemed_liquidated_date: timestamp? # 4-year statutory deadline — auto-liquidated if CBP doesn't act

  # Bond
  bond_id: UUID?
  bond_type: continuous | single_entry
  bond_sufficiency: sufficient | insufficient | pending_review
  bond_amount: decimal               # v5: Standard continuous bond = greater of $50K or 10% of annual
                                     # duties. AD/CVD entries require separate single transaction bond
                                     # at 100% of the antidumping duty margin. Thresholds are configurable.

  # -- Demurrage & detention (D&D) --
  # CRITICAL: Ocean and air D&D are structurally different.
  # Ocean: per-container-type daily rates, carrier-specific free days, demurrage vs detention split
  # Air: per-kg storage rates with tiered pricing, free hours (not days), min daily charge
  demurrage: {
    mode: ocean | air,

    # -- Ocean D&D (carrier-specific policies from CARRIER_DnD_POLICIES) --
    carrier_policy: string?,           # "Maersk Line", "MSC Mediterranean", etc.
    free_days_import: int?,            # Carrier-specific: Maersk 4, MSC 5, COSCO 7
    free_days_export: int?,            # Carrier-specific: typically 7-10 days
    combined_free_days: boolean?,      # CMA CGM combines demurrage+detention free time
    container_type: string?,           # "20GP", "40GP", "40HC", "45HC"
    demurrage_daily_rate: decimal?,    # Per container type: $125-$370/day
    detention_daily_rate: decimal?,    # Per container type: $75-$190/day
    demurrage_days: int?,              # Days past free time at port/terminal
    demurrage_charges: decimal?,       # Accrued demurrage
    detention_days: int?,              # Days past free time after pickup
    detention_charges: decimal?,       # Accrued detention

    # -- Air storage (carrier-specific policies) --
    free_hours: int?,                  # Typically 48h for all carriers
    storage_per_kg_first_5: decimal?,  # $0.12-$0.15/kg first 5 days
    storage_per_kg_after: decimal?,    # $0.22-$0.25/kg after 5 days
    min_daily_charge: decimal?,        # $20-$25/day floor
    weight_kg: decimal?,               # Cargo weight for calculation
    storage_days: int?,
    storage_charges: decimal?,

    # -- Common --
    free_time_start: timestamp?,       # Vessel discharge (ocean) or flight arrival (air)
    free_time_expires: timestamp?,
    last_dnd_calculation: timestamp?,  # Idempotency: skip if already calculated this tick
    total_accrued: decimal,            # Sum of all D&D charges
    appointment_date: timestamp?,      # Delivery appointment
    appointment_missed: boolean?,      # $150 penalty if missed
  }

  # Cage/facility storage (separate from carrier D&D — this is CBP/CFS storage)
  storage_costs: decimal

  # Payment
  payment_status: pending | scheduled | paid | partial | overdue
  payment_method: ach | pms | statement | cash_deposit
  payment_scheduled_at: timestamp?
  paid_at: timestamp?

  # Totals
  total_financial_exposure: decimal  # Sum of all obligations
  total_paid: decimal
  balance_due: decimal

  calculated_at: timestamp
  liquidated_at: timestamp?
}

Bond {
  id: UUID
  importer_id: UUID
  bond_type: continuous | single_entry
  bond_number: string
  surety_code: string
  amount: decimal                    # Continuous: typically 10% of annual duties
  effective_date: date
  expiration_date: date?             # Single entry: one-time; Continuous: annual renewal
  status: active | exhausted | expired | cancelled
  utilization: decimal               # Running total of duties charged against bond
  sufficiency_threshold: decimal     # Alert when utilization > threshold
}

DutyDrawback {
  id: UUID
  original_entry_id: UUID            # Import entry
  export_entry_id: UUID?             # Re-export entry
  drawback_type: manufacturing | unused_merchandise | rejected
  hs_code: string
  original_duty_paid: decimal
  refund_amount: decimal             # Up to 99% of original duty
  status: draft | filed | under_review | approved | paid | denied
  filed_at: timestamp?
  approved_at: timestamp?
}

Reconciliation {
  id: UUID
  declaration_id: UUID
  adjustment_type: reliquidation | post_summary_correction | protest_refund | rate_change |
                   cbp_reconciliation  # v5: Entry type 09 — file with estimated values, reconcile later
  original_amount: decimal
  adjusted_amount: decimal
  difference: decimal
  reason: string
  status: pending | applied | disputed
  effective_date: date
}
# v5: CBP Reconciliation program (entry type 09) allows importers to file entries with
# estimated values and reconcile later. Critical for transfer pricing, assists/sells,
# and complex value determinations. AD/CVD cash deposits have different liquidation
# timelines (tied to Commerce annual review, not CBP standard liquidation) and may
# result in refunds or additional duty years after the original entry.
```

**Owned Tables:** `financial_records` (new), `bonds` (new), `duty_drawbacks` (new), `reconciliations` (new)

**Financial Lifecycle:**
```
ShipmentCreated
    │
    ├──► ESTIMATED phase
    │    Pre-compute duties from TariffCalculated event
    │    Check bond sufficiency
    │    Estimate D&D exposure if ocean
    │
DeclarationSubmitted
    │
    ├──► PROVISIONAL phase
    │    Duties deposited (duty deposit = estimated duty)
    │    MPF/HMF calculated from declared value
    │    Payment scheduled
    │
AdjudicationDecision (accepted)
    │
    ├──► LIQUIDATED phase (typically 314 days post-entry for US)
    │    Final duty amount assessed
    │    Reconciliation if different from provisional
    │    Drawback eligibility assessed
    │
    └──► RECONCILED phase
         Post-entry adjustments applied
         Drawback claims processed
         Final settlement
```

**Events Produced:**
| Event | Subject |
|-------|---------|
| `FeesCalculated` | `clearance.financial.fees-calculated` |
| `DemurrageAccruing` | `clearance.financial.demurrage-accruing` |
| `DemurrageStopped` | `clearance.financial.demurrage-stopped` |
| `BondSufficiencyWarning` | `clearance.financial.bond-sufficiency-warning` |
| `PaymentScheduled` | `clearance.financial.payment-scheduled` |
| `PaymentCompleted` | `clearance.financial.payment-completed` |
| `DutyLiquidated` | `clearance.financial.duty-liquidated` |
| `DrawbackFiled` | `clearance.financial.drawback-filed` |
| `DrawbackApproved` | `clearance.financial.drawback-approved` |
| `ReconciliationApplied` | `clearance.financial.reconciliation-applied` |
| `FinancialSettled` | `clearance.financial.settled` |

**Events Consumed:**
| Event | From Domain | Action |
|-------|-------------|--------|
| `TariffCalculated` | Trade Intelligence | Update duty calculations (ESTIMATED phase) |
| `ShipmentCreated` | Shipment Lifecycle | Create financial record, check bond sufficiency |
| `DeclarationSubmitted` | Declaration Management | Advance to PROVISIONAL, schedule duty deposit |
| `AdjudicationDecision` (accepted) | Customs Adjudication | Begin liquidation timeline |
| `HUCustomsStatusChanged` (cleared) | Cargo & Handling Units | Finalize provisional duty obligations |
| `HUStatusChanged` (at_customs) | Cargo & Handling Units | Start demurrage clock (ocean: vessel discharge, air: 48h) |
| `HUStatusChanged` (delivered) | Cargo & Handling Units | Stop demurrage, calculate final D&D |
| `HUCageIntake` | Cargo & Handling Units | Start storage cost accrual |
| `HUCageReleased` | Cargo & Handling Units | Stop storage cost accrual, finalize cage fees |
| `CF29Issued` | Customs Adjudication | Potential reliquidation / rate adjustment |

**API Surface:**
| Method | Endpoint | Type | MCP Tool |
|--------|----------|------|----------|
| `GET` | `/financial/{shipment_id}` | Read (Redis) | `get_financial_record` |
| `GET` | `/financial/summary` | Read (Redis) | `get_financial_summary` |
| `GET` | `/financial/bonds` | Read (Redis) | `list_bonds` |
| `GET` | `/financial/bonds/{id}` | Read (Redis) | `get_bond` |
| `POST` | `/financial/bonds` | Write (service → event) | `create_bond` |
| `GET` | `/financial/bonds/{id}/sufficiency` | Read (Redis) | `check_bond_sufficiency` |
| `POST` | `/financial/drawback` | Write (service → event) | `file_drawback_claim` |
| `GET` | `/financial/drawback/{id}` | Read (Redis) | `get_drawback` |
| `GET` | `/financial/reconciliations/{declaration_id}` | Read (Redis) | `get_reconciliations` |
| `POST` | `/financial/{shipment_id}/payment` | Write (service → event) | `schedule_payment` |

---

### 3.12 Regulatory Intelligence Domain

**Business purpose:** Monitor regulatory changes, assess impact, model scenarios. Autonomous supporting domain.

**Dual trigger model:** Regulatory Intelligence operates on two independent trigger paths:
1. **Periodic polling** — A scheduled job (NATS-triggered cron, e.g. every 15 minutes) polls external feeds (Federal Register, CBP CSMS, trade news) via feed parsers. New signals are ingested, deduplicated (URL → content hash → semantic similarity), and emitted as `RegulatorySignalDetected`.
2. **Manual/on-demand** — Platform users or AI agents call the `/regulatory/refresh` API to trigger an immediate poll, or `/regulatory/signals` POST to manually create a signal (e.g., from direct knowledge of a policy change).

Both paths produce the same `RegulatorySignalDetected` event. Downstream consumers (Trade Intelligence for tariff recalculation, Compliance for re-screening) react identically regardless of trigger source.

**Key State Object: `RegulatorySignal`**

```
RegulatorySignal {
  id: int (autoincrement)             # Not UUID — append-only, no updates
  title: string(500)                  # Signal headline
  status: confirmed | proposed | discussed
  description: text                   # Full signal body
  affected_hs_codes: {}               # JSONB: structured HS code impact list
  affected_countries: {}?             # JSONB: affected jurisdictions
  effective_date: date?
  source: string(200)                 # "Federal Register", "CBP CSMS", etc.
  source_url: string(500)?            # Direct link to source
  impact_level: critical | high | medium | low
  category: tariff_change | trade_restriction | policy_update | fta_modification |
            rate_change | enforcement_action
  jurisdiction: string(10)            # "US", "EU", "CN", etc. (default "US")

  # -- Deduplication + change tracking --
  source_hash: string(64)?            # SHA-256 of source content — unique index for dedup
  rate_change: {}?                    # JSONB: { old_rate, new_rate, program, effective_date }
                                      # Only present for tariff/rate changes — enables diff computation

  created_at: timestamp               # Append-only: no updated_at
}
```

**Indexes:** `status`, `category`, `impact_level`, `effective_date`, `affected_hs_codes` (GIN), `source_hash` (unique)

**Owned Tables:** `regulatory_signals`

**Intelligence Services:**
| Service | Trigger | Output | Knowledge Store |
|---------|---------|--------|-----------------|
| **Regulatory Monitor** (e6) | Periodic feed polling, manual refresh | `RegulatorySignalDetected` event | Qdrant: `collection:regulatory_corpus` (global) |

**Events Produced:**
| Event | Subject |
|-------|---------|
| `RegulatorySignalDetected` | `clearance.regulatory.signal-detected` |
| `RegulatoryImpactAssessed` | `clearance.regulatory.impact-assessed` |

**Events Consumed:** None — pure producer.

**API Surface:**
| Method | Endpoint | Type | MCP Tool |
|--------|----------|------|----------|
| `GET` | `/regulatory/signals` | Read (Redis) | `list_regulatory_signals` |
| `POST` | `/regulatory/signals` | Write (service → event) | `create_regulatory_signal` |
| `POST` | `/regulatory/scenario` | Write (computation) | `model_regulatory_scenario` |
| `POST` | `/regulatory/refresh` | Write (triggers monitor) | `refresh_regulatory_feeds` |

---

### 3.13 Document Management Domain

**Business purpose:** Generate, validate, store, and manage trade documents. Cross-cutting supporting domain.

**Key State Object: `Document`**

```
Document {
  id: UUID
  order_id: UUID?                    # FK to Order (nullable — some docs are shipment-only)
  shipment_id: UUID?                 # FK to Shipment (nullable — some docs are order-only)
  document_type: commercial_invoice | packing_list | bill_of_lading |
                 airway_bill | house_bill | master_bill |
                 certificate_of_origin | pga_certificate | customs_form |
                 isf_filing | power_of_attorney | bond_certificate |
                 phytosanitary_certificate | fda_prior_notice |
                 fcc_declaration | dot_declaration | epa_declaration |
                 export_declaration | export_license | shippers_export_declaration |
                 dangerous_goods_declaration
  filename: string(300)
  storage_url: string                # v5: S3-compatible object storage (MinIO dev, S3 prod)
                                     # Pre-signed URLs with TTL for access. Replaces content_base64.
                                     # Storing document content as base64 in PostgreSQL is a known
                                     # anti-pattern causing TOAST bloat and replication lag.
  content_type: string(100)          # MIME type (default: "application/pdf")
  entity_state: active | deleted     # Soft delete (v5)
  created_at: timestamp
  updated_at: timestamp
}
```

**Key insight: document requirements are mode-specific AND direction-specific (v5.2).** Ocean shipments need HBL, MBL, ISF 10+2, AMS manifest. Air shipments need HAWB, MAWB, AES/EEI filing. Ground shipments need PAPS/PARS manifest. PGA-regulated commodities add agency-specific certificates (FDA prior notice, FCC declaration, DOT declaration, EPA declaration, phytosanitary certificates). **Export documents** include: commercial invoice, packing list, shipper's export declaration, export licenses (EAR/ITAR/dual-use), certificates of origin, dangerous goods declarations, and jurisdiction-specific forms (US EEI, EU export declaration, IN shipping bill). The architecture determines required documents at `ShipmentCreated` time based on transport mode, corridor, commodity HS code, and trade direction (export vs import).

**Current implementation note:** Documents are linked via `order_id` and/or `shipment_id` FKs (not a generic entity_type/entity_id). Content is stored in S3-compatible object storage (MinIO in development, S3 in production) with `storage_url` in PostgreSQL. Access is via pre-signed URLs with TTL. There is no separate `status` or `validation_result` column — document validation is handled by the Document Validator intelligence service, which emits `DocumentValidated` events consumed by the Declaration Management domain's checklist.

**Owned Tables:** `documents`

**Intelligence Services:**
| Service | Trigger | Output | Knowledge Store |
|---------|---------|--------|-----------------|
| **Document Engine** (e7) | Explicit request | Generated document from template + data | — |
| **Document Validator** | `DocumentUploaded` | `DocumentValidated` event | — |

**Events Produced:**
| Event | Subject |
|-------|---------|
| `DocumentUploaded` | `clearance.document.uploaded` |
| `DocumentValidated` | `clearance.document.validated` |
| `DocumentDiscrepancyFound` | `clearance.document.discrepancy` |
| `DocumentGenerated` | `clearance.document.generated` |

**Events Consumed:**
| Event | From Domain | Action |
|-------|-------------|--------|
| `ShipmentCreated` | Shipment Lifecycle | Determine required documents for jurisdiction |
| `ClassificationProduced` | Trade Intelligence | Update document requirements based on HS code |

**API Surface:**
| Method | Endpoint | Type | MCP Tool |
|--------|----------|------|----------|
| `GET` | `/documents/requirements/{entity_id}` | Read (Redis) | `get_document_requirements` |
| `POST` | `/documents/generate/{entity_id}` | Write (triggers generation) | `generate_document` |
| `POST` | `/documents/analyze` | Write (triggers validation) | `analyze_document` |
| `POST` | `/{entity_type}/{id}/documents/upload` | Write (service → event) | `upload_document` |
| `GET` | `/{entity_type}/{id}/documents` | Read (Redis) | `list_entity_documents` |
| `GET` | `/documents/{id}/content` | Read (direct) | `get_document_content` |

---

### 3.14 Party Management Domain (v5)

**Business purpose:** Manage the identity, registrations, trust programs, and relationships of all parties involved in customs transactions. Customs operations involve multiple parties — importers of record, consignees, manufacturers, sellers/buyers, brokers, carriers, freight forwarders, notify parties. These parties have compliance histories, trust scores, jurisdiction-specific registrations, and legal relationships (power of attorney, continuous bonds). The current design scattered party data across domains; this domain consolidates it as a shared reference.

**Key State Objects:**

```
Party {
  id: UUID
  name: string
  type: importer | consignee | manufacturer | seller | buyer |
        broker | carrier | forwarder | notify_party

  # -- Jurisdiction-specific identifiers (JSONB) --
  # Each jurisdiction has mandatory registration requirements that GATE filing.
  jurisdiction_identifiers: {
    "US": { ein: string?, cbp_bond_holder: boolean },
    "EU": { eori: string? },               # EORI mandatory for ALL EU customs transactions
    "BR": { cnpj: string?, radar_type: "express" | "limited" | "unlimited",
            radar_utilization: decimal? },   # RADAR GATES all Brazil imports
    "IN": { iec: string?, dsc_class: string? },  # IEC mandatory for all IN imports/exports
    "CN": { customs_reg: string?, gacc_reg: string? },  # GACC mandatory for food/cosmetics
    "MX": { rfc: string?, padron_type: string?,
            immex_auth: string? },           # Padrón mandatory for all MX imports
    "CA": { bn: string?, rpp_status: boolean? }  # BN mandatory for all CA imports
  }?

  # -- Trust programs --
  trust_programs: [{
    program: string,                 # C-TPAT, AEO, Linha Azul, OEA, PIP, etc.
    jurisdiction: string,
    tier: string?,                   # T1/T2/T3 (India), AEOC/AEOS (EU)
    certified_at: timestamp,
    expires_at: timestamp?
  }]?

  compliance_score: decimal?         # 0-1, derived from filing history, hold rates, penalty history
  carrier_affiliation: string?       # "FedEx", "UPS", "DHL" — null for independent brokers
  is_integrated_carrier_broker: boolean?  # Carrier IS the broker (express integrator model)

  entity_state: active | deleted     # Soft delete
  created_at: timestamp
  updated_at: timestamp
}

PowerOfAttorney {
  id: UUID
  importer_id: UUID                  # FK to Party (importer of record)
  broker_id: UUID                    # FK to Party (customs broker)
  poa_type: continuous | per_entry   # Continuous is standard
  status: active | revoked | expired
  jurisdiction: string               # POA is jurisdiction-specific
  granted_at: timestamp
  revoked_at: timestamp?
  expires_at: timestamp?
}
# Without a POA on file, a broker CANNOT file on behalf of an importer.
# POAs are revocable — importer can switch brokers by revoking and issuing new.

ImporterOfRecord {
  id: UUID
  party_id: UUID                     # FK to Party
  # The IOR is the legal entity responsible for customs compliance, duty payment,
  # and post-entry obligations. May differ from buyer, consignee, and shipper.
  # Express carriers may serve as IOR (DHL Express's "DHL as IOR" service).
  ior_type: self | designated_broker | carrier_as_ior
  bond_id: UUID?                     # FK to Bond (Financial Settlement)
  jurisdiction: string
  status: active | suspended | inactive
  created_at: timestamp
  updated_at: timestamp
}
```

**UX design needed:** Party management screens for managing party registrations, POA relationships, and compliance scores.

**Owned Tables:** `parties`, `power_of_attorney`, `importers_of_record`

**Events Produced:**
| Event | Subject | Trigger |
|-------|---------|---------|
| `PartyCreated` | `clearance.party.created` | New party registered |
| `PartyUpdated` | `clearance.party.updated` | Party details changed (registrations, trust programs) |
| `POAGranted` | `clearance.party.poa-granted` | Power of attorney granted (importer→broker) |
| `POARevoked` | `clearance.party.poa-revoked` | Power of attorney revoked |

**Events Consumed:** None — Party Management is a pure producer. Other domains reference party data via events.

**API Surface:**
| Method | Endpoint | Type | MCP Tool |
|--------|----------|------|----------|
| `GET` | `/parties` | Read (Redis) | `list_parties` |
| `GET` | `/parties/{id}` | Read (Redis) | `get_party` |
| `POST` | `/parties` | Write (service → event) | `create_party` |
| `PUT` | `/parties/{id}` | Write (service → event) | `update_party` |
| `POST` | `/parties/{id}/poa` | Write (service → event) | `grant_poa` |
| `DELETE` | `/parties/{id}/poa/{poa_id}` | Write (service → event) | `revoke_poa` |
| `GET` | `/parties/{id}/registrations` | Read (Redis) | `get_party_registrations` |

---

## 4. NATS Subject Hierarchy & Event Catalog

### 4.1 Subject Hierarchy

```
clearance.                              # Clearance platform root
├── product.                            # Product Catalog
│   ├── created                         # ProductCreated
│   ├── updated                         # ProductUpdated
│   └── description-quality.produced    # DescriptionQualityProduced
│
├── trade-intel.                        # Trade Intelligence
│   ├── classification.produced         # ClassificationProduced ← KEY EVENT
│   ├── tariff.calculated               # TariffCalculated ← KEY EVENT
│   └── fta.determined                  # FTADetermined
│
├── compliance.                         # Compliance & Screening
│   ├── screened                        # ComplianceScreened ← KEY EVENT
│   ├── pga-required                    # PGAReviewRequired
│   ├── dps-match                       # DPSMatchFound
│   └── uflpa-flagged                   # UFLPARiskFlagged
│
├── order.                              # Order Management
│   ├── created                         # OrderCreated
│   ├── updated                         # OrderUpdated
│   └── shipped                         # OrderShipped
│
├── shipment.                           # Shipment Lifecycle (shipper abstraction)
│   ├── created                         # ShipmentCreated ← TRIGGER EVENT
│   ├── status-changed                  # ShipmentStatusChanged
│   └── delivered                       # ShipmentDelivered
│
├── handling-unit.                      # Cargo & Handling Units
│   ├── created                         # HUCreated
│   ├── status-changed                  # HUStatusChanged ← KEY EVENT
│   ├── consolidated                    # HUConsolidated
│   ├── deconsolidated                  # HUDeconsolidated
│   ├── movement                        # HUMovement
│   ├── customs-status-changed          # HUCustomsStatusChanged
│   └── cage.                           # HU detention
│       ├── intake                      # HUCageIntake
│       ├── released                    # HUCageReleased
│       └── go-warning                  # HUGOWarning
│
├── consolidation.                      # Consolidation
│   ├── created                         # ConsolidationCreated
│   ├── closed                          # ConsolidationClosed
│   ├── status-changed                  # ConsolidationStatusChanged
│   ├── nested                          # ConsolidationNested
│   ├── movement                        # ConsolidationMovement
│   ├── customs-status-changed          # ConsolidationCustomsStatusChanged
│   ├── deconsolidation-started         # DeconsolidationStarted
│   ├── deconsolidation-blocked         # DeconsolidationBlocked
│   └── deconsolidated                  # ConsolidationDeconsolidated
│
├── declaration.                        # Declaration Management (BROKER)
│   ├── created                         # DeclarationCreated
│   ├── broker-assigned                 # BrokerAssigned
│   ├── checklist-updated               # ChecklistUpdated
│   ├── submitted                       # DeclarationSubmitted ← KEY EVENT
│   ├── amended                         # DeclarationAmended
│   ├── cf28-responded                  # CF28Responded
│   ├── cf29-responded                  # CF29Responded
│   ├── protest-filed                   # ProtestFiled
│   ├── isf-filed                       # ISFFiled
│   ├── isf-amended                     # ISFAmended
│   ├── isf-late                        # ISFLateFiling (penalty)
│   ├── ens-filed                       # ENSFiled (v5: EU ICS2)
│   ├── export-created                  # ExportDeclarationCreated (v5.2)
│   ├── export-submitted                # ExportDeclarationSubmitted (v5.2) ← KEY EVENT
│   ├── export-cleared                  # ExportDeclarationCleared (v5.2)
│   ├── export-held                     # ExportDeclarationHeld (v5.2)
│   ├── export-amended                  # ExportDeclarationAmended (v5.2)
│   └── export-rejected                 # ExportDeclarationRejected (v5.2)
│
├── adjudication.                       # Customs Adjudication (AUTHORITY)
│   ├── decision                        # AdjudicationDecision ← KEY EVENT
│   ├── cf28-issued                     # CF28Issued
│   ├── cf29-issued                     # CF29Issued
│   ├── exam-scheduled                  # ExamScheduled
│   ├── protest-resolved                # ProtestResolved
│   ├── export-decision                 # ExportAdjudicationDecision (v5.2)
│   ├── us.*                            # US-specific (CBP)
│   ├── eu.*                            # EU-specific (UCC)
│   ├── br.*                            # Brazil (Siscomex)
│   ├── in.*                            # India (ICEGATE)
│   └── cn.*                            # China (GACC)
│
├── exception.                          # Exception Management
│   ├── created                         # ExceptionCreated
│   ├── updated                         # ExceptionUpdated
│   ├── hold-released                   # HoldReleased
│   ├── hold-escalated                  # HoldEscalated
│   └── disruption.
│       └── detected
│
├── financial.                          # Financial Settlement
│   ├── fees-calculated                 # FeesCalculated
│   ├── demurrage-accruing              # DemurrageAccruing
│   ├── demurrage-stopped               # DemurrageStopped
│   ├── bond-sufficiency-warning        # BondSufficiencyWarning
│   ├── payment-scheduled               # PaymentScheduled
│   ├── payment-completed               # PaymentCompleted
│   ├── duty-liquidated                 # DutyLiquidated
│   ├── drawback-filed                  # DrawbackFiled
│   ├── drawback-approved               # DrawbackApproved
│   ├── reconciliation-applied          # ReconciliationApplied
│   └── settled                         # FinancialSettled
│
├── regulatory.                         # Regulatory Intelligence
│   ├── signal-detected                 # RegulatorySignalDetected
│   └── impact-assessed                 # RegulatoryImpactAssessed
│
├── document.                           # Document Management
│   ├── uploaded                        # DocumentUploaded
│   ├── validated                       # DocumentValidated
│   ├── discrepancy                     # DocumentDiscrepancyFound
│   └── generated                       # DocumentGenerated
│
├── party.                              # Party Management (v5)
│   ├── created                         # PartyCreated
│   ├── updated                         # PartyUpdated
│   ├── poa-granted                     # POAGranted
│   └── poa-revoked                     # POARevoked
│
sim.                                    # SIMULATION PLATFORM (separate root)
├── clock.tick | speed-changed | paused | resumed
├── actor.{name}.started | stopped | error
└── coordination.scenario-started | scenario-completed
```

### 4.2 JetStream Streams

| Stream Name | Subjects | Max Age | Storage | Purpose |
|-------------|----------|---------|---------|---------|
| `CLEARANCE_PRODUCT` | `clearance.product.>` | 30d | File | Product catalog events |
| `CLEARANCE_TRADE_INTEL` | `clearance.trade-intel.>` | 90d | File | Classification, tariff, FTA events |
| `CLEARANCE_COMPLIANCE` | `clearance.compliance.>` | 90d | File | Compliance screening events |
| `CLEARANCE_ORDER` | `clearance.order.>` | 30d | File | Order lifecycle events |
| `CLEARANCE_SHIPMENT` | `clearance.shipment.>` | 90d | File | Shipment convenience abstraction events (low volume) |
| `CLEARANCE_HANDLING_UNIT` | `clearance.handling-unit.>` | 90d | File | HU lifecycle, movement, customs, cage |
| `CLEARANCE_CONSOLIDATION` | `clearance.consolidation.>` | 90d | File | Consolidation lifecycle, movement, customs |
| `CLEARANCE_DECLARATION` | `clearance.declaration.>` | 90d | File | Broker filing events |
| `CLEARANCE_ADJUDICATION` | `clearance.adjudication.>` | 90d | File | Authority decision events |
| `CLEARANCE_EXCEPTION` | `clearance.exception.>` | 90d | File | Exception/hold/cage events |
| `CLEARANCE_FINANCIAL` | `clearance.financial.>` | 180d | File | Financial settlement events |
| `CLEARANCE_REGULATORY` | `clearance.regulatory.>` | 365d | File | Regulatory signal events |
| `CLEARANCE_DOCUMENT` | `clearance.document.>` | 90d | File | Document lifecycle events |
| `CLEARANCE_PARTY` | `clearance.party.>` | 90d | File | Party management events (v5) |
| `SIM` | `sim.>` | 7d | Memory | Simulation-only events (short-lived) |

### 4.3 Key Consumer Subscriptions

| Consumer | Subscribes To | Purpose |
|----------|--------------|---------|
| **Trade Intelligence** | `clearance.product.created`, `clearance.product.updated`, `clearance.shipment.created`, `clearance.regulatory.signal-detected` | Proactive classification + tariff |
| **Compliance** | `clearance.shipment.created`, `clearance.trade-intel.classification.produced`, `clearance.regulatory.signal-detected` | Proactive screening |
| **Order Management** | `clearance.trade-intel.classification.produced`, `clearance.trade-intel.tariff.calculated`, `clearance.compliance.screened` | Assemble analysis |
| **Shipment Lifecycle** | `clearance.order.shipped`, `clearance.handling-unit.status-changed`, `clearance.trade-intel.>`, `clearance.compliance.screened` | Create shipments, maintain HU state projection |
| **Cargo & Handling Units** | `clearance.consolidation.movement`, `clearance.consolidation.status-changed`, `clearance.consolidation.deconsolidated`, `clearance.adjudication.decision`, `clearance.exception.hold-released` | Derive position from parent, update customs state |
| **Consolidation** | `clearance.consolidation.movement` (parent), `clearance.adjudication.decision` | Derive position when nested, update customs status (v5: deconsolidation eligibility is query-time computed) |
| **Party Management** | — | Pure producer — no subscriptions |
| **Declaration Mgmt** | `clearance.shipment.created`, `clearance.handling-unit.created`, `clearance.trade-intel.>`, `clearance.compliance.screened`, `clearance.adjudication.>`, `clearance.document.validated` | Build broker workbench |
| **Customs Adjudication** | `clearance.declaration.submitted`, `clearance.declaration.amended`, `clearance.declaration.cf28-responded`, `clearance.declaration.cf29-responded`, `clearance.declaration.protest-filed` | Receive broker submissions |
| **Exception Mgmt** | `clearance.adjudication.>`, `clearance.compliance.>`, `clearance.handling-unit.status-changed` | Create holds from decisions |
| **Financial** | `clearance.trade-intel.tariff.calculated`, `clearance.handling-unit.>`, `clearance.consolidation.>`, `clearance.declaration.submitted`, `clearance.adjudication.decision`, `clearance.adjudication.cf29-issued` | Track costs, manage bonds, process payments |
| **Document Mgmt** | `clearance.shipment.created`, `clearance.trade-intel.classification.produced` | Determine requirements |
| **Redis Cache Projector** | `clearance.>` (all) | Pre-compute read models |

### 4.4 State-Transfer: What Each Domain Stores Locally

State-transfer events carry **full snapshots**. Consuming domains store what they need locally — no cross-domain queries at runtime.

| Consuming Domain | Source Event | What It Stores Locally | Why |
|-----------------|-------------|----------------------|-----|
| **Shipment Lifecycle** | `HUStatusChanged` | HU status, location, customs status | Updates handling_unit_summary projection, derives aggregate shipment status |
| **Cargo & Handling Units** | `ConsolidationMovement` | Location, timestamp | Derives own position when consolidated |
| **Cargo & Handling Units** | `ConsolidationStatusChanged` | Consolidation status | Updates own status to reflect parent |
| **Cargo & Handling Units** | `ConsolidationDeconsolidated` | Deconsolidation event | Clears consolidated_into, resumes direct tracking |
| **Cargo & Handling Units** | `AdjudicationDecision` | Decision, hold type | Updates import_clearance, hold_type |
| **Consolidation** | `ConsolidationMovement` (parent) | Location, timestamp | Derives own position when nested |
| Note (v5): Consolidation no longer subscribes to HU/child customs status changes for `can_deconsolidate`. Deconsolidation readiness is computed on-demand at query time. |
| **Declaration Mgmt** | `ClassificationProduced` | HS code, confidence, tariff heading | Populates checklist "classification" item, includes in filing data |
| **Declaration Mgmt** | `TariffCalculated` | Total duty, effective rate, programs | Populates checklist "tariff" item, broker sees duty estimate |
| **Declaration Mgmt** | `ComplianceScreened` | Overall status, PGA flags, hold flags | Populates checklist "compliance" item, gates filing readiness |
| **Declaration Mgmt** | `DocumentValidated` | Document type, validation status | Populates checklist "documents" item |
| **Declaration Mgmt** | `AdjudicationDecision` | Decision type, hold type, CF-28/CF-29 detail | Updates filing status, creates CBP response work items |
| **Financial** | `TariffCalculated` | Full tariff breakdown | Computes duty deposit, schedules payment |
| **Financial** | `HUCageIntake` / `HUCageReleased` | Facility type, storage rate, timestamps | Starts/stops storage cost accrual |
| **Financial** | `HUStatusChanged` (at_customs) | Arrival timestamp, border_crossing_mode | Starts D&D clock (ocean: vessel discharge, air: 48h) |
| **Exception Mgmt** | `AdjudicationDecision` (held) | Hold type, exam type, response deadline | Creates exception record with cage subprocess if exam |
| **Exception Mgmt** | `ComplianceScreened` (HOLD) | Compliance failure details | Creates compliance hold exception |
| **Order Mgmt** | `ClassificationProduced` | HS code per line item | Updates OrderLineItem.hs_code |
| **Order Mgmt** | `TariffCalculated` | Duty per line item | Updates OrderLineItem.duty_amount |
| **Order Mgmt** | `ComplianceScreened` | Compliance status per line item | Updates OrderLineItem.compliance_status |

**Key principle:** Each domain stores a **local copy** of the data it needs. The Declaration Management domain's entry filing includes tariff data NOT because it owns tariff calculation, but because the `TariffCalculated` event delivered it. This eliminates runtime cross-domain queries. The Redis Cache Projector aggregates ALL of this into the denormalized shipment view for the UI.

---

## 5. Intelligence Services

### 5.1 Intelligence Ownership

Each intelligence engine is a proper component **within** its owning domain. Intelligence flows to consumers **via events**.

**Pipeline → Event Choreography:** The current monolith's `AnalysisPipeline` (E1 → parallel(E2, E3, E4) → assembly) is **eliminated**. There is no orchestrator. Instead:
1. `ProductCreated` → Trade Intelligence reacts → `ClassificationProduced`
2. `ClassificationProduced` → Trade Intelligence (tariff), Compliance, Declaration Mgmt **all react independently and concurrently**
3. Results accumulate in the Redis cache via the Cache Projector
4. No service waits for another. No synchronous call chains. Pure event choreography.

| Intelligence Engine | Owning Domain | Trigger Events | Output Events | LLM Required |
|--------------------|---------------|----------------|---------------|--------------|
| **Classification** (e1) | Trade Intelligence | `ProductCreated`, `ProductUpdated` | `ClassificationProduced` | Yes |
| **Tariff** (e2) | Trade Intelligence | `ShipmentCreated`, `ClassificationProduced`, `RegulatorySignalDetected` | `TariffCalculated` | No |
| **FTA** (e4) | Trade Intelligence | `ClassificationProduced` | `FTADetermined` | No |
| **Compliance** (e3) | Compliance & Screening | `ShipmentCreated`, `ClassificationProduced`, `RegulatorySignalDetected` | `ComplianceScreened` | No |
| **DPS Screening** | Compliance & Screening | `ShipmentCreated` | `DPSMatchFound` | No |
| **Exception Analysis** (e5) | Exception Management | `ExceptionCreated` | Inline result | Yes |
| **Regulatory Monitor** (e6) | Regulatory Intelligence | Periodic/manual | `RegulatorySignalDetected` | Yes |
| **Document Engine** (e7) | Document Management | Explicit request | `DocumentGenerated` | Partial |
| **Description Quality** | Product Catalog | `ProductCreated`, `ProductUpdated` | Inline result | Yes |
| **Broker Intelligence** | Declaration Management | `DeclarationCreated`, accumulated intelligence events | Inline result | Yes |
| **Filing Intelligence** | Declaration Management | `CF28Issued`, explicit request | Inline result | Yes |
| **Pre-Clearance** (e0) | Compliance & Screening | `ShipmentCreated` | Feeds into `ComplianceScreened` | Yes |

### 5.2 Qdrant Knowledge Stores

| Collection | Scope | Domain | Content |
|-----------|-------|--------|---------|
| `cross_rulings` | Scoped | Trade Intelligence | CBP binding rulings, classification precedents |
| `tariff_rulings` | Scoped | Trade Intelligence | Tariff determination precedents |
| `compliance_precedents` | Scoped | Compliance | Past screening outcomes |
| `filing_precedents` | Scoped | Declaration Mgmt | Past entry filings, checklist patterns |
| `cbp_responses` | Scoped | Declaration Mgmt | CF-28/CF-29 response templates, protest outcomes |
| `resolution_precedents` | Scoped | Exception Mgmt | Hold resolution histories |
| `hs_taxonomy` | Global | — | HS code hierarchy, GRI notes |
| `denied_parties` | Global | — | Restricted party list embeddings |
| `regulatory_corpus` | Global | — | Federal Register, policy documents |

---

## 6. Redis State Cache Design

### 6.1 Architecture

A single **Redis Cache Projector** process subscribes to all `clearance.>` events via NATS and maintains pre-computed read models.

```
NATS (clearance.>)  ──→  Redis Cache Projector  ──→  Redis
                                                       │
                         entity:product:{id}           │
                         entity:order:{id}             │
                         entity:shipment:{id}          │ ← Shipper-facing view
                         entity:handling-unit:{id}     │ ← Full HU state
                         entity:consolidation:{id}     │ ← Full consolidation state
                         entity:declaration:{id}       │ ← Denormalized:
                         entity:classification:{id}    │    includes intelligence
                         entity:tariff:{id}            │    from Trade Intel,
                         entity:exception:{id}         │    Compliance, Financial
                         entity:financial:{id}         │
                         entity:adjudication:{id}      │
                         entity:regulatory:{id}        │
                         entity:document:{id}          │
                         entity:party:{id}             │ ← v5: Party Management
                         entity:poa:{id}               │
                                                       │
                         consolidation:{id}:all_hus    │ ← Flattened: all HU IDs
                                                       │    in consolidation (recursive)
                                                       │
                         idx:shipments:by-status:{s}   │ ← Sorted sets
                         idx:shipments:by-broker:{b}   │    for list queries
                         idx:handling-units:by-shipment:{id}  │
                         idx:handling-units:by-consolidation:{id} │
                         idx:consolidations:by-status:{s}  │
                         idx:declarations:by-status:{s}│
                         idx:exceptions:by-severity:{s}│
                                                       │
                         queue:broker:unassigned        │ ← Broker queue
                         queue:broker:{id}:active       │    (priority sorted)
                         queue:broker:cbp-responses     │
                                                       │
                         agg:dashboard:platform         │ ← Aggregates
                         agg:dashboard:broker:{id}      │
                         agg:compliance:summary         │
                         agg:corridor:matrix            │
```

### 6.2 Cached Views

Three primary cached views serve different audiences:

**Shipper-facing: Denormalized Shipment View**

The shipment view is the shipper's primary interface -- a simplified, non-authoritative projection that hides logistics/regulatory complexity. It assembles HU state projections and intelligence results into a single JSON.

```json
{
  "id": "uuid",
  "status": "in_transit",
  "origin": "CN", "destination": "US",
  "product": "Wireless Bluetooth Headphones",
  "product_id": "uuid",
  "company_name": "Apex Trading Co.",
  "declared_value": 15000.00,

  // -- Handling unit state projection (from HU events, NON-AUTHORITATIVE) --
  "handling_unit_summary": [
    {
      "handling_unit_id": "uuid-hu-1",
      "type": "pallet",
      "status": "in_transit",
      "location": "Pacific Ocean — en route to USLAX",
      "consolidated_into": "uuid-consolidation-1",
      "customs_status": "pending"
    },
    {
      "handling_unit_id": "uuid-hu-2",
      "type": "carton",
      "status": "cleared",
      "location": "USLAX",
      "consolidated_into": null,
      "customs_status": "cleared"
    }
  ],

  // -- Intelligence results (convenience copies from Trade Intelligence events) --
  "classification": {
    "hs_code": "8518.30",
    "description": "Wireless Bluetooth Headphones",
    "confidence": "HIGH",
    "classified_at": "2026-02-01T08:05Z"
  },

  "tariff": {
    "total_duty": 1234.56,
    "effective_rate": 0.082,
    "programs": [
      { "program": "Section 301 List 3", "rate": 0.25, "amount": 3750.00 }
    ],
    "mpf": 51.96,
    "hmf": 18.75,
    "fta_eligible": false
  },

  // -- Compliance (from Compliance & Screening events) --
  "compliance": {
    "overall_status": "CLEAR",
    "screened_at": "2026-02-01T08:10Z"
  },

  // -- Financial summary (from Financial Settlement events) --
  "financial": {
    "phase": "provisional",
    "total_duty": 1234.56,
    "total_fees": 70.71,
    "total_financial_exposure": 1305.27,
    "payment_status": "scheduled"
  },

  // -- Documents (commercial, attached at shipment level) --
  "documents": {
    "required": ["commercial_invoice", "packing_list"],
    "uploaded": ["commercial_invoice", "packing_list"],
    "validated": ["commercial_invoice", "packing_list"],
    "missing": []
  }
}
```

**Broker-facing: Handling Unit View**

The HU view is the broker's primary entity for customs work -- full customs state, movement history, cage/detention details, and declaration status. `GET /handling-units/{id}` returns the full `HandlingUnit` entity from Redis including `cage_status`, `transit_history`, and current `import_clearance`/`export_clearance`.

**Broker-facing: Consolidation View**

The consolidation view shows the transport grouping -- contained HUs and nested consolidations, movement events, customs state, and deconsolidation eligibility. `GET /consolidations/{id}` returns the full `Consolidation` entity. `GET /consolidations/{id}/all-handling-units` returns from the `consolidation:{id}:all_hus` Redis cache.

**Design principle:** Each view serves a different audience. The shipper gets a simplified projection. The broker gets the physical/regulatory reality. `GET /shipments/{id}` returns the shipper view. `GET /handling-units/{id}` and `GET /consolidations/{id}` return the broker views. All from Redis. Zero service calls. Zero database queries. The UI renders these directly. Every domain's events update their specific sections via the Redis Cache Projector.

### 6.3 Consistency Model

- **Eventual consistency** with <100ms typical lag
- **State-transfer events** carry full snapshots — each update is a full replacement, no partial update issues
- **Periodic reconciliation** (5-min) compares Redis with PostgreSQL source
- **Writes are immediately consistent** — domain services return the new state directly

**Write path clarification (v5):** Writes go DIRECTLY to PostgreSQL via API. The API returns the new state immediately in the HTTP response (bypassing Redis). The state store then emits an event asynchronously for other consumers. Events are the OUTPUT of writes, not the INPUT. There is no event in the write critical path. This is a documented contract — for write endpoints, the HTTP response is the authoritative new state. The Redis cache is updated asynchronously via the Cache Projector for subsequent read queries.

### 6.4 Event Delivery Guarantees

#### At-Least-Once Delivery

NATS JetStream provides **at-least-once delivery** with durable consumers. Every consumer acknowledges each message explicitly. If a consumer crashes before acknowledging, JetStream redelivers.

**Consequence:** Every event consumer MUST be idempotent. Processing the same event twice must produce the same result.

#### Idempotency Strategy

| Consumer Type | Idempotency Key | Deduplication Method |
|--------------|-----------------|---------------------|
| **Domain service** (writes to PostgreSQL) | `event_id` | `INSERT ... ON CONFLICT (event_id) DO NOTHING` on a `processed_events` table per schema. Check before processing. |
| **Redis Cache Projector** (writes to Redis) | `event_id` + `entity_id` + `version` | Compare `version` field in event payload against current cached version. Ignore if stale. State-transfer events are full snapshots, so replaying a newer version is safe. |
| **Intelligence engines** (trigger computation) | `entity_id` + `entity_version` | Skip if a result for this entity version already exists. Classification of product v3 is idempotent — re-running produces the same classification. |

Each domain schema includes a `processed_events` table:
```sql
CREATE TABLE {schema}.processed_events (
  event_id UUID PRIMARY KEY,
  event_type TEXT NOT NULL,
  processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
-- TTL: purge events older than stream max_age (matching JetStream retention)
```

#### Event Ordering

NATS JetStream delivers events **in publish order per subject**. For entity-level ordering:

- Events include `entity_id` in the subject token where ordering matters: `clearance.shipment.{shipment_id}.status-changed`
- The Redis Cache Projector uses **per-entity version numbers** (monotonically increasing). If an out-of-order event arrives with a lower version, it is discarded.
- For the denormalized shipment view, each domain section has its own version counter. A tariff update at version 5 does not conflict with a compliance update at version 3 — they update different sections.

#### Dead Letter Handling

Events that fail processing after **3 retry attempts** (with exponential backoff: 1s, 5s, 30s) are routed to a dead letter subject:

```
clearance.dlq.{original-subject}
```

| Stream | DLQ Subject | Alert | Recovery |
|--------|------------|-------|----------|
| `CLEARANCE_*` | `clearance.dlq.>` | PagerDuty alert after 10 DLQ messages in 5 min | Manual replay via admin API after root cause fix |

DLQ messages retain the full original event payload plus failure metadata (error message, attempt count, last failure timestamp). A dedicated DLQ monitor process aggregates and alerts.

**DLQ consumer contract:** Domain services expose a `/admin/replay` endpoint that accepts an event payload and reprocesses it. This is the standard recovery path after fixing the root cause.

#### Consumer Failure Modes

| Failure | JetStream Behavior | Consumer Responsibility |
|---------|-------------------|----------------------|
| **Crash before ack** | Redelivers after `AckWait` (30s default) | Idempotent processing handles duplicate |
| **Persistent failure** | Retries up to `MaxDeliver` (3) | Routes to DLQ after exhausting retries |
| **Slow consumer** | `AckWait` expires, redelivers | Consumer must ack within timeout or NAK explicitly |
| **Schema mismatch** | Consumer can't parse payload | NAK + DLQ. Events include `version` for schema evolution. |

---

## 7. Broker Intelligence Flow

Intelligence accumulates proactively in the broker's workbench:

```
Time ──────────────────────────────────────────────────────────────────►

ShipmentCreated event
    │
    ├──► Trade Intelligence subscribes
    │    └──► Classifies product → emits ClassificationProduced
    │         └──► Computes tariff → emits TariffCalculated
    │              └──► Checks FTA → emits FTADetermined
    │
    ├──► Compliance subscribes
    │    └──► Screens entity/product → emits ComplianceScreened
    │
    ├──► Document Mgmt subscribes
    │    └──► Determines required docs → updates cache
    │
    └──► Declaration Mgmt subscribes
         └──► Creates draft declaration
              └──► Subscribes to Trade Intel + Compliance events
                   └──► As each intelligence event arrives:
                        • Updates declaration checklist
                        • Evaluates readiness
                        • Broker sees items turning green

    [ALL of the above happen CONCURRENTLY within seconds]

    Broker opens workbench:
    ┌─────────────────────────────────────────────────┐
    │ Shipment SHP-2026-0142 — READY TO FILE          │
    │                                                  │
    │ ✅ Classification: HS 8471.30 (95% confidence)  │
    │ ✅ Tariff: $1,234.56 total duty (8.2%)          │
    │ ✅ Compliance: CLEAR                             │
    │ ✅ FTA: Not eligible (origin CN)                 │
    │ ✅ Documents: 3/3 uploaded, 3/3 validated        │
    │ ✅ Bond: Continuous bond on file                 │
    │                                                  │
    │ [Approve & Submit]                               │
    └─────────────────────────────────────────────────┘

    Broker clicks Submit → DeclarationSubmitted event
    → Customs Adjudication subscribes → Adjudicates → AdjudicationDecision event
    → (during transit!) → HU import_clearance → cleared → Shipment derives aggregate 'cleared'
```

**Key insight:** The broker receives intelligence by **subscribing to events from Trade Intelligence, Compliance, and Document Management**. They never call those domains' APIs. Intelligence flows TO them.

### 7.1 Filing Readiness Model

Declaration Management tracks a **readiness score** — a computed assessment of whether a declaration has all prerequisites to file. Each intelligence event advances readiness.

| Readiness Gate | Source Event | Required | Weight |
|---------------|-------------|----------|--------|
| Classification | `ClassificationProduced` | Yes | 20% |
| Tariff Calculation | `TariffCalculated` | Yes | 20% |
| Compliance Screening | `ComplianceScreened` (CLEAR or PGA_REQUIRED) | Yes | 20% |
| Documents | `DocumentValidated` (all required docs) | Yes | 20% |
| Bond | Bond on file or single-entry bond posted | Yes | 10% |
| ISF Filed | `ISFFiled` (ocean only) | Conditional | 10% |

**Readiness states:**
- `not_ready` — missing required gates (0-59%)
- `ready_with_warnings` — all required gates met, but advisory issues exist (60-89%)
- `ready_to_file` — all gates met, no warnings (90-100%)
- `filed` — declaration submitted

The readiness score is stored in the Redis cache on the denormalized declaration entity and updated incrementally as each intelligence event arrives. The broker queue sorts by readiness (ready-to-file entries surface for fast processing).

### 7.2 Broker Queue Priority

```
priority = base
  + (has_hold ? 1000 : 0)
  + (has_cbp_response ? 500 : 0)
  + (go_deadline_days < 5 ? 300 : 0)
  + (compliance_status == 'HOLD' ? 200 : 0)
  + (days_since_created * 10)
  - (is_ready_to_file ? 50 : 0)
```

---

## 8. Pre-Clearance Flow (End-to-End)

### 8.1 Happy Path

```
PHASE 1: Creation (t=0)
    Shipper creates order → ships → ShipmentCreated event

PHASE 2: Intelligence cascade (t=seconds)
    Trade Intel: ClassificationProduced (HS 8471.30, 95%)
    Trade Intel: TariffCalculated ($1,234.56, 8.2%)
    Compliance:  ComplianceScreened (CLEAR)
    Document:    Requirements determined (3 docs)
    Declaration: Draft created, checklist populating

PHASE 3: Pre-clearance (t=minutes, HU booked or early transit)
    Broker reviews → all green → Approve → Submit (declaration on HU/consolidation)
    DeclarationSubmitted event →
    Customs Adjudication: AdjudicationDecision (accepted)
    → HU: import_clearance=cleared (during transit!)
    → Shipment: derives aggregate status 'cleared' from HU events

PHASE 4: Transit & arrival (t=hours/days)
    Carrier advances consolidation: ConsolidationMovement events cascade to HUs
    HU arrives at customs → already cleared → immediate release

PHASE 5: Delivery
    HUs delivered → Shipment derives 'delivered' → Financial settles

TOTAL TIME AT CUSTOMS: 0 (pre-cleared during transit)
```

### 8.2 Exception Path

```
HU arrives at customs (import_clearance=pending, no declaration on file)
    → Exception Mgmt: ExceptionCreated (no_preclearance hold on HU)
    → Declaration Mgmt: priority escalated
    → Broker scrambles to file declaration for HU
    → DeclarationSubmitted → Adjudication → may issue CF-28 → more delay
    → Eventually: HoldReleased → HU import_clearance=cleared
    → Shipment derives aggregate status from HU events

This path should be <5% in a healthy system.
```

---

## 9. Simulation Platform

### 9.1 Separation Principle

The Simulation Platform:
- Uses **platform REST APIs** (same APIs humans use)
- Subscribes to `clearance.*` NATS subjects for **read-only awareness**
- Publishes **only** to `sim.*` NATS subjects
- **Cannot emit** clearance platform events directly

### 9.2 Simulation Actor → Platform API Mapping

| Simulation Actor | Platform APIs Used | NATS (read-only) | Simulates |
|-----------------|-------------------|-------------------|-----------|
| **ShipperBot** | `POST /orders`, `POST /orders/{id}/ship` | -- | Shipper creating orders |
| **CarrierBot** | `POST /handling-units/{id}/movement`, `POST /consolidations/{id}/movement`, `POST /consolidations/{id}/deconsolidate` | `clearance.handling-unit.customs-status-changed`, `clearance.consolidation.customs-status-changed`, `clearance.adjudication.decision` | Carrier advancing physical movement. Interacts with clearance instructions: if authority orders inspection, keeps HU under bond at customs facility. If pre-cleared, bypasses inspection. Cannot deconsolidate if any contained entity has active customs hold. |
| **CustomsAuthorityBot** | `POST /adjudication/declarations/{id}/decide`, `POST .../request-info` | `clearance.declaration.submitted` | Government authority |
| **BrokerBot** | `GET /broker/queue`, `POST /broker/queue/claim`, `POST .../submit` | `clearance.shipment.created`, `clearance.handling-unit.created` | Broker filing entries |
| **PGABot** | `POST /adjudication/declarations/{id}/decide` (PGA-specific) | `clearance.trade-intel.classification.produced` | PGA review |
| **ExceptionInjector** | `POST /exceptions` (new injection endpoint) | `clearance.handling-unit.>` | Random exceptions |
| **DisruptionInjector** | `POST /exceptions` (disruption type) | `clearance.handling-unit.>`, `clearance.consolidation.>` | Supply chain disruptions |
| **ShipperResponseBot** | `POST /resolution/upload-document` | `clearance.exception.created` | Shipper responding to holds |
| **DemurrageActor** | `POST /financial/{hu_id}/demurrage` | `clearance.handling-unit.status-changed`, `clearance.handling-unit.customs-status-changed` | D&D charge calculation by carrier policy |
| **ConsolidatorActor** | `POST /consolidations`, `POST /consolidations/{id}/add` | `clearance.handling-unit.created` | Groups handling units into consolidations (MAWB/MBL/manifest). Decides consolidation based on origin/destination/mode matching. |

### 9.3 Customs Authority: Platform vs Simulation

- **Platform side:** Customs Adjudication domain provides APIs for authorities to communicate decisions (`POST /adjudication/declarations/{id}/decide`).
- **Simulation side:** CustomsAuthorityBot subscribes to `clearance.declaration.submitted`, applies simulated logic (STP rates, risk profiling), calls platform API.
- **Production readiness:** Replace the bot with real EDI/API integration. Zero platform changes needed.

---

## 10. Intelligence & Analytics Layer (v5)

Reference: `docs/engine-audit-for-architecture.md`

The Intelligence & Analytics Layer is a **pure event consumer layer** built on the domain model. It does not own transactional state — it consumes domain events and produces intelligence that flows back to domains via events.

### 10.1 Five First-Class Intelligence Domains

The following are proper domains (already modeled in Sections 3.2, 3.3, 3.12) that collectively form the intelligence layer:

| # | Intelligence Domain | Trigger Mechanism | Uses LLM | Uses Qdrant | Key Capability |
|---|-------------------|-------------------|----------|-------------|----------------|
| 1 | **Classification** (Trade Intelligence) | `ProductCreated`, `ProductUpdated` events | Yes (E1) | No | Two-phase HS code classification (chapter → subheading). Proactive — classifies at earliest possible moment. |
| 2 | **Tariff & Duty** (Trade Intelligence) | `ShipmentCreated`, `ClassificationProduced`, `RegulatorySignalDetected` events | No (E2, fully deterministic) | No | Jurisdiction-specific duty/tax/fee computation via 6 `TaxRegimeEngine` subclasses (US, EU-27, CN, BR, IN, GB). |
| 3 | **Compliance** (Compliance & Screening) | `ShipmentCreated`, `ClassificationProduced` events | No (E3, deterministic) | No | PGA + DPS + UFLPA screening. Three sub-engines run concurrently via `asyncio.gather`. |
| 4 | **Trade Programs** (Trade Intelligence) | `ClassificationProduced` events | No (E4, deterministic) | No | FTA qualification (USMCA currently; extensible to RCEP, CPTPP, EU FTAs). |
| 5 | **Regulatory Intelligence** | Periodic feed polling + manual | Yes (E6 for signal extraction) | No | Signal feed, scenario modeling, portfolio impact. 4-source integration (Federal Register, CBP CSMS, USTR, Google News). |

Additional intelligence engines that serve specific domains:
- **Pre-Clearance** (E0): Agentic tool-calling loop for booking screening. Uses LLM. Feeds into Compliance domain.
- **Exception Analysis** (E5): CROSS rulings search + LLM response drafting. Uses LLM + Qdrant. Serves Exception Management.
- **Document Intelligence** (E7): Document requirements lookup. Deterministic. Serves Document Management.
- **Description Quality**: Pre-classification quality gate with risk assessment. Uses LLM (optional). Serves Product Catalog.
- **Broker Intelligence**: Work prioritization, complexity scoring, financial impact. Pure computation. Serves Declaration Management.

### 10.2 Analytics Sub-Layer

The analytics sub-layer is a pure event consumer that produces derived intelligence:

| Analytics Capability | Consumes | Produces | Current Location |
|---------------------|----------|----------|-----------------|
| **KPI Aggregation** | shipment.analyzed, entry.filed, entry.cleared | Dashboard snapshots (materialized views) | `dashboard.py` |
| **Compliance Monitoring** | entry.filed, shipment.held, shipment.cleared | Hold rate, compliance rate, by-status/by-jurisdiction breakdowns | `platform.py` |
| **Trade Lane Optimization** | tariff.calculated, compliance.screened | Corridor cost rankings, multi-destination comparison | `trade_lanes.py` |
| **Broker Intelligence** | entry.claimed, checklist.updated, hold.created | Priority scores, capacity planning, work sequencing | `broker_intelligence.py` |
| **Portfolio Analysis** | regulatory.signal_created, product.cataloged | Exposure analysis, affected products/value | E6 sub-capability |
| **Anomaly Detection** | All domain events | Tariff spikes, compliance pattern changes, unusual hold rates | New (v5) |

### 10.3 Capabilities from Current Code (must be preserved)

The following 17 capabilities exist in the current codebase and must be preserved in the target architecture:

1. Broker work sequencing algorithm (composite scoring with deadline pressure, dependency chains)
2. AI pre-work caching (Redis-cached CF-28 draft preparation)
3. Trade lane parallel comparison (multi-destination asyncio.gather with sorted cost output)
4. Order multi-line analysis (per-line-item E1+E2+E3 with aggregation)
5. Regulatory feed parsing (4-source integration: Federal Register API, CBP CSMS, USTR, Google News)
6. LLM-powered regulatory signal extraction (batch extraction + 3-tier dedup)
7. Disruption event generation (geographically/seasonally aware with AI descriptions)
8. ISF 10+2 filing compliance (late filing detection, mismatch types, penalty calculation)
9. Cargo exception modeling (mode/product-aware damage, shortage, overage, weight variance)
10. Terminal operations (berth delays, chassis shortage, port congestion factors)
11. Document discrepancy detection (invoice/description/weight/origin cert variance)
12. APHIS ISPM-15 wood packaging detection + fumigation requirements
13. TTB alcohol/tobacco permit type assignment
14. Payment method distribution (ACH/wire/PMS/check selection logic)
15. Exam fee determination (VACIS/intensive/tailgate fee assignment)
16. Consolidation grouping (mode+origin+destination+carrier with MAWB/MBL generation)
17. Shipper response quality modeling (channel/urgency-based response timing)

---

## 11. Jurisdiction Adapter Pattern (v5)

Reference: `docs/review-jurisdiction-deep-dive.md`

### 11.1 Pattern Overview

Each jurisdiction implements a common interface. Adding a new jurisdiction = adapter configuration, no domain model changes.

```
JurisdictionAdapter (interface)
├── import_filing_workflow: StateMachine  # Import declaration lifecycle (per jurisdiction)
├── export_filing_workflow: StateMachine  # (v5.2) Export declaration lifecycle (per jurisdiction)
├── duty_calculator: DutyEngine           # Different FORMULA per jurisdiction
├── import_status_machine: StatusMachine  # Uses two-layer model (jurisdiction_status → platform_status)
├── export_status_machine: StatusMachine  # (v5.2) Export-specific status mapping
├── party_validator: PartyValidator       # Different registrations per jurisdiction
├── import_document_requirements: DocSpec # Different docs per jurisdiction for import
├── export_document_requirements: DocSpec # (v5.2) Different docs per jurisdiction for export
├── electronic_system: SystemConnector    # Different integration per jurisdiction
├── import_checklist_generator: ChecklistSpec  # Import readiness criteria per jurisdiction
├── export_checklist_generator: ChecklistSpec  # (v5.2) Export readiness criteria per jurisdiction
├── export_license_required(hs_code, destination, value): boolean  # (v5.2) Whether export
│                                          # license/permit is required for this commodity to this
│                                          # destination. US: EAR/ITAR screening. EU: dual-use.
│                                          # CN: export control list. Drives checklist item.
├── export_filing_required(origin, destination, value, commodity): boolean  # (v5.2) Whether
│                                          # export declaration is required. Some jurisdictions
│                                          # exempt low-value or intra-bloc shipments.
│                                          # US: >$2,500 or any controlled/licensed item.
│                                          # EU: intra-EU = no export dec. CA: >$2,000 to non-US
│                                          # or controlled goods. Sets HU export_clearance to
│                                          # not_required when false.
└── register_product(product, jurisdiction): RegistrationResult?  # (v5.1) Register product in
                                           # jurisdiction-specific catalogs before filing.
                                           # For Brazil: maps to Siscomex Catalogo de Produtos registration
                                           # (mandatory for DUIMP). Returns registration_id stored in
                                           # Product.catalog_registrations. No-op for jurisdictions
                                           # without product pre-registration requirements.
```

### 11.2 Primary Jurisdictions

| Jurisdiction | Electronic System | Filing Workflow | Duty Formula | Key Party Registration |
|-------------|------------------|-----------------|-------------|----------------------|
| **US** (CBP/ACE) | ACE via ABI | Entry/release (3461) → Entry summary (7501) | Additive: duty + MPF + HMF + Section 301 + IEEPA + AD/CVD | Bond, POA, ABI access |
| **EU** (ICS2/UCC) | ICS2 + national systems (ATLAS, DELTA, AGS) | ENS (carrier) → Import declaration (H1-H7) → Supplementary | Customs duty + AD/CVD + member-state VAT | EORI (mandatory for ALL parties) |
| **CN** (Single Window/GACC) | China Single Window | Manifest → Declaration → Channel → Assessment → Release → Close | Import duty + VAT + Consumption tax | Customs registration, GACC (food/cosmetics) |
| **BR** (Portal Único/Siscomex) | Portal Único | Product catalog → LPCO → DUIMP → Parametrization → NF-e | Cascading: II → IPI → PIS/COFINS → ICMS → AFRMM | RADAR (gates ALL imports), CNPJ |
| **IN** (ICEGATE) | ICEGATE + SWIFT 2.0 | IGM (carrier) → BoE → RMS → Assessment → OOC | Sequential: BCD → SWS → IGST | IEC (mandatory), DSC |

### 11.3 Secondary Jurisdictions

| Jurisdiction | Electronic System | Filing Workflow | Key Difference |
|-------------|------------------|-----------------|---------------|
| **MX** (VUCEM/SAT) | VUCEM / SAAI | Pre-validation → Pedimento → Payment → Modulación | Mandatory pre-validation, 76 pedimento clave codes |
| **CA** (CARM/CBSA) | CARM Client Portal + API | ACI (carrier) → Release → CAD (5 days) → Statement | Two-step (like US), monthly statement payment cycle |

### 11.4 Export Filing Workflows by Jurisdiction (v5.2)

Every international shipment has an origin country that may require export declarations. The export filing workflow runs in parallel with (but independent of) the import filing workflow. The `export_filing_required` method on the adapter determines whether filing is needed.

| Jurisdiction | Export Filing System | Export Declaration Type | Filing Trigger | Key Requirements |
|-------------|---------------------|----------------------|----------------|-----------------|
| **US** (AES/AESDirect) | AES via AESDirect or ABI | EEI (Electronic Export Information) | Value >$2,500, OR any ITAR/EAR-controlled, OR destination on denied list | Schedule B code (not HTS), ECCN for dual-use, ultimate consignee type, routed vs non-routed transaction. ITN returned on acceptance. |
| **EU** (ECS/national) | ECS (Export Control System) via national customs (ATLAS, DELTA, etc.) | Export declaration (types A/B/C) | All extra-EU exports (intra-EU = free movement, NO export dec) | 4-digit CPC procedure code, export MRN assigned, physical release at office of exit. Dual-use regulation (EU 2021/821). |
| **CN** (Single Window) | China Customs Single Window | Export customs declaration | All commercial exports | Supervision mode (general trade, processing), CIQ inspection for food/cosmetics/chemicals, RMB settlement requirements. |
| **BR** (Portal Único) | Portal Único (Siscomex) | DU-E (Declaração Única de Exportação) | All exports (replaced RE+DE in 2017) | NCM classification, CFOP tax code, linked to NF-e (electronic invoice). SUFRAMA benefits for Manaus FTZ. |
| **IN** (ICEGATE) | ICEGATE | Shipping Bill (4 types: free/dutiable/drawback/ex-bond) | All exports | Let Export Order (LEO) issued by customs officer. Drawback claims on dutiable exports. GST refund on zero-rated exports. IEC mandatory. |
| **MX** (VUCEM) | VUCEM / SAAI | Pedimento de exportación (clave A1 = definitive) | All exports | Pre-validation mandatory (same as import). RFC required. IMMEX program for maquiladora temporary imports re-exported. |
| **CA** (CERS) | CERS (Canadian Export Reporting System) | CAED (Canadian Automated Export Declaration) | >$2,000 to non-US, OR controlled goods/technology, OR export permit required | BN required. Exemptions for US-bound under $2,000. Export permits for controlled goods (Export Control List). |

**Export Filing Lifecycle (generic):**
```
draft → pending_review → submitted → [accepted | held | rejected]
                                        ↓
                                     cleared → (goods released for export)
                                        ↓
                                   HU.export_clearance = cleared
```

**Export Checklist Items (jurisdiction-dependent, populated by adapter):**
| Checklist Key | Description | When Required |
|---------------|-------------|---------------|
| `export_license` | Export license/permit obtained or NLR determination | Controlled items (EAR/ITAR/dual-use/national lists) |
| `sanctions_screening` | Consignee/destination not on sanctions/denied lists | Always |
| `hs_classification` | HS/Schedule B code verified | Always |
| `commercial_invoice` | Commercial invoice for export | Always |
| `packing_list` | Packing list with weights/quantities | Always |
| `origin_certificate` | Certificate of origin (if FTA/preferential) | When claiming preferential treatment |
| `dangerous_goods` | DG declaration (IATA/IMO) | Hazardous materials |
| `dual_use_check` | EU dual-use screening (Reg 2021/821) | EU exports of listed items |
| `phytosanitary` | Phytosanitary certificate | Agricultural/plant products |
| `ciq_inspection` | CIQ quality inspection | CN food/cosmetics/chemicals exports |
| `nfe_linked` | NF-e electronic invoice linked | BR exports |

### 11.5 Status Machine Mapping (Two-Layer Model)

Each jurisdiction's raw statuses map to the common `platform_status`:

| Platform Status | US (CBP) | EU (UCC) | BR (Siscomex) | IN (ICEGATE) | CN (GACC) |
|----------------|----------|----------|---------------|-------------|-----------|
| `pending` | draft | — | registrada | filed | 已申报 |
| `under_review` | submitted, accepted | risk_analysis, document_check | yellow, red, grey channels | under_assessment, query_raised | yellow/red channel |
| `held` | cf28_pending, exam_scheduled | under_control | interrupted | on_hold | — |
| `cleared` | released | customs_released | desembaraçada | out_of_charge (OOC) | 已放行 |
| `released` | released | customs_debt_confirmed | entregue | — | 已结关 |
| `rejected` | rejected | rejected | rejected | rejected | rejected |

**Export Declaration Status Mapping (v5.2):**

Each jurisdiction's export-side raw statuses map to the same `platform_status` enum:

| Platform Status | US (AES) | EU (ECS) | BR (Portal Único) | IN (ICEGATE) | CN (Single Window) |
|----------------|----------|----------|-------------------|-------------|-------------------|
| `pending` | draft | draft | rascunho | draft | 草稿 |
| `under_review` | submitted, pending_itn | export_pending | registrada, em_analise | filed, under_examination | 已申报 |
| `held` | fatal_error, held_for_review | under_control, no_exit_results | retida | query_raised | 布控 |
| `cleared` | itn_received | release_for_export | desembaraçada | leo_issued | 已放行 |
| `released` | — | exit_confirmed (office of exit) | averbada (cargo departed) | — | 已结关 |
| `rejected` | rejected | rejected | rejeitada | rejected | rejected |

**Key differences from import:**
- US export: `itn_received` = the ITN (Internal Transaction Number) is the export clearance. No "release" equivalent — once ITN is received, goods can depart.
- EU export: Two-step — `release_for_export` (office of export clears) + `exit_confirmed` (office of exit confirms physical departure). The Export MRN must accompany goods.
- BR export: `averbada` = cargo departure confirmed by carrier. DU-E lifecycle tracks through to actual export.
- IN export: `leo_issued` = Let Export Order — the customs officer's authorization to export. This is the key clearance event.

---

## 12. Cross-Cutting Concerns (v5)

### 12.1 Two-Layer Status Model

All customs-facing entities use two status layers:
- **`jurisdiction_status: string`** — Raw status from the customs authority. Flexible, no constraints. Stores the authority's own status value (e.g., "已放行", "desembaraçada", "out_of_charge").
- **`platform_status: enum`** — Calculated/normalized: `pending | under_review | held | cleared | released | rejected`. Derived from `jurisdiction_status` via jurisdiction config mapping.

No code changes to add a jurisdiction — just configuration that maps jurisdiction statuses to platform statuses. Applied to HandlingUnit, Consolidation, EntryFiling (import), and ExportDeclaration (export) entities.

### 12.2 Soft Delete Standard

ALL entities get `entity_state: active | deleted` (string, default `active`). No physical deletes. Business rules for cancellations are documented per domain. Cancelled orders are `entity_state: active` with `status: cancelled`. Deleted entities are `entity_state: deleted` and excluded from queries by default.

### 12.3 Hot Path Solutions

**Movement cascade — lazy propagation:** When a consolidation moves, mark the consolidation as moved. Don't cascade immediately to every contained HU. Derive HU position from its parent chain on query. This is the pattern used by FedEx's internal tracking system — they don't propagate 2 million package location updates when a plane lands.

**Cache projector — split by domain:** Don't run a single monolithic projector subscribing to all `clearance.>` events. Split into per-domain projectors (or at minimum, per-functional-group). A bug in the financial event projector shouldn't affect the movement event projector.

**Classification — caching + batching:** For LLM classification calls, implement batching (group multiple product classifications into a single LLM call where possible) and aggressive caching (same product description + origin = same classification).

### 12.4 Schema Change Cascading

When an origin domain's event schema changes (e.g., HU adds a field), downstream projectors must pick it up. Strategy:
- Event schemas include a `metadata.version` field
- **Backward compatible changes** (new optional fields): Auto-include new fields in projections. Consumers ignore unknown fields.
- **Breaking changes** (field removal, type change): Require manual projector update. CI fitness function checks schema compatibility.
- Event schema versioning with backward compat is mandatory. Old consumers must not break when new fields appear.

### 12.5 Simulation Fixes (v5)

**Carrier acknowledgment loop:** Add `CarrierAcknowledgedHold` event. When CBP issues a hold, the carrier must acknowledge and confirm cargo is physically secured. Without this, the platform can't distinguish "carrier received hold instruction" from "carrier is actually holding cargo."

**DemurrageActor → periodic platform job:** D&D calculation should be a platform concern (hourly tick → compute D&D for all in-port cargo → emit `DemurrageAccruing` events), not a simulation actor calling a non-existent endpoint. Remove DemurrageActor from simulation.

**TimeService interface:** `TimeService.now()` returns `NOW()` in production, simulated clock time in simulation. This is a clean abstraction that maintains simulation-platform separation without leaking simulation concerns into domain services.

**PGA review behavior:** USDA/APHIS/FDA/CPSC/EPA are SEPARATE agencies with separate review processes, timelines, and dispositions. The PGABot should model per-agency behavior (separate from CustomsAuthorityBot), not lump all PGA reviews together.

---

## 13. Data Model (Per Domain)

### 13.1 PostgreSQL Schema Separation

```
PostgreSQL Cluster
├── schema: product_catalog
│   └── products
├── schema: trade_intelligence
│   ├── classifications (new)
│   ├── tariff_calculations (new)
│   ├── fta_determinations (new)
│   ├── htsus_chapters, htsus_headings
│   ├── section_301_lists, section_232_scope, ieepa_rates, adcvd_orders
│   ├── cross_rulings, tax_rates, fee_schedules, exchange_rates
│   ├── tax_regime_templates, fta_rules
├── schema: compliance_screening
│   ├── compliance_results (new)
│   ├── restricted_parties, pga_mappings
├── schema: order_management
│   ├── orders, order_line_items
├── schema: shipment_lifecycle
│   ├── shipments (shipper-facing convenience abstraction)
├── schema: cargo
│   ├── handling_units
├── schema: consolidation
│   ├── consolidations
├── schema: declaration_management
│   ├── entry_filings, export_declarations (v5.2), isf_filings, broker_assignments, brokers, broker_messages
├── schema: customs_adjudication
│   ├── adjudication_decisions (new)
├── schema: exception_management
│   ├── exceptions (new)
├── schema: financial_settlement
│   ├── financial_records (new), bonds (new), duty_drawbacks (new), reconciliations (new)
├── schema: regulatory_intelligence
│   ├── regulatory_signals
├── schema: document_management
│   ├── documents
├── schema: party_management            # v5
│   ├── parties, power_of_attorney, importers_of_record
└── schema: simulation
    ├── simulation_config, simulation_state
```

### 13.2 Data Decomposition: Shipments, Handling Units, and Consolidations

The `shipments` table is now a **simple convenience abstraction** -- it holds shipper-facing fields only. All physical, customs, and logistics data lives on `handling_units` and `consolidations`.

| Current Field on `shipments` | New Owner | New Location |
|------------------------------|-----------|--------------|
| `id`, `status`, `origin`, `destination`, `declared_value`, `company_name` | Shipment Lifecycle | `shipment_lifecycle.shipments` (shipper-facing fields) |
| `product_id`, `product` | Shipment Lifecycle | `shipment_lifecycle.shipments` (product reference) |
| `handling_unit_summary` (JSONB) | Shipment Lifecycle | `shipment_lifecycle.shipments` (non-authoritative HU projection) |
| `codes`, `financials`, `analysis` (JSONB) | Shipment Lifecycle | `shipment_lifecycle.shipments` (convenience copies, non-authoritative) |
| `transport_mode`, `carrier` | Consolidation | `consolidation.consolidations.border_crossing_mode`, `carrier` |
| `tracking_number`, `house_number` | Cargo & Handling Units | `cargo.handling_units` |
| `consolidation_id` | Cargo & Handling Units | `cargo.handling_units.consolidated_into` |
| `entry_number`, `entry_status` | Declaration Management | `declaration_management.entry_filings` (references HU/consolidation) |
| `cage_status` (JSONB) | Cargo & Handling Units | `cargo.handling_units.cage_status` |
| `hold_type` | Cargo & Handling Units | `cargo.handling_units.hold_type` |
| `events` (JSONB array) | Cargo & Handling Units / Consolidation | `cargo.handling_units.transit_history`, `consolidation.consolidations.events` |
| `routing`, `waypoints` (JSONB) | Consolidation | Routing intelligence on consolidation level |
| `references` (JSONB) | Split | HU-level refs on `handling_units`, consolidation-level refs on `consolidations` |
| `classification_result` (JSONB) | Trade Intelligence | `trade_intelligence.classifications` + events |
| `tariff_result` (JSONB) | Trade Intelligence | `trade_intelligence.tariff_calculations` + events |
| `compliance_result` (JSONB) | Compliance | `compliance_screening.compliance_results` + events |
| `documents` (JSONB) | Document Management | `document_management.documents` |
| `financials` (JSONB) | Financial Settlement | `financial_settlement.financial_records` |

### 13.3 Cross-Domain Data Access Rules

1. **Events** carry full state snapshots — subscribe to get another domain's data
2. **Redis cache** has denormalized views — UI queries Redis for the full picture
3. **API calls** for on-demand needs — never direct DB queries across domains

---

## 14. Deployment Topology

### 14.1 Development (Docker Compose)

```yaml
services:
  nats:        # NATS JetStream
  redis:       # State cache
  postgres:    # All schemas
  qdrant:      # Vector intelligence

  clearance-api:       # All domain services (modular monolith)
  cache-projector:     # Redis cache projector (separate process)
  simulation:          # Simulation platform (separate process, uses APIs)
  frontend:            # Vite React
  caddy:               # Reverse proxy
```

### 14.2 Migration to Microservices

| Phase | What | When |
|-------|------|------|
| 1 | Modular monolith — all domains in one process, separate schemas | Now |
| 2 | Extract Cargo & HUs + Consolidation + Declaration Management (high traffic) | When needed |
| 3 | Extract Trade Intelligence + Compliance (computation-heavy) | When needed |
| 4 | Full microservices — each domain independent | When needed |

Guided by need, not dogma.

### 14.3 Modular Monolith Boundary Enforcement

In Phase 1 (modular monolith), all 15 domains live in one process. Boundary enforcement prevents accidental coupling that would block future extraction.

#### Package Structure

```
clearance_platform/
├── domains/
│   ├── product_catalog/
│   │   ├── __init__.py          # Public API (only this is importable by other domains)
│   │   ├── service.py           # Domain service (internal)
│   │   ├── models.py            # SQLAlchemy models (internal)
│   │   ├── events.py            # Event schemas (public — shared via __init__)
│   │   ├── routes.py            # API routes (internal, registered by gateway)
│   │   └── intelligence/        # Intelligence engines (internal)
│   ├── trade_intelligence/
│   │   ├── __init__.py          # Public API
│   │   ├── ...
│   ├── cargo_handling_units/
│   │   ├── __init__.py          # Public API
│   │   ├── ...
│   ├── consolidation/
│   │   ├── __init__.py          # Public API
│   │   ├── ...
│   ├── party_management/
│   │   ├── __init__.py          # Public API
│   │   ├── ...
│   ├── ... (15 domains total)
├── shared/
│   ├── event_bus.py             # NATS client, event publishing
│   ├── cache.py                 # Redis client
│   ├── database.py              # Session factory, base model
│   └── llm.py                   # LLM gateway
├── cache_projector/             # Separate entry point
└── gateway/                     # API gateway, routing, auth
```

#### Import Rules

| From → To | Allowed? | Mechanism |
|-----------|----------|-----------|
| Domain A → Domain A internals | Yes | Normal Python imports |
| Domain A → Domain B `__init__` | **Events only** | `from domains.trade_intelligence import ClassificationProduced` |
| Domain A → Domain B internals | **NO** | Lint violation — blocked |
| Domain A → `shared/` | Yes | Infrastructure concerns |
| Domain A → Domain B's tables | **NO** | No cross-schema queries — ever |

**Enforced by:**
```python
# pyproject.toml — import-linter rules
[tool.import-linter]
root_packages = ["clearance_platform"]

[[tool.import-linter.contracts]]
name = "Domain isolation"
type = "independence"
modules = [
    "clearance_platform.domains.product_catalog",
    "clearance_platform.domains.trade_intelligence",
    "clearance_platform.domains.compliance_screening",
    "clearance_platform.domains.order_management",
    "clearance_platform.domains.shipment_lifecycle",
    "clearance_platform.domains.cargo_handling_units",
    "clearance_platform.domains.consolidation",
    "clearance_platform.domains.declaration_management",
    "clearance_platform.domains.customs_adjudication",
    "clearance_platform.domains.exception_management",
    "clearance_platform.domains.financial_settlement",
    "clearance_platform.domains.regulatory_intelligence",
    "clearance_platform.domains.document_management",
    "clearance_platform.domains.party_management",
]

[[tool.import-linter.contracts]]
name = "Event schemas are the only shared interface"
type = "layers"
layers = ["domains.*.events", "domains.*"]
```

#### Schema Isolation

Each domain owns its PostgreSQL schema. Cross-schema queries are forbidden:

```sql
-- ALLOWED: within own schema
SELECT * FROM trade_intelligence.classifications WHERE product_id = $1;

-- FORBIDDEN: cross-schema join
SELECT s.*, c.hs_code
FROM shipment_lifecycle.shipments s
JOIN trade_intelligence.classifications c ON c.product_id = s.product_id;
-- ^^^ This must go through events or Redis cache instead
```

**Enforced by:** Database roles per domain schema with `USAGE` grants restricted to own schema only. In development, a CI check scans SQL/ORM queries for cross-schema references.

#### Extraction Fitness Test

**The boundary test:** "Can I extract domain X to a separate service with its own database, without changing code in any other domain?"

| Check | What It Tests | Automated? |
|-------|--------------|------------|
| **No cross-domain imports** | Domain A doesn't import Domain B internals | Yes — `import-linter` in CI |
| **No cross-schema queries** | Domain A doesn't query Domain B's tables | Yes — SQL analysis in CI |
| **Events are the only interface** | All inter-domain data flows through NATS events | Yes — import-linter (events-only contract) |
| **No shared mutable state** | Domains don't share in-memory singletons | Code review |
| **Independent tests** | Domain A's tests pass with Domain B's service stubbed | Yes — domain test isolation in CI |

#### Architectural Fitness Functions (CI Pipeline)

```yaml
# Run on every PR
boundary-check:
  - import-linter                        # No cross-domain imports
  - schema-isolation-check               # No cross-schema SQL
  - event-contract-check                 # Event schemas backward-compatible
  - domain-test-isolation                # Each domain's tests pass independently
```

If any fitness function fails, the PR is blocked. This ensures the monolith remains extractable at all times.

---

## 15. Frontend Surfaces

The platform provides four actor-specific frontend surfaces, each with its own layout, navigation, color theme, and (for Broker and Platform) an AI assistant panel with streaming tool execution.

**Stack:** React 18 + TypeScript 5.3 + Vite 5 + Tailwind CSS 3.4 + Zustand (state) + Framer Motion (transitions)

**Totals:** 35 screens, 56 components, 2 AI assistants, 149 TypeScript files

### 15.1 Broker Surface

**Actor:** Licensed customs brokers
**Color theme:** Teal
**AI assistant:** BrokerAssistant (streaming chat, 11 MCP tools — 8 read, 3 write)
**Navigation:** Sectioned left sidebar (Workspace + Entry Work) with badge counts for pending approvals, CBP responses, and critical alerts
**Backend domain:** Declaration Management (47 endpoints at `/api/broker/*`)

The broker workspace covers the full entry filing lifecycle — from queue triage through classification review, document verification, compliance screening, and CBP submission.

#### Screens (8)

| Screen | Route | Purpose |
|--------|-------|---------|
| **Dashboard** | `/broker/dashboard` | Morning briefing with critical alerts, pipeline stats by filing status, and quick-action buttons (draft responses, verify classifications, resolve holds, screen entities). Shows unassigned shipments, GO deadlines, and compliance issues with financial impact. |
| **My Queue** | `/broker/queue` | Paginated work queue (50 items/page) with search, sorting (urgency/company/product/value/days since arrival), and status filters (draft, pending_broker_approval, submitted, cf28_pending, cf29_pending, exam_scheduled). Shows checklist progress, consolidation details, HU summary (held/exam counts). Inline Approve/Respond/Req Docs action buttons. |
| **Entry Detail** | `/broker/entries/:id` | Full entry filing view — the broker's primary workspace. Contains: hierarchical document checklist (entry/shipment/transport/HU levels with per-item upload, validation, override, and generation), ClassificationHub (AI-powered HS code review with step-by-step reasoning, competing headings, data source agreements, and accept/reclassify actions), compliance flags, fee breakdown (MPF/HMF/customs duties), AD/CVD orders panel, CBP rules engine audit trail (pass/fail/warn outcomes), bond setup, handling unit cards, and line items. Supports on-demand E1/E2/E3 analysis trigger. |
| **Authority Inquiries** | `/broker/cbp-responses` | CF-28 requests (request for information), CF-29 notices (notice of action), and exam orders with jurisdiction-aware labels, urgency countdown (days remaining), AI-drafted response guidance. Filterable by authority/jurisdiction/type. ResponseModal for drafting and submitting responses with recommended attachments. |
| **Communications** | `/broker/messages` | Inbound/outbound broker messages (email/phone/portal) with compose modal. Entry selector, recipient type (shipper/importer/carrier/CBP), purpose-based templates (missing_docs/classification_inquiry/value_clarification), and AI draft trigger that calls `draftCommunication`. Filters by direction. |
| **Export Declarations** | `/broker/export-declarations` | Export filing list with jurisdiction filter (US/EU/BR/IN/CN/MX/CA/UK) and status filter (draft/submitted/cleared/held/rejected/cancelled). Detail view shows filing data, timeline, export checklist. Amend/Submit/Delete actions. |
| **Regulatory Intel** | `/broker/regulatory` | Regulatory alert feed with jurisdiction filter, severity badges (CRITICAL/HIGH/MEDIUM/LOW), status indicators (CONFIRMED/PROPOSED/DISCUSSED), affected HS codes, source links, and tags. |
| **Party Management** | `/broker/party-management` | Party card grid with compliance score gauge, trust programs (C-TPAT, AEO), carrier affiliation. Side-out detail panel with jurisdiction identifiers, POA records, IOR records, entity state, created/updated dates. Search and type filters (importer/exporter/broker/carrier/consignee). |

#### Key Components

| Component | Purpose |
|-----------|---------|
| **ClassificationHub** | Unified classification panel — recommended HS code, confidence badge, collapsible step-by-step reasoning, data source agreements, competing headings, interactive chatbot feedback loop, Accept/Reclassify actions. Supports both entry and shipment classification. |
| **BrokerDocumentChecklist** | Hierarchical checklist (entry → shipment → transport → HU levels). Per-item upload, validation checks, override, request document via DM, generate document, bond type selector, completion percentage indicator, document viewer modal. |
| **ADCVDPanel** | Antidumping/countervailing duty orders — order type (AD/CVD), country affected, deposit amount, total deposit rollup. Green "clear" state when no orders apply. |
| **AuditTrailPanel** | CBP rules engine audit trail — collapsible filing event log with severity (hard stop/warning/info), rule name, detail, reference link, pass/fail/warn/skip outcome. |
| **ClassificationSection** | Simpler classification display with confidence tier, reasoning breakdown, alternative codes, follow-up questions form (multiple choice/yes-no/text), missing information list, gap summary. |
| **BrokerAssistant** | Right-panel streaming chat with context-aware suggestions — "Draft response to CF-28 for Entry X", "Check what's needed for this entry", "Review critical alerts", "Prioritize unassigned shipments". Tool event tracking, markdown rendering, agent activity indicator. |

#### Filing Readiness Gates

The entry detail tracks 6 readiness gates persisted in `entry_filing.summary_data`:

| Gate | Description |
|------|-------------|
| `classification` | HS code assigned with sufficient confidence |
| `tariff` | Duty/fee calculation complete |
| `compliance` | DPS/PGA/UFLPA screening passed or reviewed |
| `documents` | Required documents uploaded and validated |
| `bond` | Bond type selected and amount sufficient |
| `isf` | ISF filed (ocean) or N/A (air/ground) |

Each gate has `ready: bool`, `updated_at`, and `details`. Overall readiness score: 0.0–1.0.

### 15.2 Platform Surface

**Actor:** Operations teams, compliance analysts, supervisors
**Color theme:** Purple
**AI assistant:** OperatorAssistant (streaming chat)
**Navigation:** Left sidebar with 3 sections (Command Center, Intelligence, Tools) + simulation controls

#### Screens (15)

| Screen | Route | Purpose |
|--------|-------|---------|
| Control Tower | `/platform/dashboard` | KPI monitoring, exception metrics, ops activity feed |
| Active Shipments | `/platform/shipments` | All in-transit shipments with live status, search/filter |
| Shipment Detail | `/platform/shipments/:id` | Consolidation tree, cargo operations, cage status panel |
| Entry Detail | `/platform/entries/:id` | Tariff/HTS/duty calculations, compliance check results |
| Orders & Entries | `/platform/orders` | Order book and customs entry listings |
| Order Detail | `/platform/orders/:id` | Order processing and fulfillment tracking |
| Regulatory Intel | `/platform/regulatory` | Real-time regulatory signal feed with filtering |
| Signal Review | `/platform/regulatory/:id/review` | In-depth signal analysis, impact assessment, review workflow |
| Exception Resolution | `/platform/exceptions` | Exception queue with AI-assisted triage and resolution paths |
| Entity Screening | `/platform/screening` | OFAC/restricted party lookups against 14 federal lists |
| Product Analysis | `/platform/analysis` | Tariff classification and duty scenario modeling |
| Trade Lane Comparison | `/platform/trade-lanes` | Cross-border route analysis and landed-cost optimization |
| Compliance Dashboard | `/platform/compliance` | Compliance audit trails, CBP rule engine status, hold rates |
| Education Center | `/platform/education` | Training materials, regulatory guidance documents |
| Simulation Control | `/platform/simulation` | Live simulation controls, scenario orchestration, clock management |

#### Key Components

ConsolidationTree (multi-modal consolidation visualization), CargoOperations (HU details), CageStatusPanel (CAGE registration), SignalBoard (regulatory signal filtering), ExceptionQueue/Panel/Actions (exception triage), CompliancePanel (audit checklist), TariffStack/DutyStackBar (duty visualization), CorridorMatrix (trade lane heatmap), ScenarioComparison (side-by-side duty comparison), ScreeningResult (entity screening hit results), PGABreakdownPanel (agency breakdown).

### 15.3 Shipper Surface

**Actor:** Exporters, freight forwarders, importers
**Color theme:** Amber
**Navigation:** Left sidebar with 4 sections (Orders, Shipments, Products, Help)

#### Screens (9)

| Screen | Route | Purpose |
|--------|-------|---------|
| Orders | `/shipper/orders` | Shipment order list with status cards |
| Create Order | `/shipper/orders/new` | New order creation wizard with product selection |
| Order Detail | `/shipper/orders/:id` | Order tracking, compliance readiness, documents |
| Shipments | `/shipper/shipments` | Historical shipment records |
| Shipment Detail | `/shipper/shipments/:id` | Timeline, financials (landed cost breakdown), documents, waypoints |
| Products | `/shipper/products` | Product catalog listing |
| Add Product | `/shipper/products/new` | Register new product for export with HS pre-classification |
| Product Detail | `/shipper/products/:id` | Product tariff classification history, trade attributes |
| Resolution Center | `/shipper/resolve` | Help desk and issue resolution |

#### Key Components

ShipmentTimeline (lifecycle event timeline), ShipmentWaypoints (map-style waypoint visualization), ShipmentFinancials/DutyBreakdownCard (landed cost breakdown), AnalysisResultsPanel/AnalysisOverlay (compliance analysis), DocumentRequirements/AttachedDocuments (document checklist), ReadinessCard (pre-shipment readiness), PlainLanguageSummary (human-readable compliance explanation).

### 15.4 Buyer Surface

**Actor:** End consumers
**Color theme:** Green
**Navigation:** Header with logo (links to shop) + cart button. No sidebar.

#### Screens (3)

| Screen | Route | Purpose |
|--------|-------|---------|
| Shop Front | `/buyer/shop` | Product catalog browsing with search |
| Product Page | `/buyer/product/:id` | Product detail with landed cost estimate and duty breakdown |
| Checkout | `/buyer/checkout` | Cart review with transparent all-in pricing |

#### Key Components

PriceDisplay (price with tariff/landed cost breakdown), DutiesBadge (tariff impact badge), LandedCostBreakdown/LandedCostModal (itemized cost tables), BreakdownToggle (show/hide cost details).

### 15.5 State Management

| Store | Surface | Manages |
|-------|---------|---------|
| `brokerStore` | Broker | Queue data, selected entry, dashboard stats, alerts, AI panel state |
| `operatorStore` | Platform | AI assistant panel state, selected analysis |
| `sessionStore` | All | Current surface, user context, active session |
| `cartStore` | Buyer | Shopping cart items, totals |
| `pipelineStore` | Dev tools | Development pipeline state |

### 15.6 API Client

Centralized API client at `src/api/client.ts` handles all backend communication:
- Automatic `snake_case` → `camelCase` response transforms
- SSE streaming for chat endpoints and classification
- Error handling with retry logic
- Base URL from `VITE_API_URL` environment variable

---

## Appendix A: Event Payload Schema

All events follow the state-transfer pattern with full state snapshots:

```json
{
  "event_id": "uuid",
  "event_type": "ClassificationProduced",
  "subject": "clearance.trade-intel.classification.produced",
  "timestamp": "2026-02-09T12:00:00Z",
  "domain": "trade_intelligence",
  "entity_type": "classification",
  "entity_id": "uuid",
  "state": {
    // Full state snapshot — self-sufficient, no need to query producer
    "id": "uuid",
    "product_id": "uuid",
    "hs_code": "8471.30",
    "confidence": "HIGH",
    "reasoning_steps": ["..."],
    "classified_at": "2026-02-09T12:00:00Z"
  },
  "metadata": {
    "correlation_id": "uuid",      // Trace across domains
    "causation_id": "uuid",        // What event caused this one
    "producer": "trade-intelligence-svc",
    "version": 1
  }
}
```

## Appendix B: Domain Interaction Matrix

How events flow between domains (Producer → Consumer):

| Producer / Consumer | Prod Cat | Trade Intel | Compliance | Order | Shipment | HU | Consolidation | Declaration | Adjudication | Exception | Financial | Regulatory | Document | Party |
|---------------------|----------|-------------|------------|-------|----------|-----|---------------|-------------|--------------|-----------|-----------|------------|----------|-------|
| **Product Catalog** | -- | Y | | | | | | | | | | | | |
| **Trade Intelligence** | | -- | Y | Y | Y | | | Y | | | Y | | Y | |
| **Compliance** | | | -- | Y | Y | | | Y | | Y | | | | |
| **Order Management** | | | | -- | Y | | | | | | | | | |
| **Shipment Lifecycle** | | Y | Y | | -- | | | Y | | | | | Y | |
| **Cargo & HUs** | | | | | Y | -- | | Y | | | Y | | | |
| **Consolidation** | | | | | | Y | -- | | | | Y | | | |
| **Declaration Mgmt** | | | | | | | | -- | Y | | Y | | | Y |
| **Customs Adjudication** | | | | | | Y | Y | Y | -- | Y | Y | | | |
| **Exception Mgmt** | | | | | | Y | | | | -- | Y | | | |
| **Financial** | | | | | | | | Y | | | -- | | | |
| **Regulatory Intel** | | Y | Y | | | | | | | | | -- | | |
| **Document Mgmt** | | | | | | | | Y | | | | | -- | |
| **Party Mgmt** | | | | | | | | Y | | | | | | -- |

**Reading the matrix:** Row = produces events. Column = consumes events. ✓ = consumer subscribes to producer's events.

---

## Appendix C: Deferred Items & Known Gaps (v5.1)

Items identified during jurisdiction validation review (`docs/review-final-jurisdiction-validation.md`) that are documented for future implementation. These do not block US production or multi-jurisdiction development, but must be addressed before non-US production deployment.

### C.1 EU Member-State Sub-Jurisdictions

EU member states operate independent customs IT systems (Germany: ATLAS, France: DELTA-G/DELTA-C, Netherlands: AGS, Italy: AIDA). The current architecture treats the EU as a single jurisdiction adapter. Future work: sub-jurisdiction routing to select the correct national system based on port of entry. The `EntryFiling.jurisdiction` value of `"EU"` must be supplemented with a sub-jurisdiction identifier (e.g., `jurisdiction_config.member_state: "DE"`) to route filings to the correct national declaration system and apply the correct member-state VAT rate. Estimated effort: medium -- extends the adapter pattern with a country-to-system mapping layer.

### C.2 Mexico Mandatory Pre-Validation (VUCEM)

Mexico's VUCEM system requires mandatory pre-validation (validacion previa) before pedimento filing. The customs broker submits data to an authorized pre-validator (Prevalidadora) for automated validation; errors must be corrected before the actual filing can proceed. No other target jurisdiction has this requirement. Future work: add a pre-validation step to the filing workflow for Mexico (and potentially other jurisdictions with similar requirements). The MX adapter's `filing_workflow` state machine should include `pre_validation_pending -> pre_validated` states before `submitted`. The `jurisdiction_status` field can track this. Estimated effort: low -- additional `filing_status` states in the declaration lifecycle.

### C.3 Export Declarations — ADDRESSED (v5.2)

~~Previously deferred.~~ Export declarations are now fully modeled as first-class entities in the Declaration Management domain (section 3.8). The `ExportDeclaration` entity, export filing workflows for all 7 target jurisdictions, export-specific NATS events, export status machine mappings, export checklist items, and export API surface are all defined. The Jurisdiction Adapter interface (section 11) has been extended with `export_filing_workflow`, `export_status_machine`, `export_document_requirements`, `export_checklist_generator`, `export_license_required`, and `export_filing_required` methods. See sections 3.8, 4.1, 11.1, and 11.4 for complete details.

---

## Appendix D: Implementation Status (Updated 2026-02-27)

### Complete

- **15 domain packages** with routes, models, services, consumers, and events (`clearance_platform/domains/`) — all fully implemented
- **153 API endpoints** across 15 domains + gateway routes
- **65 ORM models** across 15 PostgreSQL schemas
- **67 event types** with NATS JetStream consumers
- **Route migration complete**: All API traffic served from domain routers via gateway registry. Legacy `app/api/routes/` retained only for 5 delegated broker LLM functions and simulation shims
- **DomainEventBus** with NATS JetStream pub/sub (`clearance_platform/shared/event_bus.py`)
- **8 analytical engines** (E0-E7) with standardized EngineOutput envelope
- **CSL ingestion pipeline**: Fetches Consolidated Screening List from trade.gov, loads 14 US federal restricted party lists into `compliance.restricted_parties` on 6-hour schedule
- **DPS expansion**: Denied party screening covers 14 US federal + 2 international lists with 3-stage fuzzy matching
- **Multi-modal clearance**: Routing engine, consolidation (MAWB/MBL), intermediate transit events, cage management with GO deadlines
- **Export declarations**: First-class entities with jurisdiction-aware filing (US/EU/BR/IN/CN/MX/CA/UK)
- **4 frontend surfaces**: Broker (8 screens), Platform (15 screens), Shipper (9 screens), Buyer (3 screens) — 35 screens total with 56 components and 2 AI assistants
- **2,200+ passing tests** (103 files) + ~240 Playwright E2E tests (10 spec files)

### Known Architecture Debt

- **Shipment model split**: Live code uses `public.shipments` (legacy ORM); domain model points to `shipment.shipments` (table does not exist). Migration required to unify.
- **Legacy broker LLM functions**: 5 functions in `app/api/routes/broker.py` delegated via `shared/legacy/broker.py` (get_briefing, draft_cf28_response, draft_communication, get_entry_insights, draft_cf29_protest). Target: migrate to declaration_management service.
- **Alembic migration gap**: Domain Base has never been migrated via Alembic; only legacy Base is tracked. All domain tables created via `init.sql`.
- **E4 FTA scope**: Only USMCA implemented; no EU FTAs, RCEP, CPTPP, CAFTA-DR, KORUS.
- **E5 ruling corpus**: 10 hardcoded CROSS rulings; production needs expanded Qdrant corpus with ruling ingestion pipeline.

---

*This document is the definitive architecture reference for the Clearance Platform. All previous architecture documents are superseded.*
