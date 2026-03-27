# Configuration and Environment Reference

Complete reference for all configuration settings, environment variables, and deployment
tunables in the Clearance Intelligence Engine.

**Source**: `backend/app/config.py`
**Framework**: Pydantic Settings (`pydantic-settings`)
**Singleton**: `get_settings()` -- LRU-cached, returns the same instance for the process

---

## Table of Contents

1. [Settings Class Reference](#settings-class-reference)
2. [.env File Format](#env-file-format)
3. [Docker Compose Environment](#docker-compose-environment)
4. [Kubernetes Configuration](#kubernetes-configuration)
5. [LLM Provider Configuration](#llm-provider-configuration)
6. [Simulation Tunables](#simulation-tunables)
7. [Adding New Configuration Fields](#adding-new-configuration-fields)

---

## Settings Class Reference

The `Settings` class inherits from `pydantic_settings.BaseSettings` and loads values from:
1. Environment variables (highest priority)
2. `.env` file in the project root
3. Default values defined on the class (lowest priority)

Configuration is **case-insensitive** (`case_sensitive=False`) and ignores unknown
environment variables (`extra="ignore"`).

### Database

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `DATABASE_URL` | `str` | `postgresql+asyncpg://clearance:clearance_dev@localhost:5440/clearance_engine` | Async connection URL for SQLAlchemy (uses `asyncpg` driver) |
| `DATABASE_SYNC_URL` | `str` | `postgresql://clearance:clearance_dev@localhost:5440/clearance_engine` | Sync connection URL for Alembic migrations and CLI scripts |

**URL format**: `postgresql+asyncpg://{user}:{password}@{host}:{port}/{database}`

The async URL must use the `+asyncpg` dialect. The sync URL uses the default `psycopg2`
dialect (no explicit driver suffix needed).

**Connection pool** (configured in `DatabaseService`, not in Settings):
- `pool_size`: 10
- `max_overflow`: 20
- `pool_pre_ping`: True (detects stale connections)

### Redis

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `REDIS_URL` | `str` | `redis://localhost:6381/0` | Redis connection URL for the cache service |

**URL format**: `redis://{host}:{port}/{db_number}`

The default port `6381` is the host-mapped port from Docker Compose (container port 6379).

### Qdrant

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `QDRANT_URL` | `str` | `http://localhost:6336` | Qdrant REST API URL for the embedding service |

The default port `6336` is the host-mapped port from Docker Compose (container REST
port 6334). gRPC is available on port 6337 (container port 6335) but is not currently
used by the application.

### API Keys

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `ANTHROPIC_API_KEY` | `str` | `""` | Anthropic API key. Empty string disables the Anthropic provider. |
| `OPENAI_API_KEY` | `str` | `""` | OpenAI API key. Empty string disables the OpenAI provider. Required for embeddings even when Anthropic is the primary LLM. |

**Important**: The `OPENAI_API_KEY` is needed for both LLM fallback and embedding
generation (text-embedding-3-small). Even if Anthropic is the primary LLM provider,
OpenAI is still required for the EmbeddingService.

### Server

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `BACKEND_HOST` | `str` | `"0.0.0.0"` | Host to bind the uvicorn server |
| `BACKEND_PORT` | `int` | `4000` | Port for the backend API server |

### LLM Configuration

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `LLM_PRIMARY_PROVIDER` | `str` | `"anthropic"` | Primary LLM provider: `"anthropic"` or `"openai"` |
| `LLM_PRIMARY_MODEL` | `str` | `"claude-sonnet-4-20250514"` | Model identifier for the primary provider |
| `LLM_FALLBACK_PROVIDER` | `str` | `"openai"` | Fallback LLM provider |
| `LLM_FALLBACK_MODEL` | `str` | `"gpt-4o"` | Model identifier for the fallback provider |
| `LLM_TEMPERATURE` | `float` | `0.1` | Default sampling temperature (low for deterministic classification) |
| `LLM_MAX_TOKENS` | `int` | `4096` | Default maximum output tokens |

### Feature Flags

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `DEBUG` | `bool` | `False` | Enable debug mode (additional logging, dev middleware) |
| `LOG_LEVEL` | `str` | `"INFO"` | Logging level: DEBUG, INFO, WARNING, ERROR, CRITICAL |

### Simulation

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `SIM_AUTO_START` | `bool` | `False` | Automatically start the simulation engine on server boot |
| `SIM_TIME_RATIO` | `float` | `10.0` | Simulation time multiplier (10.0 = 10x faster than real time) |
| `SIM_RANDOM_SEED` | `int \| None` | `None` | Random seed for reproducible simulation runs. `None` for non-deterministic. |

---

## .env File Format

Create a `.env` file in the project root (`clearance-engine/.env`) or the `backend/`
directory. The Settings class looks for `.env` in both locations.

### Minimal .env for Local Development

```bash
# API Keys (required for LLM and embedding functionality)
ANTHROPIC_API_KEY=sk-ant-api03-your-key-here
OPENAI_API_KEY=sk-your-openai-key-here

# Everything else uses defaults for local Docker Compose setup
```

### Full .env with All Options

```bash
# ── Database ──────────────────────────────────────────────
DATABASE_URL=postgresql+asyncpg://clearance:clearance_dev@localhost:5440/clearance_engine
DATABASE_SYNC_URL=postgresql://clearance:clearance_dev@localhost:5440/clearance_engine

# ── Redis ─────────────────────────────────────────────────
REDIS_URL=redis://localhost:6381/0

# ── Qdrant ────────────────────────────────────────────────
QDRANT_URL=http://localhost:6336

# ── API Keys ──────────────────────────────────────────────
ANTHROPIC_API_KEY=sk-ant-api03-your-key-here
OPENAI_API_KEY=sk-your-openai-key-here

# ── Server ────────────────────────────────────────────────
BACKEND_HOST=0.0.0.0
BACKEND_PORT=4000

# ── LLM Configuration ────────────────────────────────────
LLM_PRIMARY_PROVIDER=anthropic
LLM_PRIMARY_MODEL=claude-sonnet-4-20250514
LLM_FALLBACK_PROVIDER=openai
LLM_FALLBACK_MODEL=gpt-4o
LLM_TEMPERATURE=0.1
LLM_MAX_TOKENS=4096

# ── Debug ─────────────────────────────────────────────────
DEBUG=false
LOG_LEVEL=INFO

# ── Simulation ────────────────────────────────────────────
SIM_AUTO_START=false
SIM_TIME_RATIO=10.0
# SIM_RANDOM_SEED=42  # Uncomment for reproducible runs
```

### .env Precedence

Environment variables always take precedence over `.env` file values. This allows
Docker Compose and Kubernetes to override settings without modifying the file.

Precedence order (highest to lowest):
1. OS environment variables
2. `.env` file in `backend/` directory
3. `../.env` file in project root
4. Default values in the `Settings` class

---

## Docker Compose Environment

The `docker-compose.yml` defines infrastructure services and overrides connection URLs
to use container networking (hostnames instead of localhost).

### Infrastructure Services

| Service | Image | Container Port | Host Port | Purpose |
|---------|-------|---------------|-----------|---------|
| `postgres` | `postgres:16-alpine` | 5432 | 5440 | PostgreSQL database |
| `redis` | `redis:7-alpine` | 6379 | 6381 | Redis cache |
| `qdrant` | `qdrant/qdrant:latest` | 6334 (REST), 6335 (gRPC) | 6336, 6337 | Vector search |

### PostgreSQL Configuration

```yaml
environment:
  POSTGRES_DB: clearance_engine
  POSTGRES_USER: clearance
  POSTGRES_PASSWORD: clearance_dev
```

Data is persisted via the `pgdata` Docker volume. An `init.sql` script at
`backend/data/init.sql` runs on first initialization.

### Backend Service Overrides

When running inside Docker Compose, the `backend` service receives overridden connection
URLs that use container hostnames:

```yaml
environment:
  DATABASE_URL: postgresql+asyncpg://clearance:clearance_dev@postgres:5432/clearance_engine
  DATABASE_SYNC_URL: postgresql://clearance:clearance_dev@postgres:5432/clearance_engine
  REDIS_URL: redis://redis:6379/0
  QDRANT_URL: http://qdrant:6333
```

Note the internal container ports are used (5432, 6379, 6333) rather than the
host-mapped ports (5440, 6381, 6336).

The backend also reads from the `.env` file via `env_file: .env` for API keys and
other settings not overridden in the `environment` block.

### Frontend Service

```yaml
environment:
  VITE_API_URL: http://localhost:4000
```

### Running with Docker Compose

```bash
# Start all services
docker compose up -d

# Start with rebuild
docker compose up -d --build

# View logs
docker compose logs -f backend

# Stop all services
docker compose down

# Stop and remove volumes (destroys data)
docker compose down -v
```

---

## Kubernetes Configuration

For production Kubernetes deployments, map Settings fields to ConfigMaps and Secrets.

### ConfigMap (non-sensitive settings)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: clearance-engine-config
  namespace: clearance
data:
  DATABASE_URL: "postgresql+asyncpg://clearance:$(DB_PASSWORD)@postgres-service:5432/clearance_engine"
  DATABASE_SYNC_URL: "postgresql://clearance:$(DB_PASSWORD)@postgres-service:5432/clearance_engine"
  REDIS_URL: "redis://redis-service:6379/0"
  QDRANT_URL: "http://qdrant-service:6334"
  BACKEND_HOST: "0.0.0.0"
  BACKEND_PORT: "4000"
  LLM_PRIMARY_PROVIDER: "anthropic"
  LLM_PRIMARY_MODEL: "claude-sonnet-4-20250514"
  LLM_FALLBACK_PROVIDER: "openai"
  LLM_FALLBACK_MODEL: "gpt-4o"
  LLM_TEMPERATURE: "0.1"
  LLM_MAX_TOKENS: "4096"
  DEBUG: "false"
  LOG_LEVEL: "INFO"
  SIM_AUTO_START: "false"
  SIM_TIME_RATIO: "10.0"
```

### Secret (sensitive settings)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: clearance-engine-secrets
  namespace: clearance
type: Opaque
stringData:
  ANTHROPIC_API_KEY: "sk-ant-api03-..."
  OPENAI_API_KEY: "sk-..."
```

### Deployment Environment Reference

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clearance-backend
spec:
  template:
    spec:
      containers:
        - name: backend
          envFrom:
            - configMapRef:
                name: clearance-engine-config
            - secretRef:
                name: clearance-engine-secrets
          ports:
            - containerPort: 4000
          livenessProbe:
            httpGet:
              path: /health
              port: 4000
            initialDelaySeconds: 10
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /health
              port: 4000
            initialDelaySeconds: 5
            periodSeconds: 10
```

### Production Recommendations

| Setting | Production Value | Reason |
|---------|-----------------|--------|
| `DEBUG` | `false` | Disable debug middleware and verbose logging |
| `LOG_LEVEL` | `"WARNING"` or `"INFO"` | Reduce log volume |
| `LLM_TEMPERATURE` | `0.0` to `0.1` | Maximize determinism for classification |
| `SIM_AUTO_START` | `false` | Simulation is for demo/dev only |
| `DATABASE_URL` | Use connection pooler (PgBouncer) | Scale connections across replicas |

---

## LLM Provider Configuration

### Provider Priority

The `LLM_PRIMARY_PROVIDER` setting determines which provider is tried first.
The other provider becomes the automatic fallback.

```
LLM_PRIMARY_PROVIDER=anthropic  -->  Anthropic first, OpenAI fallback
LLM_PRIMARY_PROVIDER=openai     -->  OpenAI first, Anthropic fallback
```

### Single-Provider Mode

To use only one provider, set the other's API key to empty:

```bash
# Anthropic only (no fallback)
ANTHROPIC_API_KEY=sk-ant-api03-...
OPENAI_API_KEY=
LLM_PRIMARY_PROVIDER=anthropic
```

Note that `OPENAI_API_KEY` is still needed for the EmbeddingService even in
Anthropic-only LLM mode. If you truly cannot use OpenAI, the embedding service
will not function.

### Model Selection

Common model choices:

| Provider | Model | Use Case |
|----------|-------|----------|
| Anthropic | `claude-sonnet-4-20250514` | Default -- good balance of speed and quality |
| Anthropic | `claude-opus-4-5-20251101` | Maximum quality for complex classification |
| OpenAI | `gpt-4o` | Default fallback -- fast with good quality |
| OpenAI | `gpt-4o-mini` | Budget fallback -- lower cost |

### Temperature Guidance

| Temperature | Use Case |
|-------------|----------|
| `0.0` | Deterministic classification (highest consistency) |
| `0.1` | Default -- slight randomness helps with edge cases |
| `0.3` | More creative analysis (regulatory summaries) |
| `0.7+` | Not recommended for compliance use cases |

---

## Simulation Tunables

The simulation engine has its own detailed configuration in
`backend/app/simulation/config.py`. The top-level settings (`SIM_AUTO_START`,
`SIM_TIME_RATIO`, `SIM_RANDOM_SEED`) are passed through from the main Settings class.

### Top-Level Simulation Settings

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `SIM_AUTO_START` | `bool` | `False` | Start the simulation automatically when the server boots |
| `SIM_TIME_RATIO` | `float` | `10.0` | Time multiplier. `10.0` means 1 real second = 10 simulated seconds. Higher values run the simulation faster. |
| `SIM_RANDOM_SEED` | `int \| None` | `None` | Random seed for reproducibility. Set to an integer for deterministic simulation runs (useful for testing). |

### Actor-Level Configuration

The simulation uses an actor-based architecture. Each actor has configurable parameters
defined in `SimulationConfig`:

#### ShipperActor

| Parameter | Default | Description |
|-----------|---------|-------------|
| `tick_sim_minutes` | `60.0` | Simulated minutes per tick |
| `base_rate_per_day` | `30.0` | Base shipment creation rate per simulated day |
| `business_hours_multiplier` | `2.5` | Rate multiplier during business hours |
| `overnight_multiplier` | `0.3` | Rate multiplier during off-hours |
| `misclassification_rate` | `0.07` | Probability of generating a misclassified HS code |
| `value_error_rate` | `0.03` | Probability of a declared value error |
| `value_error_range` | `(-0.15, 0.25)` | Range of value error percentage |

#### CarrierActor

| Parameter | Default | Description |
|-----------|---------|-------------|
| `tick_sim_minutes` | `120.0` | Simulated minutes per tick |
| `delay_chance` | `0.08` | Probability of a transit delay |
| `delay_hours_range` | `(6.0, 48.0)` | Range of delay duration in simulated hours |
| `inland_days_range` | `(1.0, 3.0)` | Range for inland transit in simulated days |

#### CustomsActor

| Parameter | Default | Description |
|-----------|---------|-------------|
| `tick_sim_minutes` | `30.0` | Simulated minutes per tick |
| `stp_rate` | `0.87` | Straight-through processing rate (87% of entries) |
| `stp_minutes_range` | `(5.0, 30.0)` | Time range for STP processing |
| `manual_review_hours_range` | `(2.0, 48.0)` | Time range for manual review |
| `uflpa_detention_rate` | `0.02` | UFLPA detention probability |
| `physical_inspection_rate` | `0.04` | Physical inspection probability |
| `hs_reclassification_rate` | `0.05` | HS reclassification probability |
| `hold_rate` | `0.038` | General hold probability |
| `backlog_threshold` | `20` | Backlog count that triggers slowdown |
| `backlog_slowdown` | `1.5` | Processing time multiplier when backlogged |
| `corridor_scrutiny` | `{"CN": 1.3, "VN": 1.1, "IN": 1.1}` | Per-corridor scrutiny multiplier |
| `inspection_pass_rate` | `0.80` | Probability of passing physical inspection |

#### PGAActor

| Parameter | Default | Description |
|-----------|---------|-------------|
| `tick_sim_minutes` | `60.0` | Simulated minutes per tick |
| `fda_review_days_range` | `(1.0, 5.0)` | FDA review duration in simulated days |
| `fda_approval_rate` | `0.92` | FDA approval probability |
| `epa_review_days_range` | `(1.0, 3.0)` | EPA review duration |
| `epa_approval_rate` | `0.95` | EPA approval probability |
| `cpsc_review_days_range` | `(1.0, 2.0)` | CPSC review duration |
| `cpsc_approval_rate` | `0.97` | CPSC approval probability |

#### PreClearanceAdapter

| Parameter | Default | Description |
|-----------|---------|-------------|
| `tick_sim_minutes` | `15.0` | Simulated minutes per tick |

#### ComplianceEngineActor

| Parameter | Default | Description |
|-----------|---------|-------------|
| `tick_sim_minutes` | `15.0` | Simulated minutes per tick |
| `run_real_engines` | `False` | Whether to run real compliance engines (vs. simulated results) |

#### ResolutionActor

| Parameter | Default | Description |
|-----------|---------|-------------|
| `tick_sim_minutes` | `240.0` | Simulated minutes per tick |
| `review_days_range` | `(1.0, 5.0)` | Review duration in simulated days |
| `resolve_rate` | `0.75` | Probability of resolving the issue |
| `escalate_rate` | `0.10` | Probability of escalation |
| `pending_rate` | `0.15` | Probability of remaining pending |

#### Global Simulation

| Parameter | Default | Description |
|-----------|---------|-------------|
| `dashboard_interval_seconds` | `3.0` | Real-time interval for dashboard data refresh |

---

## Adding New Configuration Fields

### Step 1: Add the field to Settings

Edit `backend/app/config.py`:

```python
class Settings(BaseSettings):
    # ... existing fields ...

    # New field with type annotation and default
    NEW_FEATURE_ENABLED: bool = False
    NEW_FEATURE_THRESHOLD: float = 0.5
```

### Step 2: Set the environment variable

Add to `.env`:

```bash
NEW_FEATURE_ENABLED=true
NEW_FEATURE_THRESHOLD=0.8
```

Or set as an OS environment variable:

```bash
export NEW_FEATURE_ENABLED=true
```

### Step 3: Use in code

```python
from app.config import get_settings

settings = get_settings()
if settings.NEW_FEATURE_ENABLED:
    # feature logic
    threshold = settings.NEW_FEATURE_THRESHOLD
```

### Step 4: Update deployment configuration

Add the new variables to Docker Compose `environment` or Kubernetes ConfigMap/Secret
as appropriate.

### Guidelines

- **Type annotations are required**: Pydantic validates and casts values at startup.
  A type mismatch raises `ValidationError` immediately.
- **Always provide a default**: This ensures the application starts even if the
  variable is not set. Use a sensible development default.
- **Use descriptive names**: Follow the existing `CATEGORY_SETTING` naming pattern
  (e.g. `LLM_PRIMARY_MODEL`, `SIM_TIME_RATIO`).
- **Secrets should be empty-string defaults**: API keys default to `""` so the
  application can start without them (features degrade gracefully).
- **Boolean values**: Pydantic accepts `true`/`false`, `1`/`0`, `yes`/`no`, `on`/`off`.
- **The singleton is cached**: `get_settings()` uses `@lru_cache(maxsize=1)`, so
  settings are read once at startup. Changing `.env` requires a restart.
- **Case insensitive**: `DATABASE_URL` and `database_url` both work in the `.env` file,
  though uppercase is the convention.
