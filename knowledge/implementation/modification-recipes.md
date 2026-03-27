# Modification Recipes

Step-by-step recipes for common modifications to the Clearance Intelligence Engine. Each recipe lists the files to create or modify, the code patterns to follow, and verification steps.

---

## Table of Contents

1. [Add a New Engine](#1-add-a-new-engine)
2. [Add a New API Route](#2-add-a-new-api-route)
3. [Add a New Frontend Screen](#3-add-a-new-frontend-screen)
4. [Add a New Surface](#4-add-a-new-surface)
5. [Add a New Agent Tool](#5-add-a-new-agent-tool)
6. [Add a New Agent Persona](#6-add-a-new-agent-persona)
7. [Add a New Simulation Actor](#7-add-a-new-simulation-actor)
8. [Add a New Database Table](#8-add-a-new-database-table)
9. [Add a New Jurisdiction/Tariff Regime](#9-add-a-new-jurisdictiontariff-regime)
10. [Add a New Compliance List](#10-add-a-new-compliance-list)
11. [Modify the Pipeline Order](#11-modify-the-pipeline-order)
12. [Add a New Zustand Store](#12-add-a-new-zustand-store)

---

## 1. Add a New Engine

Engines are the core computation units (E1 through E6). To add E7:

### Step 1: Create the engine module

```
backend/app/engines/e7_new_engine/
  __init__.py
  engine.py
```

### Step 2: Implement the engine class

```python
# backend/app/engines/e7_new_engine/engine.py

from __future__ import annotations
from typing import Any
import structlog
from app.engines.base import BaseEngine, EngineOutput

logger = structlog.get_logger(__name__)


class NewEngine(BaseEngine):
    engine_id: str = "E7"
    engine_name: str = "New Engine Name"

    def __init__(self, db_session: Any | None = None) -> None:
        self.db = db_session

    async def execute(self, **kwargs: Any) -> EngineOutput:
        start = self._start_timer()

        try:
            # Engine computation logic here
            result = {"key": "value"}

            return EngineOutput(
                engine_id=self.engine_id,
                status="success",
                data=result,
                processing_time_ms=self._elapsed_ms(start),
            )
        except Exception as exc:
            logger.error("e7_error", error=str(exc), exc_info=True)
            return EngineOutput(
                engine_id=self.engine_id,
                status="error",
                errors=[str(exc)],
                processing_time_ms=self._elapsed_ms(start),
            )
```

### Step 3: Create the convenience wrapper in `__init__.py`

```python
# backend/app/engines/e7_new_engine/__init__.py

from app.engines.e7_new_engine.engine import NewEngine

async def run_new_engine(**kwargs):
    engine = NewEngine()
    output = await engine.execute(**kwargs)
    return output
```

### Step 4: Add the engine to the pipeline

Edit `backend/app/api/routes/analysis.py` to add the E7 step in the `_run_pipeline` function, following the existing try/except/yield pattern.

### Step 5: Create an API route (optional -- if standalone access is needed)

See [Recipe 2: Add a New API Route](#2-add-a-new-api-route).

### Step 6: Add a Pydantic schema

Create a response schema in `backend/app/api/schemas/` if the engine has a standalone endpoint.

### Step 7: Write tests

```
backend/tests/unit/engines/test_e7_new_engine.py
```

Follow the patterns in `test_e1_classification.py`: validator tests, deterministic tests, mock tests, error path tests, metadata tests.

### Verify

```bash
make test-unit
```

---

## 2. Add a New API Route

### Step 1: Create the schema (if needed)

```python
# backend/app/api/schemas/new_feature.py

from pydantic import BaseModel

class NewFeatureRequest(BaseModel):
    param: str

class NewFeatureResponse(BaseModel):
    result: str
    status: str = "success"
```

### Step 2: Create the route file

```python
# backend/app/api/routes/new_feature.py

from __future__ import annotations
import structlog
from fastapi import APIRouter, Request
from app.api.schemas.new_feature import NewFeatureRequest, NewFeatureResponse

logger = structlog.get_logger(__name__)
router = APIRouter(prefix="/api", tags=["new_feature"])


@router.post(
    "/new-feature",
    response_model=NewFeatureResponse,
    summary="Description of the endpoint",
)
async def new_feature(payload: NewFeatureRequest, request: Request) -> NewFeatureResponse:
    logger.info("new_feature_called", param=payload.param)
    # Access infrastructure via request.app.state:
    #   request.app.state.db_session_factory
    #   request.app.state.redis
    #   request.app.state.qdrant
    return NewFeatureResponse(result="computed_value")
```

### Step 3: Register the router in `main.py`

Edit `backend/app/main.py` in the `create_app()` function:

```python
from app.api.routes.new_feature import router as new_feature_router
application.include_router(new_feature_router)
```

### Step 4: Add MSW handler (if frontend will call this)

```typescript
// frontend/src/test/mocks/handlers.ts
http.post("/api/new-feature", () => {
  return HttpResponse.json({ result: "mock-value", status: "ok" });
}),
```

### Verify

```bash
curl -X POST http://localhost:4000/api/new-feature \
  -H "Content-Type: application/json" \
  -d '{"param": "test"}'
```

Check the auto-generated docs at `http://localhost:4000/docs`.

---

## 3. Add a New Frontend Screen

### Step 1: Create the screen component

```typescript
// frontend/src/surfaces/platform/screens/NewScreen.tsx

export function NewScreen() {
  return (
    <div className="p-6 space-y-6">
      <h1 className="text-2xl font-bold text-slate-100">New Screen</h1>
      {/* Screen content */}
    </div>
  );
}
```

### Step 2: Add the route

Edit `frontend/src/router.tsx`:

```typescript
import { NewScreen } from "@/surfaces/platform/screens/NewScreen";

// Inside the platform children array:
{ path: "new-screen", element: <NewScreen /> },
```

### Step 3: Add navigation link

Edit the surface layout that contains the sidebar or nav. For the platform surface, edit `frontend/src/surfaces/platform/PlatformLayout.tsx` to add a nav item.

### Step 4: Add a no-crash regression test

```typescript
// In frontend/src/test/regression/no-crash.test.tsx
it("NewScreen renders", async () => {
  const { NewScreen } = await import(
    "@/surfaces/platform/screens/NewScreen"
  );
  const { container } = wrap(<NewScreen />);
  expect(container).toBeTruthy();
});
```

### Verify

```bash
cd frontend && npm test
open http://localhost:4001/platform/new-screen
```

---

## 4. Add a New Surface

Surfaces are top-level user experiences (platform, shipper, buyer). Each has its own layout, routes, and navigation.

### Step 1: Create the surface directory

```
frontend/src/surfaces/newsurface/
  NewSurfaceLayout.tsx
  screens/
    Dashboard.tsx
```

### Step 2: Create the layout

```typescript
// frontend/src/surfaces/newsurface/NewSurfaceLayout.tsx

import { Outlet, NavLink } from "react-router-dom";

export function NewSurfaceLayout() {
  return (
    <div className="flex h-screen bg-slate-950 text-slate-100">
      {/* Sidebar navigation */}
      <nav className="w-56 border-r border-slate-800 p-4">
        <h2 className="text-lg font-bold mb-4">New Surface</h2>
        <NavLink to="/newsurface/dashboard" className="block py-2">
          Dashboard
        </NavLink>
      </nav>

      {/* Main content area */}
      <main className="flex-1 overflow-y-auto">
        <Outlet />
      </main>
    </div>
  );
}
```

### Step 3: Create the initial screen

```typescript
// frontend/src/surfaces/newsurface/screens/Dashboard.tsx

export function NewSurfaceDashboard() {
  return (
    <div className="p-6">
      <h1 className="text-2xl font-bold">New Surface Dashboard</h1>
    </div>
  );
}
```

### Step 4: Add routes

Edit `frontend/src/router.tsx`:

```typescript
import { NewSurfaceLayout } from "@/surfaces/newsurface/NewSurfaceLayout";
import { NewSurfaceDashboard } from "@/surfaces/newsurface/screens/Dashboard";

// Add to the routes array:
{
  path: "/newsurface",
  element: <NewSurfaceLayout />,
  children: [
    { index: true, element: <Navigate to="/newsurface/dashboard" replace /> },
    { path: "dashboard", element: <NewSurfaceDashboard /> },
  ],
},
```

### Step 5: Add regression tests

Add no-crash tests for each screen in the new surface.

### Verify

```bash
open http://localhost:4001/newsurface/dashboard
```

---

## 5. Add a New Agent Tool

Agent tools are functions the LLM can invoke during the agentic chat loop. The system currently has 11 tools (8 read + 3 write).

### Step 1: Define the tool schema

Edit `backend/app/api/agent/tools.py`. Add a new entry to the `TOOL_SCHEMAS` list:

```python
{
    "name": "new_tool_name",
    "description": (
        "What this tool does. Be specific -- the LLM uses this description "
        "to decide when to call the tool."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "param_name": {
                "type": "string",
                "description": "What this parameter means.",
            },
        },
        "required": ["param_name"],
    },
},
```

### Step 2: Implement the tool execution

In the same file, add the execution function and register it in the `execute_tool` dispatcher:

```python
async def _execute_new_tool(
    params: dict[str, Any],
    session_factory: Any,
) -> str:
    """Execute the new tool and return a string result for the LLM."""
    param_value = params.get("param_name", "")

    # Perform the operation (DB query, engine call, etc.)
    async with session_factory() as session:
        result = await session.execute(...)

    # Return a string -- the LLM reads this as tool output
    return json.dumps({"result": "value"})
```

Then add the tool to the `execute_tool` function's dispatch map:

```python
async def execute_tool(
    tool_name: str,
    tool_input: dict[str, Any],
    session_factory: Any,
) -> str:
    # ... existing dispatch ...
    if tool_name == "new_tool_name":
        return await _execute_new_tool(tool_input, session_factory)
    # ...
```

### Step 3: Add a label for the tool progress indicator

In `get_tool_label`, add a case for the new tool:

```python
def get_tool_label(tool_name: str, tool_input: dict[str, Any]) -> str:
    # ... existing cases ...
    if tool_name == "new_tool_name":
        return f"Running new tool for {tool_input.get('param_name', '...')}"
    # ...
```

### Step 4: Write tests

```python
# backend/tests/unit/test_agent.py
@pytest.mark.asyncio
async def test_new_tool_execution():
    result = await _execute_new_tool(
        {"param_name": "test"}, mock_session_factory
    )
    parsed = json.loads(result)
    assert "result" in parsed
```

### Verify

All agents automatically have access to the new tool since they all reference `TOOL_SCHEMAS`. Test by asking the agent a question that should trigger the tool.

---

## 6. Add a New Agent Persona

Agent personas are screen-specific LLM configurations with distinct system prompts and reasoning patterns.

### Step 1: Write the system prompt

Edit `backend/app/api/agent/prompts.py`. Create a new prompt constant following the existing pattern:

```python
NEW_SCREEN_PROMPT = _SHARED_INSTRUCTIONS + """\

You are the **New Screen Specialist**. You help operators with [specific domain].

REASONING PATTERN: [Step 1] -> [Step 2] -> [Step 3] -> [Step 4]

WHEN RESPONDING:
1. FIRST: [What to do first -- usually a tool call]
2. [Second step]
3. [Third step]
4. ALWAYS end with [specific output structure]

FACILITATION:
- Use [tool_name] to [action].
- Use [tool_name] when [condition].
"""
```

### Step 2: Register in the agent registry

Add the new config to `AGENT_REGISTRY` in the same file:

```python
AGENT_REGISTRY: dict[str, AgentConfig] = {
    # ... existing entries ...
    "new-screen-name": AgentConfig(
        name="New Screen Specialist",
        prompt=NEW_SCREEN_PROMPT,
        tools=TOOL_SCHEMAS,
    ),
}
```

### Step 3: Wire the frontend to send the screen identifier

When the frontend opens the chat from the new screen, it should include the screen name in the `ChatRequest.context.screen` field:

```typescript
const response = await fetch("/api/chat/stream", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    message: userMessage,
    context: { screen: "new-screen-name" },
  }),
});
```

The `get_agent_config("new-screen-name")` function in `prompts.py` will match the screen name to the registry entry. Unknown screen names fall back to the `"default"` (Senior Operations Specialist) agent.

### Verify

Open the chat on the new screen and ask a domain-specific question. Check the backend logs for `agent_selected` with the expected agent name.

---

## 7. Add a New Simulation Actor

Simulation actors drive the customs clearance simulation. Each actor runs a periodic tick loop.

### Step 1: Create the actor file

```python
# backend/app/simulation/actors/new_actor.py

from __future__ import annotations
from pydantic import BaseModel
import structlog
from app.simulation.actors.base import BaseActor

logger = structlog.get_logger(__name__)


class NewActorConfig(BaseModel):
    tick_sim_minutes: float = 30.0
    # Add actor-specific config fields


class NewActor(BaseActor):
    actor_id: str = "new_actor"

    async def tick(self) -> None:
        """Execute one tick of actor logic."""
        async with self.get_session() as session:
            # Query state, make decisions, update records
            logger.info("new_actor_tick", tick=self._tick_count)

            # Publish events to other actors via the event bus
            await self.event_bus.publish("new_actor.event", {
                "tick": self._tick_count,
            })
```

### Step 2: Register in the coordinator

Edit `backend/app/simulation/coordinator.py` in the `start()` method:

```python
from app.simulation.actors.new_actor import NewActor, NewActorConfig

# Inside the start() method, after existing actor creation:
new_actor = NewActor(
    clock=self.clock,
    event_bus=self.event_bus,
    session_factory=self.session_factory,
    config=NewActorConfig(),
    seed=self.config.random_seed,
)
self._actors["new_actor"] = new_actor
new_actor.start()
```

### Step 3: Subscribe to events (if needed)

If the new actor needs to react to events from other actors:

```python
async def tick(self) -> None:
    # Check for events
    events = await self.event_bus.consume("shipper.shipment_created")
    for event in events:
        # React to the event
        ...
```

### Verify

```bash
# Start the simulation
curl -X POST http://localhost:4000/api/simulation/start

# Check actor status
curl http://localhost:4000/api/simulation/status
```

---

## 8. Add a New Database Table

### Step 1: Create the SQLAlchemy model

```python
# backend/app/knowledge/models/new_table.py

from __future__ import annotations
from sqlalchemy import String, Float, Integer, ForeignKey
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column
from app.knowledge.models.base import Base, TimestampMixin


class NewTable(Base, TimestampMixin):
    __tablename__ = "new_table"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    value: Mapped[float] = mapped_column(Float, nullable=False)
    metadata_json: Mapped[dict] = mapped_column(JSONB, default={})
```

### Step 2: Ensure the model is discoverable by Alembic

Import the model in the models package or directly in `alembic/env.py`. The model must be imported before `Base.metadata` is accessed so Alembic can detect it.

### Step 3: Generate the migration

```bash
make migrate-new msg="create_new_table"
```

### Step 4: Review and apply the migration

Check the generated file in `backend/alembic/versions/`. Verify the `upgrade()` and `downgrade()` functions are correct.

```bash
make migrate
```

### Step 5: Create a seeder (if reference data is needed)

```python
# backend/data/ingestion/seed_new_table.py

import asyncio
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker
from app.config import get_settings
from app.knowledge.models.new_table import NewTable


async def seed():
    settings = get_settings()
    engine = create_async_engine(settings.DATABASE_URL)
    async_session = sessionmaker(engine, class_=AsyncSession)

    async with async_session() as session:
        records = [
            NewTable(name="Record 1", value=100.0),
            NewTable(name="Record 2", value=200.0),
        ]
        session.add_all(records)
        await session.commit()

    await engine.dispose()


if __name__ == "__main__":
    asyncio.run(seed())
```

### Step 6: Add a Make target

Edit `Makefile`:

```makefile
seed-new-table:
	cd backend && python -m data.ingestion.seed_new_table
```

Add to the `.PHONY` line and optionally to the `seed_all.py` runner.

### Verify

```bash
make migrate
make seed-new-table
make shell-db
# \d new_table
# SELECT * FROM new_table;
```

---

## 9. Add a New Jurisdiction/Tariff Regime

The E2 engine dispatches to jurisdiction-specific `TaxRegimeEngine` subclasses.

### Step 1: Create the regime file

```python
# backend/app/engines/e2_tariff/regimes/jp.py

from __future__ import annotations
from typing import Any
from app.engines.e2_tariff.tax_regime import TaxRegimeEngine, TariffResult, TariffLineItem


class JapanTaxRegime(TaxRegimeEngine):
    jurisdiction: str = "JP"
    pattern: str = "ADDITIVE"    # or "CASCADING"

    async def compute(
        self,
        hs_code: str,
        origin_country: str,
        destination_country: str,
        declared_value: float,
        currency: str = "JPY",
        entry_type: str = "FORMAL",
    ) -> dict[str, Any]:
        line_items: list[TariffLineItem] = []
        running_total = declared_value

        # Step 1: MFN Duty
        mfn_rate = 0.05  # Example: look up from DB
        mfn_amount = self._round(declared_value * mfn_rate)
        running_total += mfn_amount
        line_items.append(TariffLineItem(
            program="MFN Duty",
            description="Most Favoured Nation duty rate",
            rate=f"{mfn_rate * 100:.1f}%",
            rate_pct=mfn_rate * 100,
            amount=mfn_amount,
            citation="Customs Tariff Law of Japan",
            category="DUTY",
            base_value=declared_value,
            cumulative=running_total,
        ))

        # Step 2: Consumption Tax (10%)
        ct_rate = 0.10
        ct_base = declared_value + mfn_amount
        ct_amount = self._round(ct_base * ct_rate)
        running_total += ct_amount
        line_items.append(TariffLineItem(
            program="Consumption Tax",
            description="Japanese consumption tax",
            rate="10.0%",
            rate_pct=10.0,
            amount=ct_amount,
            citation="Consumption Tax Act",
            category="TAX",
            base_value=ct_base,
            cumulative=running_total,
        ))

        # Build result
        total_duty = sum(li.amount for li in line_items if li.category == "DUTY")
        total_taxes = sum(li.amount for li in line_items if li.category == "TAX")
        total_fees = sum(li.amount for li in line_items if li.category == "FEE")

        result = TariffResult(
            hs_code=hs_code,
            origin_country=origin_country,
            destination_country=destination_country,
            declared_value=declared_value,
            currency=currency,
            entry_type=entry_type,
            line_items=line_items,
            total_duty=total_duty,
            total_taxes=total_taxes,
            total_fees=total_fees,
            landed_cost=declared_value + total_duty + total_taxes + total_fees,
            effective_rate=(total_duty + total_taxes + total_fees) / declared_value if declared_value else 0,
        )

        return result.to_dict()
```

### Step 2: Register in the dispatch map

Edit `backend/app/engines/e2_tariff/engine.py`:

```python
from app.engines.e2_tariff.regimes.jp import JapanTaxRegime

REGIME_MAP: dict[str, type[TaxRegimeEngine]] = {
    "US": USTaxRegime,
    "EU": EUTaxRegime,
    "CN": ChinaTaxRegime,
    "BR": BrazilTaxRegime,
    "IN": IndiaTaxRegime,
    "JP": JapanTaxRegime,  # New
}
```

### Step 3: Create a seeder for jurisdiction-specific rates

```bash
# Create: backend/data/ingestion/seed_jp_tariff.py
# Add Make target: seed-jp
```

### Step 4: Write tests

Test the regime with known HS codes and verify landed cost arithmetic.

### Verify

```bash
make test-unit
curl -X POST http://localhost:4000/api/tariff \
  -H "Content-Type: application/json" \
  -d '{"hs_code":"8471.30","origin_country":"CN","destination_country":"JP","declared_value":1000,"currency":"JPY"}'
```

---

## 10. Add a New Compliance List

Compliance lists are used by the E3 Compliance Screening engine (DPS, PGA, UFLPA, etc.).

### Step 1: Create the database model (if needed)

```python
# backend/app/knowledge/models/compliance.py (extend existing)

class NewComplianceList(Base, TimestampMixin):
    __tablename__ = "new_compliance_list"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    entity_name: Mapped[str] = mapped_column(String(500), nullable=False, index=True)
    list_source: Mapped[str] = mapped_column(String(100), nullable=False)
    # Add list-specific fields
```

### Step 2: Generate and apply migration

```bash
make migrate-new msg="create_new_compliance_list"
make migrate
```

### Step 3: Create a seeder

```python
# backend/data/ingestion/seed_new_compliance_list.py
```

### Step 4: Integrate with E3 engine

Edit `backend/app/engines/e3_compliance/engine.py` to add a screening check against the new list. Follow the existing pattern of querying the list, running fuzzy matching with `rapidfuzz`, and returning results in the compliance output.

### Step 5: Add Make targets

```makefile
seed-new-list:
	cd backend && python -m data.ingestion.seed_new_compliance_list
```

### Verify

```bash
make seed-new-list
make test-unit   # Run E3 compliance tests
```

---

## 11. Modify the Pipeline Order

The analysis pipeline runs engines in sequence within `backend/app/api/routes/analysis.py` in the `_run_pipeline` function. The current order is: E1 (Classification) -> E2 (Tariff) -> E3 (Compliance) -> E4 (FTA).

### Changing the order

Each engine block in `_run_pipeline` is an independent try/except/yield block. To reorder:

1. Move the entire engine block (the `try:` section including the `except ImportError:` and `except Exception:` handlers).
2. Update variable dependencies. For example, E2 depends on E1's `classification.hs_code`. If you move E2 before E1, you need to handle the missing HS code.

### Adding a new engine to the pipeline

Insert a new engine block following the pattern:

```python
# -- E7: New Engine --
try:
    from app.engines.e7_new_engine import run_new_engine

    new_result = await run_new_engine(
        hs_code=classification.hs_code,
        # ... other params
    )
    yield sse_event("new_engine_complete", new_result.data)
except ImportError:
    logger.warning("e7_not_implemented")
    new_result = None
except Exception as exc:
    logger.error("e7_error", error=str(exc))
    yield sse_event(ENGINE_ERROR, {"engine": "e7_new_engine", "error": str(exc)})
    new_result = None
```

### Adding new SSE event types

Define new event type constants in `backend/app/api/streaming.py`:

```python
NEW_ENGINE_COMPLETE = "new_engine_complete"
```

Update the frontend to handle the new event type in the SSE stream parser.

### Verify

Run the full pipeline and confirm the SSE stream contains events in the expected order:

```bash
curl -X POST http://localhost:4000/api/analyze/stream \
  -H "Content-Type: application/json" \
  -d '{"product_description":"ceramic mug","origin_country":"CN","destination_country":"US","declared_value":100}'
```

---

## 12. Add a New Zustand Store

### Step 1: Create the store file

```typescript
// frontend/src/store/newFeatureStore.ts

import { create } from "zustand";

interface NewFeatureState {
  /* State */
  items: string[];
  loading: boolean;
  error: string | null;

  /* Actions */
  addItem: (item: string) => void;
  removeItem: (index: number) => void;
  setLoading: (loading: boolean) => void;
  setError: (error: string | null) => void;
  reset: () => void;
}

const initialState = {
  items: [] as string[],
  loading: false,
  error: null as string | null,
};

export const useNewFeatureStore = create<NewFeatureState>((set) => ({
  ...initialState,

  addItem: (item) =>
    set((state) => ({ items: [...state.items, item] })),

  removeItem: (index) =>
    set((state) => ({
      items: state.items.filter((_, i) => i !== index),
    })),

  setLoading: (loading) => set({ loading }),

  setError: (error) => set({ error, loading: false }),

  reset: () => set(initialState),
}));

/* Selectors */
export const selectItemCount = (state: NewFeatureState) => state.items.length;
```

### Step 2: Connect to components

```typescript
// In a component:
import { useNewFeatureStore, selectItemCount } from "@/store/newFeatureStore";

export function NewFeatureComponent() {
  const items = useNewFeatureStore((state) => state.items);
  const addItem = useNewFeatureStore((state) => state.addItem);
  const count = useNewFeatureStore(selectItemCount);

  return (
    <div>
      <p>Items: {count}</p>
      <button onClick={() => addItem("new item")}>Add</button>
      <ul>
        {items.map((item, i) => (
          <li key={i}>{item}</li>
        ))}
      </ul>
    </div>
  );
}
```

### Step 3: Write tests

```typescript
// frontend/src/test/regression/new-feature.test.tsx

import { describe, it, expect, beforeEach } from "vitest";
import { useNewFeatureStore } from "@/store/newFeatureStore";

describe("NewFeature Store", () => {
  beforeEach(() => {
    useNewFeatureStore.setState({ items: [], loading: false, error: null });
  });

  it("adds items", () => {
    useNewFeatureStore.getState().addItem("test");
    expect(useNewFeatureStore.getState().items).toEqual(["test"]);
  });

  it("removes items by index", () => {
    useNewFeatureStore.setState({ items: ["a", "b", "c"] });
    useNewFeatureStore.getState().removeItem(1);
    expect(useNewFeatureStore.getState().items).toEqual(["a", "c"]);
  });

  it("resets to initial state", () => {
    useNewFeatureStore.getState().addItem("test");
    useNewFeatureStore.getState().reset();
    expect(useNewFeatureStore.getState().items).toEqual([]);
  });
});
```

### Verify

```bash
cd frontend && npm test
```

---

## Cross-Reference

| Recipe | Files Modified | Files Created |
|--------|---------------|---------------|
| New Engine | `analysis.py`, `streaming.py`, `main.py` | `engines/e7_*/engine.py`, `engines/e7_*/__init__.py`, `tests/unit/engines/test_e7.py` |
| New API Route | `main.py` | `routes/new_feature.py`, `schemas/new_feature.py`, `handlers.ts` |
| New Screen | `router.tsx`, Layout component | `screens/NewScreen.tsx`, test in `no-crash.test.tsx` |
| New Surface | `router.tsx` | Layout, screens, tests |
| New Agent Tool | `tools.py` | Tests in `test_agent.py` |
| New Agent Persona | `prompts.py` | Frontend context wiring |
| New Simulation Actor | `coordinator.py` | `actors/new_actor.py` |
| New Database Table | `alembic/versions/` (generated) | `models/new_table.py`, `ingestion/seed_*.py`, Makefile target |
| New Jurisdiction | `engine.py` (E2 dispatch map) | `regimes/jp.py`, `ingestion/seed_jp_tariff.py` |
| New Compliance List | `engine.py` (E3) | `models/compliance.py` (extend), `ingestion/seed_*.py` |
| Pipeline Order | `analysis.py`, `streaming.py` | -- |
| New Zustand Store | Component files | `store/newFeatureStore.ts`, tests |
