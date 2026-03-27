# API Endpoint Catalog -- Implementation Details

Implementation-focused reference for the Clearance Intelligence Engine API. Covers exact request/response shapes as defined in the Pydantic schemas, error cases, SSE event schemas, and working examples.

Source files: `backend/app/api/routes/`, `backend/app/api/schemas/`, `backend/app/api/streaming.py`

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Request/Response Lifecycle](#requestresponse-lifecycle)
3. [SSE Streaming Infrastructure](#sse-streaming-infrastructure)
4. [Engine Endpoints (E1-E7)](#engine-endpoints-e1-e7)
5. [Entity Endpoints (Shipments, Orders, Products)](#entity-endpoints)
6. [Operational Endpoints (Chat, Simulation, Dashboard)](#operational-endpoints)
7. [Error Handling Patterns](#error-handling-patterns)
8. [Examples](#examples)

---

## Architecture Overview

The API is built on FastAPI with 17 routers registered in `app/main.py:create_app()`. All routers are imported and mounted at startup:

```
analysis_router      -> /api/analyze/*
classify_router      -> /api/classify/*
tariff_router        -> /api/tariff
compliance_router    -> /api/compliance
fta_router           -> /api/fta
exception_router     -> /api/exception/*
regulatory_router    -> /api/regulatory/*
screening_router     -> /api/screen
dashboard_router     -> /api/dashboard/*
trade_lanes_router   -> /api/trade-lanes
session_router       -> /api/session/*
products_router      -> /api/products/*
orders_router        -> /api/orders/*
shipments_router     -> /api/shipments/*
documents_router     -> /api/documents/*
chat_router          -> /api/chat/*
resolution_router    -> /api/resolution/*
simulation_router    -> /api/simulation/*
```

Infrastructure is available on `request.app.state`:
- `db_session_factory` -- SQLAlchemy async session factory
- `redis` -- aioredis client
- `qdrant` -- AsyncQdrantClient
- `simulation` -- SimulationCoordinator instance

---

## Request/Response Lifecycle

Every HTTP request passes through:

1. **CORS middleware** -- Permissive (`allow_origins=["*"]`)
2. **Timing middleware** -- Sets `X-Processing-Time-Ms` header
3. **Route handler** -- Pydantic validation of request body/query params
4. **Engine invocation** -- Lazy-imports engine modules at call time
5. **Response serialisation** -- Pydantic model or SSE stream

The lazy-import pattern is used throughout:
```python
try:
    from app.engines.e1_classification import classify
    result = await classify(...)
except ImportError:
    # Engine not yet implemented -- return stub
    result = ClassificationResult(hs_code="0000.00.0000", ...)
except Exception as exc:
    # Engine failed -- raise HTTP 500 or emit engine_error SSE
    raise HTTPException(status_code=500, detail=str(exc))
```

This pattern allows engines to be developed independently. The API layer never crashes if an engine module is missing.

---

## SSE Streaming Infrastructure

**Source:** `backend/app/api/streaming.py`

### Event Type Constants

```python
# E1 - Classification
CLASSIFICATION_STEP = "classification_step"
CLASSIFICATION_COMPLETE = "classification_complete"

# E2 - Tariff
TARIFF_PROGRAM = "tariff_program"
TARIFF_COMPLETE = "tariff_complete"

# E3 - Compliance
COMPLIANCE_PGA = "compliance_pga"
COMPLIANCE_DPS = "compliance_dps"
COMPLIANCE_UFLPA = "compliance_uflpa"

# Full pipeline
ANALYSIS_COMPLETE = "analysis_complete"

# E5 - Exception (defined in routes/exception.py)
EXCEPTION_PROGRAM = "exception_program"
EXCEPTION_COMPLETE = "exception_complete"

# Generic
ENGINE_ERROR = "engine_error"
HEARTBEAT = "heartbeat"
```

### Frame Format

```python
def sse_event(event_type: str, data: dict) -> str:
    payload = json.dumps(data, default=str)
    return f"event: {event_type}\ndata: {payload}\n\n"
```

### Response Wrapper

```python
def sse_response(generator: AsyncIterator[str]) -> StreamingResponse:
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

### Chat SSE (Different Protocol)

The chat endpoint uses a simplified data-only protocol (no `event:` field):

```
data: {"text": "chunk of response"}
data: {"tool": "lookup_shipment", "status": "running", "label": "Looking up shipment abc-123"}
data: {"tool": "lookup_shipment", "status": "complete"}
data: {"error": "No LLM provider configured"}
data: [DONE]
```

---

## Engine Endpoints (E1-E7)

### POST /api/analyze/stream -- Full Pipeline

**Request Schema:** `AnalysisRequest`

| Field | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `product_description` | string | yes | -- | 3-5000 chars |
| `origin_country` | string | no | `"CN"` | 2-3 chars, ISO |
| `destination_country` | string | no | `"US"` | 2-3 chars, ISO |
| `declared_value` | float | yes | -- | > 0 |
| `currency` | string | no | `"USD"` | 3 chars, ISO 4217 |
| `entity_name` | string | no | null | max 500 chars |

**Pipeline sequence:** E1 (classify) -> E2 (tariff, uses E1 hs_code) -> E3 (compliance, uses E1 hs_code) -> E4 (FTA, uses E1 hs_code) -> assemble final result.

Each engine failure is non-fatal: an `engine_error` event is emitted and a stub result is used for downstream engines.

### POST /api/classify/stream -- Standalone Classification

**Request Schema:** `ClassifyRequest`

| Field | Type | Required | Default |
|---|---|---|---|
| `product_description` | string | yes | -- |
| `origin_country` | string | no | `"CN"` |
| `destination_country` | string | no | `"US"` |

Individual reasoning steps are streamed as `classification_step` events before the final `classification_complete`.

### POST /api/tariff -- Tariff Calculation

**Request Schema:** `TariffRequest`

| Field | Type | Required | Default |
|---|---|---|---|
| `hs_code` | string | yes | -- |
| `origin_country` | string | yes | -- |
| `destination_country` | string | no | `"US"` |
| `declared_value` | float | yes | -- |
| `currency` | string | no | `"USD"` |

Returns synchronous JSON. Raises HTTP 500 on engine failure.

### POST /api/compliance -- Compliance Screening

**Request Schema:** `ComplianceRequest`

| Field | Type | Required | Default |
|---|---|---|---|
| `hs_code` | string | yes | -- |
| `origin_country` | string | yes | -- |
| `destination_country` | string | no | `"US"` |
| `entity_name` | string | no | null |
| `product_description` | string | no | null |

### POST /api/fta -- FTA Eligibility

**Request Schema:** `FTARequest`

| Field | Type | Required | Default |
|---|---|---|---|
| `hs_code` | string | yes | -- |
| `origin_country` | string | yes | -- |
| `destination_country` | string | no | `"US"` |
| `declared_value` | float | yes | -- |
| `currency` | string | no | `"USD"` |

### POST /api/exception/stream -- Exception Programmes

**Request Schema:** `ExceptionRequest`

| Field | Type | Required | Default |
|---|---|---|---|
| `hs_code` | string | yes | -- |
| `origin_country` | string | yes | -- |
| `destination_country` | string | no | `"US"` |
| `declared_value` | float | yes | -- |
| `entry_type` | string | no | `"01"` |
| `product_description` | string | no | `""` |

### Regulatory (E6)

Three endpoints on `/api/regulatory`:

- `GET /api/regulatory` -- List with query filters: `status`, `impact_level`, `hs_prefix`, `limit`, `offset`
- `POST /api/regulatory` -- Create signal (returns 201)
- `POST /api/regulatory/scenario` -- Model what-if scenario

### Documents (E7)

Five endpoints on `/api/documents`:

- `POST /api/documents/requirements` -- Deterministic doc requirements lookup
- `POST /api/documents/generate` -- LLM auto-fill
- `POST /api/documents/analyze` -- LLM validation
- `POST /api/documents/orders/{orderId}/upload` -- Attach doc to order
- `GET /api/documents/orders/{orderId}/list` -- List order docs
- `POST /api/documents/shipments/{shipmentId}/upload` -- Attach doc to shipment
- `GET /api/documents/shipments/{shipmentId}/list` -- List shipment docs

Document field schemas are defined inline in `routes/documents.py` for each document type (Commercial Invoice, Packing List, Bill of Lading, Certificate of Origin, ISF, etc.).

---

## Entity Endpoints

### Shipments (`/api/shipments`)

| Method | Path | Description |
|---|---|---|
| GET | `/api/shipments` | List/filter (status, origin, destination, company) |
| GET | `/api/shipments/{id}` | Full detail |
| POST | `/api/shipments` | Create |
| PUT | `/api/shipments/{id}` | Update |
| POST | `/api/shipments/{id}/events` | Add timeline event |
| POST | `/api/shipments/{id}/analyze` | Run analysis |

Auto-seeds 18 demo shipments on first access covering all 7 lifecycle states.

### Orders (`/api/orders`)

| Method | Path | Description |
|---|---|---|
| GET | `/api/orders` | List all |
| GET | `/api/orders/{id}` | Detail with line items |
| POST | `/api/orders` | Create |
| PUT | `/api/orders/{id}` | Update |
| POST | `/api/orders/{id}/line-items` | Add line item |
| POST | `/api/orders/{id}/analyze` | Run E1-E3 pipeline |
| POST | `/api/orders/{id}/ship` | Convert to Shipment |

Auto-seeds 3 demo orders (draft, analysed, ready-to-ship) on first access.

### Products (`/api/products`)

| Method | Path | Description |
|---|---|---|
| GET | `/api/products` | List catalog |
| POST | `/api/products` | Create |
| GET | `/api/products/{id}` | Detail |
| PUT | `/api/products/{id}` | Update |

Auto-seeds 13 demo products spanning electronics, textiles, chemicals, automotive, and consumer goods.

---

## Operational Endpoints

### Chat (`/api/chat/stream`)

See the [Agent System documentation](../../doc/technical/backend-agent-system.md) for full details.

### Simulation (`/api/simulation/*`)

| Method | Path | Description |
|---|---|---|
| POST | `/start` | Start all actors |
| POST | `/stop` | Stop all actors |
| POST | `/pause` | Pause clock |
| POST | `/resume` | Resume clock |
| POST | `/reset` | Stop, clear, reseed, reset clock |
| GET | `/status` | Full status with clock and actor states |
| PUT | `/clock` | Adjust `time_ratio` (0.1-1000.0) |
| GET | `/actors` | List all actor configs |
| PUT | `/actors/{actor_id}` | Hot-update actor config |
| GET | `/dashboard/stream` | SSE dashboard updates |
| GET | `/dashboard/latest` | Polling fallback |

### Dashboard (`/api/dashboard/{company_id}`)

Aggregated metrics with configurable lookback period (`period_days` query param, default 30).

### Session (`/api/session/*`)

| Method | Path | Description |
|---|---|---|
| POST | `/create` | Create session (Redis + DB) |
| GET | `/{session_id}/history` | Analysis history with pagination |

### Resolution (`/api/resolution/*`)

| Method | Path | Description |
|---|---|---|
| POST | `/lookup` | Classify + compliance check |
| POST | `/upload` | PDF extraction + field analysis |
| POST | `/suggest` | AI resolution suggestions |

### Trade Lanes (`/api/trade-lanes`)

Multi-destination comparison with parallel execution. If `hs_code` is not provided, E1 classification runs first. Destinations sorted by landed cost ascending.

---

## Error Handling Patterns

### Validation Errors (422)

Automatically handled by FastAPI/Pydantic. Returns:
```json
{
  "detail": [
    {
      "loc": ["body", "declared_value"],
      "msg": "ensure this value is greater than 0",
      "type": "value_error.number.not_gt"
    }
  ]
}
```

### Engine Errors in Streaming Endpoints

Non-fatal. An `engine_error` SSE event is emitted and the pipeline continues:
```
event: engine_error
data: {"engine": "e1_classification", "error": "Connection timeout"}
```

### Engine Errors in Synchronous Endpoints

Raised as HTTP 500:
```json
{
  "detail": "Tariff calculation failed: Connection timeout"
}
```

### Not Found (404)

Returned for missing sessions, shipments, orders, products, actors:
```json
{
  "detail": "Session abc-123 not found"
}
```

### Service Unavailable (503)

Returned when the simulation coordinator is not initialised:
```json
{
  "detail": "Simulation coordinator not initialized"
}
```

---

## Examples

### Full Pipeline Analysis

```bash
curl -N -X POST http://localhost:8000/api/analyze/stream \
  -H "Content-Type: application/json" \
  -d '{
    "product_description": "Lithium-ion battery pack 75kWh for electric vehicles",
    "origin_country": "CN",
    "destination_country": "US",
    "declared_value": 8500.00,
    "entity_name": "Shenzhen Power Co."
  }'
```

### Standalone Classification

```bash
curl -N -X POST http://localhost:8000/api/classify/stream \
  -H "Content-Type: application/json" \
  -d '{
    "product_description": "Cotton knit crew-neck pullover sweater, adult sizing"
  }'
```

### Tariff Calculation

```bash
curl -X POST http://localhost:8000/api/tariff \
  -H "Content-Type: application/json" \
  -d '{
    "hs_code": "6110.20.2075",
    "origin_country": "CN",
    "declared_value": 24500.00
  }'
```

### Entity Screening

```bash
curl -X POST http://localhost:8000/api/screen \
  -H "Content-Type: application/json" \
  -d '{
    "entity_name": "Huawei Technologies",
    "threshold": 75.0
  }'
```

### Chat with Shipment Context

```bash
curl -N -X POST http://localhost:8000/api/chat/stream \
  -H "Content-Type: application/json" \
  -d '{
    "message": "What is the hold reason for this shipment and how can we resolve it?",
    "context": {
      "screen": "exception-resolution",
      "shipment_id": "abc-123-def-456"
    }
  }'
```

### Trade Lane Comparison

```bash
curl -X POST http://localhost:8000/api/trade-lanes \
  -H "Content-Type: application/json" \
  -d '{
    "product_description": "CNC aluminum bracket 7075-T6",
    "origin_country": "DE",
    "destinations": ["US", "BR", "IN"],
    "declared_value": 145000.00
  }'
```

### Simulation Control

```bash
# Start
curl -X POST http://localhost:8000/api/simulation/start

# Adjust speed
curl -X PUT http://localhost:8000/api/simulation/clock \
  -H "Content-Type: application/json" \
  -d '{"time_ratio": 20.0}'

# Stream dashboard
curl -N http://localhost:8000/api/simulation/dashboard/stream

# Reset
curl -X POST http://localhost:8000/api/simulation/reset
```
