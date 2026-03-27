# Simulation Internals -- Implementation Details

Deep implementation reference for the actor-based customs clearance simulation. Covers the coordinator lifecycle, clock mechanics, event bus Redis patterns, state machine transition guards, each actor's tick logic, and the configuration system.

Source files:
- `backend/app/simulation/coordinator.py`
- `backend/app/simulation/clock.py`
- `backend/app/simulation/event_bus.py`
- `backend/app/simulation/state_machine.py`
- `backend/app/simulation/config.py`
- `backend/app/simulation/actors/*.py`
- `backend/app/simulation/dashboard_aggregator.py`
- `backend/app/simulation/reference_data.py`
- `backend/app/api/routes/simulation.py`

---

## Table of Contents

1. [Coordinator Lifecycle](#1-coordinator-lifecycle)
2. [Clock Implementation](#2-clock-implementation)
3. [Event Bus Redis Patterns](#3-event-bus-redis-patterns)
4. [State Machine Transitions](#4-state-machine-transitions)
5. [Actor Implementations](#5-actor-implementations)
6. [Dashboard Aggregator](#6-dashboard-aggregator)
7. [Configuration System](#7-configuration-system)
8. [REST API Integration](#8-rest-api-integration)

---

## 1. Coordinator Lifecycle

### Initialisation

The coordinator is created during FastAPI lifespan startup in `app/main.py`:

```python
sim_config = SimulationConfig(
    time_ratio=settings.SIM_TIME_RATIO,
    random_seed=settings.SIM_RANDOM_SEED,
    auto_start=settings.SIM_AUTO_START,
)
coordinator = SimulationCoordinator(
    session_factory=async_session_factory,
    redis=redis_client,
    config=sim_config,
)
app.state.simulation = coordinator
```

If `SIM_AUTO_START` is True, `coordinator.start()` is called immediately.

### Internal State

```python
self.config: SimulationConfig
self.clock: SimulationClock
self.event_bus: EventBus
self.session_factory: sessionmaker
self.redis: aioredis client
self._actors: dict[str, BaseActor]      # Actor instances by ID
self._running: bool                      # Global running flag
self._dashboard_task: asyncio.Task       # Background dashboard broadcast
self._sse_subscribers: list[asyncio.Queue]  # SSE subscriber queues
```

### Start Implementation

```python
async def start(self):
    if self._running:
        return

    await self.event_bus.start()

    # Create actors with offset seeds for reproducibility
    self._actors = {
        "shipper":       ShipperActor(clock, event_bus, session_factory, config.shipper, seed=seed),
        "preclearance":  PreClearanceAdapter(..., seed=seed+6),
        "carrier":       CarrierActor(..., seed=seed+1),
        "customs":       CustomsActor(..., seed=seed+2),
        "pga":           PGAActor(..., seed=seed+3),
        "compliance":    ComplianceEngineActor(..., seed=seed+4),
        "resolution":    ResolutionActor(..., seed=seed+5),
    }

    for actor in self._actors.values():
        actor.start()  # Launches asyncio task

    self._dashboard_task = asyncio.create_task(self._dashboard_loop())
    self._running = True
```

### Stop Implementation

```python
async def stop(self):
    for actor in self._actors.values():
        await actor.stop()  # Cancels task, awaits completion

    if self._dashboard_task:
        self._dashboard_task.cancel()
        await self._dashboard_task  # Absorb CancelledError

    await self.event_bus.stop()
    self._running = False
```

### Reset Implementation

```python
async def reset(self):
    await self.stop()

    # Truncate all shipments
    async with self.session_factory() as session:
        await session.execute(delete(Shipment))
        await session.commit()

    # Re-seed demo data
    async with self.session_factory() as session:
        await _seed_shipments(session)

    self.clock.reset()

    # Clear Redis dashboard cache
    await self.redis.delete("sim:dashboard:latest")
```

### Dashboard Broadcast Loop

```python
async def _dashboard_loop(self):
    while True:
        await asyncio.sleep(self.config.dashboard_interval_seconds)  # default 3s
        if not self._running or self.clock.paused:
            continue

        data = await get_cached_or_compute(self.session_factory, self.redis, self.clock)

        dead_queues = []
        for queue in self._sse_subscribers:
            try:
                queue.put_nowait(data)
            except asyncio.QueueFull:
                try:
                    queue.get_nowait()    # Drop oldest
                    queue.put_nowait(data)
                except:
                    dead_queues.append(queue)

        for q in dead_queues:
            self.unsubscribe_sse(q)
```

### Actor Config Hot-Update

```python
def update_actor_config(self, actor_id: str, updates: dict) -> bool:
    actor = self._actors.get(actor_id)
    if not actor:
        return False

    current = actor.config.model_dump()
    current.update(updates)
    new_config = actor.config.__class__(**current)
    actor.update_config(new_config)
    return True
```

---

## 2. Clock Implementation

### Core Mechanics

The clock uses `time.monotonic()` for drift-free real-time measurement:

```python
class SimulationClock:
    def __init__(self, time_ratio=10.0, seed_time=None):
        self._time_ratio = time_ratio
        self._start_real = time.monotonic()
        self._start_sim = seed_time or datetime.now(timezone.utc)
        self._paused = False
        self._pause_real = 0.0
        self._accumulated_pause = 0.0
```

### Time Calculation

```python
def now(self) -> datetime:
    if self._paused:
        real_elapsed = self._pause_real - self._start_real - self._accumulated_pause
    else:
        real_elapsed = time.monotonic() - self._start_real - self._accumulated_pause

    sim_minutes = real_elapsed * self._time_ratio
    return self._start_sim + timedelta(minutes=sim_minutes)
```

### Time Ratio Semantics

| time_ratio | Real 1 second = | Real 1 minute = | Real 1 hour = |
|---|---|---|---|
| 1.0 | 1 sim minute | 1 sim hour | 2.5 sim days |
| 10.0 (default) | 10 sim minutes | 10 sim hours | 25 sim days |
| 100.0 | 100 sim minutes | ~1.7 sim days | ~69 sim days |

### Sleep Conversion

```python
async def sleep_sim_minutes(self, sim_minutes: float):
    real_seconds = sim_minutes / self._time_ratio
    await asyncio.sleep(max(real_seconds, 0.05))  # Floor at 50ms
```

At default ratio (10.0), a 60 sim-minute sleep = 6 real seconds. A 240 sim-minute sleep = 24 real seconds.

### Business Hours Logic

```python
def is_business_hours(self) -> bool:
    return 8 <= self.now().hour < 18
```

Used by ShipperActor to modulate shipment creation rate.

---

## 3. Event Bus Redis Patterns

### Channel

Single Redis pub/sub channel: `sim:events`

### Publish Pattern

```python
async def publish(self, event_type: str, data: dict):
    message = {"event_type": event_type, **data}

    if self._redis:
        try:
            await self._redis.publish(self.CHANNEL, json.dumps(message, default=str))
            return
        except Exception:
            pass

    # Fallback: dispatch in-memory
    await self._dispatch(message)
```

### Subscribe Pattern

```python
def subscribe(self, event_type: str, handler: EventHandler):
    self._subscribers.setdefault(event_type, []).append(handler)
```

Handler type: `Callable[[dict[str, Any]], Coroutine[Any, Any, None]]`

### Listener Task

```python
async def _listen(self):
    async for message in self._pubsub.listen():
        if message["type"] == "message":
            data = json.loads(message["data"])
            await self._dispatch(data)
```

### Dispatch

```python
async def _dispatch(self, message: dict):
    event_type = message.get("event_type", "")
    handlers = self._subscribers.get(event_type, [])
    for handler in handlers:
        await handler(message)  # Errors logged but not propagated
```

### Fallback Behaviour

When Redis is unavailable:
- `start()` logs a warning and continues
- `publish()` catches the Redis error and calls `_dispatch()` directly
- The system operates fully, just without cross-process event distribution

---

## 4. State Machine Transitions

### Complete Transition Graph

```
booked ----[platform]----> in_transit
booked ----[platform]----> held

in_transit --[carrier]---> at_customs
in_transit --[platform]--> held

at_customs --[customs]---> cleared
at_customs --[customs]---> inspection
at_customs --[customs]---> held
at_customs --[pga]-------> held

inspection --[customs]---> cleared
inspection --[customs]---> held

held -------[resolution]-> cleared
held -------[resolution]-> at_customs

cleared ----[carrier]----> delivered

delivered: TERMINAL (no outgoing transitions)
```

### Guard Validation

```python
def can_transition(from_status, to_status, actor) -> bool:
    allowed = TRANSITIONS.get(from_status, {}).get(to_status)
    return allowed is not None and actor in allowed
```

### TransitionError

Raised when:
1. `from_status` is terminal (`delivered`)
2. The transition `from -> to` is not in the table
3. The actor is not in the allowed set

```python
raise TransitionError(
    f"Transition '{from_status}' -> '{to_status}' not allowed for actor '{actor}'. "
    f"Allowed: {TRANSITIONS.get(from_status, {})}"
)
```

### Event Logging

Every transition appends an event:

```python
{
    "timestamp": sim_time.isoformat(),
    "event_type": f"status_{to_status}",  # e.g., "status_cleared"
    "description": description,
    "actor": actor,
}
```

`flag_modified(shipment, "events")` ensures SQLAlchemy flushes the JSONB change.

---

## 5. Actor Implementations

### 5.1 BaseActor Run Loop

Every actor inherits this loop:

```python
async def _run_loop(self):
    while self._running:
        if self.clock.paused:
            await asyncio.sleep(0.5)
            continue

        try:
            await self.tick()
            self._tick_count += 1
        except Exception as exc:
            logger.error("actor_tick_error", ...)

        tick_minutes = self._get_tick_minutes()
        await self.clock.sleep_sim_minutes(tick_minutes)
```

### 5.2 ShipperActor Tick Logic

```
1. Calculate rate_per_tick = (base_rate_per_day / ticks_per_day) * time_multiplier
   - ticks_per_day = (24 * 60) / tick_sim_minutes = 24
   - rate_per_tick = 30 / 24 = 1.25
   - business hours: 1.25 * 2.5 = 3.125
   - overnight: 1.25 * 0.3 = 0.375

2. n_shipments = Poisson(rate_per_tick)

3. For each shipment:
   a. Pick corridor (weighted): CN->US (40%), MX->US (15%), DE->US (10%), ...
   b. Pick product from reference data (random choice)
   c. Declared value = uniform(product.value_range)
   d. Misclassification (7%): mutate 2 random digit positions in HS code
   e. Value error (3%): multiply by uniform(0.85, 1.25)
   f. Carrier selection: mode-weighted (air 60%, ground 20%, ocean 20%)
   g. Company name: random from COMPANY_NAMES list
   h. Create Shipment ORM object with status="booked"
   i. Publish "shipment_created" event
```

### 5.3 PreClearanceAdapter Tick Logic

```
1. SELECT up to 50 shipments WHERE status = 'booked' FOR UPDATE SKIP LOCKED
2. For each:
   a. Instantiate PreClearanceAgent(llm_service=None)  # deterministic mode
   b. agent.execute(entity_name, origin, dest, hs_code, product, value, transport_mode)
   c. On success:
      - CLEAR -> transition booked -> in_transit (actor: "platform")
      - HOLD -> set hold_type, transition booked -> held (actor: "platform")
   d. On error: transition to in_transit (fail-open)
3. Commit
```

### 5.4 CarrierActor Tick Logic

**Phase 1: In-transit processing**
```
1. SELECT in_transit shipments FOR UPDATE SKIP LOCKED
2. For each:
   a. Look up corridor for origin
   b. Get transit days for carrier mode (air: 1-3d, ocean: 14-28d)
   c. Add random delay (8% chance, 6-48h)
   d. If elapsed >= required: transition to at_customs
   e. Update waypoint: import_port status = "current"
   f. Publish "shipment_arrived"
```

**Phase 2: Cleared processing**
```
1. SELECT cleared shipments FOR UPDATE SKIP LOCKED
2. For each:
   a. Find "status_cleared" event timestamp (scan events in reverse)
   b. Inland delivery = uniform(1, 3) days
   c. If elapsed >= required: transition to delivered
   d. Update waypoints
   e. Publish "shipment_delivered"
```

### 5.5 CustomsActor Tick Logic

**Queue management:**
```python
queue_count = await session.scalar(
    select(func.count(Shipment.id)).where(Shipment.status == "at_customs")
)
backlogged = queue_count > cfg.backlog_threshold  # default 20
```

**STP probability:**
```python
scrutiny = cfg.corridor_scrutiny.get(origin, 1.0)  # CN=1.3, VN=1.1, IN=1.1
stp_chance = cfg.stp_rate * (1.0 / scrutiny)
```

For CN origin: `0.87 * (1/1.3) = 0.669` (67% STP vs 87% base)

**Manual review cascade (single random roll):**
```python
roll = self.rng.random()

if origin == "CN" and roll < 0.02:          # UFLPA detention
elif roll < 0.02 + (0.04 * scrutiny):       # Physical inspection
elif roll < above + 0.05:                    # HS reclassification
elif roll < above + 0.038:                   # CF-28/CF-29 hold
else:                                         # Cleared via manual review
```

**HS Reclassification:**
```python
def _reclassify_hs(self, shipment):
    original_hs = codes.get("final_hs_code", "")
    new_suffix = f"{self.rng.randint(0, 99):02d}"
    new_hs = original_hs[:-2] + new_suffix

    codes["final_hs_code"] = new_hs
    codes["reclassified"] = True

    factor = self.rng.uniform(0.8, 1.3)
    financials["predicted_duty"] = predicted_duty * factor
```

### 5.6 PGAActor Tick Logic

**Agency determination (three sources, priority order):**

1. `shipment.codes["pga_jurisdictions"]` -- explicit list
2. `shipment.codes["pga_flags"]` -- explicit flags
3. Keyword matching against product name:
   - pharma/drug/vaccine/tablet -> FDA
   - food/tea/coffee/supplement -> FDA
   - titanium dioxide/polyurethane/chemical -> EPA
   - cotton/textile/apparel/baby -> CPSC
   - airbag/seatbelt/inflator -> NHTSA

**Review logic per agency:**
```python
review_days = self.rng.uniform(*review_days_range)
elapsed = sim_now - arrival_time

if elapsed < timedelta(days=review_days):
    continue  # Still under review

if self.rng.random() < approval_rate:
    # Append pga_approved event (don't change status)
else:
    shipment.hold_type = "pga"
    transition(shipment, "held", "pga", ...)
    break  # Stop reviewing other agencies
```

### 5.7 ComplianceEngineActor Tick Logic

```
1. SELECT shipments WHERE analysis IS NULL AND status != 'in_transit'
2. For each:
   a. Build classification section (HS code, confidence)
   b. Build tariff section (duty breakdown with 2-3 random programs)
   c. Build compliance section (10% chance of "REVIEW", 90% "CLEAR")
   d. If run_real_engines: try real engine modules, fallback to simulated
   e. Set shipment.analysis = analysis dict
```

### 5.8 ResolutionActor Tick Logic

```
1. SELECT held shipments FOR UPDATE SKIP LOCKED
2. For each:
   a. Find "status_held" event timestamp (scan events in reverse)
   b. review_days = uniform(1, 5)
   c. If elapsed < review_days: skip (still pending)
   d. Roll outcome:
      - roll < 0.75: transition held -> cleared, publish "shipment_resolved"
      - roll < 0.85: append "hold_escalated" event, publish "shipment_resolved" (escalated)
      - else: skip (still pending, re-evaluate next tick)
```

---

## 6. Dashboard Aggregator

### SQL Queries

**Status distribution:**
```sql
SELECT status, COUNT(id) FROM shipments GROUP BY status
```

**Financial aggregates:**
```sql
SELECT COALESCE(SUM(declared_value), 0.0) FROM shipments
SELECT COALESCE(SUM(CAST(financials->>'predicted_duty' AS FLOAT)), 0.0) FROM shipments
```

**Corridor metrics (top 10):**
```sql
SELECT origin, destination, COUNT(id) AS volume
FROM shipments
GROUP BY origin, destination
ORDER BY COUNT(id) DESC
LIMIT 10
```

**Per-corridor hold rate:**
```sql
SELECT COUNT(id) FROM shipments
WHERE origin = :origin AND destination = :dest AND status = 'held'
```

### Caching Strategy

```python
async def get_cached_or_compute(session_factory, redis, clock, cache_key="sim:dashboard:latest", ttl=5):
    # Check Redis
    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)

    # Compute fresh
    async with session_factory() as session:
        data = await aggregate_dashboard(session, clock)

    # Store in Redis with 5-second TTL
    await redis.set(cache_key, json.dumps(data, default=str), ex=ttl)
    return data
```

### Derived Metrics

- `clearanceRate` = (cleared + delivered) / total * 100
- `holdRate` = held / total * 100
- `stpRate` = cleared / (cleared + held + inspection) * 100
- `ftaSavingsMonth` = total_duty * 0.032 (3.2% proxy)
- Duty by program: MFN Base 37%, IEEPA 31%, Section 301 17%, Section 232 7%, AD/CVD 4%, Fees 4%

---

## 7. Configuration System

### Environment Variables

| Variable | Default | Description |
|---|---|---|
| `SIM_TIME_RATIO` | 10.0 | Sim minutes per real second |
| `SIM_RANDOM_SEED` | None | Seed for reproducible runs |
| `SIM_AUTO_START` | False | Start simulation on app startup |

### Config Hierarchy

```
SimulationConfig
    time_ratio: 10.0
    random_seed: None
    auto_start: False
    dashboard_interval_seconds: 3.0
    shipper: ShipperConfig
    preclearance: PreClearanceConfig
    carrier: CarrierConfig
    customs: CustomsConfig
    pga: PGAConfig
    compliance: ComplianceConfig
    resolution: ResolutionConfig
```

### Hot-Update via API

```bash
curl -X PUT http://localhost:8000/api/simulation/actors/customs \
  -H "Content-Type: application/json" \
  -d '{
    "stp_rate": 0.95,
    "hold_rate": 0.01
  }'
```

The update merges into the existing config:

```python
current = actor.config.model_dump()  # Full current config as dict
current.update(updates)              # Merge new values
new_config = actor.config.__class__(**current)  # Validate through Pydantic
actor.update_config(new_config)
```

---

## 8. REST API Integration

### Route Registration

```python
router = APIRouter(prefix="/api/simulation", tags=["simulation"])
```

### Coordinator Access

Every route handler retrieves the coordinator from app state:

```python
def _get_coordinator(request: Request):
    coordinator = getattr(request.app.state, "simulation", None)
    if coordinator is None:
        raise HTTPException(status_code=503, detail="Simulation coordinator not initialized")
    return coordinator
```

### SSE Dashboard Stream

```python
@router.get("/dashboard/stream")
async def dashboard_stream(request: Request):
    coordinator = _get_coordinator(request)
    queue = coordinator.subscribe_sse()

    async def event_generator():
        try:
            while True:
                try:
                    data = await asyncio.wait_for(queue.get(), timeout=30.0)
                    yield f"event: dashboard\ndata: {json.dumps(data, default=str)}\n\n"
                except asyncio.TimeoutError:
                    yield ": keepalive\n\n"
        finally:
            coordinator.unsubscribe_sse(queue)

    return StreamingResponse(event_generator(), media_type="text/event-stream", ...)
```

### Concurrency Safety

All actors use `SELECT ... FOR UPDATE SKIP LOCKED` to prevent:
- Two actors processing the same shipment simultaneously
- Lock contention between actors (each skips locked rows)
- Deadlocks (no blocking waits)
