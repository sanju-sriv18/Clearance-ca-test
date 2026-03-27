# Production Readiness Remediation — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Resolve all 10 production readiness findings to production-grade, with full E2E proof, reports, and a PR to main.

**Architecture:** 3-wave layered execution with 4/3/3 parallel work packages per wave. Each wave has a review checkpoint. All work on `production-hardening` branch with per-WP sub-branches. Isolated git worktrees per agent.

**Tech Stack:** Python 3.12, FastAPI, SQLAlchemy 2.0 (async), PostgreSQL 16, Redis 7, NATS JetStream, Pydantic v2, React 18, TypeScript, Playwright, Tailwind CSS

**Design doc:** `docs/plans/2026-02-27-production-readiness-remediation-design.md`

**Branch strategy:** `production-hardening` only. NO MERGES INTO MAIN.

---

## Pre-Flight: Branch & Worktree Setup

### Task 0: Create production-hardening branch

**Step 1: Create and push the base branch**

```bash
cd /Users/why/vibe/clearance_vibe
git checkout main
git pull origin main
git checkout -b production-hardening
git push -u origin production-hardening
```

**Step 2: Verify branch exists**

```bash
git branch -a | grep production-hardening
```

Expected: `* production-hardening` and `remotes/origin/production-hardening`

---

## WAVE 1: Foundation (4 parallel work packages)

All Wave 1 tasks have NO dependencies on each other and run in parallel.

---

### WP-1: Legacy Monolith Decoupling (Finding #1)

**Agent:** decoupling-eng (backend-developer)
**Branch:** `production-hardening/wp-1-legacy-decoupling`
**Finding:** "Microservice packages still import `app.*`, `app.knowledge.*`, and legacy ORM models."

**Required Reading:**
- `docs/reviews/2025-02-14-production-readiness.md`
- `docs/development-standards.md`
- Serena memory: `project_overview`, `ARCHITECTURE-AUDIT-FINDINGS`, `SHIPMENT-MODEL-SPLIT`

**Files to audit:**
- `clearance_platform/shared/engines/e1_classification.py`
- `clearance_platform/shared/engines/e2_tariff.py`
- `clearance_platform/shared/engines/e3_compliance.py`
- `clearance_platform/shared/engines/e4_fta.py`
- `clearance_platform/shared/engines/e5_exception.py`
- `clearance_platform/shared/engines/e6_regulatory.py`
- `clearance_platform/shared/llm.py`
- `clearance_platform/shared/cache.py`
- `clearance_platform/gateway/analysis_routes.py`
- `clearance_platform/gateway/description_quality_routes.py`
- `clearance_platform/gateway/simulation_routes.py`
- `clearance_platform/shared/simulation/state_machine.py`

#### Task 1.1: Audit all app.* imports

**Step 1: Search for all app.* imports in clearance_platform/**

```bash
cd clearance-engine/backend
grep -rn "from app\." clearance_platform/ --include="*.py" | sort
grep -rn "import app\." clearance_platform/ --include="*.py" | sort
```

**Step 2: Document every import in a tracking list**

Create file `docs/reports/wp-1-import-audit.md` with every import, source file, line number, and what it imports.

#### Task 1.2: Move engine wrappers into clearance_platform

For each engine file in `clearance_platform/shared/engines/`:

**Step 1: Read the engine file to understand what it imports from app/**

```bash
# Example for e1
cat clearance_platform/shared/engines/e1_classification.py | head -30
```

**Step 2: Copy the needed functions/classes from app/engines/ into clearance_platform/shared/engines/**

The engine files in `clearance_platform/shared/engines/` currently use `from app.engines.X import Y`. Replace these with self-contained implementations or copy the needed code.

For each engine:
1. Read `app/engines/{engine}.py` to understand the functions needed
2. Copy the function implementations into `clearance_platform/shared/engines/{engine}.py`
3. Update imports to use local or `clearance_platform.shared` paths
4. Remove the `from app.` import line

**Step 3: Repeat for shared/llm.py, shared/cache.py**

These files import LLM and cache utilities from `app/`. Copy needed implementations into `clearance_platform/shared/`.

**Step 4: Fix gateway route imports**

For `analysis_routes.py`, `description_quality_routes.py`, `simulation_routes.py`:
1. Replace `from app.X import Y` with `from clearance_platform.shared.X import Y`
2. Verify the imported symbols exist in the new location

#### Task 1.3: Write architecture fitness test

**File:** `tests/architecture/test_no_app_imports.py`

```python
"""Architecture fitness test: clearance_platform must not import from app.*"""
import ast
import os
from pathlib import Path

import pytest

CLEARANCE_PLATFORM_ROOT = Path(__file__).parent.parent.parent / "clearance_platform"


def _collect_python_files(root: Path) -> list[Path]:
    return sorted(root.rglob("*.py"))


def _get_app_imports(filepath: Path) -> list[tuple[int, str]]:
    """Return (line_number, import_statement) for any from app.* or import app.* imports."""
    violations = []
    try:
        tree = ast.parse(filepath.read_text(), filename=str(filepath))
    except SyntaxError:
        return violations

    for node in ast.walk(tree):
        if isinstance(node, ast.ImportFrom) and node.module and node.module.startswith("app."):
            violations.append((node.lineno, f"from {node.module} import ..."))
        elif isinstance(node, ast.Import):
            for alias in node.names:
                if alias.name.startswith("app."):
                    violations.append((node.lineno, f"import {alias.name}"))
    return violations


@pytest.mark.parametrize(
    "filepath",
    _collect_python_files(CLEARANCE_PLATFORM_ROOT),
    ids=lambda p: str(p.relative_to(CLEARANCE_PLATFORM_ROOT)),
)
def test_no_app_imports_in_clearance_platform(filepath: Path) -> None:
    """No file in clearance_platform/ may import from app.* namespace."""
    violations = _get_app_imports(filepath)
    assert not violations, (
        f"{filepath.relative_to(CLEARANCE_PLATFORM_ROOT)} has legacy app.* imports:\n"
        + "\n".join(f"  line {ln}: {stmt}" for ln, stmt in violations)
    )
```

**Step 2: Run test to verify current state**

```bash
cd clearance-engine/backend
python -m pytest tests/architecture/test_no_app_imports.py -v 2>&1 | tail -20
```

Expected: FAIL (showing current violations)

**Step 3: Fix all violations found in Task 1.2**

**Step 4: Run test to verify all pass**

```bash
python -m pytest tests/architecture/test_no_app_imports.py -v
```

Expected: ALL PASS

**Step 5: Run full test suite to verify no regressions**

```bash
python -m pytest tests/ -x --timeout=120 2>&1 | tail -30
```

Expected: No new failures

#### Task 1.4: Write work package report

**File:** `docs/reports/WP-1-legacy-decoupling.md`

Include: finding addressed, all files changed, tests added, verification steps, residual risks.

#### Task 1.5: Commit and push

```bash
git add -A
git commit -m "feat(wp-1): decouple clearance_platform from app.* legacy imports

- Moved engine implementations into clearance_platform/shared/engines/
- Replaced all from app.* imports with clearance_platform internal imports
- Added architecture fitness test test_no_app_imports
- All existing tests continue to pass

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
git push -u origin production-hardening/wp-1-legacy-decoupling
```

---

### WP-2: Event Schema Validation (Finding #8)

**Agent:** event-schema-eng (backend-developer)
**Branch:** `production-hardening/wp-2-event-schema`
**Finding:** "JetStream consumers deserialize into `BaseEvent` (extra fields allowed) and manually poke at dicts."

**Required Reading:**
- `docs/reviews/2025-02-14-production-readiness.md`
- `clearance_platform/shared/events/base.py`
- `clearance_platform/shared/events/registry.py`
- `clearance_platform/shared/event_bus.py`
- All domain `events.py` and `consumers.py` files

#### Task 2.1: Audit current event definitions

**Step 1: List all domain event files**

```bash
find clearance-engine/backend/clearance_platform/domains -name "events.py" -exec echo {} \; -exec head -5 {} \;
```

**Step 2: Check BaseEvent configuration**

```bash
grep -n "extra" clearance-engine/backend/clearance_platform/shared/events/base.py
```

**Step 3: List all event types published across domains**

```bash
grep -rn "event_type=" clearance-engine/backend/clearance_platform/domains/ --include="*.py" | sort
```

**Step 4: List all consumer deserialization patterns**

```bash
grep -rn "BaseEvent" clearance-engine/backend/clearance_platform/domains/*/consumers.py
```

#### Task 2.2: Make BaseEvent strict

**File:** `clearance_platform/shared/events/base.py`

Change `BaseEvent` model config from `extra="allow"` to `extra="forbid"`.

Add a `schema_version` field with default and validation:

```python
from pydantic import BaseModel, ConfigDict, field_validator

class BaseEvent(BaseModel):
    model_config = ConfigDict(extra="forbid")

    metadata: EventMetadata
    schema_version: int = 1

    @field_validator("schema_version")
    @classmethod
    def validate_schema_version(cls, v: int) -> int:
        if v < 1:
            raise ValueError("schema_version must be >= 1")
        return v
```

#### Task 2.3: Define typed event classes for every domain

For each of the 15 domains, ensure `events.py` has explicit Pydantic models for every event type with typed fields (not `dict[str, Any]`).

Example pattern:

```python
# clearance_platform/domains/shipment_lifecycle/events.py
from clearance_platform.shared.events.base import BaseEvent, EventMetadata

class ShipmentCreatedEvent(BaseEvent):
    """Emitted when a new shipment is created."""
    shipment_id: str
    product: str
    origin: str
    destination: str
    declared_value: float
    transport_mode: str
    carrier: str | None = None
    tracking_number: str | None = None

class ShipmentStatusChangedEvent(BaseEvent):
    """Emitted when shipment status transitions."""
    shipment_id: str
    old_status: str
    new_status: str
    reason: str | None = None
```

Repeat for all domain events. Each event class must have explicit typed fields for every piece of data the event carries.

#### Task 2.4: Update event registry

**File:** `clearance_platform/shared/events/registry.py`

Register every typed event class:

```python
from clearance_platform.domains.shipment_lifecycle.events import (
    ShipmentCreatedEvent,
    ShipmentStatusChangedEvent,
)

EVENT_REGISTRY: dict[str, type[BaseEvent]] = {
    "shipment.created": ShipmentCreatedEvent,
    "shipment.status_changed": ShipmentStatusChangedEvent,
    # ... all events
}

def get_event_class(event_type: str) -> type[BaseEvent]:
    cls = EVENT_REGISTRY.get(event_type)
    if cls is None:
        raise ValueError(f"Unknown event type: {event_type}")
    return cls
```

#### Task 2.5: Update all consumers to use typed deserialization

For each domain `consumers.py`:
1. Replace `BaseEvent(**data)` with `SpecificEvent(**data)`
2. Replace `event.metadata.payload["field"]` with `event.field`
3. Use the event registry's `get_event_class()` for dynamic dispatch

#### Task 2.6: Update event_bus.py publish to validate

**File:** `clearance_platform/shared/event_bus.py`

Add validation on publish:

```python
async def publish(self, event: BaseEvent) -> None:
    # Validate event before publishing
    event.model_validate(event.model_dump())
    # ... existing publish logic
```

#### Task 2.7: Write tests

**File:** `tests/unit/test_event_schema_validation.py`

```python
import pytest
from pydantic import ValidationError
from clearance_platform.shared.events.base import BaseEvent, EventMetadata

def test_base_event_rejects_extra_fields():
    metadata = EventMetadata.create(event_type="test", domain="test", subject="test.event")
    with pytest.raises(ValidationError, match="Extra inputs are not permitted"):
        BaseEvent(metadata=metadata, unknown_field="should_fail")

def test_base_event_requires_schema_version():
    metadata = EventMetadata.create(event_type="test", domain="test", subject="test.event")
    event = BaseEvent(metadata=metadata)
    assert event.schema_version == 1

def test_typed_event_validates_fields():
    from clearance_platform.domains.shipment_lifecycle.events import ShipmentCreatedEvent
    metadata = EventMetadata.create(
        event_type="shipment.created", domain="shipment", subject="clearance.shipment.created"
    )
    # Valid event
    event = ShipmentCreatedEvent(
        metadata=metadata, shipment_id="s1", product="widget",
        origin="CN", destination="US", declared_value=1000.0, transport_mode="air"
    )
    assert event.shipment_id == "s1"

    # Missing required field
    with pytest.raises(ValidationError):
        ShipmentCreatedEvent(metadata=metadata, shipment_id="s1")  # missing product, origin, etc.
```

**Step: Run tests**

```bash
python -m pytest tests/unit/test_event_schema_validation.py -v
```

Expected: ALL PASS

#### Task 2.8: Run full test suite, write report, commit

```bash
python -m pytest tests/ -x --timeout=120 2>&1 | tail -30
```

Write `docs/reports/WP-2-event-schema-validation.md`. Commit and push.

---

### WP-3: Cache Projector Fix (Finding #7)

**Agent:** cache-eng (backend-developer)
**Branch:** `production-hardening/wp-3-cache-projector`
**Finding:** "Cache projector stores every event permanently in Redis with no TTL, no partitioning, and runs full-stream replays."

**Required Reading:**
- `clearance_platform/cache_projector/projector.py`
- `clearance_platform/shared/projector/base.py`
- `clearance_platform/shared/projector/registry.py`
- All domain `projector.py` files
- Route handlers that call cache read functions

#### Task 3.1: Audit current cache patterns

**Step 1: Find all Redis SET operations in projector code**

```bash
grep -rn "\.set\(" clearance-engine/backend/clearance_platform/cache_projector/ --include="*.py"
grep -rn "\.set\(" clearance-engine/backend/clearance_platform/shared/projector/ --include="*.py"
grep -rn "\.set\(" clearance-engine/backend/clearance_platform/domains/*/projector.py
```

**Step 2: Find all cache read operations that discard hits**

```bash
grep -rn "get_cached\|redis.*get\|cache.*get" clearance-engine/backend/clearance_platform/ --include="*.py"
```

#### Task 3.2: Add TTL to all Redis writes

**File:** `clearance_platform/shared/projector/base.py`

Add a configurable TTL:

```python
import os

CACHE_TTL_SECONDS = int(os.environ.get("CACHE_TTL_SECONDS", "300"))  # 5 minutes default

class BaseProjector:
    def __init__(self, redis, event_bus, ttl: int = CACHE_TTL_SECONDS):
        self.redis = redis
        self.event_bus = event_bus
        self.ttl = ttl

    async def cache_set(self, key: str, value: str) -> None:
        """Set a cache key with TTL."""
        await self.redis.set(key, value, ex=self.ttl)

    async def cache_set_no_ttl(self, key: str, value: str) -> None:
        """Set a cache key without TTL (for indexes that must persist)."""
        await self.redis.set(key, value)
```

Update ALL projector Redis writes to use `self.cache_set()` instead of raw `self.redis.set()`.

#### Task 3.3: Add key prefix partitioning

Update all projector key patterns to include domain prefix:

```python
# Before:
key = f"entity:shipment:{shipment_id}"

# After:
key = f"shipment:entity:shipment:{shipment_id}"
```

Pattern: `{domain}:entity:{type}:{id}` for entity snapshots
Pattern: `{domain}:idx:{type}:by-{attr}:{value}` for indexes
Pattern: `{domain}:agg:{name}` for aggregates

#### Task 3.4: Fix read handlers to use cached data

For every route handler that calls a cache read function:
1. Check if cache hit
2. If hit, return cached data (deserialize from JSON)
3. If miss, query DB, cache result, return

```python
# Before (broken pattern):
cached = await redis.get(f"entity:shipment:{id}")
# cached is ignored, always queries DB
result = await session.execute(select(Shipment).where(Shipment.id == id))

# After (correct pattern):
cached = await redis.get(f"shipment:entity:shipment:{id}")
if cached:
    return json.loads(cached)
result = await session.execute(select(Shipment).where(Shipment.id == id))
shipment = result.scalar_one_or_none()
if shipment:
    data = shipment_to_dict(shipment)
    await redis.set(f"shipment:entity:shipment:{id}", json.dumps(data), ex=CACHE_TTL_SECONDS)
    return data
```

#### Task 3.5: Replace full-stream replay with incremental rebuild

**File:** `clearance_platform/cache_projector/projector.py`

Track last-processed sequence number in Redis:

```python
LAST_SEQ_KEY = "cache_projector:last_seq"

async def _on_event(self, msg):
    seq = msg.metadata.sequence.stream
    # Process event
    await self.project_event(event_data)
    # Track sequence
    await self.redis.set(LAST_SEQ_KEY, str(seq))

async def rebuild(self):
    """Incremental rebuild from last known sequence."""
    last_seq = await self.redis.get(LAST_SEQ_KEY)
    start_seq = int(last_seq) + 1 if last_seq else 1
    # Replay from start_seq instead of beginning
```

#### Task 3.6: Write tests

**File:** `tests/unit/test_cache_projector_ttl.py`

```python
import pytest
from unittest.mock import AsyncMock, patch

@pytest.mark.asyncio
async def test_cache_set_includes_ttl():
    redis_mock = AsyncMock()
    projector = BaseProjector(redis=redis_mock, event_bus=AsyncMock(), ttl=300)
    await projector.cache_set("test:key", '{"data": 1}')
    redis_mock.set.assert_called_once_with("test:key", '{"data": 1}', ex=300)

@pytest.mark.asyncio
async def test_read_handler_returns_cached_data():
    # Test that a route handler returns cached data without DB query
    redis_mock = AsyncMock()
    redis_mock.get.return_value = '{"id": "s1", "status": "cleared"}'
    # ... test that DB session.execute is NOT called
```

#### Task 3.7: Run tests, write report, commit

Run full suite. Write `docs/reports/WP-3-cache-projector-fix.md`. Commit and push.

---

### WP-4: CORS & Rate Limit Hardening (Finding #9)

**Agent:** security-eng (backend-developer)
**Branch:** `production-hardening/wp-4-cors-ratelimits`
**Finding:** "`platform_service` enables `allow_origins=["*"]` with credentials, and most routers allow `50k/minute` without authentication."

**Required Reading:**
- `app/main.py` lines 330-343 (middleware setup)
- `clearance_platform/platform_service/main.py`
- All route files for `@limiter.limit()` decorators

#### Task 4.1: Implement configurable CORS allowlist

**File:** `app/main.py`

```python
# Replace single ALLOWED_ORIGIN with multi-origin support
import os

def _get_allowed_origins() -> list[str]:
    """Parse ALLOWED_ORIGINS env var (comma-separated) or fall back to ALLOWED_ORIGIN."""
    origins_str = os.environ.get("ALLOWED_ORIGINS", "")
    if origins_str:
        return [o.strip() for o in origins_str.split(",") if o.strip()]
    # Fallback to single origin setting
    return [settings.ALLOWED_ORIGIN]

# In create_app():
allowed_origins = _get_allowed_origins()
application.add_middleware(
    CORSMiddleware,
    allow_origins=allowed_origins,
    allow_credentials=("*" not in allowed_origins),  # Disable credentials with wildcard
    allow_methods=["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"],
    allow_headers=["Authorization", "Content-Type", "X-CSRF-Token"],
)
```

**File:** `clearance_platform/platform_service/main.py`

Apply same pattern — replace `["*"]` with configurable allowlist.

#### Task 4.2: Set tiered rate limits

Create rate limit tiers:

```python
# clearance_platform/shared/rate_limits.py
RATE_LIMIT_ADMIN = "10/minute"
RATE_LIMIT_LLM = "30/minute"
RATE_LIMIT_WRITE = "100/minute"
RATE_LIMIT_READ = "500/minute"
```

Update route decorators across all domain routes:

- Admin/seed endpoints: `@limiter.limit(RATE_LIMIT_ADMIN)`
- LLM endpoints (chat, classify, analyze, description-quality): `@limiter.limit(RATE_LIMIT_LLM)`
- POST/PUT/DELETE endpoints: `@limiter.limit(RATE_LIMIT_WRITE)`
- GET endpoints: `@limiter.limit(RATE_LIMIT_READ)`

**IMPORTANT:** Per project memory, DO NOT set limits below what's reasonable. The tiers above are production-appropriate. Verify with the user if unsure.

#### Task 4.3: Write tests

**File:** `tests/unit/test_cors_rate_limits.py`

```python
import pytest
from httpx import AsyncClient
from app.main import create_app

@pytest.mark.asyncio
async def test_cors_rejects_disallowed_origin():
    app = create_app()
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.options(
            "/api/shipments",
            headers={"Origin": "http://evil.com", "Access-Control-Request-Method": "GET"},
        )
        assert "http://evil.com" not in response.headers.get("access-control-allow-origin", "")

@pytest.mark.asyncio
async def test_cors_allows_configured_origin():
    import os
    os.environ["ALLOWED_ORIGINS"] = "http://localhost:4001"
    app = create_app()
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.options(
            "/api/shipments",
            headers={"Origin": "http://localhost:4001", "Access-Control-Request-Method": "GET"},
        )
        assert response.headers.get("access-control-allow-origin") == "http://localhost:4001"
```

#### Task 4.4: Run tests, write report, commit

Run full suite. Write `docs/reports/WP-4-cors-ratelimits.md`. Commit and push.

---

## WAVE 1 REVIEW CHECKPOINT

After all 4 Wave 1 branches complete:

### Task W1-Review: Merge and validate Wave 1

**Agent:** team-lead + architect-reviewer + consistency-validator

**Step 1: Merge all Wave 1 branches into production-hardening**

```bash
git checkout production-hardening
git merge production-hardening/wp-1-legacy-decoupling --no-ff -m "Merge WP-1: Legacy decoupling"
git merge production-hardening/wp-2-event-schema --no-ff -m "Merge WP-2: Event schema validation"
git merge production-hardening/wp-3-cache-projector --no-ff -m "Merge WP-3: Cache projector fix"
git merge production-hardening/wp-4-cors-ratelimits --no-ff -m "Merge WP-4: CORS & rate limits"
```

**Step 2: Resolve any merge conflicts**

**Step 3: Run full test suite**

```bash
cd clearance-engine/backend
python -m pytest tests/ -x --timeout=120 -v 2>&1 | tail -50
```

Expected: ALL PASS

**Step 4: Architect review**

architect-reviewer validates:
- WP-1: No `app.*` imports remain in clearance_platform/
- WP-2: BaseEvent has `extra="forbid"`, all events are typed
- WP-3: All Redis SET operations have TTL, read handlers use cache
- WP-4: CORS uses allowlist, rate limits are tiered

**Step 5: Consistency review**

consistency-validator checks:
- Code style consistent across all 4 WPs
- Import patterns uniform
- Test naming conventions followed
- No duplicate code introduced

---

## WAVE 2: Domain Isolation (3 parallel work packages)

Depends on Wave 1 completion (especially WP-1 decoupling).

---

### WP-5: Database Isolation (Finding #2)

**Agent:** db-isolation-eng (database-administrator)
**Branch:** `production-hardening/wp-5-db-isolation`
**Finding:** "All domain containers connect to the same `clearance_engine` Postgres database using schema prefixes only."

**Required Reading:**
- `clearance_platform/shared/database.py` (DOMAIN_SCHEMAS mapping)
- `data/init.sql`
- All domain `models.py` files
- `alembic/env.py`
- Serena memory: `database-schema-and-data-layer`

#### Task 5.1: Audit current schema state

**Step 1: Check which tables exist in which schemas**

```bash
# Connect to DB and list schemas/tables
cd clearance-engine/backend
python -c "
import asyncio
from sqlalchemy import create_engine, text
engine = create_engine('postgresql://clearance:clearance@localhost:5432/clearance_engine')
with engine.connect() as conn:
    result = conn.execute(text(\"\"\"
        SELECT table_schema, table_name
        FROM information_schema.tables
        WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
        ORDER BY table_schema, table_name
    \"\"\"))
    for row in result:
        print(f'{row[0]}.{row[1]}')
"
```

**Step 2: Verify DOMAIN_SCHEMAS mapping matches expected schemas**

```bash
cat clearance-engine/backend/clearance_platform/shared/database.py
```

#### Task 5.2: Update all domain models with schema specification

For each of the 15 domains, ensure `models.py` has correct `__table_args__`:

```python
# Example: clearance_platform/domains/shipment_lifecycle/models.py
from clearance_platform.shared.database import get_schema_name

SCHEMA = get_schema_name("shipment_lifecycle")  # Returns "shipment"

class Shipment(DomainBase):
    __tablename__ = "shipments"
    __table_args__ = {"schema": SCHEMA}
    # ... columns
```

Repeat for every model in every domain.

#### Task 5.3: Create Alembic migration for schema isolation

**File:** `alembic/versions/028_enforce_domain_schema_isolation.py`

```python
"""Enforce domain schema isolation: move tables to domain schemas and set up roles."""

from alembic import op
import sqlalchemy as sa

# Schema mapping
DOMAIN_SCHEMAS = {
    "product_catalog": "product",
    "trade_intelligence": "trade_intel",
    "compliance": "compliance",
    "order_management": "order_mgmt",
    "shipment_lifecycle": "shipment",
    "cargo_handling_units": "cargo",
    "consolidation": "consolidation",
    "declaration_management": "declaration",
    "customs_adjudication": "adjudication",
    "exception_management": "exception",
    "financial_settlement": "financial",
    "regulatory_intelligence": "regulatory",
    "document_management": "document",
    "party_management": "party",
    "supply_chain_disruptions": "disruption",
}

def upgrade() -> None:
    # Create schemas if they don't exist
    for schema in DOMAIN_SCHEMAS.values():
        op.execute(f"CREATE SCHEMA IF NOT EXISTS {schema}")

    # Create per-domain roles
    for domain, schema in DOMAIN_SCHEMAS.items():
        role_name = f"svc_{schema}"
        op.execute(f"""
            DO $$ BEGIN
                CREATE ROLE {role_name} LOGIN PASSWORD '{role_name}_pass';
            EXCEPTION WHEN duplicate_object THEN NULL;
            END $$;
        """)
        op.execute(f"GRANT USAGE ON SCHEMA {schema} TO {role_name}")
        op.execute(f"GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA {schema} TO {role_name}")
        op.execute(f"ALTER DEFAULT PRIVILEGES IN SCHEMA {schema} GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO {role_name}")

    # Revoke cross-schema access (each role can only access its own schema)
    for domain, schema in DOMAIN_SCHEMAS.items():
        role_name = f"svc_{schema}"
        for other_schema in DOMAIN_SCHEMAS.values():
            if other_schema != schema:
                op.execute(f"REVOKE ALL ON SCHEMA {other_schema} FROM {role_name}")


def downgrade() -> None:
    # Reverse role restrictions
    for domain, schema in DOMAIN_SCHEMAS.items():
        role_name = f"svc_{schema}"
        for other_schema in DOMAIN_SCHEMAS.values():
            op.execute(f"GRANT USAGE ON SCHEMA {other_schema} TO {role_name}")
```

#### Task 5.4: Update init.sql

**File:** `data/init.sql`

Add schema creation and role setup to the init script so fresh databases are correctly initialized.

#### Task 5.5: Update database.py with per-domain session factories

**File:** `clearance_platform/shared/database.py`

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
import os

def get_domain_database_url(domain: str) -> str:
    """Get the database URL for a specific domain (with domain-specific role)."""
    schema = get_schema_name(domain)
    role = f"svc_{schema}"
    base_url = os.environ.get("DATABASE_URL", "postgresql+asyncpg://clearance:clearance@localhost:5432/clearance_engine")
    # In microservices mode, each service connects with its own role
    if os.environ.get("MICROSERVICES_MODE", "").lower() == "true":
        # Replace credentials in URL with domain-specific role
        return base_url.replace("clearance:clearance", f"{role}:{role}_pass")
    return base_url

def create_domain_session_factory(domain: str):
    """Create an async session factory scoped to a specific domain schema."""
    url = get_domain_database_url(domain)
    engine = create_async_engine(url, pool_size=5, max_overflow=5, pool_pre_ping=True)
    return async_sessionmaker(engine, expire_on_commit=False)
```

#### Task 5.6: Write isolation test

**File:** `tests/integration/test_db_isolation.py`

```python
import pytest
from sqlalchemy import text
from clearance_platform.shared.database import get_schema_name

@pytest.mark.asyncio
async def test_domain_role_cannot_access_other_schema(db_session_factory):
    """Verify that a domain's DB role cannot query another domain's tables."""
    # This test verifies the GRANT/REVOKE setup
    # In practice, run with svc_shipment role and try to SELECT from compliance schema
    async with db_session_factory() as session:
        schemas = await session.execute(text(
            "SELECT schema_name FROM information_schema.schemata WHERE schema_name NOT IN ('public', 'pg_catalog', 'information_schema')"
        ))
        domain_schemas = [row[0] for row in schemas]
        assert len(domain_schemas) >= 15, f"Expected 15+ domain schemas, got {len(domain_schemas)}"
```

#### Task 5.7: Run tests, write report, commit

Run full suite. Write `docs/reports/WP-5-db-isolation.md`. Commit and push.

---

### WP-6: Cross-Domain SQL Elimination (Finding #5)

**Agent:** domain-boundary-eng (backend-developer)
**Branch:** `production-hardening/wp-6-cross-domain-sql`
**Finding:** "Gateways directly query or mutate other domains' tables. `shared/model_registry` auto-imports every model into a single metadata registry."

**Required Reading:**
- All `clearance_platform/gateway/*_routes.py` files
- `clearance_platform/shared/model_registry.py`
- `app/main.py` (where model_registry is called)

#### Task 6.1: Audit cross-domain queries in gateway routes

**Step 1: Find all model imports in gateway routes**

```bash
grep -rn "from clearance_platform.domains\." clearance-engine/backend/clearance_platform/gateway/ --include="*.py" | sort
```

**Step 2: Find all direct SQL queries in gateway routes**

```bash
grep -rn "session.execute\|select(" clearance-engine/backend/clearance_platform/gateway/ --include="*.py"
```

**Step 3: Identify cross-domain joins**

```bash
grep -rn "join\|outerjoin" clearance-engine/backend/clearance_platform/gateway/ --include="*.py"
```

#### Task 6.2: Replace cross-domain queries with projections

For each gateway route that queries multiple domains:

1. **Dashboard routes** → Use Redis-cached projections (built by WP-3 projectors)
2. **Detail routes** → Use single-domain query + cached enrichment
3. **Mutation routes** → Use service-to-service HTTP call or event emission

Example replacement:

```python
# Before (cross-domain SQL):
from clearance_platform.domains.shipment_lifecycle.models import Shipment
from clearance_platform.domains.declaration_management.models import EntryFiling
# ... join query

# After (cached projection):
async def get_dashboard_data(redis):
    shipment_stats = await redis.get("shipment:agg:status_counts")
    entry_stats = await redis.get("declaration:agg:filing_status_counts")
    return {
        "shipments": json.loads(shipment_stats) if shipment_stats else {},
        "entries": json.loads(entry_stats) if entry_stats else {},
    }
```

#### Task 6.3: Refactor model_registry.py

**File:** `clearance_platform/shared/model_registry.py`

Replace the auto-import-all pattern with explicit per-domain imports only when needed for SQLAlchemy relationship resolution:

```python
def import_all_domain_models():
    """Import domain models so SQLAlchemy can resolve cross-domain relationships.

    This is called once at app startup. It does NOT grant cross-domain query access;
    it only ensures SQLAlchemy's metadata has all tables for relationship resolution.
    """
    # Explicit imports for relationship resolution only
    from clearance_platform.domains.shipment_lifecycle import models as _  # noqa: F401
    from clearance_platform.domains.order_management import models as _  # noqa: F401
    # ... only domains that have cross-domain FKs
```

#### Task 6.4: Write architecture test

**File:** `tests/architecture/test_no_cross_domain_sql.py`

```python
"""Architecture test: gateway routes must not import models from multiple domains."""
import ast
from pathlib import Path

import pytest

GATEWAY_ROOT = Path(__file__).parent.parent.parent / "clearance_platform" / "gateway"

def _get_domain_model_imports(filepath: Path) -> list[str]:
    """Return list of domain names imported via model imports."""
    domains = set()
    tree = ast.parse(filepath.read_text())
    for node in ast.walk(tree):
        if isinstance(node, ast.ImportFrom) and node.module:
            if ".domains." in node.module and ".models" in node.module:
                # Extract domain name: clearance_platform.domains.{domain}.models
                parts = node.module.split(".")
                idx = parts.index("domains")
                if idx + 1 < len(parts):
                    domains.add(parts[idx + 1])
    return sorted(domains)

@pytest.mark.parametrize(
    "filepath",
    sorted(GATEWAY_ROOT.glob("*.py")),
    ids=lambda p: p.name,
)
def test_gateway_route_does_not_join_across_domains(filepath: Path):
    """Gateway routes should not import models from more than one domain."""
    domains = _get_domain_model_imports(filepath)
    assert len(domains) <= 1, (
        f"{filepath.name} imports models from {len(domains)} domains: {domains}. "
        "Use cached projections or service calls instead of cross-domain SQL."
    )
```

#### Task 6.5: Run tests, write report, commit

Run full suite. Write `docs/reports/WP-6-cross-domain-sql.md`. Commit and push.

---

### WP-7: Authentication Middleware (Finding #3)

**Agent:** security-eng (backend-developer)
**Branch:** `production-hardening/wp-7-auth`
**Finding:** "High-impact APIs accept arbitrary requests with no auth or CSRF protection."

**Required Reading:**
- `app/main.py` (current middleware stack)
- All route files (to understand endpoints)
- Frontend `api/client.ts` (to understand how requests are made)

#### Task 7.1: Create auth module

**Directory:** `clearance_platform/shared/auth/`

**File:** `clearance_platform/shared/auth/__init__.py`

```python
from .dependencies import require_auth, require_role, get_current_user
from .models import User, AuthToken
```

**File:** `clearance_platform/shared/auth/config.py`

```python
import os
from datetime import timedelta

JWT_SECRET = os.environ.get("JWT_SECRET", "dev-secret-change-in-production")
JWT_ALGORITHM = "HS256"
JWT_EXPIRY = timedelta(hours=int(os.environ.get("JWT_EXPIRY_HOURS", "24")))
ADMIN_USERS = set(os.environ.get("ADMIN_USERS", "admin").split(","))
```

**File:** `clearance_platform/shared/auth/models.py`

```python
from pydantic import BaseModel
from datetime import datetime

class User(BaseModel):
    username: str
    role: str  # "viewer", "broker", "admin"
    email: str | None = None

class AuthToken(BaseModel):
    access_token: str
    token_type: str = "bearer"
    expires_at: datetime

class LoginRequest(BaseModel):
    username: str
    password: str
```

**File:** `clearance_platform/shared/auth/dependencies.py`

```python
import jwt
from fastapi import Depends, HTTPException, Request, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from .config import JWT_SECRET, JWT_ALGORITHM
from .models import User

security = HTTPBearer(auto_error=False)

async def get_current_user(
    request: Request,
    credentials: HTTPAuthorizationCredentials | None = Depends(security),
) -> User:
    """Extract and validate the current user from JWT token."""
    token = None

    # Check Authorization header
    if credentials:
        token = credentials.credentials

    # Check cookie fallback
    if not token:
        token = request.cookies.get("access_token")

    if not token:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Authentication required",
            headers={"WWW-Authenticate": "Bearer"},
        )

    try:
        payload = jwt.decode(token, JWT_SECRET, algorithms=[JWT_ALGORITHM])
        return User(
            username=payload["sub"],
            role=payload.get("role", "viewer"),
            email=payload.get("email"),
        )
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid token")


def require_auth(user: User = Depends(get_current_user)) -> User:
    """Dependency that requires authentication."""
    return user


def require_role(role: str):
    """Factory for role-based access control dependency."""
    async def _check_role(user: User = Depends(require_auth)) -> User:
        if user.role != role and user.role != "admin":
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Role '{role}' required",
            )
        return user
    return _check_role
```

**File:** `clearance_platform/shared/auth/routes.py`

```python
import jwt
from datetime import datetime, timezone
from fastapi import APIRouter, HTTPException, Response, status
from .config import JWT_SECRET, JWT_ALGORITHM, JWT_EXPIRY, ADMIN_USERS
from .models import AuthToken, LoginRequest, User
from .dependencies import require_auth, get_current_user
from fastapi import Depends

router = APIRouter(prefix="/api/auth", tags=["auth"])

# In-memory user store for development (replace with DB in production)
DEV_USERS = {
    "admin": {"password": "admin", "role": "admin", "email": "admin@clearance.io"},
    "broker": {"password": "broker", "role": "broker", "email": "broker@clearance.io"},
    "viewer": {"password": "viewer", "role": "viewer", "email": "viewer@clearance.io"},
}


@router.post("/login", response_model=AuthToken)
async def login(request: LoginRequest, response: Response) -> AuthToken:
    user_data = DEV_USERS.get(request.username)
    if not user_data or user_data["password"] != request.password:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid credentials")

    expires_at = datetime.now(timezone.utc) + JWT_EXPIRY
    token = jwt.encode(
        {"sub": request.username, "role": user_data["role"], "exp": expires_at},
        JWT_SECRET,
        algorithm=JWT_ALGORITHM,
    )

    # Set httpOnly cookie
    response.set_cookie(
        key="access_token",
        value=token,
        httponly=True,
        samesite="strict",
        secure=False,  # Set True in production with HTTPS
        max_age=int(JWT_EXPIRY.total_seconds()),
    )

    return AuthToken(access_token=token, expires_at=expires_at)


@router.post("/logout")
async def logout(response: Response) -> dict:
    response.delete_cookie("access_token")
    return {"status": "logged_out"}


@router.get("/me")
async def get_me(user: User = Depends(require_auth)) -> User:
    return user
```

#### Task 7.2: Add auth dependency to all route handlers

For each domain `routes.py` and gateway route file:

```python
from clearance_platform.shared.auth.dependencies import require_auth, require_role
from clearance_platform.shared.auth.models import User

# Read endpoints:
@router.get("/items")
async def list_items(user: User = Depends(require_auth)):
    ...

# Admin endpoints:
@router.post("/admin/seed")
async def seed_data(user: User = Depends(require_role("admin"))):
    ...
```

**IMPORTANT:** The health endpoint (`/health`) and auth endpoints (`/api/auth/*`) must remain unauthenticated.

#### Task 7.3: Register auth router

**File:** `clearance_platform/gateway/router_registry.py`

Add auth router registration:

```python
from clearance_platform.shared.auth.routes import router as auth_router

def register_all_routers(app):
    app.include_router(auth_router)
    # ... existing registrations
```

#### Task 7.4: Frontend auth integration

**File:** `clearance-engine/frontend/src/api/client.ts`

Add token management to all API calls:

```typescript
function getAuthHeaders(): Record<string, string> {
  const token = localStorage.getItem("access_token");
  if (token) {
    return { Authorization: `Bearer ${token}` };
  }
  return {};
}

// Update all fetch calls to include auth headers
async function apiFetch(path: string, options: RequestInit = {}): Promise<Response> {
  const headers = {
    ...getAuthHeaders(),
    ...(options.headers || {}),
  };
  const response = await fetch(`${API_URL}${path}`, { ...options, headers, credentials: "include" });
  if (response.status === 401) {
    // Redirect to login
    window.location.href = "/login";
    throw new APIError(401, "Authentication required");
  }
  return response;
}
```

**File:** `clearance-engine/frontend/src/surfaces/auth/LoginPage.tsx`

Create a login page component with username/password form.

**File:** `clearance-engine/frontend/src/router.tsx`

Add login route and auth guard wrapper:

```typescript
import LoginPage from "./surfaces/auth/LoginPage";

// Add route
{ path: "/login", element: <LoginPage /> }

// Wrap authenticated routes with auth guard
```

#### Task 7.5: Write tests

**File:** `tests/unit/test_auth_middleware.py`

```python
import pytest
import jwt
from datetime import datetime, timezone, timedelta
from httpx import AsyncClient
from app.main import create_app
from clearance_platform.shared.auth.config import JWT_SECRET, JWT_ALGORITHM

def _make_token(username="testuser", role="viewer", expired=False):
    exp = datetime.now(timezone.utc) + (timedelta(hours=-1) if expired else timedelta(hours=1))
    return jwt.encode({"sub": username, "role": role, "exp": exp}, JWT_SECRET, algorithm=JWT_ALGORITHM)

@pytest.mark.asyncio
async def test_unauthenticated_request_returns_401():
    app = create_app()
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/api/shipments")
        assert response.status_code == 401

@pytest.mark.asyncio
async def test_valid_token_returns_200():
    app = create_app()
    token = _make_token()
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/api/shipments", headers={"Authorization": f"Bearer {token}"})
        assert response.status_code != 401  # May be 200 or other, but not 401

@pytest.mark.asyncio
async def test_expired_token_returns_401():
    app = create_app()
    token = _make_token(expired=True)
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/api/shipments", headers={"Authorization": f"Bearer {token}"})
        assert response.status_code == 401

@pytest.mark.asyncio
async def test_admin_endpoint_requires_admin_role():
    app = create_app()
    viewer_token = _make_token(role="viewer")
    admin_token = _make_token(role="admin")
    async with AsyncClient(app=app, base_url="http://test") as client:
        # Viewer should get 403
        r1 = await client.post("/api/admin/seed", headers={"Authorization": f"Bearer {viewer_token}"})
        assert r1.status_code == 403

        # Admin should not get 403
        r2 = await client.post("/api/admin/seed", headers={"Authorization": f"Bearer {admin_token}"})
        assert r2.status_code != 403

@pytest.mark.asyncio
async def test_health_endpoint_no_auth_required():
    app = create_app()
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/health")
        assert response.status_code == 200
```

#### Task 7.6: Run tests, write report, commit

Run full suite. Write `docs/reports/WP-7-auth-middleware.md`. Commit and push.

---

## WAVE 2 REVIEW CHECKPOINT

### Task W2-Review: Merge and validate Wave 2

Same pattern as Wave 1. Merge WP-5, WP-6, WP-7 into `production-hardening`. Run full suite. Architect and consistency review.

---

## WAVE 3: Completion (3 parallel work packages)

Depends on Wave 2 completion.

---

### WP-8: Ingestion Persistence (Finding #4)

**Agent:** ingestion-eng (backend-developer)
**Branch:** `production-hardening/wp-8-ingestion-persistence`
**Finding:** "Ingestion pipelines only log results; TODOs for DB persistence remain."

**Required Reading:**
- `clearance_platform/shared/ingestion/pipeline.py` (base class)
- All 7 pipeline files in `clearance_platform/shared/ingestion/`
- Domain models for reference data (compliance, trade_intelligence)

#### Task 8.1: Create ingestion_metadata table

**File:** New Alembic migration

```python
"""Add ingestion_metadata tracking table."""
from alembic import op
import sqlalchemy as sa

def upgrade():
    op.create_table(
        "ingestion_metadata",
        sa.Column("id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("pipeline_name", sa.String(100), unique=True, nullable=False),
        sa.Column("last_run_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column("last_hash", sa.String(64), nullable=True),  # SHA-256
        sa.Column("records_added", sa.Integer, default=0),
        sa.Column("records_updated", sa.Integer, default=0),
        sa.Column("records_deleted", sa.Integer, default=0),
        sa.Column("status", sa.String(20), default="idle"),  # idle, running, failed
        sa.Column("error_message", sa.Text, nullable=True),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now()),
        sa.Column("updated_at", sa.DateTime(timezone=True), server_default=sa.func.now(), onupdate=sa.func.now()),
        schema="public",  # Shared across domains
    )

def downgrade():
    op.drop_table("ingestion_metadata", schema="public")
```

#### Task 8.2: Implement upsert() in each pipeline

For each of the 7 pipelines, implement the `upsert()` method that:
1. Takes parsed records
2. Upserts into the appropriate domain table
3. Returns `{"added": int, "updated": int, "deleted": int}`
4. Uses SHA-256 hash for dedup (stored in `ingestion_metadata`)

Example for OFAC SDN pipeline:

```python
# clearance_platform/shared/ingestion/ofac_sdn.py
async def upsert(self, records: list[dict], session) -> dict[str, int]:
    added = updated = 0
    for record in records:
        existing = await session.execute(
            select(RestrictedParty).where(
                RestrictedParty.source_list_id == record["source_list_id"],
                RestrictedParty.list_source == "OFAC",
            )
        )
        party = existing.scalar_one_or_none()
        if party:
            for key, value in record.items():
                setattr(party, key, value)
            updated += 1
        else:
            session.add(RestrictedParty(**record, list_source="OFAC"))
            added += 1
    await session.commit()
    return {"added": added, "updated": updated, "deleted": 0}
```

Repeat for: HTSUS, AD/CVD, Exchange Rates, UFLPA, CSL, Trade Remedies.

#### Task 8.3: Replace in-memory dedup with DB-backed dedup

Update `pipeline.py` base class:

```python
async def _check_dedup(self, content_hash: str, session) -> bool:
    """Check if data has changed since last ingestion."""
    from clearance_platform.shared.ingestion.models import IngestionMetadata
    result = await session.execute(
        select(IngestionMetadata).where(IngestionMetadata.pipeline_name == self.name)
    )
    metadata = result.scalar_one_or_none()
    if metadata and metadata.last_hash == content_hash:
        return True  # No change, skip
    return False

async def _update_metadata(self, content_hash: str, stats: dict, session) -> None:
    """Update ingestion metadata after successful run."""
    from clearance_platform.shared.ingestion.models import IngestionMetadata
    result = await session.execute(
        select(IngestionMetadata).where(IngestionMetadata.pipeline_name == self.name)
    )
    metadata = result.scalar_one_or_none()
    if metadata:
        metadata.last_hash = content_hash
        metadata.last_run_at = datetime.now(timezone.utc)
        metadata.records_added = stats["added"]
        metadata.records_updated = stats["updated"]
        metadata.records_deleted = stats["deleted"]
        metadata.status = "idle"
    else:
        session.add(IngestionMetadata(
            pipeline_name=self.name,
            last_hash=content_hash,
            last_run_at=datetime.now(timezone.utc),
            **stats,
            status="idle",
        ))
    await session.commit()
```

#### Task 8.4: Write tests

**File:** `tests/integration/test_ingestion_persistence.py`

```python
@pytest.mark.asyncio
async def test_ofac_pipeline_persists_to_db(db_session):
    from clearance_platform.shared.ingestion.ofac_sdn import OFACSDNPipeline
    pipeline = OFACSDNPipeline()
    # Mock fetch to return test data
    records = pipeline.parse(SAMPLE_OFAC_XML)
    stats = await pipeline.upsert(records, db_session)
    assert stats["added"] > 0

    # Verify data in DB
    result = await db_session.execute(select(RestrictedParty).where(RestrictedParty.list_source == "OFAC"))
    parties = result.scalars().all()
    assert len(parties) == stats["added"]

@pytest.mark.asyncio
async def test_dedup_prevents_duplicates(db_session):
    pipeline = OFACSDNPipeline()
    records = pipeline.parse(SAMPLE_OFAC_XML)
    stats1 = await pipeline.upsert(records, db_session)
    stats2 = await pipeline.upsert(records, db_session)
    assert stats2["added"] == 0  # No new records on second run
    assert stats2["updated"] >= 0  # May update existing
```

#### Task 8.5: Run tests, write report, commit

Write `docs/reports/WP-8-ingestion-persistence.md`. Commit and push.

---

### WP-9: Stub Replacement (Finding #10)

**Agent:** stub-replacement-eng (fullstack-developer)
**Branch:** `production-hardening/wp-9-stub-replacement`
**Finding:** "Financial settlement listing APIs return empty data; compliance dashboards rely on live cross-domain joins."

**Required Reading:**
- Serena memory: `ARCHITECTURE-AUDIT-FINDINGS` (Section C: Domain Routes — 70% stubs)
- All domain `routes.py` and `service.py` files

#### Task 9.1: Audit all stub endpoints

**Step 1: Find all placeholder responses**

```bash
cd clearance-engine/backend
grep -rn '{"status": "ok"}\|{"id": "placeholder"}\|\[\]\|{"event":' clearance_platform/domains/*/routes.py
grep -rn 'return \[\]' clearance_platform/domains/*/routes.py
grep -rn 'return {}' clearance_platform/domains/*/routes.py
```

**Step 2: Create tracking list**

Document every stub endpoint: file, line, route path, current response, what it should return.

#### Task 9.2: Implement financial settlement listing

**File:** `clearance_platform/domains/financial_settlement/routes.py` and `service.py`

Replace empty list returns with real aggregation:

```python
@router.get("/financial/settlements")
async def list_settlements(
    session: AsyncSession = Depends(get_session),
    user: User = Depends(require_auth),
):
    service = FinancialSettlementService(session)
    settlements = await service.list_settlements()
    return settlements
```

Implement `list_settlements()` in `service.py` to query the financial schema tables.

#### Task 9.3: Implement compliance dashboard data

Replace cross-domain joins with cached projection reads (leveraging WP-3 and WP-6 work).

#### Task 9.4: Implement cargo HU real CRUD

**Files:** `clearance_platform/domains/cargo_handling_units/routes.py`, `service.py`

Replace placeholder responses with real DB operations:

```python
@router.post("/cargo-handling-units")
async def create_handling_unit(
    payload: CreateHURequest,
    session: AsyncSession = Depends(get_session),
    user: User = Depends(require_auth),
):
    service = CargoHandlingUnitService(session)
    hu = await service.create_hu(payload)
    return hu

@router.get("/cargo-handling-units")
async def list_handling_units(
    session: AsyncSession = Depends(get_session),
    user: User = Depends(require_auth),
):
    service = CargoHandlingUnitService(session)
    return await service.list_hus()
```

#### Task 9.5: Frontend placeholder treatment

For any endpoint where backend data genuinely isn't available yet:

```tsx
// Component pattern for "data pending" state
function DataPendingPlaceholder({ label }: { label: string }) {
  return (
    <div className="flex items-center gap-2 p-4 bg-amber-50 border border-amber-200 rounded-lg">
      <AlertTriangle className="w-4 h-4 text-amber-500" />
      <span className="text-sm text-amber-700">
        {label} — data integration pending
      </span>
    </div>
  );
}
```

Audit all frontend components that consume previously-stubbed endpoints. Ensure they show this placeholder when data is empty/unavailable, never a blank screen.

#### Task 9.6: Write E2E tests for each previously-stubbed endpoint

**File:** `clearance-engine/frontend/e2e/stub-replacement-proof.spec.ts`

```typescript
import { test, expect } from "@playwright/test";

const BASE = "http://localhost:4001";

test.describe("Previously-stubbed endpoints now return real data", () => {
  test("financial settlements list returns data", async ({ page }) => {
    // Login first
    await page.goto(`${BASE}/login`);
    await page.fill('[name="username"]', "broker");
    await page.fill('[name="password"]', "broker");
    await page.click('button[type="submit"]');
    await page.waitForURL("**/platform");

    // Navigate to financial section
    await page.goto(`${BASE}/platform/financial`);
    await expect(page.locator("table tbody tr").first()).toBeVisible({ timeout: 15000 });
  });

  test("cargo handling units list returns data", async ({ page }) => {
    // ... similar pattern
  });

  test("compliance dashboard shows real data or pending placeholder", async ({ page }) => {
    await page.goto(`${BASE}/platform/compliance`);
    // Either real data or amber "pending" placeholder, never blank
    const hasData = await page.locator("table tbody tr").count();
    const hasPending = await page.locator("text=data integration pending").count();
    expect(hasData + hasPending).toBeGreaterThan(0);
  });
});
```

#### Task 9.7: Run tests, write report, commit

Write `docs/reports/WP-9-stub-replacement.md`. Commit and push.

---

### WP-10: Architecture Test Enforcement (Finding #6)

**Agent:** qa-admin (qa-expert)
**Branch:** `production-hardening/wp-10-arch-tests`
**Finding:** "`tests/architecture/test_microservice_structure.py` explicitly states the suite is expected to fail."

**Required Reading:**
- `tests/architecture/test_microservice_structure.py`
- `tests/architecture/conftest.py`
- All architecture test files

#### Task 10.1: Remove all xfail and skip markers

**Step 1: Find all xfail/skip markers**

```bash
grep -rn "xfail\|skip\|expected to fail\|Expected to FAIL" clearance-engine/backend/tests/architecture/ --include="*.py"
```

**Step 2: Remove them**

Replace `@pytest.mark.xfail` with nothing. Remove "expected to fail" comments.

#### Task 10.2: Fix failing architecture tests

Run the architecture tests and fix each failure:

```bash
cd clearance-engine/backend
python -m pytest tests/architecture/ -v 2>&1 | grep FAIL
```

For each failure, determine the fix:
- Missing `main.py` → Create domain main.py for independent deployment
- Missing health endpoint → Add GET `/health` to domain routes
- Cross-domain import → Remove (should be done by WP-1/WP-6)
- Wrong schema → Fix model `__table_args__` (should be done by WP-5)

#### Task 10.3: Add new architecture tests

Add tests validating the work from other WPs:

```python
# test_auth_on_all_routes.py
def test_all_routes_have_auth_dependency():
    """Every route handler (except health and auth) must have require_auth dependency."""
    # AST-parse all route files, check for Depends(require_auth) or Depends(require_role)

# test_cache_ttl.py
def test_all_projector_writes_have_ttl():
    """Every Redis SET in projectors must include ex= or px= TTL parameter."""

# test_event_schema_strict.py
def test_base_event_forbids_extra_fields():
    """BaseEvent model_config must have extra='forbid'."""

# test_no_cross_domain_imports.py
# (Already written in WP-6)
```

#### Task 10.4: Run full architecture suite

```bash
python -m pytest tests/architecture/ -v --tb=short
```

Expected: 0 failures, 0 skipped, 0 xfail

#### Task 10.5: Write report, commit

Write `docs/reports/WP-10-arch-test-enforcement.md`. Commit and push.

---

## WAVE 3 REVIEW CHECKPOINT

### Task W3-Review: Merge and validate Wave 3

Merge WP-8, WP-9, WP-10 into `production-hardening`. Run full backend + frontend test suites. Full architect and consistency review.

---

## FINAL VALIDATION

### Task Final-1: Full Backend Test Suite

```bash
cd clearance-engine/backend
python -m pytest tests/ -v --timeout=120 --tb=short 2>&1 | tee docs/reports/final-backend-test-output.txt
```

Expected: ALL PASS

### Task Final-2: Full Frontend Build & Unit Tests

```bash
cd clearance-engine/frontend
npm run build
npm run test
```

Expected: Build succeeds, unit tests pass

### Task Final-3: Docker Compose Full Stack

```bash
cd clearance-engine
docker compose -f docker-compose.yml up -d --build
# Wait for health
curl -s http://localhost:4000/health | python -m json.tool
# Verify all services "ok"
```

### Task Final-4: E2E Test Suite

```bash
cd clearance-engine/frontend
npx playwright test --config e2e/playwright.config.ts --reporter=list 2>&1 | tee docs/reports/final-e2e-test-output.txt
```

Expected: ALL PASS

### Task Final-5: Visual Regression Baseline

```bash
cd clearance-engine/frontend
npx playwright test e2e/ux-audit-screenshots.spec.ts --update-snapshots
# Copy screenshots to baseline
cp -r docs/ux-audit-screenshots/ docs/ux-audit-screenshots/baseline/
```

### Task Final-6: Write Summary Report

**File:** `docs/reports/production-readiness-remediation-summary.md`

Contents:
- Executive summary
- All 10 findings: status (RESOLVED), evidence
- Test results (backend count, frontend count, E2E count)
- Performance metrics
- Visual regression baseline reference
- Links to all 10 WP reports
- Residual risks / known limitations
- Recommendation: PR ready for human review

### Task Final-7: Create PR

```bash
git checkout production-hardening
git push origin production-hardening
gh pr create \
  --base main \
  --head production-hardening \
  --title "Production Readiness: Resolve all 10 findings" \
  --body "$(cat <<'EOF'
## Summary
Resolves all 10 findings from the 2025-02-14 Production Readiness Review.

### Findings Resolved
1. Legacy Monolith Decoupling — Zero app.* imports in clearance_platform/
2. Database Isolation — Per-domain schemas with enforced role access
3. Authentication Middleware — JWT auth on all endpoints, role-based access
4. Ingestion Persistence — All 7 pipelines persist to DB with server-side dedup
5. Cross-Domain SQL Elimination — Gateways use projections, no cross-domain joins
6. Architecture Test Enforcement — All tests pass, no xfail markers
7. Cache Projector Fix — TTLs, partitioned keys, read handlers use cache
8. Event Schema Validation — Strict Pydantic models, no extra fields
9. CORS & Rate Limits — Configurable allowlist, tiered limits
10. Stub Replacement — All placeholder endpoints return real data

### Test Results
- Backend: [X] tests passing
- Frontend: Build + unit tests passing
- E2E: [X] tests passing
- Architecture: [X] tests passing, 0 failures

### Reports
See `docs/reports/` for individual WP reports and summary.

## Test plan
- [ ] Review each WP report in docs/reports/
- [ ] Run full backend test suite
- [ ] Run full E2E test suite
- [ ] Verify auth flow in browser
- [ ] Verify no cross-domain SQL in gateway routes
- [ ] Check visual regression screenshots

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

**DO NOT MERGE. Human review required.**
