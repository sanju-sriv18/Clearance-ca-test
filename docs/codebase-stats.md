# Codebase Statistics

> Generated: 2026-02-27 | Branch: `main`

## Overview

| Metric | Value |
|--------|-------|
| Project age | 29 days (Jan 30 – Feb 27, 2026) |
| Commits | 233 |
| Contributors | 1 (+ Claude co-author) |
| Backend framework | FastAPI (Python 3.12, uv) |
| Frontend framework | Vite + React (TypeScript) |
| Architecture | Domain-driven microservices with API gateway |

## Code Volume

| Language | Files | Lines |
|----------|------:|------:|
| Python (app + platform) | 443 | 98,081 |
| TypeScript / TSX | 149 | 34,125 |
| SQL | — | 375 |
| **Total application code** | **592** | **132,581** |
| Python (tests) | 103 | 36,680 |

## Testing

| Suite | Files | Tests | Lines |
|-------|------:|------:|------:|
| Python (pytest) | 103 | 2,200+ | 36,680 |
| E2E (Playwright) | 10 | ~240 | — |
| **Total test code** | **113** | **2,440+** | — |

Test code represents **28%** of the Python codebase.

## Microservice Domains

15 bounded contexts, each with its own models, routes, consumers, and database schema:

| Domain | Python LOC | Endpoints | Events | Description |
|--------|----------:|----------:|-------:|-------------|
| declaration_management | 6,708 | 47 | 14 | Entry filings, broker queue, checklist, export declarations, AI drafting, classification chat |
| shipment_lifecycle | 3,254 | 14 | 4 | Shipment creation, status tracking, routing, analysis |
| document_management | 2,378 | 10 | 3 | Document upload, requirements, validation, generation |
| trade_intelligence | 2,094 | 10 | 3 | HS classification, tariff calculation, FTA eligibility, landed cost |
| order_management | 1,981 | 10 | 3 | Purchase orders, line items, order-to-shipment linking |
| exception_management | 1,887 | 9 | 4 | Exception queue, resolution workflows, AI-assisted triage |
| financial_settlement | 1,789 | 6 | 4 | Duty/fee calculation, bonds, drawback, reconciliation |
| cargo_handling_units | 1,406 | 7 | 7 | Containers, pallets, crates; cage management, holds, GO deadlines |
| consolidation | 1,358 | 8 | 4 | MAWB/MBL grouping, multi-modal routing, transport legs |
| party_management | 1,301 | 10 | 4 | Importers, exporters, consignees, POA, trust programs |
| regulatory_intelligence | 1,231 | 11 | 2 | Regulatory signals, scenario modeling, portfolio impact |
| customs_adjudication | 1,214 | 4 | 6 | CBP decisions, CF-28/CF-29 issuance, exam, hold/release |
| compliance | 1,160 | 4 | 2 | DPS screening (14 federal lists), PGA flagging, UFLPA risk |
| product_catalog | 1,130 | 6 | 3 | Product master data, HS pre-classification |
| supply_chain_disruptions | 854 | 7 | 3 | Disruption detection, impact assessment |
| **Total** | **28,745** | **153** | **67** | — |

## Infrastructure

| Component | Count | Details |
|-----------|------:|---------|
| Docker services (dev) | 6 | backend, frontend, postgres, redis, nats, qdrant |
| Docker services (production) | 7 | + Caddy reverse proxy |
| Compose files | 3 | `docker-compose.yml`, `docker-compose.prod.yml`, `docker-compose.microservices.yml` |
| Dockerfiles | 3 | Backend, frontend, microservice base |
| Terraform files | 3 | EC2 provisioning in `infra/` |
| Kubernetes manifests | — | Alternative deployment in `k8s/` |
| Caddy config | 2 | Dev + production (HTTPS, basic auth, SSE flush) |

## Key Technologies

| Layer | Technology |
|-------|------------|
| API framework | FastAPI + Pydantic v2 |
| Database | PostgreSQL (domain-scoped schemas) |
| Cache / pub-sub | Redis |
| Event streaming | NATS JetStream |
| Vector search | Qdrant |
| Frontend | React 18 + TypeScript + Vite |
| E2E testing | Playwright |
| Reverse proxy | Caddy |
| Infrastructure as code | Terraform (AWS) |
| Package management | uv (Python), npm (JS) |
| Containerization | Docker Compose |

## Frontend Surfaces

| Surface | Screens | Components | AI Assistant | Description |
|---------|--------:|----------:|:------------|-------------|
| Broker | 8 | 7 | BrokerAssistant | Dashboard, queue, entry detail (hierarchical checklist, classification hub), authority inquiries, communications, export declarations, regulatory intel, party management |
| Platform | 15 | 27 | OperatorAssistant | Control tower, active shipments, shipment/entry/order detail, regulatory intel + signal review, exception resolution, entity screening (14 lists), product analysis, trade lanes, compliance dashboard, education center, simulation control |
| Shipper | 9 | 17 | — | Orders (list/create/detail), shipments (list/detail), products (catalog/add/detail), resolution center |
| Buyer | 3 | 5 | — | Storefront, product page with landed cost, checkout with duty breakdown |
| **Total** | **35** | **56** | **2** | — |

## Analytical Engines

| Engine | Name | LLM | Streaming | Key Capability |
|--------|------|:---:|:---------:|----------------|
| E0 | Pre-Clearance | Yes | No | Agentic tool-calling loop (5 tools), CLEAR/FLAG/HOLD decisions |
| E1 | Classification | Yes | Yes (SSE) | Three-step HS classification (enrichment → data gathering → GRI reasoning) |
| E2 | Tariff & Landed Cost | No | No | 6 jurisdiction regimes (US, EU, CN, BR, IN, UK) |
| E3 | Compliance Screening | No | No | PGA (20+ agencies) + DPS (14 federal lists) + UFLPA |
| E4 | FTA Qualification | No | No | USMCA rules-of-origin + RVC calculation |
| E5 | Exception Analysis | Yes | Yes (SSE) | CROSS rulings search + LLM response drafting |
| E6 | Regulatory Intelligence | No | No | Signal feed, scenario modeling, portfolio impact |
| E7 | Document Intelligence | No | No | Requirements lookup + generation |

## Database

- **Engine**: PostgreSQL with domain-scoped schemas
- **Extensions**: `pg_trgm` (trigram similarity), `btree_gin` (GIN indexing)
- **Schemas**: One per microservice domain (`cargo`, `consolidation`, `declaration`, `shipment`, etc.) plus `public` for the monolith gateway
- **Event idempotency**: `processed_events` table in each domain schema
