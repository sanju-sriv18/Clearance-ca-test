# Microservice Isolation — Multi-Repo Ready

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Achieve true microservice isolation where each domain can be extracted into its own repository with only shared contracts and infrastructure as dependencies. Zero monolith imports. Own database. Communication only through events.

**Architecture:** Each of the 15 domain services becomes fully self-contained. Two shared packages (`contracts/` for event schemas and DTOs, `infra/` for event bus, database helpers, idempotency) are the ONLY cross-cutting dependencies. Cross-domain data needs are satisfied by event-driven denormalization (local read models) rather than direct database queries. The monolith (`app/`) is completely decoupled — no importlib, no facades, no shared ORM models.

**Tech Stack:** Python 3.12, FastAPI, SQLAlchemy 2.0 async, NATS JetStream, Pydantic v2, PostgreSQL 16

**Litmus Test:** At the end, a test verifies that each domain directory can be copied to a fresh Python environment with only `contracts/` and `infra/` installed, and all its imports resolve. If this test fails, isolation is fake.

---

## Current State (Why This Plan Exists)

The "microservices" are a Potemkin village:
- **36 files** in `clearance_platform/` import from `app/*` via `importlib.import_module()`, passing static analysis but requiring the full monolith at runtime
- **`shared/domain_models.py`** aggregates ORM models from ALL 15 domains — services query each other's tables directly
- **All containers** connect to the same `clearance_engine` database (schema-level "isolation" only)
- **Event class imports** go directly across domain boundaries (`from clearance_platform.domains.X.events import Y`)

This plan eliminates ALL of these couplings.

---

## Phase Overview

| Phase | What | Validation |
|-------|------|------------|
| 0 | Create `contracts/` package, move all event definitions there | All existing tests pass with new import paths |
| 1 | Create `infra/` package, move event bus + DB helpers + idempotency there | All existing tests pass with new import paths |
| 2 | Eliminate cross-domain model queries (`domain_models.py`) | `domain_models.py` deleted; no domain imports another domain's models |
| 3 | Eliminate monolith dependencies (engines, services, legacy facades) | Zero `app.*` imports in `clearance_platform/`; `grep -r "import_module.*app\." clearance_platform/` returns nothing |
| 4 | Database isolation — per-domain connection config | Each domain's `main.py` connects to its own database URL |
| 5 | Litmus test — verify multi-repo readiness | Automated test proves each domain runs with only contracts+infra |

---

## Phase 0: Shared Contracts Package

**Goal:** Move all event definitions and shared DTOs out of domain directories into a standalone `contracts/` package that any service can depend on without importing domain code.

### Task 0.1: Create contracts package structure

**Files:**
- Create: `clearance_platform/contracts/__init__.py`
- Create: `clearance_platform/contracts/events/__init__.py`
- Create: `clearance_platform/contracts/events/base.py`
- Create: `clearance_platform/contracts/streams.py`
- Create: `clearance_platform/contracts/schemas/__init__.py`

**Step 1: Create the directory structure**

```bash
mkdir -p clearance-engine/backend/clearance_platform/contracts/events
mkdir -p clearance-engine/backend/clearance_platform/contracts/schemas
```

**Step 2: Move event base classes**

Copy `clearance_platform/shared/events/base.py` → `clearance_platform/contracts/events/base.py` (identical content).

```python
# clearance_platform/contracts/events/base.py
# Exact copy of shared/events/base.py — BaseEvent and EventMetadata
```

**Step 3: Move stream definitions**

Copy `clearance_platform/shared/streams.py` → `clearance_platform/contracts/streams.py` (identical content).

**Step 4: Create contracts __init__.py**

```python
# clearance_platform/contracts/__init__.py
"""Shared contracts package — event schemas, stream definitions, DTOs.

This package is the ONLY shared dependency between domain services.
It contains NO business logic, NO ORM models, NO service code.
"""
```

**Step 5: Create contracts/events/__init__.py**

```python
# clearance_platform/contracts/events/__init__.py
from .base import BaseEvent, EventMetadata

__all__ = ["BaseEvent", "EventMetadata"]
```

**Step 6: Run existing tests to verify nothing breaks**

```bash
cd clearance-engine/backend && python -m pytest tests/ -x -q --timeout=60
```

Expected: All tests pass (we only added files, didn't change anything).

**Step 7: Commit**

```bash
git add clearance_platform/contracts/
git commit -m "feat: create contracts package structure for event schemas"
```

---

### Task 0.2: Move domain event definitions to contracts (per domain)

**Pattern:** For each of the 15 domains, copy its `events.py` into `contracts/events/{domain}.py`. Then update the domain's `events.py` to re-export from contracts (backward compatibility during migration).

**Files (per domain):**
- Create: `clearance_platform/contracts/events/{domain_name}.py`
- Modify: `clearance_platform/domains/{domain_name}/events.py`

**Do this for all 15 domains in order:**
1. shipment_lifecycle
2. declaration_management
3. cargo_handling_units
4. trade_intelligence
5. compliance
6. consolidation
7. customs_adjudication
8. order_management
9. document_management
10. exception_management
11. financial_settlement
12. party_management
13. product_catalog
14. regulatory_intelligence
15. supply_chain_disruptions

**Step 1: Copy event definitions to contracts (example: shipment_lifecycle)**

Create `clearance_platform/contracts/events/shipment_lifecycle.py`:

```python
"""Shipment Lifecycle event contracts.

These are the canonical event definitions. Domain code re-exports
from here for backward compatibility during migration.
"""
from __future__ import annotations

from typing import Any
from uuid import UUID

from pydantic import Field

from clearance_platform.contracts.events.base import BaseEvent

# ... paste full content of domains/shipment_lifecycle/events.py
# but change the import from:
#   from clearance_platform.shared.events.base import BaseEvent
# to:
#   from clearance_platform.contracts.events.base import BaseEvent
```

**Step 2: Update domain events.py to re-export from contracts**

Replace `clearance_platform/domains/shipment_lifecycle/events.py` with:

```python
"""Shipment Lifecycle domain events — re-exported from contracts.

Canonical definitions live in clearance_platform.contracts.events.shipment_lifecycle.
This file re-exports for backward compatibility during migration.
"""
from clearance_platform.contracts.events.shipment_lifecycle import (  # noqa: F401
    ShipmentCreatedEvent,
    ShipmentStatusChangedEvent,
    ShipmentAnalyzedEvent,
    ShipmentDeliveredEvent,
)
```

**Step 3: Run tests after each domain migration**

```bash
cd clearance-engine/backend && python -m pytest tests/ -x -q --timeout=60
```

Expected: All tests pass (re-exports preserve existing import paths).

**Step 4: Commit after each domain (or batch of 3-4 domains)**

```bash
git commit -m "feat: move shipment_lifecycle events to contracts package"
```

**Repeat Steps 1-4 for all 15 domains.**

---

### Task 0.3: Move event registry to contracts

**Files:**
- Create: `clearance_platform/contracts/events/registry.py`
- Modify: `clearance_platform/shared/events/registry.py`

**Step 1: Create contracts registry**

Copy `shared/events/registry.py` → `contracts/events/registry.py`, updating ALL imports to use `contracts.events.*` instead of `domains.*.events`:

```python
# In _ensure_registry(), change ALL imports like:
#   from clearance_platform.domains.cargo_handling_units.events import ...
# to:
#   from clearance_platform.contracts.events.cargo_handling_units import ...
```

**Step 2: Update shared registry to delegate to contracts**

```python
# clearance_platform/shared/events/registry.py
"""Event registry — delegates to contracts.events.registry.

Kept for backward compatibility during migration.
"""
from clearance_platform.contracts.events.registry import (  # noqa: F401
    register_event,
    get_event_class,
    get_event_class_optional,
    all_registered,
)
```

**Step 3: Run tests**

```bash
cd clearance-engine/backend && python -m pytest tests/ -x -q --timeout=60
```

**Step 4: Commit**

```bash
git commit -m "feat: move event registry to contracts package"
```

---

### Task 0.4: Move shared schemas to contracts

**Files:**
- Copy: `clearance_platform/shared/schemas/*` → `clearance_platform/contracts/schemas/*`
- Modify: `clearance_platform/shared/schemas/*.py` (re-export from contracts)

**Step 1: Copy all schema files**

```bash
cp clearance-engine/backend/clearance_platform/shared/schemas/*.py \
   clearance-engine/backend/clearance_platform/contracts/schemas/
```

**Step 2: Verify schemas have no ORM imports**

```bash
grep -r "sqlalchemy\|from.*models import" clearance_platform/contracts/schemas/
```

Expected: No matches (schemas should be pure Pydantic).

If any ORM imports found, remove them and replace with Pydantic equivalents.

**Step 3: Update shared/schemas to re-export from contracts**

**Step 4: Run tests, commit**

---

### Task 0.5: Move shared constants to contracts

**Files:**
- Copy: `clearance_platform/shared/constants/` → `clearance_platform/contracts/constants/`
- Modify: `clearance_platform/shared/constants/__init__.py` (re-export)

**Step 1-4: Same pattern as Task 0.4**

---

### Phase 0 Validation Gate

**Run this validation before proceeding to Phase 1:**

```bash
# 1. All tests still pass
cd clearance-engine/backend && python -m pytest tests/ -x -q --timeout=120

# 2. Contracts package has NO imports from domains/* (except re-exports in domain events.py)
grep -r "from clearance_platform.domains" clearance_platform/contracts/ | grep -v "__pycache__"
# Expected: ZERO matches

# 3. Contracts package has NO imports from app/*
grep -r "from app\.\|import app\.\|import_module.*app" clearance_platform/contracts/ | grep -v "__pycache__"
# Expected: ZERO matches

# 4. Contracts package has NO SQLAlchemy imports
grep -r "sqlalchemy" clearance_platform/contracts/ | grep -v "__pycache__"
# Expected: ZERO matches
```

**Commit phase completion:**
```bash
git commit -m "feat: phase 0 complete — contracts package with all event definitions"
```

---

## Phase 1: Shared Infrastructure Package

**Goal:** Move event bus, database helpers, and idempotency into a standalone `infra/` package that has no domain or monolith dependencies.

### Task 1.1: Create infra package with event bus

**Files:**
- Create: `clearance_platform/infra/__init__.py`
- Create: `clearance_platform/infra/event_bus.py`
- Create: `clearance_platform/infra/idempotency.py`
- Create: `clearance_platform/infra/database.py`

**Step 1: Create directory**

```bash
mkdir -p clearance-engine/backend/clearance_platform/infra
```

**Step 2: Copy event_bus.py**

Copy `shared/event_bus.py` → `infra/event_bus.py`. The event bus should depend ONLY on:
- `nats` (NATS client library)
- `redis` (optional fallback)
- `pydantic` (for event serialization)
- `clearance_platform.contracts.events.base` (for BaseEvent type)
- `clearance_platform.contracts.streams` (for StreamConfig)

Audit the file. If it imports from `app.*`, `shared/services/*`, or `domains/*`, those imports must be removed or replaced.

**Step 3: Copy idempotency.py**

Copy `shared/idempotency.py` → `infra/idempotency.py`. This file only depends on `sqlalchemy` — already clean.

**Step 4: Copy database.py**

Copy `shared/database.py` → `infra/database.py`. This file only depends on `sqlalchemy` and `os` — already clean.

**Step 5: Create infra __init__.py**

```python
# clearance_platform/infra/__init__.py
"""Shared infrastructure — event bus, database, idempotency.

This package has NO business logic, NO domain models, NO monolith dependencies.
It depends only on: contracts, sqlalchemy, nats, redis, pydantic.
"""
```

**Step 6: Update shared modules to delegate to infra**

```python
# clearance_platform/shared/event_bus.py — re-export from infra
from clearance_platform.infra.event_bus import *  # noqa: F401,F403
```

(Same pattern for `shared/idempotency.py` and `shared/database.py`)

**Step 7: Run tests, commit**

---

### Task 1.2: Move circuit breaker and rate limiting to infra

**Files:**
- Copy: `clearance_platform/shared/circuit_breaker.py` → `clearance_platform/infra/circuit_breaker.py`
- Copy: `clearance_platform/shared/rate_limits.py` → `clearance_platform/infra/rate_limits.py`

**Same delegation pattern as Task 1.1.**

---

### Phase 1 Validation Gate

```bash
# 1. All tests pass
cd clearance-engine/backend && python -m pytest tests/ -x -q --timeout=120

# 2. Infra package has NO domain imports
grep -r "from clearance_platform.domains" clearance_platform/infra/ | grep -v "__pycache__"
# Expected: ZERO

# 3. Infra package has NO monolith imports
grep -r "from app\.\|import app\.\|import_module.*app" clearance_platform/infra/ | grep -v "__pycache__"
# Expected: ZERO

# 4. Infra imports ONLY from contracts (not shared)
grep -r "from clearance_platform.shared" clearance_platform/infra/ | grep -v "__pycache__"
# Expected: ZERO
```

---

## Phase 2: Eliminate Cross-Domain Model Queries

**Goal:** Delete `shared/domain_models.py`. Every place a domain currently queries another domain's ORM models must be replaced with one of:
- **Local denormalized columns** (data replicated via events)
- **HTTP API calls** to the owning service
- **Read models/projections** maintained by event consumers

This is the hardest phase. Each domain is a separate task.

### Task 2.0: Catalog all cross-domain query sites

Before changing code, create an exhaustive map of every file that imports from `domain_models.py` and what data it actually needs. This determines the replacement strategy.

**Step 1: Find all import sites**

```bash
grep -rn "from clearance_platform.shared.domain_models import\|from clearance_platform.shared.domain_models" \
  clearance-engine/backend/clearance_platform/ | grep -v __pycache__
```

**Step 2: For each site, document:**
- What models are imported
- What query is performed (SELECT, JOIN, etc.)
- What data is actually needed by the consumer
- Replacement strategy: denormalize, API call, or projection

**Step 3: Write findings to a tracking file**

Create `docs/plans/2026-02-27-cross-domain-query-elimination.md` with the full inventory.

---

### Task 2.1: Add denormalized columns to declaration_management

**Context:** `declaration_management/routes.py` imports `Shipment, Order, OrderLineItem, Consolidation, Document, HandlingUnit` from `domain_models.py`. It uses these for the broker work queue (joining shipment data with entry filings) and entry detail views.

**Replacement strategy:** Declaration management's `EntryFiling` model stores a JSONB `shipment_snapshot` column that is populated by events. When `ShipmentCreatedEvent` or `ShipmentStatusChangedEvent` arrives, the consumer stores the relevant shipment data as a denormalized snapshot.

**Step 1: Write the failing test**

```python
# tests/domains/test_declaration_denormalization.py
import pytest
from clearance_platform.domains.declaration_management.models import EntryFiling

def test_entry_filing_has_shipment_snapshot():
    """EntryFiling must have a shipment_snapshot JSONB column for denormalized data."""
    assert hasattr(EntryFiling, "shipment_snapshot")
```

**Step 2: Run test to verify it fails**

```bash
cd clearance-engine/backend && python -m pytest tests/domains/test_declaration_denormalization.py -v
```
Expected: FAIL

**Step 3: Add the column to EntryFiling model**

In `clearance_platform/domains/declaration_management/models.py`, add:

```python
shipment_snapshot: Mapped[dict | None] = mapped_column(
    JSONB, nullable=True, default=None,
    comment="Denormalized shipment data populated by ShipmentCreatedEvent"
)
```

**Step 4: Run test to verify it passes**

**Step 5: Update consumer to populate snapshot**

In `declaration_management/consumers.py`, when handling `ShipmentCreatedEvent`:
```python
# Store denormalized shipment data on related entry filings
snapshot = {
    "product": data.get("product"),
    "origin": data.get("origin"),
    "destination": data.get("destination"),
    "status": data.get("status"),
    "carrier": data.get("carrier"),
    "company_name": data.get("company_name"),
    "transport_mode": data.get("transport_mode"),
    "declared_value": data.get("declared_value"),
    "hs_code": data.get("hs_code"),
    "house_number": data.get("house_number"),
    # ... all fields the routes.py currently queries
}
```

**Step 6: Rewrite routes.py queries to use snapshot instead of JOINs**

Replace:
```python
# OLD: Cross-domain join
result = await session.execute(
    select(EntryFiling, Shipment).join(Shipment, ...)
)
```

With:
```python
# NEW: Use denormalized snapshot
result = await session.execute(select(EntryFiling).where(...))
for filing in result.scalars():
    shipment_data = filing.shipment_snapshot or {}
```

**Step 7: Remove `Shipment` import from routes.py**

**Step 8: Run full test suite, commit**

---

### Task 2.2-2.N: Repeat for each cross-domain query site

Each domain that imports from `domain_models.py` gets its own task following the same pattern:

**Task 2.2:** `document_management` — needs Shipment and Order data for document requirements
- Add `shipment_snapshot` and `order_snapshot` JSONB to Document model
- Populate via events

**Task 2.3:** `customs_adjudication` — needs EntryFiling and Shipment data
- Add `entry_snapshot` and `shipment_snapshot` JSONB columns
- Populate via DeclarationSubmittedEvent

**Task 2.4:** `financial_settlement` — needs Shipment data for fee calculation
- Add `shipment_snapshot` JSONB
- Populate via ShipmentCreatedEvent

**Task 2.5:** `exception_management` — needs Shipment data
- Add `shipment_snapshot` JSONB
- Populate via event

**Task 2.6:** `consolidation` — needs Shipment data
- Already has shipment_ids; add snapshot JSONB
- Populate via ShipmentCreatedEvent

**Task 2.7:** `shipment_lifecycle` — needs Document and Order data
- Add `order_snapshot` JSONB to Shipment model
- Populate via OrderCreatedEvent

**Task 2.8:** `order_management` — needs Shipment data
- Add `shipment_snapshot` JSONB to Order model
- Populate via ShipmentCreatedEvent

**Task 2.9:** Gateway routes (`platform_routes.py`, `hu_metrics_routes.py`) — need cross-domain data
- Gateway is a special case: it's an aggregation layer
- Replace direct model imports with HTTP calls to domain APIs
- Gateway routes call domain REST endpoints, not domain models

---

### Task 2.FINAL: Delete domain_models.py

**Step 1: Verify no imports remain**

```bash
grep -rn "domain_models" clearance-engine/backend/clearance_platform/ | grep -v __pycache__
```
Expected: ZERO matches (except the file itself and re-export shims if any).

**Step 2: Delete the file**

```bash
rm clearance-engine/backend/clearance_platform/shared/domain_models.py
```

**Step 3: Run full test suite**

**Step 4: Commit**

```bash
git commit -m "feat: delete domain_models.py — all cross-domain queries eliminated"
```

---

### Phase 2 Validation Gate

```bash
# 1. domain_models.py is gone
test ! -f clearance-engine/backend/clearance_platform/shared/domain_models.py

# 2. No domain imports another domain's models
grep -rn "from clearance_platform.domains.*models import" clearance-engine/backend/clearance_platform/domains/ \
  | grep -v "__pycache__" | grep -v "from clearance_platform.domains.${OWN_DOMAIN}"
# Must filter to only show cross-domain imports. Expected: ZERO

# 3. All tests pass
cd clearance-engine/backend && python -m pytest tests/ -x -q --timeout=120
```

---

## Phase 3: Eliminate Monolith Dependencies

**Goal:** Zero imports from `app.*` anywhere in `clearance_platform/`. This means moving engine code, LLM services, simulation wiring, and legacy broker functions into the appropriate domain services.

### Task 3.1: Move classification engine (E1) into trade_intelligence domain

**Files:**
- Create: `clearance_platform/domains/trade_intelligence/engines/classification.py`
- Modify: `clearance_platform/domains/trade_intelligence/service.py`
- Delete: `clearance_platform/shared/engines/e1_classification.py`

**Step 1: Read `app/engines/e1_classification/` to understand what it does**

Identify all dependencies. The engine likely needs:
- LLM client (for HS code classification)
- Database session (for tariff schedule lookup)
- Redis (for caching)

**Step 2: Copy the engine's core logic into the domain**

Create `clearance_platform/domains/trade_intelligence/engines/classification.py` with the actual classification logic. Replace any `app.*` imports with direct dependencies (LLM client interface, DB session).

**Step 3: Update intelligence_wiring.py to use the domain engine**

Replace:
```python
from clearance_platform.shared.engines.e1_classification import classify
```
With:
```python
from clearance_platform.domains.trade_intelligence.engines.classification import classify
```

**Step 4: Delete the facade**

```bash
rm clearance-engine/backend/clearance_platform/shared/engines/e1_classification.py
```

**Step 5: Run tests, commit**

---

### Task 3.2-3.6: Move remaining engines (E2-E6)

Same pattern for each:

| Task | Engine | Target Domain |
|------|--------|---------------|
| 3.2 | E2 Tariff | trade_intelligence |
| 3.3 | E3 Compliance | compliance |
| 3.4 | E4 FTA | trade_intelligence |
| 3.5 | E5 Exception | exception_management |
| 3.6 | E6 Regulatory | regulatory_intelligence |

---

### Task 3.7: Move LLM service to infra

**Context:** `shared/llm.py` wraps `app.services.llm.LLMService`. LLM is infrastructure (like the database), not domain logic.

**Files:**
- Create: `clearance_platform/infra/llm.py`
- Delete: `clearance_platform/shared/llm.py`

**Step 1: Read `app/services/llm.py` to understand the interface**

**Step 2: Create an LLM client in infra that either:**
- Contains the actual LLM client code (if small enough)
- Defines an interface that domains can use

**Step 3: Update all consumers/services that use LLM to import from infra**

**Step 4: Delete facade, run tests, commit**

---

### Task 3.8: Move broker LLM functions into declaration_management

**Context:** `shared/legacy/broker.py` wraps 5 LLM functions from `app.api.routes.broker`: `get_briefing`, `draft_cf28_response`, `draft_communication`, `get_entry_insights`, `draft_cf29_protest`.

These are declaration_management domain logic — they should live in `declaration_management/service.py`.

**Step 1: Read the 5 functions in `app/api/routes/broker.py`**

**Step 2: Move them into `declaration_management/service.py` or a new `declaration_management/llm_functions.py`**

**Step 3: Update `declaration_management/routes.py` to call domain service instead of legacy facade**

**Step 4: Delete `shared/legacy/broker.py`, run tests, commit**

---

### Task 3.9: Move simulation code out of clearance_platform

**Context:** `shared/simulation/` wraps `app.simulation.*`. Simulation is NOT a domain service — it's an operational tool that drives the system via HTTP. It should live in `app/simulation/` only, not be wrapped by facades.

**Step 1: Identify which domain services import from `shared/simulation/`**

```bash
grep -rn "shared.simulation\|shared\.simulation" clearance_platform/domains/ | grep -v __pycache__
```

**Step 2: For each import, replace with the actual data needed**

For example, if `order_management/service.py` imports `CARRIERS_BY_MODE` from simulation reference data, either:
- Inline the constant in the domain service
- Move it to `contracts/constants/`

**Step 3: Delete entire `shared/simulation/` directory**

**Step 4: Run tests, commit**

---

### Task 3.10: Delete remaining facade directories

**Step 1: Delete all facade/wrapper directories**

```bash
rm -rf clearance-engine/backend/clearance_platform/shared/engines/
rm -rf clearance-engine/backend/clearance_platform/shared/services/
rm -rf clearance-engine/backend/clearance_platform/shared/legacy/
rm -rf clearance-engine/backend/clearance_platform/shared/simulation/
rm -rf clearance-engine/backend/clearance_platform/shared/agent/
rm -f clearance-engine/backend/clearance_platform/shared/operational_models.py
rm -f clearance-engine/backend/clearance_platform/shared/model_registry.py
```

**Step 2: Fix any import errors**

```bash
cd clearance-engine/backend && python -c "import clearance_platform"
```

**Step 3: Run full test suite, commit**

---

### Task 3.11: Move gateway monolith re-exports to domain routes

**Context:** Three gateway routes (`simulation_routes.py`, `description_quality_routes.py`, `analysis_routes.py`) just re-export `app.api.routes.*` routers via importlib.

**Option A:** Move the actual route logic into the appropriate domain service
**Option B:** Keep these as gateway routes but implement them properly (calling domain services via HTTP)

**Step 1: For each gateway route, implement it properly without monolith imports**

**Step 2: Delete importlib wrappers, run tests, commit**

---

### Task 3.12: Clean up intelligence_wiring.py

**Context:** `intelligence_wiring.py` imports engines from `shared/engines/*` (now deleted) and event classes from domains. Update all imports to use:
- Engine code from the owning domain
- Event classes from `contracts/events/*`

**Step 1: Update all imports in intelligence_wiring.py**

**Step 2: Run tests, commit**

---

### Phase 3 Validation Gate

```bash
# THE CRITICAL TEST: Zero monolith imports in clearance_platform
grep -rn "import_module.*app\.\|from app\.\|import app\." \
  clearance-engine/backend/clearance_platform/ | grep -v __pycache__
# Expected: ZERO matches

# No shared/engines, shared/services, shared/legacy, shared/simulation directories
test ! -d clearance-engine/backend/clearance_platform/shared/engines
test ! -d clearance-engine/backend/clearance_platform/shared/services
test ! -d clearance-engine/backend/clearance_platform/shared/legacy
test ! -d clearance-engine/backend/clearance_platform/shared/simulation
test ! -d clearance-engine/backend/clearance_platform/shared/agent

# No operational_models.py or model_registry.py
test ! -f clearance-engine/backend/clearance_platform/shared/operational_models.py
test ! -f clearance-engine/backend/clearance_platform/shared/model_registry.py

# All tests pass
cd clearance-engine/backend && python -m pytest tests/ -x -q --timeout=120
```

---

## Phase 4: Database Isolation

**Goal:** Each domain connects to its own database (or at minimum, its own PostgreSQL database within the same cluster). No cross-schema queries possible.

### Task 4.1: Update database.py for per-domain DATABASE_URL

**Files:**
- Modify: `clearance_platform/infra/database.py`

**Step 1: Change `get_domain_database_url()` to read `{DOMAIN}_DATABASE_URL`**

```python
def get_domain_database_url(domain: str) -> str:
    """Get the database URL for a specific domain.

    Each domain can have its own DATABASE_URL via environment variable:
      SHIPMENT_DATABASE_URL, DECLARATION_DATABASE_URL, etc.

    Falls back to shared DATABASE_URL if domain-specific URL not set.
    """
    schema = get_schema_name(domain)
    env_key = f"{schema.upper()}_DATABASE_URL"
    domain_url = os.environ.get(env_key)
    if domain_url:
        return domain_url
    # Fallback to shared URL (monolith/dev mode)
    return os.environ.get("DATABASE_URL", _DEFAULT_DATABASE_URL)
```

**Step 2: Remove search_path hack**

The `search_path: "{schema},public"` connect_arg is a coupling mechanism. With true isolation, each domain owns ALL tables in its database — no schema prefixing needed.

```python
def create_domain_engine(domain: str) -> AsyncEngine:
    url = get_domain_database_url(domain)
    return create_async_engine(url, pool_size=5, max_overflow=10)
```

**Step 3: Run tests, commit**

---

### Task 4.2: Update docker-compose.microservices.yml

**Step 1: Add per-domain database services (or per-domain databases on shared PostgreSQL)**

For development, use a single PostgreSQL instance with separate databases:

```yaml
postgres:
  image: postgres:16-alpine
  environment:
    POSTGRES_MULTIPLE_DATABASES: >
      shipment_db,declaration_db,order_db,cargo_db,compliance_db,
      consolidation_db,trade_intel_db,adjudication_db,exception_db,
      financial_db,regulatory_db,document_db,party_db,product_db,
      disruption_db
```

**Step 2: Update each domain service's environment**

```yaml
svc-shipment:
  environment:
    SHIPMENT_DATABASE_URL: postgresql+asyncpg://svc_shipment:pw@postgres:5432/shipment_db
```

**Step 3: Create init script that creates multiple databases**

**Step 4: Run integration test, commit**

---

### Task 4.3: Per-domain Alembic migrations

**Step 1: Create migration config per domain**

Each domain gets its own `alembic.ini` and `migrations/` directory. In multi-repo, these would be in each repo. For now, they live at `clearance_platform/domains/{domain}/migrations/`.

**Step 2: Generate initial migration per domain from its models.py**

**Step 3: Verify migrations run independently**

---

### Phase 4 Validation Gate

```bash
# Each domain can connect to its own database
# Test: Start each domain service with its own DATABASE_URL
# Verify: No cross-database queries possible

# All tests pass
cd clearance-engine/backend && python -m pytest tests/ -x -q --timeout=120
```

---

## Phase 5: Update Consumer Event Imports

**Goal:** All consumers import event classes from `contracts/events/*`, not from other `domains/*/events.py`.

### Task 5.1: Update all consumer imports (per domain)

**For each of the 15 domains' `consumers.py`:**

Replace:
```python
from clearance_platform.domains.cargo_handling_units.events import HUCreatedEvent
```

With:
```python
from clearance_platform.contracts.events.cargo_handling_units import HUCreatedEvent
```

**Do this for all 32 cross-domain event imports identified in the audit.**

### Phase 5 Validation Gate

```bash
# No consumer imports events from another domain directly
grep -rn "from clearance_platform.domains.*events import" \
  clearance-engine/backend/clearance_platform/domains/*/consumers.py \
  | grep -v __pycache__ \
  | grep -v "from clearance_platform.domains.\${OWN_DOMAIN}"
# For each file, only imports from its OWN domain or from contracts are allowed
# Cross-domain event imports must come from contracts.events.*

# All tests pass
cd clearance-engine/backend && python -m pytest tests/ -x -q --timeout=120
```

---

## Phase 6: The Litmus Test — Multi-Repo Readiness

**Goal:** Prove that each domain can be extracted into its own repository.

### Task 6.1: Write the multi-repo readiness test

**Files:**
- Create: `tests/architecture/test_multirepo_readiness.py`

```python
"""Multi-repo readiness test.

For each domain, verify that its ONLY imports from clearance_platform are:
  1. clearance_platform.contracts.* (event schemas, DTOs, constants)
  2. clearance_platform.infra.* (event bus, database, idempotency)
  3. clearance_platform.domains.{OWN_DOMAIN}.* (own code)

If this test passes, any domain can be extracted into its own repository
by adding contracts/ and infra/ as pip-installable dependencies.
"""
import ast
import pathlib
import pytest

BACKEND = pathlib.Path(__file__).resolve().parents[2]
DOMAINS_DIR = BACKEND / "clearance_platform" / "domains"

ALLOWED_PREFIXES = {
    "clearance_platform.contracts",
    "clearance_platform.infra",
}

def _get_all_imports(filepath: pathlib.Path) -> list[str]:
    """Extract all import strings from a Python file."""
    source = filepath.read_text()
    tree = ast.parse(source)
    imports = []
    for node in ast.walk(tree):
        if isinstance(node, ast.ImportFrom) and node.module:
            imports.append(node.module)
        elif isinstance(node, ast.Import):
            for alias in node.names:
                imports.append(alias.name)
    return imports

def _get_domains() -> list[str]:
    return [d.name for d in DOMAINS_DIR.iterdir()
            if d.is_dir() and not d.name.startswith("_")]

@pytest.mark.parametrize("domain", _get_domains())
def test_domain_imports_only_contracts_and_infra(domain: str):
    """Each domain must only import from contracts, infra, or its own package."""
    domain_dir = DOMAINS_DIR / domain
    own_prefix = f"clearance_platform.domains.{domain}"

    violations = []
    for pyfile in domain_dir.rglob("*.py"):
        for imp in _get_all_imports(pyfile):
            if not imp.startswith("clearance_platform"):
                continue  # third-party or stdlib — allowed
            if imp.startswith(own_prefix):
                continue  # own domain — allowed
            if any(imp.startswith(p) for p in ALLOWED_PREFIXES):
                continue  # contracts or infra — allowed
            violations.append(f"{pyfile.relative_to(BACKEND)}:{imp}")

    assert not violations, (
        f"Domain '{domain}' has forbidden imports:\n"
        + "\n".join(f"  {v}" for v in violations)
    )

@pytest.mark.parametrize("domain", _get_domains())
def test_domain_has_no_monolith_imports(domain: str):
    """No domain may import from app.* (the monolith)."""
    domain_dir = DOMAINS_DIR / domain
    violations = []
    for pyfile in domain_dir.rglob("*.py"):
        source = pyfile.read_text()
        # Check for importlib.import_module("app.
        if 'import_module("app.' in source or "import_module('app." in source:
            violations.append(f"{pyfile.relative_to(BACKEND)}: importlib app import")
        for imp in _get_all_imports(pyfile):
            if imp.startswith("app."):
                violations.append(f"{pyfile.relative_to(BACKEND)}: {imp}")

    assert not violations, (
        f"Domain '{domain}' has monolith imports:\n"
        + "\n".join(f"  {v}" for v in violations)
    )

@pytest.mark.parametrize("domain", _get_domains())
def test_domain_has_no_cross_domain_model_imports(domain: str):
    """No domain may import ORM models from another domain."""
    domain_dir = DOMAINS_DIR / domain
    violations = []
    for pyfile in domain_dir.rglob("*.py"):
        for imp in _get_all_imports(pyfile):
            # Check for imports from other domains' models
            if (imp.startswith("clearance_platform.domains.")
                and ".models" in imp
                and not imp.startswith(f"clearance_platform.domains.{domain}")):
                violations.append(f"{pyfile.relative_to(BACKEND)}: {imp}")

    assert not violations, (
        f"Domain '{domain}' imports models from other domains:\n"
        + "\n".join(f"  {v}" for v in violations)
    )

def test_contracts_has_no_domain_imports():
    """Contracts package must not import from any domain."""
    contracts_dir = BACKEND / "clearance_platform" / "contracts"
    if not contracts_dir.exists():
        pytest.skip("contracts/ not yet created")
    violations = []
    for pyfile in contracts_dir.rglob("*.py"):
        for imp in _get_all_imports(pyfile):
            if imp.startswith("clearance_platform.domains"):
                violations.append(f"{pyfile.relative_to(BACKEND)}: {imp}")
    assert not violations

def test_infra_has_no_domain_imports():
    """Infra package must not import from any domain."""
    infra_dir = BACKEND / "clearance_platform" / "infra"
    if not infra_dir.exists():
        pytest.skip("infra/ not yet created")
    violations = []
    for pyfile in infra_dir.rglob("*.py"):
        for imp in _get_all_imports(pyfile):
            if imp.startswith("clearance_platform.domains"):
                violations.append(f"{pyfile.relative_to(BACKEND)}: {imp}")
    assert not violations

def test_no_domain_models_aggregator():
    """shared/domain_models.py must not exist."""
    assert not (BACKEND / "clearance_platform" / "shared" / "domain_models.py").exists(), \
        "domain_models.py still exists — cross-domain model coupling not eliminated"

def test_no_monolith_facades():
    """No facade directories should remain in shared/."""
    forbidden = [
        "shared/engines",
        "shared/services",
        "shared/legacy",
        "shared/simulation",
        "shared/agent",
    ]
    for path in forbidden:
        full = BACKEND / "clearance_platform" / path
        assert not full.exists(), f"Facade directory still exists: {path}"
```

**Step 1: Write the test file**

**Step 2: Run it — it WILL fail on the current codebase (this is expected)**

```bash
cd clearance-engine/backend && python -m pytest tests/architecture/test_multirepo_readiness.py -v
```

This test becomes the **north star** for all previous phases. Every phase should bring more of these tests from FAIL to PASS.

**Step 3: Commit the test**

```bash
git commit -m "test: add multi-repo readiness litmus test (initially failing)"
```

---

### Task 6.2: Create extraction simulation script

**Files:**
- Create: `scripts/test_domain_extraction.py`

```python
"""Simulate extracting each domain into its own repo.

For each domain:
1. Create a temporary virtualenv
2. Install only contracts/ and infra/ as packages
3. Attempt to import the domain's main module
4. Report success or failure with missing dependencies
"""
```

This is the ultimate validation — not just import analysis but actual runtime verification.

---

### Phase 6 Validation Gate

```bash
# All litmus tests pass
cd clearance-engine/backend && python -m pytest tests/architecture/test_multirepo_readiness.py -v

# Full test suite passes
cd clearance-engine/backend && python -m pytest tests/ -x -q --timeout=120

# Extraction simulation passes
cd clearance-engine/backend && python scripts/test_domain_extraction.py
```

---

## Execution Order & Dependencies

```
Phase 0 (contracts) ──→ Phase 1 (infra) ──→ Phase 2 (cross-domain queries)
                                                       │
Phase 5 (consumer imports) ←──────────────────────────┘
                                                       │
                          Phase 3 (monolith deps) ←────┘
                                                       │
                          Phase 4 (database isolation) ←┘
                                                       │
                          Phase 6 (litmus test) ←──────┘
```

**Phase 0 and Phase 1** can be done first — they're additive (create new packages, don't remove anything).

**Task 6.1 (write the litmus test)** should be done IMMEDIATELY — it's the north star that guides all work.

**Phase 2 and Phase 3** are the hard work and can be partially parallelized (different domains can be worked on simultaneously by different agents).

**Phase 4** depends on Phases 2+3 being complete.

**Phase 5** can be done alongside Phase 2 (updating consumer imports as events move to contracts).

---

## Risk Mitigation

1. **Each task has its own validation** — tests must pass after every task
2. **Re-export shims** provide backward compatibility during migration — old import paths keep working while we incrementally update callers
3. **The litmus test exists from Day 1** — we can always measure progress
4. **Cross-domain query elimination is the riskiest phase** — break it into one task per domain, validate each independently
5. **Engine migration is complex but well-scoped** — each engine maps to exactly one domain

## What Success Looks Like

```
clearance_platform/
├── contracts/          # Shared event schemas, DTOs, constants
│   ├── events/         # All 42 event types defined here
│   ├── schemas/        # Shared Pydantic DTOs
│   ├── constants/      # CBP fees, etc.
│   └── streams.py      # NATS stream definitions
├── infra/              # Shared infrastructure
│   ├── event_bus.py    # DomainEventBus
│   ├── database.py     # Per-domain DB helpers
│   ├── idempotency.py  # Event dedup
│   └── llm.py          # LLM client interface
├── domains/            # 15 fully isolated services
│   └── {domain}/
│       ├── models.py       # OWN tables only
│       ├── service.py      # Business logic (no cross-domain queries)
│       ├── routes.py       # API endpoints
│       ├── consumers.py    # Event handlers (imports from contracts only)
│       ├── events.py       # Re-exports from contracts (for convenience)
│       └── main.py         # Independent entry point
└── gateway/            # API gateway (routes to domain APIs, no model imports)
```

Each `domains/{domain}/` can be `cp`'d into its own repo. Add `contracts/` and `infra/` as pip dependencies. Done.
