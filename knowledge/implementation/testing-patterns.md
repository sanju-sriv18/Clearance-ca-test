# Testing Patterns

This document covers the testing architecture, patterns, and conventions used across the Clearance Intelligence Engine. It serves as both a reference for running existing tests and a guide for writing new ones.

---

## Overview

The project uses a layered testing strategy:

| Layer | Backend Tool | Frontend Tool | Scope |
|-------|-------------|--------------|-------|
| Unit | pytest | Vitest + Testing Library | Individual functions, classes, stores |
| Regression | -- | Vitest + MSW | Screen rendering, store behavior, connected flows |
| Integration | pytest | Vitest (api-live) | Cross-module interactions, live API calls |
| Data Validation | pytest | -- | Reference data correctness |
| End-to-End | pytest | Playwright | Full user workflows through the UI |

---

## Backend Tests (pytest)

### Directory Structure

```
backend/tests/
  __init__.py
  unit/
    __init__.py
    engines/
      __init__.py
      test_e1_classification.py     # Validator, confidence, keywords, LLM, streaming
    test_classification.py
    test_tariff.py
    test_e3_compliance.py
    test_agent.py
  integration/
    __init__.py
    test_pipeline.py                # Full E1-E6 pipeline execution
  e2e/
    __init__.py
    test_golden_path.py             # HTTP-level golden path through the API
  data_validation/
    __init__.py
    test_rate_correctness.py        # Statutory rate verification
```

### Configuration

In `pyproject.toml`:

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

The `asyncio_mode = "auto"` setting means all `async def test_*` functions are automatically treated as async tests without requiring `@pytest.mark.asyncio`, though the project uses it explicitly for clarity.

### Make Targets

```bash
make test              # pytest tests/ -v
make test-unit         # pytest tests/unit/ -v
make test-integration  # pytest tests/integration/ -v
make test-data         # pytest tests/data_validation/ -v
make test-e2e          # pytest tests/e2e/ -v
```

### Unit Test Patterns

#### Engine Tests

Every engine test file follows this structure:

1. **Validator/utility tests** -- pure functions, no mocks needed.
2. **Deterministic behavior tests** -- engine with `llm_service=None` exercises keyword fallback.
3. **Mocked LLM tests** -- `AsyncMock` simulates LLM responses.
4. **Error path tests** -- LLM failures, malformed responses, invalid inputs.
5. **Streaming tests** -- async iterator output verification.
6. **Metadata tests** -- `engine_id`, `engine_name`, timing.

Example pattern from `test_e1_classification.py`:

```python
class TestHSCodeValidator:
    """Pure function tests -- no async, no mocks."""

    def setup_method(self) -> None:
        self.validator = HSCodeValidator()

    def test_valid_dotted_format(self) -> None:
        assert self.validator.is_valid_format("6912.00.48")

    def test_invalid_format_short(self) -> None:
        assert not self.validator.is_valid_format("691")
```

```python
class TestKeywordFallback:
    """Deterministic tests with no LLM."""

    def setup_method(self) -> None:
        self.engine = ClassificationEngine(llm_service=None)

    @pytest.mark.asyncio
    async def test_ceramic_mug_keyword(self) -> None:
        output = await self.engine.execute(
            product_description="Handmade ceramic coffee mug, blue glaze"
        )
        assert output.status == "success"
        assert output.data["chapter"] == "69"
        assert output.data["hs_code"].startswith("6912")
```

```python
class TestLLMClassification:
    """Mocked LLM tests."""

    @staticmethod
    def _mock_llm(chapter_response: dict, classify_response: dict) -> AsyncMock:
        mock = AsyncMock()
        mock.complete = AsyncMock(
            side_effect=[
                json.dumps(chapter_response),
                json.dumps(classify_response),
            ]
        )
        return mock

    @pytest.mark.asyncio
    async def test_llm_chapter_and_classify(self) -> None:
        chapter = {"chapter": "69", "chapter_title": "Ceramic products", ...}
        classify = {"hs_code": "6912.00.48", "confidence": "HIGH", ...}
        llm = self._mock_llm(chapter, classify)
        engine = ClassificationEngine(llm_service=llm)

        output = await engine.execute(product_description="ceramic coffee mug")

        assert output.status == "success"
        assert output.data["hs_code"] == "6912.00.48"
        assert llm.complete.call_count == 2
```

#### Testing EngineOutput

All engines return `EngineOutput`. Tests should verify:

```python
assert output.engine_id == "E2"
assert output.status in ("success", "error", "partial")
assert output.processing_time_ms > 0
assert isinstance(output.data, dict)
assert isinstance(output.errors, list)
assert isinstance(output.warnings, list)
```

#### Testing Error Paths

Engines should not raise exceptions. They catch errors and return them in the `EngineOutput`:

```python
@pytest.mark.asyncio
async def test_llm_exception_returns_error(self) -> None:
    llm = AsyncMock()
    llm.complete = AsyncMock(side_effect=RuntimeError("API down"))
    engine = ClassificationEngine(llm_service=llm)

    output = await engine.execute(product_description="ceramic mug")

    assert output.status == "error"
    assert len(output.errors) > 0
    assert "API down" in output.errors[0]
```

#### Testing Streaming

Streaming methods yield dictionaries with `type` and `data` keys:

```python
@pytest.mark.asyncio
async def test_stream_yields_steps_and_complete(self) -> None:
    engine = ClassificationEngine(llm_service=None)
    events: list[dict[str, Any]] = []

    async for event in engine.stream_classify(
        product_description="Handmade ceramic coffee mug"
    ):
        events.append(event)

    assert len(events) >= 4
    types = [e["type"] for e in events]
    assert "classification_step" in types
    assert "classification_complete" in types

    complete = events[-1]
    assert complete["type"] == "classification_complete"
    assert "hs_code" in complete["data"]
```

### Integration Test Patterns

Integration tests exercise cross-engine interactions. They typically require a database session:

```python
# tests/integration/test_pipeline.py
@pytest.mark.asyncio
async def test_full_pipeline_execution():
    """Run the complete E1-E6 pipeline and verify output structure."""
    # Uses keyword fallback (no LLM) for deterministic results
    ...
```

### Data Validation Tests

These verify that seeded reference data matches statutory rates:

```python
# tests/data_validation/test_rate_correctness.py
def test_section_301_rate_for_list_4a():
    """Section 301 List 4A should be 7.5% per USTR-2020-0014."""
    ...
```

---

## Frontend Tests

### Directory Structure

```
frontend/
  src/test/
    setup.ts                          # Vitest global setup
    mocks/
      handlers.ts                     # MSW request handlers
      server.ts                       # MSW server instance
    integration/
      api-live.test.ts                # Live API integration tests
    regression/
      no-crash.test.tsx               # Screen render smoke tests + store tests
      connected-flows.test.tsx        # Multi-component interaction tests
      agent-tools.test.tsx            # Agent tool-use rendering tests
  e2e/
    playwright.config.ts              # Playwright configuration
    helpers/
      sse-mock.ts                     # SSE mock utilities for E2E
      store-bridge.ts                 # Store access from Playwright
    platform-workflow.spec.ts         # Platform surface E2E
    shipper-workflow.spec.ts          # Shipper surface E2E
    buyer-workflow.spec.ts            # Buyer surface E2E
    cross-surface.spec.ts             # Cross-surface navigation E2E
    agent-assistant.spec.ts           # Agent assistant E2E
```

### Test Setup (Vitest + MSW)

The test setup file at `src/test/setup.ts` configures three things:

1. **Testing Library matchers**: Imports `@testing-library/jest-dom` for DOM matchers.
2. **jsdom polyfills**: Patches `window.scrollTo` and `ResizeObserver` for framer-motion compatibility.
3. **MSW lifecycle**: Starts the mock server before all tests, resets handlers after each test, closes after all tests.

```typescript
// src/test/setup.ts
import "@testing-library/jest-dom";
import { cleanup } from "@testing-library/react";
import { afterEach, beforeAll, afterAll } from "vitest";
import { server } from "./mocks/server";

if (typeof window !== "undefined") {
  window.scrollTo = () => {};
  if (!window.ResizeObserver) {
    window.ResizeObserver = class ResizeObserver {
      observe() {}
      unobserve() {}
      disconnect() {}
    } as unknown as typeof ResizeObserver;
  }
}

beforeAll(() => server.listen({ onUnhandledRequest: "warn" }));
afterEach(() => {
  cleanup();
  server.resetHandlers();
});
afterAll(() => server.close());
```

### MSW Server

```typescript
// src/test/mocks/server.ts
import { setupServer } from "msw/node";
import { handlers } from "./handlers";

export const server = setupServer(...handlers);
```

### MSW Request Handlers

The handlers mock all API endpoints the frontend calls:

| Method | Path | Response |
|--------|------|----------|
| `POST` | `/api/analyze/stream` | SSE stream with classification, tariff, and analysis events |
| `GET` | `/api/dashboard/demo` | Dashboard statistics JSON |
| `GET` | `/api/regulatory` | Empty regulatory signals array |
| `POST` | `/api/screen` | Empty screening results (clear) |
| `POST` | `/api/tariff` | Minimal tariff calculation result |
| `POST` | `/api/compliance` | Minimal compliance result (low risk) |
| `POST` | `/api/trade-lanes` | Minimal trade lane comparison |

Example handler for SSE streaming:

```typescript
http.post("/api/analyze/stream", () => {
  const body = [
    'event: classification_step\ndata: {"step":"reasoning","index":0,"content":"Analyzing product"}\n\n',
    'event: classification_complete\ndata: {"hs_code":"8507.60","hs_description":"Test","confidence":"HIGH",...}\n\n',
    'event: tariff_complete\ndata: {"hs_code":"8507.60","declared_value":1000,...}\n\n',
    'event: analysis_complete\ndata: {"compliance":{...}}\n\n',
    "data: [DONE]\n\n",
  ].join("");

  return new HttpResponse(body, {
    headers: { "Content-Type": "text/event-stream" },
  });
});
```

### Regression Test Patterns

#### No-Crash Screen Rendering

Every screen must render without throwing:

```typescript
function wrap(element: React.ReactElement, route = "/") {
  return render(
    <MemoryRouter initialEntries={[route]}>{element}</MemoryRouter>,
  );
}

describe("Screen Render - No Crash", () => {
  it("ControlTower renders", async () => {
    const { ControlTower } = await import(
      "@/surfaces/platform/screens/ControlTower"
    );
    const { container } = wrap(<ControlTower />);
    expect(container).toBeTruthy();
  });
});
```

Key pattern: Dynamic imports (`await import(...)`) are used so that test failures in one screen do not prevent other screen tests from running.

#### Store Tests

Zustand stores are tested without DOM rendering:

```typescript
describe("Pipeline Store", () => {
  beforeEach(() => {
    usePipelineStore.setState({
      entries: usePipelineStore.getState().entries.filter((e) => e.source === "demo"),
      nextId: 49,
    });
  });

  it("createEntry adds a new entry and increments aggregates", () => {
    const store = usePipelineStore.getState();
    const id = store.createEntry({
      product: "Test product",
      origin: "CN",
      destination: "US",
      declaredValue: 1000,
      currency: "USD",
      status: "RECEIVED",
      source: "platform",
    });

    const updated = usePipelineStore.getState();
    expect(id).toMatch(/^CLR-2025-/);
    expect(updated.entries.find((e) => e.id === id)).toBeTruthy();
  });
});
```

Pattern: Use `usePipelineStore.setState(...)` in `beforeEach` to reset state. Use `usePipelineStore.getState()` to access state and actions without needing a React component.

#### Product Catalog Validation

```typescript
describe("Product Catalog", () => {
  it("each product has required fields", () => {
    for (const p of CATALOG_PRODUCTS) {
      expect(p.id).toBeTruthy();
      expect(p.name).toBeTruthy();
      expect(p.hsCode).toBeTruthy();
      expect(p.readyLight).toMatch(/^(green|yellow|red)$/);
      expect(p.buyerPrice).toBeGreaterThan(0);
    }
  });
});
```

### E2E Test Patterns (Playwright)

Playwright tests run against the live application (or a build preview).

Configuration is at `e2e/playwright.config.ts`. Test files:

| File | Scope |
|------|-------|
| `platform-workflow.spec.ts` | Platform surface: dashboard, entries, analysis |
| `shipper-workflow.spec.ts` | Shipper surface: products, orders, shipments |
| `buyer-workflow.spec.ts` | Buyer surface: storefront, cart, checkout |
| `cross-surface.spec.ts` | Navigation between surfaces |
| `agent-assistant.spec.ts` | Agent chat interactions |

Run commands:

```bash
cd frontend
npm run test:e2e              # Headless run
npm run test:e2e:ui           # Interactive Playwright UI
npm run test:e2e:debug        # Step-through debugging
```

E2E helpers:

- `helpers/sse-mock.ts` -- Utilities for mocking SSE streams in Playwright.
- `helpers/store-bridge.ts` -- Bridge to access Zustand store state from test context.

---

## How to Add New Tests

### Adding a Backend Unit Test

1. Create or locate the test file in `backend/tests/unit/`.
2. Follow the class-based grouping pattern:

```python
# backend/tests/unit/engines/test_e7_new_engine.py

from __future__ import annotations
from unittest.mock import AsyncMock
import pytest
from app.engines.e7_new.engine import NewEngine

class TestNewEngine:
    def setup_method(self) -> None:
        self.engine = NewEngine()

    @pytest.mark.asyncio
    async def test_basic_execution(self) -> None:
        output = await self.engine.execute(param="value")
        assert output.status == "success"
        assert output.engine_id == "E7"
        assert output.processing_time_ms > 0

    @pytest.mark.asyncio
    async def test_error_handling(self) -> None:
        output = await self.engine.execute(param="invalid")
        assert output.status == "error"
        assert len(output.errors) > 0
```

3. Run: `make test-unit`

### Adding a Frontend Regression Test

1. Create the test file in `frontend/src/test/regression/`.
2. Import Vitest utilities and wrap components with `MemoryRouter`:

```typescript
// src/test/regression/new-feature.test.tsx
import { describe, it, expect } from "vitest";
import { render, screen } from "@testing-library/react";
import { MemoryRouter } from "react-router-dom";

describe("NewFeature", () => {
  it("renders without crashing", async () => {
    const { NewFeature } = await import("@/surfaces/platform/screens/NewFeature");
    const { container } = render(
      <MemoryRouter><NewFeature /></MemoryRouter>
    );
    expect(container).toBeTruthy();
  });

  it("displays expected content", async () => {
    const { NewFeature } = await import("@/surfaces/platform/screens/NewFeature");
    render(<MemoryRouter><NewFeature /></MemoryRouter>);
    expect(screen.getByText("Expected Heading")).toBeInTheDocument();
  });
});
```

3. Run: `cd frontend && npm test`

### Adding an MSW Handler

When a new API endpoint is added, add a corresponding mock handler:

```typescript
// src/test/mocks/handlers.ts
http.post("/api/new-endpoint", () => {
  return HttpResponse.json({
    result: "mock-value",
    status: "ok",
  });
}),
```

### Adding an E2E Test

1. Create or extend a spec file in `frontend/e2e/`.
2. Use Playwright's page fixture:

```typescript
// e2e/new-workflow.spec.ts
import { test, expect } from "@playwright/test";

test("new workflow completes successfully", async ({ page }) => {
  await page.goto("/platform/new-feature");
  await expect(page.locator("h1")).toContainText("New Feature");

  await page.fill('[data-testid="input"]', "test value");
  await page.click('[data-testid="submit"]');

  await expect(page.locator('[data-testid="result"]')).toBeVisible();
});
```

3. Run: `cd frontend && npm run test:e2e`

---

## Test Data Conventions

- **Backend unit tests**: Use inline fixtures or `_mock_*` helper methods within the test class. Do not depend on seeded database data.
- **Backend integration tests**: May use seeded data. Run `make seed` before `make test-integration`.
- **Frontend regression tests**: MSW handlers return minimal valid responses. Tests should not depend on specific data values from the mock -- only on structural correctness.
- **Frontend E2E tests**: Run against the live stack. Ensure `make up && make migrate && make seed` has been run.

---

## Continuous Integration Notes

A typical CI pipeline runs:

```bash
# Backend
cd backend
pip install -e ".[dev]"
ruff check .
mypy .
pytest tests/unit/ -v --tb=short
pytest tests/data_validation/ -v --tb=short

# Frontend
cd frontend
npm ci
npm run lint
npm run test
npx playwright install --with-deps
npm run test:e2e
```

Integration and E2E backend tests require running infrastructure services (Postgres, Redis, Qdrant). In CI, use Docker Compose services or equivalent.
