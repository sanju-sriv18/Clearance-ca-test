# Architecture Overview: Event-Driven Clearance Platform

**Status:** Reference document
**Date:** 2026-02-08
**Source documents:** 5 detailed architecture docs (see Section 13)

---

## 1. Architecture Vision

The Clearance Vibe platform migrates from a poll-and-lock model (19 actors competing for PostgreSQL connections via `SELECT ... FOR UPDATE SKIP LOCKED`) to an **event-driven architecture** using Redis Streams and the state-transfer pattern. Core principles: (1) **Events are the source of truth** -- Redis Streams hold the authoritative record; PostgreSQL is a projection. (2) **Single-writer aggregator** -- only the Shipment Aggregator writes to the `shipments` table, eliminating all concurrency bugs by construction. (3) **Proactive evaluation** -- services act at the earliest moment their data prerequisites are met, not at sequential status gates; the broker pre-files declarations while cargo is still in transit. (4) **Contention prevention by construction** -- new actors subscribe to events; no `FOR UPDATE`, no SKIP LOCKED, no contention analysis required.

---

## 2. System Architecture Diagram

```
                              +---------------------------+
                              |    Frontend (Vite/React)   |
                              |  Broker / Shipper / Platform|
                              +------------+--------------+
                                           |
                                      SSE / REST
                                           |
                              +------------+--------------+
                              |    FastAPI (API Layer)     |
                              |  Reads from Redis stores   |
                              |  (zero SQL for dashboards) |
                              +------+-------+------+-----+
                                     |       |      |
                          +----------+  +----+----+ +----------+
                          | Redis    |  | Redis   | | PostgreSQL|
                          | Read     |  | Streams | | (projection|
                          | Stores   |  | (source | |  of event |
                          | (hashes, |  | of truth| |  stream)  |
                          |  sorted  |  |  6 strs)| |           |
                          |  sets)   |  |         | |           |
                          +----+-----+  +----+----+ +-----+-----+
                               ^             |            ^
                               |             |            |
                  +------------+      +------+------+     |
                  |                    |             |     |
          +-------+--------+   +------+------+ +----+-----+-----+
          | Dashboard      |   |  19 Actors  | | Shipment       |
          | Projector      |   |  (event-    | | Aggregator     |
          | (incremental   |   |   driven)   | | (single writer |
          |  counters)     |   |             | |  to shipments  |
          +----------------+   +------+------+ |  table)        |
                                      |        +----------------+
                                      |              ^
                                      |  publish     | consume ALL
                                      +---events---->+ 6 streams
                                           to
                                      Redis Streams

  Redis Streams (6):
  +-- stream:shipment-lifecycle      (5 lifecycle events)
  +-- stream:customs-processing      (~35 jurisdiction-specific events)
  +-- stream:transport-ops           (35 mode-specific events -- highest volume)
  +-- stream:transit-events          (13 intermediate regulatory events)
  +-- stream:exception-resolution    (5 hold/cage/escalation events)
  +-- stream:compliance              (8 analysis/screening events)
```

---

## 3. Bounded Contexts

| # | Context | Responsibility | Key Services / Actors | Owned DB Tables | Key Events Produced |
|---|---------|---------------|----------------------|----------------|-------------------|
| 1 | **Product Catalog** | Master catalog of products with trade-relevant attributes | None (CRUD via API) | `products` | `ProductCreated`, `ProductUpdated` |
| 2 | **Trade Intelligence** | HS code classification, duty/tariff calculation across 6 jurisdictions, FTA eligibility | `e1_classification`, `e2_tariff` (6 regime engines), `e4_fta`, `preclearance` actor | `htsus_chapters`, `htsus_headings`, `cross_rulings`, `section_301_lists`, `section_232_scopes`, `ieepa_rates`, `adcvd_orders`, `tax_rates`, `fee_schedules`, `exchange_rates`, `tax_regime_templates`, `fta_rules` | `ClassificationCompleted` (via preclearance) |
| 3 | **Commercial (Orders)** | Purchase orders, line-item valuation, document requirements | `shipper` actor (simulated), pipeline orchestrator | `orders`, `order_line_items`, `documents` (shared) | `ShipmentCreatedFromOrder` |
| 4 | **Transport & Logistics** | Physical movement: shipment creation, carrier transit, multi-modal routing, consolidation, intermediate events | `shipper`, `carrier`, `consolidator`, `preclearance`, `terminal`, `disruption`, `isf` | `shipments` (lifecycle fields), `consolidations` | `ShipmentCreated`, `ShipmentInTransit`, `ShipmentArrivedAtCustoms`, `ShipmentDelivered`, all mode-specific events (35), all intermediate transit events (13), consolidation events (5) |
| 5 | **Customs & Declarations** | Government-facing customs processing: screening, entry routing, authority interaction, jurisdiction-specific workflows | `customs`, `pga`, `cbp_authority`, `cage` | `entry_filings` | All jurisdiction-specific customs events (US 11, EU 6, BR 4, IN 5, CN 5), `ShipmentHeld`, `ShipmentCleared`, `DeclarationAccepted`, `DeclarationRejected`, `ShipmentArrivedWithoutClearance` |
| 6 | **Broker Operations** | Broker workflow: claim shipments, verify docs, prepare/submit entries, respond to CF-28/CF-29, communications | `broker_sim` (decomposed into 3 handlers), `broker_intelligence` (LLM) | `brokers`, `broker_assignments`, `broker_messages`, `entry_filings` (shared) | `BrokerAssigned`, `EntryFiled`, `DeclarationSubmitted`, `CF28Responded`, `CF29Accepted`, `ProtestFiled`, `ChecklistItemVerified` |
| 7 | **Compliance & Screening** | Screen against denied party lists, UFLPA, PGA requirements, restricted materials | `compliance_engine`, `pga`, `AnalysisWatchdog` | `restricted_parties`, `pga_mappings` | `AnalysisGenerated`, `PGAReviewComplete`, `ISFValidated` |
| 8 | **Exception Management** | Handle holds, exceptions, disruptions, resolutions, shipper notifications | `exceptions`, `resolution`, `shipper_response`, `disruption`, `terminal` | None exclusively (operates on shipment event data) | `ExceptionInjected`, `DisruptionGenerated`, `HoldResolved`, `TransitHoldResolved`, `HoldEscalated`, `BerthDelayReported` |
| 9 | **Financial Settlement** | Duties, tariffs, fees, bonds, demurrage, storage costs, cage dwell charges | `financial`, `demurrage`, `cage` (cost tracking) | Operates on `shipments.financials`, `shipments.cage_status` | `FeesCalculated`, `CageIntake`, `CageReleased`, `GODeadlineWarning` |
| 10 | **Regulatory Intelligence** | Monitor and disseminate regulatory changes (tariff modifications, trade restrictions) | `RegulatoryMonitor` (background service), `feed_parsers` | `regulatory_signals` | `RegulatorySignalDetected` |

---

## 4. Redis Streams

| Stream | Consumer Groups | Event Types Carried | Purpose |
|--------|----------------|-------------------|---------|
| `stream:shipment-lifecycle` | `cg:customs-actor`, `cg:pga-actor`, `cg:broker-sim-actor`, `cg:compliance-actor`, `cg:consolidator-actor`, `cg:preclearance-actor`, `cg:documents-actor`, `cg:isf-actor`, `cg:terminal-actor`, `cg:carrier-actor`, `cg:cage-actor`, `cg:financial-actor`, `cg:demurrage-actor`, `cg:dashboard-projector` (14 groups) | `ShipmentCreated`, `ClassificationCompleted`, `ShipmentInTransit`, `ShipmentArrivedAtCustoms`, `ShipmentCleared`, `ShipmentDelivered` | Core shipment state machine transitions; maxlen 10,000 |
| `stream:customs-processing` | `cg:resolution-actor`, `cg:cage-actor`, `cg:carrier-actor`, `cg:broker-sim-actor`, `cg:shipper-response`, `cg:cbp-authority-actor`, `cg:financial-actor`, `cg:dashboard-projector` (8 groups) | All jurisdiction-specific customs events (US 11, EU 6, BR 4, IN 5, CN 5) plus cross-jurisdiction shared events (`ShipmentHeld`, `HoldResolved`, `BrokerAssigned`, `CommunicationSent`, `DeclarationSubmitted`, `DeclarationAccepted`, `DeclarationRejected`, `ShipmentArrivedWithoutClearance`) ~35 types total | Jurisdiction-specific customs authority processing; maxlen 10,000 |
| `stream:transport-ops` | `cg:customs-actor`, `cg:financial-actor`, `cg:dashboard-projector` (3 groups) | Air (13): `HAWBIssued`, `ACASFiled`, `CargoTendered`, `MAWBConsolidated`, `ULDBuildUp`, `FlightDeparted`, `FlightArrived`, `ULDBreakdown`, `AirEntryFiled`, `AirCustomsCleared`, `HAWBReleased`, `AirDelivered`. Ocean (13): `HBLIssued`, `ISFFiled`, `CargoReceivedCFS`, `ContainerStuffed`, `VesselDeparted`, `VesselArrived`, `ContainerDischarged`, `OceanEntryFiled`, `OceanCustomsCleared`, `Deconsolidated`, `HBLReleased`, `OceanDelivered`. Ground (9): `BOLIssued`, `PAPSFiled`, `CargoPickedUp`, `BorderArrived`, `GroundEntryFiled`, `GroundCustomsCleared`, `ProReleased`, `GroundDelivered` | Mode-specific operational detail; highest volume (~10 events/shipment); maxlen 30,000 |
| `stream:transit-events` | `cg:compliance-actor`, `cg:resolution-actor`, `cg:carrier-actor`, `cg:dashboard-projector` (4 groups) | `ExportClearanceObtained`, `ICS2Filed`, `PLACIFiled`, `TradeNetFiled`, `TransitDeclarationFiled`, `ACASFiled`, `TransitScreening`, `TransitHeld`, `TransitCleared`, `TransitHoldResolved`, `HubSort`, `TransshipmentArrived`, `TransshipmentDeparted` (13 types) | Intermediate regulatory touchpoints during transit; maxlen 20,000 |
| `stream:exception-resolution` | `cg:carrier-actor`, `cg:cage-actor`, `cg:financial-actor`, `cg:shipper-response`, `cg:dashboard-projector` (5 groups) | `HoldResolved`, `TransitHoldResolved`, `HoldEscalated`, `CageIntake`, `CageExamScheduled`, `CageReleased`, `GODeadlineWarning` | Hold resolution, cage lifecycle, escalation; maxlen 5,000 |
| `stream:compliance` | `cg:dashboard-projector` (1 group) | `AnalysisGenerated`, `PGAReviewComplete`, `ISFValidated`, `FeesCalculated`, `DocumentDiscrepancyFound`, `BerthDelayReported`, `ExceptionInjected`, `DisruptionGenerated` (8 types) | Compliance screening results and enrichment events; maxlen 5,000 |

**Total: 6 streams, 35 consumer groups, ~106 event types**

---

## 5. Complete Event Catalog

### 5.1 Shipment Lifecycle Events (stream:shipment-lifecycle) -- 6 events

| Event | Producer | Consumers | Description |
|-------|----------|-----------|-------------|
| `ShipmentCreated` | shipper | compliance, preclearance, documents, isf, broker_sim, financial, exceptions, dashboard-projector | New shipment created -- all proactive evaluators begin readiness evaluation |
| `ClassificationCompleted` | preclearance | broker_sim, pga, financial, documents, dashboard-projector | HS code and risk flags determined -- unblocks declaration filing |
| `ShipmentInTransit` | preclearance, carrier | carrier, consolidator, exceptions, disruption, dashboard-projector | Shipment has entered transit with carrier assignment |
| `ShipmentArrivedAtCustoms` | carrier | broker_sim, customs_authority, cage, terminal, exceptions, dashboard-projector | Physical arrival at port of entry |
| `ShipmentCleared` | customs_authority, cbp_authority, resolution | carrier, cage, financial, dashboard-projector | Cleared for delivery (via STP, manual, or resolution) |
| `ShipmentDelivered` | carrier | financial, dashboard-projector | Final delivery confirmed with POD |

### 5.2 Broker-Customs Interaction Events (stream:customs-processing) -- 4 cross-jurisdiction events

| Event | Producer | Consumers | Description |
|-------|----------|-----------|-------------|
| `DeclarationSubmitted` | broker (declaration handler) | customs_authority, dashboard-projector | Broker pre-filed declaration for adjudication |
| `DeclarationAccepted` | customs_authority | broker, carrier, cage, financial, dashboard-projector | Declaration accepted -- clearance granted |
| `DeclarationRejected` | customs_authority | broker, dashboard-projector | Declaration rejected -- corrections required |
| `ShipmentArrivedWithoutClearance` | carrier | customs_authority, broker, cage, dashboard-projector | Exception: shipment arrived with no pre-filed declaration |

### 5.3 Cross-Jurisdiction Shared Events (stream:customs-processing) -- 4 events

| Event | Producer | Consumers | Description |
|-------|----------|-----------|-------------|
| `ShipmentHeld` | customs_authority, pga, cbp_authority | resolution, cage, broker_sim, shipper_response, dashboard-projector | Shipment placed on hold with hold_type and reason |
| `HoldResolved` | resolution | carrier, cage, dashboard-projector | Hold resolved -- shipment released |
| `BrokerAssigned` | broker_sim | dashboard-projector | Broker assignment recorded (no status change) |
| `CommunicationSent` | customs_authority, cbp_authority, broker_sim | shipper_response, dashboard-projector | Communication sent requiring response |

### 5.4 US -- CBP Entry Lifecycle Events (stream:customs-processing) -- 11 events

| Event | Producer | Consumers | Description |
|-------|----------|-----------|-------------|
| `ShipmentInspection` | customs | cage, dashboard-projector | Shipment selected for inspection |
| `InspectionCompleted` | customs | carrier/resolution, cage, dashboard-projector | Inspection outcome (cleared or held) |
| `EntryFiled` | broker_sim | cbp_authority, dashboard-projector | Entry filing submitted to CBP |
| `CF28Issued` | cbp_authority | broker_sim, shipper_response, dashboard-projector | CF-28 information request issued |
| `CF28Responded` | broker_sim | cbp_authority, dashboard-projector | Broker responded to CF-28 |
| `CF29Issued` | cbp_authority | broker_sim, dashboard-projector | CF-29 proposed action issued |
| `CF29Accepted` | broker_sim | carrier, cage, dashboard-projector | Broker accepted CF-29 |
| `ProtestFiled` | broker_sim | cbp_authority, dashboard-projector | Protest filed (19 USC 1514) |
| `ProtestResolved` | cbp_authority | carrier/resolution, cage, dashboard-projector | Protest adjudicated |
| `ExamScheduled` | cbp_authority | cage, dashboard-projector | Exam scheduled (VACIS/tailgate/intensive) |
| `EntryAccepted` | cbp_authority | carrier, cage, dashboard-projector | Entry accepted and released |

### 5.5 EU -- UCC Customs Lifecycle Events (stream:customs-processing) -- 6 events

| Event | Producer | Consumers | Description |
|-------|----------|-----------|-------------|
| `DeclarationLodged` | broker | customs, dashboard-projector | Declaration lodged with MRN |
| `DeclarationAccepted` | customs | broker_sim, dashboard-projector | Declaration accepted by customs |
| `DeclarationRejected` | customs | broker_sim, dashboard-projector | Declaration rejected with reason code |
| `UnderControl` | customs | cage, broker_sim, dashboard-projector | Documentary or physical control initiated |
| `CustomsReleased` | customs | carrier, cage, dashboard-projector | Released from customs control |
| `ReleasedToCleared` | customs | carrier, dashboard-projector | Final clearance granted |

### 5.6 BR -- Siscomex Lifecycle Events (stream:customs-processing) -- 4 events

| Event | Producer | Consumers | Description |
|-------|----------|-----------|-------------|
| `DIRegistered` | broker | customs, dashboard-projector | DI registered in Siscomex |
| `Parameterized` | customs | broker_sim, cage, dashboard-projector | Parametrization channel assigned (green/yellow/red/grey) |
| `ParameterizationCleared` | customs | carrier, cage, dashboard-projector | Channel processing cleared |
| `DIRejected` | customs | broker_sim, dashboard-projector | DI registration rejected |

### 5.7 IN -- ICEGATE Lifecycle Events (stream:customs-processing) -- 5 events

| Event | Producer | Consumers | Description |
|-------|----------|-----------|-------------|
| `BillOfEntryFiled` | broker | customs, dashboard-projector | BOE filed via ICEGATE |
| `DutyAssessed` | customs | broker_sim, financial, dashboard-projector | Duty assessment with first/second check |
| `OutOfCharge` | customs | carrier, cage, dashboard-projector | Out of Charge order issued |
| `OOCToCleared` | customs | carrier, dashboard-projector | Final clearance after OOC |
| `BOERejected` | customs | broker_sim, dashboard-projector | Bill of Entry rejected |

### 5.8 CN -- GACC Single Window Lifecycle Events (stream:customs-processing) -- 5 events

| Event | Producer | Consumers | Description |
|-------|----------|-----------|-------------|
| `CustomsDeclared` | broker | customs, dashboard-projector | Declaration submitted via Single Window |
| `CIQInspected` | customs | cage, broker_sim, dashboard-projector | CIQ quarantine/quality inspection |
| `GACCReleased` | customs | carrier, cage, dashboard-projector | GACC release with duty payment |
| `ReleasedToCleared_CN` | customs | carrier, dashboard-projector | Final clearance |
| `DeclarationRejected_CN` | customs | broker_sim, dashboard-projector | Declaration rejected |

### 5.9 Air Freight Events (stream:transport-ops) -- 13 events

| Event | Producer | Consumers | Description |
|-------|----------|-----------|-------------|
| `HAWBIssued` | shipper | carrier, dashboard-projector | House Air Waybill booked |
| `ACASFiled` | platform | compliance, dashboard-projector | ACAS advance filing submitted |
| `CargoTendered` | carrier | dashboard-projector | Cargo tendered to carrier (pcs, weight) |
| `MAWBConsolidated` | carrier | dashboard-projector | HAWB consolidated into Master AWB |
| `ULDBuildUp` | carrier | dashboard-projector | Cargo loaded into ULD |
| `FlightDeparted` | carrier | dashboard-projector | Flight departed origin |
| `FlightArrived` | carrier | dashboard-projector | Flight arrived at destination |
| `ULDBreakdown` | carrier | dashboard-projector | ULD breakdown at destination terminal |
| `AirEntryFiled` | platform | customs, dashboard-projector | Entry filed via ACE for HAWB |
| `AirCustomsCleared` | customs | carrier, financial, dashboard-projector | Entry cleared, duties assessed |
| `HAWBReleased` | carrier | dashboard-projector | HAWB released |
| `AirDelivered` | carrier | financial, dashboard-projector | HAWB delivered with POD |

### 5.10 Ocean Freight Events (stream:transport-ops) -- 13 events

| Event | Producer | Consumers | Description |
|-------|----------|-----------|-------------|
| `HBLIssued` | shipper | carrier, dashboard-projector | House Bill of Lading booked |
| `ISFFiled` | platform | compliance, dashboard-projector | ISF filed (24hr pre-load requirement) |
| `CargoReceivedCFS` | carrier | dashboard-projector | Cargo received at origin CFS |
| `ContainerStuffed` | carrier | dashboard-projector | Container stuffed, seal applied, MBL linked |
| `VesselDeparted` | carrier | dashboard-projector | Vessel departed origin port |
| `VesselArrived` | carrier | dashboard-projector | Vessel arrived at destination port |
| `ContainerDischarged` | carrier | dashboard-projector | Container discharged from vessel |
| `OceanEntryFiled` | platform | customs, dashboard-projector | Entry filed via ACE for HBL |
| `OceanCustomsCleared` | customs | carrier, financial, dashboard-projector | Entry cleared, duties assessed |
| `Deconsolidated` | carrier | dashboard-projector | Cargo deconsolidated from container |
| `HBLReleased` | carrier | dashboard-projector | HBL released |
| `OceanDelivered` | carrier | financial, dashboard-projector | HBL delivered with POD |

### 5.11 Ground/Truck Events (stream:transport-ops) -- 9 events

| Event | Producer | Consumers | Description |
|-------|----------|-----------|-------------|
| `BOLIssued` | shipper | carrier, dashboard-projector | Pro# and BOL issued |
| `PAPSFiled` | platform | compliance, dashboard-projector | PAPS pre-arrival filed |
| `CargoPickedUp` | carrier | dashboard-projector | Cargo picked up at origin |
| `BorderArrived` | carrier | dashboard-projector | Truck arrived at border crossing |
| `GroundEntryFiled` | platform | customs, dashboard-projector | Entry filed with CBP at border |
| `GroundCustomsCleared` | customs | carrier, financial, dashboard-projector | Entry cleared at border |
| `ProReleased` | carrier | dashboard-projector | Pro# released |
| `GroundDelivered` | carrier | financial, dashboard-projector | Pro# delivered with POD |

### 5.12 Intermediate Transit Events (stream:transit-events) -- 13 events

| Event | Producer | Consumers | Description |
|-------|----------|-----------|-------------|
| `ExportClearanceObtained` | customs | dashboard-projector | Export clearance at origin country |
| `ICS2Filed` | platform | compliance, dashboard-projector | ICS2/ENS filed for EU transit territory |
| `PLACIFiled` | platform | dashboard-projector | PLACI pre-loading data filed for EU |
| `TradeNetFiled` | platform | dashboard-projector | TradeNet declaration filed (Singapore) |
| `TransitDeclarationFiled` | platform | dashboard-projector | Generic transit declaration at any territory |
| `ACASFiled` | platform | compliance, dashboard-projector | ACAS pre-arrival filing for US import |
| `TransitScreening` | customs | compliance, dashboard-projector | Sanctions/security screening at transit territory |
| `TransitHeld` | customs | resolution, dashboard-projector | Transit hold (sanctions, dual-use, port inspection) |
| `TransitCleared` | customs | dashboard-projector | Transit clearance granted |
| `TransitHoldResolved` | resolution | carrier, dashboard-projector | Transit hold resolved or escalated |
| `HubSort` | carrier | dashboard-projector | Cargo sorted at integrator hub (air only) |
| `TransshipmentArrived` | carrier | dashboard-projector | Arrived at transshipment port (ocean only) |
| `TransshipmentDeparted` | carrier | dashboard-projector | Departed transshipment port (ocean only) |

### 5.13 Exception, Cage & Resolution Events (stream:exception-resolution) -- 5 events

| Event | Producer | Consumers | Description |
|-------|----------|-----------|-------------|
| `HoldEscalated` | resolution | shipper_response, dashboard-projector | Hold escalated after review period |
| `CageIntake` | cage | demurrage, dashboard-projector | Cargo placed in exam facility with location and GO deadline |
| `CageExamScheduled` | cage | broker_sim, dashboard-projector | Exam type and time scheduled |
| `CageReleased` | cage | financial, dashboard-projector | Cargo released with dwell/cost summary |
| `GODeadlineWarning` | cage | broker_sim, dashboard-projector | General Order deadline warning (5, 3, 1 day) |

### 5.14 Compliance & Enrichment Events (stream:compliance) -- 8 events

| Event | Producer | Consumers | Description |
|-------|----------|-----------|-------------|
| `AnalysisGenerated` | compliance_engine | dashboard-projector | Full compliance analysis completed |
| `PGAReviewComplete` | pga | dashboard-projector | PGA agency review outcome |
| `ISFValidated` | isf | dashboard-projector | ISF filing status validated |
| `FeesCalculated` | financial | dashboard-projector | Duties, fees, and financial breakdown computed |
| `DocumentDiscrepancyFound` | documents | dashboard-projector | Document validation discrepancy detected |
| `BerthDelayReported` | terminal | dashboard-projector | Berth delay at port terminal |
| `ExceptionInjected` | exceptions | dashboard-projector | Exception generated on shipment |
| `DisruptionGenerated` | disruption | dashboard-projector | Supply chain disruption affecting port/shipments |

### 5.15 Consolidation Lifecycle Events (stream:shipment-lifecycle) -- 5 events

| Event | Producer | Consumers | Description |
|-------|----------|-----------|-------------|
| `ConsolidationClosed` | consolidator | carrier, dashboard-projector | Consolidation group closed with member shipments |
| `ConsolidationInTransit` | carrier | dashboard-projector | Consolidation entered transit |
| `ConsolidationArrived` | carrier | dashboard-projector | Consolidation arrived at destination |
| `ConsolidationDeconsolidating` | carrier | dashboard-projector | Deconsolidation in progress |
| `ConsolidationDeconsolidated` | carrier | dashboard-projector | Deconsolidation complete |

---

## 6. Service/Actor Registry

### 6.1 Proactive Evaluators

| Actor | Readiness Criteria | Events Consumed | Events Produced | Earliest Action |
|-------|-------------------|----------------|----------------|----------------|
| `broker_sim` (declaration handler) | HS code + origin + destination + declared_value + company_name | `ShipmentCreated`, `ClassificationCompleted` | `DeclarationSubmitted` | Pre-files declaration while shipment is `in_transit` |
| `broker_sim` (claim handler) | Product + origin + destination + value + complexity | `ShipmentCreated`, `ClassificationCompleted` | `BrokerAssigned`, `EntryDraftCreated` | Assigns broker and begins prep during `booked`/`in_transit` |
| `pga` | HS code + product category mapped to PGA agency | `ClassificationCompleted`, `ShipmentArrivedAtCustoms` | `PGAReviewComplete`, `ShipmentHeld` | Determines PGA requirements as soon as HS code known |
| `compliance_engine` | Product + shipper entity info | `ShipmentCreated`, `ClassificationCompleted`, `RegulatorySignalDetected` | `AnalysisGenerated` | Re-screens on regulatory changes, not just at creation |
| `financial` | Tariff result + declared_value | `ClassificationCompleted`, `ShipmentCleared`, `CageReleased`, `DutyAssessed` (IN) | `FeesCalculated` | Pre-calculates fees as soon as tariff data available |
| `documents` | At least one document exists for shipment | `ShipmentCreated`, `DocumentUploaded`, `ClassificationCompleted` | `DocumentDiscrepancyFound` | Validates documents as soon as uploaded (pre-departure) |
| `isf` | Ocean shipment + carrier + HS code + company | `ShipmentCreated` | `ISFValidated` | Files ISF pre-departure for ocean shipments |
| `preclearance` | Shipment exists with product info | `ShipmentCreated` | `ClassificationCompleted`, `ShipmentInTransit`, `ShipmentHeld` | Runs immediately after creation |

### 6.2 Reactive Responders

| Actor | Trigger Events | Events Consumed | Events Produced | Why Reactive |
|-------|---------------|----------------|----------------|-------------|
| `customs_authority` | `DeclarationSubmitted`, `ShipmentArrivedWithoutClearance` | stream:customs-processing | `DeclarationAccepted`, `DeclarationRejected`, `ShipmentCleared`, `ShipmentHeld`, jurisdiction-specific events | Government authority adjudicates submitted declarations -- does not seek work |
| `cbp_authority` | `EntryFiled`, `CF28Responded`, `ProtestFiled` | stream:customs-processing | `EntryAccepted`, `CF28Issued`, `CF29Issued`, `ExamScheduled`, `ProtestResolved` | Government authority responds to broker submissions |
| `cage` (intake) | `ShipmentHeld`, `ShipmentInspection` | stream:customs-processing | `CageIntake`, `CageExamScheduled` | Physically-triggered -- cargo must be held first |
| `terminal` | `ShipmentArrivedAtCustoms` | stream:shipment-lifecycle | `BerthDelayReported` | Terminal ops require physical arrival |
| `shipper_response` | `CommunicationSent`, `CF28Issued`, `HoldEscalated` | stream:customs-processing, stream:exception-resolution | Shipper response events | Responds to communications directed at shipper |
| `consolidator` | `ShipmentInTransit` (buffered consumer) | stream:shipment-lifecycle | `ConsolidationClosed` | Groups shipments already in transit; accumulates then flushes |

### 6.3 Hybrid Actors (Event-Triggered Start + Sim-Clock Polling)

| Actor | Event Trigger | Time-Dependent Check | Events Consumed | Events Produced |
|-------|--------------|---------------------|----------------|----------------|
| `carrier` | `ShipmentInTransit` starts tracking | `clock.now() - started_at >= transit_days` | `ShipmentInTransit`, `ShipmentCleared`, `EntryAccepted`, `HoldResolved`, `TransitHoldResolved` | `ShipmentArrivedAtCustoms`, `ShipmentDelivered`, all mode-specific transport events (35), intermediate transit events (13) |
| `resolution` | `ShipmentHeld` starts tracking | `clock.now() - hold_time >= review_days` | `ShipmentHeld`, `TransitHeld` | `HoldResolved`, `TransitHoldResolved`, `HoldEscalated` |
| `cage` (dwell/release) | `CageIntake` starts tracking | Lazy computation for dwell/cost; GO deadline warnings on timer | `ShipmentCleared`, `ShipmentDelivered` | `CageReleased`, `GODeadlineWarning` |
| `demurrage` | `ShipmentArrivedAtCustoms` (ocean only) | Lazy computation -- stores arrival time + carrier policy, computes charges on demand | `ShipmentArrivedAtCustoms` | Demurrage accrual events |

### 6.4 Timer-Driven Producers

| Actor | Behavior | Events Produced |
|-------|----------|----------------|
| `shipper` | Creates shipments on Poisson timer | `ShipmentCreated` |
| `exceptions` | Probabilistic exception injection on timer; subscribes to lifecycle events for shipment discovery | `ExceptionInjected` |
| `disruption` | Timer-based port disruption generation; subscribes to lifecycle events for port-shipment mapping | `DisruptionGenerated` |

---

## 7. Infrastructure Components

| Component | Purpose |
|-----------|---------|
| **Shipment Aggregator** | Single writer to the `shipments` table. Subscribes to all 6 streams via `cg:shipment-aggregator`. Projects every event into the shipment record: status, events timeline, hold_type, entry_number, cage_status, financials, references, compliance_result, classification_result, tariff_result, fta_result. Processes ~200-500 events/second at ~5ms per single-row PK update. |
| **EventStreamBus** | Redis Streams-based event bus replacing the fire-and-forget Redis Pub/Sub EventBus. Methods: `publish()`, `read_stream()`, `ack()`, `create_groups()`, `get_pending_info()`, `trim_streams()`. Provides persistence, consumer groups, acknowledgment, replay, and backpressure. |
| **ProactiveEvaluatorMixin** | Mixin for proactive evaluator actors. Maintains an in-memory `_readiness` dict (shipment_id to accumulated state). Methods: `_accumulate()` merges event data, `_is_ready()` checks all `READINESS_FIELDS` are satisfied, `_cleanup()` removes processed shipments. Memory: ~700KB total for all proactive evaluators at peak. Rebuilt from Redis Streams PEL on restart. |
| **ReadinessAccumulator** | Per-shipment state within the `ProactiveEvaluatorMixin`. Tracks which data prerequisites have been satisfied from incoming events. Each event merges its payload fields into the accumulator. When all fields in `READINESS_FIELDS` are non-null, the actor acts immediately. |
| **DashboardProjector** | Subscribes to all 6 streams. Maintains `read:dashboard:platform` Redis hash via incremental counter updates (no SQL). Includes lazy cage/demurrage computation via `compute_cage_metrics()` pure function. Periodic 5-minute SQL reconciliation as safety net. |
| **ShipmentProjector** | Maintains `read:shipments:by_status:{status}` sorted sets and `read:shipment:{id}:summary` hashes. Enables zero-SQL shipment list and broker queue endpoints. |
| **ProcessingOutcome** | Dataclass returned by pure business logic functions. Fields: `new_status`, `hold_type`, `description`, `event_type`, `stream`, `references_update`, `extra_data`. Actors compute outcomes, publish as events. Aggregator projects into DB. |
| **TestEventBus** | In-memory event bus for unit/integration tests. No Redis dependency. Maintains streams, consumer positions, and published events list. Enables Layer 2 and Layer 3 testing without infrastructure. |

---

## 8. Data Flow: Happy Path (Pre-Clearance Success)

```
Shipper                Broker               Customs Auth        Carrier           Aggregator
  |                      |                      |                  |                  |
  |-- ShipmentCreated -->|                      |                  |                  |
  |  (stream:lifecycle)  |                      |                  |                  |
  |                      |                      |                  |                  |
  |              [Accumulate readiness]         |                  |                  |
  |                      |                      |                  |                  |
  |                  ClassificationCompleted     |                  |                  |
  |                  (from preclearance)         |                  |                  |
  |                      |                      |                  |                  |
  |              [Readiness MET: HS +           |                  |                  |
  |               origin + dest + value]        |                  |                  |
  |                      |                      |                  |                  |
  |                      |-- DeclarationSubmitted ------>|         |                  |
  |                      |  (stream:customs)    |        |         |                  |
  |                      |                      |        |         |                  |
  |                      |                      | [Adjudicate]     |                  |
  |                      |                      |        |         |                  |
  |                      |<-- DeclarationAccepted -------|         |                  |
  |                      |                      |                  |                  |
  |                      |                      |       (Shipment is still in_transit)|
  |                      |                      |                  |                  |
  |                      |                      |     [transit_days elapsed]          |
  |                      |                      |                  |                  |
  |                      |                      |  ShipmentCleared |                  |
  |                      |                      |  (in_transit --> cleared:           |
  |                      |                      |   pre-clearance accepted,           |
  |                      |                      |   skip at_customs entirely)         |
  |                      |                      |                  |                  |
  |                      |                      |                  |-- ShipmentDelivered
  |                      |                      |                  |                  |
  |                      |                      |                  |          [All events
  |                      |                      |                  |           projected
  |                      |                      |                  |           into DB]
```

**Key insight:** The broker files the declaration during transit. The customs authority accepts it before physical arrival. When the carrier reports arrival, the shipment transitions directly from `in_transit` to `cleared`, skipping `at_customs` entirely.

---

## 9. Data Flow: Exception Path (Arrival Without Pre-Clearance)

```
Carrier                Customs Auth        Broker               Resolution        Cage
  |                      |                  |                      |               |
  |-- ShipmentArrivedAtCustoms ------------>|                      |               |
  |  (no declaration was pre-filed)         |                      |               |
  |                      |                  |                      |               |
  |  ShipmentArrivedWithoutClearance ------>|                      |               |
  |  (stream:customs)    |                  |                      |               |
  |                      |                  |                      |               |
  |              [Default: HOLD]            |                      |               |
  |                      |                  |                      |               |
  |              ShipmentHeld ------------->|--------------------->|-------------->|
  |              (hold_type:                |                      |               |
  |               no_preclearance)          |                      |               |
  |                      |                  |                      |               |
  |                      |          [Emergency filing]     [Start review]  [CageIntake]
  |                      |                  |                      |               |
  |                      |  EntryFiled ---->|                      |               |
  |                      |  (broker        [Adjudicate entry]     |               |
  |                      |   scrambles      |                      |               |
  |                      |   to file)       |                      |               |
  |                      |                  |              [review_days elapsed]    |
  |                      |          EntryAccepted                  |               |
  |                      |                  |              HoldResolved             |
  |                      |                  |                      |               |
  |<------------ ShipmentCleared -----------|                      |       CageReleased
  |                      |                  |                      |               |
  |-- ShipmentDelivered  |                  |                      |               |
```

**Key insight:** A shipment arriving without pre-clearance is a **failure of the proactive system**. The customs authority holds it by default. The broker must scramble to file. Resolution tracks the hold review period. This path should be rare (<5% in a healthy system).

---

## 10. State Machines

### 10.1 Shipment Lifecycle (Proactive Model)

```
                                    +------ ClassificationCompleted (while in_transit)
                                    |       Broker pre-files declaration
                                    |       Customs accepts
                                    v
booked --> in_transit ------------> cleared --> delivered
               |                      ^
               |                      |
               | (pre-clearance       | (hold resolved,
               |  incomplete or       |  entry accepted,
               |  physical exam       |  protest resolved)
               |  required)           |
               v                      |
          at_customs --> entry_filed --+
               |              |
               |              +--> cf28_pending --> entry_filed (re-submit)
               |              |
               |              +--> cf29_pending --> cleared (pay) / protest_filed
               |              |
               |              +--> exam_scheduled --> cleared / held
               |
               +--> held --------> cleared / at_customs / in_transit (resolution)
               |
               +--> inspection --> cleared / held
```

**Note:** `at_customs` is **optional** in the proactive model. The `in_transit --> cleared` transition is the happy path when pre-clearance succeeds.

### 10.2 Jurisdiction-Specific State Machines

| Jurisdiction | Unique States | State Flow |
|-------------|--------------|------------|
| **US (CBP)** | `entry_filed`, `cf28_pending`, `cf29_pending`, `exam_scheduled`, `protest_filed` (12 total) | `at_customs --> entry_filed --> {cleared, cf28_pending, cf29_pending, exam_scheduled, held}` |
| **EU (UCC)** | `lodged`, `accepted`, `under_control`, `released` (10 total) | `at_customs --> lodged --> accepted --> {under_control, released, cleared}` |
| **BR (Siscomex)** | `registered`, `parameterized` (8 total) | `at_customs --> registered --> parameterized --> cleared` (channels: green/yellow/red/grey) |
| **IN (ICEGATE)** | `filed`, `assessed`, `out_of_charge` (9 total) | `at_customs --> filed --> assessed --> out_of_charge --> cleared` |
| **CN (GACC)** | `declared`, `inspected`, `released` (9 total) | `at_customs --> declared --> {inspected, released} --> cleared` |
| **UK** | Uses EU transition graph with `jurisdiction: "UK"` | Same as EU; UK-specific tariff rates handled by Trade Intelligence |

### 10.3 Consolidation State Machine

```
booked --> closed --> in_transit --> arrived --> deconsolidating --> deconsolidated
```

---

## 11. Migration Phases (Summary)

| Phase | Duration | Key Deliverables | Files Changed |
|-------|----------|-----------------|---------------|
| **Phase 0: Infrastructure** | Days 1-2 | Separate connection pools, `EventStreamBus` (6 streams), `EventDrivenActor` base class, `ProactiveEvaluatorMixin`, `ShipmentEventPayload`, `ProcessingOutcome`, `ClassificationCompleted` event, `in_transit --> cleared` state machine transition, transitional JSONB atomic append | 11 (8 modified + 3 new) |
| **Phase 1: Highest-Value Actors** | Days 3-6 | 11 actors migrated: customs (pilot, proactive evaluator), compliance_engine, shipper_response, financial, documents (Phase 1a); pga, broker_sim claim handler, resolution, cage, terminal, preclearance, isf (Phase 1b). Shipper event publishing added. Shadow mode validation for customs. | 13 modified |
| **Phase 2: Complex Actors + Aggregator** | Days 7-10 | broker_sim decomposition into 3 handlers, cbp_authority (reactive responder), carrier (hybrid with mode-specific event publishing), consolidator (buffered consumer). `transition()` split into `validate_transition()`. Shipment Aggregator deployed (sole writer). DashboardProjector replaces 12-15 SQL queries with Redis reads. | 8 (6 modified + 2 new) |
| **Phase 3: Cleanup + Cutover** | Days 11-12 | demurrage (lazy computation), exceptions + disruption (hybrid reclassification). Aggregator becomes sole writer -- remove all `transition()` event appends and atomic JSONB code. Decommission old `EventBus`. `TestEventBus` for tests. | 6 modified |

**Total: 11-12 days, 38 files (33 modified + 5 new), 97-157 new tests**

---

## 12. Key Design Decisions

1. **Redis Streams over Pub/Sub** -- current EventBus has 38 publishes and zero subscribers; Streams add persistence, consumer groups, acknowledgment, and replay.
2. **State-transfer pattern with bounded payloads** -- events carry ~1-2KB `ShipmentEventPayload` (sufficient for consumer decisions); excludes large fields (events history, analysis).
3. **Single-writer Shipment Aggregator** -- eliminates JSONB race conditions, multi-writer contention, and "who last touched this record?" debugging mysteries by construction.
4. **Proactive evaluation over sequential processing** -- services evaluate data prerequisites (readiness accumulator), not status gates; broker pre-files during transit.
5. **`in_transit --> cleared` state transition** -- `at_customs` becomes optional; pre-clearable shipments skip it entirely; mirrors real-world pre-clearance operations.
6. **Two event semantics: information signals vs commands** -- proactive evaluators consume information signals ("new data available, re-evaluate readiness"); reactive responders consume commands ("a declaration was submitted, adjudicate it").
7. **Hybrid actors for time-dependent logic** -- event-triggered start + sim-clock polling for completion; respects dynamic time ratio and pause behavior; no real-time sorted set timers.
8. **Lazy computation for cage dwell and demurrage** -- store parameters at event time (intake_time, storage_rate), compute current values on demand; eliminates per-tick updates.
9. **Persisted random durations** -- carrier transit_days, resolution review_days computed once on first evaluation and stored in `references` JSONB; prevents non-deterministic re-rolls.
10. **No outbox pattern** -- single-process, single-instance; event publish after commit; periodic 5-minute reconciliation as safety net.
11. **Jurisdiction-specific first-class events** -- no generic `CustomsProcessed`; each jurisdiction's intermediate states (EU `UnderControl`, BR `Parameterized`, IN `OutOfCharge`, CN `CIQInspected`) are distinct event types.
12. **Mode-specific transport events** -- `FlightDeparted` is not `VesselDeparted`; 35 mode-polymorphic events preserve air/ocean/ground operational detail.
13. **6-stream topology** -- one stream per concern (lifecycle, customs, transport-ops, transit, exception, compliance); prevents mode-specific noise from drowning lifecycle events.
14. **Dead letter after 3 retries** -- move to `{stream}:dead-letters` stream; prevents PEL accumulation from poison events; health endpoint reports dead letter counts.
15. **Broker is proactive evaluator, customs authority is reactive responder** -- the broker decides when to file; the authority adjudicates what it receives; a shipment arriving without pre-clearance is a system failure, not normal operation.

---

## 13. Document Index

| Document | Path | Description |
|----------|------|-------------|
| Event-Driven Design | `docs/architecture-event-driven-design.md` | Core architecture: 10 bounded contexts, ~106 event catalog, 6 stream topology, actor redesign patterns, Shipment Aggregator, read model projections, proactive evaluator pattern, consistency model |
| Business Capabilities | `docs/architecture-business-capabilities.md` | Business capability map: 14 capabilities across all platform surfaces, tables, API endpoints, engines, actors, and frontend screens; bounded context recommendations |
| Trade-Offs & Risks | `docs/architecture-trade-offs.md` | 7 implementation challenges analyzed, design decisions endorsed (including proactive evaluation and single-writer aggregator), risk matrix, consumer group topology assessment |
| Migration Plan | `docs/architecture-migration-plan.md` | 4-phase migration: infrastructure, highest-value actors, complex actors + aggregator, cleanup; file change matrix (38 files), testing strategy (97-157 new tests), rollback plan |
| Feasibility Check | `docs/architecture-feasibility-check.md` | Line-by-line analysis of all 19 actors: 12 clean, 5 adaptation, 2 significant rework; 4 systemic issues identified (JSONB races, time-dependent logic, multi-table atomicity, enrichment triggers) |
