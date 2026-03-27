# Local Development Deployment

> **Author:** Simon Kissler (simon.kissler@accenture.com) | **Co-Author:** Claude Code (Anthropic)

This guide covers setting up and running the Clearance Intelligence Engine locally using Docker Compose.

## Prerequisites

- **Docker** (with Docker Compose v2)
- **Git**
- At least one LLM API key (Anthropic or OpenAI)

Optional (for running tools outside Docker):

- **Python 3.12+** with [uv](https://docs.astral.sh/uv/)
- **Node.js 20+** with npm

## Architecture Overview

The local dev stack runs six containers:

| Service      | Image              | Host Port | Container Port | Purpose                      |
|--------------|--------------------|-----------|----------------|------------------------------|
| `postgres`   | postgres:16-alpine | 5440      | 5432           | Primary database             |
| `redis`      | redis:7-alpine     | 6381      | 6379           | Cache / session store        |
| `qdrant`     | qdrant/qdrant      | 6336/6337 | 6334/6335      | Vector search                |
| `nats`       | nats:2.10-alpine   | 4222/8222 | 4222/8222      | Event bus (JetStream)        |
| `backend`    | ./backend          | 4000      | 4000           | FastAPI API server           |
| `frontend`   | ./frontend         | 4001      | 4001           | Vite React dev server        |

NATS, Redis, and Qdrant ports can be overridden via `NATS_PORT`, `REDIS_PORT`, `QDRANT_HTTP_PORT` env vars to avoid conflicts with other services on the same machine.

There is no reverse proxy in dev. The frontend connects directly to the backend at `http://localhost:4000/api`, and CORS is set to `*`.

## Quick Start

```bash
# 1. Clone and enter the project
git clone https://github.com/warum26/clearance_vibe.git
cd clearance_vibe/clearance-engine

# 2. Create environment file
cp .env.example .env
# Edit .env and add at least one API key:
#   ANTHROPIC_API_KEY=sk-ant-...
#   OPENAI_API_KEY=sk-...

# 3. Start all services
make up
# Or to force a fresh build:
make up-build

# 4. Run database migrations
make migrate

# 5. Seed reference data
make seed
```

The app is now running:

- **Frontend:** http://localhost:4001
- **Backend API:** http://localhost:4000/api
- **Health check:** http://localhost:4000/health
- **API docs (Swagger):** http://localhost:4000/docs (only when `DEBUG=true` in `.env`)
- **NATS monitoring:** http://localhost:8222

### Frontend Surfaces

The frontend has four actor-specific surfaces. Use the hamburger menu (top-left) to switch between them:

| Surface | Default Screen | Actor | Description |
|---------|----------------|-------|-------------|
| **Platform** | Control Tower | Operations team | Compliance monitoring, shipment tracking, exception resolution, regulatory intelligence, entity screening, simulation control |
| **Broker** | Dashboard | Customs broker | Entry filing queue, document checklist, CBP response drafting, classification review, export declarations, party management |
| **Shipper** | Orders | Exporter/importer | Order creation, shipment tracking, product catalog, landed cost analysis, document upload |
| **Buyer** | Shop Front | End consumer | Product browsing with transparent landed cost pricing and checkout |

The Broker and Platform surfaces include an AI assistant panel (right sidebar) that supports streaming chat with context-aware tool execution.

## Environment Variables

The `.env` file is loaded by both Docker Compose and the backend. Key variables:

| Variable                  | Default                                      | Description                           |
|---------------------------|----------------------------------------------|---------------------------------------|
| `DATABASE_URL`            | `postgresql+asyncpg://clearance:clearance_dev@localhost:5440/clearance_engine` | Async DB connection (host tools)      |
| `DATABASE_SYNC_URL`       | `postgresql://clearance:clearance_dev@localhost:5440/clearance_engine`          | Sync DB connection (Alembic)          |
| `REDIS_URL`               | `redis://localhost:6381/0`                   | Redis URL (host tools)                |
| `QDRANT_URL`              | `http://localhost:6336`                      | Qdrant URL (host tools)               |
| `NATS_URL`                | `nats://localhost:4222`                      | NATS JetStream URL (host tools)       |
| `ANTHROPIC_API_KEY`       | *(empty)*                                    | Anthropic API key                     |
| `OPENAI_API_KEY`          | *(empty)*                                    | OpenAI API key                        |
| `LLM_PRIMARY_PROVIDER`    | `anthropic`                                  | Primary LLM provider                  |
| `LLM_PRIMARY_MODEL`       | `claude-sonnet-4-5-20250929`                 | Primary model                         |
| `LLM_FALLBACK_PROVIDER`   | `openai`                                     | Fallback LLM provider                 |
| `LLM_FALLBACK_MODEL`      | `gpt-4o`                                     | Fallback model                        |
| `DEBUG`                    | `false`                                      | Enables Swagger docs, verbose logging |
| `LOG_LEVEL`               | `INFO`                                       | Python log level                      |
| `SIM_AUTO_START`           | `false`                                      | Auto-start simulation on boot         |
| `SIM_TIME_RATIO`           | `10`                                         | Simulation time multiplier            |
| `ALLOWED_ORIGIN`           | `http://localhost:4001`                      | CORS allowed origin (`*` in dev compose) |
| `LLM_TEMPERATURE`          | `0.1`                                        | LLM sampling temperature              |
| `LLM_MAX_TOKENS`           | `4096`                                       | Max tokens per LLM response           |
| `REGULATORY_MONITOR_ENABLED` | `false`                                    | Enable regulatory feed monitor        |
| `REGULATORY_MONITOR_INTERVAL_HOURS` | `2`                                 | Monitor polling interval (hours)      |
| `REGULATORY_MONITOR_EXTRACTION_MODEL` | *(empty)*                         | Override LLM model for signal extraction (empty = use primary) |
| `ANALYSIS_WATCHDOG_ENABLED` | `false`                                      | Enable background analysis watchdog (analysis runs on-demand by default) |
| `ANALYSIS_WATCHDOG_INTERVAL_SECONDS` | `15`                              | Watchdog polling interval (seconds)   |
| `ANALYSIS_WATCHDOG_BATCH_SIZE` | `5`                                      | Shipments analyzed per cycle          |
| `CSL_INGESTION_ENABLED`   | `true`                                        | Enable Consolidated Screening List ingestion from trade.gov |
| `CSL_INGESTION_INTERVAL_HOURS` | `6.0`                                   | Hours between CSL refresh cycles      |

**Note:** In the dev `docker-compose.yml`, `REGULATORY_MONITOR_ENABLED` is set to `true` by default. The monitor requires at least one LLM API key to extract signals from feeds.

**Note:** The Analysis Watchdog is disabled by default. Analysis runs on-demand when users view an entry detail page. Enable the watchdog only for background catch-up on bulk imports.

**Note:** CSL Ingestion is enabled by default. It fetches the Consolidated Screening List from trade.gov and loads 14 US federal restricted party lists into the database every 6 hours. The DPSScreener (E3) uses this data for live denied entity matching. No API key required.

**Note:** When running inside Docker Compose, the `DATABASE_URL`, `REDIS_URL`, and `QDRANT_URL` are overridden by the `environment` block in `docker-compose.yml` to use container hostnames (`postgres`, `redis`, `qdrant`). The `.env` values are only used when running tools directly on the host.

## Make Commands Reference

All commands run from `clearance-engine/`.

### Service Lifecycle

| Command            | Description                                   |
|--------------------|-----------------------------------------------|
| `make up`          | Start all containers (detached)               |
| `make up-build`    | Start with fresh image builds                 |
| `make down`        | Stop and remove containers                    |
| `make restart`     | Restart all containers                        |
| `make logs`        | Tail logs from all services                   |
| `make logs-backend`| Tail backend logs only                        |
| `make logs-frontend`| Tail frontend logs only                      |

### Database

| Command                           | Description                              |
|-----------------------------------|------------------------------------------|
| `make migrate`                    | Run all pending Alembic migrations       |
| `make migrate-new msg="add foo"`  | Generate a new auto-migration            |
| `make migrate-down`               | Roll back one migration                  |

### Data Seeding

| Command              | Description                                    |
|----------------------|------------------------------------------------|
| `make seed`          | Seed all reference data                        |
| `make seed-htsus`    | HTSUS tariff lines (US)                        |
| `make seed-htsus-chapters` | HTSUS chapter metadata                   |
| `make seed-eu`       | EU tariff schedule                             |
| `make seed-cn`       | China tariff schedule                          |
| `make seed-br`       | Brazil tariff schedule                         |
| `make seed-in`       | India tariff schedule                          |
| `make seed-sdn`      | Restricted parties seed data (14 US federal lists — representative subset for offline dev; live data comes from CSL ingestion) |
| `make seed-301`      | Section 301 tariff lists                       |
| `make seed-232`      | Section 232 scope (steel/aluminum)             |
| `make seed-ieepa`    | IEEPA country rates                            |
| `make seed-adcvd`    | AD/CVD orders                                  |
| `make seed-fees`     | CBP fee schedules                              |
| `make seed-taxes`    | Tax rates by jurisdiction                      |
| `make seed-tax-regimes` | Landed-cost computation templates           |
| `make seed-pga`      | PGA agency jurisdiction mappings               |
| `make seed-fta`      | FTA rules of origin                            |
| `make seed-cross`    | CBP CROSS binding rulings                      |
| `make seed-signals`  | Regulatory signal samples                      |
| `make seed-demo`     | Demo shipment entries                          |
| `make seed-fx`       | Exchange rates                                 |

### Testing

| Command                | Description                     |
|------------------------|---------------------------------|
| `make test`            | Run full backend test suite     |
| `make test-unit`       | Unit tests only                 |
| `make test-integration`| Integration tests only          |
| `make test-data`       | Data validation tests           |
| `make test-e2e`        | End-to-end tests                |

### Code Quality

| Command        | Description                              |
|----------------|------------------------------------------|
| `make lint`    | Run ruff (backend) + eslint (frontend)   |
| `make format`  | Auto-format with ruff + npm              |

### Shell Access

| Command              | Description                         |
|----------------------|-------------------------------------|
| `make shell-backend` | Bash shell in backend container     |
| `make shell-frontend`| Shell in frontend container         |
| `make shell-db`      | psql session in postgres container  |

## How It Works

### Backend (Hot Reload)

The dev compose mounts `./backend` as a volume into the container and overrides the production `CMD` with:

```
sh -c "uv sync --frozen --extra dev && uv run uvicorn app.main:app --host 0.0.0.0 --port 4000 --reload"
```

- `uv sync --frozen --extra dev` installs all dependencies including dev deps (pytest, ruff, mypy) into the container's virtualenv.
- An anonymous volume at `/app/.venv` isolates the container's virtualenv from the host's, preventing conflicts.
- `--reload` watches for file changes and restarts automatically.

### Frontend (Vite Dev Server)

The frontend container mounts `./frontend` and runs Vite's dev server with HMR:

```
npm run dev -- --host 0.0.0.0 --port 4001
```

A separate anonymous volume (`/app/node_modules`) prevents the host's `node_modules` from interfering with the container's architecture-specific modules.

### Named Volumes

Docker Compose creates persistent named volumes for stateful data:

| Volume (compose name) | Full Docker name              | Contents                 |
|-----------------------|-------------------------------|--------------------------|
| `pgdata`              | `clearance-engine_pgdata`     | PostgreSQL data files    |
| `qdrant_data`         | `clearance-engine_qdrant_data`| Qdrant vector storage    |

These survive `make down` and `make restart`. To truly reset, you must remove them explicitly (see "Resetting the database" below).

### Database Initialization

On first start, PostgreSQL runs `backend/data/init.sql` from a mounted volume, which creates the `pg_trgm` and `btree_gin` extensions needed for fuzzy matching and composite GIN indexes.

## Common Workflows

### Adding a new Python dependency

```bash
# Add to pyproject.toml, then:
cd backend && uv add <package>
# Rebuild containers to lock:
cd .. && make up-build
```

### Creating a new database migration

```bash
# After modifying SQLAlchemy models:
make migrate-new msg="add widget table"
# Review the generated file in backend/alembic/versions/
# Apply:
make migrate
```

### Resetting the database

```bash
make down
docker volume rm clearance-engine_pgdata
make up
make migrate
make seed
```

To also reset vector data: `docker volume rm clearance-engine_qdrant_data`

### Viewing API docs

Set `DEBUG=true` in `.env`, restart (`make restart`), then visit http://localhost:4000/docs.

## Troubleshooting

**Backend fails to start with "uv sync" errors:**
The volume mount may have stale lockfile state. Run `make up-build` to rebuild from scratch.

**Port conflicts:**
The dev stack uses non-standard host ports (5440, 6381, 6336) to avoid clashing with local services. If you still have conflicts, override ports in `.env`:

```bash
NATS_PORT=4223
NATS_MONITOR_PORT=8223
POSTGRES_PORT=5441
```

The `docker-compose.yml` reads these with fallback defaults (e.g., `${NATS_PORT:-4222}:4222`).

**Database connection refused from host tools (alembic, pytest):**
Host-side tools use the `.env` file URLs which point to `localhost:5440`. Make sure the postgres container is running (`make up`).

**Frontend can't reach backend:**
In dev, the frontend's `VITE_API_URL` is set to `http://localhost:4000/api`. Ensure the backend container is healthy: `curl http://localhost:4000/health`.
