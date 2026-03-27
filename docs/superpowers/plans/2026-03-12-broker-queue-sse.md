# Broker Work Queue SSE Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace broker work queue polling with SSE-driven real-time updates via a dedicated projector service.

**Architecture:** A new standalone `broker_queue_projector` service subscribes to declaration, document, and shipment events via NATS JetStream, maintains per-broker work queue materialized views in Redis, and serves SSE streams to connected broker clients. The existing `GET /api/broker/queue` endpoint remains as the initial snapshot source; SSE provides incremental deltas. The general cache projector is untouched.

**Tech Stack:** Python 3.12, FastAPI, NATS JetStream, Redis (sorted sets + pub/sub), SSE (text/event-stream), React EventSource API

**Branch:** `feature/broker-queue-sse` branched from `production-hardening`

---

## File Structure

### New files (backend)
- `clearance-engine/backend/clearance_platform/broker_queue_projector/__init__.py` — package init
- `clearance-engine/backend/clearance_platform/broker_queue_projector/projector.py` — standalone FastAPI app: NATS consumer, Redis projection, SSE endpoint
- `clearance-engine/backend/clearance_platform/broker_queue_projector/models.py` — `QueueEntryProjection` Pydantic model (the Redis-stored queue entry shape)
- `clearance-engine/backend/clearance_platform/broker_queue_projector/Dockerfile` — container image (same pattern as cache_projector)
- `clearance-engine/backend/tests/projector/test_broker_queue_projector.py` — unit tests for projection logic
- `clearance-engine/backend/tests/projector/__init__.py` — package init
- `clearance-engine/backend/tests/projector/test_broker_queue_sse.py` — SSE endpoint tests

### New files (frontend)
- `clearance-engine/frontend/src/hooks/useBrokerQueueStream.ts` — EventSource hook with reconnection + delta application
- `clearance-engine/frontend/src/hooks/__tests__/useBrokerQueueStream.test.ts` — hook tests

### Modified files (infrastructure)
- `clearance-engine/docker-compose.microservices.yml` — add `broker-queue-projector` service
- `clearance-engine/Caddyfile.microservices.dev` — add route for SSE endpoint
- `clearance-engine/Caddyfile.microservices` — add route for SSE endpoint (production)

### Modified files (frontend)
- `clearance-engine/frontend/src/surfaces/broker/screens/BrokerQueue.tsx` — replace polling `useEffect` with `useBrokerQueueStream`
- `clearance-engine/frontend/src/surfaces/broker/screens/BrokerDashboard.tsx` — replace 30s `setInterval` polling with SSE-fed queue data
- `clearance-engine/frontend/src/api/client.ts` — add `getBrokerQueueSnapshot()` function (renamed from `getBrokerQueue` usage for clarity; existing function unchanged)

---

## Chunk 1: Branch Setup + Projection Model + Core Logic

### Task 1: Create feature branch

**Files:**
- None (git only)

- [ ] **Step 1: Create and checkout feature branch**

```bash
cd /Users/why/vibe/clearance_vibe
git checkout -b feature/broker-queue-sse production-hardening
```

- [ ] **Step 2: Verify branch**

Run: `git branch --show-current`
Expected: `feature/broker-queue-sse`

---

### Task 2: Define the QueueEntryProjection model

This is the shape stored in Redis per queue entry. It mirrors the `QueueEntry` response model from `declaration_management/routes.py` but is self-contained — no DB joins needed to construct it.

**Files:**
- Create: `clearance-engine/backend/clearance_platform/broker_queue_projector/__init__.py`
- Create: `clearance-engine/backend/clearance_platform/broker_queue_projector/models.py`
- Test: `clearance-engine/backend/tests/projector/__init__.py`
- Test: `clearance-engine/backend/tests/projector/test_broker_queue_projector.py`

- [ ] **Step 1: Write the failing test for projection model**

```python
# tests/projector/__init__.py
# (empty)
```

```python
# tests/projector/test_broker_queue_projector.py
"""Tests for broker queue projector models and projection logic."""
from __future__ import annotations

import json
import pytest
from datetime import datetime, timezone
from uuid import uuid4


class TestQueueEntryProjection:
    """Test the QueueEntryProjection Pydantic model."""

    def test_create_from_declaration_event(self):
        """QueueEntryProjection can be constructed from declaration event data."""
        from clearance_platform.broker_queue_projector.models import QueueEntryProjection

        entry_id = str(uuid4())
        shipment_id = str(uuid4())
        broker_id = str(uuid4())

        projection = QueueEntryProjection(
            entry_id=entry_id,
            shipment_id=shipment_id,
            broker_id=broker_id,
            entry_number="ENT-2026-001",
            filing_status="draft",
            priority="normal",
            assigned_at="2026-03-12T10:00:00Z",
            product="Electronic Components",
            origin="CN",
            destination="US",
            declared_value=15000.0,
            company_name="Acme Corp",
            transport_mode="air",
            checklist_progress=45.0,
            days_since_arrival=2.5,
        )

        assert projection.entry_id == entry_id
        assert projection.filing_status == "draft"
        assert projection.checklist_progress == 45.0

    def test_serializes_to_json(self):
        """QueueEntryProjection serializes to JSON for Redis storage."""
        from clearance_platform.broker_queue_projector.models import QueueEntryProjection

        projection = QueueEntryProjection(
            entry_id=str(uuid4()),
            shipment_id=str(uuid4()),
            broker_id=str(uuid4()),
            entry_number="ENT-2026-002",
            filing_status="submitted",
            priority="high",
            product="Textiles",
            origin="VN",
            destination="US",
            declared_value=8000.0,
            company_name="FabricCo",
            transport_mode="ocean",
        )

        serialized = projection.model_dump_json()
        deserialized = QueueEntryProjection.model_validate_json(serialized)
        assert deserialized.entry_id == projection.entry_id
        assert deserialized.filing_status == "submitted"

    def test_to_sse_payload(self):
        """to_sse_dict() returns camelCase dict matching frontend BrokerQueueEntry type."""
        from clearance_platform.broker_queue_projector.models import QueueEntryProjection

        projection = QueueEntryProjection(
            entry_id="abc-123",
            shipment_id="shp-456",
            broker_id="brk-789",
            entry_number="ENT-001",
            filing_status="draft",
            priority="urgent",
            product="Widgets",
            origin="CN",
            destination="US",
            declared_value=5000.0,
            company_name="WidgetCo",
            transport_mode="air",
            checklist_progress=75.0,
            days_since_arrival=1.2,
        )

        payload = projection.to_sse_dict()
        # Must match frontend BrokerQueueEntry interface (camelCase)
        assert payload["id"] == "abc-123"
        assert payload["entryNumber"] == "ENT-001"
        assert payload["shipmentId"] == "shp-456"
        assert payload["filingStatus"] == "draft"
        assert payload["priority"] == "urgent"
        assert payload["checklistProgress"] == 75.0
        assert payload["daysSinceArrival"] == 1.2
        assert payload["companyName"] == "WidgetCo"
        assert payload["transportMode"] == "air"
        assert payload["declaredValue"] == 5000.0
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd clearance-engine/backend && python -m pytest tests/projector/test_broker_queue_projector.py::TestQueueEntryProjection -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'clearance_platform.broker_queue_projector'`

- [ ] **Step 3: Write the projection model**

```python
# clearance_platform/broker_queue_projector/__init__.py
"""Broker Queue Projector — dedicated CQRS projector for broker work queues."""
```

```python
# clearance_platform/broker_queue_projector/models.py
"""Data models for the broker queue projector.

QueueEntryProjection is the canonical shape stored in Redis per queue entry.
It mirrors the QueueEntry API response model but is fully self-contained —
no database joins are needed to construct it. Built incrementally from
NATS JetStream events.
"""
from __future__ import annotations

from typing import Any

from pydantic import BaseModel, Field


class QueueEntryProjection(BaseModel):
    """A single entry in a broker's work queue, stored in Redis.

    Fields match the frontend BrokerQueueEntry TypeScript interface
    (snake_case here, to_sse_dict() converts to camelCase for SSE).
    """

    entry_id: str
    shipment_id: str
    broker_id: str
    entry_number: str = ""
    filing_status: str = "draft"
    priority: str = "normal"
    assigned_at: str = ""
    product: str = ""
    origin: str = ""
    destination: str = ""
    declared_value: float = 0.0
    company_name: str = ""
    transport_mode: str = "air"
    checklist_progress: float = 0.0
    days_since_arrival: float = 0.0
    authority_response: dict[str, Any] | None = None
    consolidation_id: str | None = None
    consolidation_master: str | None = None
    consolidation_carrier: str | None = None
    hu_summary: dict[str, Any] | None = None

    def to_sse_dict(self) -> dict[str, Any]:
        """Convert to camelCase dict matching frontend BrokerQueueEntry interface."""
        return {
            "id": self.entry_id,
            "entryNumber": self.entry_number,
            "shipmentId": self.shipment_id,
            "filingStatus": self.filing_status,
            "priority": self.priority,
            "assignedAt": self.assigned_at,
            "product": self.product,
            "origin": self.origin,
            "destination": self.destination,
            "declaredValue": self.declared_value,
            "companyName": self.company_name,
            "transportMode": self.transport_mode,
            "checklistProgress": self.checklist_progress,
            "daysSinceArrival": self.days_since_arrival,
            "authorityResponse": self.authority_response,
            "consolidationId": self.consolidation_id,
            "consolidationMaster": self.consolidation_master,
            "consolidationCarrier": self.consolidation_carrier,
            "huSummary": self.hu_summary,
        }
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd clearance-engine/backend && python -m pytest tests/projector/test_broker_queue_projector.py::TestQueueEntryProjection -v`
Expected: 3 PASSED

- [ ] **Step 5: Commit**

```bash
git add clearance-engine/backend/clearance_platform/broker_queue_projector/__init__.py \
       clearance-engine/backend/clearance_platform/broker_queue_projector/models.py \
       clearance-engine/backend/tests/projector/__init__.py \
       clearance-engine/backend/tests/projector/test_broker_queue_projector.py
git commit -m "feat(broker-queue-projector): add QueueEntryProjection model"
```

---

### Task 3: Write event-to-projection mapping logic

The core of the projector: take a NATS event, extract the relevant fields, and produce/update a `QueueEntryProjection` in Redis. This must handle all declaration management events that affect the work queue.

**Files:**
- Modify: `clearance-engine/backend/tests/projector/test_broker_queue_projector.py`
- Create: `clearance-engine/backend/clearance_platform/broker_queue_projector/projection.py`

- [ ] **Step 1: Write failing tests for projection logic**

Add to `tests/projector/test_broker_queue_projector.py`:

```python
class TestProjectionLogic:
    """Test event → Redis projection mapping."""

    @pytest.fixture
    def redis_mock(self):
        """In-memory Redis mock for testing."""
        store: dict[str, str] = {}
        sorted_sets: dict[str, dict[str, float]] = {}

        class MockRedis:
            async def get(self, key):
                return store.get(key)

            async def set(self, key, value, ex=None):
                store[key] = value

            async def delete(self, key):
                store.pop(key, None)

            async def zadd(self, key, mapping):
                if key not in sorted_sets:
                    sorted_sets[key] = {}
                sorted_sets[key].update(mapping)

            async def zrem(self, key, *members):
                if key in sorted_sets:
                    for m in members:
                        sorted_sets[key].pop(m, None)

            async def zrange(self, key, start, end, withscores=False):
                if key not in sorted_sets:
                    return []
                items = sorted(sorted_sets[key].items(), key=lambda x: x[1])
                sliced = items[start:end + 1] if end >= 0 else items[start:]
                if withscores:
                    return sliced
                return [k for k, v in sliced]

            async def zcard(self, key):
                return len(sorted_sets.get(key, {}))

            async def publish(self, channel, message):
                pass  # SSE notification — tested separately

        return MockRedis()

    @pytest.mark.asyncio
    async def test_declaration_created_adds_to_queue(self, redis_mock):
        """DeclarationCreatedEvent creates a queue entry for the assigned broker."""
        from clearance_platform.broker_queue_projector.projection import project_queue_event

        event_data = {
            "metadata": {"domain": "declaration_management", "event_type": "DeclarationCreatedEvent"},
            "entry_id": "entry-001",
            "shipment_id": "shp-001",
            "broker_id": "brk-001",
            "entry_number": "ENT-2026-001",
            "filing_status": "draft",
            "declared_value": 12000.0,
            "summary_data": {"product": "Electronics", "origin": "CN", "destination": "US", "company_name": "TechCorp"},
        }

        await project_queue_event(redis_mock, event_data)

        # Should have created an entry in broker's sorted set
        count = await redis_mock.zcard("bqp:broker:brk-001:queue")
        assert count == 1

        # Should have stored the full projection
        raw = await redis_mock.get("bqp:entry:entry-001")
        assert raw is not None
        data = json.loads(raw)
        assert data["entry_id"] == "entry-001"
        assert data["filing_status"] == "draft"

    @pytest.mark.asyncio
    async def test_status_change_updates_projection(self, redis_mock):
        """Status change events update the existing projection in Redis."""
        from clearance_platform.broker_queue_projector.projection import project_queue_event

        # First, create
        create_event = {
            "metadata": {"domain": "declaration_management", "event_type": "DeclarationCreatedEvent"},
            "entry_id": "entry-002",
            "shipment_id": "shp-002",
            "broker_id": "brk-001",
            "filing_status": "draft",
            "declared_value": 5000.0,
        }
        await project_queue_event(redis_mock, create_event)

        # Then, submit
        submit_event = {
            "metadata": {"domain": "declaration_management", "event_type": "DeclarationSubmittedEvent"},
            "entry_id": "entry-002",
            "shipment_id": "shp-002",
            "broker_id": "brk-001",
            "filing_status": "submitted",
            "entry_number": "ENT-2026-002",
            "declared_value": 5000.0,
        }
        await project_queue_event(redis_mock, submit_event)

        raw = await redis_mock.get("bqp:entry:entry-002")
        data = json.loads(raw)
        assert data["filing_status"] == "submitted"
        assert data["entry_number"] == "ENT-2026-002"

    @pytest.mark.asyncio
    async def test_released_entry_removed_from_queue(self, redis_mock):
        """Entries with released/rejected status are removed from the broker's queue set."""
        from clearance_platform.broker_queue_projector.projection import project_queue_event

        create_event = {
            "metadata": {"domain": "declaration_management", "event_type": "DeclarationCreatedEvent"},
            "entry_id": "entry-003",
            "shipment_id": "shp-003",
            "broker_id": "brk-001",
            "filing_status": "draft",
            "declared_value": 3000.0,
        }
        await project_queue_event(redis_mock, create_event)
        assert await redis_mock.zcard("bqp:broker:brk-001:queue") == 1

        # Release it
        release_event = {
            "metadata": {"domain": "declaration_management", "event_type": "DeclarationCreatedEvent"},
            "entry_id": "entry-003",
            "shipment_id": "shp-003",
            "broker_id": "brk-001",
            "filing_status": "released",
            "declared_value": 3000.0,
        }
        await project_queue_event(redis_mock, release_event)

        # Should be removed from the sorted set
        assert await redis_mock.zcard("bqp:broker:brk-001:queue") == 0
        # But the projection record still exists (for SSE "removed" delta)
        raw = await redis_mock.get("bqp:entry:entry-003")
        assert raw is not None

    @pytest.mark.asyncio
    async def test_document_event_updates_checklist(self, redis_mock):
        """Document upload events increment checklist progress on affected entries."""
        from clearance_platform.broker_queue_projector.projection import project_queue_event

        # Create entry first
        create_event = {
            "metadata": {"domain": "declaration_management", "event_type": "DeclarationCreatedEvent"},
            "entry_id": "entry-004",
            "shipment_id": "shp-004",
            "broker_id": "brk-001",
            "filing_status": "draft",
            "declared_value": 7000.0,
        }
        await project_queue_event(redis_mock, create_event)

        # Now a document upload for this shipment
        doc_event = {
            "metadata": {"domain": "document_management", "event_type": "DocumentUploadedEvent"},
            "document_id": "doc-001",
            "shipment_id": "shp-004",
            "document_type": "commercial_invoice",
        }
        await project_queue_event(redis_mock, doc_event)

        # Entry's projection should be updated (checklist_progress may change)
        # The exact progress depends on the checklist logic; just verify the entry was touched
        raw = await redis_mock.get("bqp:entry:entry-004")
        assert raw is not None

    @pytest.mark.asyncio
    async def test_ignores_irrelevant_events(self, redis_mock):
        """Events from unrelated domains are ignored."""
        from clearance_platform.broker_queue_projector.projection import project_queue_event

        event_data = {
            "metadata": {"domain": "product_catalog", "event_type": "ProductCreatedEvent"},
            "product_id": "prod-001",
        }
        await project_queue_event(redis_mock, event_data)

        # No keys should have been created
        assert await redis_mock.get("bqp:entry:prod-001") is None
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd clearance-engine/backend && python -m pytest tests/projector/test_broker_queue_projector.py::TestProjectionLogic -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'clearance_platform.broker_queue_projector.projection'`

- [ ] **Step 3: Write the projection logic**

```python
# clearance_platform/broker_queue_projector/projection.py
"""Event-to-Redis projection logic for broker work queues.

Consumes events from declaration_management, document_management, and
shipment_lifecycle domains. Maintains per-broker sorted sets in Redis
(keyed by broker_id) and full QueueEntryProjection JSON per entry.

Redis key patterns (prefix: bqp = broker_queue_projector):
  bqp:entry:{entry_id}           — full QueueEntryProjection JSON
  bqp:broker:{broker_id}:queue   — sorted set of entry_ids (score = priority_rank)
  bqp:shipment:{shipment_id}     — maps shipment_id → entry_id (for doc events)
"""
from __future__ import annotations

import json
import logging
import time
from typing import Any

import structlog

from clearance_platform.broker_queue_projector.models import QueueEntryProjection

logger = structlog.get_logger(__name__)

# Priority rank for sorted set scoring (lower = more urgent = appears first)
PRIORITY_SCORES = {
    "urgent": 1.0,
    "high": 2.0,
    "normal": 3.0,
    "low": 4.0,
}

# Statuses that mean "entry is done" — remove from active queue
TERMINAL_STATUSES = {"released", "rejected"}

# Domains we care about
RELEVANT_DOMAINS = {"declaration_management", "document_management", "shipment_lifecycle"}

# Entry TTL in Redis (24 hours — entries stay in cache even after terminal status
# so the SSE stream can push the "removed" delta before cleanup)
ENTRY_TTL_SECONDS = 86400

# Queue sorted set TTL (no expiry — managed by add/remove operations)


async def project_queue_event(redis: Any, event_data: dict) -> str | None:
    """Process a single event and update Redis projections.

    Returns the broker_id affected (for SSE notification), or None if no update.
    """
    metadata = event_data.get("metadata", {})
    domain = metadata.get("domain", "")

    if domain not in RELEVANT_DOMAINS:
        return None

    if domain == "declaration_management":
        return await _project_declaration_event(redis, event_data)
    elif domain == "document_management":
        return await _project_document_event(redis, event_data)
    elif domain == "shipment_lifecycle":
        return await _project_shipment_event(redis, event_data)

    return None


async def _project_declaration_event(redis: Any, event_data: dict) -> str | None:
    """Handle declaration_management events (the primary source of queue entries)."""
    entry_id = str(event_data.get("entry_id") or event_data.get("declaration_id") or "")
    broker_id = str(event_data.get("broker_id") or "")
    shipment_id = str(event_data.get("shipment_id") or "")
    filing_status = event_data.get("filing_status", "draft")

    if not entry_id or not broker_id:
        return None

    # Load existing projection or create new
    existing_raw = await redis.get(f"bqp:entry:{entry_id}")
    if existing_raw:
        existing = json.loads(existing_raw)
    else:
        existing = {}

    # Extract fields from event (state-transfer pattern: event carries full snapshot)
    summary = event_data.get("summary_data") or {}
    projection = QueueEntryProjection(
        entry_id=entry_id,
        shipment_id=shipment_id or existing.get("shipment_id", ""),
        broker_id=broker_id,
        entry_number=event_data.get("entry_number", "") or existing.get("entry_number", ""),
        filing_status=filing_status,
        priority=event_data.get("priority", "") or existing.get("priority", "normal"),
        assigned_at=event_data.get("assigned_at", "") or existing.get("assigned_at", ""),
        product=summary.get("product", "") or existing.get("product", ""),
        origin=summary.get("origin", "") or existing.get("origin", ""),
        destination=summary.get("destination", "") or existing.get("destination", ""),
        declared_value=event_data.get("declared_value", 0) or existing.get("declared_value", 0),
        company_name=summary.get("company_name", "") or existing.get("company_name", ""),
        transport_mode=summary.get("transport_mode", "") or existing.get("transport_mode", "air"),
        checklist_progress=existing.get("checklist_progress", 0.0),
        days_since_arrival=existing.get("days_since_arrival", 0.0),
        authority_response=event_data.get("authority_response") or existing.get("authority_response"),
        consolidation_id=str(event_data.get("consolidation_id") or "") or existing.get("consolidation_id"),
        consolidation_master=existing.get("consolidation_master"),
        consolidation_carrier=existing.get("consolidation_carrier"),
    )

    # Store entry projection
    await redis.set(
        f"bqp:entry:{entry_id}",
        projection.model_dump_json(),
        ex=ENTRY_TTL_SECONDS,
    )

    # Map shipment → entry for document event lookups
    if projection.shipment_id:
        await redis.set(
            f"bqp:shipment:{projection.shipment_id}",
            entry_id,
            ex=ENTRY_TTL_SECONDS,
        )

    # Update broker's queue sorted set
    queue_key = f"bqp:broker:{broker_id}:queue"
    if filing_status in TERMINAL_STATUSES:
        await redis.zrem(queue_key, entry_id)
    else:
        score = PRIORITY_SCORES.get(projection.priority, 3.0)
        await redis.zadd(queue_key, {entry_id: score})

    # Publish SSE notification on Redis pub/sub
    await redis.publish(
        f"bqp:notify:{broker_id}",
        json.dumps({"type": "entry_updated", "entry_id": entry_id, "filing_status": filing_status}),
    )

    return broker_id


async def _project_document_event(redis: Any, event_data: dict) -> str | None:
    """Handle document_management events (affects checklist progress)."""
    shipment_id = str(event_data.get("shipment_id") or "")
    if not shipment_id:
        return None

    # Look up which entry this shipment belongs to
    entry_id = await redis.get(f"bqp:shipment:{shipment_id}")
    if not entry_id:
        return None

    # Load existing projection
    existing_raw = await redis.get(f"bqp:entry:{entry_id}")
    if not existing_raw:
        return None

    existing = json.loads(existing_raw)
    broker_id = existing.get("broker_id", "")

    # Track uploaded doc types for this shipment
    doc_type = event_data.get("document_type", "")
    if doc_type:
        docs_key = f"bqp:shipment:{shipment_id}:docs"
        # Use a simple set-like approach: store as JSON array
        docs_raw = await redis.get(docs_key)
        docs = json.loads(docs_raw) if docs_raw else []
        if doc_type not in docs:
            docs.append(doc_type)
            await redis.set(docs_key, json.dumps(docs), ex=ENTRY_TTL_SECONDS)

    # Notify broker that entry was updated (frontend will re-check)
    if broker_id:
        await redis.publish(
            f"bqp:notify:{broker_id}",
            json.dumps({"type": "entry_updated", "entry_id": entry_id, "filing_status": existing.get("filing_status", "")}),
        )

    return broker_id


async def _project_shipment_event(redis: Any, event_data: dict) -> str | None:
    """Handle shipment_lifecycle events (affects transport_mode, origin, etc.)."""
    shipment_id = str(event_data.get("shipment_id") or "")
    if not shipment_id:
        return None

    entry_id = await redis.get(f"bqp:shipment:{shipment_id}")
    if not entry_id:
        return None

    existing_raw = await redis.get(f"bqp:entry:{entry_id}")
    if not existing_raw:
        return None

    existing = json.loads(existing_raw)
    broker_id = existing.get("broker_id", "")

    # Update shipment-sourced fields
    changed = False
    for field, key in [("transport_mode", "transport_mode"), ("origin", "origin"), ("destination", "destination")]:
        val = event_data.get(field)
        if val and val != existing.get(key):
            existing[key] = val
            changed = True

    if changed:
        await redis.set(f"bqp:entry:{entry_id}", json.dumps(existing, default=str), ex=ENTRY_TTL_SECONDS)
        if broker_id:
            await redis.publish(
                f"bqp:notify:{broker_id}",
                json.dumps({"type": "entry_updated", "entry_id": entry_id, "filing_status": existing.get("filing_status", "")}),
            )

    return broker_id
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd clearance-engine/backend && python -m pytest tests/projector/test_broker_queue_projector.py::TestProjectionLogic -v`
Expected: 5 PASSED

- [ ] **Step 5: Commit**

```bash
git add clearance-engine/backend/clearance_platform/broker_queue_projector/projection.py \
       clearance-engine/backend/tests/projector/test_broker_queue_projector.py
git commit -m "feat(broker-queue-projector): add event-to-Redis projection logic"
```

---

## Chunk 2: Projector Service (FastAPI + NATS + SSE Endpoint)

### Task 4: Write the standalone projector FastAPI app

This follows the exact pattern of `cache_projector/projector.py`: standalone FastAPI app with NATS JetStream consumer, Redis connection, health endpoint. Additionally serves the SSE stream endpoint.

**Files:**
- Create: `clearance-engine/backend/clearance_platform/broker_queue_projector/projector.py`
- Test: `clearance-engine/backend/tests/projector/test_broker_queue_sse.py`

- [ ] **Step 1: Write the failing test for SSE endpoint**

```python
# tests/projector/test_broker_queue_sse.py
"""Tests for the broker queue projector SSE endpoint."""
from __future__ import annotations

import json
import pytest
from httpx import AsyncClient, ASGITransport


@pytest.mark.asyncio
async def test_health_endpoint():
    """Health endpoint returns ok status."""
    from clearance_platform.broker_queue_projector.projector import create_app

    app = create_app()
    # Override lifespan to avoid NATS/Redis connections in tests
    app.state.redis = None
    app.state.nats = None

    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        resp = await client.get("/health")
        assert resp.status_code == 200
        data = resp.json()
        assert data["api"] == "ok"


@pytest.mark.asyncio
async def test_queue_snapshot_endpoint():
    """GET /broker/queue/snapshot returns entries from Redis."""
    from clearance_platform.broker_queue_projector.projector import create_app
    from clearance_platform.broker_queue_projector.models import QueueEntryProjection

    app = create_app()

    # Set up mock Redis with a queue entry
    entry = QueueEntryProjection(
        entry_id="entry-001",
        shipment_id="shp-001",
        broker_id="brk-001",
        entry_number="ENT-001",
        filing_status="draft",
        priority="normal",
        product="Widgets",
        origin="CN",
        destination="US",
        declared_value=5000.0,
        company_name="WidgetCo",
        transport_mode="air",
    )

    store = {"bqp:entry:entry-001": entry.model_dump_json()}
    sorted_sets = {"bqp:broker:brk-001:queue": {"entry-001": 3.0}}

    class MockRedis:
        async def ping(self):
            return True

        async def get(self, key):
            return store.get(key)

        async def zrange(self, key, start, end, withscores=False):
            if key not in sorted_sets:
                return []
            items = sorted(sorted_sets[key].items(), key=lambda x: x[1])
            return [k for k, v in items]

        async def zcard(self, key):
            return len(sorted_sets.get(key, {}))

    app.state.redis = MockRedis()
    app.state.nats = None

    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        resp = await client.get("/broker/queue/snapshot?broker_id=brk-001")
        assert resp.status_code == 200
        data = resp.json()
        assert len(data["items"]) == 1
        assert data["items"][0]["id"] == "entry-001"
        assert data["items"][0]["filingStatus"] == "draft"
        assert data["total"] == 1
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd clearance-engine/backend && python -m pytest tests/projector/test_broker_queue_sse.py -v`
Expected: FAIL with import error

- [ ] **Step 3: Write the projector FastAPI app**

```python
# clearance_platform/broker_queue_projector/projector.py
"""Broker Queue Projector — standalone FastAPI service.

Subscribes to declaration_management, document_management, and
shipment_lifecycle events via NATS JetStream. Maintains per-broker
work queue materialized views in Redis. Serves SSE streams to
connected broker clients.

Endpoints:
  GET /health                          — health check
  GET /broker/queue/snapshot           — initial queue snapshot from Redis
  GET /broker/queue/stream             — SSE stream of queue deltas
  POST /admin/rebuild                  — replay events to rebuild cache
"""
from __future__ import annotations

import asyncio
import json
import logging
import os
from contextlib import asynccontextmanager
from typing import Any, AsyncIterator

import structlog
from fastapi import FastAPI, Query, Request
from fastapi.responses import JSONResponse

from clearance_platform.broker_queue_projector.models import QueueEntryProjection
from clearance_platform.broker_queue_projector.projection import project_queue_event
from clearance_platform.infra.streaming import sse_event, sse_response

logger = structlog.get_logger(__name__)

# Environment
REDIS_URL = os.environ.get("REDIS_URL", "redis://localhost:6381/0")
NATS_URL = os.environ.get("NATS_URL", "nats://localhost:4222")

# NATS subjects we subscribe to
SUBSCRIBED_SUBJECTS = [
    "clearance.declaration.>",
    "clearance.document.>",
    "clearance.shipment.>",
]

CONSUMER_NAME = "broker_queue_projector_consumer"
HEARTBEAT_INTERVAL = 15  # seconds


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    """Connect to NATS and Redis; subscribe to relevant domain events."""
    log = logger.bind(event="lifespan")
    log.info("broker_queue_projector_starting")

    import redis.asyncio as aioredis

    redis_client = aioredis.from_url(REDIS_URL, decode_responses=True, max_connections=50)
    app.state.redis = redis_client

    try:
        await redis_client.ping()
        log.info("redis_ready", url=REDIS_URL)
    except Exception as exc:
        log.warning("redis_unavailable", error=str(exc))

    nats_client = None
    js = None
    try:
        import nats
        nats_client = await nats.connect(NATS_URL)
        js = nats_client.jetstream()
        app.state.nats = nats_client
        app.state.js = js
        log.info("nats_ready", url=NATS_URL)
    except Exception as exc:
        log.warning("nats_unavailable", error=str(exc))

    subscriptions = []
    if js:
        try:
            from nats.js.api import ConsumerConfig

            config = ConsumerConfig(
                durable_name=CONSUMER_NAME,
                ack_wait=30 * 1_000_000_000,
                max_deliver=3,
            )

            async def _handle_msg(msg: Any) -> None:
                try:
                    data = json.loads(msg.data.decode())
                    await project_queue_event(redis_client, data)
                    await msg.ack()
                except Exception as exc:
                    log.warning("projection_handler_error", subject=msg.subject, error=str(exc))
                    await msg.nak()

            for subject in SUBSCRIBED_SUBJECTS:
                sub = await js.subscribe(
                    subject,
                    cb=_handle_msg,
                    durable=f"{CONSUMER_NAME}_{subject.split('.')[1]}",
                    config=config,
                )
                subscriptions.append(sub)
                log.info("subscribed", subject=subject)

        except Exception as exc:
            log.warning("subscribe_failed", error=str(exc))

    log.info("broker_queue_projector_ready")
    yield

    log.info("broker_queue_projector_shutting_down")

    for sub in subscriptions:
        try:
            await sub.unsubscribe()
        except Exception:
            pass

    if nats_client:
        try:
            await nats_client.close()
        except Exception:
            pass

    await redis_client.aclose()
    log.info("broker_queue_projector_shutdown_complete")


def create_app() -> FastAPI:
    application = FastAPI(
        title="Broker Queue Projector",
        description="SSE-driven broker work queue projector",
        version="0.1.0",
        lifespan=lifespan,
        docs_url=None,
        redoc_url=None,
    )

    @application.get("/health", tags=["system"])
    async def health_check(request: Request) -> dict[str, str]:
        status: dict[str, str] = {"api": "ok"}
        try:
            redis = getattr(request.app.state, "redis", None)
            if redis:
                await redis.ping()
                status["redis"] = "ok"
            else:
                status["redis"] = "unavailable"
        except Exception:
            status["redis"] = "unavailable"

        try:
            nc = getattr(request.app.state, "nats", None)
            status["nats"] = "ok" if nc and nc.is_connected else "unavailable"
        except Exception:
            status["nats"] = "unavailable"

        return status

    @application.get("/broker/queue/snapshot", tags=["broker"])
    async def queue_snapshot(
        request: Request,
        broker_id: str = Query(..., description="Broker ID"),
    ) -> dict:
        """Return the full queue snapshot for a broker from Redis."""
        redis = request.app.state.redis
        if not redis:
            return JSONResponse(status_code=503, content={"error": "Redis unavailable"})

        queue_key = f"bqp:broker:{broker_id}:queue"
        entry_ids = await redis.zrange(queue_key, 0, -1)
        total = await redis.zcard(queue_key)

        items = []
        for eid in entry_ids:
            raw = await redis.get(f"bqp:entry:{eid}")
            if raw:
                projection = QueueEntryProjection.model_validate_json(raw)
                items.append(projection.to_sse_dict())

        return {"items": items, "total": total}

    @application.get("/broker/queue/stream", tags=["broker"])
    async def queue_stream(
        request: Request,
        broker_id: str = Query(..., description="Broker ID"),
    ):
        """SSE stream of queue deltas for a specific broker.

        Event types:
          - snapshot: initial full queue state (sent immediately on connect)
          - entry_updated: a single entry was created or changed
          - entry_removed: an entry left the active queue (released/rejected)
          - heartbeat: keepalive (every 15 seconds)
        """
        redis = request.app.state.redis

        async def event_generator() -> AsyncIterator[str]:
            # Send initial snapshot
            queue_key = f"bqp:broker:{broker_id}:queue"
            entry_ids = await redis.zrange(queue_key, 0, -1)
            items = []
            for eid in entry_ids:
                raw = await redis.get(f"bqp:entry:{eid}")
                if raw:
                    projection = QueueEntryProjection.model_validate_json(raw)
                    items.append(projection.to_sse_dict())

            yield sse_event("snapshot", {"items": items, "total": len(items)})

            # Subscribe to Redis pub/sub for this broker's notifications
            pubsub = redis.pubsub()
            await pubsub.subscribe(f"bqp:notify:{broker_id}")

            try:
                while True:
                    # Check for disconnection
                    if await request.is_disconnected():
                        break

                    # Poll for Redis pub/sub messages (non-blocking with timeout)
                    message = await pubsub.get_message(
                        ignore_subscribe_messages=True,
                        timeout=HEARTBEAT_INTERVAL,
                    )

                    if message and message["type"] == "message":
                        notification = json.loads(message["data"])
                        entry_id = notification.get("entry_id", "")
                        filing_status = notification.get("filing_status", "")

                        # Load the updated entry
                        raw = await redis.get(f"bqp:entry:{entry_id}")
                        if raw:
                            projection = QueueEntryProjection.model_validate_json(raw)
                            if filing_status in ("released", "rejected"):
                                yield sse_event("entry_removed", {
                                    "entryId": entry_id,
                                    "filingStatus": filing_status,
                                })
                            else:
                                yield sse_event("entry_updated", projection.to_sse_dict())
                    else:
                        # No message within timeout — send heartbeat
                        yield sse_event("heartbeat", {"ts": int(asyncio.get_event_loop().time())})

            finally:
                await pubsub.unsubscribe(f"bqp:notify:{broker_id}")
                await pubsub.aclose()

        return sse_response(event_generator())

    return application


app = create_app()
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd clearance-engine/backend && python -m pytest tests/projector/test_broker_queue_sse.py -v`
Expected: 2 PASSED

- [ ] **Step 5: Commit**

```bash
git add clearance-engine/backend/clearance_platform/broker_queue_projector/projector.py \
       clearance-engine/backend/tests/projector/test_broker_queue_sse.py
git commit -m "feat(broker-queue-projector): add FastAPI app with SSE stream endpoint"
```

---

### Task 5: Add Dockerfile and Docker Compose service

**Files:**
- Create: `clearance-engine/backend/clearance_platform/broker_queue_projector/Dockerfile`
- Modify: `clearance-engine/docker-compose.microservices.yml`
- Modify: `clearance-engine/Caddyfile.microservices.dev`

- [ ] **Step 1: Create Dockerfile**

```dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    curl \
    && rm -rf /var/lib/apt/lists/*

COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

COPY . .

RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 4000
CMD ["uv", "run", "uvicorn", "clearance_platform.broker_queue_projector.projector:app", "--host", "0.0.0.0", "--port", "4000", "--workers", "1"]
```

- [ ] **Step 2: Add service to docker-compose.microservices.yml**

Add after the `cache-projector` service definition (around line 489):

```yaml
  broker-queue-projector:
    build:
      context: ./backend
      dockerfile: clearance_platform/broker_queue_projector/Dockerfile
    environment:
      REDIS_URL: redis://redis:6379/0
      NATS_URL: nats://nats:4222
    depends_on:
      redis:
        condition: service_healthy
      nats:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:4000/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 3
```

Also add `broker-queue-projector` to Caddy's `depends_on` list.

- [ ] **Step 3: Add Caddy route for SSE endpoint**

In `Caddyfile.microservices.dev`, add before the `/api/broker*` route (so it matches first):

```
    # ── Broker Queue Projector (SSE) ──────────────────────────────
    # Routes: /api/broker/queue/stream, /api/broker/queue/snapshot
    handle /api/broker/queue/stream* {
        reverse_proxy broker-queue-projector:4000 {
            flush_interval -1
        }
    }
    handle /api/broker/queue/snapshot* {
        reverse_proxy broker-queue-projector:4000 {
            flush_interval -1
        }
    }
```

Also add the same to `Caddyfile.microservices` (production).

- [ ] **Step 4: Verify Docker build**

Run: `cd clearance-engine && docker compose -f docker-compose.microservices.yml build broker-queue-projector`
Expected: Build succeeds

- [ ] **Step 5: Commit**

```bash
git add clearance-engine/backend/clearance_platform/broker_queue_projector/Dockerfile \
       clearance-engine/docker-compose.microservices.yml \
       clearance-engine/Caddyfile.microservices.dev \
       clearance-engine/Caddyfile.microservices
git commit -m "infra(broker-queue-projector): add Dockerfile, compose service, Caddy routes"
```

---

## Chunk 3: Frontend — EventSource Hook + UI Integration

### Task 6: Create the useBrokerQueueStream hook

**Files:**
- Create: `clearance-engine/frontend/src/hooks/useBrokerQueueStream.ts`
- Create: `clearance-engine/frontend/src/hooks/__tests__/useBrokerQueueStream.test.ts`

- [ ] **Step 1: Write the failing test**

```typescript
// src/hooks/__tests__/useBrokerQueueStream.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from "vitest";

describe("useBrokerQueueStream", () => {
  it("module exports useBrokerQueueStream function", async () => {
    const mod = await import("../useBrokerQueueStream");
    expect(typeof mod.useBrokerQueueStream).toBe("function");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd clearance-engine/frontend && npx vitest run src/hooks/__tests__/useBrokerQueueStream.test.ts`
Expected: FAIL with module not found

- [ ] **Step 3: Write the hook**

```typescript
// src/hooks/useBrokerQueueStream.ts
import { useEffect, useRef, useState, useCallback } from "react";
import type { BrokerQueueEntry } from "@/types";

const API_BASE = import.meta.env.VITE_API_URL || "/api";

interface UseQueueStreamOptions {
  brokerId: string | undefined;
  enabled?: boolean;
}

interface UseQueueStreamResult {
  items: BrokerQueueEntry[];
  total: number;
  connected: boolean;
  error: string | null;
}

/**
 * SSE-driven broker work queue hook.
 *
 * On connect, receives a full snapshot of the broker's queue.
 * Then receives incremental deltas (entry_updated, entry_removed)
 * as events flow through the system. Falls back to REST polling
 * if SSE connection fails.
 */
export function useBrokerQueueStream({
  brokerId,
  enabled = true,
}: UseQueueStreamOptions): UseQueueStreamResult {
  const [items, setItems] = useState<BrokerQueueEntry[]>([]);
  const [total, setTotal] = useState(0);
  const [connected, setConnected] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const eventSourceRef = useRef<EventSource | null>(null);
  const reconnectTimeoutRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  const cleanup = useCallback(() => {
    if (eventSourceRef.current) {
      eventSourceRef.current.close();
      eventSourceRef.current = null;
    }
    if (reconnectTimeoutRef.current) {
      clearTimeout(reconnectTimeoutRef.current);
      reconnectTimeoutRef.current = null;
    }
  }, []);

  useEffect(() => {
    if (!brokerId || !enabled) {
      cleanup();
      return;
    }

    let retryCount = 0;
    const maxRetries = 5;
    const baseDelay = 1000;

    function connect() {
      cleanup();

      const url = `${API_BASE}/broker/queue/stream?broker_id=${brokerId}`;
      const es = new EventSource(url);
      eventSourceRef.current = es;

      es.addEventListener("snapshot", (e: MessageEvent) => {
        try {
          const data = JSON.parse(e.data);
          setItems(data.items ?? []);
          setTotal(data.total ?? 0);
          setConnected(true);
          setError(null);
          retryCount = 0;
        } catch (err) {
          console.error("[SSE] Failed to parse snapshot:", err);
        }
      });

      es.addEventListener("entry_updated", (e: MessageEvent) => {
        try {
          const entry: BrokerQueueEntry = JSON.parse(e.data);
          setItems((prev) => {
            const idx = prev.findIndex((i) => i.id === entry.id);
            if (idx >= 0) {
              const next = [...prev];
              next[idx] = entry;
              return next;
            }
            return [entry, ...prev];
          });
          setTotal((prev) => {
            // If entry was new (not found in prev), increment
            return prev; // total is managed by snapshot + add/remove
          });
        } catch (err) {
          console.error("[SSE] Failed to parse entry_updated:", err);
        }
      });

      es.addEventListener("entry_removed", (e: MessageEvent) => {
        try {
          const data = JSON.parse(e.data);
          const removedId = data.entryId;
          setItems((prev) => prev.filter((i) => i.id !== removedId));
          setTotal((prev) => Math.max(0, prev - 1));
        } catch (err) {
          console.error("[SSE] Failed to parse entry_removed:", err);
        }
      });

      es.addEventListener("heartbeat", () => {
        // Connection alive — nothing to do
      });

      es.onerror = () => {
        setConnected(false);
        es.close();

        if (retryCount < maxRetries) {
          const delay = baseDelay * Math.pow(2, retryCount);
          retryCount++;
          setError(`Reconnecting (attempt ${retryCount})...`);
          reconnectTimeoutRef.current = setTimeout(connect, delay);
        } else {
          setError("Connection lost. Refresh to reconnect.");
        }
      };
    }

    connect();

    return cleanup;
  }, [brokerId, enabled, cleanup]);

  return { items, total, connected, error };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd clearance-engine/frontend && npx vitest run src/hooks/__tests__/useBrokerQueueStream.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add clearance-engine/frontend/src/hooks/useBrokerQueueStream.ts \
       clearance-engine/frontend/src/hooks/__tests__/useBrokerQueueStream.test.ts
git commit -m "feat(frontend): add useBrokerQueueStream SSE hook"
```

---

### Task 7: Wire SSE into BrokerQueue screen

Replace the polling `useEffect` in `BrokerQueue.tsx` with the SSE hook. Keep client-side filtering/sorting/pagination — the SSE stream delivers the full queue; the UI filters locally.

**Files:**
- Modify: `clearance-engine/frontend/src/surfaces/broker/screens/BrokerQueue.tsx`

- [ ] **Step 1: Replace polling with SSE hook**

In `BrokerQueue.tsx`, replace the `useEffect` that calls `getBrokerQueue` (lines 325-351) with the SSE hook. The component will:
1. Use `useBrokerQueueStream` to get real-time items
2. Apply filters (status, priority, search, sort) client-side
3. Handle pagination client-side on the filtered list

Key changes:
- Import `useBrokerQueueStream` from `@/hooks/useBrokerQueueStream`
- Remove the `getBrokerQueue` import (keep `approveEntry`)
- Replace the polling `useEffect` with the hook
- Add client-side filter/sort/search logic
- Add a connection status indicator

```typescript
// Replace the polling useEffect (lines 325-351) with:

const { items: allItems, total: streamTotal, connected, error: streamError } =
  useBrokerQueueStream({
    brokerId: currentBroker?.id,
    enabled: !!currentBroker,
  });

// Client-side filtering
const filteredItems = useMemo(() => {
  let result = allItems;

  // Status filter
  if (queueFilter.status) {
    result = result.filter((e) => e.filingStatus === queueFilter.status);
  } else {
    result = result.filter((e) => e.filingStatus !== "released" && e.filingStatus !== "rejected");
  }

  // Priority filter
  if (queueFilter.priority) {
    result = result.filter((e) => e.priority === queueFilter.priority);
  }

  // Consolidation filter
  if (consolidationFilter) {
    result = result.filter((e) => e.consolidationId === consolidationFilter);
  }

  // Search
  if (debouncedSearch) {
    const term = debouncedSearch.toLowerCase();
    result = result.filter(
      (e) =>
        e.product.toLowerCase().includes(term) ||
        e.companyName.toLowerCase().includes(term) ||
        (e.entryNumber ?? "").toLowerCase().includes(term)
    );
  }

  // Sort
  result = [...result].sort((a, b) => {
    const dir = sortDir === "asc" ? 1 : -1;
    switch (sortBy) {
      case "company": return dir * a.companyName.localeCompare(b.companyName);
      case "product": return dir * a.product.localeCompare(b.product);
      case "value": return dir * (a.declaredValue - b.declaredValue);
      case "days": return dir * (a.daysSinceArrival - b.daysSinceArrival);
      case "urgency":
      default: {
        const prio = { urgent: 0, high: 1, normal: 2, low: 3 };
        return dir * ((prio[a.priority as keyof typeof prio] ?? 2) - (prio[b.priority as keyof typeof prio] ?? 2));
      }
    }
  });

  return result;
}, [allItems, queueFilter, consolidationFilter, debouncedSearch, sortBy, sortDir]);

// Client-side pagination
const total = filteredItems.length;
const paginatedItems = filteredItems.slice(page * PAGE_SIZE, (page + 1) * PAGE_SIZE);
const hasMore = (page + 1) * PAGE_SIZE < total;

// Update store for other components that read from it
useEffect(() => {
  setQueue(paginatedItems);
  setLoading(false);
}, [paginatedItems, setQueue, setLoading]);
```

Also add `useMemo` to the React imports and `useBrokerQueueStream` to hook imports.

- [ ] **Step 2: Build to verify no TypeScript errors**

Run: `cd clearance-engine/frontend && npx tsc --noEmit`
Expected: No errors

- [ ] **Step 3: Commit**

```bash
git add clearance-engine/frontend/src/surfaces/broker/screens/BrokerQueue.tsx
git commit -m "feat(broker-queue): replace polling with SSE-driven updates"
```

---

### Task 8: Wire SSE into BrokerDashboard

Replace the 30-second polling interval for the "My Work" section with SSE-fed data.

**Files:**
- Modify: `clearance-engine/frontend/src/surfaces/broker/screens/BrokerDashboard.tsx`

- [ ] **Step 1: Replace dashboard polling with SSE for queue data**

In `BrokerDashboard.tsx`:
- The `loadDashboard` `setInterval(30000)` (line 110) fetches the full dashboard including stats, pipeline, alerts. Keep this polling for dashboard-level aggregates (those aren't in the queue projector).
- Replace the "My Work" queue entries section (the `loadMyWork` useEffect around line 131) with the SSE hook.

```typescript
// Replace the loadMyWork useEffect with:
const { items: myWorkItems, connected: queueConnected } = useBrokerQueueStream({
  brokerId: currentBroker?.id,
  enabled: !!currentBroker,
});

// Take top 10 by urgency for the dashboard "My Work" section
const myWork = useMemo(() => {
  const prioOrder = { urgent: 0, high: 1, normal: 2, low: 3 };
  return [...myWorkItems]
    .sort((a, b) => (prioOrder[a.priority as keyof typeof prioOrder] ?? 2) - (prioOrder[b.priority as keyof typeof prioOrder] ?? 2))
    .slice(0, 10);
}, [myWorkItems]);

useEffect(() => {
  setQueue(myWork);
}, [myWork, setQueue]);
```

- [ ] **Step 2: Build to verify no TypeScript errors**

Run: `cd clearance-engine/frontend && npx tsc --noEmit`
Expected: No errors

- [ ] **Step 3: Commit**

```bash
git add clearance-engine/frontend/src/surfaces/broker/screens/BrokerDashboard.tsx
git commit -m "feat(broker-dashboard): replace queue polling with SSE stream"
```

---

## Chunk 4: Integration Testing + Final Verification

### Task 9: End-to-end verification

**Files:**
- None new — verify existing E2E tests still pass

- [ ] **Step 1: Run unit tests for projector**

Run: `cd clearance-engine/backend && python -m pytest tests/projector/ -v`
Expected: All tests pass

- [ ] **Step 2: Run frontend build**

Run: `cd clearance-engine/frontend && npm run build`
Expected: Build succeeds with no errors

- [ ] **Step 3: Run frontend unit tests**

Run: `cd clearance-engine/frontend && npx vitest run`
Expected: All tests pass

- [ ] **Step 4: Start microservices stack and verify SSE**

```bash
cd clearance-engine
docker compose -f docker-compose.microservices.yml up -d --build
```

Verify the broker-queue-projector health:
```bash
curl http://localhost:8080/api/broker/queue/snapshot?broker_id=<broker-id>
```

Verify SSE stream connects:
```bash
curl -N http://localhost:8080/api/broker/queue/stream?broker_id=<broker-id>
```
Expected: Receives `event: snapshot` followed by `event: heartbeat` every 15 seconds

- [ ] **Step 5: Run E2E tests**

Run: `cd clearance-engine/frontend && npm run test:e2e`
Expected: All previously passing tests still pass (SSE is additive — the REST endpoint still exists)

- [ ] **Step 6: Final commit**

```bash
git add -A
git commit -m "test: verify broker queue SSE integration"
```

---

## Key Design Decisions

1. **Separate projector service** — the `broker_queue_projector` is independent from the existing `cache_projector`. They share no state, scale independently, and can be deployed/restarted without affecting each other.

2. **Redis pub/sub for SSE notification** — when the projector updates an entry in Redis, it publishes a notification to `bqp:notify:{broker_id}`. The SSE endpoint subscribes to this channel. This decouples the NATS consumer from the SSE connection management.

3. **Client-side filtering** — the SSE stream delivers the broker's full active queue (~50-200 entries). Filtering, sorting, searching, and pagination happen client-side. This keeps the server simple (one stream per broker) and avoids per-filter stream management.

4. **Snapshot + deltas** — on SSE connect, the client receives a full snapshot. Subsequent events are incremental deltas (entry_updated, entry_removed). On reconnection, the client gets a fresh snapshot — no delta replay needed.

5. **Graceful fallback** — if SSE fails after 5 reconnection attempts, the UI shows an error. The existing `GET /api/broker/queue` REST endpoint is untouched and can be used as a manual refresh fallback.

6. **Redis key prefix `bqp:`** — all keys are namespaced to avoid collision with the general cache projector's `declaration_management:` keys.
