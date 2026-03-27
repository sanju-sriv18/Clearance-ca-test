# Implementation Backlog: v5.2 Architecture Completion

**Status:** Active — Phase 2 (Sprints 11-16)
**Date:** 2026-02-09
**Scope:** Complete the v5.2 modular monolith architecture. Phase 1 (Sprints 1-10) established domain packages, event schemas, CQRS scaffolding, and gateway routing. Phase 2 fills all skeleton domains with real models, implements export declarations, completes CQRS projectors, and wires event-driven intelligence.

---

## Phase 1 Summary (Sprints 1-10 — COMPLETE)

| Sprint | Deliverable | Tests |
|--------|------------|-------|
| 1 | Foundation: 14 domain packages, shared infra, event bus | 857 |
| 2 | DB schemas: 7 migrations (016-022), 18 entity tables, 25 tables migrated | 857 |
| 3 | Core domain services: Product Catalog, Trade Intelligence, Compliance | 866 |
| 4 | Logistics domains: Cargo, Consolidation, Shipment Lifecycle | 876 |
| 5 | Declaration Management & Customs Adjudication | 885 |
| 6 | Remaining 6 domains: Exception, Financial, Regulatory, Document, Party, Order wiring | 901 |
| 7 | CQRS Redis cache projector infrastructure + 5 projectors | 914 |
| 8 | Frontend type alignment for 13 domain entities | 914 |
| 9 | Simulation platform separation (sim.* namespace, SimDomainBridge) | 932 |
| 10 | Integration tests, documentation, architecture status | 957 |

**Current state:** 957 tests passing, frontend builds clean, all domain routes in gateway, 45 DB tables across 14 schemas.

---

## Phase 2 Gap Analysis

### What Exists vs. What Architecture Requires

**Database (45 tables):** All architecture-specified tables exist with correct column schemas. No new migrations needed.

**Domain Models (ORM):**
- 5 domains SKELETON (11-12 LOC stub): cargo_handling_units, customs_adjudication, exception_management, financial_settlement, party_management
- 9 domains FUNCTIONAL (44-432 LOC): product_catalog, trade_intelligence, compliance, order_management, shipment_lifecycle, consolidation, declaration_management, regulatory_intelligence, document_management

**Domain Routes/Services:** All 14 domains have routes and services, but skeleton-model domains delegate to raw SQL in the legacy `app/` layer rather than using proper ORM models.

**CQRS Projectors:** 5 of 15 exist (product_catalog, shipment_lifecycle, cargo_handling_units, consolidation, declaration_management). 10 missing.

**Export Declarations (v5.2):** DB table exists with correct schema. Zero application code: no model, no events, no routes, no service logic.

**Event Infrastructure:** Domain event classes defined for all 14 domains. In-process bus working. NATS JetStream stream definitions exist but bus still uses Redis pub/sub fallback.

**NATS Stream Naming:** `CLEARANCE_CARGO` stream listens to `clearance.cargo.>` but events use `clearance.handling-unit.>` subject. Mismatch needs fixing.

**Frontend:** Comprehensive (31 screens, 62+ components, 80+ API functions). No export declaration, party management, or financial management screens. Types for v5.2 domain entities already defined in `types.ts`.

---

## Sprint 11: Entity Model Completion — Batch A

**Goal:** Fill the 5 skeleton domain models with full SQLAlchemy ORM classes mapping to existing DB tables.

### Items

#### 11.1 Cargo & Handling Units Model
- **File:** `clearance_platform/domains/cargo_handling_units/models.py`
- **Map to:** `cargo.hu_transit_events` (transit events table already separate)
- **Note:** The main `handling_units` table lives in legacy `app/` — this domain needs a model that maps to the shipments table's HU-related columns or the handling_units concept embedded in shipments
- **Actually:** Check if `cargo.handling_units` table exists — if not, HU is modeled as shipment sub-entity
- **Acceptance:** Model imports cleanly, matches DB schema, passes `uv run pytest`

#### 11.2 Customs Adjudication Model
- **File:** `clearance_platform/domains/customs_adjudication/models.py`
- **Map to:** `adjudication.adjudication_decisions` (id, entry_filing_id, decision_type, authority, decision_date, reason, details JSONB, protest_filed, protest_resolved, created_at, updated_at)
- **Acceptance:** Model matches DB columns, imports cleanly

#### 11.3 Exception Management Model
- **File:** `clearance_platform/domains/exception_management/models.py`
- **Map to:** `exception.exceptions` (id, handling_unit_id, consolidation_id, exception_type, hold_type, status, severity, transit_hold_detail JSONB, cage_status JSONB, resolution JSONB, created_at, updated_at)
- **Acceptance:** Model matches DB columns

#### 11.4 Financial Settlement Models
- **File:** `clearance_platform/domains/financial_settlement/models.py`
- **Map to:** `financial.financial_records`, `financial.bonds`, `financial.duty_drawbacks`, `financial.reconciliations`
- **Models:** FinancialRecord (30+ columns), Bond, DutyDrawback, Reconciliation
- **Acceptance:** All 4 models match DB schemas

#### 11.5 Party Management Models
- **File:** `clearance_platform/domains/party_management/models.py`
- **Map to:** `party.parties`, `party.power_of_attorney`, `party.importers_of_record`
- **Models:** Party (with jurisdiction_identifiers JSONB, trust_programs JSONB, compliance_score), PowerOfAttorney, ImporterOfRecord
- **Acceptance:** All 3 models match DB schemas

#### 11.6 Fix NATS Stream Naming
- **File:** `clearance_platform/shared/streams.py`
- **Change:** Rename `CLEARANCE_CARGO` stream to `CLEARANCE_HANDLING_UNIT`, update subjects from `clearance.cargo.>` to `clearance.handling-unit.>`
- **Acceptance:** Stream definitions align with event subject naming

---

## Sprint 12: Export Declarations (v5.2 Feature)

**Goal:** Implement the entire export declaration feature — model, events, service, routes.

### Items

#### 12.1 ExportDeclaration Model
- **File:** `clearance_platform/domains/declaration_management/models.py` (extend)
- **Map to:** `declaration.export_declarations` (id, handling_unit_id, consolidation_id, exporter_id, agent_id, jurisdiction, jurisdiction_filing_type, export_reference, export_license, jurisdiction_status, platform_status, filing_status, filing_data JSONB, filed_at, released_at, created_at, updated_at)
- **Also:** ISFFiling and ENSFiling models if not already present
- **Acceptance:** Models match DB tables

#### 12.2 Export Declaration Events
- **File:** `clearance_platform/domains/declaration_management/events.py` (extend)
- **Add:** ExportDeclarationCreated, ExportDeclarationSubmitted, ExportDeclarationCleared, ExportDeclarationHeld, ExportDeclarationAmended, ExportDeclarationRejected
- **Subject:** `clearance.declaration.export-*`
- **Acceptance:** All 6 events defined with proper fields

#### 12.3 Export Declaration Service
- **File:** `clearance_platform/domains/declaration_management/service.py` (extend)
- **Add:** create_export_declaration, submit_export_declaration, amend_export_declaration, get_export_declaration, list_export_declarations, get_export_status
- **Logic:** Jurisdiction-specific filing validation, two-layer status (jurisdiction_status + platform_status), export checklist generation
- **Acceptance:** Service methods work end-to-end with DB

#### 12.4 Export Declaration Routes
- **File:** `clearance_platform/domains/declaration_management/routes.py` (extend)
- **Add:** 9 endpoints per architecture Section 3.8:
  - `POST /api/broker/export-declarations` — create
  - `GET /api/broker/export-declarations` — list
  - `GET /api/broker/export-declarations/{id}` — get
  - `POST /api/broker/export-declarations/{id}/submit` — submit
  - `POST /api/broker/export-declarations/{id}/amend` — amend
  - `GET /api/broker/export-declarations/{id}/status` — status
  - `GET /api/broker/export-declarations/{id}/checklist` — checklist
  - `POST /api/broker/export-declarations/{id}/checklist` — update checklist
  - `DELETE /api/broker/export-declarations/{id}` — cancel/soft delete
- **Acceptance:** All 9 endpoints respond correctly

#### 12.5 Export Adjudication Event
- **File:** `clearance_platform/domains/customs_adjudication/events.py` (extend)
- **Add:** ExportAdjudicationDecisionEvent
- **Acceptance:** Event defined, matches architecture

#### 12.6 Unit Tests for Export Declarations
- **File:** `tests/domains/test_export_declarations.py` (new)
- **Coverage:** Model creation, service CRUD, event emission, jurisdiction validation
- **Acceptance:** 15+ tests passing

---

## Sprint 13: CQRS Projectors

**Goal:** Complete the CQRS read-side by building all 10 missing Redis projectors.

### Items

#### 13.1 Trade Intelligence Projector
- **File:** `clearance_platform/domains/trade_intelligence/projector.py` (new)
- **Keys:** `entity:classification:{id}`, `entity:tariff-calc:{id}`, `entity:fta-determination:{id}`, `idx:classification:by-product:{product_id}`
- **Subscribes to:** ClassificationProduced, TariffCalculated, FTADetermined

#### 13.2 Compliance Projector
- **File:** `clearance_platform/domains/compliance/projector.py` (new)
- **Keys:** `entity:compliance-result:{id}`, `idx:compliance:by-shipment:{shipment_id}`
- **Subscribes to:** ComplianceScreened

#### 13.3 Order Management Projector
- **File:** `clearance_platform/domains/order_management/projector.py` (new)
- **Keys:** `entity:order:{id}`, `idx:order:by-status:{status}`
- **Subscribes to:** OrderCreated, OrderUpdated, OrderShipped

#### 13.4 Customs Adjudication Projector
- **File:** `clearance_platform/domains/customs_adjudication/projector.py` (new)
- **Keys:** `entity:adjudication:{id}`, `idx:adjudication:by-entry:{entry_id}`
- **Subscribes to:** AdjudicationDecision, CF28Issued, CF29Issued, ExamScheduled

#### 13.5 Exception Management Projector
- **File:** `clearance_platform/domains/exception_management/projector.py` (new)
- **Keys:** `entity:exception:{id}`, `idx:exception:by-hu:{hu_id}`, `idx:exception:by-status:{status}`
- **Subscribes to:** ExceptionCreated, ExceptionUpdated, HoldReleased

#### 13.6 Financial Settlement Projector
- **File:** `clearance_platform/domains/financial_settlement/projector.py` (new)
- **Keys:** `entity:financial:{id}`, `idx:financial:by-shipment:{shipment_id}`
- **Subscribes to:** FeesCalculated, DemurrageAccruing, PaymentCompleted

#### 13.7 Regulatory Intelligence Projector
- **File:** `clearance_platform/domains/regulatory_intelligence/projector.py` (new)
- **Keys:** `entity:regulatory-signal:{id}`, `idx:regulatory:by-type:{type}`
- **Subscribes to:** RegulatorySignalDetected, RegulatoryImpactAssessed

#### 13.8 Document Management Projector
- **File:** `clearance_platform/domains/document_management/projector.py` (new)
- **Keys:** `entity:document:{id}`, `idx:document:by-shipment:{shipment_id}`
- **Subscribes to:** DocumentUploaded, DocumentValidated, DocumentGenerated

#### 13.9 Party Management Projector
- **File:** `clearance_platform/domains/party_management/projector.py` (new)
- **Keys:** `entity:party:{id}`, `idx:party:by-type:{type}`, `idx:party:by-name:{name}`
- **Subscribes to:** PartyCreated, PartyUpdated

#### 13.10 Verify Existing Projectors
- Review 5 existing projectors for completeness against architecture Section 6
- Ensure Redis key naming follows `entity:{type}:{id}`, `idx:{type}:by-{field}:{value}`

#### 13.11 Projector Registration
- **File:** `clearance_platform/shared/projector_registry.py` (new or extend existing)
- Register all 15 projectors for startup initialization
- Wire projectors to event bus subscriptions

---

## Sprint 14: Service Enrichment + Event Wiring

**Goal:** Flesh out domain services with real business logic and connect intelligence engines to event subscriptions.

### Items

#### 14.1 Service Enrichment — Skeleton Domains
- Fill service methods for cargo_handling_units, customs_adjudication, exception_management, financial_settlement, party_management
- Each service method: validate → write to DB via ORM model → emit domain event → return result
- Use the new ORM models from Sprint 11 instead of raw SQL passthrough

#### 14.2 Filing Readiness Model
- **File:** `clearance_platform/domains/declaration_management/service.py` (extend)
- Track readiness gates: classification, tariff, compliance, documents, bond, ISF
- Each intelligence event updates relevant gate
- Compute readiness_score, cache in Redis
- Broker queue sorted by readiness + priority

#### 14.3 Intelligence Event Subscriptions
- Wire E1 (Classification) to subscribe to ProductCreated/Updated
- Wire E2 (Tariff) to subscribe to ClassificationProduced, ShipmentCreated
- Wire E3 (Broker Intelligence) to subscribe to DeclarationCreated
- Wire E4 (FTA) to subscribe to ClassificationProduced
- Wire E5 (Exception) to subscribe to ExceptionCreated
- Each intelligence engine becomes an event consumer, not a synchronously-called function

#### 14.4 Cross-Domain Event Reactions
- HU service subscribes to AdjudicationDecision → create hold / cage intake
- HU service subscribes to ExportDeclarationCleared → set export_clearance
- Consolidation subscribes to HU status changes → update consolidation state
- Exception subscribes to AdjudicationDecision (held) → create exception

#### 14.5 Product Catalog v5.1 Fields
- Add manufacturer_id, material_composition, unit_of_measure, catalog_registrations to Product model
- Add DescriptionQualityProduced event

---

## Sprint 15: Frontend Alignment

**Goal:** Add frontend screens and API client functions for new v5.2 features.

### Items

#### 15.1 Export Declaration Types
- **File:** `frontend/src/types.ts` (extend)
- Add/verify ExportDeclaration, ExportDeclarationItem types
- Add ExportFilingStatus, ExportJurisdictionType enums

#### 15.2 Export Declaration API Client
- **File:** `frontend/src/api/client.ts` (extend)
- Add functions: createExportDeclaration, listExportDeclarations, getExportDeclaration, submitExportDeclaration, amendExportDeclaration, getExportStatus, getExportChecklist, updateExportChecklist
- Transform functions for snake_case→camelCase

#### 15.3 Export Declaration Screen
- **File:** `frontend/src/surfaces/broker/screens/ExportDeclarations.tsx` (new)
- List view with filters (jurisdiction, status)
- Detail view with checklist, filing data, status timeline
- Submit/amend actions

#### 15.4 Party Management Screen
- **File:** `frontend/src/surfaces/platform/screens/PartyManagement.tsx` (new)
- Party list with type filter
- Party detail with POA, IOR, compliance score

#### 15.5 Router Updates
- **File:** `frontend/src/router.tsx`
- Add routes for export declarations and party management screens

#### 15.6 Frontend Build Verification
- Ensure `npm run build` succeeds with zero errors
- Verify no broken imports or type mismatches

---

## Sprint 16: Integration Tests + Cleanup

**Goal:** Comprehensive integration testing, legacy code cleanup, architecture fitness verification.

### Items

#### 16.1 Domain Integration Tests
- Test each skeleton domain's model ↔ DB ↔ service ↔ route chain end-to-end
- Test export declaration full lifecycle (create → submit → clear/hold)
- Test cross-domain event reactions

#### 16.2 Legacy Route Cleanup
- Move analysis and description_quality routes from `app/api/routes/` into appropriate domain packages
- Remove legacy route imports from gateway
- Verify no remaining `app/api/routes/` imports in gateway

#### 16.3 Architecture Fitness
- Verify all 15 projectors registered and initializing
- Verify all domain models map to correct schema tables
- Verify no cross-domain imports (except via shared/)
- Verify all writes emit domain events
- Verify NATS stream definitions match event subjects

#### 16.4 Final Test Suite
- All tests pass (target: 1000+)
- Frontend builds clean
- Health endpoint returns all OK

---

## Dependency Graph

```
Sprint 11 (Entity Models)
    ↓
Sprint 12 (Export Declarations) ← depends on Sprint 11 for declaration model patterns
    ↓
Sprint 13 (CQRS Projectors) ← depends on Sprint 11 for model definitions
    ↓
Sprint 14 (Service Enrichment) ← depends on Sprint 11 models + Sprint 13 projectors
    ↓
Sprint 15 (Frontend) ← depends on Sprint 12 export routes
    ↓
Sprint 16 (Integration + Cleanup) ← depends on all prior sprints
```

Sprint 11 blocks everything. Sprints 12 and 13 can partially overlap since they touch different files. Sprint 15 can start once Sprint 12 export routes are done.

---

## Risk Register

| Risk | Mitigation |
|------|-----------|
| Skeleton models don't match DB schema exactly | Inspect DB columns with `\d schema.table` before writing model |
| Export declaration routes conflict with existing broker routes | Use distinct prefix `/api/broker/export-declarations` |
| Projector Redis keys collide | Follow strict naming convention from architecture Section 6.2 |
| Intelligence engine wiring breaks existing test assumptions | Run full test suite after each wiring change |
| Frontend changes break existing screens | Build verification after every change |

---

## Total Estimated Scope

| Sprint | LOC Estimate | New Tests |
|--------|-------------|-----------|
| 11 | 800-1200 | 30-40 |
| 12 | 600-900 | 15-25 |
| 13 | 1500-2500 | 10-20 |
| 14 | 800-1200 | 15-25 |
| 15 | 600-1000 | 5-10 |
| 16 | 200-400 | 20-30 |
| **Total** | **4500-7200** | **95-150** |
