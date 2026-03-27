# Architecture Trade-Off Analysis: Making the Event-Driven Design Excellent

**Author:** design-challenger (architecture reviewer)
**Status:** Final — Proactive Evaluator principle adopted (Sections 5.16-5.17), single-writer Aggregator, 10 contexts / ~106 events / 6 streams
**Date:** 2026-02-08
**Inputs:** `architecture-event-driven-design.md` (event-architect), `architecture-feasibility-check.md` (codebase-analyst)

---

## 1. Executive Summary

The Clearance Vibe platform is migrating to an event-driven architecture (EDA) using Redis Streams and the state-transfer pattern. This document identifies **implementation challenges, design trade-offs, and architectural risks** that must be addressed during migration.

The feasibility check (`docs/architecture-feasibility-check.md`) validates the design: **12 of 19 actors convert cleanly**, 5 need adaptation, and 2 require significant rework. No showstoppers. The event-driven design solves both the performance crisis (7-60s API response times) and the correctness problem (silent SKIP LOCKED processing gaps).

**Seven implementation challenges** need concrete answers (plus the proactive evaluator principle — Sections 5.16-5.17):

1. **JSONB `events` shared mutable column** — eliminated in target architecture by Shipment Aggregator (single-writer projection); atomic append as transitional safety net during migration
2. **Time-dependent actor re-evaluation** — 7 actors with sim-clock-dependent logic need persisted durations and lazy computation
3. **Customs actor cross-shipment dependency** — backlog-aware STP logic needs a shared counter from the read model
4. **Pure function extraction** — 415+ lines of customs logic need structural refactoring via the ProcessingOutcome pattern
5. **BrokerSimActor decomposition** — 5 methods, 4-5 tables, self-referential pipeline must be split into 3 focused handlers
6. **Event payload sizing** — bounded ShipmentEventPayload schema excludes large fields (events history, analysis)
7. **Schedule risk** — 11-12 days realistic; customs + broker_sim refactoring is the critical path

---

## 2. Why Event-Driven Is the Right Direction

### 2.1 The Correctness Argument (Not Just Performance)

The current poll-and-lock architecture has a **silent processing gap problem** that no amount of LIMIT tuning fixes.

**Today (unbounded FOR UPDATE SKIP LOCKED):**
- Customs locks 50 `at_customs` rows → PGA skips all 50
- PGA gets those 50 rows on the next tick when customs releases

**The fundamental problem with SKIP LOCKED:**
- SKIP LOCKED is non-deterministic — a shipment can sit in `at_customs` for multiple ticks while different actors keep skipping it
- There's no guarantee every shipment is processed by every actor that needs to see it
- You'd never know unless you manually check

**With event-driven consumer groups:**
- `ShipmentArrivedAtCustoms` is delivered to BOTH the customs consumer group AND the PGA consumer group
- Each actor sees every shipment. Nothing is silently skipped.
- If an actor fails to process, the event stays in XPENDING — visible, auditable, retryable.

This is a **correctness improvement**, not just a performance improvement.

### 2.2 Contention Prevention by Construction

With the poll-and-lock model, the codebase has 19 actors with 35 `FOR UPDATE SKIP LOCKED` queries. Every new actor, every new jurisdiction, every new status type risks reintroducing the contention bug if the developer doesn't know the conventions.

With event-driven, the architecture **prevents contention by construction**. A new actor subscribes to events. There is no `FOR UPDATE`. There is no SKIP LOCKED. The correct pattern is the only pattern available.

### 2.3 The Dashboard Query Problem

The dashboard aggregator runs 12-15 SQL queries per SSE cycle. A dashboard read model projection eliminates ALL of this — every event increments/decrements counters in a Redis hash. The API reads one hash. Zero SQL queries.

### 2.4 Feasibility-Confirmed Benefits

The feasibility check identified concrete wins beyond the architecture's stated goals:

- **compliance_engine race condition fixed** — current code has no FOR UPDATE, so two ticks can both find `analysis IS NULL` and generate analysis. Consumer group guarantees single processing.
- **shipper_response full table scan eliminated** — currently scans ALL shipments with no WHERE clause. Event-driven processes only relevant events.
- **Enrichment actor trigger model clarified** — 4 actors (compliance, financial, documents, terminal) enrich without changing status. They consume status events and produce domain-specific events for the dashboard projector.

---

## 3. Feasibility Data: The 12/5/2 Split

The codebase-analyst's line-by-line analysis of all 19 actors produced a concrete conversion difficulty assessment:

### 3.1 Clean Conversions (12 actors)

| Actor | Key Finding |
|-------|------------|
| shipper | Pure producer, timer-driven — publishes `ShipmentCreated` after INSERT |
| preclearance | Already has BATCH_LIMIT. Event-driven is batch of 1. |
| customs | Backlog detection via read model (minor). Clean otherwise. |
| compliance_engine | Event-driven **fixes** existing race condition |
| shipper_response | Event-driven **eliminates** full table scan |
| terminal | Mode filter in handler guard. Trivial. |
| financial | Guard-flag idempotency already works |
| documents | Guard-flag idempotency already works |
| isf | Mode filter for ocean shipments. Trivial. |
| disruption | Timer-based injector, reclassified to hybrid for shipment discovery |
| exceptions | Timer-based injector, reclassified to hybrid for shipment discovery |
| preclearance | Clean, no issues |

### 3.2 Adaptation Needed (5 actors)

| Actor | Adaptation Required |
|-------|-------------------|
| consolidator | Buffered consumer pattern — accumulates candidates, flushes when group threshold met |
| pga | Time-dependent review periods — persisted durations + hybrid poll completion |
| resolution | Time-dependent review windows — persisted durations + hybrid poll completion |
| cage | Dwell tracking → lazy computation; intake is event-driven, dwell/release is timer |
| demurrage | Recurring accumulation → lazy computation; stores arrival time + carrier policy |

### 3.3 Significant Rework (2 actors)

| Actor | Rework Required |
|-------|----------------|
| carrier | Most complex actor — time-dependent transit, intermediate regulatory events, consolidation side-effects, per-shipment commits. Hybrid pattern with persisted durations. |
| broker_sim | 5 methods, 4-5 tables, self-referential pipeline. Must decompose into 3 focused event handlers (BrokerClaimHandler, BrokerProgressHandler, BrokerCF28Handler). |

---

## 4. Implementation Challenges

### 4.1 Challenge: JSONB `events` Shared Mutable Column → Single-Writer Projection

**Original severity: HIGH — affects 12 of 19 actors**
**Target-state severity: ELIMINATED — single-writer Aggregator removes the problem by construction**

The `events` JSONB column is an append-only array. In the current architecture, every actor that calls `transition()` does a read-modify-write. With `FOR UPDATE SKIP LOCKED` removed in the event-driven model, 12 actors could concurrently append to the same shipment's `events` column — last-write-wins, events get lost.

**Stakeholder insight (accepted):** The shared `events` JSONB column is itself the anti-pattern. In a proper event-driven architecture:
- Events live in Redis Streams (the immutable event log)
- The `shipments.events` column is a **read projection** — a denormalized timeline aggregated from stream events
- A single **Shipment Aggregator** consumes all 6 streams and maintains `shipments.events` (and all other shipment columns) as projections
- No actor writes to `shipments.events` directly — they emit events to streams
- The race condition **disappears entirely** because there is only one writer

**Target architecture:**
1. Actors compute `ProcessingOutcome` (pure function) and **publish events to Redis Streams**
2. The Shipment Aggregator consumes events and applies them to the `shipments` record as projections
3. `transition()` is split: `validate_transition()` (pure guard function) + event emission. The Aggregator applies the status change.
4. The `events` timeline column is maintained exclusively by the Aggregator — single writer, no race

**Transitional safety net (Phase 0 through Phase 2a):**

During migration, some actors still call the old `transition()` which appends to `events`. The atomic SQL append is needed as a transitional fix:

```sql
UPDATE shipments SET events = COALESCE(events, '[]'::jsonb) || :event::jsonb WHERE id = :id
```

This atomic append is **removed in Phase 3** when the Aggregator becomes the sole writer.

**Migration timeline:**
- **Phase 0:** Atomic append added to `transition()` as safety net
- **Phase 1-2a:** Actors progressively stop calling `transition()` for event append; they emit events instead
- **Phase 2b:** Shipment Aggregator deployed, begins writing `events` column from stream consumption
- **Phase 3:** Remove event appends from `transition()` entirely. Aggregator is sole writer. Atomic append code removed.

**Trade-off:** The Shipment Aggregator becomes a single point of write responsibility for the `shipments` table. See Section 5.15 for the SPOF risk assessment.

### 4.2 Challenge: Time-Dependent Actors and the Simulation Clock

**Severity: MEDIUM — affects 7 actors**

Seven actors have time-dependent logic: carrier, resolution, PGA (completion), cage (dwell), demurrage, broker_sim (CF-28 response delay), cbp_authority (exam/CF-28 deadlines).

**Problem 1: Non-deterministic re-rolls.** Carrier and resolution use `self.rng.random()` to determine durations. On event re-delivery, the actor recalculates and gets a **different random duration**. Non-deterministic behavior.

**Solution: Persist computed durations on first evaluation.** Store `transit_days`, `review_days`, etc. in the shipment's `references` JSONB on first processing. On re-delivery, read the persisted value. The design document (Section 5.3) implements this correctly.

**Problem 2: Clock speed changes.** If `time_ratio` changes mid-simulation, real-time scheduled delays (e.g., Redis sorted set timers) become wrong.

**Solution: Hybrid pattern.** Event-triggered start + sim-clock polling for completion. `clock.now()` is called fresh on every tick — automatically handles ratio changes and pause/resume. No real-time sorted sets needed. The design document (Section 5.3, Category B actors) implements this correctly.

**Problem 3: Recurring accumulation (cage dwell, demurrage).** Currently recalculate storage costs and dwell days on every tick.

**Solution: Lazy computation.** Store `intake_time` and `storage_rate_per_day`. Compute dwell/cost at read time. Eliminates 2 of the 7 time-dependent actors from needing periodic updates entirely. The design document (Section 5.3) recommends this.

### 4.3 Challenge: Customs Actor Cross-Shipment Dependency

**The problem:** `_process_at_customs` runs `COUNT(*)` across all `at_customs` shipments for backlog detection. In the per-event model, this cross-shipment dependency doesn't naturally exist.

**Recommended solution:** Read from the shipment list read model. The shipment projector maintains `read:shipments:by_status:at_customs` as a sorted set. The customs actor reads the cardinality — zero DB queries, no cross-shipment dependency.

```python
at_customs_count = await self.redis.zcard("read:shipments:by_status:at_customs")
backlogged = at_customs_count > cfg.backlog_threshold
```

This aligns with the design document Section 11.5, which uses `ZCARD` on the sorted set (actual count of shipments in that status) rather than a separate dashboard counter.

**Trade-off:** Eventually consistent (1-2 second lag). Acceptable — backlog slowdown is a simulation realism feature, not a safety mechanism.

### 4.4 Challenge: Pure Function Extraction via ProcessingOutcome

**The problem:** The customs actor (415 lines in `_manual_review`) deeply interleaves business logic with ORM mutations. Six code paths (UFLPA, entity screening, ADD/CVD, inspection, HS reclassification, CF-28/29) all directly mutate ORM objects.

**Recommended solution:** `ProcessingOutcome` dataclass as the interface between business logic and ORM persistence. Business logic returns a `ProcessingOutcome`. A generic `apply_outcome()` handles ORM mutations and event publishing. Pattern established once for customs, then reused across all actors.

**Schedule impact:** This is the single largest refactoring task. Budget 1-2 days for customs alone.

### 4.5 Challenge: BrokerSimActor Decomposition

**The problem:** BrokerSimActor has 5 processing methods, writes to 4-5 tables, and has a self-referential pipeline where later methods read records created by earlier methods in the same tick.

**Recommended solution (from design doc Section 5.6):** Decompose into 3 focused event handlers:
1. **BrokerClaimHandler** — consumes `ShipmentArrivedAtCustoms`, creates assignment + entry filing (multi-table atomic)
2. **BrokerProgressHandler** — consumes `EntryDraftCreated` (internal), progresses checklists, submits entries
3. **BrokerCF28Handler** — consumes `CF28Issued`, generates response after delay (hybrid)

The implicit sequential pipeline becomes an explicit event chain: `ShipmentArrivedAtCustoms → claim → EntryDraftCreated → progress → EntryFiled`.

### 4.6 Challenge: Event Payload Sizing

**The problem:** Full state-transfer payloads would include the `events` JSONB (unbounded), `analysis` JSONB (large LLM text), and `cargo_detail` — potentially 5-20KB per event.

**Recommended solution:** Bounded `ShipmentEventPayload` dataclass (design doc Section 3.7). Includes all fields consumers need (~1-2KB). Excludes events history, analysis, cargo_detail. Consumers that need excluded fields fetch by primary key (~1ms single-row read).

**Trade-off:** This is a "fat reference" pattern, not pure state-transfer. No consumer needs to reconstruct the full shipment from the event alone. The design already accounts for this in Section 6.2.3 (write-through cache for shipment detail).

### 4.7 Challenge: Schedule Realism

**The problem:** The feasibility check reveals that broker_sim decomposition and carrier hybrid conversion are more complex than initially scoped.

**Agreed estimate:** 11-12 days across 4 phases (Phase 0 foundation, Phase 1a/1b actor migration, Phase 2a/2b complex actors + read models, Phase 3 cleanup). See the migration plan (`docs/architecture-migration-plan.md`) for detailed phase-by-phase breakdown, aligned with the design document Section 15.

**Critical path:** Customs refactoring (Phase 1a, 1-2 days) establishes the ProcessingOutcome pattern. BrokerSim decomposition (Phase 2a, 1-2 days) and carrier hybrid conversion (Phase 2a, 1-2 days) are the other high-complexity items.

---

## 5. Design Decisions Endorsed

### 5.1 Redis Streams Over Pub/Sub
Correct. The current EventBus has 38 publishes and zero subscribers. Redis Streams adds persistence, consumer groups, acknowledgment, and replay — all necessary.

### 5.2 State-Transfer Pattern (With Bounded Payload)
Correct with the bounded `ShipmentEventPayload` modification (Challenge 4.6).

### 5.3 Hybrid Actor Pattern for Time-Dependent Logic
Correct. Validated by feasibility check: 5 actors need hybrid (carrier, resolution, PGA completion, cage dwell, demurrage). Lazy computation eliminates 2 from needing periodic ticks (cage dwell, demurrage).

### 5.4 No Outbox Pattern (Write Path Reversed)
Correct. Single-process, single-instance. The write path is now **publish-then-project**: actors emit events to Redis Streams, the Shipment Aggregator projects state to PostgreSQL. If the Aggregator falls behind, events queue in Redis Streams (persistent, acknowledged). Periodic reconciliation (5-minute SQL recompute) and startup seeding are the safety nets. No outbox pattern needed.

### 5.5 Dead Letter Queues (Added)
Correct — added back per event-architect's revision. Simple: move to dead letter stream after 3 retries, alert on any dead letter. Prevents PEL accumulation from poison events. Low-cost operational hygiene.

### 5.6 No Fallback-to-Polling Dual Paths
Correct. Redis is already a hard dependency. Maintaining two actor models doubles test surface for zero benefit.

### 5.7 Idempotency via State Machine Guards
Correct. In the target architecture, `transition()` is split into `validate_transition()` (pure guard function — checks `can_transition(from_status, to_status)`) and event emission. The Shipment Aggregator applies the status change. Actors validate transitions before emitting events, and the Aggregator re-validates on application. Combined with `session.get()` status checks, this handles at-least-once delivery without a deduplication store.

### 5.8 Three-Layer Testing Strategy
Correct. Layer 1 (existing business logic tests, unchanged), Layer 2 (event handler tests with mocked DB), Layer 3 (integration tests with TestEventBus). Preserves 197 existing tests.

### 5.9 Exceptions/Disruption Reclassification to Hybrid
Correct — event-architect's revision. Both actors use `FOR UPDATE SKIP LOCKED` on `in_transit`/`at_customs` rows. Reclassified to hybrid: subscribe to events for shipment discovery (eliminates FOR UPDATE), timer-based for probabilistic injection.

### 5.10 Expanded Event Catalog (7 new events)
Correct — event-architect's revision added: `InspectionCompleted`, `CF28Responded`, `CF29Accepted`, `ProtestFiled`, `ProtestResolved`, 5 consolidation lifecycle events, `ExceptionInjected`, `DisruptionGenerated`, `CommunicationSent`. These fill gaps in the US state machine mapping.

### 5.11 Jurisdiction-Specific Event Catalog (~106 Event Types)
Correct. The fully reconciled event catalog now defines **~106 event types** across 10 bounded contexts:

- **Shipment lifecycle:** 5 events
- **Customs processing (jurisdiction-specific):** US CBP 11, EU UCC 6, BR Siscomex 4, IN ICEGATE 5, CN GACC 5, plus 4 shared cross-jurisdiction = ~35 events
- **Mode-specific transport operations:** Air 13, Ocean 13, Ground 9 = 35 events (see Section 5.14)
- **Intermediate transit:** 13 events (filing, screening, hub/transshipment)
- **Exception, cage & resolution:** 5 events
- **Compliance & enrichment:** 8 events
- **Consolidation lifecycle:** 5 events

Each jurisdiction's intermediate states (EU `UnderControl`, BR `Parameterized`, IN `OutOfCharge`, CN `CIQInspected`) are first-class events — not abstracted into generic events. Similarly, mode-specific events (air `FlightDeparted`, ocean `VesselDeparted`, ground `BorderArrived`) preserve domain-specific operational detail.

**Key design choice endorsed:** No generic `CustomsProcessed` or generic `VehicleDeparted` events. Domain-specific event names enable consumers to handle jurisdiction-specific and mode-specific logic without parsing generic payloads. The `jurisdiction` and `transport_mode` fields in event envelopes/payloads enable consumer filtering.

### 5.12 6-Stream Topology (Revised from 5)
Correct. The final topology has **6 streams**:

1. `stream:shipment-lifecycle` — 5 core lifecycle events (maxlen 10,000)
2. `stream:customs-processing` — jurisdiction-specific customs events, 5 jurisdictions (maxlen 10,000)
3. `stream:transport-ops` — **NEW** — 35 mode-specific operational events: air 13, ocean 13, ground 9 (maxlen 30,000 — highest volume stream, ~10 events per shipment)
4. `stream:transit-events` — 13 intermediate regulatory transit events (maxlen 20,000)
5. `stream:exception-resolution` — hold resolution, cage lifecycle (maxlen 5,000)
6. `stream:compliance` — compliance screening, analysis, ISF (maxlen 5,000)

The 6th stream (`stream:transport-ops`) is justified. Mode-specific transport events (HAWBIssued, FlightDeparted, VesselArrived, ContainerStuffed, BOLIssued, etc.) are the highest volume event category (~10 per shipment) and represent operational detail that is fundamentally different from lifecycle state transitions. Mixing them into `stream:shipment-lifecycle` would drown the 5 core lifecycle events in operational noise. The `maxlen: 30,000` reflects this volume profile.

Transport-ops has 3 consumer groups: `cg:customs-actor` (mode-specific entry filed events), `cg:financial-actor` (mode-specific customs cleared events for fee calculation), `cg:dashboard-projector` (all events for timeline display).

### 5.13 Ten Business Domains (Revised from 8, Originally 5 Actor Groups)
Correct. The design doc Section 2 now defines **10 bounded contexts** after reconciliation with codebase-analyst's business capability map:

1. **Product Catalog** — CRUD master data, no actors, no events
2. **Trade Intelligence** — Classification + Tariff computation engine, shared kernel
3. **Commercial (Orders)** — order lifecycle, spawns shipments via `ship_order`
4. **Transport & Logistics** — physical movement: shipper, carrier, consolidator, preclearance, terminal, disruption, isf
5. **Customs & Declarations** — customs, pga, cbp_authority, cage
6. **Broker Operations** — broker_sim, 30+ API endpoints, richest UI — elevated to core domain
7. **Compliance & Screening** — compliance_engine, pga
8. **Exception Management** — exceptions, resolution, shipper_response, disruption, terminal — extracted as cross-cutting concern
9. **Financial Settlement** — financial, demurrage, cage (cost tracking)
10. **Regulatory Intelligence** — background monitoring service, no actors

**Key decisions endorsed:**
- **Product split from Classification:** Products are pure CRUD; classification is stateless computation. Different lifecycles.
- **Broker Operations elevated:** 30+ endpoints, richest UI, own state/lifecycle. Too large to be a sub-domain of Customs.
- **Exception Management extracted:** 5 actors, cross-cutting, own resolution lifecycle.
- **Regulatory Intelligence separated:** Autonomous service with no shipment lifecycle dependency.
- **Regulatory Authority NOT separated** — correct, it's simulated actors within Customs.
- **Reporting NOT a context** — correct, it's a read projection pattern.

**Migration impact: MINIMAL.** The 4 new/split domains (Product, Broker, Exception, Regulatory) don't change the actor migration order (still 12 clean / 5 adaptation / 2 rework), stream topology (6 streams), or read model projections. The conceptual separation formalizes what the event-driven design already models — Broker Operations consuming Customs events was always an event boundary, now it's explicitly a bounded context boundary. Exception Management's 5 actors were already migrated individually across Phases 1-3.

### 5.14 Mode-Specific Transport Events (35 Event Types)
Correct. The event catalog now includes **35 mode-specific transport events** across 3 modes:

- **Air freight (13 types):** HAWBIssued → CargoTendered → MAWBConsolidated → ULDBuildUp → FlightDeparted → FlightArrived → ULDBreakdown → AirEntryFiled → AirCustomsCleared → HAWBReleased → AirDelivered (plus ACASFiled)
- **Ocean freight (13 types):** HBLIssued → CargoReceivedCFS → ContainerStuffed → VesselDeparted → VesselArrived → ContainerDischarged → OceanEntryFiled → OceanCustomsCleared → Deconsolidated → HBLReleased → OceanDelivered (plus ISFFiled)
- **Ground/truck (9 types):** BOLIssued → CargoPickedUp → BorderArrived → GroundEntryFiled → GroundCustomsCleared → ProReleased → GroundDelivered (plus PAPSFiled)

**Key design choice endorsed:** Mode-polymorphic document references. Air events carry `house_number`/`master_number`/`flight_number`/`uld_id`; ocean carries `container_number`/`seal_number`/`vessel`/`voyage`; ground carries `pro_number`/`bol_number`. These are NOT abstracted into generic events — `FlightDeparted` is fundamentally different from `VesselDeparted` with different references, timing characteristics, and operational workflows.

**Migration impact: MODERATE on carrier actor (Phase 2a).** The carrier actor becomes the primary producer of mode-specific events, publishing to `stream:transport-ops`. This expands the carrier's event publishing surface from ~5 lifecycle events to ~10+ mode-specific events per shipment. The carrier was already the highest-risk Phase 2a item — this adds scope but doesn't change the approach (hybrid actor with persisted transit_days). The 35 event types are primarily consumed by dashboard-projector for timeline display, with customs and financial also subscribing for mode-specific entry/clearance events.

### 5.15 Shipment Aggregator — Single-Writer Projection Pattern

**Endorsed.** This is the most significant architectural improvement in the final design revision. The Shipment Aggregator replaces the current pattern where 12+ actors independently write to the `shipments` table, creating the JSONB race condition and god object problem.

**What the Aggregator does:**
- Subscribes to all 6 streams via `cg:shipment-aggregator` consumer group
- Maintains the entire `shipments` record as a projection:
  - `status` — projected from lifecycle events
  - `events` — timeline projection (single writer = no race)
  - `hold_type` — projected from `ShipmentHeld` events
  - `entry_number` — projected from `EntryFiled` events
  - `cage_status` — projected from `CageIntake`/`CageReleased` events
  - `financials` — projected from `FeesCalculated` events
  - `references` — projected from various actor events
  - `compliance_result`, `classification_result`, `tariff_result`, `fta_result` — projected from respective domain events
- Also maintains Redis read stores (sorted sets, summary hashes) as before

**SPOF risk assessment:**

| Concern | Assessment | Mitigation |
|---------|-----------|------------|
| **Aggregator crash** | Events queue in Redis Streams PEL. No data loss. | Health check monitors `cg:shipment-aggregator` pending count. On restart, replays unacknowledged events. |
| **Aggregator falls behind** | Read stores and DB become stale. API returns stale data. | `pending > 50` health check alert. At current volume (<1000 events/min), unlikely. |
| **Aggregator bug corrupts projection** | Single writer means single point of corruption. | Shadow mode validation during Phase 2a-2b: compare Aggregator writes against `transition()` writes. 5-minute reconciliation catches drift. |
| **Throughput bottleneck** | Aggregator must process all ~106 event types from all 6 streams. | At current volume (~200 shipments/hour, ~10 events/shipment), this is ~2000 events/hour — trivial for a single consumer. Single-row DB writes at ~5ms each = ~10s/hour of actual DB work. |
| **Ordering requirements** | Cross-stream ordering not guaranteed. | Each event carries sufficient state for independent application. Events within the same stream maintain order. Status changes are guarded by `validate_transition()`. |

**Bottom line:** The SPOF risk is **low at current scale**. The Aggregator processes ~2000 events/hour with ~5ms per write — 99.7% idle time. Redis Streams provides the durability guarantee (events survive Aggregator crashes). The 5-minute reconciliation is the ultimate safety net. At 100x scale (~200,000 events/hour), the Aggregator would still handle the throughput (100s/hour of DB work out of 3600s). Sharding would only be needed at ~1000x current volume.

**Architectural benefit:** Eliminates the entire category of JSONB concurrency bugs by construction. No atomic append, no FOR UPDATE, no read-modify-write. The correct pattern is the only pattern. This is the same architectural principle as "contention prevention by construction" (Section 2.2) applied to the data layer.

### 5.16 Proactiveness as Architectural Quality Attribute

**Endorsed.** This is the platform's most significant architectural principle.

**Stakeholder directive:** "Every service must be designed not to follow a process, but to always evaluate if the requirements are met for it to do its job." (Design doc Section 5.5)

**Why this matters:** The current architecture encodes a strict sequential pipeline: `booked → in_transit → at_customs → entry_filed → cleared → delivered`. Each status gate adds artificial latency. A declaration cannot be filed until the shipment physically arrives at customs (`at_customs`), even if all required data (HS code, origin, destination, declared value, shipper info) was available while the shipment was still `in_transit`. This is the opposite of how real customs brokerage works — in practice, brokers file declarations days before physical arrival to ensure goods can keep moving.

**The sequential pipeline anti-pattern (design doc Section 5.5.3):**

```
SEQUENTIAL MODEL (current):
  booked → in_transit → at_customs → entry_filed → cleared → delivered
  (each step must happen before the next can begin)

PROACTIVE MODEL (target):
  booked → in_transit ────────────────────────────────→ cleared → delivered
                │ (while in transit, in parallel:)            ▲
                ├── Classification completed                  │
                ├── Broker assigned, prep work begins         │
                ├── PGA requirements determined               │
                ├── Declaration pre-filed                     │
                ├── Documents validated                       │
                ├── Fees pre-calculated                       │
                └── If ALL pre-clearance passed ──────────────┘
                    (on arrival: instant clearance, skip at_customs)
```

The proactive model eliminates the `at_customs` bottleneck entirely for pre-clearable shipments. The customs queue shrinks dramatically because most shipments arrive already cleared or with clearance in progress.

**Trade-off analysis:**

| Dimension | Sequential (current) | Proactive (target) |
|-----------|---------------------|-------------------|
| **Latency** | Every gate adds wait time; broker cannot act until `at_customs` | Services act as soon as data prerequisites are met; filing can start at `booked` |
| **Correctness** | Simple — each service sees a complete, stable state at its gate | Data may change after action (HS reclassification after filing → amendment needed) |
| **Complexity** | Status-triggered — one event, one action | Readiness-evaluated — multiple events contribute to prerequisites; readiness accumulator + amendment logic needed |
| **Realism** | Unrealistic — no real brokerage waits for goods to arrive before filing | Realistic — mirrors actual customs operations (pre-clearance is the industry norm) |
| **State model** | Single axis (shipment status = physical location + processing state) | `at_customs` becomes optional — `in_transit → cleared` when pre-clearance succeeds |

**Verdict: Proactive evaluation is the correct architecture.** The added complexity (readiness accumulator, amendment handling) is justified because:
1. It's the platform's competitive differentiator — pre-clearance reduces end-to-end clearance time by 50-80%
2. The complexity mirrors real-world customs operations, making the simulation more realistic
3. The readiness accumulator pattern (Section 5.17) is well-defined with clean restart recovery

**State machine change — `in_transit → cleared` (design doc Section 5.5.3):**

The proactive model adds a new state machine transition: **`in_transit → cleared`** (allowed for `customs` actor when pre-clearance is complete). The `at_customs` state becomes **optional** — it only occurs when:
- Pre-clearance was incomplete (missing data, broker didn't file in time)
- Physical inspection is required (CBP selected the shipment for exam)
- Exception path (holds, enforcement actions)

This is a fundamental shift from the current single-axis model where `at_customs` was a mandatory waypoint. In the proactive model, a shipment that has been fully pre-cleared can transition directly from `in_transit` → `cleared` when the carrier reports arrival.

**Events as information signals, not commands (design doc Section 5.5.4):**

In the proactive model, events have a fundamentally different semantic:
- **Old mental model:** `ShipmentArrivedAtCustoms` → "start customs processing" (command)
- **New mental model:** `ShipmentArrivedAtCustoms` → "physical arrival data now available; all services re-evaluate readiness" (information signal)

Multiple events can satisfy the same prerequisite: the HS code might come from `ClassificationCompleted` (automated) or `ProductUpdated` (manual correction). The service doesn't care which — it just checks "do I have an HS code?"

### 5.17 Service Readiness Criteria Framework

**Endorsed with challenges.** The design doc (Section 5.5) defines the `ProactiveEvaluatorMixin` with an in-memory readiness accumulator. This section evaluates the pattern and its trade-offs.

**Core pattern (design doc Section 5.5.1):** Each service declares `READINESS_FIELDS` — the minimum data prerequisites it needs. The `ProactiveEvaluatorMixin` maintains an in-memory `_readiness` dict that accumulates state from events. On each event, the mixin merges new data and checks `_is_ready()`. When all criteria are met, the actor acts immediately.

**Service taxonomy — three behavioral patterns (design doc Section 5.3):**

| Pattern | Description | Trigger Model | Examples |
|---------|------------|--------------|---------|
| **Proactive Evaluator** | Acts as soon as data prerequisites are met, regardless of shipment physical status | Subscribes to all data-producing events; accumulates state; evaluates readiness on each | customs (declaration), broker_sim (assignment), compliance, financial, documents, PGA, ISF |
| **Reactive Responder** | Responds to commands or requests submitted to it; does not seek work | Subscribes to command events (declarations submitted, requests received) | cbp_authority (adjudicates filings) |
| **Hybrid/Physical** | Requires physical presence, time-dependent, or reactive to physical state | Event-triggered start + timer/physical trigger | Carrier, terminal, cage, demurrage, resolution |

**Important distinction — `customs` actor vs `cbp_authority` actor:**

The design doc correctly classifies the `customs` actor as a **Proactive Evaluator** (it pre-files declarations, runs risk screening — this is the BROKER/FILING function) and `cbp_authority` as reactive (it adjudicates submissions). The stakeholder's clarification about "customs authority being reactive" applies to `cbp_authority`, not `customs`. The `customs` actor in our codebase performs the declaration/screening function (what a broker does), not the government adjudication function.

**Readiness criteria per service (aligned with design doc Section 5.5.2):**

*Proactive Evaluators:*

| Service | Readiness Criteria | Subscribes To (proactive) | Earliest Action | Amendment Trigger |
|---------|-------------------|--------------------------|----------------|-------------------|
| **customs** (declaration) | HS code + origin + dest + value + shipper | `ShipmentCreated`, `ClassificationCompleted`, `ShipmentArrivedAtCustoms` | Pre-files declaration while `in_transit` | `HSCodeReclassified` → amend |
| **broker_sim** (assignment) | Product + origin + dest + value + complexity | `ShipmentCreated`, `ClassificationCompleted` | Assigns broker and begins prep during `booked`/`in_transit` | N/A — assignment doesn't need amendment |
| **compliance** | HS code + parties + origin + regulatory signal | `ShipmentCreated`, `ClassificationCompleted`, `RegulatorySignalDetected` | Re-screens on regulatory changes (not one-shot) | Continuous re-evaluation by design |
| **financial** | Tariff result + declared value | `ClassificationCompleted`, `ShipmentCleared`, `CageReleased` | Pre-calculates fees when tariff data available | `ValueCorrected` → recalculate |
| **documents** | At least one document exists | `ShipmentCreated`, `DocumentUploaded`, `ClassificationCompleted` | Validates as soon as docs uploaded (pre-departure) | Doc change → revalidate |
| **PGA** | HS code → PGA mapping exists | `ClassificationCompleted`, `ShipmentArrivedAtCustoms` | Determines PGA requirements as soon as HS code known | HS reclassification → re-determine |
| **ISF** | Ocean + carrier + HS code + company | `ShipmentCreated` | Already proactive — files pre-departure | Carrier change → amend |

*Reactive Responders:*

| Service | Trigger Events (commands/exceptions) | Response | Re-evaluation Trigger |
|---------|--------------------------------------|----------|----------------------|
| **cbp_authority** | `EntryFiled` (broker submits) | Analyze → accept/reject/flag for exam | `DeclarationAmended` → re-adjudicate |
| **cbp_authority** | `ShipmentArrivedWithoutClearance` (exception) | Default hold — not pre-cleared | N/A (exception path) |

*Hybrid/Physical Actors:*

| Service | Trigger | Why Not Proactive |
|---------|---------|-------------------|
| **Carrier** | Timer-based transit simulation | Physical movement cannot be accelerated by data |
| **Terminal** | Physical vessel arrival | Berth processing requires physical presence at port |
| **Cage** | `ShipmentHeld`/`ShipmentInspection` | Cargo must be physically in custody |
| **Demurrage** | Ocean arrival at port | Storage charges require physical presence of container |

**Challenge analysis — readiness accumulator vs stateless pull:**

The design doc uses an in-memory readiness accumulator (`_readiness: dict[str, dict]`) that merges event data incrementally. An alternative is the "stateless pull" pattern where each event triggers a DB read and readiness evaluation against the current shipment record.

| Dimension | Readiness Accumulator (design doc) | Stateless Pull (alternative) |
|-----------|-----------------------------------|------------------------------|
| **DB load** | Zero DB reads for readiness evaluation — all from event payloads | One DB read per event per service — at current volume (~2000 events/hour × 7 proactive services = ~14,000 reads/hour) this is tolerable |
| **Memory** | ~500 bytes per tracked shipment × ~200 active shipments × 7 services = ~700KB total | Zero in-memory state |
| **Restart recovery** | Must replay from Redis Streams PEL to rebuild accumulator | No recovery needed — stateless |
| **Correctness** | Accumulator could drift if events are lost or reordered | Always reads authoritative DB state — no drift possible |
| **Complexity** | Mixin with accumulate/ready/cleanup lifecycle | Simpler — just read and evaluate |

**Assessment: The readiness accumulator is the BETTER choice** despite slightly more complexity. Rationale:
1. Zero DB reads for readiness evaluation aligns with the core design goal (decouple actors from PostgreSQL)
2. The state-transfer pattern already puts all needed data in event payloads — the accumulator leverages this
3. At scale, 14,000 DB reads/hour from stateless pull would partially re-introduce the actor-DB coupling we're eliminating
4. Restart recovery via Redis Streams replay is already part of the event-driven design — no new mechanism needed
5. Memory overhead (~700KB) is negligible

**Challenge: `in_transit → cleared` state machine change:**

The new transition `in_transit → cleared` (design doc Section 5.5.3) is architecturally sound but creates downstream risks:
- **UI surfaces must handle the missing `at_customs` step** — broker queue, dashboard metrics, timeline display. A shipment that was never `at_customs` will look anomalous to users expecting the traditional sequence.
- **Reporting/analytics**: Pre-clearance success rate becomes a new metric. The distinction between "arrived and cleared" vs "pre-cleared before arrival" matters for operational analytics.
- **Mitigation**: UI should show the pre-clearance timeline (classification, filing, review — all while `in_transit`), making it clear that work happened in parallel rather than the shipment "skipping" steps.

**Challenge: Proactive broker assignment — wasted work risk:**

If the broker is assigned and begins prep while the shipment is `in_transit`, but the shipment is rerouted or cancelled:
- **Assessment: LOW RISK.** Shipment rerouting/cancellation is rare in the simulation. The broker assignment is lightweight (create assignment + entry filing records). The actual heavy work (filing, document review) only happens after more prerequisites are met.
- **Mitigation**: On `ShipmentCancelled` or `ShipmentRerouted` events, the broker handler unwinds its work (cancel entry filing, release assignment).

**Challenge: Replay-based recovery speed:**

On restart, proactive evaluator actors must replay unacknowledged events from Redis Streams to rebuild their readiness accumulators. Is this fast enough?
- **Assessment: YES.** With `maxlen` caps (10K-30K per stream) and typical PEL sizes of <100 events per consumer group at restart, replay takes <1 second. The bottleneck is event processing, not stream reading — and readiness accumulation is a pure in-memory merge, not a DB operation.
- **Worst case**: All 6 streams × maxlen entries replayed = ~80K events. At ~1ms per event (in-memory merge only), this is ~80 seconds. This worst case only happens if ALL consumers have never ACKed any events — effectively a fresh start, which also triggers seeding from DB.

**Soft vs hard prerequisites:**

- **Soft prerequisites** (data can change after action): HS code, declared value, origin determination. Services acting on soft prerequisites MUST implement amendment handlers.
- **Hard prerequisites** (data is final once set): Entry number assigned, CBP response received, physical arrival confirmed. Services gating on hard prerequisites don't need amendment logic.

**Amendment/re-evaluation flow:**

When a prerequisite field changes after a service has already acted:
1. The change event (e.g., `HSCodeReclassified`) is published to the relevant stream
2. Services that previously acted based on the old value re-evaluate
3. If the change affects their output, they emit an amendment event (e.g., `DeclarationAmended`, `FeesRecalculated`)
4. Downstream services (customs authority) receive the amendment and re-adjudicate

This creates amendment cascades: `HSCodeReclassified → DeclarationAmended → CustomsReReview → FeesRecalculated`. These cascades are **correct behavior** — they mirror real-world amendment workflows. The risk is not the cascade itself but the frequency. If data changes frequently before stabilizing, services may thrash. Mitigation: debounce re-evaluation (e.g., 5-second window after a change event before re-evaluating) or act only after a "data stable" signal.

**The "should not happen" scenario:**

A shipment arriving at the port of entry without pre-clearance is a **failure of the proactive system**. It means the broker didn't file in time (missing data, broker overload, system error). The customs authority's response (hold by default) is the correct exception path. The platform should track "pre-clearance miss rate" as a key operational metric — a healthy system should have >95% pre-clearance success rate.

---

## 6. Dropped Patterns (Confirmed)

| Pattern | Why Dropped |
|---------|------------|
| **Version vectors / optimistic UI** | Read models make API reads fast; no UI-level consistency needed |
| **Event schema versioning / registry** | Single deployment unit; schema changes are code changes |
| **Full read model projections for shipment detail** | Superseded — Shipment Aggregator now maintains full `shipments` record as projection. Write-through cache retained for API detail endpoint (30s TTL). |
| **All-19-actor migration to pure event-driven** | Timer-based actors are better expressed as polls; hybrid is the correct pattern |
| **LIMIT + per-row commits as standalone fix** | Band-aid that doesn't solve correctness problems; event-driven solves both performance and correctness |

---

## 7. Risk Matrix

| Risk | Severity | Likelihood | Mitigation |
|------|----------|-----------|------------|
| Shipment Aggregator SPOF / sole writer risk | **Medium** | **Low** | Health check on pending count; shadow mode validation during Phase 2a-2b; 5-minute reconciliation; Redis Streams PEL durability; ~99.7% idle at current volume (see Section 5.15) |
| Customs refactoring introduces bugs in screening logic | **High** | Medium | Pure function extraction with existing test coverage; side-by-side comparison |
| BrokerSim decomposition breaks entry filing pipeline | **High** | Medium | Explicit event chain with integration tests; shadow mode comparison |
| Non-deterministic re-rolls in time-dependent actors | Medium | Medium | Persist computed durations on first evaluation |
| Event payload too large, Redis memory pressure | Medium | Low | Bounded `ShipmentEventPayload` (~1-2KB); 6 streams with MAXLEN 5K-30K |
| Dashboard counter drift over time | Medium | Low | 5-minute full SQL reconciliation; startup seeding |
| Hybrid actor `_tracking` lost on restart | Low | Low | DB re-query on startup; `_tracking` is cache only |
| Consumer group fan-out overhead at scale | Low | Low | Unmeasurable at current volume (<1000 events/min); documented for future |
| Schedule slip due to customs + broker_sim complexity | Medium | **High** | Budget 1-2 days for customs, 2-3 days for broker_sim; pilot customs first |
| Redis Streams operational unfamiliarity | Medium | Medium | 6 streams with clear domain mapping; health check + dead letter alerting |
| Premature action on soft prerequisites | **Medium** | Medium | Amendment handlers on all proactive evaluators; soft vs hard prerequisite classification; debounce window for rapid data changes |
| Amendment cascade complexity | **Medium** | Low | Cascades mirror real-world customs amendments; debounce prevents thrashing; "data stable" signal optional; track cascade depth metric |
| Pre-clearance failure path — shipment arrives without clearance | **High** | Low | Default hold by customs authority (exception path); track pre-clearance miss rate as operational metric; state machine must allow `cleared → inspection` for post-entry exams |
| State machine redesign — dual state dimensions | **High** | Medium | Physical state and clearance state progress independently; requires careful migration from single-axis model; validate all UI surfaces handle dual state correctly |

---

## 8. Event Catalog Corrections (from feasibility check)

The feasibility check identified 3 corrections to the event catalog:

| Actor | Design Originally Said | Correction |
|-------|----------------------|-----------|
| consolidator | Subscribes to `ShipmentCreated` | Should subscribe to `ShipmentInTransit` (actor processes `in_transit` shipments with `consolidation_id IS NULL`) |
| demurrage | Subscribes to `CageIntake` | Should subscribe to `ShipmentArrivedAtCustoms` (ocean filter) — demurrage is independent of caging |
| shipper_response | Subscribes to `CF28Issued`, `HoldEscalated` | Should subscribe to `CommunicationSent` event type (any communication requiring response) |

These corrections are reflected in the event-architect's revised design document.

---

## 9. Consumer Group Topology Assessment

With the fully reconciled event catalog (~106 event types across 10 bounded contexts), the topology has **6 streams** with a total of **36 consumer groups**:

- `stream:shipment-lifecycle` — **14 consumer groups**. 5 core lifecycle event types (ShipmentCreated through ShipmentDelivered). Fan-out overhead at our scale: unmeasurable. `maxlen: 10,000`.
- `stream:customs-processing` — **8 consumer groups** (resolution, cage, carrier, broker_sim, shipper_response, cbp_authority, financial, dashboard-projector). Carries jurisdiction-specific events: US 11, EU 6, BR 4, IN 5, CN 5, plus 4 cross-jurisdiction shared events (~35 types total). Consumers filter by `jurisdiction` field. `maxlen: 10,000`.
- `stream:transport-ops` — **3 consumer groups** (customs, financial, dashboard-projector). **NEW — highest volume stream.** 35 mode-specific operational events (air 13, ocean 13, ground 9). ~10 events per shipment. Customs subscribes for mode-specific entry events (AirEntryFiled, OceanEntryFiled, GroundEntryFiled). Financial subscribes for mode-specific clearance events. `maxlen: 30,000`.
- `stream:transit-events` — **4 consumer groups** (compliance, resolution, carrier, dashboard-projector). 13 intermediate transit event types (filing, screening, hub sort, transshipment). `maxlen: 20,000`.
- `stream:exception-resolution` — **5 consumer groups** (carrier, cage, financial, shipper_response, dashboard-projector). Unchanged. `maxlen: 5,000`.
- `stream:compliance` — **1 consumer group** (dashboard-projector). Unchanged. `maxlen: 5,000`.

**Total Redis memory estimate:** ~106 event types × 1-2KB payloads × maxlen caps = under 200MB worst case. Well within single-instance Redis capacity.

**Assessment:** The 6-stream topology is justified. Each stream maps to a distinct concern:
1. **Lifecycle** (state machine transitions) — consumed by nearly all actors
2. **Customs** (jurisdiction-specific authority decisions) — consumed by actors in the customs pipeline
3. **Transport-ops** (mode-specific operational detail) — highest volume, primarily dashboard visualization
4. **Transit** (intermediate regulatory touchpoints) — compliance screening + dashboard
5. **Exception** (hold resolution, cage lifecycle) — targeted consumers
6. **Compliance** (analysis, screening results) — dashboard-only

The transport-ops stream prevents mode-specific events from drowning lifecycle events. The transit-events stream keeps intermediate regulatory events separate from destination customs processing. Both separations improve consumer efficiency — actors subscribe only to the streams they need.

**Jurisdiction filtering**: Jurisdiction-specific events all flow through `stream:customs-processing`. Jurisdiction-aware consumers (e.g., `cbp_authority` only processes US events) filter by the `jurisdiction` field in the event envelope. Jurisdiction-agnostic consumers (like `dashboard-projector`) process all events. No per-jurisdiction streams needed at current scale.

**Mode filtering**: Mode-specific events on `stream:transport-ops` carry `transport_mode` in the payload. Consumers can filter by mode if needed, though currently all 3 consumer groups process all modes.

---

## 10. Closed Questions (Resolved During Review)

1. **JSONB events column ownership**: **Resolved.** Stakeholder insight accepted: the shared `events` JSONB column is an anti-pattern. In the target architecture, the Shipment Aggregator is the sole writer to the entire `shipments` table (including `events`). Atomic SQL append (`||`) retained in Phase 0-2a as a transitional safety net only, removed in Phase 3 when the Aggregator becomes sole writer. Separate events table evaluated as a follow-on sprint. Consensus.

2. **Event payload schema enforcement**: **Implementation-level decision.** Design doc uses `@dataclass`. Recommend Pydantic for runtime validation (~0.1ms overhead, negligible at <1000 events/min). Decide during Phase 0 implementation.

3. **Stream trimming responsibility**: **Resolved.** Design doc Section 4.4 confirms trimming on coordinator dashboard cycle. Consensus.

4. **Consolidator buffered consumer pattern**: **Resolved.** Design doc Section 11.2 provides full implementation with event accumulation + periodic flush. Consensus on option (a) — hybrid actor.

5. **Customs authority behavioral pattern**: **Resolved.** Stakeholder clarification: the customs authority (CBP) is a Reactive Responder, not a Proactive Evaluator. It responds to `DeclarationSubmitted` commands and `ShipmentArrivedWithoutClearance` exceptions. The BROKER is the proactive evaluator who decides when to file. This distinction mirrors real-world customs operations — brokers initiate, authorities adjudicate.

---

*This analysis validates the event-driven design against the feasibility data. The architecture is sound — 12 of 19 actors convert cleanly, 5 need moderate adaptation, and 2 require careful rework. All seven implementation challenges have been resolved in the design document (Section 11 systemic issues, Section 5.6 BrokerSim decomposition, Section 3.9 payload schema). The fully reconciled design (10 bounded contexts, ~106 event types, 6 Redis Streams, 36 consumer groups) is comprehensive yet pragmatic.*

*The Proactive Evaluator principle (Sections 5.16-5.17) introduces a fundamental shift aligned with the design doc (Section 5.5): services act at the earliest possible moment when data prerequisites are met, not at sequential status gates. This enables pre-clearance (filing while goods are in transit) via the new `in_transit → cleared` state machine transition. Three service patterns: Proactive Evaluators (7 services — customs, broker, compliance, financial, documents, ISF, PGA — using the readiness accumulator pattern), Reactive Responders (cbp_authority — adjudicates filings submitted to it), and Hybrid/Physical Actors (carrier, terminal, cage, demurrage — require physical presence or time). The readiness accumulator is endorsed over stateless pull for zero-DB-read evaluation. Four challenges analyzed: accumulator memory (negligible), `in_transit → cleared` UI impact (manageable), proactive broker wasted work (low risk), replay recovery speed (sub-second typical). The design-challenger endorses the Proactive Evaluator pattern as architecturally sound and commercially vital.*
