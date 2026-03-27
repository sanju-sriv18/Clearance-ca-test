# Clearance Platform — Architecture Document

**Date:** 2026-02-27
**Version:** 1.1
**Branch:** `main`

---

## Table of Contents

1. [Conceptual Architecture](#1-conceptual-architecture)
2. [Logical Architecture](#2-logical-architecture)
3. [Implementation Architecture](#3-implementation-architecture)

---

## 1. Conceptual Architecture

### 1.1 What Is the Clearance Platform?

The Clearance Platform is a **customs clearance intelligence system** that automates the end-to-end flow of international trade: from purchase order creation through shipment, customs declaration, authority adjudication, and financial settlement. It serves four actor types — **shippers**, **customs brokers**, **platform operators**, and **buyers** — through dedicated user interfaces, while AI agents and a simulation engine exercise the same APIs that humans use.

### 1.2 Business Domain

International customs clearance involves coordinating physical cargo movement with regulatory compliance across multiple jurisdictions. A single shipment from Shanghai to New York may require:

- Export declaration in China (Single Window)
- Transit clearance in the EU (ICS2/ECS at CDG)
- Import entry filing in the US (CBP/ACE 3461 + 7501)
- Separate document requirements, duty calculations, and compliance screening at each border crossing

The platform models this reality as **15 functional business domains**, each representing a distinct operational concern in the clearance lifecycle.

### 1.3 Core Concepts

```
Purchase Order ──→ Shipment ──→ Handling Units ──→ Entry Filing ──→ Clearance
     │                │              │                   │              │
  Products        Routing      Consolidation         Broker         Authority
  Line Items      Legs         MAWB/MBL             Checklist      Decision
  Values          Events       Cage/Hold            Documents      Release/Hold
```

| Concept | Description |
|---------|-------------|
| **Order** | A purchase order with line items, each referencing a product from the catalog |
| **Product** | A trade-relevant item with description, HS codes, origin, materials, value |
| **Shipment** | A shipper-facing abstraction grouping goods from origin to destination |
| **Handling Unit** | Physical cargo (container, pallet, crate) — the broker's primary operational unit |
| **Consolidation** | A master air waybill (MAWB) or master bill of lading (MBL) grouping multiple shipments |
| **Entry Filing** | The customs declaration submitted to an authority (CBP entry, EU ICS2, etc.) |
| **Adjudication** | The authority's processing of an entry — review, exam, hold, release |
| **Document** | Trade documents (commercial invoice, packing list, bill of lading, certificates) |

### 1.4 Architectural Principles

| # | Principle | Meaning |
|---|-----------|---------|
| 1 | **Domain-first design** | Services organized by business function, not by actor, screen, or technical concern |
| 2 | **State-transfer events** | Domain events carry full state snapshots — consumers never need to query the producer |
| 3 | **Share-on-principle** | Events make data available by default; new consumers subscribe to existing event streams |
| 4 | **Actors are automation, not architecture** | Simulation bots and AI agents use the same APIs as humans — they sit on top, never inside |
| 5 | **Intelligence as domains** | Classification, compliance, and tariff calculation are proper domains with persistent state |
| 6 | **CQRS everywhere** | Writes go through domain services (business logic → state change → event); reads come from Redis cache |
| 7 | **Proactive clearance** | Intelligence services produce results as soon as data prerequisites are met, enabling pre-filing during transit |
| 8 | **APIs as MCP tools** | Every API endpoint doubles as an MCP tool for AI agents |

### 1.5 Separation of Concerns

```
┌──────────────────────────────────────────────────────────────┐
│                    SIMULATION LAYER                            │
│  Shipper Bot · Carrier Bot · Customs Authority Bot · Buyer    │
│  Simulation Coordinator (clock, lifecycle)                     │
│  Uses platform APIs (REST) · Subscribes to clearance.* events │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                   CLEARANCE PLATFORM                          │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                    API GATEWAY                           │ │
│  │  Routing · Auth · Rate Limiting · MCP · SSE Proxy       │ │
│  └─────────────────────────┬───────────────────────────────┘ │
│                            │                                  │
│  ┌─────────────── FUNCTIONAL DOMAINS ──────────────────────┐ │
│  │  15 bounded contexts, each owning:                       │ │
│  │  State · Logic · Events · API · Consumers                │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌─────────────── EVENT BACKBONE ──────────────────────────┐ │
│  │  NATS JetStream — 16 streams, subject-based routing      │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │ PostgreSQL│  │  Redis   │  │   NATS   │  │  Qdrant  │    │
│  │ Per-domain│  │  State   │  │ JetStream│  │  Vector  │    │
│  │ schemas  │  │  Cache   │  │  Events  │  │  Search  │    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │
└──────────────────────────────────────────────────────────────┘
```

The **Simulation Platform** is a consumer of the Clearance Platform, not a component. It exercises the platform through the same REST APIs that humans use, and subscribes to `clearance.*` events as a read-only observer. The Clearance Platform has zero knowledge of the simulation.

---

## 2. Logical Architecture

### 2.1 Functional Domains (15)

The platform is decomposed into 15 bounded contexts organized by business function:

**Intelligence Domains** — produce analytical results proactively via events:

| # | Domain | Purpose | Key Entity |
|---|--------|---------|------------|
| 1 | Product Catalog | Master catalog with trade-relevant attributes | `Product` |
| 2 | Trade Intelligence | HS classification, tariff calculation, FTA eligibility, landed cost | `Classification`, `TariffCalculation` |
| 3 | Compliance & Screening | Denied party screening, UFLPA, PGA, risk scoring | `ComplianceResult` |

**Operational Domains** — manage the physical and administrative lifecycle:

| # | Domain | Purpose | Key Entity |
|---|--------|---------|------------|
| 4 | Order Management | Purchase orders from creation to shipment linking | `Order`, `OrderLineItem` |
| 5 | Shipment Lifecycle | Shipper-facing shipment abstraction with routing legs | `Shipment` |
| 6 | Cargo & Handling Units | Physical cargo: containers, pallets, crates; cage/hold management | `HandlingUnit` |
| 7 | Consolidation | MAWB/MBL grouping, multi-modal transport, nesting | `Consolidation` |

**Clearance Domains** — manage the regulatory filing and adjudication process:

| # | Domain | Purpose | Key Entity |
|---|--------|---------|------------|
| 8 | Declaration Management | Broker's workflow: entries, checklists, assignments, export declarations | `EntryFiling`, `ExportDeclaration` |
| 9 | Customs Adjudication | Authority processing: review, exam, hold, release | `AdjudicationDecision` |

**Supporting Domains** — provide cross-cutting capabilities:

| # | Domain | Purpose | Key Entity |
|---|--------|---------|------------|
| 10 | Exception Management | Cross-domain escalation, SLA monitoring | `ClearanceException` |
| 11 | Financial Settlement | Duty/fee calculation, bonds, drawback, reconciliation | `FinancialRecord` |
| 12 | Regulatory Intelligence | Regulatory change monitoring, scenario modeling | `RegulatorySignal` |
| 13 | Document Management | Document generation, validation, storage, requirements | `Document` |
| 14 | Party Management | Parties, power of attorney, importer of record, trust programs | `Party` |
| 15 | Supply Chain Disruptions | Port closures, weather, strikes, system outages | `Disruption` |

### 2.2 Domain Interaction Model

Domains communicate exclusively through **NATS JetStream events**. There are no synchronous inter-domain API calls.

```
Product Created ──→ Trade Intelligence (auto-classify)
                 ──→ Compliance (auto-screen)

Shipment Created ──→ Cargo HU (create handling units)
                 ──→ Trade Intelligence (calculate tariffs)
                 ──→ Document Management (determine requirements)
                 ──→ Declaration Management (create entry filing)

Adjudication Decision ──→ Cargo HU (update hold/release)
                       ──→ Declaration Management (update entry status)

Consolidation Movement ──→ Cargo HU (update location)
                        ──→ Shipment Lifecycle (update transit status)
```

### 2.3 Event Architecture

Events follow the **state-transfer pattern**: each event carries the full state snapshot of the entity, so consumers never need to query back to the producer.

**Event structure:**

```
BaseEvent {
  metadata: {
    event_id: UUID          — Unique event identifier
    event_type: string      — e.g. "shipment.created"
    subject: string         — NATS subject: clearance.{domain}.{event-type}
    domain: string          — Source domain name
    timestamp: datetime     — UTC timestamp
    version: int            — Schema version
    correlation_id: UUID?   — Request-level tracing
    causation_id: UUID?     — What event caused this one
  }
  ...payload fields        — Full entity state snapshot
}
```

**16 JetStream streams** (15 domain + 1 DLQ) with retention policies ranging from 30 days (product) to 365 days (financial/declaration/adjudication):

| Stream | Subjects | Retention |
|--------|----------|-----------|
| CLEARANCE_PRODUCT | `clearance.product.>` | 30 days |
| CLEARANCE_TRADE_INTEL | `clearance.trade-intel.>` | 90 days |
| CLEARANCE_COMPLIANCE | `clearance.compliance.>` | 90 days |
| CLEARANCE_ORDER | `clearance.order.>` | 90 days |
| CLEARANCE_SHIPMENT | `clearance.shipment.>` | 180 days |
| CLEARANCE_CARGO | `clearance.cargo.>` | 180 days |
| CLEARANCE_CONSOLIDATION | `clearance.consolidation.>` | 180 days |
| CLEARANCE_DECLARATION | `clearance.declaration.>` | 365 days |
| CLEARANCE_ADJUDICATION | `clearance.adjudication.>` | 365 days |
| CLEARANCE_EXCEPTION | `clearance.exception.>` | 365 days |
| CLEARANCE_FINANCIAL | `clearance.financial.>` | 365 days |
| CLEARANCE_REGULATORY | `clearance.regulatory.>` | 90 days |
| CLEARANCE_DOCUMENT | `clearance.document.>` | 365 days |
| CLEARANCE_PARTY | `clearance.party.>` | 365 days |
| CLEARANCE_DISRUPTION | `clearance.disruption.>` | 365 days |
| CLEARANCE_DLQ | `clearance.dlq.>` | 90 days |

Consumer configuration: 30s ACK wait, max 3 delivery attempts before DLQ routing.

### 2.4 CQRS Read/Write Split

```
                    ┌─────────────┐
  Write ──────────→ │   Domain    │ ────→ NATS Event
  (POST/PUT/DELETE) │  Service    │
                    └─────────────┘
                                          │
                                          ▼
                    ┌─────────────┐   ┌──────────┐
  Read ───────────→ │   Redis     │ ← │  Cache   │
  (GET)             │   Cache     │   │ Projector│
                    └─────────────┘   └──────────┘
```

- **Writes** flow through domain services, which apply business logic, persist to PostgreSQL, and emit events.
- The **Cache Projector** subscribes to all `clearance.>` events and maintains pre-computed read models in Redis.
- **Reads** are served from Redis for fast, consistent responses.
- A 5-minute reconciliation task compares Redis state vs PostgreSQL, triggering cache refresh on discrepancy.

### 2.5 Intelligence Engine Pipeline

Eight intelligence engines run as event-driven subscribers, triggered automatically when prerequisites are met:

| Engine | Trigger Event | Output Event | Purpose |
|--------|--------------|--------------|---------|
| E0: Pre-Clearance | `shipment.booked` | `preclearance.completed` | Agentic screening (entity, HS validation, value, docs, origin risk) |
| E1: Classification | `product.created` | `trade-intel.classification.produced` | Three-step HS code classification (enrichment → data gathering → GRI-based LLM) |
| E2: Tariff | `shipment.created` | `trade-intel.tariff.produced` | Duty/fee calculation per jurisdiction (6 regimes: US, EU, CN, BR, IN, UK) |
| E3: Compliance | `product.created` | `compliance.screening.produced` | PGA (20+ agencies) + DPS (14 US federal lists via CSL) + UFLPA screening |
| E4: FTA | `shipment.created` | `trade-intel.fta.produced` | USMCA rules-of-origin + RVC calculation |
| E5: Exception | `exception.created` | `exception.resolution.produced` | CROSS rulings search + LLM response drafting |
| E6: Regulatory | `regulatory.signal.created` | `regulatory.analysis.produced` | Signal feed, scenario modeling, portfolio impact analysis |
| E7: Documents | `entry.created` | `document.requirements.produced` | Document requirements lookup + generation |

### 2.6 Jurisdiction Adapter Pattern

The platform supports 7 jurisdictions through an abstract adapter interface. Each jurisdiction adapter implements 12 methods covering import/export workflows, duty calculation, status machines, document requirements, and checklist generation:

| Jurisdiction | Import System | Export System | Duty Model |
|-------------|--------------|--------------|------------|
| United States | CBP/ACE (3461 + 7501) | AES (EEI + ITN) | Additive (duty + MPF + HMF + Section 301/232/IEEPA + AD/CVD) |
| European Union | ICS2/UCC | ECS | Duty + AD/CVD + member-state VAT |
| China | Single Window | Single Window | Duty + VAT (13%) + consumption tax |
| Brazil | Portal Unico (DUIMP) | DU-E | Cascading (II → IPI → PIS/COFINS → ICMS → AFRMM) |
| India | ICEGATE (Bill of Entry) | Shipping Bill | BCD → SWS → IGST |
| Mexico | VUCEM/SAT | VUCEM/SAT | Duty + IVA + DTA |
| Canada | CARM/CBSA | CAED | Duty + GST/HST/PST |

### 2.7 Multi-Modal Transport Model

Every international shipment is multi-modal. A "FedEx Air" shipment includes ground legs (warehouse pickup, airport transfer, last-mile delivery). The platform models this through:

- **Transport mode** = the border-crossing mode that determines the document hierarchy (air, ocean, rail, road)
- **Consolidation** = MAWB/MBL grouping with full transport details (flight, vessel, container)
- **Routing engine** = multi-leg route planning with transshipment hubs as regulatory clearance gates
- **Intermediate events** = transit events at each jurisdiction boundary

### 2.8 Frontend Surface Architecture

Four actor-specific surfaces provide tailored views of the platform. Each has a dedicated layout, navigation, color theme, and (for Broker and Platform) an AI assistant panel.

| Surface | Actor | Screens | Components | AI Assistant |
|---------|-------|--------:|----------:|:------------|
| **Broker** | Licensed customs broker | 8 | 7 | Yes (BrokerAssistant) |
| **Platform** | Operations & compliance | 15 | 27 | Yes (OperatorAssistant) |
| **Shipper** | Exporter/importer | 9 | 17 | No |
| **Buyer** | End consumer | 3 | 5 | No |

#### Broker Surface (`/broker`)

The broker workspace is the entry filing and compliance review environment for licensed customs brokers. It covers the full lifecycle from queue triage through CBP submission.

| Screen | Route | Purpose |
|--------|-------|---------|
| Dashboard | `/broker/dashboard` | Morning briefing with critical alerts, queue stats, and quick-action buttons (draft responses, verify classifications, resolve holds, screen entities) |
| My Queue | `/broker/queue` | Paginated work queue (50 items/page) with search, sorting (urgency/company/product/value/days), status filters (draft → released), inline Approve/Respond/Req Docs actions |
| Entry Detail | `/broker/entries/:id` | Full entry filing view: hierarchical document checklist (entry/shipment/transport/HU levels), classification hub with AI chat, compliance flags, fee breakdown (MPF/HMF/duties), AD/CVD orders, audit trail, bond setup, handling unit cards |
| Authority Inquiries | `/broker/cbp-responses` | CF-28 requests, CF-29 notices, and exam orders with jurisdiction-aware labels, urgency countdown, AI-drafted responses |
| Communications | `/broker/messages` | Inbound/outbound messages (email/phone/portal) with compose modal, AI draft trigger, entry selector, purpose-based templates |
| Export Declarations | `/broker/export-declarations` | Export filing list with jurisdiction filter (US/EU/BR/IN/CN/MX/CA/UK), status workflow (draft → submitted → cleared), amend/submit/delete actions |
| Regulatory Intel | `/broker/regulatory` | Regulatory alert feed with jurisdiction/severity filtering, affected HS codes, source links |
| Party Management | `/broker/party-management` | Party cards with compliance score gauge, trust programs (C-TPAT), POA/IOR records, jurisdiction identifiers |

**Key broker components:** ClassificationHub (AI-powered HS code review with step-by-step reasoning, competing headings, and accept/reclassify actions), BrokerDocumentChecklist (hierarchical checklist with per-item upload, validation, override, and document generation), ADCVDPanel (antidumping/countervailing duty orders), AuditTrailPanel (CBP rules engine results with pass/fail/warn outcomes).

**Broker AI Assistant:** Streaming chat panel with 11 tools (8 read, 3 write) providing context-aware suggestions — "Draft response to CF-28 for Entry X", "Check what's needed for this entry", "Review critical alerts".

**Backend:** 47 API endpoints in `declaration_management/routes.py` at `/api/broker/*`, including entry filing CRUD, checklist management, AI drafting (CF-28, CF-29, communications), classification chat streaming, export declaration lifecycle, and audit trail.

#### Platform Surface (`/platform`)

| Screen | Route | Purpose |
|--------|-------|---------|
| Control Tower | `/platform/dashboard` | KPI monitoring, exception metrics, activity feed |
| Active Shipments | `/platform/shipments` | All in-transit shipments with live status |
| Shipment Detail | `/platform/shipments/:id` | Consolidation tree, cargo operations, cage status |
| Entry Detail | `/platform/entries/:id` | Tariff/HTS/duty calculations, compliance checks |
| Orders & Entries | `/platform/orders` | Order book and customs entry listings |
| Regulatory Intel | `/platform/regulatory` | Real-time regulatory signal feed |
| Signal Review | `/platform/regulatory/:id/review` | In-depth signal analysis and impact assessment |
| Exception Resolution | `/platform/exceptions` | Exception queue with AI-assisted triage |
| Entity Screening | `/platform/screening` | OFAC/restricted party lookups (14 federal lists) |
| Product Analysis | `/platform/analysis` | Tariff classification and duty scenario modeling |
| Trade Lane Comparison | `/platform/trade-lanes` | Cross-border route analysis and optimization |
| Compliance Dashboard | `/platform/compliance` | Compliance audit trails, CBP rule engine status |
| Education Center | `/platform/education` | Training materials, regulatory guidance |
| Simulation Control | `/platform/simulation` | Live simulation controls and scenario orchestration |

#### Shipper Surface (`/shipper`)

| Screen | Route | Purpose |
|--------|-------|---------|
| Orders | `/shipper/orders` | Shipment order list |
| Create Order | `/shipper/orders/new` | New order creation wizard |
| Order Detail | `/shipper/orders/:id` | Order tracking, compliance readiness |
| Shipments | `/shipper/shipments` | Historical shipment records |
| Shipment Detail | `/shipper/shipments/:id` | Shipment view with timeline, financials, documents |
| Products | `/shipper/products` | Product catalog listing |
| Add Product | `/shipper/products/new` | Register new product for export |
| Product Detail | `/shipper/products/:id` | Product tariff classification and history |
| Resolution Center | `/shipper/resolve` | Help desk and issue resolution |

#### Buyer Surface (`/buyer`)

| Screen | Route | Purpose |
|--------|-------|---------|
| Shop Front | `/buyer/shop` | Product catalog browsing |
| Product Page | `/buyer/product/:id` | Product with landed cost estimate |
| Checkout | `/buyer/checkout` | Cart review with transparent duty breakdown |

---

## 3. Implementation Architecture

### 3.1 Repository Structure

```
clearance_vibe/
├── clearance-engine/                    # Main application
│   ├── backend/                         # Python backend
│   │   ├── app/                         # Legacy monolith (deprecated, retained for reference)
│   │   │   └── api/                     # Old routes/schemas (NOT used at runtime)
│   │   ├── clearance_platform/          # LIVE microservices codebase
│   │   │   ├── domains/                 # 15 bounded contexts
│   │   │   │   ├── product_catalog/
│   │   │   │   ├── trade_intelligence/
│   │   │   │   ├── compliance/
│   │   │   │   ├── order_management/
│   │   │   │   ├── shipment_lifecycle/
│   │   │   │   ├── cargo_handling_units/
│   │   │   │   ├── consolidation/
│   │   │   │   ├── declaration_management/
│   │   │   │   ├── customs_adjudication/
│   │   │   │   ├── exception_management/
│   │   │   │   ├── financial_settlement/
│   │   │   │   ├── regulatory_intelligence/
│   │   │   │   ├── document_management/
│   │   │   │   ├── party_management/
│   │   │   │   └── supply_chain_disruptions/
│   │   │   ├── gateway/                 # API gateway: router registry + cross-cutting routes
│   │   │   ├── shared/                  # Shared infrastructure library
│   │   │   │   ├── agent/               # AI agent prompts and tools
│   │   │   │   ├── engines/             # Intelligence engines (E0-E7)
│   │   │   │   ├── events/              # BaseEvent, EventMetadata
│   │   │   │   ├── jurisdiction/        # 7 jurisdiction adapters
│   │   │   │   ├── schemas/             # Shared Pydantic response schemas
│   │   │   │   ├── services/            # Shared services (analysis, dashboard, etc.)
│   │   │   │   ├── simulation/          # Simulation coordinator, state machine
│   │   │   │   ├── database.py          # Domain-to-schema mapping
│   │   │   │   ├── domain_models.py     # Cross-domain model re-exports
│   │   │   │   ├── event_bus.py         # NATS JetStream event bus
│   │   │   │   ├── streams.py           # 16 stream definitions
│   │   │   │   ├── idempotency.py       # Event deduplication
│   │   │   │   ├── circuit_breaker.py   # Circuit breaker for external calls
│   │   │   │   ├── mcp.py              # MCP tool registry (150+ tools)
│   │   │   │   └── ...
│   │   │   ├── cache_projector/         # CQRS read model projection service
│   │   │   └── platform_service/        # Cross-cutting platform service
│   │   ├── data/
│   │   │   └── init.sql                 # Schema + role creation DDL
│   │   ├── alembic/                     # Database migrations
│   │   └── tests/                       # 2,200+ tests (103 files)
│   │       ├── architecture/            # Architecture compliance tests
│   │       ├── domains/                 # Domain integration tests
│   │       ├── integration/             # Cross-service integration tests
│   │       └── unit/                    # Unit tests
│   ├── frontend/                        # React + TypeScript + Vite
│   │   └── src/
│   │       ├── api/
│   │       │   └── client.ts            # API client (3,118 lines), snake→camelCase transforms
│   │       ├── hooks/                   # Custom React hooks (9 hooks)
│   │       ├── surfaces/                # Actor-specific UI surfaces
│   │       │   ├── broker/              # 8 screens + 7 components + AI assistant
│   │       │   ├── shipper/             # 9 screens + 17 components
│   │       │   ├── platform/            # 15 screens + 27 components + AI assistant
│   │       │   └── buyer/               # 3 screens + 5 components
│   │       └── types.ts                 # Shared TypeScript types
│   ├── docker-compose.yml               # Dev compose (6 services)
│   ├── docker-compose.microservices.yml  # Full microservices (24 services)
│   ├── docker-compose.prod.yml          # Production compose
│   ├── Caddyfile                        # Dev reverse proxy
│   └── k8s/                             # Kubernetes manifests
├── infra/                               # Terraform (AWS EC2 provisioning)
│   ├── main.tf
│   ├── variables.tf
│   └── user-data.sh
└── docs/                                # Documentation
```

### 3.2 Per-Domain Service Structure

Each of the 15 domains follows an identical internal structure:

```
clearance_platform/domains/{domain}/
├── __init__.py          # Package init, public API exports
├── main.py              # Independent FastAPI app with lifespan
├── models.py            # SQLAlchemy ORM models (domain-scoped schema)
├── events.py            # Domain event definitions (state-transfer)
├── routes.py            # REST API endpoints (APIRouter)
├── service.py           # Business logic layer
├── consumers.py         # JetStream consumer handlers
├── projector.py         # Cache projection (where applicable)
└── Dockerfile           # Independent container build
```

Each domain's `main.py` creates a standalone FastAPI application with its own lifespan that:
1. Connects to PostgreSQL, Redis, and NATS
2. Creates JetStream streams
3. Wires consumer subscriptions
4. Exposes a `/health` endpoint
5. Handles graceful shutdown

### 3.3 Database Architecture

**Engine:** PostgreSQL 16 (Alpine) with `pg_trgm` and `btree_gin` extensions.

**Schema isolation:** 15 PostgreSQL schemas map to the 15 bounded contexts. All tables reside in the `public` schema during the transition period, with cross-schema access granted to all domain roles.

| Domain | Schema | DB Role | ORM Models |
|--------|--------|---------|------------|
| Product Catalog | `product` | `svc_product` | Product |
| Trade Intelligence | `trade_intel` | `svc_trade_intel` | Classification, TariffCalculation, FTADetermination + 12 reference models |
| Compliance | `compliance` | `svc_compliance` | RestrictedParty, PGAMapping, ComplianceResult |
| Order Management | `order_mgmt` | `svc_order_mgmt` | Order, OrderLineItem |
| Shipment Lifecycle | `shipment` | `svc_shipment` | Shipment |
| Cargo Handling Units | `cargo` | `svc_cargo` | HandlingUnit, HUTransitEvent |
| Consolidation | `consolidation` | `svc_consolidation` | Consolidation |
| Declaration Management | `declaration` | `svc_declaration` | Broker, BrokerAssignment, EntryFiling, BrokerMessage, ExportDeclaration, ISFFiling, ENSFiling |
| Customs Adjudication | `adjudication` | `svc_adjudication` | AdjudicationDecision |
| Exception Management | `exception` | `svc_exception` | ClearanceException |
| Financial Settlement | `financial` | `svc_financial` | FinancialRecord, Bond, DutyDrawback, Reconciliation |
| Regulatory Intelligence | `regulatory` | `svc_regulatory` | RegulatorySignal |
| Document Management | `document` | `svc_document` | Document |
| Party Management | `party` | `svc_party` | Party, PowerOfAttorney, ImporterOfRecord |
| Supply Chain Disruptions | `disruption` | `svc_disruption` | Disruption |

**Total:** 15 schemas, 16 roles (15 domain + `svc_platform`), 65 ORM models.

**Cross-domain model access:** `clearance_platform/shared/domain_models.py` is the sole allowed import path. Architecture tests enforce that no domain directly imports another domain's models.

**Event idempotency:** Each domain schema contains a `processed_events` table for consumer deduplication.

### 3.4 API Gateway

The gateway backend (`clearance_platform/gateway/`) serves as the single entry point:

**Router Registry** (`router_registry.py`): Imports and registers all domain routers plus cross-cutting gateway routers (dashboard, session, platform, chat, trade lanes, simulation, analysis, description quality).

**Development mode:** The gateway runs as a single FastAPI process (`clearance-backend` container) with all domain routes registered in-process. The backend code is bind-mounted for hot reload.

**Microservices mode:** Caddy (`Caddyfile.microservices`) routes by URL prefix to individual domain service containers. Each domain runs its own FastAPI process on port 4000.

| URL Prefix | Service |
|-----------|---------|
| `/api/products*` | svc-product-catalog |
| `/api/classify/*`, `/api/tariff`, `/api/fta`, `/api/jurisdiction*`, `/api/countries`, `/api/classifications*` | svc-trade-intelligence |
| `/api/compliance*`, `/api/screen` | svc-compliance |
| `/api/orders*` | svc-order-management |
| `/api/shipments*` | svc-shipment-lifecycle |
| `/api/cargo-handling-units*`, `/api/handling-units*` | svc-cargo-handling-units |
| `/api/consolidations*` | svc-consolidation |
| `/api/broker*`, `/api/export-declarations*` | svc-declaration-management |
| `/api/adjudication*` | svc-customs-adjudication |
| `/api/exception*`, `/api/resolution*` | svc-exception-management |
| `/api/financial*` | svc-financial-settlement |
| `/api/regulatory*` | svc-regulatory-intelligence |
| `/api/documents*` | svc-document-management |
| `/api/parties*` | svc-party-management |
| `/api/disruptions*` | svc-supply-chain-disruptions |
| `/api/dashboard*` | cache-projector |
| `/api/chat*`, `/api/session*`, `/api/platform*`, `/api/simulation*`, `/api/analysis*`, `/api/description-quality*`, `/api/trade-lanes*` | svc-platform |
| `/health` | svc-platform (aggregated) |
| `*` (fallback) | frontend |

Security headers on all responses: `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin`.

### 3.5 Container Topology

**Development mode** (6 containers via `docker-compose.yml`):

```
┌────────────────────────────────────────────────────────────┐
│  clearance-backend (port 4000)                              │
│  All domain routes in single FastAPI process                │
│  Bind-mounted source for hot reload                         │
├────────────────────────────────────────────────────────────┤
│  clearance-frontend (port 4001)                             │
│  Vite dev server with HMR                                   │
├─────────────┬──────────────┬──────────────┬────────────────┤
│  PostgreSQL │    Redis     │     NATS     │    Qdrant      │
│  port 5440  │  port 6381   │  port 4222   │  port 6336     │
└─────────────┴──────────────┴──────────────┴────────────────┘
```

**Microservices mode** (24 containers via `docker-compose.microservices.yml`):

```
┌─────────────────────────────────────────────────────────────┐
│  Caddy API Gateway (port 8080/8443)                          │
│  URL-prefix routing · TLS · Security headers · SSE flush     │
├─────────────────────────────────────────────────────────────┤
│                   Domain Services (15)                        │
│  svc-product-catalog      svc-trade-intelligence             │
│  svc-compliance           svc-order-management               │
│  svc-shipment-lifecycle   svc-cargo-handling-units            │
│  svc-consolidation        svc-declaration-management         │
│  svc-customs-adjudication svc-exception-management           │
│  svc-financial-settlement svc-regulatory-intelligence        │
│  svc-document-management  svc-party-management               │
│  svc-supply-chain-disruptions                                │
├─────────────────────────────────────────────────────────────┤
│  Cross-Cutting Services                                      │
│  svc-platform (simulation, chat, session, analysis)          │
│  cache-projector (CQRS read model projection)                │
├─────────────────────────────────────────────────────────────┤
│  clearance-frontend (nginx, port 4001)                       │
├─────────────┬──────────────┬──────────────┬─────────────────┤
│  PostgreSQL │    Redis     │     NATS     │    Qdrant       │
└─────────────┴──────────────┴──────────────┴─────────────────┘
```

Domain services do not expose host ports — all traffic routes through Caddy.

### 3.6 Shared Infrastructure Library

All shared code lives in `clearance_platform/shared/`:

| Module | Purpose |
|--------|---------|
| `event_bus.py` | NATS JetStream event bus with publish, subscribe, DLQ routing, in-process fallback |
| `streams.py` | 16 JetStream stream definitions with retention policies |
| `database.py` | Domain-to-PostgreSQL-schema mapping (15 entries) |
| `domain_models.py` | Cross-domain ORM model re-exports (sole allowed import path) |
| `events/base.py` | `BaseEvent` + `EventMetadata` with factory method |
| `idempotency.py` | Per-domain `processed_events` table and event deduplication |
| `mcp.py` | MCP tool registry with `@mcp_tool` decorator (150+ tools registered) |
| `circuit_breaker.py` | Three-state circuit breaker (CLOSED/OPEN/HALF_OPEN) for external calls |
| `time_service.py` | Production clock + simulation accelerated clock |
| `reconciliation.py` | 5-minute Redis vs PostgreSQL consistency check |
| `dlq_monitor.py` | Dead letter queue consumer with alerting |
| `intelligence_wiring.py` | Event-driven intelligence engine trigger wiring |
| `llm.py` | LLM client abstraction |
| `streaming.py` | SSE streaming utilities |
| `schemas/` | 12 shared Pydantic schema modules (broker, products, shipments, etc.) |
| `services/` | 10 shared service modules (analysis, dashboard, jurisdiction, etc.) |
| `engines/` | 8 intelligence engines (E0 pre-clearance, E1 classification, E2 tariff, E3 compliance, E4 FTA, E5 exception, E6 regulatory, E7 documents) |
| `jurisdiction/` | Abstract adapter + 7 country implementations (US, EU, CN, BR, IN, MX, CA) |
| `simulation/` | Coordinator, state machine, routing, reference data |
| `agent/` | AI agent prompts and MCP tool definitions |

### 3.7 Frontend Implementation

**Stack:** React 18 + TypeScript + Vite 5 + Tailwind CSS

**Key files:**
- `src/api/client.ts` (3,118 lines) — centralized API client with snake_case→camelCase transforms
- `src/types.ts` (1,376 lines) — all shared TypeScript type definitions
- `src/hooks/` — 9 custom hooks (analysis, classification, dashboard streaming, jurisdiction, simulation, etc.)

**Surface structure** — 4 actor-specific layouts with dedicated screens and components:

| Surface | Screens | Components | AI Assistant | Color Theme |
|---------|--------:|----------:|:------------|:-----------|
| Broker | 8 | 7 | BrokerAssistant (streaming chat, 11 tools) | Teal |
| Shipper | 9 | 17 | — | Amber |
| Platform | 15 | 27 | OperatorAssistant (streaming chat) | Purple |
| Buyer | 3 | 5 | — | Green |
| **Total** | **35** | **56** | — | — |

**State management:** Zustand stores — `operatorStore` (Platform), `brokerStore` (Broker queue/alerts/selected entry), `sessionStore` (surface/user), `cartStore` (Buyer).

**Frontend production build:** Multi-stage Docker build → nginx serving static assets on port 4001.

### 3.8 Infrastructure as Code

**Terraform** (`infra/`): Provisions AWS EC2 instances with:
- `main.tf` — EC2 instance, security groups, EBS volumes
- `variables.tf` — Configurable instance type, region, SSH key
- `user-data.sh` — Bootstrap script (Docker, Docker Compose, app deployment)

**Kubernetes** (`k8s/`): Alternative deployment manifests for container orchestration.

**Production deployment** (`deploy.sh`): One-command deployment script that builds images, pushes, and restarts services on the target host.

### 3.9 Deployment Configurations

| Config | File | Services | Use Case |
|--------|------|----------|----------|
| Development | `docker-compose.yml` | 6 | Local dev with hot reload, all routes in single backend process |
| Microservices | `docker-compose.microservices.yml` | 24 | Full microservices with Caddy gateway, per-domain containers |
| Production | `docker-compose.prod.yml` | 6+ | Production with Caddy, no exposed DB ports, optimized builds |

### 3.10 Testing Architecture

| Category | Files | Tests | Coverage |
|----------|------:|------:|----------|
| Architecture compliance | 23 | ~200 | Enforce domain boundaries, event contracts, CQRS discipline, NATS governance |
| Domain integration | 14 | ~500 | Cross-domain flows, event propagation, state transitions |
| Unit tests | 50+ | ~1,300 | Business logic, jurisdiction adapters, duty calculations, engine tests, ingestion |
| Integration tests | 5 | ~100 | JetStream round-trips, full-stack flows, CSL ingestion |
| E2E (Playwright) | 10 | ~240 | Browser-based user workflow testing across all surfaces |
| **Total** | **103** | **2,200+** | — |

Architecture tests enforce:
- No domain directly imports another domain's models (must use `domain_models.py`)
- All JetStream subjects are covered by stream definitions
- Retention policies match data classification requirements
- No TTL on Redis entity keys (CQRS consistency)
- Gateway registers all domain routers

### 3.11 Key Technology Stack

| Layer | Technology | Version |
|-------|------------|---------|
| Backend framework | FastAPI + Pydantic v2 | Python 3.12, uv |
| Database | PostgreSQL | 16 (Alpine) |
| Cache | Redis | 7 (Alpine) |
| Event streaming | NATS JetStream | 2.10 (Alpine) |
| Vector search | Qdrant | latest |
| Frontend | React + TypeScript + Vite | React 18, Vite 5 |
| E2E testing | Playwright | latest |
| Reverse proxy | Caddy | latest |
| IaC | Terraform | AWS provider |
| Container runtime | Docker Compose | v2 |
| Package management | uv (Python), npm (JS) | — |

### 3.12 Current Deployment State

As of 2026-02-27, the platform runs in **development mode** (6 containers): all domain routes in a single FastAPI backend process via the gateway router registry. The microservices-mode compose file is available for production-grade isolation.

**Active containers (dev):** 6 (backend, frontend, postgres, redis, nats, qdrant)

**Database:** Single PostgreSQL instance with 15 domain schemas + `public` schema. Extensions: `pg_trgm`, `btree_gin`.

**Background services:**
- **CSL Ingestion** — fetches Consolidated Screening List from trade.gov every 6 hours, loads 14 US federal restricted party lists into `compliance.restricted_parties`
- **Analysis Watchdog** — optional background catch-up for bulk imports (disabled by default; analysis runs on-demand)
- **Regulatory Monitor** — optional periodic regulatory signal scanning

**API stats:** 153 endpoints across 15 domains + gateway routes. 67 event types. 65 ORM models.
