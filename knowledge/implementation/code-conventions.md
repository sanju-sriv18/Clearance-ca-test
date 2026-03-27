# Clearance Intelligence Engine -- Code Conventions

Coding standards, patterns, and conventions enforced across the codebase.

Last updated: 2026-02-03

---

## 1. Python Conventions (Backend)

### 1.1 General Style

- **Python version:** 3.12 (use modern syntax: `X | Y` unions, `type` aliases).
- **Formatter:** Ruff (`ruff format .`).
- **Linter:** Ruff (`ruff check .`).
- **Line length:** 88 characters (Ruff default).
- **Quotes:** Double quotes for strings.
- **Docstrings:** Google/Sphinx hybrid style with explicit `Args:`, `Returns:`, `Raises:` sections. Module-level docstrings on every file explaining purpose and key classes.

```python
"""Async database session management for PostgreSQL.

Wraps SQLAlchemy 2.0 async engine and session creation, providing both
async (for FastAPI request handlers) and sync (for Alembic migrations
and one-off scripts) session factories.
"""
```

### 1.2 Imports

All modules use `from __future__ import annotations` as the first import line to enable PEP 604 union syntax everywhere. Imports are organized in three groups separated by blank lines:

1. Standard library
2. Third-party packages
3. Local application imports

```python
from __future__ import annotations

import asyncio
import time
from collections.abc import AsyncIterator
from typing import Any, TYPE_CHECKING

import structlog
from fastapi import FastAPI, Request

from app.config import get_settings
from app.engines.base import EngineOutput
```

Use `TYPE_CHECKING` guards for imports only needed by type checkers to avoid circular imports:

```python
if TYPE_CHECKING:
    from app.config import Settings
```

### 1.3 Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Modules | `snake_case` | `error_handling.py`, `fuzzy_match.py` |
| Classes | `PascalCase` | `AnalysisPipeline`, `CacheService` |
| Functions/methods | `snake_case` | `execute_tool()`, `search_rulings()` |
| Constants | `UPPER_SNAKE_CASE` | `COLLECTION_NAME`, `VECTOR_DIM` |
| Private methods | `_leading_underscore` | `_dispatch()`, `_safe_output()` |
| Engine IDs | `E{n}` short string | `"E1"`, `"E2"` |
| Config classes | `{Thing}Config` | `CustomsConfig`, `SimulationConfig` |
| ORM models | `PascalCase` singular | `Shipment`, `HTSUSHeading` |
| Table names | `snake_case` plural | `"htsus_headings"`, `"restricted_parties"` |
| Index names | `ix_{table}_{column}` | `"ix_restricted_parties_entity_name_trgm"` |

### 1.4 Type Hints

All function signatures have complete type annotations. Return types are always specified. Use modern union syntax:

```python
async def execute(
    self,
    product_description: str,
    origin_country: str,
    destination_country: str,
    declared_value: float,
    currency: str = "USD",
    entity_name: str = "",
) -> dict[str, Any]:
```

For optional values, use `X | None` (not `Optional[X]`):

```python
self.openai: AsyncOpenAI | None = (
    AsyncOpenAI(api_key=config.OPENAI_API_KEY)
    if config.OPENAI_API_KEY
    else None
)
```

ORM models use SQLAlchemy `Mapped[T]` annotations with explicit `mapped_column()`:

```python
hs_code: Mapped[str] = mapped_column(String(15), nullable=False)
aliases: Mapped[Optional[dict]] = mapped_column(JSONB, nullable=True)
```

### 1.5 Async Patterns

The entire backend is async-first. Key patterns:

**Session context manager** -- sessions are obtained via `async with session_factory()` and auto-close:

```python
async with session_factory() as session:
    stmt = select(Shipment).where(Shipment.id == sid)
    result = await session.execute(stmt)
    shipment = result.scalar_one_or_none()
```

**Parallel execution** -- `asyncio.gather` with `return_exceptions=True` to capture individual failures:

```python
e2_raw, e3_raw, e4_raw = await asyncio.gather(
    self.e2.execute(...),
    self.e3.execute(...),
    self.e4.execute(...),
    return_exceptions=True,
)
```

**Async generators for SSE** -- pipeline and classification engines yield event dicts:

```python
async def stream(self, ...) -> AsyncIterator[dict[str, Any]]:
    async for event in self.e1.stream_classify(product_description, origin_country):
        yield event
```

**Background tasks** -- simulation actors run as `asyncio.Task` instances created via `asyncio.create_task()`:

```python
self._dashboard_task = asyncio.create_task(self._dashboard_loop())
```

### 1.6 Error Handling

**Fail-open for infrastructure services.** Redis and Qdrant operations are always wrapped in try/except blocks. Failures log a warning and return a degraded result rather than raising:

```python
async def get(self, key: str) -> Any | None:
    try:
        raw: str | None = await self.redis.get(key)
        if raw is None:
            return None
        return json.loads(raw)
    except Exception as exc:
        logger.warning("cache_get_failed", key=key, error=str(exc))
        return None
```

**Exception-to-EngineOutput conversion.** Engine exceptions from `asyncio.gather` are converted to error-status `EngineOutput` objects so the pipeline always returns partial results:

```python
def _safe_output(result: EngineOutput | BaseException, engine_id: str) -> EngineOutput:
    if isinstance(result, BaseException):
        logger.error("engine_exception_in_pipeline", engine_id=engine_id, error=str(result))
        return EngineOutput(engine_id=engine_id, status="error", errors=[str(result)])
    return result
```

**Critical vs non-critical failure.** E1 (Classification) and E2 (Tariff) failures are critical -- the pipeline cannot produce meaningful results. E3 (Compliance) and E4 (FTA) failures are non-critical -- the pipeline returns partial results with the failed engine's data set to its error envelope.

**LLM failover.** The `LLMService` tries the primary provider, catches any exception, logs a warning, and retries with the fallback provider. Only if both providers fail does the exception propagate:

```python
try:
    return await primary_fn(system_prompt, user_prompt, temp, tokens)
except Exception as exc:
    logger.warning("primary_llm_failed", provider=self._primary, error=str(exc))
    if fallback_fn is not None:
        return await self._try_fallback(fallback_fn, ...)
    raise
```

**Deterministic fallback for LLM-dependent engines.** E0 (PreClearance) has a deterministic fallback path that uses pg_trgm fuzzy matching and static risk scoring when the LLM is unavailable.

### 1.7 Logging with structlog

All modules use `structlog.get_logger(__name__)` and structured key-value logging. Log levels follow a consistent pattern:

```python
logger = structlog.get_logger(__name__)

# Info: normal operations
logger.info("startup_complete")
logger.info("pipeline_complete", hs_code=hs_code, e1_status=e1_result.status)

# Warning: degraded but functional
logger.warning("redis_unavailable", error=str(exc))
logger.warning("primary_llm_failed", provider=self._primary, error=str(exc))

# Error: failures that need attention
logger.error("engine_exception_in_pipeline", engine_id=engine_id, error=str(result))
logger.error("all_llm_providers_failed", error=str(fallback_exc))
```

Every log event has a descriptive `snake_case` name as the first positional argument, followed by keyword arguments providing context. Sensitive data (API keys, passwords) is never logged. Database URLs are logged with credentials stripped (`url=settings.DATABASE_URL.split("@")[-1]`).


## 2. TypeScript Conventions (Frontend)

### 2.1 General Style

- **TypeScript:** Strict mode (`"strict": true` in tsconfig).
- **Linter:** ESLint.
- **Formatter:** Prettier (via `npm run format`).
- **Path aliases:** `@/` maps to `src/`.
- **File extensions:** `.tsx` for files with JSX, `.ts` for pure TypeScript.

### 2.2 Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Components | `PascalCase` | `ControlTower.tsx`, `ProductInputForm.tsx` |
| Hooks | `camelCase` with `use` prefix | `useAnalysis.ts`, `usePipelineAnalysis.ts` |
| Stores | `camelCase` with `Store` suffix | `pipelineStore.ts`, `analysisStore.ts` |
| Interfaces/Types | `PascalCase` | `ClearanceEntry`, `ProductInput` |
| Functions | `camelCase` | `streamAnalysis()`, `transformTariff()` |
| Constants | `UPPER_SNAKE_CASE` | `BASE_URL`, `DEMO_AGGREGATES` |
| CSS classes | Tailwind utility classes (no custom CSS) | `"bg-white rounded-lg shadow-sm"` |

### 2.3 Component Patterns

**Functional components only** -- no class components. All components are exported as named exports (not default) where they are referenced by multiple consumers:

```tsx
export function ControlTower() {
  // ...
  return <div>...</div>;
}
```

Layout components (`PlatformLayout`, `ShipperLayout`, `BuyerLayout`) use React Router `<Outlet />` for nested routing.

**Framer Motion** is used for page transitions and micro-interactions. The `App.tsx` wraps all routes in `<AnimatePresence mode="wait">`.

### 2.4 Hook Patterns

Custom hooks encapsulate API calls, SSE streaming, and store interactions. They manage lifecycle concerns (abort controllers, cleanup, error state):

```tsx
export function usePipelineAnalysis() {
  const pipeline = usePipelineStore();
  const analysisStore = useAnalysisStore();
  const abortRef = useRef<(() => void) | null>(null);

  const analyze = useCallback(
    async (input: ProductInput, options: PipelineAnalysisOptions) => {
      abortRef.current?.();  // abort previous
      // ...stream and update stores...
    },
    [pipeline, analysisStore],
  );

  return { analyze, abort: () => abortRef.current?.() };
}
```

### 2.5 Store Patterns (Zustand)

Five Zustand stores provide client-side state. Each store follows this pattern:

1. **Types defined above the store** -- interfaces for state shape and entries.
2. **Demo data as module-level constants** -- pre-loaded for development.
3. **Store created with `create<StateType>()`** -- typed state + actions in a single object.
4. **Immutable updates via `set()`** -- never mutate state directly.
5. **Selector functions as methods** -- `getEntry(id)`, `getByStatus(...)`, `getExceptionQueue()`.

```tsx
interface PipelineState {
  entries: ClearanceEntry[];
  aggregates: PlatformAggregates;
  nextId: number;

  createEntry: (entry: Omit<ClearanceEntry, "id" | "createdAt">) => string;
  updateEntryStatus: (id: string, status: EntryStatus) => void;
  getEntry: (id: string) => ClearanceEntry | undefined;
  getExceptionQueue: () => ClearanceEntry[];
}

export const usePipelineStore = create<PipelineState>((set, get) => ({
  entries: makeDemoEntries(),
  // ...actions using set() for updates, get() for reads...
}));
```

### 2.6 SSE Consumption Pattern

The frontend consumes SSE streams via the Fetch API with `ReadableStream`, not the `EventSource` API. This allows POST requests with JSON bodies (EventSource only supports GET).

```typescript
const response = await fetch(url, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(payload),
  signal: controller.signal,
});

const reader = response.body?.getReader();
const decoder = new TextDecoder();
let buffer = "";

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  buffer += decoder.decode(value, { stream: true });
  const lines = buffer.split("\n");
  buffer = lines.pop() || "";

  for (const line of lines) {
    if (line.startsWith("event: ")) {
      currentEventType = line.slice(7).trim();
    }
    if (line.startsWith("data: ")) {
      const data = line.slice(6).trim();
      if (data === "[DONE]") return;
      const payload = JSON.parse(data);
      // dispatch based on currentEventType...
    }
  }
}
```

Key patterns:
- 30-second AbortController timeout.
- Line buffering to handle partial SSE frames.
- `event:` line sets the current type; `data:` line carries the payload.
- `[DONE]` sentinel terminates the stream.
- De-duplication guards (`seenClassificationComplete`, `seenTariffComplete`).


## 3. API Schema Conventions

### 3.1 Case Convention Boundary

The backend uses `snake_case` for all JSON field names. The frontend uses `camelCase`. The transformation happens explicitly in `api/client.ts` via hand-written transform functions -- no automatic middleware or library is used.

**Backend Pydantic model (snake_case):**
```python
class EngineResult(BaseModel):
    engine_id: str
    status: str = "completed"
    processing_time_ms: float = 0.0
```

**Frontend TypeScript interface (camelCase):**
```typescript
interface ClassificationResult {
  htsCode: string;
  description: string;
  confidence: "HIGH" | "MEDIUM" | "LOW";
  reasoning: Array<{ step: number; title: string; content: string }>;
}
```

**Explicit transform function:**
```typescript
function transformClassification(raw: Record<string, unknown>): ClassificationResult {
  return {
    htsCode: (raw.hs_code as string) ?? "",
    description: (raw.hs_description as string) ?? "",
    confidence: (raw.confidence as "HIGH" | "MEDIUM" | "LOW") ?? "LOW",
    reasoning: steps.map((s, i) => ({
      step: i + 1,
      title: `Step ${i + 1}`,
      content: typeof s === "string" ? s : JSON.stringify(s),
    })),
  };
}
```

### 3.2 Pydantic Model Hierarchy

```
BaseModel (pydantic)
  |
  +-- EngineResult (common.py) -- base for all engine responses
  +-- ErrorResponse (common.py) -- standardized error payload
  +-- SessionInfo (common.py)   -- session descriptor
  +-- TradeLaneRequest (trade_lanes.py)
  +-- RegulatorySignalResponse (regulatory.py)
  +-- Settings (config.py, pydantic-settings) -- configuration
```

### 3.3 Request/Response Patterns

All POST endpoints accept JSON bodies. Field names are `snake_case`:

```json
{
  "product_description": "EV lithium-ion battery pack",
  "origin_country": "CN",
  "destination_country": "US",
  "declared_value": 12500,
  "currency": "USD",
  "entity_name": "Apex Trading Co."
}
```

Engine results follow the `EngineOutput` envelope pattern:

```json
{
  "engine_id": "E2",
  "status": "success",
  "data": { "total_duty": 4800, "landed_cost": 17300, "line_items": [...] },
  "processing_time_ms": 42.3,
  "errors": [],
  "warnings": []
}
```

Assembled pipeline results flatten the envelope:

```json
{
  "classification": { "hs_code": "8507.60", ... },
  "tariff": { "total_duty": 4800, ... },
  "compliance": { "overall_risk": "LOW", ... },
  "fta": { "eligible": false, ... },
  "summary": { "hs_code": "8507.60", "landed_cost": 17300 },
  "engine_timings": { "E1": 1200.5, "E2": 42.3, "E3": 89.1, "E4": 15.7 }
}
```

### 3.4 Error Response Format

All error responses use the `ErrorResponse` schema:

```json
{
  "error": "classification_failed",
  "detail": "Unable to determine HS code for the given description.",
  "request_id": "abc123",
  "timestamp": "2025-01-15T08:00:00Z",
  "context": { "engine_id": "E1", "product_description": "..." }
}
```


## 4. Database Patterns

### 4.1 ORM Model Base

All models inherit from `Base` (SQLAlchemy `DeclarativeBase`). Most also use `TimestampMixin` which adds:

```python
created_at: Mapped[datetime] = mapped_column(
    DateTime(timezone=True), server_default=func.now(), nullable=False,
)
updated_at: Mapped[datetime] = mapped_column(
    DateTime(timezone=True), server_default=func.now(), onupdate=func.now(), nullable=False,
)
```

Timestamps use `TIMESTAMP WITH TIME ZONE` and are populated server-side.

### 4.2 JSONB Usage

JSONB columns are used for semi-structured data that varies by context:

| Table | JSONB Columns | Purpose |
|-------|--------------|---------|
| `shipments` | `events`, `waypoints`, `codes`, `financials`, `analysis` | Flexible event log and nested analysis results |
| `orders` | `analysis`, `transit_points`, `document_status` | Order analysis cache and document tracking |
| `products` | `hs_codes`, `source_locations`, `cached_analysis` | Multi-code classification and source data |
| `restricted_parties` | `aliases`, `addresses`, `identification`, `programs` | Varied entity metadata |
| `pga_mappings` | `requirements` | Agency-specific requirement structures |
| `fta_rules` | `documentation` | Rule-specific documentation lists |
| `tax_regime_templates` | `template` | Ordered computation step definitions |
| `section_232_scope` | `country_exemptions` | Per-country exemption data |
| `adcvd_orders` | `hs_codes` | List of affected HS codes |
| `regulatory_signals` | `affected_hs_codes`, `affected_countries` | Arrays of affected codes/countries |

When modifying JSONB columns on existing ORM objects, use `flag_modified()`:

```python
from sqlalchemy.orm.attributes import flag_modified

events = list(shipment.events or [])
events.append(new_event)
shipment.events = events
flag_modified(shipment, "events")
await session.commit()
```

### 4.3 pg_trgm Fuzzy Matching

The `restricted_parties.entity_name` column has a `gin_trgm_ops` GIN index for fast fuzzy matching. This is created by the `pg_trgm` extension (initialized in `data/init.sql`).

```python
Index(
    "ix_restricted_parties_entity_name_trgm",
    "entity_name",
    postgresql_using="gin",
    postgresql_ops={"entity_name": "gin_trgm_ops"},
)
```

Queries use the `similarity()` function or `%` operator for fuzzy matching with configurable thresholds.

### 4.4 Index Strategy

Indexes are explicitly defined in `__table_args__` on each model (no implicit index generation):

- **B-tree indexes** (default): for exact lookups and range queries on scalar columns.
- **GIN indexes**: for JSONB containment queries (`affected_hs_codes`, `hs_codes`) and pg_trgm fuzzy text.
- **Composite indexes**: for multi-column lookups (`(hs_code, jurisdiction)`, `(jurisdiction, tax_type)`).
- **Unique constraints**: enforced at the database level (e.g., `(hs_code, jurisdiction)` on `htsus_headings`).

Naming convention: `ix_{table}_{column_names}` for indexes, `uq_{table}_{column_names}` for unique constraints.

### 4.5 Primary Key Strategy

- **Tariff/Compliance/Reference tables**: Auto-incrementing `Integer` primary keys. These tables have natural keys (e.g., `ruling_number`, `hs_code+jurisdiction`) enforced via unique constraints.
- **Operational tables** (`products`, `orders`, `shipments`, `documents`): `UUID` primary keys generated client-side via `uuid4()`. This supports distributed ID generation and avoids sequential ID exposure.

### 4.6 Migrations

Alembic is configured in `backend/alembic.ini` and `backend/alembic/env.py`. All ORM models are imported in `env.py` for autogenerate support.

```bash
# Create a new migration
make migrate-new msg="add foo column"

# Apply migrations
make migrate

# Rollback one step
make migrate-down
```


## 5. Engine Patterns

### 5.1 BaseEngine Contract

Every engine inherits from `BaseEngine` and implements the async `execute()` method:

```python
class ClassificationEngine(BaseEngine):
    engine_id = "E1"
    engine_name = "Classification Intelligence"

    async def execute(self, **kwargs) -> EngineOutput:
        start = self._start_timer()
        # ... engine logic ...
        return EngineOutput(
            engine_id=self.engine_id,
            status="success",
            data={"hs_code": "8507.60", ...},
            processing_time_ms=self._elapsed_ms(start),
        )
```

### 5.2 EngineOutput Envelope

All engine results use the same `EngineOutput` dataclass:

```python
@dataclass(slots=True)
class EngineOutput:
    engine_id: str                              # "E1", "E2", etc.
    status: str = "success"                     # "success", "error", "partial"
    data: dict[str, Any] = field(default_factory=dict)
    processing_time_ms: float = 0.0
    errors: list[str] = field(default_factory=list)
    warnings: list[str] = field(default_factory=list)
    metadata: dict[str, Any] = field(default_factory=dict)
```

**Status values:**
- `"success"` -- engine completed normally.
- `"error"` -- engine failed entirely (data is empty, errors list populated).
- `"partial"` -- engine produced some results but with caveats (warnings populated).

### 5.3 Engine Dependencies

| Engine | Needs DB | Needs LLM | Needs Qdrant | Needs Redis |
|--------|----------|-----------|--------------|-------------|
| E0 PreClearance | Yes | Yes (agentic) | No | No |
| E1 Classification | Optional | Yes (two-phase) | No | Optional (cache) |
| E2 Tariff | Yes | No | No | Optional (cache) |
| E3 Compliance | Yes | Optional (UFLPA) | No | Optional (cache) |
| E4 FTA | Yes | No | No | No |
| E5 Exception | Optional | Yes (drafting) | Yes (search) | Optional (cache) |
| E6 Regulatory | Yes | Yes (scenario) | No | No |
| E7 Documents | Optional | Yes (generation) | No | No |


## 6. SSE Streaming Patterns

### 6.1 Backend SSE Helpers

The `api/streaming.py` module provides two core utilities:

```python
def sse_event(event_type: str, data: dict) -> str:
    """Format a single SSE frame."""
    payload = json.dumps(data, default=str)
    return f"event: {event_type}\ndata: {payload}\n\n"

def sse_response(generator: AsyncIterator[str]) -> StreamingResponse:
    """Wrap an async generator in a text/event-stream response."""
    return StreamingResponse(
        content=generator,
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",
        },
    )
```

### 6.2 Event Type Constants

Defined in `streaming.py` for consistency:

```python
CLASSIFICATION_STEP = "classification_step"
CLASSIFICATION_COMPLETE = "classification_complete"
TARIFF_PROGRAM = "tariff_program"
TARIFF_COMPLETE = "tariff_complete"
COMPLIANCE_PGA = "compliance_pga"
COMPLIANCE_DPS = "compliance_dps"
COMPLIANCE_UFLPA = "compliance_uflpa"
ANALYSIS_COMPLETE = "analysis_complete"
ENGINE_ERROR = "engine_error"
HEARTBEAT = "heartbeat"
```

### 6.3 Streaming Lifecycle

1. Frontend opens POST request to SSE endpoint.
2. Backend returns `StreamingResponse` with `text/event-stream` media type.
3. Pipeline yields typed event dicts from engines.
4. Route handler converts each dict to an SSE frame via `sse_event()`.
5. `data: [DONE]` sentinel is sent when the pipeline completes.
6. Frontend closes the connection via AbortController.

### 6.4 Chat Streaming

The chat SSE stream has a different payload format from the analysis stream:

```
data: {"text": "Based on my analysis..."}     -- text chunk
data: {"tool": "lookup_shipment", "status": "running", "label": "Looking up shipment abc"}
data: {"tool": "lookup_shipment", "status": "complete"}
data: {"text": "The shipment is currently..."}
data: {"error": "Tool execution failed"}      -- error event
data: [DONE]                                   -- stream end
```


## 7. Error Handling Philosophy

### 7.1 Degradation Hierarchy

The system is designed to degrade gracefully through four levels:

1. **Full operation** -- all services healthy, LLM available.
2. **Cache degraded** -- Redis down. Cache misses everywhere, simulation event bus falls back to in-memory. No user-visible impact except slightly slower responses.
3. **Vector search degraded** -- Qdrant down. CROSS ruling search returns empty results. E5 Exception engine returns empty match list. All other engines unaffected.
4. **LLM degraded** -- Primary provider fails, automatic failover to fallback. If both fail, LLM-dependent engines (E0, E1, E5, E6, E7) fall back to deterministic logic where available (E0 has full deterministic fallback; others return error status).

### 7.2 Never Crash on Infrastructure Failure

Every external service call (Redis, Qdrant, LLM) is wrapped in try/except. The philosophy is:
- Log the failure with structured context.
- Return a degraded but valid response.
- Never let infrastructure failures cascade to 500 errors for the user.

### 7.3 Pipeline Partial Results

The analysis pipeline always returns results, even if individual engines fail. The `_safe_output()` function converts exceptions to error-status `EngineOutput` objects, and the `_assemble()` function merges them into the final result. The frontend checks each engine's status independently and renders available data.


## 8. File Organization Rules

### 8.1 Backend Directory Structure

```
app/
  main.py                     Application entry point (one file)
  config.py                   Configuration (one file)
  engines/                    One sub-package per engine (e0-e7)
    base.py                   Shared base class
    e{n}_{name}/
      __init__.py             Package marker + convenience imports
      engine.py               Engine implementation
      prompts.py              LLM prompt templates (if applicable)
      *.py                    Engine-specific utilities
  orchestration/              Pipeline coordination
  api/
    routes/                   One file per route module (18 total)
    schemas/                  Pydantic request/response models
    agent/                    Agent configurations and tools
    streaming.py              SSE utilities
  services/                   Infrastructure service wrappers
  knowledge/
    models/                   ORM models grouped by domain
    repositories/             Repository pattern (placeholder)
  simulation/
    coordinator.py            Simulation lifecycle
    actors/                   One file per actor
    *.py                      Clock, event bus, state machine, config
```

### 8.2 Frontend Directory Structure

```
src/
  main.tsx, App.tsx           Application entry
  router.tsx                  Route definitions
  types.ts                    Shared type definitions
  api/                        API client (single file)
  store/                      Zustand stores (one per domain)
  hooks/                      Custom hooks (one per concern)
  components/                 Shared/global components
  surfaces/
    {surface}/
      {Surface}Layout.tsx     Layout wrapper
      screens/                One file per screen
      components/             Surface-specific components
  lib/                        Pure utility functions
  data/                       Static data
  test/                       Test files
```

### 8.3 Naming Patterns for New Files

- **New engine:** `engines/e{n}_{name}/engine.py` implementing `BaseEngine`.
- **New route:** `api/routes/{name}.py` with a `router = APIRouter()` that is imported and registered in `main.py`.
- **New actor:** `simulation/actors/{name}.py` implementing `BaseActor`, registered in `coordinator.py`.
- **New store:** `store/{name}Store.ts` with `export const use{Name}Store = create<{Name}State>(...)`.
- **New hook:** `hooks/use{Name}.ts` with `export function use{Name}()`.
- **New screen:** `surfaces/{surface}/screens/{ScreenName}.tsx`.
- **New component:** `surfaces/{surface}/components/{ComponentName}.tsx` or `components/{ComponentName}.tsx` for shared.

### 8.4 Package Markers

Every Python package has an `__init__.py`. These are either empty or contain convenience imports for the package's public API:

```python
# engines/e1_classification/__init__.py
"""Classification engine package."""

from app.engines.e1_classification.engine import ClassificationEngine

async def classify(product_description: str, origin_country: str = "CN") -> dict:
    """Convenience function for standalone classification."""
    engine = ClassificationEngine()
    result = await engine.execute(
        product_description=product_description,
        origin_country=origin_country,
    )
    return result.data
```


## 9. Testing Conventions

### 9.1 Backend Testing (pytest)

- **Test directory structure** mirrors `app/` structure.
- **Fixtures** in `conftest.py`: async engine, session, test client.
- **Unit tests** (`tests/unit/`): mock external dependencies (LLM, DB).
- **Integration tests** (`tests/integration/`): use real DB, mock LLM.
- **Data validation tests** (`tests/data_validation/`): verify seeded data correctness.
- **E2E tests** (`tests/e2e/`): full stack golden path through all engines.

```bash
make test          # all tests
make test-unit     # unit only
make test-integration  # integration only
make test-data     # data validation
make test-e2e      # end-to-end
```

### 9.2 Frontend Testing (Vitest + MSW)

- **MSW (Mock Service Worker)** intercepts API calls in tests.
- **Regression tests** (`test/regression/`): smoke tests, cross-screen consistency, agent tools.
- **Integration tests** (`test/integration/`): live API tests (requires running backend).
- **E2E tests** (`e2e/`): Playwright browser tests.

### 9.3 Makefile Targets

All test and development commands are available via Makefile targets for consistency:

```makefile
make up             # docker compose up -d
make down           # docker compose down
make logs           # docker compose logs -f
make migrate        # alembic upgrade head
make seed           # run all data seeders
make test           # run all backend tests
make lint           # ruff check + npm run lint
make format         # ruff format + npm run format
make shell-backend  # docker compose exec backend bash
make shell-db       # psql into postgres
```
