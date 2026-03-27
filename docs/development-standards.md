# Development Standards

> **Author:** Simon Kissler (simon.kissler@accenture.com) | **Co-Author:** Claude Code (Anthropic)

This document establishes mandatory patterns and practices to ensure code quality and prevent common errors.

## API URL Standards

### The One Rule

**All frontend API calls MUST use the centralized `VITE_API_URL` pattern with `/api` fallback.**

```typescript
// CORRECT - The only acceptable pattern
const API_BASE = import.meta.env.VITE_API_URL || "/api";
fetch(`${API_BASE}/broker/dashboard`);

// WRONG - Do not use undefined environment variables
const API_BASE = import.meta.env.VITE_API_BASE ?? "";  // VITE_API_BASE doesn't exist
fetch(`${API_BASE}/api/broker/dashboard`);  // Double /api/ if VITE_API_URL is set
```

### How It Works

| Environment | VITE_API_URL value | Resulting URL |
|-------------|-------------------|---------------|
| Vite dev server | (undefined) | `/api/broker/dashboard` → proxied to backend |
| Docker compose | `http://localhost:4000/api` | `http://localhost:4000/api/broker/dashboard` |
| Production | `/api` | `/api/broker/dashboard` → Caddy proxies to backend |

### Adding New Environment Variables

1. **Document in `vite-env.d.ts`** - Add TypeScript type
2. **Add to docker-compose.yml** - Set value for container environment
3. **Add to Dockerfile.prod** - Set build-time default
4. **Document in this file** - Explain purpose

### Current Environment Variables

| Variable | Purpose | Default | Required |
|----------|---------|---------|----------|
| `VITE_API_URL` | Backend API base URL | `/api` | No |

---

## Frontend/Backend Contract

### Response Format Alignment

When adding new endpoints:

1. **Define the backend schema first** using Pydantic models
2. **Match frontend types exactly** - use code generation if possible
3. **Transform snake_case → camelCase at the boundary** (in API client, not in components)
4. **Test the contract** with integration tests

### snake_case ↔ camelCase

- Backend: snake_case (Python convention)
- Frontend: camelCase (JavaScript convention)
- Transform in `src/api/client.ts`, not in individual components

```typescript
// CORRECT - Transformation in client.ts
function transformBrokerEntry(raw: Record<string, unknown>): BrokerEntry {
  return {
    entryNumber: raw.entry_number as string,
    filingStatus: raw.filing_status as string,
    // ...
  };
}

// WRONG - Transformation scattered in components
const data = await res.json();
setEntry({
  entryNumber: data.entry_number,  // Don't do this in components
  // ...
});
```

---

## Testing Requirements

### ZERO TOLERANCE FOR FAILING TESTS

**ALL tests must pass. No exceptions. No "unrelated" failures.**

- If you run tests and something fails, you fix it — even if you didn't cause it
- Never dismiss a failure as "pre-existing" or "unrelated"
- Never merge with failing tests
- If a test is flaky, fix the test to be deterministic or remove it
- Tests that depend on LLM output must accept reasonable variation (e.g., HIGH or MEDIUM confidence)

This is non-negotiable. We do not ship broken code.

### Before Merging

1. **TypeScript compiles**: `npm run build` (frontend)
2. **ALL tests pass**: `pytest` (backend), `npm run test` (frontend)
3. **API contracts match**: Integration tests verify frontend/backend alignment

### Integration Test Pattern

```typescript
// frontend/src/test/integration/api-contracts.test.ts
describe("Broker API Contract", () => {
  it("GET /broker/dashboard returns expected shape", async () => {
    const res = await fetch(`${API_URL}/broker/dashboard`);
    const data = await res.json();

    // Verify shape matches what frontend expects
    expect(data).toHaveProperty("stats");
    expect(data.stats).toHaveProperty("draft");
    // ...
  });
});
```

### What to Test

| Layer | Tests | Command |
|-------|-------|---------|
| Backend unit | Business logic, models | `pytest tests/unit` |
| Backend integration | API endpoints, DB | `pytest tests/integration` |
| Frontend unit | Components, hooks | `npm run test` |
| Frontend integration | API contracts | `npm run test:integration` |
| E2E | Full user flows | `npm run test:e2e` |

---

## Code Review Checklist

### For Every PR

- [ ] No hardcoded URLs (localhost, port numbers)
- [ ] Environment variables documented if new
- [ ] API client uses correct pattern (`VITE_API_URL || "/api"`)
- [ ] Response transformations in client.ts, not components
- [ ] TypeScript compiles without errors
- [ ] Relevant tests added/updated

### For API Changes

- [ ] Backend schema defined with Pydantic
- [ ] Frontend types match backend schema
- [ ] Transformation function added to client.ts
- [ ] Integration test verifies contract
- [ ] API documentation updated

---

## Common Mistakes to Avoid

### 1. Creating New Environment Variable Names

**Wrong:**
```typescript
const API_BASE = import.meta.env.VITE_API_BASE ?? "";  // Invented variable
```

**Right:**
```typescript
const API_BASE = import.meta.env.VITE_API_URL || "/api";  // Existing standard
```

### 2. Hardcoding URLs

**Wrong:**
```typescript
fetch("http://localhost:4000/api/broker/dashboard");
```

**Right:**
```typescript
fetch(`${API_BASE}/broker/dashboard`);
```

### 3. Duplicating /api/ Prefix

**Wrong:**
```typescript
const API_BASE = import.meta.env.VITE_API_URL || "";  // Empty fallback
fetch(`${API_BASE}/api/broker/dashboard`);  // Assumes no /api in base
```

**Right:**
```typescript
const API_BASE = import.meta.env.VITE_API_URL || "/api";  // Includes /api
fetch(`${API_BASE}/broker/dashboard`);  // Path without /api prefix
```

### 4. Different Patterns in Different Files

All files should use the same pattern. If you're adding API calls to a new file, copy the pattern from `src/api/client.ts`, not from some random component.

---

## v5.2 Domain Architecture

### Domain Package Structure

Each domain lives under `clearance_platform/domains/{domain_name}/` with:

```
{domain_name}/
  __init__.py       # Package marker
  events.py         # Domain event definitions (Pydantic models extending BaseEvent)
  service.py        # Domain service with event emission methods
  projector.py      # CQRS read-model projector (optional, for domains needing cache)
  routes.py         # API routes (currently empty placeholders)
```

15 domains: `product_catalog`, `trade_intelligence`, `compliance`, `shipment_lifecycle`, `cargo_handling_units`, `consolidation`, `declaration_management`, `customs_adjudication`, `exception_management`, `financial_settlement`, `regulatory_intelligence`, `document_management`, `party_management`, `order_management`, `supply_chain_disruptions`

### Event Emission Pattern

Domain services emit events after successful operations, wrapped in `try/except` to prevent event failures from breaking the primary operation:

```python
async def emit_shipment_created(self, shipment_id: UUID, ...) -> None:
    try:
        event = ShipmentCreatedEvent(
            metadata=EventMetadata(
                event_type="shipment.created",
                subject="clearance.shipment.created",
                domain="shipment_lifecycle",
            ),
            shipment_id=shipment_id, ...
        )
        await self._bus.publish(event)
    except Exception as exc:
        logger.warning("event_failed", extra={"error": str(exc)})
```

### Domain Event Naming

Events use NATS-style dot-delimited subjects:

- Pattern: `clearance.{domain-short-name}.{event-type}`
- Examples: `clearance.shipment.created`, `clearance.product.updated`
- Wildcard subscription: `clearance.shipment.>` matches all shipment events

### Simulation Event Namespace

Simulation events use the `sim.*` prefix and are bridged to domain events via `SimDomainBridge`:

- Sim events: `sim.shipment_created`, `sim.entry_submitted`, etc.
- The bridge subscribes to sim events and calls domain service `emit_*` methods

### CQRS Projector Pattern

Projectors subscribe to domain events and maintain Redis read-model caches. Registered via `ProjectorRegistry` in `app/main.py`.

### Import Conventions

- Domain code: `from clearance_platform.domains.{domain}.{module} import ...`
- Shared infrastructure: `from clearance_platform.shared.event_bus import DomainEventBus`
- Legacy routes (still serving traffic): `from app.api.routes.{module} import ...`

---

## Quick Reference

### Adding a New API Endpoint

1. Backend: Add route in `backend/app/api/routes/`
2. Backend: Add Pydantic models for request/response
3. Backend: Add tests in `backend/tests/`
4. Frontend: Add function in `src/api/client.ts`
5. Frontend: Add types in `src/types/`
6. Frontend: Add tests in `src/test/`

### Adding a New Environment Variable

1. Add type to `frontend/src/vite-env.d.ts`
2. Add to `clearance-engine/docker-compose.yml`
3. Add to `clearance-engine/frontend/Dockerfile.prod`
4. Document in this file
5. Update `clearance-engine/.env.example`
