# Architecture Review — Clearance Platform Microservices Design

**Reviewer:** arch-reviewer
**Document Reviewed:** `docs/architecture-clearance-platform.md` (revised)
**Date:** 2026-02-09
**Review Type:** Adversarial review against stakeholder mandates

---

## Executive Summary

The architecture document is **excellent**. It represents a genuine clean-sheet redesign organized around functional business domains, connected by NATS JetStream, with clear CQRS and simulation separation. The 11 functional domains are well-defined with proper state objects, event contracts, intelligence placement, and API surfaces.

The document satisfies all 9 stakeholder mandates. The review identifies **0 critical issues**, **3 significant concerns** requiring design clarification, and **5 minor issues** that should be addressed for completeness.

**Verdict:** Ready to serve as the definitive architecture reference. The concerns below are strengthening suggestions, not blockers.

---

## Mandate Compliance Matrix

| # | Mandate | Status | Notes |
|---|---------|--------|-------|
| 1 | NATS JetStream event bus | **PASS** | 12 JetStream streams, proper subject hierarchy, clear consumer subscriptions. |
| 2 | Microservices by functional domain with decoupled state | **PASS** | 11 domains, each with own schema, state objects, events, APIs. Modular monolith deployment is a deployment choice, not architectural compromise. |
| 3 | State-transfer events on every state change | **PASS** | Appendix A shows full state snapshot pattern. Consistent across all domains. |
| 4 | Simulation separate — uses APIs only, no clearance events | **PASS** | Section 10 is clean. `sim.*` subjects only. CustomsAuthorityBot uses platform APIs. |
| 5 | Intelligence as domains, not utilities | **PASS** | Trade Intelligence is a proper domain owning `Classification`, `TariffCalculation`, `FTADetermination` state objects. Classification Engine moved from Product Catalog to Trade Intelligence. Intelligence flows via events, not API calls. |
| 6 | CQRS: Redis reads, domain writes | **PASS** | Section 6 well-designed. Single projector, entity hashes, index sets, aggregates. All API surfaces consistently mark Read (Redis) vs Write (service → event). |
| 7 | Share-on-principle | **PASS** | Order Management is an "assembler" receiving intelligence via event subscriptions. Litmus test articulated (Section 3.2, line 213). |
| 8 | APIs as MCP tools | **PASS** | Every domain's API surface includes MCP tool column. ~60+ tools across 11 domains. |
| 9 | Actors are automation, not architecture | **PASS** | Principle 4 is clear. All actors in Simulation Platform. No actor logic in platform domains. |

---

## Key Design Decisions — Assessed

### Classification Moved to Trade Intelligence ✓

**Decision:** Classification Engine (e1) now lives in Trade Intelligence, not Product Catalog. Product Catalog owns "what the shipper says" (description, materials, weight). Trade Intelligence owns "what the system determines" (HS code, tariff, FTA).

**Assessment:** This is the correct boundary. Classification is an intelligence function that produces events consumed by multiple domains. The litmus test — "can I subscribe to `clearance.trade-intel.classification.produced`?" — passes cleanly. Product Catalog is now a simpler domain focused on product master data, which reduces its coupling.

### Customs Adjudication as Separate Domain ✓

**Decision:** Customs Adjudication is domain #7, separate from Declaration Management (#6). Declaration Management is the broker's domain; Customs Adjudication is the authority's domain.

**Assessment:** Excellent separation. This resolves what was the most subtle boundary question in the architecture. The event flow is clean:
- Broker submits → `DeclarationSubmitted` event
- Adjudication consumes → processes → emits `AdjudicationDecision`
- Declaration Management consumes `AdjudicationDecision` → updates broker workbench

This means the platform has a clean integration point for real customs authority systems — just replace the bot with real EDI/API. The `POST /adjudication/declarations/{id}/decide` API is correctly named (records authority decision).

### Trade Intelligence Owns State ✓

**Decision:** Trade Intelligence is described as a "proper domain that owns state" with persistent `Classification`, `TariffCalculation`, and `FTADetermination` state objects and 15+ owned tables.

**Assessment:** Correct. The proactive event production pattern (subscribes to `ProductCreated` → emits `ClassificationProduced`, subscribes to `ShipmentCreated` → emits `TariffCalculated`) is the right approach. Consumers subscribe to events, not call APIs. The on-demand API surface is explicitly marked "for interactive use only."

---

## Significant Concerns

### CONCERN-1: Missing Event Delivery Guarantees — Idempotency, Ordering, Dead Letters

**Location:** Section 4 (JetStream config) and Section 6 (Redis Projector)

**Gap:** The document specifies JetStream streams with max ages and storage types, but doesn't address three critical operational concerns:

1. **Idempotency**: If NATS JetStream delivers a message more than once (network retry, consumer restart), all consumers must handle duplicates. Every event handler must be idempotent. This needs to be stated as an architectural principle.

2. **Event ordering within a domain**: JetStream provides per-subject ordering, but the Redis Cache Projector subscribes to `clearance.>` (wildcard across all subjects). If the projector has multiple workers for throughput, events from different domains may interleave unpredictably. For a single entity (e.g., a shipment), events from Shipment Lifecycle and Declaration Management could arrive out of causal order.

   **Recommendation:** The projector should use `event.metadata.causation_id` and entity-level ordering (hash the entity_id to a partition/worker) to ensure causal consistency per entity.

3. **Dead letter handling**: What happens when an event fails processing after all retries? There's no DLQ or alerting mechanism specified.

   **Recommendation:** Add a `clearance.dlq.>` subject and a DLQ monitor that logs and alerts on failed events. Define the retry policy (e.g., 3 attempts with exponential backoff).

**Severity:** Significant — these are operational necessities that will bite in production. However, they don't require architectural changes, just additional specification in Sections 4 and 6.

### CONCERN-2: Modular Monolith Deployment Needs Explicit Boundary Enforcement

**Location:** Section 11.2 (Migration to Microservices)

**Current text:** "Guided by need, not dogma."

**Concern:** The modular monolith is a pragmatic deployment decision — all domain services in one FastAPI process, communicating via NATS and separate schemas. This is valid. But without explicit boundary enforcement, developers will inevitably take shortcuts:
- Importing another domain's models directly
- Querying another domain's PostgreSQL schema
- Calling another domain's function in-process instead of going via NATS

**Recommendation:** Add an enforcement section:
1. **Code structure rule:** Each domain is a top-level Python package under `app/domains/`. No domain may import from another domain's package. Lint rules should enforce this.
2. **Schema isolation rule:** Database connections are scoped to the domain's schema. Migration scripts are per-domain.
3. **Testing rule:** Each domain's unit tests must pass with its own schema only — no fixtures from other domains.
4. **The boundary test:** "Can I extract this domain into a separate process without changing any code, only changing the deployment?" If yes, the boundary is maintained.

**Severity:** Significant — without enforcement, the modular monolith will degrade into a regular monolith. The migration path (Phases 2-4) becomes impossible if boundaries aren't maintained from day one.

### CONCERN-3: Financial Settlement Domain Is Underspecified

**Location:** Section 3.9

**Concern:** Financial Settlement is the thinnest domain definition. It has 1 state object, 1 table, 3 events produced, and 7 events consumed. For a customs clearance platform, this domain should also cover:
- **Bond management**: Single-entry vs continuous bonds, bond sufficiency validation, bond holder lookups
- **Duty payment workflow**: Estimated vs liquidated duties, periodic monthly statements (PMS)
- **Drawback claims**: When goods are re-exported, duties can be recovered
- **Currency conversion**: At point of settlement, not point of calculation
- **Reconciliation**: CBP ACE reconciliation for importers with continuous bonds

**Recommendation:** Either expand the domain definition to address these concerns or add a note: "Financial Settlement will be elaborated in a dedicated sprint. The current definition covers the minimum viable financial tracking."

**Severity:** Significant for completeness, but not blocking for initial implementation.

---

## Minor Issues

### MINOR-1: Missing Readiness Concept in Broker Intelligence Flow

Section 7 (Broker Intelligence Flow) shows intelligence accumulating concurrently, but doesn't define how the UI knows when "readiness" is achieved. The broker workbench shows checkmarks, but what's the underlying data model?

**Recommendation:** Add a `readiness` field to the broker's Redis cache entry — a hash tracking which domain signals have arrived (classification: ✓, tariff: ✓, compliance: ✓, documents: pending). The Declaration Management domain should evaluate readiness on each incoming event and update the declaration's `filing_status` accordingly.

### MINOR-2: Regulatory Intelligence Dual Trigger Model Unclear

Section 3.10 says "Events Consumed: None — pure producer" but the Intelligence Trigger Matrix (Section 5.2) shows Regulatory Monitor triggered by "Periodic feed polling." The domain also clearly needs to re-evaluate impact when new shipments are created (e.g., "does this new shipment from China fall under the IEEPA increase detected yesterday?").

**Recommendation:** Clarify: Regulatory Intelligence is a **primarily autonomous** domain that produces signals on its own schedule. It does NOT consume platform events to trigger its core intelligence. However, other domains (Trade Intelligence, Compliance) consume `RegulatorySignalDetected` events and re-evaluate their own state. The "impact on existing shipments" is driven by Trade Intelligence re-computing tariffs, not by Regulatory Intelligence.

### MINOR-3: Missing Cage Management Subdomain in Exception Management

The existing codebase has extensive cage management functionality. Exception Management (Section 3.8) includes cage fields in its state object and cage events (`CageIntake`, `CageExamScheduled`, `CageReleased`, `GODeadlineWarning`), which is correct. However, the cage lifecycle (intake → storage → exam → release) could benefit from being documented as a subdomain or subprocess within Exception Management.

**Recommendation:** Add a paragraph in Section 3.8 explicitly describing the cage lifecycle as a subprocess: "Cage management is a subprocess within Exception Management, not a separate domain. A cage event always relates to an existing exception."

### MINOR-4: Appendix B Domain Interaction Matrix — Verify Completeness

The matrix shows 11 domains × 11 domains, but some interactions seem missing:
- Trade Intelligence → Compliance: Trade Intelligence produces `ClassificationProduced`, which Compliance consumes (Section 4.3 confirms). But the matrix doesn't show this.
- Document Management → Declaration Management: `DocumentValidated` events are consumed by Declaration Management (Section 3.6, line 556) but the matrix doesn't show Document Management producing for Declaration.

**Recommendation:** Audit the matrix against the consumer subscription table (Section 4.3) to ensure all event flows are represented.

### MINOR-5: Description Quality Intelligence Placement

Section 5.2 (Intelligence Trigger Matrix) lists "Description Quality" as producing "Inline result (no event)." This is the only intelligence service that doesn't produce an event. If Description Quality is important enough to be in the trigger matrix, it should produce an event (e.g., `DescriptionQualityAssessed` on `clearance.trade-intel.description-quality.assessed`).

**Recommendation:** Either add a proper event for Description Quality or remove it from the trigger matrix and treat it as a pure API-driven utility within Trade Intelligence.

---

## Cross-Reference: Codebase Audit — 11 Misplaced Items

The codebase audit (`docs/architecture-codebase-audit.md`) identified 11 pieces of business logic currently living in simulation actors that must become platform services. **Does the architecture document account for all 11?**

| # | Misplaced Logic | Audit's Target | Architecture Document Placement | Status |
|---|----------------|---------------|-------------------------------|--------|
| 1 | **Customs Processing** (STP scoring, hold determination, reclassification, inspection scheduling) | Clearance Service | **Split correctly across 3 domains**: STP scoring → Customs Adjudication domain (simulation-side: authority logic). Hold determination → Compliance & Screening + Exception Management (platform-side: `ComplianceScreened` HOLD → `ExceptionCreated`). Reclassification → Trade Intelligence (re-classification on `ProductUpdated`). Inspection scheduling → Customs Adjudication (exam events). | **PASS** |
| 2 | **State Machine** (guarded transitions, multi-jurisdiction, event logging) | Shipment Service | **Shipment Lifecycle domain** (Section 3.5): jurisdiction-specific sub-machines (US, EU, BR, IN, CN), guarded transitions with actor permissions and preconditions, every transition logged as event. | **PASS** |
| 3 | **Cage Management** (cargo tracking, dwell time, GO deadline, storage costs) | Cargo Management Service | **Exception Management domain** (Section 3.8): cage status fields in `Exception` state object (`facility`, `intake_at`, `exam_type`, `go_deadline`, `dwell_days`, `storage_cost`). Cage events: `CageIntake`, `CageExamScheduled`, `CageReleased`, `GODeadlineWarning`. Financial Settlement consumes `CageIntake`/`CageReleased` for storage cost accrual. | **PASS** |
| 4 | **Document Validation** (invoice/packing list cross-validation, weight/quantity variance, origin cert, USMCA checking) | Document Service | **Document Management domain** (Section 3.11): Document Validator produces `DocumentValidated` and `DocumentDiscrepancyFound` events. Discrepancy detection is explicitly listed. Declaration Management consumes `DocumentValidated` to update checklist. | **PASS** |
| 5 | **Financial Calculations** (bond requirement, fee schedules, payment processing, duty deposits) | Financial Service | **Financial Settlement domain** (Section 3.9): `FinancialRecord` state object includes `bond_amount`, `bond_type`, `mpf`, `hmf`, `broker_fees`. Consumes `TariffCalculated` for duty updates. | **PASS** (but underspecified — see CONCERN-3) |
| 6 | **D&D Tracking** (free time, demurrage/detention accrual, appointments) | Financial Service | **Financial Settlement domain** (Section 3.9): `demurrage` field with `accruing`, `days`, `daily_rate`, `total`, `free_time_days`. `DemurrageAccruing` event. Triggered by `ShipmentArrivedAtCustoms` (start clock) and `CageIntake`/`CageReleased` (storage costs). | **PASS** |
| 7 | **ISF Filing** (10+2 deadline enforcement, data validation, late penalties) | Clearance Service | **Declaration Management domain** (Section 3.6): Dedicated `ISFFiling` state object. `isf_filings` table. Events: `ISFFiled`, `ISFAmended`, `ISFLateFiling`. Explicit paragraph covering deadline enforcement (24h before vessel loading), data validation, late filing penalty assessment, amendment tracking. | **PASS** |
| 8 | **PGA Processing** (agency-specific review workflows, approval/rejection) | Compliance Service | **Split across 2 domains**: Compliance & Screening produces `PGAReviewRequired` event with PGA determination logic. Customs Adjudication has `PGABot` in simulation (line 1361) calling `POST /adjudication/declarations/{id}/decide` for PGA-specific decisions. This correctly models PGA agencies as external authorities. | **PASS** |
| 9 | **Dashboard Aggregation** (SQL aggregation, KPIs, status distributions) | Dashboard Service (CQRS) | **Redis Cache Projector** (Section 6): `agg:dashboard:platform` and `agg:dashboard:broker:{id}` aggregation keys. Dashboard data is built from events, not polling SQL queries. This IS the CQRS read model the audit recommended. | **PASS** |
| 10 | **Reference Number Generation** (HAWB, MBL, entry, tracking) | Shipment Service | **Shipment Lifecycle domain** (Section 3.5, lines 492-504): Explicit "Reference Number Generation" subsection with format table covering HAWB, MAWB, MBL, Entry Number, Tracking Number, ISF Transaction. Marked as "platform concerns, not simulation artifacts." | **PASS** |
| 11 | **Routing Intelligence** (hub routing, regulatory touchpoints, border-crossing mode) | Logistics Service | **Shipment Lifecycle domain** (Section 3.5, lines 506-516): Explicit "Routing Intelligence" subsection covering hub routing tables, regulatory touchpoint derivation, border-crossing mode determination, multi-modal leg generation. Marked as "platform knowledge that exists in production." | **PASS** |

**Result: ALL 11 ITEMS ACCOUNTED FOR. 11/11 PASS.**

Every piece of business logic identified as misplaced in simulation has a clear home in a platform domain. The architecture document is explicit about routing intelligence and reference number generation being platform concerns (not simulation artifacts), which directly addresses the audit's findings.

**Note on STP placement:** The audit categorized STP scoring as platform logic. The architecture correctly places it in the simulation (CustomsAuthorityBot) because STP is what the **customs authority** does — it's their risk assessment algorithm, not the importer's. In production, CBP runs STP internally; the Clearance Platform receives the authority's decision via the Customs Adjudication domain API. The platform-side hold logic (UFLPA, DPS, documentation holds) correctly lives in Compliance & Screening and Exception Management.

### Orchestration Pipeline → Event Choreography

**Audit concern:** "The current orchestration pipeline (E1 → parallel E2/E3/E4 → assembly) should be replaced by NATS event choreography."

**Architecture document response:** This is fully addressed. The architecture uses **event choreography, not orchestration**:

1. `ShipmentCreated` event fans out to multiple subscribers concurrently:
   - Trade Intelligence → `ClassificationProduced` → `TariffCalculated` → `FTADetermined`
   - Compliance → `ComplianceScreened`
   - Document Management → determines requirements
   - Declaration Management → creates draft declaration

2. Each domain independently subscribes to events and produces its own output events. There is NO central orchestrator calling E1 then E2/E3/E4 then assembly.

3. Order Management is explicitly an "assembler" — it receives intelligence via event subscriptions, not by orchestrating calls (Section 3.4, line 398).

4. The Broker Intelligence Flow (Section 7) visualizes this: all intelligence accumulation happens concurrently within seconds via event subscriptions, not sequential pipeline execution.

**Result: PASS — orchestration pipeline fully replaced by event choreography.**

---

## Strengths

1. **Clean-sheet achieved.** No vestiges of the actor-centric Redis Streams design. The architecture genuinely organizes around business domains, not actors or screens.

2. **Trade Intelligence as a proper domain** with Classification moved into it is the right call. The proactive event production pattern (subscribe to `ProductCreated` → emit `ClassificationProduced`) is the textbook event-driven approach.

3. **Customs Adjudication separation** is the most significant architectural improvement. The clean boundary between broker (Declaration Management) and authority (Customs Adjudication) makes the platform production-ready for real customs authority integration.

4. **Simulation separation is excellent.** The `sim.*` vs `clearance.*` subject distinction, combined with the bot-uses-APIs pattern, means the clearance platform genuinely doesn't know about the simulation.

5. **CQRS via Redis Cache Projector** is well-designed. The denormalized read model (Section 6.2) with multi-domain event composition into a single entity view is the right UX pattern.

6. **Comprehensive MCP tool coverage** — ~60+ tools across 11 domains ensures AI agents have full platform access.

7. **State-transfer events with full snapshots** (Appendix A) eliminate the stale-read problem and make consumers self-sufficient.

8. **The litmus test** (Section 3.2, line 213) is a powerful architectural guard: "Can I add a new consumer by just subscribing to an event?" This should be applied to every domain.

---

## Review Checklist (Team Lead's 12 Points)

| # | Check | Result |
|---|-------|--------|
| 1 | Every domain emits state-transfer events? | **PASS** — All 11 domains produce events with full state snapshots |
| 2 | No domain calls another domain's API? | **PASS** — Order Management is an event assembler. Trade Intelligence produces events, doesn't expose sync APIs for routine computation. |
| 3 | Simulation is a pure API consumer? | **PASS** — Section 10 is clean. `sim.*` subjects only. |
| 4 | Intelligence lives inside domains? | **PASS** — Classification, Tariff, FTA in Trade Intelligence. Compliance Engine in Compliance. Broker/Filing Intel in Declaration Mgmt. Regulatory Monitor in Regulatory Intel. |
| 5 | CQRS: Redis for reads, services for writes? | **PASS** — Consistent across all domain API surfaces. |
| 6 | NATS JetStream (not Redis Streams)? | **PASS** — 12 JetStream streams, proper configuration. |
| 7 | Share-on-principle via events? | **PASS** — Litmus test defined. Consumers subscribe, producers emit. |
| 8 | APIs as MCP tools? | **PASS** — Comprehensive mapping in every domain. |
| 9 | Actors sit on top, not inside? | **PASS** — Principle 4. All actors in simulation. |
| 10 | Trade Intelligence owns state? | **PASS** — 3 state objects, 15+ owned tables, Qdrant collection. "Proper domain that owns state." |
| 11 | Declaration vs Customs Adjudication separation? | **PASS** — Domain #6 (broker) vs Domain #7 (authority). Clean event boundary. |
| 12 | Product Catalog vs Trade Intelligence boundary? | **PASS** — Product Catalog = shipper's description. Trade Intelligence = system's classification + tariff + FTA. Classification Engine correctly in Trade Intelligence. |

---

---

## V3 Verification (Final)

All 3 significant concerns and all 5 minor issues have been addressed in v3:

### CONCERN-1: Event Delivery Guarantees → RESOLVED (Section 6.4)
- At-least-once delivery with explicit ack
- Per-consumer-type idempotency strategy with `processed_events` tables per schema
- Per-entity version numbers for ordering; per-section version counters for denormalized views
- DLQ at `clearance.dlq.{original-subject}` with 3-retry exponential backoff (1s/5s/30s), PagerDuty alerting, `/admin/replay` recovery endpoint
- Consumer failure modes table covering crash, persistent failure, slow consumer, and schema mismatch

### CONCERN-2: Boundary Enforcement → RESOLVED (Section 11.3)
- `clearance_platform/domains/` package structure with `__init__.py` as public API (events only)
- `import-linter` in `pyproject.toml` with independence contracts for all 11 domains
- Schema isolation via database roles with per-schema `USAGE` grants + CI SQL analysis
- Extraction fitness test with 5 automated checks
- 4 CI fitness functions that block PRs on violation

### CONCERN-3: Financial Settlement → RESOLVED (Section 3.9)
- 4 state objects (was 1): `FinancialRecord`, `Bond`, `DutyDrawback`, `Reconciliation`
- 4-phase lifecycle: estimated → provisional → liquidated → reconciled
- Bond management with continuous vs single-entry, sufficiency tracking, utilization alerts
- D&D with mode-specific free time (ocean 5d, air 48h), appointment tracking
- Drawback claims (manufacturing, unused, rejected)
- Reconciliation (reliquidation, post-summary correction, protest refund, rate change)
- 11 events (was 3), 10 consumed events (was 7), 10 API endpoints (was 2), 4 tables (was 1)

### Minor Issues — ALL RESOLVED
| # | Issue | Resolution |
|---|-------|-----------|
| 1 | Missing readiness concept | Section 7.1: Readiness model with gates (classification 20%, tariff 20%, compliance 20%, documents 20%, bond 10%, ISF 10%), three states (not_ready, ready_with_warnings, ready_to_file), stored in Redis cache |
| 2 | Regulatory dual trigger | Line 1027: Explicit dual model — periodic polling (15-min cron) + manual/on-demand via API |
| 3 | Cage subprocess | Line 796: Full cage lifecycle documented as subprocess within Exception Management with flow diagram |
| 4 | Interaction matrix gaps | Matrix expanded to 11×11 with Customs Adjudication. Trade Intel → Compliance and Document → Declaration now present |
| 5 | Description Quality event | Line 192: `DescriptionQualityProduced` added as event in Product Catalog domain |

---

## Final Verdict

**The architecture document is complete and ready to serve as the definitive reference.**

- 9/9 stakeholder mandates: **PASS**
- 12/12 review checklist items: **PASS**
- 11/11 misplaced logic items accounted for: **PASS**
- 3/3 significant concerns: **RESOLVED**
- 5/5 minor issues: **RESOLVED**
- Orchestration → event choreography: **VERIFIED**

*Review complete. No open items remaining.*
