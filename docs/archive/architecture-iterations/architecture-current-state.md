# Clearance Vibe Platform — Current Architecture Map

> Produced by **codebase-analyst** for the event-arch-design team.
> Date: 2026-02-08

---

## 1. System Overview

The Clearance Vibe platform is a **customs clearance simulation and management system** built on:

- **Backend**: FastAPI (Python 3.12, async/await, SQLAlchemy 2.x async)
- **Database**: PostgreSQL (JSONB-heavy schema, `pg_trgm` + `btree_gin` extensions)
- **Cache/PubSub**: Redis (dashboard caching, event bus pub/sub)
- **Frontend**: Vite + React (surfaces: broker, buyer, shipper, platform)
- **Vector DB**: Qdrant (for AI/embedding features)

### Execution Model

The simulation runs **19 concurrent actors** as `asyncio.Task` instances inside a single Python process. Each actor:
1. Runs an infinite tick loop at a configurable interval (sim-minutes)
2. Creates its own `AsyncSession` per tick via `session_factory()`
3. Uses `SELECT ... FOR UPDATE SKIP LOCKED` for row-level locking
4. Publishes events to a Redis-backed `EventBus` (fire-and-forget, no actor subscribes to events from other actors at runtime)
5. Commits its own transactions

The `SimulationCoordinator` manages actor lifecycle, clock, and SSE dashboard broadcast. A background `_dashboard_loop` periodically queries aggregate stats and pushes them to SSE subscribers.

---

## 2. All 19 Simulation Actors

### 2.1 ShipperActor (`shipper`)
- **What it does**: Generates new shipments using a Poisson arrival process modulated by business hours and seasonality. Randomly injects HS misclassifications and value errors.
- **Tables written**: `shipments` (INSERT), `orders` + `order_line_items` (INSERT, ~30% of shipments)
- **Tables read**: None (uses in-memory reference data)
- **Status transitions**: Creates shipments in `booked` status (no transition call — sets status directly on INSERT)
- **FOR UPDATE queries**: 0
- **Session pattern**: `async with self.get_session() as session:` — single session, single commit at end

### 2.2 ConsolidatorActor (`consolidator`)
- **What it does**: Groups `in_transit` shipments without a consolidation into MAWB/MBL/manifest records by (mode, origin, destination, carrier).
- **Tables written**: `consolidations` (INSERT), `shipments` (UPDATE: `consolidation_id`, `references`, `events`)
- **Tables read**: `shipments` (WHERE status='in_transit' AND consolidation_id IS NULL)
- **Status transitions**: None on shipments. Consolidation: created as `booked`, immediately set to `closed`.
- **FOR UPDATE queries**: 1 — locks `in_transit` shipments without consolidation_id
- **Session pattern**: `async with session.begin()` — single transaction

### 2.3 PreClearanceAdapter (`preclearance`)
- **What it does**: Runs the E0 PreClearanceAgent (entity screening, sanctions, UFLPA) on `booked` shipments. Transitions to `in_transit` (passed) or `held` (blocked).
- **Tables written**: `shipments` (UPDATE: status, events, hold_type)
- **Tables read**: `shipments` (WHERE status='booked')
- **Status transitions**: `booked` -> `in_transit` (via "platform" actor) or `booked` -> `held` (via "platform" actor)
- **FOR UPDATE queries**: 1 — locks `booked` shipments (LIMIT 50)
- **Session pattern**: `async with session.begin()` — single transaction

### 2.4 CarrierActor (`carrier`)
- **What it does**: Advances shipments through transit and final delivery. Handles intermediate regulatory events, transit holds, consolidation lifecycle advancement.
- **Tables written**: `shipments` (UPDATE: status, events, references, waypoints, entry_number), `consolidations` (UPDATE: status, events)
- **Tables read**: `shipments` (WHERE status='in_transit'), `shipments` (WHERE status='cleared'), `consolidations`
- **Status transitions**:
  - `in_transit` -> `at_customs` (carrier) — main arrival
  - `in_transit` -> `held` (carrier) — transit hold
  - `cleared` -> `delivered` (carrier)
  - Consolidation: `closed` -> `in_transit` -> `arrived` -> `deconsolidating` -> `deconsolidated`
- **FOR UPDATE queries**: 4 — in_transit shipments, cleared shipments, consolidation arrival, consolidation deconsolidation
- **Session pattern**: Individual `session.commit()` per shipment (NOT batched) — each shipment gets its own commit

### 2.5 CustomsActor (`customs`)
- **What it does**: The **primary tunable actor**. Screens `at_customs` shipments through STP fast-lane or manual review (UFLPA detention, physical inspection, HS reclassification, CF-28/CF-29 holds, entity screening, ADD/CVD). Resolves `inspection` shipments.
- **Tables written**: `shipments` (UPDATE: status, events, codes, financials, hold_type)
- **Tables read**: `shipments` (WHERE status='at_customs'), `shipments` (WHERE status='inspection'), count of at_customs for backlog detection
- **Status transitions**:
  - `at_customs` -> `cleared` (STP, manual clear, HS reclassification)
  - `at_customs` -> `held` (UFLPA, entity screening, CF-28/CF-29, ADD/CVD review)
  - `at_customs` -> `inspection` (physical inspection)
  - `inspection` -> `cleared` (passed)
  - `inspection` -> `held` (failed)
- **FOR UPDATE queries**: 2 — at_customs shipments, inspection shipments
- **Session pattern**: `async with session.begin()` — single transaction for ALL shipments in tick

### 2.6 PGAActor (`pga`)
- **What it does**: Reviews `at_customs` shipments for Partner Government Agency clearance (FDA, EPA, CPSC, NHTSA, APHIS, TTB). Applies time-based review windows and approval rates.
- **Tables written**: `shipments` (UPDATE: events, references, hold_type, status)
- **Tables read**: `shipments` (WHERE status='at_customs')
- **Status transitions**: `at_customs` -> `held` (PGA review failed)
- **FOR UPDATE queries**: 1 — at_customs shipments
- **Session pattern**: `async with session.begin()` — single transaction

### 2.7 ComplianceEngineActor (`compliance`)
- **What it does**: Populates the `analysis` JSONB column on shipments that have arrived anywhere in the pipeline. Generates classification, tariff, and compliance data.
- **Tables written**: `shipments` (UPDATE: analysis JSONB)
- **Tables read**: `shipments` (WHERE analysis IS NULL AND status != 'in_transit')
- **Status transitions**: None
- **FOR UPDATE queries**: 0 (no locking!)
- **Session pattern**: `async with session.begin()` — single transaction

### 2.8 ResolutionActor (`resolution`)
- **What it does**: Resolves `held` shipments after a configurable review period. Supports hold-type-specific profiles (e.g., UFLPA takes longer). Handles transit holds separately (returns to `in_transit` instead of `cleared`).
- **Tables written**: `shipments` (UPDATE: status, events, references, hold_type)
- **Tables read**: `shipments` (WHERE status='held')
- **Status transitions**:
  - `held` -> `cleared` (resolved)
  - `held` -> `in_transit` (transit hold resolved)
  - Escalation: stays `held`, appends event
- **FOR UPDATE queries**: 1 — held shipments
- **Session pattern**: `async with session.begin()` — single transaction

### 2.9 ShipperResponseActor (`shipper_response`)
- **What it does**: Generates realistic shipper responses to outbound communications after a realistic delay. Auto-attaches documents when shipper provides correct docs.
- **Tables written**: `shipments` (UPDATE: events), `documents` (INSERT — simulated doc attachments)
- **Tables read**: `shipments` (ALL — no status filter!)
- **Status transitions**: None
- **FOR UPDATE queries**: 0 (no locking!)
- **Session pattern**: Plain `session.commit()` at end — no explicit transaction block, queries ALL shipments

### 2.10 CageActor (`cage`)
- **What it does**: Tracks physical cargo location, dwell time, GO deadlines, and storage costs for held/inspection shipments. Assigns facilities, tracks dwell, emits GO warnings, releases on clearance.
- **Tables written**: `shipments` (UPDATE: cage_status JSONB, events)
- **Tables read**: `shipments` (WHERE status IN [held, inspection] AND cage_status IS NULL), `shipments` (WHERE status IN [held, inspection] AND cage_status IS NOT NULL), `shipments` (WHERE status IN [cleared, delivered] AND cage_status IS NOT NULL)
- **Status transitions**: None (only updates cage_status metadata)
- **FOR UPDATE queries**: 3 — intake, dwell update, release
- **Session pattern**: `async with session.begin()` — single transaction

### 2.11 TerminalActor (`terminal`)
- **What it does**: Simulates port terminal operations — berth delays for vessels and chassis shortages affecting container pickup.
- **Tables written**: `shipments` (UPDATE: events, financials)
- **Tables read**: `shipments` (WHERE status='in_transit' AND transport_mode='ocean'), `shipments` (WHERE status='cleared' AND transport_mode='ocean')
- **Status transitions**: None (only appends events)
- **FOR UPDATE queries**: 2 — arriving vessels, cleared ocean shipments
- **Session pattern**: `async with session.begin()` — single transaction

### 2.12 DemurrageActor (`demurrage`)
- **What it does**: Calculates demurrage/detention charges for ocean containers past free time, air cargo storage, and appointment miss fees.
- **Tables written**: `shipments` (UPDATE: financials, events)
- **Tables read**: `shipments` (WHERE transport_mode='ocean' AND status IN [at_customs, inspection, held, cleared]), `shipments` (WHERE transport_mode='air' AND status IN [at_customs, inspection, held, cleared]), `shipments` (WHERE status='cleared')
- **Status transitions**: None (only financial updates)
- **FOR UPDATE queries**: 3 — ocean demurrage, air storage, appointments
- **Session pattern**: `async with session.begin()` — single transaction

### 2.13 DisruptionActor (`disruption`)
- **What it does**: Generates weather and operational disruption events affecting ports. Uses DisruptionGenerator (optionally AI-powered) and attaches disruption events to affected shipments.
- **Tables written**: `shipments` (UPDATE: events)
- **Tables read**: `shipments` (destinations for active shipments), `shipments` (individual by ID)
- **Status transitions**: None (only appends events)
- **FOR UPDATE queries**: 1 — individual shipment locks when recording disruption
- **Session pattern**: `async with session.begin()` — single transaction

### 2.14 ExceptionsActor (`exceptions`)
- **What it does**: Generates cargo damage, shortage, overage, and misdeclared weight events on in-transit shipments, plus refused delivery on cleared shipments. Optionally AI-powered descriptions.
- **Tables written**: `shipments` (UPDATE: events, references)
- **Tables read**: `shipments` (WHERE status IN [in_transit, at_customs]), `shipments` (WHERE status='cleared')
- **Status transitions**: None (only appends events and flags)
- **FOR UPDATE queries**: 2 — in_transit/at_customs shipments, cleared shipments
- **Session pattern**: `async with session.begin()` — single transaction

### 2.15 DocumentsActor (`documents`)
- **What it does**: Validates documents and generates discrepancy events — CF-28 triggers, USMCA certification issues, weight/quantity variances, origin certificate problems.
- **Tables written**: `shipments` (UPDATE: events, references, hold_type)
- **Tables read**: `shipments` (WHERE status='at_customs'), `shipments` (WHERE status='at_customs' AND origin IN [MX, CA])
- **Status transitions**: None (sets hold_type but doesn't call `transition()`)
- **FOR UPDATE queries**: 2 — customs docs, origin docs
- **Session pattern**: `async with session.begin()` — single transaction

### 2.16 FinancialActor (`financial`)
- **What it does**: Calculates MPF, HMF, bond type, exam fees, and broker fees for shipments at customs and beyond.
- **Tables written**: `shipments` (UPDATE: financials)
- **Tables read**: `shipments` (WHERE status IN [at_customs, cleared]), `shipments` (WHERE status IN [inspection, held]), `shipments` (WHERE status IN [at_customs, cleared])
- **Status transitions**: None
- **FOR UPDATE queries**: 3 — entry fees, exam fees, broker fees
- **Session pattern**: `async with session.begin()` — single transaction

### 2.17 ISFActor (`isf`)
- **What it does**: Handles ISF 10+2 filing scenarios for ocean shipments — late filing penalties ($5,000), data mismatches requiring amendments.
- **Tables written**: `shipments` (UPDATE: references, financials, events)
- **Tables read**: `shipments` (WHERE transport_mode='ocean' AND status='booked'), `shipments` (WHERE transport_mode='ocean' AND status IN [in_transit, at_customs])
- **Status transitions**: None
- **FOR UPDATE queries**: 2 — booked ocean, in_transit/at_customs ocean
- **Session pattern**: `async with session.begin()` — single transaction

### 2.18 CBPAuthorityActor (`cbp_authority`)
- **What it does**: Simulates CBP responses to broker entry submissions. Risk-based response selection: accept, CF-28, CF-29, physical exam, reject. Handles CF-28 deadlines, exam results, and protest decisions.
- **Tables written**: `entry_filings` (UPDATE: filing_status, authority_response, released_at), `shipments` (UPDATE: status, events, hold_type)
- **Tables read**: `entry_filings` (WHERE filing_status='submitted'), `entry_filings` (WHERE filing_status='cf28_pending'), `entry_filings` (WHERE filing_status='exam_scheduled'), `shipments` (WHERE status='protest_filed'), individual shipments by ID
- **Status transitions**:
  - `entry_filed` -> `cleared` (accepted)
  - `entry_filed` -> `cf28_pending` (CF-28 issued)
  - `entry_filed` -> `cf29_pending` (CF-29 issued)
  - `entry_filed` -> `exam_scheduled` (exam)
  - `entry_filed` -> `held` (rejected)
  - `cf28_pending` -> `held` (timeout)
  - `exam_scheduled` -> `cleared` (exam passed)
  - `exam_scheduled` -> `held` (exam failed)
  - `protest_filed` -> `cleared` (approved)
  - `protest_filed` -> `held` (denied)
- **FOR UPDATE queries**: 4 — submitted entries, cf28_pending entries, exam_scheduled entries, protest_filed shipments
- **Session pattern**: `async with session.begin()` — single transaction

### 2.19 BrokerSimActor (`broker_sim`)
- **What it does**: Simulates broker workflow in demo mode — seeds demo brokers, auto-claims entries, progresses checklists by creating Document records, submits ready entries, responds to CF-28s. Protects certain brokers from auto-progression for manual user interaction.
- **Tables written**: `brokers` (INSERT — seed), `broker_assignments` (INSERT + UPDATE), `entry_filings` (INSERT + UPDATE), `documents` (INSERT), `shipments` (UPDATE: entry_number, status, events)
- **Tables read**: `brokers`, `shipments` (WHERE status IN [at_customs, held, inspection] AND NOT IN entry_filings), `entry_filings` (WHERE filing_status='draft'), `entry_filings` (WHERE filing_status='pending_broker_approval'), `entry_filings` (WHERE filing_status='cf28_pending'), `documents`
- **Status transitions**:
  - `at_customs` -> `entry_filed` (broker submits)
  - `at_customs`/`entry_filed` -> `cf28_pending` (for protected broker diversification)
  - `cf28_pending` -> `entry_filed` (CF-28 response)
- **FOR UPDATE queries**: 3 — draft entries, pending_broker_approval entries, cf28_pending entries
- **Session pattern**: `async with session.begin()` — single transaction

---

## 3. State Machines

### 3.1 US Shipment Lifecycle (Primary)

```
booked ─────────┬──> in_transit ────┬──> at_customs ──┬──> cleared ────> delivered
                │    (platform)     │    (carrier)    │    (customs)     (carrier)
                │                   │                 │
                ├──> held           ├──> held         ├──> inspection ──┬──> cleared
                │    (platform)     │    (platform,   │    (customs)    │    (customs)
                │                   │     carrier)    │                 ├──> held
                │                   │                 │                 │    (customs)
                │                   │                 ├──> entry_filed ─┤
                │                   │                 │    (broker)     │
                │                   │                 │                 ├──> cleared (cbp_authority)
                │                   │                 │                 ├──> cf28_pending ──> entry_filed (broker) / held
                │                   │                 │                 ├──> cf29_pending ──> cleared (broker) / protest_filed
                │                   │                 │                 ├──> exam_scheduled ──> cleared / held
                │                   │                 │                 └──> held (cbp_authority)
                │                   │                 │
                │                   │                 ├──> held (customs, pga)
                │                   │                 │
held ───────────┼──> cleared (resolution)
                ├──> at_customs (resolution)
                └──> in_transit (resolution)
```

**Terminal states**: `delivered`, `rejected`

### 3.2 Consolidation Lifecycle

```
booked -> closed -> in_transit -> arrived -> deconsolidating -> deconsolidated
          (consolidator) (carrier)  (carrier)    (carrier)          (carrier)
```

### 3.3 Non-US Jurisdictions

| Jurisdiction | Unique States | Key Differences |
|---|---|---|
| **EU/UK** | `lodged`, `accepted`, `under_control`, `released` | lodged->accepted->under_control->released->cleared |
| **CN** | `declared`, `inspected`, `released` | declared->inspected->released->cleared |
| **BR** | `registered`, `parameterized` | registered->parameterized->cleared |
| **IN** | `filed`, `assessed`, `out_of_charge` | filed->assessed->out_of_charge->cleared |

All share common prefix: `booked -> in_transit -> at_customs` and suffix: `cleared -> delivered`.

### 3.4 Entry Filing Lifecycle

```
draft -> pending_broker_approval -> submitted -> accepted/cf28_pending/cf29_pending/exam_scheduled/rejected
```

---

## 4. Data Flow — Shipment Lifecycle

### Phase 1: Creation (ShipperActor)
1. ShipperActor generates shipment with Poisson arrival process
2. Picks corridor, product, carrier service, generates route
3. Creates `Shipment` (status=`booked`) + optional `Order` + `OrderLineItem`s
4. Publishes `shipment_created` event

### Phase 2: Pre-Clearance Screening (PreClearanceAdapter)
1. Queries `booked` shipments (FOR UPDATE SKIP LOCKED, LIMIT 50)
2. Runs E0 PreClearanceAgent (entity screening, sanctions check)
3. Transitions: `booked` -> `in_transit` (passed) or `booked` -> `held` (entity screening hit)

### Phase 3: Consolidation (ConsolidatorActor)
1. Queries `in_transit` shipments without consolidation_id (FOR UPDATE SKIP LOCKED)
2. Groups by (mode, origin, destination, carrier)
3. Creates `Consolidation` record, links child shipments, adds master_number to references

### Phase 4: Transit (CarrierActor, TerminalActor, ISFActor, ExceptionsActor, DisruptionActor)
- **CarrierActor**: Waits for transit time to elapse, generates intermediate regulatory events, handles transit holds, transitions `in_transit` -> `at_customs`
- **TerminalActor**: Adds berth delay events for ocean shipments
- **ISFActor**: Checks for late ISF filings (ocean only), adds penalties
- **ExceptionsActor**: Generates damage/shortage/overage events
- **DisruptionActor**: Generates weather/operational disruption events

### Phase 5: Customs Processing (CustomsActor, PGAActor, DocumentsActor, BrokerSimActor, CBPAuthorityActor, FinancialActor)
- **BrokerSimActor**: Auto-claims entries, creates `EntryFiling` + `BrokerAssignment` + `Document` records, progresses through checklist, submits to CBP
- **CustomsActor**: STP fast-lane (~87%) or manual review (UFLPA, inspection, HS reclass, CF-28/CF-29, entity screening, ADD/CVD)
- **PGAActor**: FDA/EPA/CPSC/NHTSA/APHIS/TTB review windows
- **DocumentsActor**: Invoice quantity/value mismatches, description mismatches, weight variances, origin certificate issues, USMCA certification
- **CBPAuthorityActor**: Processes submitted entries — accept, CF-28, CF-29, exam, reject
- **FinancialActor**: MPF, HMF, bond, exam fees, broker fees

### Phase 6: Hold Management (ResolutionActor, CageActor, DemurrageActor, ShipperResponseActor)
- **CageActor**: Assigns facility to held/inspection shipments, tracks dwell time, GO deadline warnings, releases on clearance
- **DemurrageActor**: Calculates demurrage/detention/storage charges
- **ResolutionActor**: Resolves holds after review period (hold-type-specific profiles)
- **ShipperResponseActor**: Generates shipper responses to communications

### Phase 7: Delivery (CarrierActor)
1. Queries `cleared` shipments (FOR UPDATE SKIP LOCKED)
2. Waits for inland delivery time
3. Transitions `cleared` -> `delivered`
4. Advances consolidation to `deconsolidated` when all children cleared

### Phase 8: Compliance Analysis (ComplianceEngineActor)
1. Queries shipments where `analysis IS NULL` and status != `in_transit`
2. Generates classification, tariff, and compliance analysis JSONB
3. Stores in shipment's `analysis` column

---

## 5. API Surface

### 5.1 Simulation Control (`/api/simulation/`)
**Surface**: Platform admin
| Endpoint | Method | Purpose |
|---|---|---|
| `/start` | POST | Start simulation |
| `/stop` | POST | Stop simulation |
| `/pause` | POST | Pause clock |
| `/resume` | POST | Resume clock |
| `/reset` | POST | Reset (delete all, reseed) |
| `/status` | GET | Full simulation status |
| `/clock` | PUT | Adjust time ratio |
| `/actors` | GET | List actor configs |
| `/actors/{id}` | PUT | Hot-update actor config |
| `/dashboard/stream` | GET | SSE dashboard broadcast |
| `/dashboard/latest` | GET | Latest dashboard snapshot |

**Tables**: `shipments` (via dashboard aggregator)

### 5.2 Broker Surface (`/api/broker/`)
**Surface**: Broker
| Endpoint | Method | Purpose |
|---|---|---|
| `/brokers` | GET | List brokers |
| `/brokers/{id}` | GET | Broker detail |
| `/dashboard` | GET | Broker dashboard KPIs |
| `/alerts` | GET | Priority alerts |
| `/queue` | GET | Paginated entry queue |
| `/queue/claim` | POST | Claim an entry |
| `/entries/{id}` | GET | Entry filing detail |
| `/entries/{id}/checklist` | PUT | Update checklist |
| `/entries/{id}/bond` | PUT | Update bond info |
| `/entries/{id}/checklist/refresh` | GET | Refresh computed checklist |
| `/entries/{id}/approve` | POST | Broker approval |
| `/entries/{id}/submit` | POST | Submit to CBP |
| `/entries/{id}/summary` | GET | Entry summary data |
| `/cbp-responses` | GET | CBP response list |
| `/cbp-responses/{id}/respond-cf28` | POST | Respond to CF-28 |
| `/cbp-responses/{id}/respond-cf29` | POST | Respond to CF-29 |
| `/messages` | GET/POST | Broker messages |
| `/briefing` | GET | Morning briefing |
| `/authority-inquiries` | GET | Authority inquiries |
| `/ai/draft-cf28-response` | POST | AI: draft CF-28 response |
| `/ai/draft-communication` | POST | AI: draft communication |
| `/ai/entry-insights` | GET | AI: entry insights |
| `/ai/draft-cf29-protest` | POST | AI: draft CF-29 protest |

**Tables**: `entry_filings`, `shipments`, `brokers`, `broker_assignments`, `documents`, `broker_messages`

### 5.3 Shipments (`/api/shipments/`)
**Surface**: Platform, Shipper
**Tables**: `shipments`, `orders`

### 5.4 Platform (`/api/platform/`)
**Surface**: Platform
**Tables**: `shipments` (aggregates)

### 5.5 Dashboard (`/api/dashboard/`)
**Surface**: Platform
**Tables**: `shipments` (via dashboard_aggregator)

### 5.6 LLM-Powered Endpoints (10/min rate limit)
| Route File | Endpoint | Purpose |
|---|---|---|
| `chat.py` | `POST /api/chat/stream` | AI chat assistant |
| `classify.py` | `POST /api/classify/stream` | HS classification |
| `analysis.py` | `POST /api/analyze/stream` | Compliance analysis |
| `description_quality.py` | `POST /api/description-quality` | Product description quality |
| `resolution.py` | `POST /api/resolution/*` | Resolution suggestions |

### 5.7 Other Routes
| Route | Surface | Tables |
|---|---|---|
| `/api/orders/` | Shipper | `orders`, `order_line_items` |
| `/api/products/` | Shipper, Platform | (product catalog) |
| `/api/documents/` | Broker, Shipper | `documents` |
| `/api/consolidations/` | Platform | `consolidations`, `shipments` |
| `/api/compliance/` | Platform | `shipments` |
| `/api/screening/` | Platform | (entity screening) |
| `/api/trade-lanes/` | Platform | `shipments` (aggregates) |
| `/api/tariff/` | Platform | (tariff lookup) |
| `/api/fta/` | Platform | (FTA eligibility) |
| `/api/exception/` | Platform | `shipments` |
| `/api/exception-status/` | Platform | `shipments` (status transitions) |
| `/api/regulatory/` | Platform | (regulatory intelligence) |
| `/api/jurisdiction/` | Platform | (jurisdiction config) |
| `/api/session/` | All | (session management) |

---

## 6. Background Services

### 6.1 Dashboard Aggregator (`dashboard_aggregator.py`)
- Runs on `SimulationCoordinator._dashboard_loop` at configurable interval
- Queries `shipments` table for: total counts, status distribution, financial aggregates, corridor metrics, cage metrics, STP rate, UFLPA counts
- Caches in Redis (`sim:dashboard:latest`, TTL=5s)
- Broadcasts to SSE subscribers
- **Heavy queries**: 10+ SQL queries per iteration (count, group by status, sum values, per-corridor hold rates, JSONB extraction for financials and cage_status)

### 6.2 Event Bus (`event_bus.py`)
- Redis pub/sub on channel `sim:events`
- Falls back to in-memory dispatch if Redis unavailable
- Actors publish events but **no actors subscribe** — the event bus is currently fire-and-forget for logging/SSE purposes only
- Not used for actor-to-actor coordination

### 6.3 Simulation Clock (`clock.py`)
- Virtual clock with configurable `time_ratio` (sim-minutes per real-second)
- Supports pause/resume
- Business hours detection for shipper volume modulation

---

## 7. Contention Map — The Core Performance Problem

### 7.1 Status-Based Contention Matrix

The table below shows which actors lock rows in which statuses. **Every cell with 2+ actors represents a contention point.**

| Shipment Status | Actors Competing for FOR UPDATE Lock |
|---|---|
| **`booked`** | `preclearance`, `isf` (ocean only) |
| **`in_transit`** | `carrier`, `consolidator`, `terminal` (ocean), `isf` (ocean), `exceptions` |
| **`at_customs`** | **`customs`, `pga`, `documents`, `financial`, `broker_sim`**, `exceptions`, `demurrage` |
| **`inspection`** | `customs`, `cage`, `financial`, `demurrage` |
| **`held`** | `resolution`, `cage`, `financial`, `demurrage`, `broker_sim` |
| **`cleared`** | `carrier`, `cage`, `terminal` (ocean), `financial`, `demurrage`, `exceptions` |
| **`entry_filed`** | `cbp_authority` (via entry_filings) |

### 7.2 Critical Contention Hotspots

#### Hotspot 1: `at_customs` — 7 actors competing
This is the **worst contention point**. Seven actors all query `WHERE status = 'at_customs'` with `FOR UPDATE SKIP LOCKED`:
- `customs` — the primary screener, needs every row
- `pga` — PGA review, also needs every row
- `documents` — document validation, needs every row
- `financial` — fee calculation, needs every row
- `broker_sim` — entry creation, needs every row (via subquery)
- `exceptions` — exception injection
- `demurrage` — D&D calculation

**Impact**: With `SKIP LOCKED`, actors that run second on a tick will skip rows already locked by the first actor. This means some shipments may not get processed by all required actors in the same tick, leading to:
- Delayed fee calculations
- Missing document validation
- Broker entries not created
- PGA reviews not triggered

#### Hotspot 2: `cleared` — 6 actors competing
- `carrier` — delivery
- `cage` — release
- `terminal` — chassis shortage
- `financial` — fee calculation
- `demurrage` — D&D and appointment fees
- `exceptions` — refused delivery

#### Hotspot 3: `held` — 5 actors competing
- `resolution` — resolution logic
- `cage` — dwell tracking
- `financial` — exam fees
- `demurrage` — D&D
- `broker_sim` — entry creation for held shipments

#### Hotspot 4: `in_transit` — 5 actors competing
- `carrier` — transit completion
- `consolidator` — consolidation grouping
- `terminal` — berth delays
- `isf` — ISF mismatches
- `exceptions` — in-transit exceptions

### 7.3 Non-Locking Actors (Silent Contenders)

Two actors query broadly **without FOR UPDATE**:
- **`shipper_response`**: Queries ALL shipments (no status filter, no locking) — reads entire table every tick
- **`compliance`**: Queries by `analysis IS NULL` without locking — may read dirty data or face phantom reads

### 7.4 JSONB Mutation Races

Multiple actors mutate the same JSONB columns on the same rows:
- `events` (list): customs, pga, cage, terminal, demurrage, disruption, exceptions, documents, carrier, isf, broker_sim, shipper_response
- `references` (dict): carrier, consolidator, pga, isf, exceptions, documents
- `financials` (dict): customs, demurrage, terminal, financial, isf
- `codes` (dict): customs

With `SKIP LOCKED`, last-writer-wins is avoided — but the skip means some actors simply don't run for certain shipments on certain ticks.

### 7.5 The ShipperResponseActor Full-Table Scan

`shipper_response` does `select(Shipment)` with **no WHERE clause and no locking**. On a large simulation with thousands of shipments, this actor:
1. Loads the entire shipments table into memory every tick
2. Iterates over all events on every shipment looking for pending communications
3. Doesn't use `FOR UPDATE` so it can silently conflict with other writers

### 7.6 Dashboard Aggregator Contention

The dashboard aggregator runs 10+ queries per iteration against the `shipments` table, including:
- `COUNT(*)` with various status filters
- `SUM(declared_value)` over all shipments
- JSONB extraction queries (`financials['predicted_duty']`, `cage_status['dwell_days']`)
- Per-corridor `COUNT` queries with `GROUP BY`

These read queries don't lock rows but they do acquire share locks and can interfere with the many `FOR UPDATE` queries from actors.

### 7.7 Total FOR UPDATE Queries per Tick Cycle

| Actor | FOR UPDATE Queries |
|---|---|
| carrier | 4 |
| cbp_authority | 4 |
| cage | 3 |
| demurrage | 3 |
| financial | 3 |
| broker_sim | 3 |
| customs | 2 |
| documents | 2 |
| exceptions | 2 |
| isf | 2 |
| terminal | 2 |
| consolidator | 1 |
| preclearance | 1 |
| pga | 1 |
| resolution | 1 |
| disruption | 1 |
| **Total** | **35** FOR UPDATE queries per tick cycle |

All 35 queries hit the **same `shipments` table** (plus some on `entry_filings` and `consolidations`).

---

## 8. Database Schema Summary

### Primary Tables
| Table | Key Columns | JSONB Columns |
|---|---|---|
| `shipments` | id, product, origin, destination, status, carrier, tracking_number, declared_value, transport_mode, house_number, entry_number, consolidation_id, hold_type, order_id | events, codes, financials, references, cargo_detail, waypoints, analysis, cage_status |
| `orders` | id, status, origin, destination, shipment_id | transit_points, analysis, document_status |
| `order_line_items` | id, order_id, product_id, product_name, quantity, hs_code | — |
| `consolidations` | id, transport_mode, consolidation_type, master_number, carrier, status | events |
| `entry_filings` | id, shipment_id, broker_id, entry_number, filing_status, port_of_entry | checklist_state, summary_data, broker_approval, authority_response |
| `documents` | id, order_id, shipment_id, document_type, filename, content | — |
| `brokers` | id, name, license_number, status | specializations, contact_info |
| `broker_assignments` | id, broker_id, shipment_id, status, priority | — |
| `broker_messages` | id, shipment_id, broker_id, direction, channel | — |

### JSONB Column Sizes (Estimated)
- `events`: Grows unboundedly — 10-50+ events per shipment, each 200-500 bytes. A mature shipment can have 5-20KB in events.
- `references`: 500-2000 bytes, includes regulatory_touchpoints array
- `financials`: 200-800 bytes
- `codes`: 100-300 bytes
- `cargo_detail`: 200-500 bytes
- `waypoints`: 500-2000 bytes
- `cage_status`: 200-500 bytes (when present)
- `analysis`: 500-1500 bytes (when present)

---

## 9. Key Architectural Observations

### 9.1 Actors Don't Communicate — They Compete
Despite the EventBus, actors don't subscribe to each other's events. They all independently poll the database for rows matching their status criteria. This creates a **poll-and-lock** pattern where the database is the sole coordination mechanism.

### 9.2 Every Actor Is a Full-Table Scanner
Most actors do `SELECT * FROM shipments WHERE status = X FOR UPDATE SKIP LOCKED` — fetching ALL rows in a given status. There's no pagination, no batch limits (except preclearance LIMIT 50), and no targeting.

### 9.3 JSONB Columns Are the Primary Data Store
Seven JSONB columns on the `shipments` table serve as the primary data store for events, financial tracking, compliance codes, references, cargo details, waypoints, and cage management. This means:
- No referential integrity on financial/events/references data
- No indexing on frequently queried JSONB paths
- Full column replacement on every update (`flag_modified`)

### 9.4 Transaction Patterns Are Inconsistent
- Some actors use `async with session.begin()` (single transaction for all work)
- CarrierActor commits per-shipment (`await session.commit()` inside the loop)
- ShipperResponseActor uses a bare `session.commit()` without explicit transaction block
- ComplianceEngineActor has no FOR UPDATE locking at all

### 9.5 The `shipments` Table Is the Single Bottleneck
All 19 actors read from and 17 actors write to the `shipments` table. The `entry_filings` table is secondary (2 actors). The `consolidations` table is tertiary (2 actors). Everything else is peripheral.

---

## 10. Proactive vs Sequential Actor Behavior

> **Stakeholder principle**: "Every service must be designed not to follow a process, but to always evaluate if the requirements are met for it to do its job." Services should act at the **earliest possible time** that prerequisites are met, not wait for a specific status.

### 10.1 Actor Archetype Classification

Four distinct actor archetypes emerge from the codebase analysis:

| Archetype | Description | Event Subscription Model |
|-----------|-------------|--------------------------|
| **PROACTIVE EVALUATOR** | Subscribes to data-readiness events; acts at the earliest moment prerequisites are met | Multi-event subscription → readiness evaluation → action |
| **REACTIVE RESPONDER** | Responds to commands (filings, declarations) or exceptions (failures in the proactive pipeline) | Command-driven + exception-driven triggers |
| **TIME-CORRECT** | Correctly depends on physical time passage or real-world state | Timer-based or physical-state triggers |
| **PROBABILISTIC** | Generates events stochastically to simulate real-world disruptions | Probability-weighted triggers |

| Archetype | Actor | Current Trigger | Actual Prerequisites / Correct Trigger | Status |
|-----------|-------|----------------|----------------------------------------|--------|
| **PROACTIVE** | `preclearance` | `status == "booked"` | Shipment exists with product info | ✅ Runs immediately after creation |
| **PROACTIVE** | `isf` | `status == "booked"` (ocean only) | Ocean shipment with shipper data | ✅ Files ISF pre-departure |
| **PROACTIVE** | `consolidator` | `status == "in_transit"` | In-transit shipments with matching routes | ✅ Groups as soon as in transit |
| **PROACTIVE** | `broker_sim` (claim) | `status.in_(["at_customs", ...])` | Enough shipment data to assess complexity → `ShipmentCreated` | ❌ Should assign broker pre-departure |
| **PROACTIVE** | `pga` | `status == "at_customs"` | HS code + product category → PGA mapping → `ClassificationCompleted` | ❌ Should determine PGA pre-arrival |
| **PROACTIVE** | `documents` | `status == "at_customs"` | Documents exist for the shipment → `DocumentUploaded` | ❌ Should validate pre-arrival |
| **PROACTIVE** | `financial` (entry fees) | `status.in_(["at_customs", "cleared"])` | Tariff result available → `TariffCalculated` | ❌ Should calculate pre-arrival |
| **PROACTIVE** | `compliance_engine` | Pipeline call during order analysis | Product + shipper entity info → `RegulatorySignalDetected` | ⚠️ One-shot; should re-screen on reg changes |
| **REACTIVE** | `customs` (authority) | `status == "at_customs"` | **Command**: `DeclarationSubmitted` (broker files) | ❌ See §10.2a |
| **REACTIVE** | `customs` (exception) | `status == "at_customs"` | **Exception**: `ShipmentArrivedWithoutClearance` | ❌ See §10.2a |
| **TIME-CORRECT** | `carrier` | `status == "in_transit"`, `elapsed >= required_delta` | Physical transit time elapsed | ✅ Correctly time-dependent |
| **TIME-CORRECT** | `cage` | `status.in_(["held", "inspection"])` | Cargo physically held | ✅ Correctly tracks physical dwell |
| **TIME-CORRECT** | `demurrage` | `status.in_(["at_customs", ...])` | Days since arrival > free time | ✅ Correctly time-dependent |
| **TIME-CORRECT** | `resolution` | Held shipments, time-based | Hold exists + resolution criteria met | ✅ Timer + criteria based |
| **PROBABILISTIC** | `disruption`, `terminal`, `exceptions` | Various statuses | Random/probability triggers | N/A — event generators |

### 10.2 Anti-Patterns and Correct Patterns in Detail

#### 10.2a `customs` actor — REACTIVE RESPONDER, not Proactive Evaluator (`actors/customs.py:77-96`)

**Critical distinction**: The customs AUTHORITY is fundamentally different from proactive evaluators. It does NOT evaluate data readiness — it **responds** to two types of triggers:

1. **Command-driven**: A broker submits a declaration → customs processes it (accept, query, reject, examine)
2. **Exception-driven**: A shipment arrives at port without pre-clearance → customs must inspect

```python
# CURRENT: Polls for shipments at_customs — conflates authority with filing
select(Shipment).where(Shipment.status == "at_customs")

# CORRECT EVENT MODEL:
# Trigger 1 (command): DeclarationSubmitted → customs evaluates the filing
# Trigger 2 (exception): ShipmentArrivedWithoutClearance → customs flags for inspection
#
# Customs does NOT subscribe to ClassificationCompleted or other data-readiness events.
# The BROKER is the proactive evaluator who files declarations; customs RESPONDS to filings.
```

In real-world trade, customs authorities (CBP, HMRC, CBIC, etc.) are reactive by design. They process what is submitted to them. The current codebase conflates two distinct roles into one `customs` actor:
- **Broker role** (proactive): Decides when to file, gathers prerequisites, submits declaration
- **Authority role** (reactive): Receives the filing, evaluates it, issues a response (accept/query/reject/examine)

A shipment arriving at port without pre-clearance is a **failure of the proactive system** — it means the broker/pre-clearance pipeline didn't complete in time. The customs authority actor should handle this as an exception path, not the normal flow.

#### 10.2b `broker_sim._claim_entries()` — should be PROACTIVE EVALUATOR (`actors/broker_sim.py:156`)
```python
# CURRENT: Waits for at_customs/held/inspection
select(Shipment).where(
    Shipment.status.in_(["at_customs", "held", "inspection"]),
    ~Shipment.id.in_(subq),  # no entry filing yet
)

# SHOULD: Assign broker as soon as shipment has enough data
# Prerequisites: product, origin, destination, declared_value, HS code
# Broker can begin prep work while shipment is in transit
```
In practice, brokers are engaged well before cargo arrives. They prepare documentation, verify classifications, and pre-file declarations. **The broker is the proactive evaluator for declaration filing** — it subscribes to `ClassificationCompleted`, evaluates readiness, and submits the declaration when prerequisites are met. The customs authority then responds to that filing.

#### 10.2c `pga` actor (`actors/pga.py:89`)
```python
# CURRENT: Waits for at_customs
select(Shipment).where(Shipment.status == "at_customs")

# SHOULD: Determine PGA requirements as soon as HS code is known
# Prerequisites: HS code → PGA mapping table lookup
# PGA determination is a data lookup, not a physical-arrival dependency
```

#### 10.2d `documents` actor (`actors/documents.py:85`)
```python
# CURRENT: Waits for at_customs
select(Shipment).where(Shipment.status == "at_customs")

# SHOULD: Validate documents as soon as they exist
# Prerequisites: Documents uploaded for this shipment
# Document discrepancies should be caught EARLY (pre-departure if possible)
```

#### 10.2e `financial._calculate_entry_fees()` (`actors/financial.py:58`)
```python
# CURRENT: Waits for at_customs or cleared
select(Shipment).where(Shipment.status.in_(["at_customs", "cleared"]))

# SHOULD: Calculate fees as soon as tariff result is available
# Prerequisites: tariff_result JSONB populated, declared_value known
```

### 10.3 Prerequisite Maps by Archetype

#### Proactive Evaluators — readiness defined by **data prerequisites**, not status:

| Service (Actor) | Data Prerequisites | Earliest Trigger Event | Current Status Gate |
|-----------------|-------------------|----------------------|-------------------|
| **Broker assignment** (`broker_sim`) | Shipment data sufficient for complexity assessment | `ShipmentCreated` | `at_customs` |
| **Declaration filing** (`broker_sim`) | HS code + origin + destination + declared_value + shipper info | `ClassificationCompleted` | `at_customs` |
| **PGA determination** (`pga`) | HS code + product category → PGA mapping exists | `ClassificationCompleted` | `at_customs` |
| **Document validation** (`documents`) | At least one document exists for the shipment | `DocumentUploaded` | `at_customs` |
| **Entry fee calculation** (`financial`) | Tariff result available | `TariffCalculated` | `at_customs` / `cleared` |
| **Compliance re-screening** (`compliance_engine`) | Regulatory signal change + affected HS codes | `RegulatorySignalDetected` | Never (one-shot at order time) |

#### Reactive Responders — triggered by **commands or exceptions**, not data readiness:

| Service (Actor) | Trigger Type | Event Subscription | Current Status Gate |
|-----------------|-------------|-------------------|-------------------|
| **Customs evaluation** (`customs` authority) | Command | `DeclarationSubmitted` (broker files declaration) | `at_customs` |
| **Customs exception handling** (`customs` authority) | Exception | `ShipmentArrivedWithoutClearance` (pre-clearance pipeline failed) | `at_customs` |

**Key insight**: The `customs` actor in the current codebase conflates two roles. In the event-driven redesign, the broker's proactive declaration filing produces a `DeclarationSubmitted` event, and the customs authority's reactive evaluation consumes it. The exception path (`ShipmentArrivedWithoutClearance`) fires only when the proactive pipeline fails — this is an error recovery path, not the happy path.

### 10.4 State Machine Implications

The current state machine enforces a physical-arrival-centric process:
```
booked → in_transit → at_customs → [entry_filed → ...] → cleared → delivered
```

The proactive model separates the **broker pipeline** (proactive, data-driven) from **customs authority processing** (reactive, command-driven):

```
HAPPY PATH (pre-clearance succeeds):
booked → in_transit ──────────────────────────────────→ cleared → delivered
              │ BROKER PIPELINE (proactive evaluators):       ▲
              ├── Broker assigned (on ShipmentCreated)        │
              ├── Classification completed                    │
              ├── PGA requirements determined                 │
              ├── Documents validated                         │
              ├── Fees pre-calculated                         │
              ├── Declaration filed → DeclarationSubmitted ───┤
              │                                               │
              │ CUSTOMS AUTHORITY (reactive responder):       │
              └── Receives DeclarationSubmitted ──→ Evaluates │
                  Accept/Query/Reject/Examine ────────────────┘

EXCEPTION PATH (pre-clearance fails or incomplete):
booked → in_transit → at_customs ──→ [customs exception handling] → cleared → delivered
                          │
                          └── ShipmentArrivedWithoutClearance
                              Customs authority handles as inspection/exception
```

The state machine's `at_customs` state becomes the **exception path** — a shipment that has been fully pre-cleared can transition directly from `in_transit` to `cleared` when the customs authority accepts the pre-filed declaration. A shipment arriving at `at_customs` means the proactive pipeline **failed** to complete in time. This is a critical design signal: high `at_customs` arrival rates indicate the proactive system needs improvement.

### 10.5 Event Architecture Impact

In the event-driven architecture, two distinct subscription patterns emerge:

#### Proactive Evaluator Pattern (broker, pga, documents, financial, compliance)

1. **Events are information signals, not commands.** `ShipmentCreated` doesn't mean "start customs processing" — it means "new information is available; re-evaluate your readiness."

2. **Each service maintains a readiness state per shipment.** The broker service tracks: "for shipment X, do I have HS code? ✅ Origin? ✅ Value? ✅ Shipper? ✅ → File declaration now."

3. **Multiple events can satisfy the same prerequisite differently.** The HS code might come from `ClassificationCompleted` (automated) or from `ProductUpdated` (manual correction). The broker service doesn't care which — it just checks "do I have an HS code?"

4. **Idempotent re-evaluation.** A service receiving the same event twice should re-evaluate and reach the same conclusion (or do nothing if already acted).

5. **The readiness evaluator replaces the status filter.** Instead of `WHERE status = 'at_customs'`, the service evaluates `WHERE hs_code IS NOT NULL AND origin IS NOT NULL AND declared_value IS NOT NULL AND NOT already_filed`.

#### Reactive Responder Pattern (customs authority)

6. **Commands are explicit action requests, not information signals.** `DeclarationSubmitted` is a command from the broker to the customs authority: "here is a filing — evaluate it." This is fundamentally different from information events.

7. **The authority does NOT maintain readiness state.** It processes each command as received. Its inputs are fully formed declarations, not partial data prerequisites.

8. **Exception events signal system failures.** `ShipmentArrivedWithoutClearance` indicates the proactive pipeline failed. The customs authority enters exception handling mode — inspection, manual processing, potential penalties.

9. **The authority's response generates events consumed by others.** `DeclarationAccepted`, `CustomsQueryIssued` (CF-28 equivalent), `ExamScheduled`, `DeclarationRejected` — these are authority outputs that feed back into the proactive services (e.g., the broker responds to queries, the cage actor handles exams).

### 10.6 The Broker↔Customs Authority Event Chain

The most important event chain in the platform flows between the broker (proactive) and customs authority (reactive):

```
BROKER (proactive evaluator)              CUSTOMS AUTHORITY (reactive responder)
─────────────────────────                 ────────────────────────────────────
Subscribes to:                            Subscribes to:
  ClassificationCompleted                   DeclarationSubmitted
  DocumentUploaded                          ShipmentArrivedWithoutClearance
  TariffCalculated
  ShipmentCreated

Evaluates readiness → Files declaration
  → publishes DeclarationSubmitted ──────→ Receives filing → Evaluates
                                            → publishes DeclarationAccepted
                                            OR CustomsQueryIssued (CF-28)
                                            OR ExamScheduled
                                            OR DeclarationRejected

Receives CustomsQueryIssued ←────────────   (broker responds to queries)
  → prepares response → publishes
    QueryResponseFiled ──────────────────→ Receives response → Re-evaluates
```

This chain replaces the current monolithic `customs` actor that conflates both roles. The separation enables pre-clearance (broker files while in transit, authority processes before arrival) and makes exception handling explicit (arrivals without clearance become a measurable failure mode, not the default path).
