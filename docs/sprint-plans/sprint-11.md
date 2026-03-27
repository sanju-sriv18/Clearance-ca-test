# Sprint 11: Entity Model Completion

**Goal:** Fill all 5 skeleton domain models with full SQLAlchemy ORM classes mapping to existing DB tables. Fix NATS stream naming mismatch.

**Acceptance:** All 957+ tests pass, frontend builds, all new models import cleanly.

---

## Task Assignments

### Agent A: Financial + Party Models

**Files owned:**
- `clearance_platform/domains/financial_settlement/models.py`
- `clearance_platform/domains/party_management/models.py`
- `tests/domains/test_financial_models.py` (new)
- `tests/domains/test_party_models.py` (new)

**Instructions:**

1. **FinancialRecord** — map to `financial.financial_records`:
   - id (UUID pk), shipment_id (UUID), declaration_id (UUID nullable), phase (varchar20 default 'estimated')
   - duty_total (numeric 15,2), duty_line_items (JSONB), adcvd_cash_deposit (numeric), adcvd_deposit_rate (numeric), adcvd_bond_id (UUID nullable)
   - mpf, hmf, exam_fees, broker_fees, terminal_fees (all numeric 15,2), other_fees (JSONB)
   - pms_enrollment (bool), pms_statement_period (varchar10), pms_payment_due (timestamptz)
   - liquidation_status (varchar30 default 'pending'), liquidation_due (timestamptz), liquidation_extensions (JSONB), deemed_liquidated_date (timestamptz)
   - bond_id (UUID nullable), bond_type (varchar20), bond_sufficiency (varchar20), bond_amount (numeric 15,2)
   - demurrage (JSONB)
   - created_at, updated_at (TimestampMixin)
   - `__table_args__ = {"schema": "financial"}`

2. **Bond** — map to `financial.bonds`:
   - id (UUID pk), importer_id (UUID), bond_type (varchar20), bond_number (varchar50), surety_code (varchar20)
   - amount (numeric 15,2), effective_date (date), expiration_date (date nullable)
   - status (varchar20 default 'active'), utilization (numeric 15,2 default 0), sufficiency_threshold (numeric 15,2 nullable)
   - created_at (TimestampMixin — but note: table only has created_at, no updated_at)

3. **DutyDrawback** — map to `financial.duty_drawbacks`:
   - id (UUID pk), original_entry_id (UUID), export_entry_id (UUID nullable), drawback_type (varchar30)
   - hs_code (varchar20 nullable), original_duty_paid (numeric 15,2), refund_amount (numeric 15,2)
   - status (varchar20 default 'draft'), filed_at (timestamptz nullable), approved_at (timestamptz nullable)
   - created_at (only)

4. **Reconciliation** — map to `financial.reconciliations`:
   - id (UUID pk), declaration_id (UUID), adjustment_type (varchar40)
   - original_amount, adjusted_amount, difference (all numeric 15,2)
   - reason (text nullable), status (varchar20 default 'pending'), effective_date (date)
   - created_at (only)

5. **Party** — map to `party.parties`:
   - id (UUID pk), name (varchar255), type (varchar30)
   - jurisdiction_identifiers (JSONB nullable), trust_programs (JSONB nullable)
   - compliance_score (numeric 3,2 nullable), carrier_affiliation (varchar50 nullable)
   - is_integrated_carrier_broker (bool default false), entity_state (varchar10 default 'active')
   - created_at, updated_at (TimestampMixin)

6. **PowerOfAttorney** — map to `party.power_of_attorney`:
   - id (UUID pk), importer_id (UUID FK→parties), broker_id (UUID FK→parties)
   - poa_type (varchar20 default 'continuous'), status (varchar20 default 'active')
   - jurisdiction (varchar10), granted_at (timestamptz), revoked_at (timestamptz nullable), expires_at (timestamptz nullable)
   - created_at (only)

7. **ImporterOfRecord** — map to `party.importers_of_record`:
   - id (UUID pk), party_id (UUID FK→parties), ior_type (varchar30 default 'self')
   - bond_id (UUID nullable), jurisdiction (varchar10), status (varchar20 default 'active')
   - created_at, updated_at (TimestampMixin)

8. Write unit tests: model instantiation, field types, schema assignment.

---

### Agent B: Adjudication + Exception + Cargo Models

**Files owned:**
- `clearance_platform/domains/customs_adjudication/models.py`
- `clearance_platform/domains/exception_management/models.py`
- `clearance_platform/domains/cargo_handling_units/models.py`
- `clearance_platform/shared/streams.py` (NATS fix only)
- `tests/domains/test_adjudication_models.py` (new)
- `tests/domains/test_exception_models.py` (new)
- `tests/domains/test_cargo_models.py` (new)

**Instructions:**

1. **AdjudicationDecision** — map to `adjudication.adjudication_decisions`:
   - id (UUID pk), declaration_id (UUID), handling_unit_id (UUID nullable), consolidation_id (UUID nullable)
   - jurisdiction (varchar10), decision_type (varchar30), decision_details (JSONB nullable)
   - hold_type (varchar30 nullable), response_deadline (timestamptz nullable)
   - exam_type (varchar20 nullable), decided_at (timestamptz default now()), decided_by (varchar100 nullable)
   - created_at (only, via server_default=func.now())
   - `__table_args__ = {"schema": "adjudication"}`

2. **Exception** — map to `exception.exceptions`:
   - id (UUID pk), handling_unit_id (UUID nullable), consolidation_id (UUID nullable)
   - exception_type (varchar30), hold_type (varchar30 nullable)
   - status (varchar30 default 'open'), severity (varchar10 default 'medium')
   - transit_hold_detail (JSONB nullable), cage_status (JSONB nullable), resolution (JSONB nullable)
   - created_at, updated_at (TimestampMixin)
   - `__table_args__ = {"schema": "exception"}`

3. **HUTransitEvent** — map to `cargo.hu_transit_events`:
   - id (UUID pk), hu_id (UUID), facility_name (varchar255), country (varchar3)
   - city (varchar100 nullable), timestamp (timestamptz), event_type (varchar50)
   - derived_from (varchar100 nullable), created_at (only)
   - `__table_args__ = {"schema": "cargo"}`

4. **NATS streams fix** in `shared/streams.py`:
   - Rename `CLEARANCE_CARGO` to `CLEARANCE_HANDLING_UNIT`
   - Change subjects from `["clearance.cargo.>"]` to `["clearance.handling-unit.>"]`

5. Write unit tests for all 3 model classes.

---

## Verification Steps

After both agents complete:

1. `cd clearance-engine/backend && uv run pytest tests/ -v --tb=short` — all 957+ tests pass
2. `cd clearance-engine/frontend && npm run build` — zero errors
3. `uv run python -c "from clearance_platform.domains.financial_settlement.models import FinancialRecord, Bond, DutyDrawback, Reconciliation; print('OK')"` — imports work
4. `uv run python -c "from clearance_platform.domains.party_management.models import Party, PowerOfAttorney, ImporterOfRecord; print('OK')"` — imports work
5. `uv run python -c "from clearance_platform.domains.customs_adjudication.models import AdjudicationDecision; print('OK')"` — imports work
6. `uv run python -c "from clearance_platform.domains.exception_management.models import Exception as ExceptionModel; print('OK')"` — imports work (note: use alias to avoid shadowing builtin)
7. `uv run python -c "from clearance_platform.domains.cargo_handling_units.models import HUTransitEvent; print('OK')"` — imports work

---

## Patterns to Follow

```python
# Standard model pattern (from compliance/models.py):
from __future__ import annotations
from sqlalchemy import String, Index
from sqlalchemy.dialects.postgresql import JSONB, UUID as PG_UUID
from sqlalchemy.orm import Mapped, mapped_column
from clearance_platform.shared.models import Base, TimestampMixin
import uuid

class ModelName(TimestampMixin, Base):
    __tablename__ = "table_name"
    id: Mapped[uuid.UUID] = mapped_column(PG_UUID(as_uuid=True), primary_key=True, server_default=...)
    # ... columns matching DB schema exactly
    __table_args__ = (
        Index("ix_schema_table_column", "column"),
        {"schema": "schema_name"},
    )
```

**Important:** Use `server_default` for defaults, not Python-side `default`. Match the DB column types exactly. Use `Mapped[Optional[...]]` for nullable columns.
