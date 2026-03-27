# Architecture Migration Plan: From Poll-and-Lock to Event-Driven

**Author:** design-challenger (architecture reviewer)
**Status:** Final — Proactive Evaluator principle adopted, single-writer Aggregator, 10 contexts / ~106 events / 6 streams
**Date:** 2026-02-08
**Inputs:** `architecture-event-driven-design.md`, `architecture-feasibility-check.md`, `architecture-trade-offs.md`

---

## 1. Overview

This plan describes the migration from the current poll-and-lock architecture to the event-driven architecture. The migration addresses both performance (7-60s API response times) and correctness (silent SKIP LOCKED processing gaps).

**Architecture target:**
- **Proactive Evaluator pattern** — 7 actors evaluate readiness criteria and act at the earliest possible moment (customs, broker_sim, compliance, financial, documents, PGA, ISF), not at sequential status gates
- 5 actors use a hybrid pattern — event-triggered start + sim-clock polling for completion
- Remaining actors are reactive responders (cbp_authority) or timer-driven producers
- **New state machine transition: `in_transit → cleared`** — `at_customs` becomes optional when pre-clearance succeeds
- **Shipment Aggregator** — single writer to the entire `shipments` table; consumes all 6 streams and projects state
- Dashboard and shipment list served from Redis read model projections — zero SQL queries
- Redis Streams replace Redis Pub/Sub EventBus — persistent, acknowledged, consumer-group-based delivery
- `transition()` split into `validate_transition()` (pure guard) + event emission — actors validate and emit, Aggregator applies

**Business domains:** The design document (Section 2) defines **10 bounded contexts**: Product Catalog, Trade Intelligence, Commercial, Transport & Logistics, Customs & Declarations, Broker Operations, Compliance & Screening, Exception Management, Financial Settlement, and Regulatory Intelligence. Four of these (Product, Trade Intelligence, Commercial, Regulatory) have no simulation actors or only reference data — they are unaffected by the migration. The remaining 6 domains contain all 19 actors that must be migrated.

**Migration ordering** is driven by the feasibility check's actor-by-actor analysis (12 clean, 5 adaptation, 2 significant rework) and aligns with the design document's phasing (Section 15). The migration proceeds in 4 phases: infrastructure → highest-value actors → complex actors + read models → cleanup.

---

## 2. Phase 0: Event Bus Infrastructure (Days 1-2)

**Goal:** Stand up the event bus infrastructure and prepare foundational patterns. Pool separation is included as an enabler for safe migration, not as a standalone fix.
**Risk:** Low — no actor behavior changes
**Effort:** 2 days (1 engineer)

### 2.1 Separate Connection Pools (Enabler)

**File:** `clearance-engine/backend/app/main.py`

Create two SQLAlchemy engines to isolate actor writes from API reads:

```python
# Actor pool — used by simulation actors for writes
actor_engine = create_async_engine(
    settings.DATABASE_URL,
    pool_size=10, max_overflow=5,
    pool_timeout=10,     # Actors can queue
)
actor_session_factory = sessionmaker(actor_engine, class_=AsyncSession, expire_on_commit=False)

# API pool — used by HTTP endpoints
api_engine = create_async_engine(
    settings.DATABASE_URL,
    pool_size=15, max_overflow=5,
    pool_timeout=3,      # API fails fast
)
api_session_factory = sessionmaker(api_engine, class_=AsyncSession, expire_on_commit=False)
```

- Pass `actor_session_factory` to `SimulationCoordinator`
- Set `app.state.db_session_factory = api_session_factory` for API routes
- Dispose both engines on shutdown

**Why this is an enabler:** Pool separation prevents API starvation during migration when some actors are still poll-based. Once all actors are event-driven, actor DB usage drops to ~5ms per single-row update — the actor pool becomes a burst-capacity pool.

### 2.2 Implement EventStreamBus

**File:** `clearance-engine/backend/app/simulation/event_bus.py` (extend existing file)

Build `EventStreamBus` class alongside existing `EventBus`:

```python
class EventStreamBus:
    """Redis Streams-based event bus with consumer groups."""

    async def publish(self, stream: str, event_type: str, payload: dict) -> str:
        """Publish event to Redis Stream. Returns event ID."""

    async def read_stream(
        self, stream: str, group: str, consumer: str,
        count: int = 10, block_ms: int = 500
    ) -> list[tuple[str, dict]]:
        """Read events from a consumer group."""

    async def ack(self, stream: str, group: str, event_id: str) -> None:
        """Acknowledge an event as processed."""

    async def create_groups(self) -> None:
        """Create consumer groups for all 6 streams."""

    async def get_pending_info(self, stream: str, group: str, event_id: str) -> dict:
        """Get delivery count for dead letter decisions."""

    async def trim_streams(self) -> None:
        """Trim all streams to configured MAXLEN."""
```

- Create consumer groups for all 6 streams per the design document (Section 4.2-4.3): `stream:shipment-lifecycle`, `stream:customs-processing`, `stream:transport-ops` (NEW — highest volume), `stream:transit-events`, `stream:exception-resolution`, `stream:compliance`
- `stream:transport-ops` has `maxlen: 30_000` (highest volume: ~10 mode-specific events per shipment — air/ocean/ground operational detail)
- `stream:transit-events` has `maxlen: 20_000` (high volume: multiple intermediate events per shipment per transit leg)
- Wire `EventStreamBus` into coordinator alongside existing `EventBus`
- All existing behavior unchanged — actors still poll, but event infrastructure is ready

### 2.3 Define ShipmentEventPayload Schema

**File:** `clearance-engine/backend/app/simulation/events.py` (new file)

Bounded event payload schema (addresses Trade-off Challenge 4.6):

```python
@dataclass
class ShipmentEventPayload:
    id: str
    status: str
    previous_status: str | None
    product: str
    origin: str
    destination: str
    carrier: str | None
    transport_mode: str
    declared_value: float
    company_name: str
    hs_code: str | None
    hold_type: str | None
    entry_number: str | None
    consolidation_id: str | None
    cage_status: dict | None
    financials: dict | None
    references: dict | None
    codes: dict | None
    waypoints: list[dict] | None
    jurisdiction: str | None           # Destination jurisdiction (US/EU/BR/IN/CN)
    territory: str | None              # Transit territory (for intermediate events)
    created_at: str
    updated_at: str
    # EXCLUDED: events (history), analysis (large LLM text), cargo_detail

    @classmethod
    def from_shipment(cls, shipment: Shipment) -> "ShipmentEventPayload": ...
    def to_dict(self) -> dict: ...
```

### 2.4 Define ProcessingOutcome Pattern

**File:** `clearance-engine/backend/app/simulation/actors/outcomes.py` (new file)

Interface between pure business logic and ORM persistence (addresses Trade-off Challenge 4.4):

```python
@dataclass
class ProcessingOutcome:
    new_status: str | None = None
    hold_type: str | None = None
    description: str = ""
    codes_update: dict | None = None
    financials_update: dict | None = None
    cage_status_update: dict | None = None
    references_update: dict | None = None
    event_type: str = ""
    stream: str = ""
    extra_event_data: dict | None = None

async def apply_outcome(
    session: AsyncSession, shipment: Shipment,
    outcome: ProcessingOutcome, actor_id: str,
    sim_time: Any, event_bus: EventStreamBus,
) -> None:
    """Apply ProcessingOutcome to ORM object and publish events."""
```

### 2.5 Implement EventDrivenActor Base Class

**File:** `clearance-engine/backend/app/simulation/actors/base.py`

Add `EventDrivenActor` alongside existing `BaseActor`:

```python
class EventDrivenActor(BaseActor):
    """Base class for event-driven actors.
    Replaces tick-based poll loop with event consumption.
    Hybrid actors override both handle_event() AND tick().
    """
    subscriptions: list[tuple[str, str, list[str]]] = []

    async def _run_loop(self) -> None:
        """Consume events, optionally run periodic ticks for hybrid actors."""
        # As specified in architecture-event-driven-design.md Section 5.4

    async def handle_event(self, event: dict) -> None:
        """Override to process events. Must be idempotent."""
        raise NotImplementedError

    async def _handle_processing_error(self, stream, group, event_id, event, exc):
        """Dead letter after 3 retries, structured logging."""
        # As specified in architecture-event-driven-design.md Section 9.1
```

### 2.6 JSONB Append — Transitional Safety Net

**File:** `clearance-engine/backend/app/simulation/state_machine.py` (or equivalent `transition()` utility)

**Context:** In the target architecture, the Shipment Aggregator is the sole writer to the `shipments` table (including the `events` JSONB column). During migration (Phase 0 through Phase 2a), some actors still call the old `transition()` which appends to `events`. The atomic SQL append is a **transitional safety net**, not the long-term solution.

```python
# BEFORE (race-prone read-modify-write):
events = list(shipment.events or [])
events.append(new_event)
shipment.events = events
flag_modified(shipment, "events")

# AFTER (atomic SQL append — transitional safety net):
from sqlalchemy import text
await session.execute(
    text("UPDATE shipments SET events = COALESCE(events, '[]'::jsonb) || :event::jsonb WHERE id = :id"),
    {"event": json.dumps([new_event]), "id": str(shipment.id)}
)
```

**Migration timeline for this code:**
- **Phase 0:** Atomic append added to `transition()` — safety net while actors still call it
- **Phase 1-2a:** Actors progressively stop calling `transition()` for event append; they emit events to streams instead
- **Phase 2b:** Shipment Aggregator deployed, begins writing `events` column from stream consumption
- **Phase 3:** Remove event appends from `transition()` entirely. Aggregator is sole writer. Atomic append code removed.

### 2.7 Proactive Evaluator Infrastructure (Aligned with Design Doc Section 5.5)

**File:** `clearance-engine/backend/app/simulation/actors/proactive.py` (new file)

Implement `ProactiveEvaluatorMixin` with readiness accumulator:

```python
class ProactiveEvaluatorMixin:
    READINESS_FIELDS: set[str] = set()  # Subclass defines

    def __init__(self, ...):
        self._readiness: dict[str, dict] = {}  # shipment_id → accumulated state

    def _accumulate(self, shipment_id: str, payload: dict) -> dict:
        """Merge new event data into readiness accumulator."""
        state = self._readiness.setdefault(shipment_id, {})
        for field in self.READINESS_FIELDS:
            if field in payload and payload[field] is not None:
                state[field] = payload[field]
        return state

    def _is_ready(self, state: dict) -> bool:
        return all(state.get(f) is not None for f in self.READINESS_FIELDS)

    def _cleanup(self, shipment_id: str) -> None:
        self._readiness.pop(shipment_id, None)
```

### 2.8 Add `ClassificationCompleted` Event + State Machine Transition

**Files:**
- `clearance-engine/backend/app/simulation/actors/preclearance.py` — add `ClassificationCompleted` event publish after preclearance screening (enables downstream proactive evaluation)
- `clearance-engine/backend/app/simulation/state_machines/us.py` (and other jurisdictions) — add `in_transit → cleared` transition (allowed for `customs` actor when pre-clearance succeeds)

```python
# In us.py — add new transition:
"in_transit": {
    "at_customs": {"carrier"},
    "cleared": {"customs"},     # NEW — pre-clearance success path
    "held": {"platform", "carrier"},
},
```

The `at_customs` state becomes **optional** — it only occurs when pre-clearance was incomplete or physical inspection is required. This is the key state machine change that enables the proactive model.

### Phase 0 Success Criteria

- [ ] Actor pool and API pool are separate
- [ ] `EventStreamBus` can publish to and read from Redis Streams
- [ ] All 6 streams exist with consumer groups created (including `stream:transport-ops` and `stream:transit-events`)
- [ ] `EventDrivenActor` base class implemented
- [ ] `ProactiveEvaluatorMixin` implemented with readiness accumulator (design doc Section 5.5)
- [ ] `ShipmentEventPayload` and `ProcessingOutcome` defined
- [ ] `ClassificationCompleted` event added to preclearance actor output
- [ ] State machine transition `in_transit → cleared` added (pre-clearance success path)
- [ ] JSONB `events` append uses atomic SQL (transitional safety net — sole writer is Aggregator in Phase 2b)
- [ ] All 197 unit tests pass (no actor behavior changed)
- [ ] E2E tests pass

---

## 3. Phase 1: Highest-Value Actor Migration (Days 3-6)

**Goal:** Migrate the highest-contention actors and cleanest conversions, then moderate adaptation actors. Aligns with design document Section 15, Phase 1a/1b.
**Risk:** Low-Medium — clean conversions per feasibility check; adaptation actors use established hybrid pattern
**Effort:** 3-4 days (1-2 engineers)

### 3.1 Phase 1a — Proactive Evaluators, Highest Contention (Days 3-4)

| Order | Actor | Pattern | Impact | Subscribes To (proactive) | Earliest Action |
|-------|-------|---------|--------|--------------------------|----------------|
| 1 | **customs** | **Proactive Evaluator** | CRITICAL — eliminates `at_customs` hotspot + enables pre-clearance | `ShipmentCreated`, `ClassificationCompleted`, `ShipmentArrivedAtCustoms` | Pre-files declaration while `in_transit` |
| 2 | **compliance_engine** | **Proactive Evaluator** | MEDIUM — fixes existing race condition + re-screens on regulatory changes | `ShipmentCreated`, `ClassificationCompleted`, `RegulatorySignalDetected` | Re-screens on regulatory changes (not one-shot) |
| 3 | **shipper_response** | Event-reactive | LOW — eliminates full table scan | `CommunicationSent`, `CF28Issued`, `HoldEscalated` | Event-reactive (not proactive) |
| 4 | **financial** | **Proactive Evaluator** | MEDIUM — pre-calculates fees | `ClassificationCompleted`, `ShipmentCleared`, `CageReleased` | Pre-calculates fees as soon as tariff data available |
| 5 | **documents** | **Proactive Evaluator** | LOW — validates docs pre-departure | `ShipmentCreated`, `DocumentUploaded`, `ClassificationCompleted` | Validates as soon as docs uploaded |

Also: Add `ShipmentCreated` event publishing to `shipper` actor (timer-driven producer — minimal change).

### 3.2 Migrate Customs Actor (Pilot — Days 3-4)

**File:** `clearance-engine/backend/app/simulation/actors/customs.py`

**Step 1: Extract business logic into pure functions**

Current `_manual_review` (415 lines) becomes:
- `_determine_screening_outcome(shipment_state: dict, config, sim_time, rng) -> ProcessingOutcome`
  - Pure function: takes dict input, returns ProcessingOutcome
  - All 6 code paths (UFLPA, entity screening, ADD/CVD, inspection, HS reclass, CF-28/29, manual clear) preserved
  - `ProcessingOutcome.event_type` is now jurisdiction-specific (e.g., `ShipmentInspection` for US, `UnderControl` for EU, `Parameterized` for BR — design doc Section 3.3)
  - No session parameter, no ORM mutations, no event publishing

- `_determine_stp_outcome(shipment_state: dict, config, sim_time, rng, backlogged: bool) -> ProcessingOutcome`
  - Pure function for the STP fast-lane path

**Step 2: Address cross-shipment dependency (Trade-off 4.3)**

Replace DB `COUNT(*)` with Redis sorted set cardinality (design doc Section 11.5):
```python
async def handle_event(self, event: dict) -> None:
    shipment_state = event["payload"]
    at_customs_count = await self.redis.zcard("read:shipments:by_status:at_customs")
    backlogged = at_customs_count > self.config.backlog_threshold

    outcome = self._determine_screening_outcome(
        shipment_state, self.config, self.clock.now(), self.rng
    )

    async with self.get_session() as session:
        async with session.begin():
            shipment = await session.get(Shipment, shipment_state["id"])
            if not shipment or shipment.status != "at_customs":
                return  # Idempotent skip
            await apply_outcome(session, shipment, outcome, self.actor_id,
                                self.clock.now(), self.event_bus)
```

**Step 3: Wire proactive event subscriptions**
```python
class CustomsActor(EventDrivenActor, ProactiveEvaluatorMixin):
    READINESS_FIELDS = {"hs_code", "origin", "destination", "declared_value", "company_name"}
    subscriptions = [
        ("stream:shipment-lifecycle", "cg:customs-actor", ["ShipmentCreated", "ShipmentArrivedAtCustoms"]),
        ("stream:compliance", "cg:customs-actor", ["ClassificationCompleted"]),
    ]
    # Evaluates readiness on EACH event — pre-files declaration as soon as all data available
    # ShipmentArrivedAtCustoms triggers instant clearance if already pre-cleared
```

**Verification:**
- [ ] All existing customs unit tests pass (test pure functions directly)
- [ ] New event handler tests verify correct event consumption and publishing
- [ ] Side-by-side comparison: run poll-based and event-driven customs, verify identical outcomes for 100 shipments

### 3.3 Migrate Remaining Phase 1a Actors (Day 4)

| Actor | Migration Notes |
|-------|----------------|
| **compliance_engine** | Subscribes to all lifecycle events. Check `analysis IS NOT NULL` for idempotency. Consumer group fixes the existing race condition. |
| **shipper_response** | Subscribes to `CommunicationSent`, `CF28Issued`, `HoldEscalated`. Replaces full table scan. |
| **financial** | Subscribes to `ShipmentCleared`, `CageReleased`, `ShipmentDelivered`, `DutyAssessed` (IN jurisdiction). Guard-flag `mpf_calculated` already handles idempotency. Receives from `stream:customs-processing` (for `DutyAssessed`) and `stream:exception-resolution` (for `CageReleased`). |
| **documents** | Subscribes to `ShipmentCreated`. Guard-flag prevents re-processing. Trivial conversion. |

### 3.4 Wire Event Publishing Into Transition Points

**Note:** In the target architecture (Phase 3), actors emit events only — the Shipment Aggregator writes to DB. During Phase 1, the write path is transitional: actors still write to DB via `transition()` AND publish events. This dual-write enables Phase 1 event consumers to receive events from actors not yet fully migrated.

All actors that call `transition()` and produce status changes must now also publish events via `EventStreamBus`. For actors not yet migrated, add event publishing after existing DB commits:

```python
# In carrier.py (not yet migrated), after committing a status change:
await session.commit()
try:
    await self.event_bus_streams.publish(
        "stream:shipment-lifecycle", "ShipmentArrivedAtCustoms",
        ShipmentEventPayload.from_shipment(shipment).to_dict()
    )
except Exception:
    logger.error("event_publish_failed", shipment_id=str(shipment.id))
```

This is critical — Phase 1 event-driven actors need events from actors that aren't yet migrated.

### 3.5 Phase 1b — Adaptation Actors + Proactive Broker (Days 5-6)

Migrate adaptation actors, remaining clean conversions, and the broker claim handler as a proactive evaluator.

| Order | Actor | Pattern | Key Adaptation |
|-------|-------|---------|---------------|
| 1 | **pga** | **Proactive Evaluator** + Hybrid | Determines PGA requirements as soon as HS code known (`ClassificationCompleted`); hybrid with persisted review_days for completion |
| 2 | **broker_sim** (claim handler) | **Proactive Evaluator** | Assigns broker and begins prep during `booked`/`in_transit` — subscribes to `ShipmentCreated`, `ClassificationCompleted`. No longer waits for `at_customs`. |
| 3 | **resolution** | Hybrid | Time-dependent review windows — persisted durations |
| 4 | **cage** | Hybrid | Intake is event-driven; dwell → lazy computation (Section 11.3); GO warnings on timer |
| 5 | **terminal** | Event-driven | Mode filter for ocean shipments; trivial |
| 6 | **preclearance** | Event-driven | Already has BATCH_LIMIT=50; trivial conversion |
| 7 | **isf** | **Proactive Evaluator** | Already proactive — files ISF pre-departure for ocean shipments |

### Phase 1 Success Criteria

- [ ] 11 actors migrated (5 clean event-driven + 3 hybrid + 3 remaining clean)
- [ ] `at_customs` contention eliminated — customs, PGA, cage, compliance, terminal no longer poll
- [ ] **Proactive Evaluators act within 1 event cycle of prerequisites being met** — customs pre-files during `in_transit`, broker claims during `booked`, financial pre-calculates on `ClassificationCompleted`
- [ ] **Readiness accumulators track prerequisite state** — verified via unit tests that partial data accumulates correctly and `_is_ready()` triggers at the right moment
- [ ] Hybrid actors persist computed durations — no non-deterministic re-rolls
- [ ] Cage uses lazy dwell computation — no per-tick dwell updates
- [ ] All 197+ unit tests pass
- [ ] New event handler tests for each migrated actor
- [ ] Shadow mode comparison validates event-driven outcomes match poll-based for customs
- [ ] No `FOR UPDATE` queries in any pure event-driven actor
- [ ] Structured logging with correlation IDs on all event processing

---

## 4. Phase 2: Complex Actors + Read Models (Days 7-10)

**Goal:** Rework the 2 hardest actors, build read model projections. Aligns with design document Section 15, Phase 2a/2b.
**Risk:** Medium-High — most complex actors (carrier, broker_sim), but patterns established in Phase 1
**Effort:** 3-4 days (1-2 engineers)

### 4.1 Phase 2a — Complex Actor Rework + transition() Split (Days 7-8)

**Critical change: `transition()` → `validate_transition()` split**

At this phase, `transition()` is split into two parts:
- **`validate_transition(shipment, to_status, actor, jurisdiction)`** — pure guard function. Checks `can_transition()`, raises `TransitionError` if invalid. Does NOT modify the shipment record. Does NOT append to `events`.
- **Event emission** — actors call `validate_transition()`, then emit the event to Redis Streams. The Shipment Aggregator (Phase 2b) applies the status change and appends to the `events` timeline.

During Phase 2a, the old `transition()` remains available for actors not yet fully migrated, but migrated actors use `validate_transition()` + event emission exclusively.

| Order | Actor | Conversion | Key Challenge |
|-------|-------|-----------|--------------|
| 1 | **broker_sim** | Significant rework | Decompose into 3 handlers (design doc Section 5.6): BrokerClaimHandler, BrokerProgressHandler, BrokerCF28Handler |
| 2 | **cbp_authority** | Moderate | **Reactive Responder** — adjudicates filings, does NOT evaluate readiness proactively. Two trigger types: command-driven (`DeclarationSubmitted`, `EntryFiled`) and exception-driven (`ShipmentArrivedWithoutClearance`). Multi-table writes (EntryFiling + Shipment), 4 subscription types. |
| 3 | **carrier** | Significant rework | Hybrid with persisted transit_days, consolidation side-effects. Publishes mode-specific operational events to `stream:transport-ops` (35 types: air 13, ocean 13, ground 9 — design doc Section 3.5) AND intermediate transit events to `stream:transit-events` (13 types — design doc Section 3.4). Highest event publishing volume. |
| 4 | **consolidator** | Adaptation | Buffered consumer pattern (design doc Section 11.2): event-triggered candidate registration + timer-based group flush |

### 4.2 BrokerSim Actor Decomposition (Trade-off Challenge 4.5)

---

## 5. Phase 3: Read Models + Cleanup + Polish (Days 11-12)

**Goal:** Build read model projections, convert remaining actors, full test suite, decommission old patterns. Aligns with design document Section 15, Phase 2b + Phase 3.
**Risk:** Low-Medium — read models are additive, remaining actors are straightforward
**Effort:** 1-2 days (1-2 engineers)

### 5.1 Shipment Aggregator + Read Model Projections (Phase 2b from design doc)

**Shipment Aggregator (single writer to `shipments` table):**

**Files:** `clearance-engine/backend/app/simulation/projectors/shipment_aggregator.py` (new)

The Shipment Aggregator is the most significant new component. It replaces the current pattern where 12+ actors independently write to the `shipments` table.

1. **Subscribes to all 6 streams** via `cg:shipment-aggregator` consumer group
2. **Maintains the entire `shipments` record as a projection:**
   - `status` — projected from lifecycle events (applies `validate_transition()` guard before writing)
   - `events` — timeline projection (single writer = no JSONB race)
   - `hold_type` — projected from `ShipmentHeld` events
   - `entry_number` — projected from `EntryFiled` events
   - `cage_status` — projected from `CageIntake`/`CageReleased` events
   - `financials` — projected from `FeesCalculated` events
   - `references` — projected from various actor events
   - `compliance_result`, `classification_result`, `tariff_result`, `fta_result` — projected from respective domain events
3. **Also maintains Redis read stores** (sorted sets, summary hashes) — combined with ShipmentProjector role
4. **Shadow mode validation** (Phase 2a-2b): Aggregator runs alongside old `transition()` writes, comparing outputs. Alerts on drift. Cutover only when validated.
5. **Health check**: Monitors `cg:shipment-aggregator` pending count. Alert threshold: `pending > 50`.

**DashboardProjector:**

**Files:** `clearance-engine/backend/app/simulation/projectors/dashboard.py` (new), `dashboard_aggregator.py` (modify)

1. **DashboardProjector** subscribes to all 6 streams (including `stream:transport-ops` and `stream:transit-events`), maintains `read:dashboard:platform` Redis hash via incremental counter updates (design doc Section 6.2.1)
2. Includes lazy cage/demurrage computation via `compute_cage_metrics()` pure function (Section 11.3)
3. **Periodic reconciliation**: Keep `aggregate_dashboard()` as 5-minute safety net
4. **Startup seeding**: On fresh Redis, seed from SQL

**API endpoint migration:**

| Endpoint | Before | After |
|----------|--------|-------|
| `GET /api/platform/aggregates` | 12-15 SQL queries | `HGETALL read:dashboard:platform` |
| SSE `/api/simulation/events` | Dashboard aggregator → PostgreSQL | Dashboard projector → Redis hash |
| `GET /api/shipments` | PostgreSQL SELECT | Redis sorted sets + hashes |
| `GET /api/broker/queue` | PostgreSQL JOIN | Redis sorted set + hashes (filtered by broker) |
| `GET /api/shipments/{id}` | PostgreSQL SELECT | Redis cache → PostgreSQL on miss (30s TTL) |

### 5.2 Convert Remaining Actors

| Actor | Change |
|-------|--------|
| **demurrage** | Convert to lazy computation model — event-triggered parameter storage on `ShipmentArrivedAtCustoms` (ocean filter), no timer needed |
| **exceptions** | Reclassified to hybrid — subscribe to `ShipmentInTransit`/`ShipmentArrivedAtCustoms` for shipment discovery. Timer-based probabilistic injection. Publishes `ExceptionInjected`. |
| **disruption** | Reclassified to hybrid — subscribe to lifecycle events for port-to-shipment mapping. Timer-based disruption generation. Publishes `DisruptionGenerated`. |

### 5.3 Build TestEventBus

**File:** `clearance-engine/backend/tests/conftest.py`

In-memory event bus for unit and integration tests (no Redis needed):

```python
class TestEventBus:
    def __init__(self):
        self.streams: dict[str, list] = defaultdict(list)
        self.consumer_positions: dict[str, int] = defaultdict(int)
        self.published_events: list[dict] = []

    async def publish(self, stream, event_type, payload): ...
    async def read_stream(self, stream, group, consumer, count=10, block_ms=0): ...
    async def ack(self, stream, group, event_id): ...
```

### 5.4 Decommission Old Infrastructure + Aggregator Sole Writer Cutover

- **Aggregator becomes sole writer**: Remove event appends from `transition()`. Remove atomic JSONB append code. All shipment table writes now flow through the Aggregator exclusively.
- `transition()` reduced to `validate_transition()` only — a pure guard function used by actors before emitting events
- Remove old `EventBus` class (Redis Pub/Sub, 38 publishes, 0 subscribers)
- Remove `FOR UPDATE SKIP LOCKED` from all pure event-driven actors
- Remove per-tick batch `SELECT` queries from migrated actors
- Hybrid actors retain sim-clock polling for completion checks only (no `FOR UPDATE`)
- Verify all 197 existing tests still pass
- Add event-driven integration tests (Layer 2 and Layer 3)

### Phase 3 Success Criteria

- [ ] All 19 actors migrated (12 event-driven, 5 hybrid, 2 timer-driven)
- [ ] **Shipment Aggregator is sole writer to `shipments` table** — no actor writes directly
- [ ] `transition()` replaced by `validate_transition()` (pure guard) — no event append, no status write
- [ ] Atomic JSONB append code removed (transitional safety net no longer needed)
- [ ] Dashboard served from Redis — zero SQL per request
- [ ] Shipment list served from Redis sorted sets
- [ ] Old `EventBus` (Pub/Sub) decommissioned
- [ ] `TestEventBus` enables integration tests without Redis
- [ ] No `FOR UPDATE` in any event-driven actor
- [ ] All 197+ tests pass plus new event-driven tests
- [ ] Zero `at_customs` contention in health check metrics

---

## 6. File Change Matrix

### Phase 0 (Event Bus Infrastructure)

| File | Change |
|------|--------|
| `backend/app/main.py` | Dual engine creation, pool separation |
| `backend/app/simulation/coordinator.py` | Accept `actor_session_factory`, create `EventStreamBus` |
| `backend/app/simulation/event_bus.py` | Add `EventStreamBus` class alongside `EventBus` |
| `backend/app/simulation/state_machine.py` | Atomic JSONB append in `transition()` |
| `backend/app/simulation/actors/base.py` | Add `EventDrivenActor` base class |
| `backend/app/simulation/events.py` | **NEW** — `ShipmentEventPayload` |
| `backend/app/simulation/actors/outcomes.py` | **NEW** — `ProcessingOutcome`, `apply_outcome()` |
| `backend/app/simulation/actors/proactive.py` | **NEW** — `ProactiveEvaluatorMixin` with readiness accumulator |
| `backend/app/simulation/actors/preclearance.py` | Add `ClassificationCompleted` event publish after screening |
| `backend/app/simulation/state_machines/us.py` | Add `in_transit → cleared` transition (`{"customs"}`) |
| `backend/app/simulation/state_machines/eu.py` | Add equivalent pre-clearance transition |
| `backend/app/simulation/state_machines/br.py` | Add equivalent pre-clearance transition |

**Files modified:** 8 existing + 3 new = 11 total

### Phase 1 (Highest-Value Actor Migration)

| File | Change | Sub-phase |
|------|--------|-----------|
| `backend/app/simulation/actors/customs.py` | Extract pure functions, migrate to `EventDrivenActor` | 1a |
| `backend/app/simulation/actors/compliance_engine.py` | Migrate to `EventDrivenActor` | 1a |
| `backend/app/simulation/actors/shipper_response.py` | Migrate to `EventDrivenActor` | 1a |
| `backend/app/simulation/actors/financial.py` | Migrate to `EventDrivenActor` | 1a |
| `backend/app/simulation/actors/documents.py` | Migrate to `EventDrivenActor` | 1a |
| `backend/app/simulation/actors/shipper.py` | Add `ShipmentCreated` event publish | 1a |
| `backend/app/simulation/actors/carrier.py` | Add event publishing after status transitions (not yet migrated) | 1a |
| `backend/app/simulation/actors/pga.py` | Migrate to hybrid `EventDrivenActor` | 1b |
| `backend/app/simulation/actors/resolution.py` | Migrate to hybrid `EventDrivenActor` | 1b |
| `backend/app/simulation/actors/cage.py` | Migrate to hybrid `EventDrivenActor` (lazy dwell) | 1b |
| `backend/app/simulation/actors/terminal.py` | Migrate to `EventDrivenActor` | 1b |
| `backend/app/simulation/actors/preclearance.py` | Migrate to `EventDrivenActor` | 1b |
| `backend/app/simulation/actors/isf.py` | Migrate to `EventDrivenActor` | 1b |

**Files modified:** 13 existing

### Phase 2 (Complex Actors + Read Models + Aggregator)

| File | Change | Sub-phase |
|------|--------|-----------|
| `backend/app/simulation/actors/broker_sim.py` | Decompose into 3 handlers | 2a |
| `backend/app/simulation/actors/cbp_authority.py` | Migrate to `EventDrivenActor` (4 subscriptions) | 2a |
| `backend/app/simulation/actors/carrier.py` | Migrate to hybrid `EventDrivenActor` (full rework) + publish mode-specific events to `stream:transport-ops` + transit events to `stream:transit-events` | 2a |
| `backend/app/simulation/actors/consolidator.py` | Migrate to hybrid (buffered consumer) | 2a |
| `backend/app/simulation/state_machine.py` | Split `transition()` into `validate_transition()` + keep old `transition()` for non-migrated paths | 2a |
| `backend/app/simulation/projectors/shipment_aggregator.py` | **NEW** — Shipment Aggregator: sole writer to `shipments` table, consumes all 6 streams | 2b |
| `backend/app/simulation/projectors/dashboard.py` | **NEW** — `DashboardProjector` | 2b |
| `backend/app/simulation/dashboard_aggregator.py` | Reduce to 5-minute reconciliation only | 2b |
| `backend/app/api/routes/dashboard.py` | Read from Redis instead of aggregator | 2b |
| `backend/app/api/routes/broker.py` | Shipment list from Redis sorted sets | 2b |

**Files modified:** 6 existing + 2 new = 8 total

### Phase 3 (Cleanup + Polish + Aggregator Sole Writer Cutover)

| File | Change |
|------|--------|
| `backend/app/simulation/actors/demurrage.py` | Convert to lazy computation model |
| `backend/app/simulation/actors/exceptions.py` | Reclassify to hybrid (event discovery + timer injection) |
| `backend/app/simulation/actors/disruption.py` | Reclassify to hybrid (event discovery + timer injection) |
| `backend/app/simulation/state_machine.py` | Remove old `transition()` and atomic append code; retain only `validate_transition()` |
| `backend/app/simulation/event_bus.py` | Remove old `EventBus` class |
| `backend/tests/conftest.py` | Add `TestEventBus` fixture |

**Files modified:** 6

### Total Files Changed

| Phase | Existing Modified | New Files | Total |
|-------|------------------|-----------|-------|
| Phase 0 | 8 | 3 | 11 |
| Phase 1 | 13 | 0 | 13 |
| Phase 2 | 6 | 2 | 8 |
| Phase 3 | 6 | 0 | 6 |
| **Total** | **33** | **5** | **38** |

---

## 7. Effort Summary

| Phase | Effort | Cumulative | Key Deliverable |
|-------|--------|-----------|-----------------|
| Phase 0: Infrastructure + Systemic Fixes | 2-3 days | 2-3 days | EventStreamBus + EventDrivenActor + ProcessingOutcome + JSONB fix + ProactiveEvaluatorMixin + ClassificationCompleted + state machine transitions |
| Phase 1a: Clean Conversions (highest contention) | 2 days | 4-5 days | 5 actors migrated + shipper event publishing |
| Phase 1b: Moderate Adaptation | 2 days | 6-7 days | 6 more actors migrated (3 hybrid + 3 clean) |
| Phase 2a: Complex Actor Rework | 2 days | 8-9 days | broker_sim, cbp_authority, carrier, consolidator |
| Phase 2b: Shipment Aggregator + Read Models | 2 days | 10-11 days | Shipment Aggregator (sole writer) + Dashboard from Redis |
| Phase 3: Cleanup + Aggregator Sole Writer Cutover | 1-2 days | 11-12 days | demurrage, exceptions, disruption, Aggregator sole writer cutover, TestEventBus, decommission |

**Total: 11-12 days** (aligned with design document estimate)

**Critical path:** Customs actor refactoring (Phase 1a, 1-2 days) establishes the ProcessingOutcome pattern. BrokerSim decomposition (Phase 2a, 1-2 days) and carrier hybrid conversion (Phase 2a, 1-2 days) are the other high-complexity items. These three actors dictate the timeline.

---

## 8. Rollback Plan

### Phase 0 Rollback
- Revert `main.py` to single engine
- `EventStreamBus`, `EventDrivenActor`, `ProcessingOutcome` are additive — can be left in place
- JSONB atomic append is backwards-compatible — no rollback needed
- **Time to rollback:** 10 minutes (git revert `main.py`)

### Phase 1 Rollback (per-actor)
- Each actor has the old poll-based `tick()` in git history
- Revert individual actor files to restore poll-based behavior
- Dashboard falls back to SQL aggregator
- Consumer groups persist in Redis but cause no harm if unused
- **Time to rollback:** 5 minutes per actor (git checkout of specific files)

### Phase 2 Rollback
- **Phase 2a (complex actors):** Carrier and broker_sim are the highest-risk reverts. Revert individual actor files from git history.
- **Phase 2b (read models):** Projectors are additive. Dashboard/shipment list fall back to PostgreSQL queries. Redis read stores are cache — stale data is harmless.
- **Time to rollback:** 15 minutes (git revert)

### Phase 3 Rollback
- Remaining actors (demurrage, exceptions, disruption) revert to poll-based
- Old `EventBus` can be restored from git
- **Time to rollback:** 10 minutes (git revert Phase 3 commit)

---

## 9. Testing Strategy

### Preserved Tests (197 existing)

All existing unit tests test per-shipment business logic directly. These methods are preserved as pure functions (via `ProcessingOutcome` extraction) — the tests continue to pass unchanged.

### New Tests (Per Phase)

| Phase | New Test Type | Count (estimated) |
|-------|--------------|-------------------|
| Phase 0 | `EventStreamBus` publish/read/ack tests | 5-10 |
| Phase 0 | `ProcessingOutcome` + `apply_outcome` tests | 5-10 |
| Phase 0 | `ProactiveEvaluatorMixin` readiness accumulator tests (partial data, `_is_ready()` trigger, `_cleanup()`) | 5-8 |
| Phase 0 | `ClassificationCompleted` event emission from preclearance actor | 2-3 |
| Phase 0 | State machine `in_transit → cleared` transition tests (per jurisdiction) | 3-5 |
| Phase 0 | Atomic JSONB append tests | 2-3 |
| Phase 1a | Event handler tests per clean actor (mock DB) | 10-15 |
| Phase 1a | Proactive evaluator readiness tests: verify customs/financial/documents act at earliest data-ready moment, not at status gate | 5-8 |
| Phase 1a | Shadow mode comparison (customs) | 1-2 |
| Phase 1b | Hybrid actor tests (event + tick) for pga, resolution, cage | 10-15 |
| Phase 1b | Proactive broker claim handler — verifies assignment during `booked`/`in_transit` | 3-5 |
| Phase 1b | Proactive ISF — verifies filing triggers on `ShipmentCreated` for ocean mode | 2-3 |
| Phase 2a | BrokerSim decomposition tests | 8-12 |
| Phase 2a | Carrier hybrid actor tests | 5-8 |
| Phase 2b | Shipment Aggregator projection tests (all column types) | 8-12 |
| Phase 2b | Aggregator shadow mode validation tests | 3-5 |
| Phase 2b | Dashboard projector incremental update tests | 5-10 |
| Phase 3 | Integration tests with TestEventBus (end-to-end event chains) | 10-15 |
| **Total** | | **97-157 new tests** |

### TestEventBus for Integration Tests

In-memory event bus enables integration tests that verify event chains without Redis:

```python
async def test_shipment_flows_through_customs_pipeline():
    bus = TestEventBus()
    customs = CustomsActor(event_bus=bus, ...)
    carrier = CarrierActor(event_bus=bus, ...)

    await bus.publish("stream:shipment-lifecycle", "ShipmentCreated", {...})
    await carrier.process_pending_events()
    await customs.process_pending_events()

    assert bus.published_events[-1]["event_type"] == "ShipmentCleared"


async def test_proactive_customs_pre_clears_during_transit():
    """Verify proactive evaluator acts before physical arrival."""
    bus = TestEventBus()
    customs = CustomsActor(event_bus=bus, ...)

    # Ship created — partial readiness (missing hs_code)
    await bus.publish("stream:shipment-lifecycle", "ShipmentCreated",
        {"id": "s1", "origin": "CN", "destination": "US", "declared_value": 5000, "company_name": "Acme"})
    await customs.process_pending_events()
    assert not customs._is_ready(customs._readiness.get("s1", {}))

    # Classification completes — readiness met, pre-files while still in_transit
    await bus.publish("stream:compliance", "ClassificationCompleted",
        {"id": "s1", "hs_code": "8471.30"})
    await customs.process_pending_events()
    assert any(e["event_type"] == "DeclarationFiled" for e in bus.published_events)
```

---

## 10. Observability

### Structured Logging

All event processing includes correlation IDs:

```python
logger.info("event_processed",
    actor=self.actor_id,
    event_type=event["event_type"],
    event_id=event["event_id"],
    correlation_id=event["correlation_id"],
    processing_ms=elapsed_ms,
)
```

### Health Check Extension

```python
@app.get("/health")
async def health_check(request: Request):
    status = {"api": "ok", ...}
    for stream in STREAM_CONFIG:
        for group in await redis.xinfo_groups(stream):
            pending = group["pending"]
            status[f"stream:{stream}:{group['name']}"] = "ok" if pending < 50 else f"behind:{pending}"
        # Dead letter count
        try:
            dl_len = await redis.xlen(f"{stream}:dead-letters")
            if dl_len > 0:
                status[f"{stream}:dead-letters"] = dl_len
        except Exception:
            pass
    return status
```

### Debugging Workflow

**"Why didn't this shipment clear?"**
1. `grep correlation_id=<shipment-id>` in structured logs → see full event chain
2. If event is stuck: `XPENDING stream:customs-processing cg:customs-actor` → shows unacknowledged events with delivery count and idle time
3. Two steps. Clear chain of custody.

---

*This plan describes the migration FROM poll-and-lock TO event-driven architecture with Proactive Evaluator pattern and single-writer Aggregator, aligned with the design document (Section 15) and informed by the feasibility check's actor-by-actor analysis. The architecture spans 10 bounded contexts, ~106 event types, 6 Redis Streams, 36 consumer groups, and a Shipment Aggregator as sole writer to the `shipments` table. The **Proactive Evaluator principle** — 7 services (customs, broker_sim, compliance, financial, documents, PGA, ISF) evaluate readiness criteria and act at the earliest data-ready moment via `ProactiveEvaluatorMixin` readiness accumulators — eliminates the sequential pipeline anti-pattern where `at_customs` was a mandatory gate. The `in_transit → cleared` state machine transition enables pre-clearance success without physical arrival. `cbp_authority` remains a Reactive Responder (command/exception-driven adjudication). Phase 0 stands up event bus infrastructure (6 streams), ProactiveEvaluatorMixin, ClassificationCompleted event, state machine transitions, and transitional JSONB safety net. Phase 1a/1b migrates 11 actors (proactive evaluators + hybrid + clean). Phase 2a splits `transition()` into `validate_transition()` + event emission and reworks complex actors. Phase 2b deploys the Shipment Aggregator and DashboardProjector. Phase 3 converts remaining actors, cuts over to Aggregator as sole writer, builds TestEventBus, and decommissions old infrastructure. Total: 11-12 days, 38 files changed, 97-157 new tests.*
