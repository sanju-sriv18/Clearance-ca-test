# Business Capability Map — Clearance Vibe Platform

> **Purpose**: Domain-centric mapping of ALL platform business capabilities. Organized by what the business does, not how the code is structured. Intended to inform bounded context design for the event-driven architecture.

---

## 1. Product Catalog & Management

**Business purpose**: Maintain the master catalog of products that can be shipped internationally. Each product has trade-relevant attributes (HS codes, country of origin, descriptions, compliance flags).

### Data
| Table | Role |
|-------|------|
| `products` | Master product record — id, name, hs_code, description, country_of_origin, category, unit_price, weight_kg, material_composition, product_image_url |

### API Endpoints
| Route File | Endpoints | Surface |
|------------|-----------|---------|
| `products.py` | `GET /products`, `GET /products/{id}`, `POST /products`, `PUT /products/{id}`, `DELETE /products/{id}`, `POST /products/seed` | Shipper (catalog), Buyer (storefront), Platform (product analysis) |

### Engines
| Engine | Role |
|--------|------|
| `e1_classification` | HS code classification (LLM-powered) — validator, confidence scoring |
| `e0_preclearance` | Pre-clearance risk assessment for products before shipping |

### Services
| Service | Role |
|---------|------|
| `description_quality` | Evaluates product description quality for customs accuracy |
| `embedding` | Generates vector embeddings for semantic product search (Qdrant) |

### Frontend Surfaces
| Surface | Screen | Role |
|---------|--------|------|
| Shipper | `ShipperCatalog` | Product listing and management |
| Shipper | `ShipperProductDetail` | Product detail view with classification |
| Shipper | `AddProduct` | Create new product |
| Buyer | `StoreFront` | Browse products for purchase |
| Buyer | `ProductPage` | View product details and pricing |
| Platform | `ProductAnalysis` | Analyze product classification and compliance |

### Actors
- **None directly** — Product catalog is managed through API, not simulation actors.

### Key Insight
Products are a **pure master data domain**. They have no lifecycle states, no simulation actors, and no events in the current system. In an event-driven architecture, this is the simplest bounded context — CRUD operations that publish `ProductCreated`, `ProductUpdated`, `ProductClassified` events for downstream consumption.

---

## 2. HS Classification & Tariff Intelligence

**Business purpose**: Determine the correct Harmonized System (HS) code for products, calculate applicable duties/taxes across multiple jurisdictions, apply trade programs and special tariff regimes.

### Data
| Table | Role |
|-------|------|
| `htsus_chapters` | US Harmonized Tariff Schedule — chapter-level duty rates |
| `htsus_headings` | HTS heading-level detail with descriptions and rates |
| `section_301_lists` | China Section 301 tariff lists and exclusions |
| `section_232_scope` | Steel/aluminum Section 232 tariffs |
| `ieepa_rates` | International Emergency Economic Powers Act rates |
| `adcvd_orders` | Antidumping/Countervailing duty orders |
| `cross_rulings` | CBP cross-ruling database (Qdrant vector-linked) |
| `tax_rates` | Jurisdiction-specific tax rates (VAT, GST, etc.) |
| `fee_schedules` | MPF, HMF, and other fee structures |
| `exchange_rates` | Currency exchange rates for duty calculation |
| `tax_regime_templates` | Pre-built tariff regime configurations by country |
| `fta_rules` | Free Trade Agreement eligibility rules |

### API Endpoints
| Route File | Endpoints | Surface |
|------------|-----------|---------|
| `tariff.py` | `POST /tariff/calculate` | All surfaces (tariff computation) |
| `classify.py` | `POST /classify/stream` (LLM, rate-limited) | Shipper, Platform |
| `fta.py` | `POST /fta/check` | Platform, Broker |
| `trade_lanes.py` | `POST /trade-lanes/compare` | Platform |
| `jurisdiction.py` | `GET /jurisdictions`, `GET /jurisdictions/{code}`, `GET /jurisdictions/countries` | Platform |

### Engines
| Engine | Role |
|--------|------|
| `e1_classification` | LLM-powered HS code classification with confidence scoring and validation |
| `e2_tariff/engine` | Main tariff calculation orchestrator |
| `e2_tariff/regimes/us` | US-specific tariff rules (Section 301, 232, IEEPA, ADCVD, de minimis) |
| `e2_tariff/regimes/eu` | EU tariff regime |
| `e2_tariff/regimes/uk` | UK tariff regime |
| `e2_tariff/regimes/india` | India tariff regime |
| `e2_tariff/regimes/br` | Brazil tariff regime |
| `e2_tariff/regimes/cn` | China tariff regime |
| `e2_tariff/programs` | Trade preference programs (GSP, USMCA, etc.) |
| `e2_tariff/tax_regime` | Tax regime framework (VAT/GST calculation) |
| `e2_tariff/landed_cost` | Full landed cost calculation |
| `e4_fta` | Free Trade Agreement eligibility engine |

### Services
| Service | Role |
|---------|------|
| `currency` | Exchange rate service for multi-currency tariff calculations |
| `jurisdiction` | Jurisdiction-specific regulatory requirements |

### Frontend Surfaces
| Surface | Screen | Role |
|---------|--------|------|
| Buyer | `CheckoutPrice` | Display landed cost breakdown at checkout |
| Platform | `TradeLaneComparison` | Compare tariff impact across corridors |
| Platform | `ProductAnalysis` | Shared — shows tariff analysis for products |

### Actors
- **`preclearance`** — Runs pre-clearance assessment which includes classification validation
- **`compliance_engine`** — Triggers tariff calculation as part of compliance pipeline

### Key Insight
Classification and Tariff are **computation-heavy, stateless operations**. They don't own lifecycle state — they are invoked by other domains (Order, Shipment, Broker) to compute tariff obligations. In bounded context terms, this is a **shared kernel / supporting service** rather than a core domain. It consumes `ProductClassified` events and produces `TariffCalculated`, `FTAEligibilityDetermined` events.

---

## 3. Order Lifecycle

**Business purpose**: Manage the lifecycle of purchase orders from creation through to shipment. An order groups line items (products × quantities) and transitions from draft to shipped.

### Data
| Table | Role |
|-------|------|
| `orders` | Order header — id, status, buyer_name, shipping_address, destination_country, total_value, etc. |
| `order_line_items` | Line items — product_id, quantity, unit_price, line_total |

### API Endpoints
| Route File | Endpoints | Surface |
|------------|-----------|---------|
| `orders.py` | `GET /orders`, `GET /orders/{id}`, `POST /orders`, `PUT /orders/{id}`, `POST /orders/{id}/analyze`, `POST /orders/{id}/ship` | Shipper (primary), Platform (read-only view) |
| `documents.py` | `POST /orders/{id}/documents/upload`, `GET /orders/{id}/documents` | Shipper, Broker |

### Engines
- **None directly** — Orders use classification/tariff engines via orchestration pipeline when analyzed.

### Services
| Service | Role |
|---------|------|
| `shipment_analysis` | Analyzes orders to assess shipment risk/complexity |

### Orchestration
| Module | Role |
|--------|------|
| `pipeline.py` | Orchestrates the classification → tariff → compliance pipeline when order is analyzed |
| `parallel.py` | Parallel execution of independent pipeline stages |
| `error_handling.py` | Pipeline error recovery |

### Frontend Surfaces
| Surface | Screen | Role |
|---------|--------|------|
| Shipper | `CreateOrder` | Create new order with line items |
| Shipper | `OrderList` | View all orders |
| Shipper | `OrderDetail` | Order detail with analysis results |
| Platform | `PlatformOrders` | Platform view of all orders |
| Platform | `PlatformOrderDetail` | Platform order detail view |

### Actors
- **`shipper`** — Creates shipments FROM orders (transitions order to "shipped", creates associated shipment)

### Key Insight
Orders have a **short, simple lifecycle** (draft → confirmed → shipped). The moment an order is "shipped", the **Shipment Lifecycle** domain takes over. Orders and Shipments share a creation relationship but have very different lifecycles, stakeholders, and state machines. They should be **separate bounded contexts** with a clear handoff event: `ShipmentCreatedFromOrder`.

---

## 4. Shipment Lifecycle

**Business purpose**: Track the physical and regulatory lifecycle of shipments from origin to final delivery. This is the **core domain** — the longest-lived entity with the most complex state machine, touching the most actors and the most business rules.

### Data
| Table | Role |
|-------|------|
| `shipments` | Master shipment record — id, order_id, status, origin/destination, transport_mode, carrier, tracking, hs_code, duty_rate, classification_result (JSONB), compliance_result (JSONB), tariff_result (JSONB), fta_result (JSONB), events (JSONB), intermediate_events (JSONB), routing (JSONB), documents (JSONB) |
| `consolidations` | Shipment grouping for multi-modal transport — id, master_tracking, status, shipment_count, consolidation_type |
| `documents` | Generated/uploaded documents associated with shipments |
| `demo_shipments` | Template shipments for demonstration purposes |

### API Endpoints
| Route File | Endpoints | Surface |
|------------|-----------|---------|
| `shipments.py` | `GET /shipments`, `GET /shipments/lookup`, `GET /shipments/{id}`, `POST /shipments`, `POST /shipments/{id}/analyze`, `GET /shipments/{id}/events`, `GET /shipments/{id}/documents` | All surfaces |
| `consolidations.py` | `GET /consolidations`, `GET /consolidations/{id}`, `GET /consolidations/{id}/siblings` | Broker, Platform |
| `documents.py` | `GET /documents/requirements/{id}`, `POST /documents/generate/{id}`, `POST /documents/analyze`, `POST /shipments/{id}/documents/upload`, `GET /shipments/{id}/documents`, `GET /documents/{id}/content` | Broker, Shipper |

### Engines
| Engine | Role |
|--------|------|
| `e7_documents` | Document generation (commercial invoice, packing list, bill of lading, etc.) |

### Services
| Service | Role |
|---------|------|
| `shipment_analysis` | Risk/complexity analysis for shipments |

### Frontend Surfaces
| Surface | Screen | Role |
|---------|--------|------|
| Shipper | `ShipmentDetail` | Track shipment status and events |
| Shipper | `ShipmentHistory` | Historical shipment list |
| Platform | `ActiveShipments` | Real-time view of all active shipments |
| Platform | `PlatformShipmentDetail` | Detailed shipment view with all data |
| Platform | `ControlTower` | Aggregated shipment monitoring dashboard |

### Actors (State Machine Participants)
| Actor | Role in Shipment Lifecycle |
|-------|---------------------------|
| `shipper` | Creates shipment (→ `pending`) |
| `preclearance` | Pre-clearance assessment (→ `preclearance_review`) |
| `consolidator` | Groups into consolidation (→ `consolidated`) |
| `carrier` | Transit management (→ `in_transit`, `at_port`, intermediate legs) |
| `customs` | Customs submission (→ `at_customs`, `customs_review`) |
| `cage` | Customs exam (→ `in_exam`) |
| `cbp_authority` | CBP processing (→ `entry_under_review`, `cleared`, `held`) |
| `pga` | PGA review (→ `pga_review`, `pga_cleared`) |
| `resolution` | Hold resolution (→ `resolution_pending`, `resolved`) |
| `terminal` | Terminal delays (→ `terminal_hold`) |
| `disruption` | Supply chain disruptions (→ `disrupted`) |
| `exceptions` | Exception generation (→ various exception states) |
| `demurrage` | Demurrage/detention cost accrual |
| `financial` | Duty/fee calculation finalization |
| `documents` | Document generation and verification |
| `isf` | ISF filing for ocean shipments |
| `shipper_response` | Shipper notification/response to holds |

### Key Insight
The shipment lifecycle is the **heart of the platform** but it's currently a monolith — 16+ actors all mutating the same `shipments` table rows with 8 JSONB columns. The state machine has ~30 status values. In bounded context terms, the shipment lifecycle should probably be decomposed into sub-domains:
- **Transport & Logistics** (carrier, consolidator, terminal, disruption) — physical movement
- **Customs Processing** (customs, cbp_authority, cage, pga) — regulatory clearance
- **Exception & Resolution** (exceptions, resolution, shipper_response) — problem handling
- **Financial Settlement** (financial, demurrage) — cost accrual and finalization

The `shipments` table is the **shared aggregate root** — all sub-domains need it, which is the core architectural challenge.

---

## 5. Compliance & Risk Screening

**Business purpose**: Screen shipments, entities, and products against regulatory requirements — denied party lists, UFLPA, PGA requirements, restricted materials. This is a **regulatory obligation**, not optional.

### Data
| Table | Role |
|-------|------|
| `restricted_parties` | Denied/restricted party lists (DPS, SDN, Entity List) — uses pg_trgm for fuzzy matching |
| `pga_mappings` | Partner Government Agency requirements by HS code (FDA, EPA, CPSC, etc.) |
| `fta_rules` | Free Trade Agreement eligibility rules and requirements |

### API Endpoints
| Route File | Endpoints | Surface |
|------------|-----------|---------|
| `compliance.py` | `POST /compliance/screen` | Platform, Broker |
| `screening.py` | `POST /screening/entity` | Platform (Entity Screening) |
| `fta.py` | `POST /fta/check` | Platform |

### Engines
| Engine | Role |
|--------|------|
| `e3_compliance/engine` | Compliance screening orchestrator |
| `e3_compliance/dps` | Denied Party Screening engine |
| `e3_compliance/uflpa` | UFLPA (Uyghur Forced Labor Prevention Act) screening |
| `e3_compliance/pga` | PGA requirement determination |
| `e3_compliance/fuzzy_match` | Fuzzy name matching for restricted party screening |
| `e4_fta` | FTA eligibility engine |

### Frontend Surfaces
| Surface | Screen | Role |
|---------|--------|------|
| Platform | `EntityScreening` | Screen entities against denied party lists |
| Platform | `ComplianceDashboard` | Compliance status overview |

### Actors
| Actor | Role |
|-------|------|
| `compliance_engine` | Runs compliance screening on shipments entering clearance pipeline |
| `pga` | Simulates PGA review timelines and outcomes |

### Key Insight
Compliance is a **cross-cutting concern** — it's invoked during order analysis, shipment processing, and broker operations. It doesn't own shipment state, but its results (compliance_result JSONB on shipments) influence state transitions (a compliance failure can cause a hold). In event terms, it **consumes** `ShipmentEnteredClearance` and **produces** `ComplianceScreeningCompleted` (with pass/flag/fail).

---

## 6. Broker Operations

**Business purpose**: Customs brokers manage the clearance process — they claim shipments, verify documentation, prepare and submit entry filings to CBP, respond to CBP requests (CF-28/CF-29), communicate with shippers, and track broker-specific financials (fees, duty calculations).

### Data
| Table | Role |
|-------|------|
| `brokers` | Broker identity — id, name, license_number, specializations |
| `broker_assignments` | Links brokers to shipments — broker_id, shipment_id, assigned_at, status |
| `entry_filings` | CBP entry filings — filing_number, shipment_id, status, filing_data (JSONB), cbp_response (JSONB) |
| `broker_messages` | Communications between brokers, shippers, and CBP — from_role, to_role, subject, body |

### API Endpoints
| Route File | Endpoints | Surface |
|------------|-----------|---------|
| `broker.py` | **30+ endpoints** including: queue management, claim/release, checklist verification, entry submission, CF-28/CF-29 responses, broker insights (LLM), fee calculation, communication, document management | Broker (primary) |

### Services
| Service | Role |
|---------|------|
| `broker_intelligence` | LLM-powered broker assistance — classification suggestions, document analysis, risk insights |

### Frontend Surfaces
| Surface | Screen | Role |
|---------|--------|------|
| Broker | `BrokerDashboard` | Overview dashboard — workload, KPIs |
| Broker | `BrokerQueue` | Work queue — shipments awaiting broker action |
| Broker | `EntryDetail` | Full entry detail — documents, checklist, filing |
| Broker | `CBPResponses` | CBP response management (CF-28, CF-29) |
| Broker | `Communications` | Broker-shipper-CBP messaging |
| Broker | `RegulatoryIntelligence` | Regulatory intelligence feed |
| Broker | `BrokerDocumentChecklist` (component) | Document verification checklist |

### Actors
| Actor | Role |
|-------|------|
| `broker_sim` | Simulates broker workflow: claim → verify → prepare → submit → respond. Most complex actor (5 sequential phases, 4-5 tables per transaction) |
| `cbp_authority` | Simulates CBP processing of broker-submitted entries |

### Key Insight
Broker Operations is the **most user-facing** domain — it has the richest UI, the most API endpoints, and the most complex actor. It's currently tightly coupled to the Shipment Lifecycle domain because `broker_sim` reads/writes the `shipments` table directly. In bounded context terms, Broker Operations should be a **separate context** that consumes shipment events and manages its own state (assignments, filings, messages, checklist items). The broker doesn't need to write to `shipments` — it should produce events like `EntryFilingSubmitted`, `ChecklistItemVerified`, `BrokerCommunicationSent`.

---

## 7. Customs Authority (CBP Simulation)

**Business purpose**: Simulate the behavior of U.S. Customs and Border Protection — process entry filings, issue holds, request additional information (CF-28), issue penalty notices (CF-29), and ultimately clear or reject entries.

### Data
| Table | Role |
|-------|------|
| `entry_filings` | Shared with Broker Operations — CBP writes `cbp_response` JSONB, updates `status` |
| `shipments` | CBP transitions shipment status (→ cleared, held, entry_under_review) |

### API Endpoints
| Route File | Endpoints | Surface |
|------------|-----------|---------|
| `exception_status.py` | `GET /exception-status/{id}` | Platform |
| `broker.py` | Shared — CF-28/CF-29 response endpoints are used by both broker and platform |

### Actors
| Actor | Role |
|-------|------|
| `cbp_authority` | Main CBP simulation — processes filings, issues holds, clears entries |
| `pga` | Partner Government Agency review (FDA, EPA, etc.) — parallel authority |
| `cage` | Customs exam facility — physical inspection process |

### Frontend Surfaces
| Surface | Screen | Role |
|---------|--------|------|
| Broker | `CBPResponses` | Shared — broker views and responds to CBP actions |
| Platform | `EntryDetail` | Platform view of entry processing |

### Key Insight
The Customs Authority is a **separate regulatory entity** from the broker. Today, `cbp_authority`, `pga`, and `cage` all directly mutate shipment rows. In the event model, these should consume `EntryFilingSubmitted` and produce `CBPDecisionMade`, `PGAReviewCompleted`, `CageExamCompleted`. They are **event consumers that produce authoritative decisions**, which then drive the shipment state machine.

---

## 8. Exception Management & Resolution

**Business purpose**: Handle exceptions, holds, and problems that arise during clearance — from tariff disputes to documentation deficiencies to supply chain disruptions. Includes the shipper's response to hold notifications.

### Data
| Table | Role |
|-------|------|
| `shipments` | Exception data stored in `events` and `intermediate_events` JSONB columns |
| `broker_messages` | Communications about exceptions |

### API Endpoints
| Route File | Endpoints | Surface |
|------------|-----------|---------|
| `resolution.py` | `POST /resolution/lookup`, `POST /resolution/upload-document`, `POST /resolution/suggest` (LLM, rate-limited) | Shipper, Platform |
| `exception.py` | `POST /exception/stream` (LLM, rate-limited) | Platform |
| `exception_status.py` | `GET /exception-status/{id}` | Platform |

### Engines
| Engine | Role |
|--------|------|
| `e5_exception` | Exception analysis and resolution suggestion engine (LLM-powered) |

### Frontend Surfaces
| Surface | Screen | Role |
|---------|--------|------|
| Shipper | `ResolutionCenter` | Shipper-side hold resolution and document upload |
| Platform | `ExceptionResolution` | Platform-wide exception management |

### Actors
| Actor | Role |
|-------|------|
| `exceptions` | Generates exceptions on shipments (random + rule-based) |
| `resolution` | Processes hold resolutions (time-based + document-based) |
| `shipper_response` | Simulates shipper responding to hold notifications |
| `disruption` | Generates supply chain disruptions |
| `terminal` | Generates terminal-related delays and holds |

### Key Insight
Exception Management is **reactive and cross-cutting** — exceptions can be triggered by compliance failures, CBP holds, physical inspections, or supply chain events. The current implementation has 5 actors that all independently scan and mutate shipments. In event-driven terms, these should react to events like `ShipmentHeld`, `CBPRequestIssued`, `ExamRequired` and produce `ExceptionRaised`, `ResolutionSubmitted`, `HoldReleased`.

---

## 9. Financial & Duty Settlement

**Business purpose**: Calculate final duty obligations, track fees (MPF, HMF), manage demurrage/detention costs, and settle financial obligations associated with customs clearance.

### Data
| Table | Role |
|-------|------|
| `shipments` | Financial data stored in `tariff_result` JSONB — duty_rate, total_duty, mpf, hmf |
| `fee_schedules` | Fee structures (MPF, HMF rates and caps) |
| `exchange_rates` | Currency conversion for duty calculation |
| `tax_rates` | Jurisdiction-specific VAT/GST rates |

### API Endpoints
| Route File | Endpoints | Surface |
|------------|-----------|---------|
| `tariff.py` | `POST /tariff/calculate` | All surfaces |
| `broker.py` | Fee-related endpoints within broker API | Broker |

### Engines
| Engine | Role |
|--------|------|
| `e2_tariff/landed_cost` | Full landed cost with all fees and taxes |
| `e2_tariff/tax_regime` | Tax regime calculations |

### Services
| Service | Role |
|---------|------|
| `currency` | Exchange rate management |

### Actors
| Actor | Role |
|-------|------|
| `financial` | Finalizes duty/fee calculations on cleared shipments |
| `demurrage` | Accrues demurrage/detention costs during delays |

### Key Insight
Financial Settlement has a **clear trigger point**: it happens after clearance. `financial` runs on cleared shipments, `demurrage` runs on delayed shipments. These are clean event consumers: `ShipmentCleared` → finalize duties, `ShipmentDelayed` → start accruing costs. Currently, financial writes to the same JSONB blobs on `shipments`. In the event model, financial results should be separate projections.

---

## 10. Regulatory Intelligence

**Business purpose**: Monitor and disseminate regulatory changes — tariff modifications, new trade restrictions, policy changes — that affect clearance operations.

### Data
| Table | Role |
|-------|------|
| `regulatory_signals` | Regulatory change signals — source, type, affected_hs_codes, severity, full_text |

### API Endpoints
| Route File | Endpoints | Surface |
|------------|-----------|---------|
| `regulatory.py` | `GET /regulatory/signals`, `POST /regulatory/signals`, `PUT /regulatory/signals/{id}`, `DELETE /regulatory/signals/{id}`, `POST /regulatory/scenario`, `POST /regulatory/refresh`, `GET /regulatory/monitor-status` | Platform, Broker |

### Engines
| Engine | Role |
|--------|------|
| `e6_regulatory/engine` | Regulatory signal analysis engine |
| `e6_regulatory/db` | Regulatory signal persistence |

### Services
| Service | Role |
|---------|------|
| `regulatory_monitor` | Background monitoring service for regulatory changes |
| `feed_parsers` | Parse regulatory feeds (Federal Register, CBP bulletins, etc.) |

### Frontend Surfaces
| Surface | Screen | Role |
|---------|--------|------|
| Broker | `RegulatoryIntelligence` | Broker view of regulatory changes affecting their work |
| Platform | `RegulatoryIntel` | Platform-wide regulatory intelligence dashboard |

### Actors
- **None** — Regulatory monitoring runs as a background service, not a simulation actor.

### Key Insight
Regulatory Intelligence is an **autonomous supporting service** — it has no dependency on the shipment state machine. It publishes `RegulatorySignalDetected` events that other domains can subscribe to (e.g., tariff engine updates rates, compliance engine updates screening lists). Clean, natural bounded context.

---

## 11. Dashboard, Reporting & Analytics

**Business purpose**: Provide aggregated views, KPIs, compliance metrics, and operational dashboards across all surfaces.

### Data
| Table | Role |
|-------|------|
| All tables | Dashboards aggregate across shipments, orders, brokers, compliance results |

### API Endpoints
| Route File | Endpoints | Surface |
|------------|-----------|---------|
| `dashboard.py` | `GET /dashboard` (duty breakdown, compliance alerts, volume metrics) | Platform |
| `platform.py` | `GET /platform/compliance-dashboard` | Platform |
| `analysis.py` | `POST /analyze/stream` (LLM, rate-limited) | Platform |

### Services
| Service | Role |
|---------|------|
| `dashboard` | Aggregation service for dashboard data |
| `analysis_watchdog` | Background service monitoring for anomalies |

### Frontend Surfaces
| Surface | Screen | Role |
|---------|--------|------|
| Platform | `ControlTower` | Main operational dashboard — real-time shipment monitoring |
| Platform | `ComplianceDashboard` | Compliance metrics and trends |
| Broker | `BrokerDashboard` | Broker-specific KPIs |

### Actors
- **None directly** — Dashboards are read-only projections.

### Key Insight
Dashboards are **pure read models** — they aggregate state from other domains. In the event-driven architecture, dashboards are the **canonical use case for read model projections**. Instead of querying operational tables (current approach, which contributes to connection pool contention), dashboards should be materialized views updated by events. This is where the event architecture delivers the most immediate value.

---

## 12. Communication & Collaboration

**Business purpose**: Manage multi-party communication between brokers, shippers, CBP, and the platform — including chat, messaging, and session management.

### Data
| Table | Role |
|-------|------|
| `broker_messages` | Structured messages between parties |

### API Endpoints
| Route File | Endpoints | Surface |
|------------|-----------|---------|
| `chat.py` | `POST /chat/stream` (LLM, rate-limited) | All surfaces |
| `session.py` | Session management endpoints | All surfaces |

### Services
| Service | Role |
|---------|------|
| `llm` | LLM service for AI-powered chat |
| `cache` | Session/conversation caching (Redis) |

### Frontend Surfaces
| Surface | Screen | Role |
|---------|--------|------|
| Broker | `Communications` | Broker messaging with shippers and CBP |

### Key Insight
Communication is a **supporting service** — it doesn't own domain state but facilitates collaboration between domains. In event-driven terms, it could consume events to auto-notify stakeholders (e.g., `ShipmentHeld` → auto-notify shipper).

---

## 13. Transport & Logistics

**Business purpose**: Manage the physical movement of shipments — carrier transit, multi-modal routing, consolidation, terminal operations, ISF filing for ocean freight.

### Data
| Table | Role |
|-------|------|
| `shipments` | Routing (JSONB), transport_mode, carrier, tracking data |
| `consolidations` | Multi-shipment grouping for transport |

### API Endpoints
| Route File | Endpoints | Surface |
|------------|-----------|---------|
| `consolidations.py` | `GET /consolidations`, `GET /consolidations/{id}`, `GET /consolidations/{id}/siblings` | Broker, Platform |
| `shipments.py` | Events and tracking endpoints | All surfaces |

### Actors
| Actor | Role |
|-------|------|
| `carrier` | Transit simulation — manages physical movement through route legs, handles intermediate clearance events |
| `consolidator` | Groups shipments into consolidations based on routing |
| `terminal` | Terminal operations — delays, container handling |
| `disruption` | Supply chain disruption events |
| `isf` | ISF (Importer Security Filing) for ocean shipments |

### Key Insight
Transport & Logistics is a distinct sub-domain of the shipment lifecycle. The `carrier` actor is the **most complex actor** in the system — it manages multi-leg routing, intermediate regulatory events at transshipment hubs, and consolidation side-effects. It's the one most likely to benefit from decomposition. In event-driven terms: `carrier` produces `ShipmentDeparted`, `ShipmentArrived`, `LegCompleted`, `IntermediateClearanceRequired`.

---

## 14. Simulation Control

**Business purpose**: Control the simulation engine — start/stop/reset, configure actor parameters, manage tick rates.

### API Endpoints
| Route File | Endpoints | Surface |
|------------|-----------|---------|
| `simulation.py` | Simulation control endpoints (start, stop, reset, configure) | Platform |

### Frontend Surfaces
| Surface | Screen | Role |
|---------|--------|------|
| Platform | `SimulationControl` | Simulation management UI |

### Key Insight
Simulation Control is **infrastructure**, not business domain. It's the mechanism that drives actors, not a business capability. In the event-driven architecture, this becomes the **event bus orchestrator** rather than a domain.

---

## Summary: Business Capabilities → Bounded Context Mapping

| # | Business Capability | Core Tables | Actor Count | API Richness | UI Surfaces | Proposed Bounded Context |
|---|---------------------|-------------|-------------|--------------|-------------|--------------------------|
| 1 | Product Catalog | `products` | 0 | Low (CRUD) | Shipper, Buyer, Platform | **Product Catalog** |
| 2 | HS Classification & Tariff | 7 reference tables | 0 direct | Medium | Buyer, Platform | **Trade Intelligence** (shared kernel) |
| 3 | Order Lifecycle | `orders`, `order_line_items` | 1 (shipper) | Medium | Shipper, Platform | **Order Management** |
| 4 | Shipment Lifecycle | `shipments`, `consolidations`, `documents` | 16+ | High | All surfaces | **Shipment Processing** (core domain, subdivide further) |
| 5 | Compliance & Screening | `restricted_parties`, `pga_mappings`, `fta_rules` | 2 | Medium | Platform | **Compliance & Screening** |
| 6 | Broker Operations | `brokers`, `broker_assignments`, `entry_filings`, `broker_messages` | 2 (broker_sim, cbp_authority) | Very High (30+ endpoints) | Broker | **Broker Operations** (core domain) |
| 7 | Customs Authority | `entry_filings` (shared) | 3 | Low | Broker, Platform | **Regulatory Authority** |
| 8 | Exception & Resolution | event JSONB data | 5 | Medium | Shipper, Platform | **Exception Management** |
| 9 | Financial Settlement | tariff JSONB data, fee/rate tables | 2 | Low | All (embedded) | **Financial Settlement** |
| 10 | Regulatory Intelligence | `regulatory_signals` | 0 | Medium | Broker, Platform | **Regulatory Intelligence** |
| 11 | Dashboard & Reporting | all (read-only) | 0 | Low | Platform, Broker | **Reporting** (read projection) |
| 12 | Communication | `broker_messages` | 0 | Low | Broker | Part of **Broker Operations** |
| 13 | Transport & Logistics | `shipments` routing, `consolidations` | 5 | Low | Platform | Sub-domain of **Shipment Processing** |
| 14 | Simulation Control | none | all (meta) | Low | Platform | **Infrastructure** (not a BC) |

---

## Critical Observations for Bounded Context Design

### 1. Products ARE Missing from the Current Architecture Design
The current event-driven design has no Product bounded context. Products are the **starting point** of the entire value chain: Product → Order → Shipment → Clearance → Settlement. Products have their own master data lifecycle (created, classified, updated), their own API surface, and their own UI across 3 surfaces. **Product Catalog should be an explicit bounded context.**

### 2. Orders and Shipments Are Distinct Domains
The current design lumps orders and shipments into a single "Shipment Lifecycle" context. But:
- **Orders** are owned by the shipper, have a simple lifecycle (draft → confirmed → shipped), and are about *what* is being shipped
- **Shipments** are owned by the logistics/customs pipeline, have a complex 30-state lifecycle, and are about *how* things move and clear
- The handoff point is clear: `POST /orders/{id}/ship` creates a shipment from an order
- Different stakeholders: shippers care about orders, brokers care about shipments

### 3. Broker Operations Deserves First-Class Status
With 30+ API endpoints and the richest UI, Broker Operations is not a sub-domain of shipment processing — it's a **parallel core domain**. Brokers have their own state (assignments, filings, messages, checklist items), their own lifecycle (claim → verify → prepare → submit → respond), and their own users.

### 4. The Shipments Table is a God Object
`shipments` has 8 JSONB columns holding state from at least 6 different business capabilities:
- `classification_result` → Trade Intelligence
- `compliance_result` → Compliance & Screening
- `tariff_result` → Trade Intelligence / Financial
- `fta_result` → Trade Intelligence
- `events` → Transport & Logistics + Exception Management
- `intermediate_events` → Transport & Logistics
- `routing` → Transport & Logistics
- `documents` → Broker Operations / Document Management

This is the **primary architectural anti-pattern**. Event-driven architecture should decompose this into domain-specific stores.

### 5. Compliance is Cross-Cutting, Not a Phase
Compliance screening happens at multiple points: order analysis, pre-clearance, broker review, and potentially on regulatory signal changes. It's not a step in the lifecycle — it's a capability invoked by multiple domains.

### 6. Financial Settlement is a Distinct Outcome Domain
Financial data (duties, fees, demurrage) is currently embedded in shipment JSONB. But financial settlement has its own triggers (clearance completion, delay events), its own calculations (tariff + fees + taxes + demurrage), and its own stakeholders (finance team, broker fee calculation). It should produce its own events and maintain its own projections.

---

## Appendix A: Shipment State Tracking Richness

> **CRITICAL**: The stakeholder identifies shipment state tracking richness as a **platform differentiator**. The event-driven architecture must be **at least as expressive** as the current system. Do not flatten, simplify, or lose any granularity.

### A.1 Five Jurisdiction-Specific State Machines

The platform does NOT have a single state machine. It has **5 jurisdiction-specific lifecycles**, each reflecting real customs authority workflows:

#### US (CBP) — 12 statuses, most complex
```
booked → in_transit → at_customs → entry_filed → {cleared, cf28_pending, cf29_pending, exam_scheduled, held}
cf28_pending → {entry_filed (re-submit), held}
cf29_pending → {cleared (pay), protest_filed}
exam_scheduled → {cleared, held}
protest_filed → {cleared, held}
held → {cleared, at_customs, in_transit} (resolution)
cleared → delivered
```
**US-specific states**: `entry_filed`, `cf28_pending`, `cf29_pending`, `exam_scheduled`, `protest_filed` — these map to actual CBP processes with no equivalent in other jurisdictions.

#### EU — 10 statuses
```
booked → in_transit → at_customs → lodged → accepted → {under_control, released, cleared}
under_control → {released, cleared, held}
released → cleared → delivered
```
**EU-specific states**: `lodged`, `accepted`, `under_control`, `released` — reflects EU customs declaration lifecycle.

#### Brazil (Receita Federal) — 8 statuses
```
booked → in_transit → at_customs → registered → parameterized → cleared → delivered
```
**BR-specific states**: `registered`, `parameterized` — reflects Siscomex channels (green/yellow/red/grey parameterization).

#### India (ICEGATE/CBIC) — 9 statuses
```
booked → in_transit → at_customs → filed → assessed → out_of_charge → cleared → delivered
```
**IN-specific states**: `filed`, `assessed`, `out_of_charge` — reflects Indian Bill of Entry lifecycle.

#### China (GACC/Single Window) — 9 statuses
```
booked → in_transit → at_customs → declared → {inspected, released} → cleared → delivered
```
**CN-specific states**: `declared`, `inspected`, `released` — reflects GACC Single Window workflow.

#### UK — uses EU transitions (pending expansion)

**Architectural implication**: The event-driven architecture must support **jurisdiction-polymorphic state machines**. A `ShipmentStatusChanged` event must carry the jurisdiction context. Read model projections must handle jurisdiction-specific statuses. The `can_transition()` function dispatches to the correct jurisdiction map — this dispatch logic must be preserved.

### A.2 Consolidation State Machine

Parallel to shipment lifecycle, consolidations have their own 6-state lifecycle:
```
booked → closed → in_transit → arrived → deconsolidating → deconsolidated
```
Consolidation transitions are controlled by actor permissions (consolidator creates, carrier manages transit/decon). Terminal states: `{deconsolidated}`.

**Architectural implication**: Consolidations group multiple shipments. When a consolidation transitions, **all member shipments** may need to react (e.g., when a consolidation arrives at port, individual shipments transition to `at_customs`). This is a **one-to-many event fan-out** pattern.

### A.3 Mode-Specific Event Granularity (40+ Event Types)

The platform tracks shipments with **mode-specific event vocabularies** — different transport modes generate different event types with different terminology. This is NOT generic "shipment moved" tracking.

#### Air Freight Events (13 types)
| Event Type | Description | Actor |
|------------|-------------|-------|
| `hawb_issued` | House Air Waybill booked on carrier | shipper |
| `acas_filed` | ACAS advance filing submitted | platform |
| `cargo_tendered` | Cargo tendered to carrier at origin (pcs, weight) | carrier |
| `mawb_consolidated` | HAWB consolidated into Master AWB | carrier |
| `uld_buildup` | Cargo loaded into ULD (type, id) on MAWB | carrier |
| `flight_departed` | Flight departed origin with MAWB | carrier |
| `flight_arrived` | Flight arrived at destination | carrier |
| `uld_breakdown` | ULD breakdown at destination terminal | carrier |
| `entry_filed` | Entry filed via ACE for HAWB | platform |
| `customs_cleared` | Entry cleared, duties assessed | customs |
| `released` | HAWB released | carrier |
| `delivered` | HAWB delivered, POD obtained | carrier |

#### Ocean Freight Events (13 types)
| Event Type | Description | Actor |
|------------|-------------|-------|
| `hbl_issued` | House Bill of Lading booked | shipper |
| `isf_filed` | ISF filed (24hr pre-load requirement) | platform |
| `cargo_received_cfs` | Cargo received at origin CFS | carrier |
| `container_stuffed` | Container stuffed, seal applied — MBL | carrier |
| `vessel_departed` | Vessel departed origin port | carrier |
| `vessel_arrived` | Vessel arrived at destination | carrier |
| `container_discharged` | Container discharged from vessel | carrier |
| `entry_filed` | Entry filed via ACE for HBL | platform |
| `customs_cleared` | Entry cleared, duties assessed | customs |
| `deconsolidated` | Cargo deconsolidated from container at CFS | carrier |
| `released` | HBL released | carrier |
| `delivered` | HBL delivered, POD obtained | carrier |

#### Ground/Truck Events (9 types)
| Event Type | Description | Actor |
|------------|-------------|-------|
| `bol_issued` | Pro# — BOL issued | shipper |
| `paps_filed` | PAPS pre-arrival filed | platform |
| `cargo_picked_up` | Cargo picked up at origin | carrier |
| `border_arrived` | Truck arrived at border crossing | carrier |
| `entry_filed` | Entry filed with CBP | platform |
| `customs_cleared` | Entry cleared at border | customs |
| `released` | Pro# released | carrier |
| `delivered` | Pro# delivered, POD obtained | carrier |

**Architectural implication**: Events carry mode-specific document references (`house_number`, `master_number`, `container_number`, `seal_number`, `uld_id`, `pro_number`, `flight_number`, `vessel`, `voyage`). The event schema must accommodate **mode-polymorphic payloads**. A flat, generic event schema would destroy this richness.

### A.4 Intermediate Transit Events (12 Types)

Generated at regulatory touchpoints along multi-modal routes. These represent **mid-transit regulatory interactions** — not just origin/destination clearance.

| Event Type | Description | Trigger |
|------------|-------------|---------|
| `export_clearance` | Export clearance obtained at origin | export touchpoint |
| `ics2_filed` | ICS2/ENS filed for EU transit | EU transit touchpoint |
| `placi_filed` | PLACI pre-loading advance data | EU transit touchpoint |
| `tradenet_filed` | TradeNet transit declaration (Singapore) | SG transit touchpoint |
| `transit_declaration_filed` | Generic transit declaration | transit touchpoint |
| `transit_screening` | Sanctions/security screening at transit territory | transit with sanctions risk |
| `transit_cleared` | Transit clearance granted | transit touchpoint (no hold) |
| `transit_held` | Transit hold — sanctions/dual-use/inspection | transit touchpoint (hold triggered) |
| `hub_sort` | Cargo sorted at integrator hub (air only) | air transit touchpoint |
| `transshipment_arrived` | Arrived at transshipment port (ocean only) | ocean transit touchpoint |
| `transshipment_departed` | Departed transshipment port (ocean only) | ocean transit touchpoint |
| `acas_filed` | ACAS pre-arrival filing for US import | US import touchpoint |

**Transit hold probabilities**:
- EU sanctions hold: 1.5% for EU transit
- EU dual-use controls: 0.5% for EU transit
- Transshipment port inspection: 1% for SG, KR, LK transit

**Architectural implication**: Intermediate events are stored in a **separate JSONB column** (`intermediate_events`) from the main event log (`events`). They have additional fields: `territory`, `triggers_hold`, `hold_reason`. Transit holds can **block final arrival** and require separate resolution (`transit_hold_escalated` event). The event architecture must model intermediate events as a **distinct event stream** with territory context.

### A.5 Cage Management State

Shipments that enter `held` or `inspection` status get cage management tracking — a rich sub-lifecycle:

#### Cage Status Fields (JSONB `cage_status` column)
| Field | Description |
|-------|-------------|
| `facility_type` | "CES" (Centralized Exam Station), "CFS" (Container Freight Station), "carrier_warehouse" |
| `facility_name` | Specific facility (e.g., "FedEx CES Long Beach") |
| `cage_location` | Physical location within facility (Bay A-1, Section C-2, etc.) |
| `intake_time` | When cargo was received at facility |
| `dwell_days` | Current dwell time (continuously updated) |
| `storage_cost_accrued` | Running storage cost ($25-75/day depending on facility type) |
| `storage_rate_per_day` | Rate for this facility type |
| `go_deadline` | General Order deadline (15 days from intake) |
| `go_days_remaining` | Countdown to GO deadline (continuously updated) |
| `exam_type` | If inspection: "VACIS", "Tailgate", "Intensive", "Devanning" |
| `exam_description` | Human-readable exam description |
| `exam_scheduled` | When exam is scheduled |
| `exam_duration_hours` | Expected exam duration |

#### Cage Events
| Event Type | Description |
|------------|-------------|
| `cage_intake` | Cargo placed in facility with location and GO deadline |
| `go_warning` | General Order deadline warning (at 5, 3, 1 day thresholds) |
| `cage_released` | Cargo released with dwell time and total storage charges |

#### Cage Lifecycle Phases
1. **Intake**: `held`/`inspection` shipments without `cage_status` → assign facility, location, set GO deadline, schedule exam if inspection
2. **Dwell**: Update dwell_days, storage_cost, go_days_remaining continuously; emit `go_warning` at thresholds; publish `go_deadline_warning` to event bus
3. **Release**: `cleared`/`delivered` shipments with `cage_status` → log release event with dwell summary, clear `cage_status`

**Architectural implication**: Cage management is a **stateful sub-domain** that runs in parallel with the main shipment lifecycle. It has its own timer-based logic (dwell accrual, GO countdown), its own events (intake, warning, release), and its own financial implications (storage costs feed into Financial Settlement). In the event model, cage should be a **separate event consumer** that maintains its own projection of facility state, triggered by `ShipmentHeld` and `ShipmentCleared` events.

### A.6 Multi-Modal Routing Model

Routes are **structured as ordered legs** with regulatory touchpoints:

#### RouteLeg Structure
```
origin: str           # "Shanghai, CN"
destination: str      # "Paris CDG, FR"
mode: str             # "ground", "air", "ocean", "rail"
carrier_segment: str  # "FedEx", "local_trucking"
regulatory_zone: str  # "CN", "EU", "US", "SG"
clearance_type: str   # "export", "transit", "import", None
is_international: bool
```

#### RegulatoryTouchpoint Structure
```
territory: str        # "EU", "US", "SG"
location: str         # "Paris CDG, FR"
touchpoint_type: str  # "transit", "import", "export"
filings: list[str]    # ["ICS2/ENS", "PLACI"]
risks: list[str]      # ["sanctions_screening", "dual_use_controls"]
status: str           # "pending", "cleared", "held"
```

#### Integrator Hub Network
| Carrier | Hubs |
|---------|------|
| FedEx | Paris CDG (EU), Memphis (US), Guangzhou (CN) |
| DHL Express | Leipzig (EU), Cincinnati (US), Hong Kong (HK) |
| UPS | Cologne (EU), Louisville (US), Shenzhen (CN) |

#### Ocean Transshipment Ports
| Port | Zone | Origin Countries |
|------|------|-----------------|
| Singapore | SG | VN, TH, IN |
| Busan | KR | CN, JP, KR |
| Colombo | LK | IN |
| Algeciras | EU | IT |

**Architectural implication**: A single shipment generates a **multi-leg route** with regulatory touchpoints at each territory boundary. The carrier actor processes legs sequentially, generating intermediate events at each touchpoint. In the event model, each leg completion could be a distinct event (`LegCompleted`), and regulatory touchpoints could drive a **sub-workflow** at each transit territory. The routing model is stored as JSONB on the shipment — in the event architecture, it should be a **route projection** maintained by the Transport bounded context.

### A.7 Event Expressiveness Budget

The current system records **~15-25 events per typical shipment** (air, single transit hub, no holds) and **40-60+ events for complex shipments** (ocean, transshipment, transit hold, cage exam, CBP hold with CF-28/CF-29).

The event-driven architecture must preserve this resolution. Specifically:

1. **Status transition events** must carry jurisdiction context (US CF-28 ≠ EU under_control)
2. **Mode-specific operational events** must carry the right document references per mode
3. **Intermediate transit events** must carry territory and filing/risk metadata
4. **Cage events** must carry facility, location, financial, and deadline data
5. **Consolidation events** must fan out to member shipments
6. **Event ordering** must be preserved within a shipment's event stream

**The event catalog must have AT LEAST 50 distinct event types** to match current expressiveness. Fewer would mean losing tracking granularity.
