# Event-Driven Architecture Feasibility Check

**Author:** codebase-analyst
**Status:** Complete
**Date:** 2026-02-08

---

## 1. Executive Summary

This document maps all 19 simulation actors against the proposed `EventDrivenActor` pattern from `docs/architecture-event-driven-design.md`. For each actor, I evaluate six dimensions:

1. **Conversion difficulty** — clean / needs adaptation / significant rework
2. **Events consumed and produced**
3. **Transaction boundary issues** — multi-table atomic writes
4. **Time-dependent logic** — polling intervals, time-based decisions
5. **JSONB mutation patterns** — race conditions the event model doesn't address
6. **Cross-actor dependencies** — reads state written by other actors

**Overall verdict:** 12 of 19 actors can be cleanly or near-cleanly converted. 5 need moderate adaptation. 2 require significant rework. The design is sound but has **four systemic issues** that affect multiple actors and must be addressed in the architecture:

1. **Multi-table atomicity** in broker_sim and cbp_authority (write Shipment + EntryFiling + BrokerAssignment in one transaction)
2. **Time-dependent re-evaluation** in 7 actors (carrier, resolution, pga, demurrage, broker_sim, isf, cage)
3. **JSONB append races** on `events` column (12 actors append to the same JSONB array)
4. **Enrichment actors that don't change status** need a different trigger model (no status event to consume)

---

## 2. Actor-by-Actor Analysis

### 2.1 ShipperActor — `shipper.py:43-110`

**Conversion: Clean (no change needed)**

The shipper is a pure producer. It creates shipments on a Poisson timer and publishes `shipment_created`. The design correctly identifies it as timer-driven, unchanged.

| Dimension | Assessment |
|-----------|-----------|
| Events consumed | None (timer-driven) |
| Events produced | `ShipmentCreated` (already publishes `shipment_created`) |
| Transaction boundary | Multi-table: INSERT `Shipment` + `Order` + `OrderLineItem` with circular FK. **Atomic requirement:** Order and shipment must be linked in same transaction (lines 56-70). No issue — event published after commit. |
| Time-dependent logic | Poisson arrival rate only — no elapsed-time checks |
| JSONB mutations | None — creates fresh records |
| Cross-actor deps | None |

**Issues:** None. Publishes event after `session.commit()` (line 103) — already compatible with outbox pattern.

---

### 2.2 PreClearanceAdapter — `preclearance.py:33-96`

**Conversion: Clean**

| Dimension | Assessment |
|-----------|-----------|
| Events consumed | `ShipmentCreated` (currently polls `status == "booked"`) |
| Events produced | `ShipmentInTransit` or `ShipmentHeld` |
| Transaction boundary | Single table (Shipment). Clean. |
| Time-dependent logic | None — processes immediately |
| JSONB mutations | Writes to `events` via `transition()` |
| Cross-actor deps | None |

**Issues:** Already has `BATCH_LIMIT = 50` (line 44). In event-driven mode, processes one shipment per event — natural batch of 1. Clean conversion.

---

### 2.3 ConsolidatorActor — `consolidator.py:37-222`

**Conversion: Needs Adaptation**

| Dimension | Assessment |
|-----------|-----------|
| Events consumed | Design says `ShipmentCreated`, but actor actually polls `status == "in_transit"` with `consolidation_id IS NULL` (line 47-53). **Mismatch:** Should consume `ShipmentInTransit`, not `ShipmentCreated`. |
| Events produced | `ConsolidationCreated` (already publishes `consolidation_created`) |
| Transaction boundary | **Multi-table:** Creates `Consolidation` record + updates N `Shipment` records (sets `consolidation_id`, updates `references`, appends `events`). All must be atomic — a consolidation with unlinked children is corrupt data. (lines 127-194) |
| Time-dependent logic | None — processes immediately when group threshold met |
| JSONB mutations | Writes `references` and `events` on every child shipment (lines 176-194). **Race risk:** If carrier or another actor is simultaneously processing the same `in_transit` shipment and writing `references`, last-write-wins on the JSONB column. |
| Cross-actor deps | Reads `transport_mode`, `carrier`, `tracking_number` — all set by shipper at creation, stable. |

**Issues:**
1. **Event type mismatch** — design says `ShipmentCreated`, actor needs `ShipmentInTransit` (after preclearance transitions booked→in_transit).
2. **Grouping logic** — actor groups N shipments by (mode, origin, dest, carrier). In event-driven mode, it receives one event at a time. Must accumulate events and periodically flush groups. This is a **buffered consumer** pattern not covered in the design. Options: (a) keep a timer-based tick that checks accumulated candidates, or (b) use a Redis-based accumulation buffer. Either way, this isn't a simple event→process→emit flow.
3. **Multi-table atomicity** — creating a Consolidation + linking N shipments must stay in one transaction.

---

### 2.4 CarrierActor — `carrier.py:42-376`

**Conversion: Significant Rework**

This is the most complex actor. Two separate processing phases with fundamentally different patterns.

| Dimension | Assessment |
|-----------|-----------|
| Events consumed | `ShipmentInTransit` (for arrival), `ShipmentCleared` (for delivery), `TransitHoldResolved` (for resume) |
| Events produced | `ShipmentArrivedAtCustoms`, `ShipmentDelivered`, `TransitHold`, `ConsolidationUpdated` |
| Transaction boundary | **Per-shipment commits** (lines 235, 314) — already the target pattern, but does multi-entity work: updates `Shipment` + advances `Consolidation` in separate commits within the same method (lines 258-268). |
| Time-dependent logic | **Heavy.** Both phases check elapsed time: `_process_in_transit` checks `sim_now - shipment.created_at` against `transit_days` (line 78-81). `_process_cleared` checks `sim_now - cleared_at` against `inland_days` (line 293-298). Both use randomized durations per shipment. |
| JSONB mutations | Writes `references`, `events`, `waypoints` — all three JSONB columns in a single shipment processing (lines 105-115, 218-230, 306-313). High mutation surface. |
| Cross-actor deps | Reads `references.transit_hold_active` (set by itself on a previous tick, line 95). Reads `consolidation_id` (set by consolidator, line 258). |

**Issues:**
1. **Time-dependent re-evaluation is the core challenge.** The design proposes delayed re-emission (Section 5.5), but the carrier's logic is: "receive `ShipmentInTransit` event → calculate random transit_days → if elapsed < required, schedule re-check." The problem: `transit_days` includes a random delay injection (lines 83-88) that's different each time. If the event is re-emitted and the actor recalculates, it gets a **different random duration**. The actor must persist the calculated transit time on the first evaluation, or the delayed re-emission will produce non-deterministic behavior.
2. **Intermediate regulatory events** — the carrier generates transit touchpoint events (lines 117-170) including transit holds. This is a complex sub-workflow that produces holds, which trigger the resolution actor. In event-driven mode, a single `ShipmentInTransit` event can produce either `ShipmentArrivedAtCustoms` OR `TransitHold` — the consumer group must route correctly.
3. **Consolidation advancement** — after transitioning a shipment to `at_customs`, the carrier advances the parent consolidation (lines 258-268). This is a **cross-entity side effect** within the same event handler. The consolidation is fetched with `FOR UPDATE SKIP LOCKED` and transitioned. In event-driven mode, this should be a separate event (`ShipmentArrivedAtCustoms` → consolidation projector updates consolidation), not an in-handler side effect.
4. **Per-shipment commits** — the carrier already commits per-shipment (line 235), but publishes events AFTER the commit (lines 240-252). This is correct for the outbox pattern but means the event publish happens outside the transaction — if it fails, the shipment advanced but no event was published. This is the exact scenario the outbox pattern solves.

---

### 2.5 CustomsActor — `customs.py:52-481`

**Conversion: Clean**

| Dimension | Assessment |
|-----------|-----------|
| Events consumed | `ShipmentArrivedAtCustoms` (currently polls `status == "at_customs"`) |
| Events produced | `ShipmentCleared` (STP path), `ShipmentHeld` (various hold types), `ShipmentInspection` |
| Transaction boundary | Single table (Shipment). All mutations are on the shipment record. Clean. |
| Time-dependent logic | None — processes immediately on arrival. STP minutes calculation is cosmetic (description text), not a real delay. |
| JSONB mutations | Appends to `events` via `transition()`. Writes `hold_type`, `codes`. Moderate mutation surface but all within a single-shipment scope. |
| Cross-actor deps | Reads `codes.misclassified` (set by shipper at creation). Reads queue depth for backlog calculation (line 73-76) — **this requires a count query** (`SELECT count(*) WHERE status='at_customs'`). In event-driven mode, the actor doesn't have visibility into queue depth unless the read model provides it. |

**Issues:**
1. **Backlog detection** — `_process_at_customs` counts the total `at_customs` queue to detect backlogs (lines 73-76) and adjusts STP processing time. In event-driven mode, each event is processed independently. The actor would need to query the read model for current queue depth, or the Dashboard Projector could publish a `QueueDepthChanged` event. This is a minor adaptation — the backlog check is cosmetic (only affects STP description timing), not correctness-critical.
2. **Two processing phases** — `_process_at_customs` and `_process_inspections` are currently in one transaction. In event-driven mode, they become separate subscriptions: `ShipmentArrivedAtCustoms` for the first, and a new `ShipmentInspection` event for the second. The design covers this correctly.

---

### 2.6 PGAActor — `pga.py:79-197`

**Conversion: Needs Adaptation**

| Dimension | Assessment |
|-----------|-----------|
| Events consumed | `ShipmentArrivedAtCustoms` (currently polls `status == "at_customs"`) |
| Events produced | `PGAReviewComplete` (approval event only — no status change), `ShipmentHeld` (on failure) |
| Transaction boundary | Single table (Shipment). Clean. |
| Time-dependent logic | **Heavy.** Reviews have agency-specific durations (e.g., FDA 1-3 days, EPA 2-5 days). The actor checks `sim_now - arrival_time` against `review_days` (lines 137-149). **Same re-evaluation problem as carrier.** |
| JSONB mutations | Appends to `events` (PGA approval events, lines 154-165). |
| Cross-actor deps | Reads `events` to find arrival time (line 113-114 in `_find_arrival_time`). This arrival event is written by the carrier actor. **If the event payload includes arrival time, this dependency is eliminated.** |

**Issues:**
1. **Time-dependent re-evaluation** — PGA reviews take 1-5 days. On receiving `ShipmentArrivedAtCustoms`, the actor must: (a) determine if PGA review is needed, (b) if yes, schedule a delayed re-check for after the review period. Same delayed re-emission pattern as carrier.
2. **Multiple agencies per shipment** — a single shipment may need review by FDA, EPA, AND CPSC. The actor iterates through agencies sequentially (lines 128-197). If one fails (hold), it breaks. In event-driven mode, this is fine — the handler processes all agencies for the single event.
3. **PGA approval doesn't change status** — approved PGA reviews just append an event (line 154-165), they don't transition the shipment. The customs actor is the one that clears it. This means PGA's approval event needs to be consumed by... nobody? The dashboard projector should pick it up, but there's no actor that reacts to PGA approval. This is actually correct — PGA approval is informational, customs clearing is the real transition.

---

### 2.7 CBPAuthorityActor — `cbp_authority.py:86-148`

**Conversion: Needs Adaptation**

| Dimension | Assessment |
|-----------|-----------|
| Events consumed | `EntryFiled` (from broker_sim), `CF28Response` (from broker_sim) |
| Events produced | `EntryAccepted`, `CF28Issued`, `CF29Issued`, `ExamScheduled` |
| Transaction boundary | **Multi-table:** Reads `EntryFiling` + `Shipment` together. Updates both: changes `EntryFiling.filing_status` AND transitions `Shipment.status`. (lines 113-114 for the shipment fetch, then `_accept_entry` etc. modify both). Must be atomic — an accepted entry with a shipment still at `entry_filed` is inconsistent. |
| Time-dependent logic | **Medium.** `_process_cf28_deadlines` checks elapsed time since CF-28 issuance. `_process_scheduled_exams` checks elapsed time since exam scheduling. Both need delayed re-emission. |
| JSONB mutations | Writes `authority_response` on EntryFiling. Writes `events` on Shipment via `transition()`. |
| Cross-actor deps | Reads `EntryFiling` records created by `broker_sim`. Reads `Shipment` associated with each filing. |

**Issues:**
1. **Cross-table atomicity** — the actor's `_accept_entry`, `_issue_cf28`, etc. methods update both `EntryFiling` and `Shipment` in the same transaction. The event payload must include enough state for both entities, or the handler must fetch the shipment by ID (single-row, no FOR UPDATE contention). The design's `EntryFiled` event payload includes `shipment_id` but not the full shipment state — the handler would need to `session.get(Shipment, shipment_id)`, which is a single-row fetch, acceptable.
2. **Four separate subscription types** — the actor processes submitted entries, CF-28 deadlines, scheduled exams, and protests. Each is a different event subscription. The `EventDrivenActor` pattern supports multiple subscriptions, but the handler routing logic needs to distinguish between them cleanly.

---

### 2.8 BrokerSimActor — `broker_sim.py:104-651`

**Conversion: Significant Rework**

The most complex actor in the system. 5 distinct processing methods, multi-table writes, protected broker logic.

| Dimension | Assessment |
|-----------|-----------|
| Events consumed | `ShipmentArrivedAtCustoms` (for auto-claim), `CF28Issued` (for response) |
| Events produced | `BrokerAssigned`, `EntryFiled`, `CF28ResponseSent` |
| Transaction boundary | **Heavily multi-table:** `_auto_claim_entries` creates `BrokerAssignment` + `EntryFiling` + updates `Shipment.entry_number` + optionally creates `Document` records + optionally transitions `Shipment.status` (lines 137-348). All in one transaction. Five tables touched atomically. |
| Time-dependent logic | **Heavy.** `_respond_to_cf28s` checks `days_elapsed = (sim_time - issued_at) / 86400` against a random response delay (lines 607-612). `_progress_checklists` may also have time logic. |
| JSONB mutations | Writes `broker_approval` on EntryFiling, `authority_response` on EntryFiling, `events` on Shipment, `checklist_state` on EntryFiling. High JSONB mutation surface across two tables. |
| Cross-actor deps | Reads `Shipment.status` (set by carrier/customs), `Broker` records (created by itself via `_ensure_demo_brokers`), `EntryFiling` records (created by itself), `Document` records. **Heavy self-dependency** — later methods in the same tick read records created by earlier methods. |

**Issues:**
1. **Five methods, three event sources** — `_ensure_demo_brokers` is a one-time setup (should be moved to initialization), `_auto_claim_entries` reacts to `ShipmentArrivedAtCustoms`, `_progress_checklists` and `_submit_ready_entries` react to internal state changes (checklist completion), `_respond_to_cf28s` reacts to `CF28Issued`. This actor needs to be **decomposed** into at least 2-3 event handlers or sub-actors.
2. **Protected broker logic** — the actor has complex logic for "protected brokers" (human-playable brokers) that receive assignments but don't auto-progress (lines 175-196, 231-284). This involves counting active entries per protected broker, diversifying lifecycle stages, creating documents at specific stages. This business logic is fine in event-driven mode but adds handler complexity.
3. **Self-referential pipeline** — `_auto_claim_entries` creates entries in `draft` status, `_progress_checklists` moves them to `pending_broker_approval`, `_submit_ready_entries` submits them. Currently these run sequentially in one tick. In event-driven mode, each step would need to produce an internal event that triggers the next step. This creates an internal event chain: `ShipmentArrivedAtCustoms` → claim → `EntryDraftCreated` → progress → `EntryReadyForApproval` → submit → `EntryFiled`. Three internal events for what's currently one tick.
4. **Atomic multi-table writes** — `_auto_claim_entries` writes to 4-5 tables in one transaction. In event-driven mode with single-row processing, the handler must still do all of this atomically. A single `session.begin()` around the handler's DB writes is sufficient — the issue isn't event-driven incompatibility, it's just that the event payload must include enough context to avoid re-querying. The `ShipmentArrivedAtCustoms` event should include the full shipment state (which the design specifies), and the handler fetches brokers from DB (acceptable, low contention on the `brokers` table).

---

### 2.9 ResolutionActor — `resolution.py:43-352`

**Conversion: Needs Adaptation**

| Dimension | Assessment |
|-----------|-----------|
| Events consumed | `ShipmentHeld` |
| Events produced | `HoldResolved`, `TransitHoldResolved`, `HoldEscalated` |
| Transaction boundary | Single table (Shipment). Clean. |
| Time-dependent logic | **Heavy.** Core logic is time-based: checks `sim_now - hold_time` against `review_days` (lines 99-110). Transit holds have different time ranges (lines 220-226). |
| JSONB mutations | Appends to `events` (escalation events, lines 147-160). Modifies `references` (clears transit hold markers, lines 241-245). |
| Cross-actor deps | Reads `events` to find `status_held` timestamp (lines 343-352). Reads `references.transit_hold_active/territory/reason` (set by carrier, lines 74-78, 207-209). |

**Issues:**
1. **Time-dependent re-evaluation** — same pattern as carrier/PGA. On receiving `ShipmentHeld`, must schedule delayed re-check based on review period. Must persist the calculated `review_days` on first evaluation to avoid non-deterministic re-rolls.
2. **Hold-type-specific profiles** — `HOLD_TYPE_PROFILES` provide per-hold-type review durations and resolution rates (lines 88-97). The event payload should include `hold_type` (which the design specifies) so the handler can look up the correct profile.
3. **Transit hold resolution** — returns shipment to `in_transit` (not `cleared`), which triggers the carrier to resume processing. The event chain is: `ShipmentHeld` → resolution → `TransitHoldResolved` → carrier resumes. The design covers this flow.

---

### 2.10 CageActor — `cage.py:78-299`

**Conversion: Needs Adaptation**

| Dimension | Assessment |
|-----------|-----------|
| Events consumed | `ShipmentHeld`, `ShipmentInspection` (for intake), `ShipmentCleared`, `ShipmentDelivered` (for release) |
| Events produced | `CageIntake`, `CageReleased`, `GODeadlineWarning` |
| Transaction boundary | Single table (Shipment). Clean. |
| Time-dependent logic | **Heavy.** `_process_dwell` updates dwell days and storage costs every tick based on `sim_now - intake_time` (lines 181-255). GO deadline warnings are time-based. |
| JSONB mutations | **Heavy.** Writes entire `cage_status` JSONB (lines 130-151). Appends to `events` (lines 170-178). `_process_dwell` updates `cage_status` incrementally every tick (dwell_days, storage_cost, go_days_remaining). |
| Cross-actor deps | Reads `Shipment.status` to determine intake eligibility and release eligibility. |

**Issues:**
1. **Dwell tracking is inherently poll-based** — `_process_dwell` updates storage costs and dwell days for ALL caged shipments every tick. This isn't event-triggered — it's a continuous accumulation. In event-driven mode, options: (a) keep a timer-based tick for dwell updates (the design doesn't address this), (b) calculate dwell on-demand when cage_status is read (lazy computation). **Option (b) is strongly preferred** — store `intake_time` and `storage_rate_per_day`, compute dwell/cost at read time. Eliminates the need for periodic updates entirely.
2. **Three sub-methods = three event types** — intake (reacts to hold/inspection), dwell (timer), release (reacts to cleared/delivered). These should be separate handlers or one handler with routing.
3. **GO deadline warnings** — currently generated by dwell processing when `go_days_remaining < 5`. In event-driven mode with lazy dwell computation, GO warnings could be scheduled via delayed re-emission when intake occurs (schedule a check for 10 days out).

---

### 2.11 ComplianceEngineActor — `compliance_engine.py:43-74`

**Conversion: Needs Adaptation**

| Dimension | Assessment |
|-----------|-----------|
| Events consumed | Any non-`in_transit` status event where `analysis IS NULL` |
| Events produced | `AnalysisGenerated` |
| Transaction boundary | Single table (Shipment). Clean. |
| Time-dependent logic | None |
| JSONB mutations | Writes entire `analysis` JSONB (line 67). **No locking (no FOR UPDATE)** — current code has a race condition where two compliance engine ticks could both find the same shipment with `analysis IS NULL` and both generate analysis. In event-driven mode, consumer group guarantees single processing, **fixing** this race. |
| Cross-actor deps | Reads shipment state to generate analysis. No dependency on other actors' JSONB fields. |

**Issues:**
1. **Trigger model** — the actor's trigger isn't a specific status change, it's `analysis IS NULL AND status != 'in_transit'`. In event-driven mode, it should consume `ShipmentArrivedAtCustoms` (the first non-in_transit status). But what if the shipment goes through multiple status changes before compliance processes it? Consumer group deduplication handles this — the first event triggers analysis, subsequent events for the same shipment find `analysis IS NOT NULL` and skip. Actually, the design says "subscribes to all non-in_transit events" — this means it might receive `ShipmentCleared` for a shipment it already analyzed. The handler must be idempotent: check `analysis IS NOT NULL` before generating. This is already the current pattern (lines 51-53).
2. **Event-driven actually improves this actor** — eliminates the race condition and reduces unnecessary DB queries.

---

### 2.12 ShipperResponseActor — `shipper_response.py:32-74`

**Conversion: Clean**

| Dimension | Assessment |
|-----------|-----------|
| Events consumed | `CF28Issued`, `HoldEscalated`, or any event with pending communications |
| Events produced | None (writes responses directly to shipment events) |
| Transaction boundary | Multi-table: updates `Shipment.events` AND creates `Document` records (lines 55-63). Must be atomic — a response event without its attached documents is inconsistent. But these are INSERT + UPDATE in one transaction, manageable. |
| Time-dependent logic | None — responds immediately |
| JSONB mutations | Appends to `events` (line 64-65). |
| Cross-actor deps | Reads `events` to find pending communications (line 39). These communications are written by customs, cbp_authority, and other actors. |

**Issues:**
1. **Full table scan eliminated** — the current actor scans ALL shipments (line 37, no WHERE clause). In event-driven mode, it only processes events that contain communications requiring response. **This is a major improvement** — the design correctly identifies this.
2. **Trigger identification** — the actor needs events that indicate "a communication was sent to the shipper." Currently it scans all events for pending comms. In event-driven mode, the actors that create communications (customs CF-28, CBP authority, etc.) should publish a `CommunicationSent` event or include a flag in their existing events.

---

### 2.13 TerminalActor — `terminal.py:35-109`

**Conversion: Clean**

| Dimension | Assessment |
|-----------|-----------|
| Events consumed | `ShipmentInTransit` (for berth delays, ocean only), `ShipmentCleared` (for chassis availability, ocean only) |
| Events produced | `TerminalBerthDelay`, `TerminalChassisShortage` (informational) |
| Transaction boundary | Single table (Shipment). Clean. |
| Time-dependent logic | None — probabilistic injection, not time-elapsed checks |
| JSONB mutations | Appends to `events`. One-time write per shipment (checks for existing event to avoid duplicates, line 68). |
| Cross-actor deps | Reads `references.container_number`, `references.vessel` (set by consolidator/carrier). Stable by the time terminal processes. |

**Issues:** None significant. Clean conversion. Mode filter (`transport_mode == "ocean"`) should be included in the event subscription filter or handler guard.

---

### 2.14 DemurrageActor — `demurrage.py:34-150`

**Conversion: Needs Adaptation**

| Dimension | Assessment |
|-----------|-----------|
| Events consumed | The design says `CageIntake` but the actor actually processes ocean/air shipments in `at_customs`/`inspection`/`held`/`cleared` statuses (lines 53-56). It's not cage-dependent. |
| Events produced | `DemurrageAccruing` (informational) |
| Transaction boundary | Single table (Shipment). Clean. |
| Time-dependent logic | **Heavy.** Core logic calculates `days_since_arrival` from events timeline (lines 89-96), then computes demurrage charges based on free days, daily rates, and carrier policies. Recalculates every tick. |
| JSONB mutations | Writes `financials` (demurrage charges, lines 104-112). Appends to `events` (demurrage alerts, lines 116-128). |
| Cross-actor deps | Reads `events` to find arrival timestamp (lines 83-88). Reads `cargo_detail` for container type (line 99). Reads `carrier` for policy lookup (line 77). |

**Issues:**
1. **Event type mismatch** — design says `CageIntake`, but demurrage calculates for ALL ocean shipments at customs/inspection/held/cleared, not just caged ones. Should consume `ShipmentArrivedAtCustoms` for ocean shipments.
2. **Recurring accumulation** — like cage dwell tracking, demurrage accrues over time. Every tick recalculates charges. **Same lazy computation recommendation as cage:** store arrival time and carrier policy, compute charges at read time. Or use delayed re-emission: on `ShipmentArrivedAtCustoms`, schedule daily re-checks after free days expire.
3. **`last_dnd_calculation` guard** — the actor uses a per-tick timestamp guard to avoid double processing (lines 67-69). In event-driven mode with proper consumer group acknowledgment, this guard is unnecessary but harmless.

---

### 2.15 DisruptionActor — `disruption.py:45-159`

**Conversion: Clean (keep timer-based)**

| Dimension | Assessment |
|-----------|-----------|
| Events consumed | None (timer-driven, random injection) |
| Events produced | `DisruptionEvent` (informational) |
| Transaction boundary | Single table (Shipment), but affects multiple shipments per disruption (up to 10, line 120). |
| Time-dependent logic | Probabilistic trigger only (`rng.random() > disruption_probability`, line 51) |
| JSONB mutations | Appends to `events` for each affected shipment (lines 130-137). Individual `FOR UPDATE SKIP LOCKED` per shipment (line 121-124). |
| Cross-actor deps | Reads `status` and `destination` to find active ports and affected shipments. |

**Issues:** Design correctly identifies this as timer-based (unchanged). The only concern is that it fetches individual shipments with `FOR UPDATE SKIP LOCKED` (line 121) — in event-driven mode, it should use the read model to find affected shipment IDs, then fetch-and-update individual records.

---

### 2.16 ExceptionsActor — `exceptions.py:53-106`

**Conversion: Clean (keep timer-based)**

| Dimension | Assessment |
|-----------|-----------|
| Events consumed | Design says timer-based (unchanged). Could alternatively consume `ShipmentInTransit` and `ShipmentCleared` for targeted injection. |
| Events produced | Exception events (informational, appended to shipment events) |
| Transaction boundary | Single table (Shipment). Clean. |
| Time-dependent logic | None — probabilistic injection |
| JSONB mutations | Appends to `events`. One-time write per shipment (checks for existing exception, line 82-83). |
| Cross-actor deps | None significant |

**Issues:** None. Clean timer-based actor. Could optionally convert to event-driven for better targeting, but timer-based is simpler and appropriate.

---

### 2.17 FinancialActor — `financial.py:37-117`

**Conversion: Clean**

| Dimension | Assessment |
|-----------|-----------|
| Events consumed | `ShipmentArrivedAtCustoms` (for MPF/HMF), `ShipmentInspection` or `ShipmentHeld` (for exam fees), `ShipmentCleared` (for broker fees) |
| Events produced | None currently (writes directly to `financials` JSONB) — should produce `FeesCalculated` |
| Transaction boundary | Single table (Shipment). Clean. |
| Time-dependent logic | None — calculates once and guards with `mpf_calculated` flag (line 64) |
| JSONB mutations | Writes `financials` JSONB (lines 77-109). Guard flag prevents re-calculation. |
| Cross-actor deps | Reads `declared_value`, `transport_mode` — stable, set at creation. |

**Issues:**
1. **Idempotency via guard flag** — `mpf_calculated` flag (line 64) prevents re-processing. In event-driven mode, the handler might receive multiple events for the same shipment (e.g., `ShipmentArrivedAtCustoms` then `ShipmentCleared`). The guard flag handles this correctly — already idempotent.
2. **Three sub-methods = three event subscriptions** — entry fees (at_customs/cleared), exam fees (inspection/held), broker fees (at_customs/cleared). Each reacts to different status events.

---

### 2.18 DocumentsActor — `documents.py:65-103`

**Conversion: Clean**

| Dimension | Assessment |
|-----------|-----------|
| Events consumed | `ShipmentArrivedAtCustoms` (for customs doc validation) |
| Events produced | Document discrepancy events (informational) |
| Transaction boundary | Single table (Shipment). Clean. |
| Time-dependent logic | None — processes immediately |
| JSONB mutations | Appends to `events` (discrepancy events). One-time write per shipment (checks for existing doc event, line 90-91). |
| Cross-actor deps | Reads `origin` for USMCA validation. Reads `references` for document cross-checks. |

**Issues:** None significant. Clean conversion. Idempotent via existing event guard.

---

### 2.19 ISFActor — `isf.py:31-73`

**Conversion: Clean**

| Dimension | Assessment |
|-----------|-----------|
| Events consumed | `ShipmentCreated` (for late filing check, ocean booked), `ShipmentInTransit` (for data mismatch check, ocean) |
| Events produced | ISF events (informational, penalties, mismatches) |
| Transaction boundary | Single table (Shipment). Clean. |
| Time-dependent logic | Minor — checks if ISF was filed late relative to booking time. |
| JSONB mutations | Writes `references.isf_late_filing` (line 64). Appends to `events`. |
| Cross-actor deps | Reads `references` for ISF filing status. |

**Issues:** None significant. Mode filter (`transport_mode == "ocean"`) needed in subscription or handler guard.

---

## 3. Systemic Issues

### 3.1 JSONB `events` Append Race Condition

**Severity: HIGH — affects 12 of 19 actors**

The `events` JSONB column is an append-only array. Every actor that calls `transition()` or manually appends events does:

```python
events = list(shipment.events or [])
events.append({...})
shipment.events = events
flag_modified(shipment, "events")
```

In the current poll-and-lock model, `FOR UPDATE SKIP LOCKED` prevents concurrent writes to the same row. In event-driven mode with consumer groups, a single event goes to one consumer, but **different events for different actors may trigger concurrent writes to the same shipment's events column.**

Example: `ShipmentArrivedAtCustoms` goes to customs actor, PGA actor, financial actor, documents actor, terminal actor, and broker_sim. If they all process concurrently and append to `events`, last-write-wins — events get lost.

**Proposed solutions:**
1. **Serialized event writes** — use a dedicated "event appender" service that receives `AppendShipmentEvent` commands and serializes writes. Adds latency.
2. **PostgreSQL array append** — use `UPDATE shipments SET events = events || $1::jsonb WHERE id = $2` instead of read-modify-write. This is atomic at the DB level. **Recommended — simplest fix.**
3. **Separate events table** — move events out of the JSONB column into a proper `shipment_events` table with `(shipment_id, timestamp, event_type, data)`. Eliminates the race entirely and improves queryability. **Best long-term solution but largest migration effort.**

### 3.2 Time-Dependent Actors and Delayed Re-emission

**Severity: MEDIUM — affects 7 actors**

Actors with time-dependent logic: carrier, resolution, PGA, demurrage, cage (dwell), broker_sim (CF-28 response), cbp_authority (exam/CF-28 deadlines).

The design's delayed re-emission pattern (Section 5.5) works but has a critical issue with the **simulation clock**:

1. **Clock speed changes** — if `time_ratio` changes from 10x to 20x mid-simulation, all scheduled delays in the Redis sorted set are wrong. They were calculated based on the old ratio. A clock-speed-change event must trigger recalculation of all pending delays.
2. **Clock pause/resume** — if the simulation is paused, delayed events should not fire. The re-emission poller must check `clock.paused` before processing.
3. **Non-deterministic re-rolls** — carrier and resolution both use `self.rng.random()` to determine durations. On re-emission, the actor gets the same event but a different random number. **Solution:** Calculate and persist the required duration (and outcome probability) on first evaluation. Store in Redis alongside the delayed event.

**Alternative approach:** For actors where dwell/accumulation is the pattern (cage, demurrage), use **lazy computation** instead of periodic updates. Store the start timestamp and rate, compute the current value on demand. This eliminates 2 of the 7 time-dependent actors from needing delayed re-emission.

### 3.3 Multi-Table Atomicity

**Severity: MEDIUM — affects 3 actors**

| Actor | Tables in single transaction | Can decompose? |
|-------|------------------------------|---------------|
| broker_sim._auto_claim_entries | Shipment + EntryFiling + BrokerAssignment + Document (4-5 tables) | Partially — claim (assignment+entry) must be atomic, document creation can be separate |
| cbp_authority._process_submitted_entries | EntryFiling + Shipment | No — entry acceptance and shipment status must be consistent |
| consolidator.tick | Consolidation + N × Shipment | No — orphaned consolidation or unlinked children is corrupt |

In event-driven mode, handlers still use database transactions for their writes — the event triggers the handler, which opens a transaction, does its multi-table work, and commits. **This is not fundamentally incompatible with event-driven.** The key constraint is: the event publish must happen AFTER the transaction commits (outbox pattern), not within it.

### 3.4 Enrichment Actors Without Status Triggers

**Severity: LOW — affects 4 actors**

Actors that enrich shipments without changing status: compliance_engine, financial, documents, terminal. Their work is triggered by status changes but their output is JSONB field updates, not status transitions.

In event-driven mode, these actors consume status events but don't produce status events. They should produce domain-specific events (`AnalysisGenerated`, `FeesCalculated`, `DocumentDiscrepancyFound`, `BerthDelayReported`) that the dashboard projector consumes for read model updates.

---

## 4. Conversion Difficulty Summary

| Actor | Difficulty | Key Challenge |
|-------|-----------|---------------|
| shipper | **Clean** | None — pure producer |
| preclearance | **Clean** | None |
| consolidator | **Adaptation** | Grouping/buffering pattern, multi-table atomicity |
| carrier | **Significant Rework** | Time-dependent logic, intermediate events, consolidation side-effects, per-shipment commits with event publishing |
| customs | **Clean** | Backlog detection needs read model (minor) |
| pga | **Adaptation** | Time-dependent review periods |
| cbp_authority | **Adaptation** | Multi-table writes, 4 subscription types, time-dependent deadlines |
| broker_sim | **Significant Rework** | 5-method decomposition, self-referential pipeline, multi-table atomicity, protected broker complexity |
| resolution | **Adaptation** | Time-dependent resolution periods |
| cage | **Adaptation** | Dwell tracking (recommend lazy computation) |
| compliance_engine | **Clean** | Event-driven fixes existing race condition |
| shipper_response | **Clean** | Event-driven eliminates full table scan |
| terminal | **Clean** | None |
| demurrage | **Adaptation** | Recurring accumulation (recommend lazy computation) |
| disruption | **Clean** | Keep timer-based |
| exceptions | **Clean** | Keep timer-based |
| financial | **Clean** | Guard-flag idempotency already works |
| documents | **Clean** | Guard-flag idempotency already works |
| isf | **Clean** | None |

### Count: 12 Clean, 5 Adaptation, 2 Significant Rework

---

## 5. Design Corrections Needed

Based on this feasibility check, the event-driven design (`architecture-event-driven-design.md`) should be updated:

### 5.1 Event Catalog Corrections

| Actor | Design Says | Should Be |
|-------|------------|-----------|
| consolidator | Subscribes to `ShipmentCreated` | Should subscribe to `ShipmentInTransit` (actor processes `in_transit` shipments) |
| demurrage | Subscribes to `CageIntake` | Should subscribe to `ShipmentArrivedAtCustoms` (ocean filter) — demurrage is independent of caging |
| shipper_response | Subscribes to `CF28Issued`, `HoldEscalated` | Should subscribe to any event containing a communication requiring shipper response — need a `CommunicationSent` event type |

### 5.2 Missing Patterns

1. **Buffered consumer pattern** for consolidator — accumulate candidates, flush when group threshold met
2. **Lazy computation pattern** for cage dwell and demurrage accumulation — eliminate timer-based updates
3. **Atomic JSONB append** solution for the `events` column race condition
4. **Clock-aware delayed re-emission** — must handle pause/resume/speed changes
5. **Persisted duration calculation** — carrier and resolution must store their randomly-generated durations on first evaluation

### 5.3 Actor Decomposition

**BrokerSimActor** should be split into:
1. `BrokerClaimHandler` — consumes `ShipmentArrivedAtCustoms`, creates assignment + entry filing
2. `BrokerProgressHandler` — consumes `EntryDraftCreated` (internal), progresses checklists, submits entries
3. `BrokerCF28Handler` — consumes `CF28Issued`, generates response after delay

---

## 6. Recommendations for Migration Priority

Based on feasibility and impact, the recommended migration order:

### Phase 2a — Highest Value, Cleanest Conversion (3-4 days)
1. **customs** — eliminates the #1 contention hotspot, clean conversion
2. **compliance_engine** — fixes existing race condition, clean conversion
3. **shipper_response** — eliminates full table scan, clean conversion
4. **financial** — clean conversion, already idempotent
5. **documents** — clean conversion, already idempotent

### Phase 2b — High Value, Moderate Adaptation (3-4 days)
6. **pga** — high contention, needs delayed re-emission for review periods
7. **resolution** — high contention, needs delayed re-emission for hold reviews
8. **cage** — high contention, recommend lazy dwell computation

### Phase 2c — Complex Actors, Careful Rework (4-5 days)
9. **cbp_authority** — multi-table writes, multiple subscriptions
10. **carrier** — time-dependent, consolidation side-effects, most complex
11. **broker_sim** — decompose into 3 handlers, multi-table atomicity

### Phase 2d — Low-Priority Remaining (2 days)
12-19. consolidator, preclearance, terminal, demurrage, isf, disruption, exceptions — lower contention, some can stay timer-based

---

*This feasibility check is based on line-by-line analysis of all 19 actor files. File references are to the current codebase as of 2026-02-08.*
