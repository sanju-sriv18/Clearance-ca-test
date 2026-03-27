# 2026-02-27 — Production Readiness Remediation Design

## Context

The 2025-02-14 Production Readiness Review identified 10 critical findings preventing production deployment of the Clearance Intelligence Platform. This design document specifies the full remediation plan to bring all 10 findings to production-grade resolution.

**Source review**: `docs/reviews/2025-02-14-production-readiness.md`

## Branch Strategy

- **Working branch**: `production-hardening` (created from current `main`)
- **Sub-branches**: `production-hardening/wp-{N}-{slug}` for each work package
- **Merge flow**: Sub-branches merge into `production-hardening` only. **NO MERGES INTO MAIN.**
- **Final delivery**: PR from `production-hardening` → `main`, requires human review
- **Agent isolation**: Each engineer agent operates in a git worktree to prevent conflicts

## Team Structure (13 Agents)

### Tier 1 — Leadership & Quality
| Agent | Type | Role |
|-------|------|------|
| team-lead | general-purpose | Orchestrates waves, assigns tasks, manages dependencies, produces summary report |
| architect-reviewer | architect-reviewer | Reviews every WP completion against findings, validates no regressions |
| consistency-validator | code-reviewer | Ensures code style, naming, import patterns, cross-WP consistency |
| qa-admin | qa-expert | Designs test strategy, validates E2E coverage, owns final proof-of-work test run |

### Tier 2 — Wave 1 Engineers (4 parallel)
| Agent | Type | Work Package |
|-------|------|-------------|
| decoupling-eng | backend-developer | WP-1: Legacy Monolith Decoupling |
| event-schema-eng | backend-developer | WP-2: Event Schema Validation |
| cache-eng | backend-developer | WP-3: Cache Projector Fix |
| security-eng | backend-developer | WP-4: CORS & Rate Limits + WP-7: Auth Middleware |

### Tier 3 — Wave 2 Engineers (3 parallel)
| Agent | Type | Work Package |
|-------|------|-------------|
| db-isolation-eng | database-administrator | WP-5: Database Isolation |
| domain-boundary-eng | backend-developer | WP-6: Cross-Domain SQL Elimination |
| security-eng (continues) | backend-developer | WP-7: Authentication Middleware |

### Tier 4 — Wave 3 Engineers (3 parallel)
| Agent | Type | Work Package |
|-------|------|-------------|
| ingestion-eng | backend-developer | WP-8: Ingestion Persistence |
| stub-replacement-eng | fullstack-developer | WP-9: Stub Replacement |
| arch-test-eng (qa-admin) | qa-expert | WP-10: Architecture Test Enforcement |

## Documentation Requirements

### Before Starting Work (ALL agents MUST):
1. Read `docs/reviews/2025-02-14-production-readiness.md`
2. Read `docs/development-standards.md`
3. Read relevant Serena project memories (project_overview, database-schema-and-data-layer, ARCHITECTURE-AUDIT-FINDINGS)
4. Read CLAUDE.md for conventions

### On Completion (ALL agents MUST):
1. Write a work package report to `docs/reports/WP-{N}-{title}.md` containing:
   - Finding addressed (verbatim from review)
   - Changes made (files modified/created/deleted)
   - Tests added/modified
   - How to verify (manual + automated)
   - Residual risks or known limitations
2. Update relevant Serena project memories if architectural patterns change

## Work Package Specifications

### WP-1: Legacy Monolith Decoupling (Finding #1)

**Finding**: "Microservice packages still import `app.*`, `app.knowledge.*`, and legacy ORM models. Any deployment requires the monolith environment, so services cannot operate independently."

**Goal**: Zero `app.*`, `app.knowledge.*` imports in `clearance_platform/`.

**Scope**:
- All files under `clearance_platform/` that import from `app/`
- `clearance_platform/shared/engines/` (LLM engine wrappers)
- `clearance_platform/shared/llm.py`, `shared/cache.py`
- `clearance_platform/gateway/` route files

**Actions**:
1. Audit all `from app.` imports in `clearance_platform/` tree
2. For each import, either:
   a. Move the needed code into `clearance_platform/shared/` (if it's a utility/model)
   b. Replace with a `clearance_platform`-internal equivalent
   c. Create an interface/protocol that decouples the dependency
3. Verify `app/` simulation actors and legacy code still function
4. Ensure `clearance_platform/` can be packaged independently

**Tests**:
- Architecture fitness test: `test_no_app_imports_in_clearance_platform`
- Existing test suite continues to pass
- New test: import `clearance_platform` in isolation (without `app/` on PYTHONPATH)

**File Ownership**: `clearance_platform/shared/engines/`, `clearance_platform/shared/llm.py`, `clearance_platform/shared/cache.py`, `clearance_platform/gateway/` (import modifications only)

---

### WP-2: Event Schema Validation (Finding #8)

**Finding**: "JetStream consumers deserialize into `BaseEvent` (extra fields allowed) and manually poke at dicts, so versioning/compatibility regressions go undetected."

**Goal**: Strict Pydantic event models with `extra="forbid"`, typed deserialization, schema versioning.

**Scope**:
- `clearance_platform/shared/events/base.py`
- `clearance_platform/shared/events/registry.py`
- All domain `events.py` files (15 domains)
- All domain `consumers.py` files (deserialization logic)

**Actions**:
1. Change `BaseEvent` from `extra="allow"` to `extra="forbid"`
2. Define typed event classes for every domain event with explicit fields
3. Add `schema_version: int` field with validation (reject unknown versions)
4. Update all consumers to deserialize into the specific typed event class (not generic BaseEvent)
5. Add event registry mapping NATS subjects → event classes
6. Update `event_bus.py` publish to validate outgoing events

**Tests**:
- Unit test for every event type (valid payload accepted, extra fields rejected)
- Integration test: publish event with unknown field → consumer rejects
- Integration test: publish event with wrong version → consumer rejects

**File Ownership**: `clearance_platform/shared/events/`, all `domains/*/events.py`, all `domains/*/consumers.py` (deserialization sections only)

---

### WP-3: Cache Projector Fix (Finding #7)

**Finding**: "Cache projector stores every event permanently in Redis with no TTL, no partitioning, and runs full-stream replays. Read APIs repeatedly call `get_cached()` but discard hits, so PostgreSQL remains the bottleneck."

**Goal**: Bounded Redis cache with TTLs, partitioned keys, read handlers that actually use cached data.

**Scope**:
- `clearance_platform/cache_projector/projector.py`
- `clearance_platform/shared/projector/base.py`
- `clearance_platform/shared/projector/registry.py`
- All domain `projector.py` files
- Route handlers that call `get_cached()` or equivalent

**Actions**:
1. Add configurable TTL (default 300s) to all Redis `SET` operations in projectors
2. Add key prefix partitioning: `{domain}:entity:{type}:{id}`
3. Fix read API handlers to return cached data on cache hit (not discard and re-query DB)
4. Replace full-stream replay with incremental rebuild (last-processed sequence number)
5. Add cache eviction monitoring endpoint
6. Add `CACHE_TTL_SECONDS` env var (configurable per deployment)

**Tests**:
- Unit test: cached entity expires after TTL
- Unit test: read handler returns cached data without DB query
- Integration test: cache rebuild from last sequence (not full replay)

**File Ownership**: `cache_projector/`, `shared/projector/`, all domain `projector.py` files, route handlers (cache read sections only)

---

### WP-4: CORS & Rate Limit Hardening (Finding #9)

**Finding**: "`platform_service` enables `allow_origins=["*"]` with credentials, and most routers allow `50k/minute` without authentication, exposing the entire API surface to abuse."

**Goal**: Proper CORS allowlist, tiered rate limits, CSRF protection.

**Scope**:
- `app/main.py` (CORS middleware section)
- `clearance_platform/platform_service/main.py` (CORS)
- All route files with `@limiter.limit()` decorators

**Actions**:
1. Replace `allow_origins=["*"]` with configurable `ALLOWED_ORIGINS` env var (comma-separated list)
2. Default: `["http://localhost:4001"]` in dev, explicit domain in production
3. Set tiered rate limits:
   - Admin/seed endpoints: 10/minute
   - LLM endpoints (chat, classify, analyze): 30/minute
   - Write endpoints (POST/PUT/DELETE): 100/minute
   - Read endpoints (GET): 500/minute
4. Remove `allow_credentials=True` when origin is `*`
5. Add `SameSite=Strict` to any session cookies

**Tests**:
- Unit test: request from disallowed origin → rejected
- Unit test: rate limit exceeded → 429
- E2E test: CORS headers correct in browser

**File Ownership**: `app/main.py` (middleware section), `clearance_platform/platform_service/main.py` (CORS), rate limit decorators across routes

---

### WP-5: Database Isolation (Finding #2)

**Finding**: "All domain containers connect to the same `clearance_engine` Postgres database using schema prefixes only. Bad actors or runaway migrations in one domain can corrupt the rest."

**Goal**: Per-domain PostgreSQL schemas with enforced role-based access control.

**Scope**:
- All domain `models.py` files (15 domains)
- `clearance_platform/shared/database.py`
- `alembic/` (new migration)
- `data/init.sql`

**Actions**:
1. Create Alembic migration to move all domain tables from `public` to their respective schemas
2. Update every domain model's `__table_args__` to specify `schema="{domain_schema}"`
3. Create per-domain PostgreSQL roles (`svc_{domain}`) in `init.sql`
4. Grant each role USAGE + CRUD only on its own schema
5. Revoke cross-schema access
6. Update `database.py` to create domain-specific session factories with appropriate role
7. Update `docker-compose.microservices.yml` DATABASE_URL per service

**Tests**:
- Integration test: domain A's session cannot SELECT from domain B's tables
- Migration test: up/down migration works cleanly
- All existing tests pass with new schema layout

**File Ownership**: All `domains/*/models.py`, `shared/database.py`, `alembic/`, `data/init.sql`

---

### WP-6: Cross-Domain SQL Elimination (Finding #5)

**Finding**: "Gateways directly query or mutate other domains' tables. `shared/model_registry` auto-imports every model into a single metadata registry, erasing bounded-context isolation."

**Goal**: Zero cross-domain SQL. Gateways use cached projections or service HTTP calls.

**Scope**:
- `clearance_platform/gateway/*_routes.py`
- `clearance_platform/shared/model_registry.py`
- Any service that imports models from another domain

**Actions**:
1. Audit all gateway routes for cross-domain model imports and SQL joins
2. For each cross-domain query, replace with:
   a. Redis-cached projection (preferred for read-heavy dashboards)
   b. Service-to-service HTTP call with typed contract (for mutations or complex queries)
3. Remove `model_registry.py` auto-import pattern
4. Replace with explicit per-domain model imports only where needed (within the domain boundary)
5. Add architecture test: no gateway route imports models from more than one domain

**Tests**:
- Architecture test: no cross-domain model imports in gateway routes
- Runtime test: gateway dashboard returns correct aggregated data via projections
- Performance test: gateway response time not degraded vs direct SQL

**File Ownership**: `gateway/*_routes.py`, `shared/model_registry.py`

---

### WP-7: Authentication Middleware (Finding #3)

**Finding**: "High-impact APIs accept arbitrary requests with no auth or CSRF protection while allowing destructive operations."

**Goal**: JWT-based auth on all endpoints. Role-based access control. Admin endpoint protection.

**Scope**:
- New module: `clearance_platform/shared/auth/`
- All route files (add `Depends(require_auth)`)
- Frontend: login page, token management in API client

**Actions**:
1. Create `clearance_platform/shared/auth/` with:
   - `middleware.py`: JWT validation middleware
   - `dependencies.py`: `require_auth`, `require_role(role)` FastAPI dependencies
   - `models.py`: User, Session, Role models
   - `routes.py`: `/api/auth/login`, `/api/auth/logout`, `/api/auth/refresh`
   - `config.py`: `JWT_SECRET`, `JWT_EXPIRY`, `ADMIN_USERS` from env
2. Add `Depends(require_auth)` to all route handlers
3. Add `Depends(require_role("admin"))` to seed/admin endpoints
4. Frontend: Add login page, store JWT in httpOnly cookie, add Authorization header to all API calls
5. Seed a default admin user for development

**Tests**:
- Unit test: unauthenticated → 401
- Unit test: wrong role → 403
- Unit test: valid JWT → 200
- Unit test: expired JWT → 401
- E2E test: login → navigate → logout flow in browser

**File Ownership**: New `shared/auth/` module, `Depends()` additions to all route files, frontend login components

---

### WP-8: Ingestion Persistence (Finding #4)

**Finding**: "Ingestion pipelines only log results; TODOs for DB persistence remain. Restricted-party lists, HTS data, and tariff references are never updated."

**Goal**: All 7 pipelines persist to database. Server-side dedup with hash tracking.

**Scope**:
- `clearance_platform/shared/ingestion/` (all 7 pipeline files)
- Domain models for reference data tables
- New `ingestion_metadata` tracking table

**Actions**:
1. Implement `upsert()` in each pipeline to write to the correct domain table:
   - OFAC SDN → `compliance.restricted_parties`
   - HTSUS → `trade_intel.htsus_headings`
   - AD/CVD → `trade_intel.adcvd_orders`
   - Exchange rates → `trade_intel.exchange_rates`
   - UFLPA → `compliance.restricted_parties` (with `list_source=UFLPA`)
   - CSL → `compliance.restricted_parties` (with appropriate `list_source`)
   - Trade remedies → `trade_intel.trade_remedies`
2. Create `ingestion_metadata` table tracking per-pipeline: last_run_at, last_hash, records_added, records_updated, records_deleted
3. Replace in-memory dedup with DB-backed hash comparison
4. Add Alembic migration for `ingestion_metadata` table
5. Wire scheduler registration in app startup

**Tests**:
- Integration test: run pipeline → data in DB
- Integration test: run again with same data → no duplicates (dedup works)
- Integration test: run with changed data → updates applied
- Integration test: `ingestion_metadata` correctly updated

**File Ownership**: `shared/ingestion/`, new migration, domain reference data models (additions only)

---

### WP-9: Stub Replacement (Finding #10)

**Finding**: "Financial settlement listing APIs return empty data; compliance dashboards rely on live cross-domain joins; ingestion dedup is in-memory only."

**Goal**: All placeholder endpoints return real data with real service logic.

**Scope**:
- All domain `routes.py` and `service.py` with stub responses
- Frontend components that consume these endpoints

**Actions**:
1. Audit every route handler across all 15 domains for placeholder responses:
   - `{"status": "ok"}`, `{"id": "placeholder"}`, `[]`, `{"event": "recorded"}`
2. Replace each stub with:
   a. Real service method call
   b. DB persistence
   c. Event emission (where applicable)
   d. Proper error handling
3. Focus areas (from review):
   - Financial settlement listing → real duty/fee aggregation
   - Compliance dashboard → use cached projections (from WP-3/WP-6)
   - Cargo HU endpoints → real CRUD with DB persistence
4. Frontend: Add "Data pending" skeleton states for any endpoint where backend data genuinely isn't available yet

**Tests**:
- E2E test for each previously-stubbed endpoint
- Verify response contains real data (not placeholder)
- Verify state changes persist across requests

**File Ownership**: All domain `routes.py`, `service.py` (business logic), frontend components consuming stub endpoints

---

### WP-10: Architecture Test Enforcement (Finding #6)

**Finding**: "`tests/architecture/test_microservice_structure.py` explicitly states the suite is expected to fail, so none of the architectural promises are enforced in CI."

**Goal**: All architecture tests pass. No `xfail` markers. Added to CI gate.

**Scope**:
- `tests/architecture/` (all test files)
- CI configuration

**Actions**:
1. Remove all `xfail`, "expected to fail", and `skip` markers from architecture tests
2. Fix any tests that fail after removing markers (coordinating with other WP completions)
3. Add new architecture tests for:
   - Auth middleware presence on all routes
   - Cache TTL presence in projectors
   - Event schema strictness (no `extra="allow"`)
   - No cross-domain SQL in gateways
4. Add architecture test suite to CI pipeline as **blocking gate**
5. Update CI config to run `pytest tests/architecture/ -v --tb=short` on every PR

**Tests**:
- `pytest tests/architecture/ -v` → 0 failures, 0 skipped
- CI pipeline blocks on architecture test failure

**File Ownership**: `tests/architecture/`, CI config files

---

## Wave Execution Plan

```
Wave 1 (4 parallel agents):
  WP-1 (decoupling-eng)     ─┐
  WP-2 (event-schema-eng)   ─┤── All run in parallel
  WP-3 (cache-eng)          ─┤   No inter-dependencies
  WP-4 (security-eng)       ─┘
     │
     ▼ Review checkpoint: architect-reviewer + consistency-validator
     │ Merge all WP branches → production-hardening
     │
Wave 2 (3 parallel agents):
  WP-5 (db-isolation-eng)    ─┐
  WP-6 (domain-boundary-eng) ─┤── All run in parallel
  WP-7 (security-eng)        ─┘   Depends on WP-1 (imports clean)
     │
     ▼ Review checkpoint: architect-reviewer + consistency-validator
     │ Merge all WP branches → production-hardening
     │
Wave 3 (3 parallel agents):
  WP-8 (ingestion-eng)        ─┐
  WP-9 (stub-replacement-eng) ─┤── All run in parallel
  WP-10 (qa-admin)             ─┘   Depends on WP-5,6,7 (isolation done)
     │
     ▼ Final review: all quality agents
     │ Merge all WP branches → production-hardening
     │
Final: QA admin runs full E2E suite
       Summary report generated
       PR: production-hardening → main (human review required)
```

## E2E Proof Requirements

1. **Browser navigability**: Every surface loads, all navigation links work
2. **Functional proof**: Each finding has E2E test proving remediation
3. **UI prototype match**: Screenshot comparison against `docs/ux-audit-screenshots/` baseline (<2% diff threshold)
4. **Placeholder treatment**: "Data pending" skeleton states where backend data unavailable — never blank/broken
5. **Auth flow**: Login → navigate → logout works in browser
6. **Performance**: No endpoint >5s response (except LLM calls)

## Deliverables

1. 10 work package reports: `docs/reports/WP-{N}-{title}.md`
2. Summary report: `docs/reports/production-readiness-remediation-summary.md`
3. Passing test suite output embedded in summary
4. Visual regression baseline: `docs/ux-audit-screenshots/baseline/`
5. PR: `production-hardening` → `main`
