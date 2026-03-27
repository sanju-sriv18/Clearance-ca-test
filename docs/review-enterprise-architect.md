# Enterprise Architect Review: Clearance Platform Architecture (v4)

**Reviewer:** Principal Enterprise Architect (30 years, carrier clearance & customs brokerage systems)
**Document Reviewed:** `docs/architecture-clearance-platform.md` (v4, 2026-02-08)
**Review Date:** 2026-02-09

---

## Executive Summary

This architecture document describes a well-considered domain decomposition for a customs clearance intelligence platform. The author clearly understands the domain — the separation of commercial (shipper), physical (HU/consolidation), regulatory (declaration/adjudication), and intelligence concerns is correct and reflects how real customs brokerage operations work. The event-driven choreography model, CQRS read/write split, and simulation-as-consumer principle are architecturally sound choices.

However, the document has significant gaps between its ambition and the practical reality of implementation. The current codebase is a traditional monolith with zero event infrastructure, zero schema isolation, and zero CQRS — making this document a target-state vision, not a description of what exists. Several design choices will create operational problems at scale. The state-transfer event pattern, while simple, will hit bandwidth walls. The single Redis Cache Projector is a catastrophic single point of failure. The movement event cascade will create event storms. And the 5-year audit retention strategy for customs compliance is completely absent.

**Bottom line:** The domain model is strong. The event architecture needs hardening. The gap between current state and target state needs a realistic migration plan, not the four-line table in Section 11.2.

---

## 1. Domain Boundary Assessment

### 1.1 What's Right

The 13-domain decomposition (Section 3) is **fundamentally correct** and aligns with how customs brokerage actually works. Specific wins:

- **Shipment Lifecycle as "convenience abstraction" (Section 3.5):** This is exactly right. In every customs system I've built, the shipper's mental model (shipment) and the customs authority's model (handling unit / bill of lading) are different things. Making the shipment a non-authoritative projection that derives status from HUs is the correct architectural choice. I've seen three major systems fail because they made the shipment the source of truth for customs status.

- **Declaration Management vs. Customs Adjudication separation (Sections 3.8, 3.9):** This is one of the best decisions in the document. These are fundamentally different bounded contexts with different owners (broker vs. authority). I've seen systems that merge these, and they always end up with tangled state machines where filing status and authority decisions are mixed in one entity. The clean separation here will pay dividends when integrating with real CBP ACE or EU ICS2.

- **Trade Intelligence as a proper domain with persistent state (Section 3.2):** Correct. Classification results, tariff calculations, and FTA determinations are first-class business objects with version history, not throwaway computation results. This is a lesson most customs platforms learn too late.

- **Product Catalog owning "what the shipper says" vs. Trade Intelligence owning "what the system determines" (Sections 3.1, 3.2):** This boundary is clean and correct. In real brokerage, shipper-declared HS codes diverge from system-classified codes in ~40% of entries. These must be separate.

### 1.2 What Needs Work

**The Cargo & Handling Units / Consolidation boundary creates problematic coupling.**

Sections 3.6 and 3.7 describe two domains that are tightly coupled in practice:
- HU subscribes to Consolidation movement events (Section 3.6, Events Consumed)
- Consolidation subscribes to HU customs status changes (Section 3.7, Events Consumed)
- Both store `consolidated_into` references to each other
- The `can_deconsolidate` flag requires aggregating child customs states

This is bidirectional event coupling. In every microservices migration I've done, bidirectional event dependencies between two domains signal they should be **one domain** or there needs to be a **coordination service**. The document says these are separate domains but they cannot function independently — HU behavior changes based on whether it's consolidated, and consolidation behavior changes based on HU customs state.

**Recommendation:** Merge Cargo & Handling Units and Consolidation into a single **Physical Cargo** domain. The nesting relationship (HU → Consolidation → parent Consolidation) is an internal concern of that domain, not a cross-domain event flow. This eliminates the bidirectional coupling and the movement cascade complexity (Section 3.7.1). If you later need to extract for scaling, split by read/write path (movement writes vs. customs status reads), not by entity type.

**Exception Management scope is too broad.**

Section 3.10 describes a domain handling holds, cage/exam, resolutions, AND supply chain disruptions. These are different concerns:
- **Holds and cage management** are tightly coupled to the HU/Consolidation lifecycle — they affect `can_deconsolidate`, customs status, and financial accrual. In real customs operations, cage management is a physical cargo concern, not an "exception."
- **Supply chain disruptions** (port closures, weather, strikes) are a completely different domain — they affect routing and timeline, not customs clearance directly.

Having them in one domain creates a god service that touches too many other domains' concerns. The Exception Management domain already consumes events from Adjudication, Compliance, AND Cargo & HUs (Section 3.10, Events Consumed), and its events are consumed by Cargo & HUs AND Financial.

**Recommendation:** Move holds/cage management into the Physical Cargo domain (or Cargo & HUs if you keep them separate). Create a separate **Supply Chain Disruption** domain for disruption monitoring and impact assessment. Exception Management becomes a thin coordination/workflow layer, not a state-owning domain.

**Financial Settlement is underspecified relative to its complexity.**

Section 3.11 covers duties, fees, demurrage, bonds, drawback, and reconciliation. In real customs operations, this is typically **two or three** separate services:
1. **Duty & Fee Calculation** — deterministic computation from tariff data
2. **Billing & Payment** — accounts receivable, payment processing, statement generation
3. **Post-Entry Reconciliation** — liquidation, drawback, protests, adjustments (this alone takes 10+ years to complete for some entries)

The document treats this as one domain with one lifecycle (estimated → provisional → liquidated → reconciled). In reality, the D&D lifecycle is completely independent from the duty lifecycle. A shipment's demurrage can be accruing while its duty is already liquidated. The bond lifecycle is also independent — bonds span multiple entries and have their own renewal/sufficiency/exhaustion lifecycle.

**Recommendation:** At minimum, acknowledge the sub-domain split within Financial Settlement and design the internal module boundaries to support future extraction. At the current scale, one service is fine, but the data model should reflect the independent lifecycles.

### 1.3 Domain Count

13 domains is on the high end for a system processing the volume described. In my experience, 8-10 well-defined domains are optimal for a customs platform. The merge of Cargo & HUs + Consolidation into Physical Cargo and the narrowing of Exception Management would bring this to 11-12, which is reasonable.

---

## 2. Event Architecture Quality

### 2.1 NATS Subject Hierarchy

The subject hierarchy (Section 4.1) is **well-designed**. The `clearance.{domain}.{event-type}` pattern is clean and follows NATS best practices. Specific notes:

- The `sim.*` prefix for simulation events (Section 4.1, bottom) is correct — clean namespace separation.
- The jurisdiction-specific subjects under `clearance.adjudication.{jurisdiction}.*` (Section 4.1) are a good choice for jurisdiction-specific routing.
- The `clearance.handling-unit.cage.*` sub-hierarchy is appropriate for cage-specific subscribers.

**Missing: entity ID in subject hierarchy for partitioning.** Section 6.4 mentions `clearance.shipment.{shipment_id}.status-changed` for entity-level ordering, but this contradicts the hierarchy in Section 4.1 which shows `clearance.shipment.status-changed` without entity ID. This inconsistency needs resolution. If you use entity IDs in subjects, the JetStream stream subjects need to include wildcards (`clearance.shipment.*.status-changed`), which changes stream configuration.

**Missing: version prefix.** The hierarchy should include a version prefix (`clearance.v1.product.created`) to support schema evolution without breaking existing consumers. The `metadata.version` field (Appendix A) is not sufficient — consumers need to route by subject, not by inspecting payloads.

### 2.2 JetStream Stream Design

The 13 streams (Section 4.2) map 1:1 to domains, which is the correct starting point. Specific concerns:

**Stream retention is too short for regulatory compliance.** The `CLEARANCE_FINANCIAL` stream has `180d` max age; `CLEARANCE_DECLARATION` has `90d`. US CBP requires **5-year retention** of all entry-related records (19 CFR Part 163). The events themselves are the authoritative record of what happened — if someone replays 2 years later for a liquidation protest, the events must be available.

**Recommendation:** Set a minimum 5-year retention for all `CLEARANCE_*` streams except `CLEARANCE_PRODUCT` (30d is fine). Use tiered storage — hot data in NATS JetStream (90d), warm data archived to S3/object storage (5 years). NATS JetStream was not designed for long-term archival at scale.

**The `SIM` stream (7d, Memory storage):** Good choice. Simulation events should be ephemeral.

**Missing: stream replication configuration.** No mention of NATS cluster replication factor (R1, R3). For a customs system, `R3` with synchronous acknowledgment is the minimum. The [Jepsen analysis of NATS 2.12.1](https://jepsen.io/analyses/nats-2.12.1) found that NATS JetStream loses acknowledged writes under default fsync settings. You **must** configure `sync_always: true` for clearance streams or risk losing regulatory records.

### 2.3 State-Transfer Events (Full Snapshots)

The document commits to state-transfer events carrying full state snapshots (Principle 2, Section 1.2). This is a **defensible but expensive choice**.

**Pros (correctly identified in the doc):**
- Consumers are self-sufficient — no need to query the producer
- Late-joining consumers get full state on every event
- Idempotency is simpler — replaying a full snapshot is safe
- No event ordering issues for missed events

**Cons (not addressed in the doc):**

**Bandwidth explosion at scale.** A `HandlingUnit` entity (Section 3.6) includes `transit_history` (JSONB array that grows with every movement), `cage_status` (nested JSONB), and `current_location` (JSONB). At 100K entries/day, with an average 4 movement events per HU, each carrying the full HU snapshot including the growing transit_history array, you're looking at:

- Average HU snapshot size: ~2-5 KB (with history)
- 100K entries × ~3 HUs per entry × 4 movements × 5 KB = **6 GB/day** of movement events alone
- Plus cascade multiplier (each movement generates child events)

This is before counting consolidation events, status changes, customs events, etc.

**Recommendation:** Use a **hybrid approach**: state-transfer events for entity lifecycle events (created, status changed, cleared) and **notification events** with entity ID + delta for high-frequency events (movement, cage dwell updates). Consumers that need full state can query the Redis cache. This is the pattern used by every logistics platform I've worked with that processes more than 50K shipments/day.

**The `transit_history` array growing unbounded inside the state-transfer event is an anti-pattern.** It means every new movement event is larger than the last, for the entire lifecycle of the HU. Cap the history in the event payload (last N events) and keep the full history in PostgreSQL.

### 2.4 Movement Event Cascade

The cascade pattern (Section 3.7.1) is **the most dangerous design in this architecture.**

A single top-level consolidation movement generates:
1. One `ConsolidationMovement` for the top-level consolidation
2. One `ConsolidationMovement` per nested consolidation
3. One `HUMovement` per HU in each nested consolidation
4. One `HUMovement` per HU directly in the top-level consolidation
5. One `ShipmentStatusChanged` per affected shipment

For a realistic ocean consolidation: 1 MBL with 3 containers, each container with 20 HBLs, each HBL mapping to 1-3 HUs — that's **60-180 HUs**. A single movement event cascades to:
- 3 nested consolidation events
- 60-180 HU events
- 60-180 shipment projection updates

**Total: 123-363 events from ONE consolidation movement.** At 10 movements per ocean voyage (departure, transshipment arrival, transshipment departure, arrival, etc.), a single MAWB/MBL generates **~1,230-3,630 events**.

At 10K consolidations/day, that's **12-36 million events/day** from movement alone. NATS can handle this throughput, but the Redis Cache Projector (a single process, Section 6.1) and the PostgreSQL write paths in each domain service cannot.

**Recommendation:**
1. **Batch the cascade.** When a consolidation moves, compute the full set of affected entities and publish a **single batch event** containing all affected entity IDs + the new location. Let the Redis projector update all entities in one atomic operation.
2. **Use lazy propagation.** Don't cascade immediately. Mark the top-level consolidation as moved. When an HU or child consolidation is queried, compute its position from its parent chain. This is the pattern used by FedEx's internal tracking system — they don't propagate 2 million package location updates when a plane lands.
3. **Introduce a Movement Projection service** that subscribes to top-level movement events and maintains a materialized view of "where is everything." This replaces the cascade entirely.

### 2.5 Missing Events

- **`DeclarationReadyToFile`**: The readiness model (Section 7.1) is described but no event is emitted when a declaration transitions to `ready_to_file`. The broker queue needs this for sorting.
- **`BondExhausted`** / **`BondRenewalDue`**: Bond lifecycle events are missing. A bond becoming exhausted blocks all new entries for that importer — this is a critical operational event.
- **`ISFDeadlineApproaching`**: ISF has hard deadlines (24h before vessel loading). There's no warning event.
- **`LiquidationCompleted`**: The financial lifecycle goes to `liquidated` but there's no event for downstream consumers (reconciliation, drawback eligibility).
- **`ReclassificationTriggered`**: When a regulatory signal causes reclassification of existing products, there's no event to inform brokers that previously filed entries may need amendment.

---

## 3. CQRS Implementation

### 3.1 Redis as Read Store

The Redis Cache Projector design (Section 6.1) is **architecturally sound for reads** but has critical operational gaps.

**The single Cache Projector is a catastrophic single point of failure.** Section 6.1 describes a single process subscribing to all `clearance.>` events. If this process crashes:
- All read APIs return stale data
- The broker queue stops updating
- Dashboard aggregates freeze
- No real-time UI updates

The document mentions "periodic reconciliation (5-min)" in Section 6.3 but says nothing about how the projector recovers after a crash. With at-least-once delivery from NATS, the projector would replay all unacknowledged messages — but if it was down for 30 minutes during a peak period, it could face millions of events to replay.

**Recommendation:**
1. **Run multiple projector instances** with NATS consumer groups (round-robin delivery). Each instance handles a subset of subjects.
2. **Implement a warm-standby projector** that receives all events but only writes to Redis if the primary fails (using Redis `SETNX` for leadership election).
3. **Add a full-rebuild capability** — the projector must be able to rebuild the entire Redis state from PostgreSQL on cold start (not just from NATS replay, since streams have max_age).
4. **Split the projector by domain.** Having one process subscribe to all 13 domain streams is a scaling bottleneck and a blast radius issue. A bug in the financial event projector shouldn't affect the movement event projector.

### 3.2 Consistency Model

The eventual consistency model (Section 6.3, "<100ms typical lag") is **adequate for a customs platform**. Customs operations are inherently asynchronous — brokers don't need real-time millisecond consistency. The 5-minute reconciliation is a reasonable safety net.

**However:** The document doesn't address the **read-your-own-writes problem**. When a broker submits an entry (write to PostgreSQL → event → projector → Redis), what does the API return? If the submit endpoint returns immediately and the broker refreshes their queue, they might not see the status change for up to 100ms. In a high-throughput broker workbench, this creates confusion.

**Recommendation:** For write endpoints, return the new state directly in the HTTP response (bypassing Redis). The document hints at this in Section 6.3 ("Writes are immediately consistent — domain services return the new state directly") but doesn't make it explicit in the API surface definitions. Make this a documented contract.

### 3.3 Redis Key Design

The key structure (Section 6.1) is well-designed for the access patterns described. The sorted sets for index queries (`idx:shipments:by-status:{s}`) and the broker queue priority structure (`queue:broker:unassigned`) are correct Redis patterns.

**Concern: no TTL strategy.** What happens to Redis keys for completed shipments? A shipment that delivered 6 months ago still has keys in Redis. At 100K entries/day, that's 18M+ entities/year accumulating in Redis memory.

**Recommendation:** Set TTLs on entity keys (e.g., 90 days after final status) and remove from sorted set indexes when TTL expires. For historical queries, fall back to PostgreSQL.

---

## 4. Data Model Quality

### 4.1 Entity Models

The entity models are **generally well-designed** with appropriate use of JSONB for flexible/nested data and proper normalization for structured fields. Specific issues:

**The `HandlingUnit.transit_history` JSONB array (Section 3.6) will become a performance problem.** JSONB arrays in PostgreSQL are not efficient for append-only growth patterns. Every movement event rewrites the entire JSONB column. For an HU with 20+ movements (common for multi-modal shipments), this becomes expensive.

**Recommendation:** Extract `transit_history` into a separate `hu_transit_events` table with proper indexing. Keep a `last_known_location` column on the HU for the common query pattern.

**The `FinancialRecord` entity (Section 3.11) tries to be too many things.** It has ocean D&D, air storage, cage fees, duty lifecycle, payment status, and bond references all in one entity. This is a god entity. The `demurrage` JSONB blob has 20+ fields covering two completely different rate structures (ocean vs. air).

**Recommendation:** Split into: `DutyRecord` (duty/fee lifecycle), `DemurrageRecord` (D&D tracking with mode-specific structures), and `PaymentRecord` (payment tracking). These have independent lifecycles and different update frequencies.

### 4.2 Cross-Schema Foreign Keys

The document correctly prohibits cross-schema JOINs (Section 10.3) but the entity models contain cross-domain UUIDs as implicit foreign keys (e.g., `HandlingUnit.shipment_id`, `EntryFiling.handling_unit_id`). These are **reference IDs**, not enforced foreign keys — which is correct for eventual extraction.

**However:** The document doesn't describe how referential integrity is maintained when domains are separate services. If a shipment is deleted, what happens to HUs referencing it? The event model doesn't include deletion cascades.

**Recommendation:** Document the **reference ID contract**: reference IDs are eventually consistent, not foreign keys. Include `EntityDeleted` events for each domain. Consumers that hold reference IDs must handle the referenced entity not existing.

### 4.3 JSONB vs. Separate Tables

JSONB is used appropriately in most cases:
- `checklist_state` on EntryFiling — correct, this is a dynamic structure that varies by jurisdiction
- `cage_status` on HandlingUnit — acceptable for now, but should be extracted if cage management becomes a hot path
- `transit_history` on HandlingUnit — **incorrect**, should be a separate table (see 4.1)
- `demurrage` on FinancialRecord — **incorrect**, too complex for JSONB (see 4.1)
- `content_base64` on Document — **incorrect**, storing document content as base64 in PostgreSQL is a known anti-pattern that will cause vacuum bloat and replication lag at scale

---

## 5. State Machine Design

### 5.1 Status Enums and Transitions

The status enums are **mostly realistic** for US customs. Specific issues:

**EntryFiling `filing_status` (Section 3.8) is missing several real-world states:**
- `pending_payment` — CBP won't release until duty deposit is confirmed (PMS/ACH cycle takes 1-3 business days)
- `suspended` — entry suspended pending investigation (distinct from rejected)
- `cancelled` — filer-initiated cancellation before CBP acceptance
- `in_bond` — goods moving under bond to another port (T&E, IT entries)
- `cf28_pending` and `cf29_pending` are listed but `penalty_assessment` is not — CF-29s can lead to formal penalty proceedings separate from the notice itself

**The `can_deconsolidate` computed flag (Section 3.7) is necessary but not sufficient.**

Real-world deconsolidation requires:
1. No active customs holds (modeled by `can_deconsolidate`)
2. All required documents validated (not modeled)
3. Carrier release obtained (not modeled — this is the carrier's authorization to break the consolidation)
4. Terminal availability confirmed (not modeled — at congested ports, you can't deconsolidate without a terminal appointment)

**Recommendation:** Change `can_deconsolidate` from a boolean to a **readiness checklist** similar to the filing readiness model in Section 7.1. Each prerequisite is tracked independently.

### 5.2 Edge Cases

**Partial clearance (Section 3.5, `partially_cleared` status):** This is handled correctly — the shipment derives `partially_cleared` when some HUs are cleared and others aren't. However, the declaration model doesn't explicitly support **partial release** — where CBP releases some line items from an entry but holds others. This happens with PGA splits (FDA clears lines 1-3, but CPSC holds line 4).

**Split shipments:** The document mentions "future: many-to-many via join table" for Order-to-Shipment (Section 3.4) but doesn't address the inverse — a **single HU appearing in multiple entries**. This happens with informal entries (< $2,500 threshold) that later get consolidated into formal entries, and with FTZ (Foreign Trade Zone) entries where goods are admitted under one entry and withdrawn under another.

**Amendment after filing:** Handled via `DeclarationAmended` event (Section 3.8). But the model doesn't track **amendment history** — which fields were changed, by whom, why. CBP requires amendment justification (19 CFR 173.4). The `EntryFiling` entity should include an `amendments` JSONB array or separate amendment records.

**Re-export / drawback chain:** The `DutyDrawback` entity (Section 3.11) has `original_entry_id` and `export_entry_id`. In practice, drawback claims can chain across multiple import entries (substitution drawback under 19 USC 1313(j)(2)). The model needs to support many-to-one (multiple import entries → one drawback claim).

---

## 6. Scalability Concerns

### 6.1 Hot Paths

The hottest paths in this architecture:

1. **Movement event cascade** (Section 3.7.1) — See analysis in Section 2.4 above. This is the primary scaling concern.
2. **Redis Cache Projector** — Single process handling all events is the bottleneck.
3. **Classification Engine** (LLM-dependent) — Every new product triggers an LLM call. At 1K new products/day, this is manageable. At 50K/day (common for large importers), LLM latency and cost dominate.
4. **Declaration Management event consumption** — This domain subscribes to events from 6 other domains (Section 4.3). Each event triggers checklist evaluation and readiness scoring. At scale, this becomes CPU-bound.

### 6.2 Scaling Scenarios

**10K entries/day (current target):** The architecture as described works with the single-process modular monolith. The movement cascade is manageable (est. 1-3M events/day). Redis Cache Projector is a risk but recoverable.

**100K entries/day (medium customs broker):** The movement cascade becomes problematic (est. 30-50M events/day). The Redis Cache Projector must be split. LLM classification costs become significant ($10K-50K/month depending on model). The PostgreSQL single-instance will need read replicas. NATS JetStream needs a 3-node cluster minimum.

**1M entries/day (top-5 broker):** The architecture as designed breaks. State-transfer events at this volume generate terabytes of event data per day. The monolithic PostgreSQL database cannot handle the write load even with schema isolation. Physical database separation is mandatory. The entire movement cascade design must be replaced with lazy propagation.

### 6.3 The `consolidation:{id}:all_hus` Cache

Section 3.7 describes a Redis set `consolidation:{id}:all_hus` — a flattened set of all HU IDs inside a consolidation, regardless of nesting depth.

**Concern:** For a deeply nested consolidation (MBL → 3 containers → 60-180 HUs), this set has 60-180 members. That's fine. But who maintains it? The document says "maintained by a NATS subscriber that listens to consolidation/HU events" — is this the Cache Projector, or a separate process? If it's the Cache Projector, it's yet another responsibility on that single process. If it's separate, it's another process that can fail independently.

**More concerning:** The set must be updated atomically when HUs are added/removed from consolidations. If the set gets out of sync (race condition during deconsolidation), the broker sees incorrect HU counts and makes wrong filing decisions.

**Recommendation:** This should be computed on-demand from the consolidation's `consolidated_into` chain, cached with a short TTL (30s), not maintained as a persistent event-driven projection. The recursive CTE mentioned in Section 3.7 ("rebuildable via PostgreSQL recursive CTE") should be the primary source, with Redis as a cache, not the other way around.

---

## 7. Migration Feasibility

### 7.1 Current State vs. Target State

**Critical observation:** The current codebase is a traditional monolith with:
- Single FastAPI application
- Single PostgreSQL schema (no domain isolation)
- No NATS, no event bus of any kind
- No CQRS — Redis is used for caching, not as a read store
- No event-driven intelligence — engines are called synchronously in a pipeline
- All models in a single `operational.py` file
- Direct ORM queries across all entities (cross-domain by design)

The document describes a **greenfield target architecture** while the codebase is a **brownfield monolith**. The four-line migration table in Section 11.2 ("Phase 1: Now, Phase 2: When needed") is dangerously incomplete.

### 7.2 Realistic Migration Assessment

**Phase 0 (Missing from doc): Schema isolation within existing monolith**
- Move models to per-domain modules
- Create PostgreSQL schemas per domain
- Migrate tables into schemas
- Remove cross-schema ORM relationships
- **Risk: HIGH** — this touches every query in the application
- **Duration: 4-8 weeks** for an experienced team

**Phase 1: Introduce event bus**
- Add NATS JetStream to Docker Compose
- Wrap existing write operations with event emission
- Build the Cache Projector to populate Redis from events
- Keep existing synchronous read paths as fallback
- **Risk: MEDIUM** — dual-write consistency issues
- **Duration: 6-10 weeks**

**Phase 2: CQRS read path migration**
- Switch read APIs from PostgreSQL to Redis
- Keep PostgreSQL as write-only
- Implement reconciliation checks
- **Risk: MEDIUM** — subtle inconsistency bugs
- **Duration: 4-6 weeks**

**Phase 3: Intelligence event decoupling**
- Replace synchronous engine pipeline with event-driven choreography
- Each intelligence engine subscribes to trigger events
- **Risk: HIGH** — changes the entire intelligence flow
- **Duration: 6-10 weeks**

**Total realistic timeline: 5-8 months** of dedicated architecture work before the system matches the document's description. This is not unusual for a migration of this scope, but the document should acknowledge it.

### 7.3 Import Linter (Section 11.3)

The `import-linter` approach for boundary enforcement is **correct and necessary**. However:

- The `independence` contract type in `import-linter` doesn't allow even event schema imports between domains. The doc says event schemas should be importable via `__init__.py` (Section 11.3, Import Rules), but this requires a `layers` or `forbidden` contract, not `independence`.
- The SQL cross-schema check ("CI check scans SQL/ORM queries for cross-schema references") is described but not shown. This is the harder problem — ORM queries can cross schemas implicitly via relationship loading. A simple SQL scanner won't catch `selectin` or `joined` eager loads.

**Recommendation:** Build the schema isolation check as a custom SQLAlchemy event listener that logs cross-schema query paths in test mode, not a static analysis tool.

---

## 8. Simulation Architecture

### 8.1 Separation Principle

The simulation-as-consumer design (Section 9) is **excellent**. This is the pattern I wish every logistics platform I've worked on had used from the start. Specific wins:

- Simulation bots use the same APIs as humans — validates API design
- Simulation publishes only to `sim.*` — cannot corrupt clearance state
- Simulation subscribes to `clearance.*` read-only — gets real-time state awareness
- "Replace the bot with real EDI/API integration. Zero platform changes needed." — this is the right goal

### 8.2 Where the Separation Breaks Down

**CarrierBot interaction with clearance instructions (Section 9.2):** The CarrierBot "interacts with clearance instructions: if authority orders inspection, keeps HU under bond at customs facility." This means the CarrierBot subscribes to `clearance.adjudication.decision` and modifies its behavior based on authority decisions. In production, this is the carrier's own system making these decisions — but the clearance platform needs to **know** whether the carrier acknowledged the hold instruction.

**Missing: Carrier acknowledgment loop.** In real ACE operations, when CBP issues a hold, the carrier must acknowledge and confirm the cargo is physically secured. The architecture has no event for `CarrierAcknowledgedHold` or `CarrierConfirmedInspection`. Without this, the platform can't distinguish between "carrier received the hold instruction" and "carrier is actually holding the cargo."

**DemurrageActor (Section 9.2):** This simulation actor calls `POST /financial/{hu_id}/demurrage` — but this endpoint doesn't exist in the Financial Settlement API surface (Section 3.11). The D&D calculation should be a platform concern (triggered by time-based events), not a simulation actor. In production, no one "calls" a demurrage endpoint — D&D accrues automatically.

**Recommendation:** Move D&D calculation to a periodic platform job (hourly tick → compute D&D for all in-port cargo → emit `DemurrageAccruing` events). Remove the DemurrageActor from the simulation.

### 8.3 Simulation Clock

The simulation uses a `sim.clock.tick` event (Section 4.1) for time advancement. The document doesn't describe how domain services handle simulated time vs. real time. If the simulation accelerates time (e.g., 1 hour = 1 real minute), how do time-dependent features work?
- GO deadline computation (15 days from arrival)
- ISF deadline (24 hours before vessel loading)
- Demurrage free time expiry
- Liquidation timeline (314 days post-entry)

If domain services use `NOW()` for these calculations, the simulation can't test them meaningfully. If domain services accept a `timestamp` parameter, that's a simulation concern leaking into the platform.

**Recommendation:** Use a `TimeService` interface that returns the current time. In production, it returns `NOW()`. In simulation, it returns the simulated clock time. This is a clean abstraction that maintains separation.

---

## 9. Security & Compliance

### 9.1 Audit Trail

**The architecture has no explicit audit trail.** This is a critical gap for a customs platform.

19 CFR Part 163 requires retention of all records relating to customs transactions for **5 years** from the date of entry. This includes:
- Every change to an entry filing
- Every authority decision and communication
- Every classification determination and its reasoning
- Every tariff calculation and its inputs
- Every document uploaded and its validation result
- Every hold/release and the resolution path

The event store (NATS JetStream) is **not sufficient** for audit:
- Max ages of 90-180 days (Section 4.2) violate 5-year retention
- NATS is not designed as an archival system
- Events don't capture "who" performed the action (no actor/user identity in event schema)

**Recommendation:**
1. Add `actor_id`, `actor_type` (user/system/bot), and `actor_ip` to the event metadata (Appendix A)
2. Implement an **Audit Event Sink** that archives all `clearance.*` events to immutable storage (S3 + Glacier with legal hold) with 5-year retention
3. Add a `compliance_audit` PostgreSQL schema with append-only tables for regulatory record-keeping
4. Implement tamper-evident logging (hash chain) for audit events

### 9.2 Data Sovereignty

The document mentions multi-jurisdiction support (US, EU, BR, IN, CN in Section 3.9) but doesn't address **data sovereignty requirements**:
- EU GDPR requires personal data to stay in the EU
- China's PIPL requires personal data of Chinese citizens to be stored in China
- India's DPDP Act has data localization requirements for certain data categories

For a customs platform handling shipper information, consignee details, and entity screening results across jurisdictions, this is a real concern.

**Recommendation:** Add a **data residency** section to the architecture describing how per-jurisdiction data is isolated. At minimum, the declaration and entity data for EU shipments must be processable in EU infrastructure.

### 9.3 Entity Screening Data Security

The DPS (Denied Party Screening) data (Section 3.3) includes matches against SDN, Entity List, and UFLPA lists. False positive matches are common (40%+ in my experience), and the screening results contain sensitive national security information.

**Missing:** Access controls on screening results. Not every user should see DPS match details. The broker needs to know there's a match; the compliance officer needs the details. The shipper should never see the match score or matched lists.

---

## 10. Anti-Patterns & Technical Debt Risks

### 10.1 Distributed Monolith Risk

The architecture describes 13 domains communicating via events, but in the modular monolith phase (Section 11.1), they're all in one process sharing one database connection pool, one Redis connection, and one NATS connection. If the event bus is introduced without proper isolation:
- A slow intelligence engine (LLM call) blocks event processing for all domains
- A database lock in one domain's schema blocks writes in another domain
- A Redis connection exhaustion from the Cache Projector starves API reads

**This is the textbook distributed monolith anti-pattern** — you've added the complexity of event-driven architecture (eventual consistency, event ordering, replay) without the isolation benefits (independent deployment, scaling, failure domains).

**Mitigation:** Use separate thread pools/async task queues per domain within the monolith. Each domain gets its own database session factory with connection limits. The Cache Projector runs in a separate process (as shown in Section 11.1) with its own connections.

### 10.2 Event Schema Evolution

Appendix A shows `metadata.version: 1` but the document doesn't describe a schema evolution strategy. When `ClassificationProduced` gains a new field in v2:
- Do old consumers break?
- How does the projector handle v1 and v2 events in the same stream?
- Is there a schema registry?

**Recommendation:** Adopt a schema registry (Protobuf with buf.build, or JSON Schema with a registry service). Enforce backward compatibility: new fields are optional, old fields are never removed (only deprecated). The CI fitness function in Section 11.3 should include schema compatibility checking.

### 10.3 God Entity: Shipment

Despite the document's correct identification of shipment as a "convenience abstraction," the `Shipment` entity (Section 3.5) accumulates data from 5 other domains: HU state (Cargo), classification/tariff (Trade Intelligence), compliance (Compliance), financial (Financial), and documents (Documents). The Redis cached view (Section 6.2) for a shipment is a massive denormalized JSON.

This is a **read-side god entity**. Any change in any of the 5 source domains requires updating the shipment cache. The Cache Projector becomes a bottleneck because every domain's events potentially update every shipment.

**Recommendation:** Instead of one giant shipment cache entry, use **composed views** — the UI queries `entity:shipment:{id}` for basic shipment data, `entity:shipment:{id}:intelligence` for classification/tariff, `entity:shipment:{id}:compliance` for compliance status, etc. Each sub-view is updated independently by the relevant domain's events. The frontend composes them. This distributes the Cache Projector's write load.

### 10.4 Document Storage Anti-Pattern

Section 3.13: `content_base64: text — Document content stored as base64-encoded string`. Storing document content as base64 in PostgreSQL:
- Increases document size by 33% (base64 encoding overhead)
- Causes TOAST table bloat (large values are stored out-of-line)
- Makes database backups, vacuuming, and replication significantly heavier
- Cannot leverage CDN caching or content-addressable storage

The document acknowledges this ("future migration to object storage is a known evolution path") but this should be Phase 0, not Phase N. A customs platform's document volume is non-trivial — commercial invoices, packing lists, certificates of origin, BOLs, AWBs, PGA certificates, etc. A single entry can have 5-15 documents.

**Recommendation:** Use S3-compatible object storage from day one. Store a `storage_url` in PostgreSQL. Serve via pre-signed URLs with TTL. This is a standard pattern and there's no reason to start with base64 in PostgreSQL.

---

## 11. What's Missing

### 11.1 Party Management Domain

The architecture has no **party/entity management domain**. Customs operations involve multiple parties:
- Importers of Record (IOR)
- Consignees
- Manufacturers
- Sellers/Buyers
- Customs brokers (modeled in Section 3.8, but only the broker entity)
- Carriers
- Freight forwarders
- Notify parties

These parties have relationships, compliance histories, trust scores, and regulatory obligations (power of attorney, continuous bonds). The current design scatters party data across domains (shipper info on Shipment, broker on Declaration, carrier on Consolidation).

**This is a core domain.** In every customs system I've built, Party Management is one of the first 5 domains established. It's the shared reference for entity screening (DPS matches against party records), bond sufficiency (bonds are per-importer), broker licensing (district restrictions), and carrier compliance.

### 11.2 Trade Lane / Corridor Analytics Domain

The architecture mentions trade lanes in the API surface (`POST /trade-lanes/compare`, Section 3.2) but has no domain for **trade lane intelligence**. In real customs operations, corridor-level analytics are critical:
- Duty/landed cost trends by corridor over time
- Clearance time distributions by port of entry
- Hold/exam rates by origin-destination pair
- Carrier performance metrics (D&D rates, transit time reliability)
- Seasonal patterns (Chinese New Year, Black Friday import surges)

This is distinct from Regulatory Intelligence (which monitors regulatory changes) and Trade Intelligence (which computes individual tariffs). It's a **decision support** domain that helps importers choose optimal routes, ports, and carriers.

### 11.3 Communication / Notification Domain

The architecture has `BrokerMessage` in Declaration Management (Section 3.8) but no general-purpose communication domain. Real customs operations involve notifications to:
- Shippers (shipment status updates, hold notifications, document requests)
- Brokers (queue updates, deadline warnings, authority responses)
- Importers (duty estimates, payment reminders, compliance alerts)
- Carriers (hold instructions, release notifications)

These should flow through a unified communication domain that handles channel routing (email, portal, EDI, SMS), template management, delivery tracking, and audit trail. The current design buries communications inside Declaration Management, which is incorrect — not all communications are declaration-related.

### 11.4 Idempotency Token Management

The idempotency strategy (Section 6.4) describes `processed_events` tables per schema, but doesn't address **API-level idempotency**. When a broker double-clicks "Submit Entry," the system must not create two declarations. Standard practice:
- Accept an `Idempotency-Key` header on all POST endpoints
- Store the key + response for 24 hours
- Return the cached response for duplicate requests

This is especially critical for financial endpoints (duty payment, bond creation).

### 11.5 Observability Architecture

The document doesn't describe:
- Distributed tracing (the `correlation_id` in Appendix A is a start, but how is it propagated through NATS?)
- Metrics collection (Prometheus? NATS built-in metrics?)
- Log aggregation strategy
- Health check contracts beyond `/health`
- Alerting strategy (the DLQ alert in Section 6.4 is one case)

For a production customs platform, observability is not optional — it's a regulatory requirement (you must be able to demonstrate system integrity to CBP during an audit).

### 11.6 Disaster Recovery / Business Continuity

No mention of:
- RPO/RTO targets
- Database backup strategy
- NATS cluster recovery procedures
- Redis data loss recovery (what happens if Redis loses all data?)
- Failover strategy for the single-process modular monolith

For a customs system, losing data means losing the ability to prove compliance. CBP can assess penalties for missing records. This needs to be a first-class architectural concern.

---

## Summary of Recommendations

| Priority | Recommendation | Section |
|----------|---------------|---------|
| **CRITICAL** | Add 5-year audit trail with immutable storage | 9.1 |
| **CRITICAL** | Fix NATS JetStream fsync configuration for data durability | 2.2 |
| **CRITICAL** | Replace single Cache Projector with distributed/redundant design | 3.1 |
| **CRITICAL** | Redesign movement event cascade to prevent event storms | 2.4 |
| **HIGH** | Merge Cargo & HUs + Consolidation into Physical Cargo domain | 1.2 |
| **HIGH** | Add Party Management domain | 11.1 |
| **HIGH** | Move document storage to object storage (S3) | 10.4 |
| **HIGH** | Add a realistic migration plan with timelines | 7.2 |
| **HIGH** | Implement hybrid state-transfer + notification events | 2.3 |
| **HIGH** | Add schema evolution strategy with registry | 10.2 |
| **MEDIUM** | Add missing events (DeclarationReadyToFile, BondExhausted, etc.) | 2.5 |
| **MEDIUM** | Split Financial Settlement into sub-domains internally | 1.2 |
| **MEDIUM** | Add TTL strategy for Redis cached entities | 3.3 |
| **MEDIUM** | Extract transit_history from JSONB to separate table | 4.1 |
| **MEDIUM** | Add data sovereignty/residency design | 9.2 |
| **MEDIUM** | Add Communication/Notification domain | 11.3 |
| **LOW** | Add version prefix to NATS subject hierarchy | 2.1 |
| **LOW** | Add Trade Lane/Corridor Analytics domain | 11.2 |
| **LOW** | Add observability architecture section | 11.5 |
| **LOW** | Add disaster recovery section | 11.6 |

---

## Final Assessment

This is a **strong domain-driven architecture** designed by someone who understands both the customs domain and modern distributed systems patterns. The domain boundaries are largely correct, the event choreography model is sound, and the simulation-as-consumer principle is excellent.

The main risks are:
1. **The gap between document and reality** — the current codebase needs 5-8 months of migration work to reach this architecture
2. **The movement event cascade** — this will become the scaling bottleneck and needs redesign before implementation
3. **Operational resilience** — the single Cache Projector, lack of audit trail, and absence of DR planning are gaps that would be showstoppers in a regulated environment
4. **Regulatory compliance** — a customs platform cannot launch without proper audit trails and data retention

I would approve this architecture for implementation with the CRITICAL and HIGH recommendations addressed, and a detailed migration plan that acknowledges the current state of the codebase.

---

*Review completed by Principal Enterprise Architect. Available for follow-up discussion on any section.*
