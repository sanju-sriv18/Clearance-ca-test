# Sprint 12: Export Declarations (v5.2 Feature)

**Goal:** Implement the entire export declaration feature — 3 new models, 6 export events, 1 adjudication event, service methods, 9 API endpoints, and unit tests.

**Acceptance:** All 1096+ tests pass, frontend builds, all new models import cleanly, export endpoints respond.

---

## Task Assignments

### Agent A: Models + Events + Adjudication Event

**Files owned:**
- `clearance_platform/domains/declaration_management/models.py` (extend with 3 new classes)
- `clearance_platform/domains/declaration_management/events.py` (extend with 6 new events)
- `clearance_platform/domains/customs_adjudication/events.py` (extend with 1 new event)
- `tests/domains/test_export_declaration_models.py` (new)

**Instructions:**

1. **ExportDeclaration model** — add to `declaration_management/models.py` after BrokerMessage:
   Map to `declaration.export_declarations`:
   - id: UUID pk, server_default=func.gen_random_uuid()
   - handling_unit_id: UUID nullable
   - consolidation_id: UUID nullable
   - exporter_id: UUID nullable
   - agent_id: UUID nullable
   - jurisdiction: String(10), nullable=False
   - jurisdiction_filing_type: String(30), nullable=True
   - export_reference: String(100), nullable=True
   - export_license: String(100), nullable=True
   - jurisdiction_status: String(50), nullable=True
   - platform_status: String(20), nullable=False, server_default="pending"
   - filing_status: String(20), nullable=False, server_default="draft"
   - filing_data: JSONB, nullable=True
   - filed_at: DateTime(timezone=True), nullable=True
   - released_at: DateTime(timezone=True), nullable=True
   - created_at, updated_at (TimestampMixin)
   - Indexes: ix_declaration_export_declarations_hu_id, ix_declaration_export_declarations_consol_id, ix_declaration_export_declarations_jurisdiction
   - `__table_args__ = (..., {"schema": "declaration"})`

2. **ISFFiling model** — add to `declaration_management/models.py`:
   Map to `declaration.isf_filings`:
   - id: UUID pk, server_default=func.gen_random_uuid()
   - handling_unit_id: UUID nullable
   - consolidation_id: UUID nullable
   - declaration_id: UUID nullable
   - transaction_number: String(50), nullable=True
   - filing_status: String(20), nullable=False, server_default="pending"
   - deadline: DateTime(timezone=True), nullable=False
   - is_late: Boolean, nullable=False, server_default=text("false")
   - data: JSONB, nullable=True
   - validation_issues: JSONB, nullable=True
   - filed_at: DateTime(timezone=True), nullable=True
   - created_at only (no TimestampMixin, use server_default=func.now())
   - Indexes: ix_declaration_isf_filings_hu_id, ix_declaration_isf_filings_consol_id

3. **ENSFiling model** — add to `declaration_management/models.py`:
   Map to `declaration.ens_filings`:
   - id: UUID pk, server_default=func.gen_random_uuid()
   - consolidation_id: UUID, nullable=False
   - carrier_id: String(100), nullable=False
   - ens_status: String(20), nullable=False, server_default="pending"
   - mrn: String(50), nullable=True
   - filing_deadline: DateTime(timezone=True), nullable=False
   - hs_codes: JSONB, nullable=True
   - consignor: String(255), nullable=True
   - consignee: String(255), nullable=True
   - filed_at: DateTime(timezone=True), nullable=True
   - created_at only (no TimestampMixin, use server_default=func.now())
   - Index: ix_declaration_ens_filings_consol_id

4. **Export Declaration Events** — add to `declaration_management/events.py`:
   Follow existing event pattern (BaseEvent + EventMetadata). Add 6 events:
   - `ExportDeclarationCreatedEvent`: export_declaration_id, jurisdiction, filing_type, handling_unit_id, consolidation_id
   - `ExportDeclarationSubmittedEvent`: export_declaration_id, jurisdiction, filing_data
   - `ExportDeclarationClearedEvent`: export_declaration_id, jurisdiction, export_reference, released_at
   - `ExportDeclarationHeldEvent`: export_declaration_id, jurisdiction, hold_reason, jurisdiction_status
   - `ExportDeclarationAmendedEvent`: export_declaration_id, jurisdiction, amendments (dict)
   - `ExportDeclarationRejectedEvent`: export_declaration_id, jurisdiction, rejection_reason

   All subjects should be `clearance.declaration.export-created`, `clearance.declaration.export-submitted`, etc.

5. **ExportAdjudicationDecisionEvent** — add to `customs_adjudication/events.py`:
   - export_declaration_id, decision_type, jurisdiction, decision_details (dict), decided_by (str optional)
   - Subject: `clearance.adjudication.export-decision`

6. **Unit tests** in `tests/domains/test_export_declaration_models.py`:
   - Test ExportDeclaration model: instantiation, all columns, defaults, schema
   - Test ISFFiling model: instantiation, all columns, defaults, schema
   - Test ENSFiling model: instantiation, all columns, defaults, schema
   - Test all 6 export events: creation, serialization, correct subjects
   - Test ExportAdjudicationDecisionEvent: creation, serialization
   - At least 20 tests

**Patterns to follow:**
- Models: Look at existing `EntryFiling` class in the same file for pattern
- Events: Look at existing `DeclarationCreatedEvent` in the same events.py
- For models without updated_at: Do NOT use TimestampMixin, manually add `created_at` column

---

### Agent B: Service + Routes

**Files owned:**
- `clearance_platform/domains/declaration_management/service.py` (extend)
- `clearance_platform/domains/declaration_management/routes.py` (extend)
- `tests/domains/test_export_declaration_service.py` (new)

**Instructions:**

1. **Service methods** — add to `DeclarationManagementService` class:
   - `emit_export_declaration_created(export_declaration_id, jurisdiction, filing_type, handling_unit_id, consolidation_id)` — emit ExportDeclarationCreatedEvent
   - `emit_export_declaration_submitted(export_declaration_id, jurisdiction, filing_data)` — emit ExportDeclarationSubmittedEvent
   - `emit_export_declaration_cleared(export_declaration_id, jurisdiction, export_reference, released_at)` — emit ExportDeclarationClearedEvent
   - `emit_export_declaration_held(export_declaration_id, jurisdiction, hold_reason, jurisdiction_status)` — emit ExportDeclarationHeldEvent
   - `emit_export_declaration_amended(export_declaration_id, jurisdiction, amendments)` — emit ExportDeclarationAmendedEvent
   - `emit_export_declaration_rejected(export_declaration_id, jurisdiction, rejection_reason)` — emit ExportDeclarationRejectedEvent

   Follow existing emit_* method pattern in the same service class.

2. **Route endpoints** — add to routes.py (at the bottom, before any catch-all routes):
   Add 9 endpoints under the existing `router`:

   ```python
   # --- Export Declaration Endpoints ---

   @router.post("/export-declarations", status_code=201)
   async def create_export_declaration(body: CreateExportDeclarationRequest, session: AsyncSession = Depends(_get_session)):
       # Insert new ExportDeclaration, emit event, return

   @router.get("/export-declarations")
   async def list_export_declarations(jurisdiction: str | None = None, status: str | None = None, session: ...):
       # Query with optional filters

   @router.get("/export-declarations/{export_id}")
   async def get_export_declaration(export_id: uuid.UUID, session: ...):
       # Get by ID or 404

   @router.post("/export-declarations/{export_id}/submit")
   async def submit_export_declaration(export_id: uuid.UUID, session: ...):
       # Update filing_status to 'submitted', emit event

   @router.post("/export-declarations/{export_id}/amend")
   async def amend_export_declaration(export_id: uuid.UUID, body: AmendExportDeclarationRequest, session: ...):
       # Update filing_data, emit event

   @router.get("/export-declarations/{export_id}/status")
   async def get_export_declaration_status(export_id: uuid.UUID, session: ...):
       # Return jurisdiction_status + platform_status + filing_status

   @router.get("/export-declarations/{export_id}/checklist")
   async def get_export_checklist(export_id: uuid.UUID, session: ...):
       # Generate export checklist based on jurisdiction

   @router.post("/export-declarations/{export_id}/checklist")
   async def update_export_checklist(export_id: uuid.UUID, body: dict, session: ...):
       # Update filing_data checklist portion

   @router.delete("/export-declarations/{export_id}")
   async def cancel_export_declaration(export_id: uuid.UUID, session: ...):
       # Soft delete: set filing_status='cancelled', platform_status='cancelled'
   ```

   Request/Response models:
   - `CreateExportDeclarationRequest`: jurisdiction (str), handling_unit_id (UUID optional), consolidation_id (UUID optional), exporter_id (UUID optional), jurisdiction_filing_type (str optional), export_license (str optional), filing_data (dict optional)
   - `AmendExportDeclarationRequest`: amendments (dict), reason (str optional)

   Use existing `_get_session` dependency from the same file. Use ORM model `ExportDeclaration` from models.py.
   Import the new model: `from clearance_platform.domains.declaration_management.models import ExportDeclaration`

   **IMPORTANT:** The existing routes.py uses raw SQL via `session.execute(text(...))` extensively. For the NEW export endpoints, follow the same pattern — use `select(ExportDeclaration)` ORM queries where possible, but if the file's style is raw SQL, match that style. Consistency within the file matters.

   **Rate limiting:** Apply `@limiter.limit("50000/minute")` to each endpoint (same as existing endpoints).

3. **Unit tests** in `tests/domains/test_export_declaration_service.py`:
   - Test each of the 6 emit methods produces the correct event
   - Test the Pydantic request models for create and amend
   - At least 10 tests

---

## Verification Steps

After both agents complete:

1. `cd clearance-engine/backend && uv run pytest tests/ -v --tb=short` — all 1096+ tests pass
2. `cd clearance-engine/frontend && npm run build` — zero errors
3. Import checks:
   ```
   uv run python -c "from clearance_platform.domains.declaration_management.models import ExportDeclaration, ISFFiling, ENSFiling; print('OK')"
   uv run python -c "from clearance_platform.domains.declaration_management.events import ExportDeclarationCreatedEvent, ExportDeclarationSubmittedEvent; print('OK')"
   uv run python -c "from clearance_platform.domains.customs_adjudication.events import ExportAdjudicationDecisionEvent; print('OK')"
   ```
4. Endpoint check: `curl -s http://localhost:4000/api/broker/export-declarations | head -c 200`

---

## Dependency Note

Agent B depends on Agent A's models and events. However, Agent B can work on service emit methods and route structure concurrently — it just needs the import names to be correct. Both agents should use the exact class names specified above to avoid conflicts.

Agent B should be started slightly after Agent A, or should be instructed to create any needed import stubs if the actual classes aren't yet available. In practice, both can work concurrently since the file boundaries are clean.
