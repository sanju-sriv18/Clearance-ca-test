# Scaling Analysis: FedEx-Scale Operations

**Target:** Billions of messages/day + 6,000 concurrent brokers + hundreds of SMB customers

## 1. Load Profile

### Message Volume
- FedEx: ~15M packages/day, each generating 30-100 lifecycle events
- **Target: 500M-1.5B events/day = 6,000-17,000 events/second sustained**
- Peak (Monday morning, holiday season): 3-5x sustained = **50,000+ events/second**

### Broker Workload
- 6,000 licensed customs brokers, ~4,000 concurrent during US business hours
- Work queue updates delivered via SSE (persistent connections, push-on-change) — **not polling**
- Broker API traffic: detail views, mutations, initial page loads only
- **Target: 200-500 broker API requests/second** (detail views + mutations; work queue is SSE, not REST)
- **SSE connections: 4,000-6,000 persistent** (negligible server cost)

### SMB E-Commerce
- Hundreds of customers, each pushing orders via API
- Order ingest: 10K-100K orders/day per large customer
- **Target: 500-2,000 order API requests/second**

### Combined
- **Events: 17,000-50,000/sec**
- **API reads: 1,000-3,000/sec** (broker polling eliminated by SSE)
- **API writes: 1,000-5,000/sec**
- **SSE connections: 4,000-6,000 persistent**

---

## 2. Current State

| Component | Today | Capacity |
|-----------|-------|----------|
| PostgreSQL | 1 instance, 500 max connections | ~200-500 req/sec |
| NATS JetStream | 1 instance, no limits configured | ~100K msg/sec (single node) |
| Redis | 1 instance, 400 client connections | ~100K ops/sec (single node) |
| API Gateway | 1 Caddy instance | ~10K req/sec (proxy) |
| Domain Services | 15 services x 2 workers = 30 processes | ~500-1,000 req/sec total |
| Infrastructure | 1 x t4g.xlarge (4 vCPU, 16 GB) | All of the above on one box |

**Current effective throughput: ~200-500 requests/second before degradation.**
**Target: 20,000-60,000 operations/second.**

**Gap: 40-120x current capacity.**

---

## 3. What Must Change

### 3.1 Database: From Single Instance to Distributed

**Problem:** Single PostgreSQL instance is the primary bottleneck. 500 max connections shared across 15 services. No partitioning. No read replicas.

**Required:**

| Change | Purpose |
|--------|---------|
| **Read replicas (2-3 per region)** | Offload broker work queue queries, status scans, dashboard aggregations. All GET endpoints read from replicas. |
| **Time-based partitioning on hot tables** | `shipments`, `entry_filings`, `handling_units`, `hu_transit_events` partitioned by month. Old partitions archived to cold storage. Queries on active data scan 1/12th the rows. |
| **Connection pooling via PgBouncer** | Pool connections at the infrastructure level (not per-service). 15 services x N replicas need centralized pooling. Target: 10,000+ virtual connections mapped to 500 real connections per instance. |
| **Separate write primary per domain group** | Phase 2: Split into 3-4 PostgreSQL clusters by data gravity (declaration+adjudication, shipment+cargo, financial+compliance, everything else). Each cluster with its own primary + replicas. |

**Index additions for scale:**
- `(broker_id, filing_status, priority)` composite on entry_filings (work queue)
- `(consolidation_id, status)` on shipments
- `(importer_id, status)` on bonds (sufficiency checks)
- `(shipment_id, created_at)` on hu_transit_events (timeline queries)

### 3.2 Event Bus: NATS JetStream Cluster

**Problem:** Single NATS instance. No clustering. No backpressure. Consumer failure = DLQ after 3 attempts with no alerting.

**Required:**

| Change | Purpose |
|--------|---------|
| **NATS 3-node cluster** | Fault tolerance + throughput. JetStream replicates across nodes. |
| **Stream partitioning by corridor** | High-volume streams (CLEARANCE_SHIPMENT, CLEARANCE_DECLARATION) partitioned by origin-destination corridor. US-CN, US-EU, US-MX as separate subject hierarchies. |
| **Consumer groups with horizontal scaling** | Each domain consumer runs N instances (not 1). NATS distributes messages across consumer group members. Declaration management needs 10+ consumer instances. |
| **Backpressure + flow control** | Configure max_pending per consumer. Slow consumers don't accumulate unbounded backlogs. |
| **Stream retention limits** | Max bytes per stream. Regulatory streams (365-day retention) need tiered storage: hot (7 days in memory) + warm (90 days on disk) + cold (365 days in object storage). |

### 3.3 Compute: Horizontal Service Scaling

**Problem:** 2 uvicorn workers per domain service on a single host. No autoscaling.

**Required:**

| Change | Purpose |
|--------|---------|
| **Kubernetes deployment** | Each domain service as a Deployment with HPA (Horizontal Pod Autoscaler). Scale based on CPU/request latency. |
| **Service-specific scaling profiles** | Declaration management: 20-50 pods (6000 brokers). Shipment lifecycle: 10-30 pods (event volume). Trade intelligence: 5-10 pods (reference lookups, cached). Product catalog: 3-5 pods (low volume). |
| **Worker count per pod** | 4 uvicorn workers per pod (match vCPU count). |
| **Regional deployment** | US-East (primary), US-West (DR), EU-West (GDPR + EU customs). Broker traffic routed to nearest region. |

### 3.4 API Gateway: From Caddy to Load-Balanced Fleet

**Problem:** Single Caddy instance. No rate limiting at gateway. No circuit breaking.

**Required:**

| Change | Purpose |
|--------|---------|
| **AWS ALB + API Gateway** | ALB for L7 routing. API Gateway for rate limiting, API key management, throttling per SMB customer. |
| **Per-customer rate limits** | SMB customers get allocated throughput (e.g., 1000 req/min per customer). Brokers get higher limits. Internal services unlimited. |
| **Circuit breaker at gateway** | If a domain service is degraded, return cached data or 503 — don't queue requests until the service drowns. |

### 3.5 Cache: Redis Cluster + Aggressive CQRS

**Problem:** Single Redis instance. Cache projector is a single consumer. Cache miss = DB query.

**Required:**

| Change | Purpose |
|--------|---------|
| **Redis Cluster (6-node)** | Horizontal sharding. Broker work queues, shipment status, dashboard metrics all cached. |
| **Cache-first read model** | Broker work queue MUST be served from cache, not DB. Cache projector maintains materialized views in Redis. DB is the fallback, not the primary read path. |
| **Projector scaling** | Multiple cache projector instances, each responsible for a subset of domains. Declaration projector separate from shipment projector. |
| **TTL strategy** | Work queue: 5s TTL (near-real-time). Dashboard: 30s. Reference data: 1 hour. Regulatory: 24 hours. |

### 3.6 Architecture Pattern Changes

#### Broker Work Queue: SSE-Driven Materialized View

The broker work queue is the single hottest read path in the system. A naive polling model (6,000 brokers × every 5-15 seconds) would generate 400-1,200 queries/second — but polling is the wrong pattern. The rest of the architecture is event-driven; the broker experience should be too.

**Architecture: Server-Sent Events (SSE) fed by NATS event consumers.**

1. Every state change (new entry, status change, broker assignment, checklist update) publishes an event to NATS.
2. A dedicated work queue projector consumes these events and maintains a per-broker work queue in Redis as a sorted set.
3. The projector pushes **deltas** (not full snapshots) to an SSE channel keyed by `broker_id`.
4. Connected brokers receive real-time updates over persistent SSE connections — **zero polling, zero database queries on the hot path**.
5. `GET /api/broker/queue` serves the initial snapshot from Redis on page load; SSE handles all subsequent updates.
6. The projector runs N instances for throughput, partitioned by broker_id hash.

**Why SSE over WebSockets:** SSE is simpler (unidirectional, auto-reconnect, works through HTTP/2), sufficient for this read-heavy pattern, and the Caddy gateway is already configured with `flush_interval -1` for SSE support. Brokers don't push data through this channel — they use standard REST mutations.

**Load profile with SSE:**
- 6,000 persistent TCP connections (negligible resource cost vs. 1,200 queries/second)
- Push-on-change only — most brokers see updates every few seconds
- Initial page load: 1 Redis read per broker session start
- Broker mutations (accept entry, update checklist): standard REST → event → projector → SSE push to affected brokers

#### Shipment Tracking: Append-Only Event Store

Billions of tracking events cannot be UPDATEd into a `shipments` table. The shipment lifecycle must become an event store:

1. Inbound tracking messages (carrier EDI, IoT, API) append to NATS stream.
2. Consumer projects current state into Redis (latest status, location, ETA).
3. PostgreSQL stores the event log (partitioned by date) for audit and replay.
4. Queries for "current shipment status" hit Redis. Queries for "shipment history" hit PostgreSQL partitioned table.

#### Compliance Screening: Async Pipeline

At 15M packages/day, synchronous compliance screening per package is not viable. Screening must be:

1. Triggered by event (ShipmentCreatedEvent).
2. Processed asynchronously by a pool of compliance workers.
3. Results published as ComplianceScreenedEvent.
4. Broker sees results when they arrive — not blocking on screening completion.

The current proxy-to-engine pattern (synchronous HTTP to engine layer) must be replaced with event-driven async processing.

---

## 4. Infrastructure Sizing (Target State)

### Kubernetes Cluster

| Resource | Specification |
|----------|---------------|
| Node pool (compute) | 10-20 x m6g.2xlarge (8 vCPU, 32 GB) ARM64 |
| Node pool (database) | 3-5 x r6g.2xlarge (8 vCPU, 64 GB) for PgBouncer + replicas |
| Node pool (cache) | 3 x r6g.xlarge (4 vCPU, 32 GB) for Redis Cluster |

### PostgreSQL (RDS or Aurora)

| Cluster | Instance | Replicas | Purpose |
|---------|----------|----------|---------|
| Declaration + Adjudication | db.r6g.2xlarge | 2 read | Broker queue, entry filings, CBP decisions |
| Shipment + Cargo | db.r6g.2xlarge | 3 read | Tracking events, handling units (highest write volume) |
| Financial + Compliance | db.r6g.xlarge | 1 read | Duty calcs, screening results |
| Reference + Other | db.r6g.large | 1 read | Products, trade intel, parties, regulatory |

### NATS JetStream

| Cluster | Nodes | Purpose |
|---------|-------|---------|
| Primary | 3 x m6g.xlarge | All 16 streams, clustered for HA |
| High-volume | 3 x m6g.2xlarge | CLEARANCE_SHIPMENT + CLEARANCE_CARGO (billions of events) |

### Redis

| Cluster | Nodes | Purpose |
|---------|-------|---------|
| Application cache | 6-node cluster (3 primary + 3 replica) | Work queues, status projections, session |
| Event projections | 6-node cluster | CQRS read models, dashboard aggregations |

---

## 5. What's Already in Place

The production-hardening branch built the foundation:

| Capability | Status | Scaling Readiness |
|------------|--------|-------------------|
| Domain microservices (15) | Done | Ready for horizontal pod scaling |
| Per-domain DB schemas | Done | Ready for schema-per-cluster split |
| Per-domain service roles | Done | Ready for RBAC per cluster |
| NATS JetStream streams (16) | Done | Ready for clustering |
| Durable consumers with DLQ | Done | Ready for consumer groups |
| Event contracts (Pydantic) | Done | Schema evolution ready |
| Cache projector (CQRS) | Done | Needs horizontal scaling |
| Idempotent consumers | Done | Safe for at-least-once delivery |
| Rate limiting infrastructure | Done | Needs real limits per tier |
| JWT authentication | Done | Ready for API key extension |
| Health aggregation | Done | Ready for K8s probes |
| Caddy gateway routing | Done | Pattern transfers to ALB rules |

**The domain boundaries are correct.** The event contracts are clean. The isolation is real. This is the hard part — and it's done. Everything above is infrastructure scaling, not architectural redesign.

---

## 6. Migration Path

### Phase 1: Kubernetes + Read Replicas (Weeks)
- Move from Docker Compose on EC2 to EKS
- Add PostgreSQL read replicas (RDS)
- PgBouncer for connection pooling
- Redis Cluster (ElastiCache)
- NATS 3-node cluster
- **Result: 10x current capacity (~2,000-5,000 req/sec)**

### Phase 2: CQRS Work Queue + Event Store (Weeks)
- Broker work queue as Redis materialized view
- Shipment tracking as append-only event log
- Cache-first reads on all GET endpoints
- Consumer group scaling (N instances per domain)
- **Result: 50x current capacity (~10,000-25,000 req/sec)**

### Phase 3: Database Sharding + Regional (Months)
- Split PostgreSQL into 4 clusters by domain group
- Time-partition hot tables (shipments, entries, transit_events)
- Regional deployment (US-East, US-West, EU)
- Per-customer API gateway rate limiting
- **Result: 100x+ current capacity (50,000+ ops/sec)**

### Phase 4: Carrier Integration Pipeline (Months)
- Dedicated ingest pipeline for carrier EDI/API (FedEx, UPS, DHL)
- Message normalization layer (carrier format → platform events)
- Bulk batch processing for overnight reconciliation
- **Result: Billions of messages/day throughput**

---

## 7. Cost Estimate (Rough Order of Magnitude)

| Phase | Monthly AWS Cost | Timeline |
|-------|-----------------|----------|
| Current (1 x t4g.xlarge) | ~$150 | Running |
| Phase 1 (EKS + RDS + ElastiCache) | ~$3,000-5,000 | 2-4 weeks |
| Phase 2 (Scaled services + CQRS) | ~$5,000-8,000 | 2-4 weeks |
| Phase 3 (Multi-cluster + regional) | ~$15,000-25,000 | 2-3 months |
| Phase 4 (Full FedEx scale) | ~$30,000-60,000 | 2-3 months |

These are infrastructure costs only. The application code changes for Phase 2 (CQRS work queue, event store pattern) are the most significant development effort.

---

## 8. Risk Assessment

| Risk | Severity | Mitigation |
|------|----------|------------|
| Broker work queue query at scale | Critical | Phase 2 CQRS materialized view eliminates the problem |
| Event ordering guarantees | High | NATS JetStream provides per-subject ordering; partition by shipment_id |
| Cache consistency (stale reads) | Medium | 5s TTL on work queue acceptable; brokers expect near-real-time, not real-time |
| Cross-domain data consistency | Medium | Already event-driven with idempotent consumers; eventual consistency is the design |
| Carrier message format diversity | High | Normalization layer in Phase 4; start with FedEx TrackAPI format |
| GDPR / data residency | Medium | EU regional deployment in Phase 3; schema-per-domain already enables selective replication |

---

## 9. Global Deployment: The Regional Gate Model

### Design Philosophy: Data Carries Its Own Affinity

CockroachDB's REGIONAL BY ROW model gets something fundamentally right: **the data tells you where it lives.** Region affinity isn't a routing decision made by middleware — it's metadata on the entity itself. Every read, write, and event automatically goes to the right place because the record carries its own regional gate.

We adopt this pattern as the organizing principle for the entire global architecture. Every entity in the platform has a **regional gate** — metadata that determines where it's authoritative, where it's replicated, and how events flow.

### The Three Tiers of Regional Affinity

Not all entities have the same relationship to regions. There are exactly three patterns:

| Tier | Pattern | Regional Gate | Example |
|------|---------|---------------|---------|
| **Tier 1: Jurisdictional** | Single-region. Never leaves. | `home_region` = jurisdiction of filing | Entry filings, adjudication decisions, bonds, compliance screening results, broker work queues |
| **Tier 2: Corridor-Scoped** | Multi-region. Lives in 2-3 regions along the shipment corridor. | `home_region` + `export_region` + `transit_regions[]` | Canonical shipments, orders, documents, financial settlement |
| **Tier 3: Global Reference** | All regions need read access. Write-rare. | `owner_region` (writes) + global read replicas | Parties, products, HTS classifications, regulatory rules, carrier profiles |

This classification drives every architectural decision: persistence, event routing, replication, consistency model, and failure modes.

### The Canonical Shipment: Region-Gated Entity

When a carrier message is normalized into our canonical shipment model, **the regional gate is computed at creation and travels with the entity forever:**

```
CanonicalShipment:
  shipment_id: "SHP-2026-0312-A7X9"

  # ─── REGIONAL GATE (computed at creation, updated on itinerary change) ───
  home_region:      "us-east"          # Import destination — authoritative region
  export_region:    "apac"             # Origin — has export clearance authority
  transit_regions:  ["eu-west"]        # Leipzig hub — T1 transit declaration
  active_regions:   ["apac", "eu-west", "us-east"]  # Computed: union of all

  # ─── ROUTING MANIFEST (living itinerary) ───
  itinerary:
    - leg: 1, facility: "CAN-GZ", country: "CN", region: "apac",    role: "origin_hub"
    - leg: 2, facility: "LEJ",    country: "DE", region: "eu-west",  role: "transit_hub", customs: "T1"
    - leg: 3, facility: "MEM",    country: "US", region: "us-east",  role: "destination_hub"
    - leg: 4, facility: "ORD",    country: "US", region: "us-east",  role: "delivery"
  itinerary_version: 1                 # Incremented on every change
  itinerary_source: "routing_engine"   # or "carrier_update" or "scan_inference"

  # ─── CANONICAL DATA (normalized from carrier format) ───
  origin: { city: "Guangzhou", country: "CN", port: "CNGZH" }
  destination: { city: "Chicago", country: "US", port: "USORD" }
  carrier: "FEDEX"
  service_type: "INTERNATIONAL_PRIORITY"
  transport_mode: "AIR"
  status: "IN_TRANSIT"
```

**The `active_regions` field is the CRDB REGIONAL BY ROW concept.** It's not a routing table lookup — it's a property of the shipment itself. When any system component needs to know "who cares about this shipment?", the answer is on the record.

### Regional Gate for Every Entity Type

| Entity | Tier | Gate Fields | How Gate is Determined |
|--------|------|-------------|----------------------|
| **Canonical Shipment** | 2 | `home_region`, `export_region`, `transit_regions[]` | Destination country → home. Origin country → export. Itinerary analysis → transit. |
| **Entry Filing** | 1 | `home_region` | Jurisdiction where filed. US entry → us-east. EU declaration → eu-west. Immutable. |
| **Adjudication Decision** | 1 | `home_region` | Same as the entry it adjudicates. Never crosses regions. |
| **Bond** | 1 | `home_region` | Jurisdiction of the customs authority requiring the bond. |
| **Compliance Screening** | 1 | `home_region` | Jurisdiction performing the screening. OFAC → us-east. EU sanctions → eu-west. |
| **Broker Work Queue** | 1 | `home_region` | Broker's license jurisdiction. US CHB → us-east. EU AEO → eu-west. |
| **Order** | 2 | `home_region`, `origin_region` | Destination country → home. Origin country → origin_region (for tracking visibility). |
| **Commercial Documents** | 2 | `home_region`, `shared_regions[]` | Commercial invoice shared across corridor. Entry summary jurisdictional only. |
| **Financial Settlement** | 2 | `home_region`, `billing_region` | Duty payment → jurisdiction. Consolidated invoice → customer's billing region. |
| **Party (Importer/Exporter)** | 3 | `owner_region`, replicated globally | HQ country → owner. Read replicas everywhere. Identified by EIN/EORI, not PII. |
| **Product Catalog** | 3 | `owner_region`, replicated globally | Creator's region → owner. HTS classifications per-country but product is global. |
| **HTS Schedule** | 3 | `owner_region`, replicated globally | Published by customs authority. US HTS → us-east. EU TARIC → eu-west. Read everywhere. |
| **Regulatory Rules** | 3 | `owner_region`, replicated globally | Jurisdiction-specific but all regions need read access for pre-screening. |
| **Carrier Profile** | 3 | `owner_region`, replicated globally | Carrier's primary registration. Read everywhere. |

### Where CockroachDB Fits (and Where It Doesn't)

CRDB's model is elegant, but it solves a specific problem: **global consensus on writes to the same logical table.** We need to ask: which entities actually need that?

**Tier 1 (Jurisdictional) — CRDB adds no value.** Entry filings are never written from outside their jurisdiction. A regional PostgreSQL cluster with zero cross-region writes gives sub-5ms latency with no consensus overhead. CRDB would add Raft latency for zero benefit.

**Tier 2 (Corridor-Scoped) — CRDB adds complexity, not clarity.** The canonical shipment is written by multiple regions (APAC writes export events, US-East writes import events), but they're writing to **different fields** of the same logical entity. This isn't a write conflict — it's a merge. Event-sourcing handles this naturally: each region appends its events to the shipment's event log, and the materialized view is computed per-region from the events it has received. No consensus needed.

**Tier 3 (Global Reference) — CRDB GLOBAL tables are genuinely useful here.** Parties, products, HTS codes, and regulatory rules are read-heavy, write-rare, and needed everywhere with low latency. CRDB's GLOBAL table pattern (non-blocking reads in every region, consensus only on writes) is exactly right. A party profile update happens maybe once a week — paying 200ms write latency for global consistency is fine when reads are local and free.

**Recommendation: Hybrid persistence.**

```
┌──────────────────────────────────────────────────────────────┐
│                     PERSISTENCE LAYER                        │
│                                                              │
│  Tier 1 + 2: Regional PostgreSQL                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │ US-East  │  │ EU-West  │  │  APAC    │                   │
│  │ PG Clstr │  │ PG Clstr │  │ PG Clstr │                   │
│  │          │  │          │  │          │                   │
│  │ entries  │  │ entries  │  │ entries  │  ← Tier 1: local  │
│  │ adjudic  │  │ adjudic  │  │ adjudic  │    only, no       │
│  │ bonds    │  │ bonds    │  │ bonds    │    replication     │
│  │ shipments│  │ shipments│  │ shipments│  ← Tier 2: event  │
│  │ orders   │  │ orders   │  │ orders   │    sourced, async  │
│  └──────────┘  └──────────┘  └──────────┘    replication     │
│                                                              │
│  Tier 3: CockroachDB Global Cluster                          │
│  ┌──────────────────────────────────────────┐                │
│  │  GLOBAL tables: parties, products,       │                │
│  │  hts_schedule, regulatory_rules,         │                │
│  │  carrier_profiles                        │                │
│  │                                          │                │
│  │  Nodes in US-East, EU-West, APAC         │                │
│  │  Reads: local, non-blocking (<5ms)       │                │
│  │  Writes: global consensus (~200ms) — OK, │                │
│  │          these change rarely              │                │
│  └──────────────────────────────────────────┘                │
└──────────────────────────────────────────────────────────────┘
```

### Event Replication: Region-Aware NATS Subjects

Events carry the regional gate of their parent entity. The NATS subject hierarchy encodes region affinity:

```
# Region-scoped subjects (Tier 1 + 2 events)
clearance.{region}.shipment.{shipment_id}.tracking.{event_type}
clearance.{region}.declaration.{entry_id}.status.{event_type}
clearance.{region}.adjudication.{entry_id}.{event_type}

# Global subjects (Tier 3 events)
clearance.global.party.{party_id}.{event_type}
clearance.global.product.{product_id}.{event_type}
clearance.global.regulatory.{jurisdiction}.{event_type}
```

**Each regional NATS cluster subscribes to `clearance.{its-region}.**` + `clearance.global.**`.** Region-scoped events only flow to subscribed regions. Global events replicate everywhere via NATS supercluster gateways.

**The carrier adapter publishes corridor-scoped events to multiple regional subjects:**

```
Tracking event: SHP-12345 departs Guangzhou

Adapter reads manifest → active_regions: [apac, eu-west, us-east]

Publishes to:
  clearance.apac.shipment.SHP-12345.tracking.departed      ← local
  clearance.eu-west.shipment.SHP-12345.tracking.departed    ← via gateway
  clearance.us-east.shipment.SHP-12345.tracking.departed    ← via gateway
```

**Events that enter in one region but need selective distribution** — this is the core complexity. The pattern:

1. **Event origin** is always a single region (the carrier scan happened in Guangzhou → APAC).
2. **The event's parent entity** (the canonical shipment) carries `active_regions`.
3. **The publisher** (adapter or domain service) reads `active_regions` and publishes to each region's subject.
4. **NATS gateway replication** handles the cross-region transport — the publisher doesn't need to know network topology, only subject names.

This means: **adding a region to a shipment's gate is a data change, not an infrastructure change.** If a shipment gets rerouted through Dubai, the manifest update adds `"mena"` to `active_regions`, and the next event automatically publishes to `clearance.mena.**`. No NATS reconfiguration. No routing table update. The data drives the distribution.

### Itinerary Changes: The Living Manifest

The routing manifest is not static. Three sources of truth compete:

| Source | When | Reliability |
|--------|------|-------------|
| **Routing engine prediction** | At shipment creation | High for known carriers/lanes, uncertain for new corridors |
| **Carrier itinerary update** | When carrier provides routing data (EDI 990, API) | Authoritative but may arrive late |
| **Scan inference** | When tracking events reveal actual path | Ground truth but after-the-fact |

**The reconciliation model:**

```
Manifest lifecycle:

  CREATE (routing engine)
    │
    ├── Carrier provides itinerary → CONFIRM or UPDATE
    │
    ├── Tracking scan matches prediction → CONFIRM leg
    │
    ├── Tracking scan at unexpected facility → ADD region
    │     └── Publish ShipmentItineraryChangedEvent (v2)
    │         └── New region receives backfill of shipment context
    │
    ├── Expected scan window passes with no event → MARK UNCERTAIN
    │     └── Query carrier API for updated routing
    │     └── If rerouted: UPDATE manifest, DEACTIVATE old region
    │
    └── Carrier sends explicit reroute → UPDATE
          └── Publish ShipmentItineraryChangedEvent (v3)
          └── Old transit region receives cancellation
          └── New transit region receives activation + backfill
```

**Subscribe early, unsubscribe late.** When the routing engine predicts a Leipzig transit, EU-West starts receiving events immediately. If the shipment gets rerouted and skips Leipzig, EU-West receives a `ShipmentCorridorDeactivatedEvent` and cleans up. The cost of a few unnecessary tracking events is negligible compared to missing a transit declaration.

**The `itinerary_version` field** is critical. Every manifest update increments the version. Events carry the version they were published under. If a region receives events from version 1 and then a manifest update to version 2 that removes it, it knows all version-1 events are historical and can archive them. No ambiguity about which events were "real" versus "before we knew about the reroute."

### Corridor-Scoped Consistency: Event Sourcing, Not Consensus

The canonical shipment exists in multiple regions simultaneously. How do we keep them consistent without global consensus?

**We don't keep them identical. We keep them eventually consistent via event sourcing.**

Each region maintains its own materialized view of the shipment, built from the events it has received:

```
APAC's view of SHP-12345:
  status: EXPORT_CLEARED
  events: [created, picked_up, arrived_gz_hub, export_cleared, departed_gz]
  knows about: origin, export clearance
  doesn't know about: Leipzig transit, US import (hasn't happened yet)

EU-West's view of SHP-12345:
  status: IN_TRANSIT_TO_TRANSIT_HUB
  events: [created, departed_gz, ← replicated from APAC]
  knows about: inbound from APAC, T1 transit pending
  doesn't know about: US import (hasn't happened yet)

US-East's view of SHP-12345:
  status: IN_TRANSIT
  events: [created, departed_gz, ← replicated from APAC]
  knows about: expected arrival, import entry pre-filed
  doesn't know about: Leipzig transit details (doesn't need to)
```

**Each region's view is complete for its jurisdictional purpose.** The US broker doesn't need Leipzig T1 transit details to file the US entry. The EU transit declarant doesn't need the US duty calculation. The views diverge by design — they serve different regulatory purposes.

**Conflict-free by construction.** APAC writes export events. EU-West writes transit events. US-East writes import events. They never write to the same fields because each jurisdiction's customs process is independent. This isn't optimistic concurrency — it's domain-partitioned writes. No conflicts are possible.

### The Region-Aware Adapter

The carrier adapter is the bridge between the global carrier firehose and the regional architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                   CARRIER INGEST LAYER                       │
│                                                              │
│  ┌─────────────┐   ┌──────────────┐   ┌─────────────────┐  │
│  │ Carrier API  │   │  Normalize   │   │  Regional Gate  │  │
│  │ (FedEx,UPS,  │──→│  (carrier →  │──→│  Resolution     │  │
│  │  DHL, etc)   │   │  canonical)  │   │                 │  │
│  └─────────────┘   └──────────────┘   │ 1. Known shipment│  │
│                                        │    → read gate   │  │
│                                        │    from manifest │  │
│                                        │                 │  │
│                                        │ 2. New shipment  │  │
│                                        │    → routing     │  │
│                                        │    engine builds │  │
│                                        │    gate + itinry │  │
│                                        │                 │  │
│                                        │ 3. Unexpected   │  │
│                                        │    facility     │  │
│                                        │    → extend gate│  │
│                                        └────────┬────────┘  │
│                                                 │           │
│  ┌──────────────────────────────────────────────┼────────┐  │
│  │           ROUTING MANIFEST STORE              │        │  │
│  │       (DynamoDB Global Tables or              │        │  │
│  │        CockroachDB GLOBAL table)              │        │  │
│  │                                               │        │  │
│  │  SHP-12345 → {active_regions, itinerary, v3}  │        │  │
│  └───────────────────────────────────────────────┘        │  │
│                                                 │           │
│                    ┌────────────┬────────────────┤           │
│                    ▼            ▼                ▼           │
│              APAC NATS    EU-West NATS     US-East NATS     │
│              (export)     (transit)        (import)          │
└─────────────────────────────────────────────────────────────┘
```

The adapter is **stateless in compute** but reads from a **globally-available manifest store.** The manifest store is small (one record per active shipment), keyed by shipment_id, and needs sub-10ms reads globally. DynamoDB Global Tables or a CockroachDB GLOBAL table both work here — this is exactly the access pattern CRDB's GLOBAL tables are designed for.

### Regional Architecture (Detailed)

```
┌──────────────────────────────────────────────────────────────┐
│                    GLOBAL CONTROL PLANE                        │
│    DNS routing, customer mgmt, carrier adapter, monitoring    │
│                                                               │
│    ┌──────────────────────────────────────────────────┐       │
│    │  CockroachDB Global Cluster (Tier 3 reference)   │       │
│    │  Nodes in each region. GLOBAL tables.             │       │
│    │  Parties, products, HTS, regulatory rules         │       │
│    └──────────────────────────────────────────────────┘       │
│                                                               │
│    ┌──────────────────────────────────────────────────┐       │
│    │  Routing Manifest Store (DynamoDB Global Tables)  │       │
│    │  Shipment → regional gate, itinerary, version     │       │
│    └──────────────────────────────────────────────────┘       │
└──────────┬──────────────────┬──────────────────┬─────────────┘
           │                  │                  │
    ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐
    │   US-EAST   │    │   EU-WEST   │    │    APAC     │
    │             │    │             │    │             │
    │ K8s cluster │    │ K8s cluster │    │ K8s cluster │
    │             │    │             │    │             │
    │ PostgreSQL  │    │ PostgreSQL  │    │ PostgreSQL  │
    │ (Tier 1+2)  │    │ (Tier 1+2)  │    │ (Tier 1+2)  │
    │ • entries   │    │ • entries   │    │ • entries   │
    │ • adjudic   │    │ • adjudic   │    │ • adjudic   │
    │ • shipments │    │ • shipments │    │ • shipments │
    │ • bonds     │    │ • bonds     │    │ • bonds     │
    │             │    │             │    │             │
    │ NATS clstr  │    │ NATS clstr  │    │ NATS clstr  │
    │ clearance.  │    │ clearance.  │    │ clearance.  │
    │ us-east.**  │    │ eu-west.**  │    │ apac.**     │
    │ + global.** │    │ + global.** │    │ + global.** │
    │             │    │             │    │             │
    │ Redis clstr │    │ Redis clstr │    │ Redis clstr │
    │ (SSE work   │    │ (SSE work   │    │ (SSE work   │
    │  queues,    │    │  queues,    │    │  queues,    │
    │  projctns)  │    │  projctns)  │    │  projctns)  │
    │             │    │             │    │             │
    │ US brokers  │    │ EU AEOs     │    │ APAC agents │
    │ CBP/ACE     │    │ UCC/ICS2    │    │ CN/JP/SG    │
    │ US SMB cust │    │ EU SMB cust │    │ APAC cust   │
    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
           │                  │                  │
           └──────────────────┴──────────────────┘
                    NATS Supercluster Gateways
              (selective region-scoped replication)
```

### Consensus Latencies: Where They Apply

Cross-region round-trip times:

| Route | RTT |
|-------|-----|
| US-East ↔ US-West | ~60-80ms |
| US-East ↔ EU-West (Ireland) | ~80-120ms |
| US-East ↔ APAC (Singapore) | ~200-250ms |
| EU-West ↔ APAC | ~150-200ms |

**Where consensus is paid and why it's acceptable:**

| Operation | Consensus scope | Latency | Frequency | Acceptable? |
|-----------|----------------|---------|-----------|-------------|
| Entry filing (Tier 1 write) | Intra-region PostgreSQL | <5ms | High volume | Yes — this is the hot path |
| Tracking event publish (Tier 2) | Intra-region NATS R3 | <5ms | 50K/sec | Yes — local write, async replicate |
| Party profile update (Tier 3) | Global CRDB consensus | ~200ms | Rare (weekly) | Yes — read-heavy, write-rare |
| Routing manifest update | DynamoDB Global Tables | <10ms local | Rare (per-shipment lifecycle) | Yes |
| Cross-region event delivery | Async (no consensus) | 80-250ms | Continuous | Yes — not on critical path |

**We never pay global consensus on the hot path.** The only global consensus is Tier 3 reference data writes (party updates, product reclassifications) — these happen so rarely that 200ms is irrelevant.

### Data Residency and GDPR

The tiered model handles data residency precisely:

- **Tier 1 (Jurisdictional):** EU entry filings, broker data, adjudication — stays in EU-West. Never replicated. GDPR deletion is a single-region operation.
- **Tier 2 (Corridor-Scoped):** Cross-region shipment events carry shipment IDs and status codes, not PII. The canonical shipment in EU-West contains EU-relevant fields; the US-East version contains US-relevant fields. PII (consignee name, address) is **only in the home region's materialized view**, not in the replicated events.
- **Tier 3 (Global Reference):** Party records use business identifiers (EIN, EORI, DUNS) — not personal data. If a party record contains PII (contact person name), it's stored only in the `owner_region`'s CRDB node and served to other regions via non-blocking stale reads. GDPR erasure updates the owner region; stale reads expire naturally.

### The Global Read Layer: Customer Dashboards

SMB customers shipping globally need a unified view. Three options:

1. **Fan-out query** — frontend queries each region's API in parallel, merges results client-side. Simple. Works for customers with <1,000 active shipments per region. Latency = slowest region (~250ms to APAC).

2. **Global projection service** — a dedicated service consumes corridor-scoped events from all regions via NATS supercluster, maintains a global materialized view per customer. Serves dashboard queries from a single store (DynamoDB Global Tables or read-optimized PostgreSQL). Latency = local read. Required at scale.

3. **CDN-cached aggregates** — hourly pre-computed metrics (total shipments, clearance rate, cost summary) cached at the edge. For executive dashboards, not operational views.

### Global Deployment Cost Estimate

| Component | Specification | Monthly Cost |
|-----------|--------------|-------------|
| US-East (full region) | K8s + 4 PG clusters + NATS 3-node + Redis 6-node | $30-60K |
| EU-West (full region) | K8s + 4 PG clusters + NATS 3-node + Redis 6-node | $25-50K |
| APAC (full region) | K8s + 2-3 PG clusters + NATS 3-node + Redis 6-node | $15-30K |
| US-West (DR standby) | Standby K8s + PG replicas | $10-15K |
| CockroachDB Global (Tier 3) | 3-region, 9 nodes | $5-10K |
| DynamoDB Global Tables (manifests) | On-demand, 3-region | $1-3K |
| Global control plane | DNS, monitoring, carrier adapter | $2-5K |
| **Total** | | **$90-175K/mo** |

---

## Summary

The platform architecture is correctly designed for scale — both single-region and global.

**The Regional Gate Model** gives every entity in the platform an explicit regional affinity:
- **Tier 1 (Jurisdictional):** Entry filings, adjudication, bonds, compliance — single-region, no replication, sub-5ms writes. Regional PostgreSQL.
- **Tier 2 (Corridor-Scoped):** Canonical shipments, orders, documents — multi-region along the shipment corridor, event-sourced, eventually consistent. Regional PostgreSQL + selective NATS replication.
- **Tier 3 (Global Reference):** Parties, products, HTS codes, regulatory rules — read everywhere, write rarely, global consistency acceptable. CockroachDB GLOBAL tables.

**No global consensus on the hot path.** Customs is jurisdictional — writes are always region-local. Cross-region data flows are async event replication via NATS supercluster gateways, filtered by region-scoped subjects. The canonical shipment carries its own `active_regions` list, so adding or removing a region from a shipment's corridor is a data change, not an infrastructure change.

**The carrier adapter** bridges the global carrier firehose to the regional architecture by computing a regional gate for each shipment (from the routing engine at creation, carrier updates mid-flight, and scan inference as ground truth). The routing manifest — a lightweight global store — maps each shipment to its active regions and living itinerary, updated as routes change.

**The hardest problems in distributed systems — global consensus, cross-region write conflicts, distributed transactions — are problems we don't have.** The jurisdictional nature of customs clearance means the domain itself is pre-sharded by region. The architecture follows the domain.
