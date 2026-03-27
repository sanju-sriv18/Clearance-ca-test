# 2025-02-14 – Production Readiness Review

## Findings
1. **Legacy Monolith Coupling** – Microservice packages still import `app.*`, `app.knowledge.*`, and legacy ORM models (`shared/operational_models.py`, `shared/legacy/*`, engine and service facades). Any deployment requires the monolith environment, so services cannot operate independently.
2. **Shared Database Instead of Isolated Stores** – All domain containers connect to the same `clearance_engine` Postgres database using schema prefixes only (`docker-compose.microservices.yml`, `shared/database.py`). Bad actors or runaway migrations in one domain can corrupt the rest and the single DB cannot sustain tens of millions of daily writes.
3. **Unauthenticated High-Impact APIs** – `/api/admin/seed*`, `/api/chat/stream`, `/api/session/*`, `/api/financial/demurrage`, `/api/adjudication/*`, etc., accept arbitrary requests with no auth or CSRF protection while allowing destructive operations (data wipes, LLM tool execution, shipment mutations).
4. **Compliance & Reference Feeds Not Persisted** – Ingestion pipelines (`shared/ingestion/*.py`) only log results; TODOs for DB persistence remain. Restricted-party lists, HTS data, and tariff references are never updated, rendering compliance engines inaccurate.
5. **Cross-Domain SQL Joins & Imports** – Gateways (e.g., `platform_routes`, `financial_settlement`, `consolidation`) directly query or mutate other domains’ tables. `shared/model_registry` auto-imports every model into a single metadata registry, erasing bounded-context isolation.
6. **Architecture Tests Expected to Fail** – `tests/architecture/test_microservice_structure.py` explicitly states the suite is expected to fail, so none of the architectural promises are enforced in CI.
7. **Cache Projector & Redis Usage Unsustainable** – Cache projector stores every event permanently in Redis with no TTL, no partitioning, and runs full-stream replays. Read APIs repeatedly call `get_cached()` but discard hits, so PostgreSQL remains the bottleneck.
8. **Event Schema Validation Missing** – JetStream consumers deserialize into `BaseEvent` (extra fields allowed) and manually poke at dicts, so versioning/compatibility regressions go undetected.
9. **CORS/Ratelimits Open** – `platform_service` enables `allow_origins=["*"]` with credentials, and most routers allow `50k/minute` without authentication, exposing the entire API surface to abuse.
10. **Placeholder/Stub Endpoints** – Financial settlement listing APIs return empty data; compliance dashboards rely on live cross-domain joins; ingestion dedup is in-memory only, so restarts reload duplicates.

## Remediation Priorities
- Complete the microservice extraction: move engines/services/models out of `app.*`, drop legacy bridges, and give each service its own deployable artifact along with isolated databases or at least separate Postgres instances per domain.
- Implement authentication/authorization across every API, restrict admin endpoints, and tighten CORS/rate limits.
- Finish ingestion persistence for HTSUS, OFAC/BIS lists, etc., and persist dedup metadata server-side.
- Replace cross-domain SQL with event-driven projections (Redis/CQRS) or service-to-service calls with contracts; enforce this through passing architecture tests in CI.
- Bound Redis cache growth and ensure read handlers actually use cached projections.

## Exhaustiveness Check
I re-read the entire backend `clearance_platform` tree (domains, shared, platform_service), the ingestion layer, event bus, cache projector, gateway routes, and frontend API usage. The findings above reflect issues seen across routing, services, ingestion, infra, and testing. While large codebases can always hide additional defects, these items capture every architectural and production-readiness flaw observed during this review. EOF
