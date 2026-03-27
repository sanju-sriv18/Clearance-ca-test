# Operations & Maintenance Guide

> **Author:** Simon Kissler (simon.kissler@accenture.com) | **Co-Author:** Claude Code (Anthropic)

This guide covers day-to-day operational tasks for keeping the Clearance Intelligence Engine healthy and secure in both local dev and production environments.

## Table of Contents

- [Service Management](#service-management)
- [Database Operations](#database-operations)
- [Monitoring & Health Checks](#monitoring--health-checks)
- [Log Management](#log-management)
- [Backup & Recovery](#backup--recovery)
- [Security Maintenance](#security-maintenance)
- [Updating Dependencies](#updating-dependencies)
- [Scaling & Performance](#scaling--performance)
- [Regulatory Monitor](#regulatory-monitor)
- [Analysis Watchdog](#analysis-watchdog)
- [Troubleshooting Runbook](#troubleshooting-runbook)

---

## Service Management

### Viewing Container Status

```bash
# Dev
cd clearance-engine
docker compose ps

# Production (on the EC2 instance)
cd ~/clearance_vibe/clearance-engine
docker compose -f docker-compose.prod.yml ps
```

### Restarting Services

```bash
# Restart all services
docker compose -f docker-compose.prod.yml restart

# Restart a single service
docker compose -f docker-compose.prod.yml restart backend

# Full stop and start (re-reads compose file)
docker compose -f docker-compose.prod.yml down
docker compose -f docker-compose.prod.yml up -d
```

### Rebuilding After Code Changes

Use the deploy script for production:

```bash
bash deploy.sh
```

Or rebuild individual services:

```bash
# Rebuild only the backend
docker compose -f docker-compose.prod.yml up --build -d backend

# Rebuild only the frontend
docker compose -f docker-compose.prod.yml up --build -d frontend
```

---

## Database Operations

### Running Migrations

```bash
# Production
docker compose -f docker-compose.prod.yml exec backend uv run alembic upgrade head

# Dev
make migrate
```

### Rolling Back a Migration

```bash
# Production — roll back one step
docker compose -f docker-compose.prod.yml exec backend uv run alembic downgrade -1

# Dev
make migrate-down
```

### Creating a New Migration

After modifying SQLAlchemy models in `backend/app/knowledge/models/`:

```bash
# Dev (from clearance-engine/)
make migrate-new msg="describe the change"

# Review the generated file in backend/alembic/versions/
# Then apply:
make migrate
```

### Connecting to the Database

```bash
# Dev
make shell-db
# or: docker compose exec postgres psql -U clearance -d clearance_engine

# Production
docker compose -f docker-compose.prod.yml exec postgres psql -U clearance -d clearance_engine
```

Useful psql commands:

```sql
-- List all tables
\dt

-- Check table sizes
SELECT relname, pg_size_pretty(pg_total_relation_size(relid))
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- Check active connections
SELECT count(*) FROM pg_stat_activity WHERE datname = 'clearance_engine';

-- Check if extensions are loaded
SELECT * FROM pg_extension;
```

### Re-Seeding Reference Data

To refresh reference data (tariffs, restricted parties, etc.) without touching operational data:

```bash
# Production — exec into backend container
docker compose -f docker-compose.prod.yml exec backend uv run python -m data.ingestion.seed_all

# Or seed specific datasets:
docker compose -f docker-compose.prod.yml exec backend uv run python -m data.ingestion.seed_htsus
docker compose -f docker-compose.prod.yml exec backend uv run python -m data.ingestion.seed_restricted_parties
```

### Required PostgreSQL Extensions

The database requires two extensions, created automatically by `backend/data/init.sql` on first container start:

- **pg_trgm** — trigram similarity for fuzzy entity name matching (used by restricted party screening)
- **btree_gin** — allows B-tree operators inside GIN indexes (used for composite JSONB indexes)

If you recreate the database manually (e.g., `CREATE DATABASE`), you must install these extensions before running migrations:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS btree_gin;
```

### Resetting the Database (Dev Only)

```bash
make down
docker volume rm clearance-engine_pgdata
make up
make migrate
make seed
```

---

## Monitoring & Health Checks

### Health Endpoint

The backend exposes a `/health` endpoint that checks all dependencies:

```bash
# From the instance
curl -sf http://localhost:4000/health | python3 -m json.tool

# From outside (through Caddy, requires auth)
curl -u operator:password https://clearance.yourdomain.com/api/health
```

Response (flat dictionary, one key per backing service):

```json
{
  "api": "ok",
  "redis": "ok",
  "database": "ok",
  "qdrant": "ok"
}
```

### Container Health Checks

Docker Compose includes built-in health checks for all infrastructure services:

| Service    | Check                                    | Interval |
|------------|------------------------------------------|----------|
| PostgreSQL | `pg_isready -U clearance -d clearance_engine` | 5s       |
| Redis      | `redis-cli ping`                         | 5s       |
| Qdrant     | TCP connect on port 6333                 | 5s       |

Check Docker's view of health:

```bash
docker compose -f docker-compose.prod.yml ps
# Look for "(healthy)" in the STATUS column
```

### Resource Usage

```bash
# Live container resource usage
docker stats --no-stream

# Disk usage by Docker
docker system df

# Instance disk usage
df -h
```

---

## Log Management

### Viewing Logs

```bash
# All services
docker compose -f docker-compose.prod.yml logs -f

# Single service, last 100 lines
docker compose -f docker-compose.prod.yml logs -f --tail 100 backend

# Caddy access logs
docker compose -f docker-compose.prod.yml logs -f caddy
```

### Bootstrap / Init Logs

```bash
# User-data script output (first boot only)
cat /var/log/clearance-init.log

# Cloud-init logs
cat /var/log/cloud-init-output.log
```

### Log Rotation

Docker uses the `json-file` logging driver by default. To prevent disk exhaustion, configure log rotation on the instance by creating `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  }
}
```

Then restart Docker:

```bash
sudo systemctl restart docker
```

---

## Backup & Recovery

### Database Backup

```bash
# Create a compressed backup
docker compose -f docker-compose.prod.yml exec postgres \
  pg_dump -U clearance clearance_engine | gzip > backup_$(date +%Y%m%d_%H%M%S).sql.gz

# Copy backup to local machine
scp ec2-user@<elastic-ip>:~/clearance_vibe/clearance-engine/backup_*.sql.gz ./
```

### Database Restore

```bash
# Stop the backend to prevent writes
docker compose -f docker-compose.prod.yml stop backend

# Restore from backup
gunzip -c backup_20260205_120000.sql.gz | \
  docker compose -f docker-compose.prod.yml exec -T postgres \
  psql -U clearance clearance_engine

# Restart the backend
docker compose -f docker-compose.prod.yml start backend
```

### Qdrant Data

Qdrant stores vector data in a Docker volume (`qdrant_data`). To back it up:

```bash
# Find the volume mount path
docker volume inspect clearance-engine_qdrant_data --format '{{ .Mountpoint }}'

# Tar the volume
sudo tar czf qdrant_backup_$(date +%Y%m%d).tar.gz \
  $(docker volume inspect clearance-engine_qdrant_data --format '{{ .Mountpoint }}')
```

### Volume Listing

```bash
# See all project volumes
docker volume ls | grep clearance
```

Volumes in production:

| Volume                         | Contents                          |
|--------------------------------|-----------------------------------|
| `clearance-engine_pgdata`      | PostgreSQL data files             |
| `clearance-engine_qdrant_data` | Qdrant vector storage             |
| `clearance-engine_caddy_data`  | TLS certificates (Let's Encrypt)  |
| `clearance-engine_caddy_config`| Caddy runtime configuration       |

---

## Security Maintenance

### Rotating the Basic Auth Password

```bash
# SSH into the instance
ssh ec2-user@<elastic-ip>
cd ~/clearance_vibe/clearance-engine

# Generate new hash
NEW_HASH=$(docker run --rm caddy:alpine caddy hash-password --plaintext 'new-password-here')

# Escape for Docker Compose
NEW_HASH_ESCAPED=$(echo "$NEW_HASH" | sed 's/\$/\$\$/g')

# Update .env.prod
sed -i "s|AUTH_PASS_HASH=.*|AUTH_PASS_HASH=$NEW_HASH_ESCAPED|" .env.prod

# Restart Caddy to pick up the change
docker compose -f docker-compose.prod.yml restart caddy
```

### Rotating the Database Password

```bash
# 1. Generate new password
NEW_PG_PASS=$(openssl rand -base64 24)

# 2. Change the password inside PostgreSQL
docker compose -f docker-compose.prod.yml exec postgres \
  psql -U clearance -d clearance_engine -c "ALTER USER clearance WITH PASSWORD '$NEW_PG_PASS';"

# 3. Update .env.prod
sed -i "s|POSTGRES_PASSWORD=.*|POSTGRES_PASSWORD=$NEW_PG_PASS|" .env.prod

# 4. Update the symlinked .env (used for compose variable substitution)
# (Already handled since .env is a symlink to .env.prod)

# 5. Restart backend to pick up new connection string
docker compose -f docker-compose.prod.yml restart backend
```

### Rotating API Keys

```bash
# Update the key in .env.prod
sed -i "s|ANTHROPIC_API_KEY=.*|ANTHROPIC_API_KEY=sk-ant-new-key-here|" .env.prod

# Restart the backend
docker compose -f docker-compose.prod.yml restart backend
```

### Updating Container Base Images

Periodically pull updated base images to get security patches:

```bash
docker compose -f docker-compose.prod.yml pull postgres redis qdrant/qdrant caddy
docker compose -f docker-compose.prod.yml up -d
```

### Restricting SSH Access

Update `ssh_cidr_blocks` in your `terraform.tfvars` to your IP only:

```hcl
ssh_cidr_blocks = ["203.0.113.42/32"]
```

Then apply:

```bash
cd infra && terraform apply
```

### Checking for Exposed Ports

Verify only ports 22, 80, and 443 are reachable from outside:

```bash
# From your local machine
nmap -Pn <elastic-ip>
```

### TLS Certificate Management

Caddy handles TLS certificates automatically via Let's Encrypt. No manual intervention is needed. Certificates are stored in the `caddy_data` volume and are renewed automatically before expiration.

To check certificate status:

```bash
docker compose -f docker-compose.prod.yml exec caddy caddy list-certificates
```

---

## Updating Dependencies

### Backend (Python)

```bash
# SSH into the instance
cd ~/clearance_vibe/clearance-engine/backend

# Update a specific package
uv add <package>@latest

# Update all packages
uv lock --upgrade

# Rebuild
cd .. && docker compose -f docker-compose.prod.yml up --build -d backend
```

In dev, update locally and rebuild:

```bash
cd clearance-engine/backend
uv add <package>@latest
cd .. && make up-build
```

### Frontend (Node.js)

```bash
cd clearance-engine/frontend
npm update
npm audit fix

# Rebuild
cd .. && docker compose -f docker-compose.prod.yml up --build -d frontend
```

### System Packages (EC2 Instance)

```bash
sudo dnf update -y
sudo systemctl restart docker
```

### Docker Engine

```bash
sudo dnf update docker -y
sudo systemctl restart docker
docker compose -f docker-compose.prod.yml up -d
```

---

## Scaling & Performance

### Backend Workers

The production backend runs 2 uvicorn workers. To increase:

1. Edit `backend/Dockerfile`, change the `--workers` argument in `CMD`
2. Redeploy with `bash deploy.sh`

Or override at runtime:

```bash
docker compose -f docker-compose.prod.yml exec backend \
  uv run uvicorn app.main:app --host 0.0.0.0 --port 4000 --workers 4
```

### Database Connection Pool

The backend uses SQLAlchemy async with:

- `pool_size=20` — steady-state connections
- `max_overflow=10` — burst capacity (up to 30 total)

To adjust, edit `backend/app/main.py` where `create_async_engine` is called.

### Redis

Redis runs with default settings. To add memory limits:

```bash
docker compose -f docker-compose.prod.yml exec redis redis-cli CONFIG SET maxmemory 512mb
docker compose -f docker-compose.prod.yml exec redis redis-cli CONFIG SET maxmemory-policy allkeys-lru
```

### Rate Limits

Rate limit decorators exist on all endpoints via `slowapi` middleware, but limits are currently set to **50,000 requests/minute per IP** (effectively unlimited). This infrastructure is preserved for future production hardening.

To enable meaningful rate limits, edit the `@limiter.limit()` decorators in `clearance_platform/domains/*/routes.py` and `clearance_platform/gateway/*_routes.py`. **Do not lower limits without explicit approval** — the current permissive setting is intentional for development and demo environments.

### Instance Sizing

The default `t4g.xlarge` (4 vCPU, 16 GB RAM) is suitable for moderate workloads. For higher throughput:

```hcl
# In terraform.tfvars
instance_type = "t4g.2xlarge"   # 8 vCPU, 32 GB RAM
volume_size   = 50               # More disk for data
```

Then `terraform apply`. Note: changing instance type requires a stop/start cycle.

---

## Regulatory Monitor

The regulatory monitor is a background service that automatically fetches government feeds and trade news, extracts structured regulatory signals using the LLM, and stores them in the `regulatory_signals` database table. It runs on a configurable interval via APScheduler.

### Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `REGULATORY_MONITOR_ENABLED` | `false` (`true` in dev compose) | Enable/disable the monitor |
| `REGULATORY_MONITOR_INTERVAL_HOURS` | `2` | Feed check interval |
| `REGULATORY_MONITOR_EXTRACTION_MODEL` | *(empty)* | Override LLM model for extraction (empty = use primary) |

### Checking Monitor Status

```bash
# Dev
curl http://localhost:4000/api/regulatory/monitor/status

# Production
curl -u operator:password https://clearance.yourdomain.com/api/regulatory/monitor/status
```

Response:

```json
{
  "enabled": true,
  "last_run": "2026-02-05T14:30:00Z",
  "last_result": {"articles": 25, "extracted": 12, "stored": 3},
  "next_run": "2026-02-05T16:30:00Z",
  "total_signals": 47,
  "error_count": 0
}
```

### Triggering a Manual Refresh

```bash
# Dev
curl -X POST http://localhost:4000/api/regulatory/refresh

# Production
curl -u operator:password -X POST https://clearance.yourdomain.com/api/regulatory/refresh
```

The refresh runs asynchronously and returns immediately with `{"status": "refresh_started"}`. Check the monitor status endpoint or backend logs for results.

### Viewing Signals

```bash
# List all signals
curl http://localhost:4000/api/regulatory

# Filter by impact level
curl "http://localhost:4000/api/regulatory?impact_level=CRITICAL"

# Filter by status
curl "http://localhost:4000/api/regulatory?status=CONFIRMED"

# Filter by HS code prefix
curl "http://localhost:4000/api/regulatory?hs_prefix=8471"
```

### Feed Sources

The monitor fetches from four sources per cycle:

| Source | Type | Auth | Content |
|--------|------|------|---------|
| Federal Register API | JSON | None | CBP, ITC, Commerce rules/notices |
| CBP CSMS | RSS/XML | None | Cargo Systems Messaging Service |
| USTR Press Releases | HTML | None | Trade policy announcements |
| Google News RSS | RSS/XML | None | Global trade/tariff/sanctions news |

Each feed fetcher has independent error handling. One feed failure does not block the others.

### Enabling / Disabling

To disable the monitor without restarting:

```bash
# Set env var and restart only the backend
# Dev:
# Edit docker-compose.yml, set REGULATORY_MONITOR_ENABLED: "false"
docker compose restart backend

# Production:
sed -i 's/REGULATORY_MONITOR_ENABLED=true/REGULATORY_MONITOR_ENABLED=false/' .env.prod
docker compose -f docker-compose.prod.yml restart backend
```

### Multi-Worker Safety

The monitor uses a Redis distributed lock (`regulatory_monitor:lock` with 30-minute TTL) to ensure only one backend worker runs the feed check per cycle. This is safe with multiple uvicorn workers and prevents duplicate signal extraction.

### Monitoring Logs

The monitor logs structured events via `structlog`. Key log messages:

```bash
# Dev
docker compose logs -f backend | grep -i "regulatory\|monitor\|feed"

# Production
docker compose -f docker-compose.prod.yml logs -f backend | grep -i "regulatory\|monitor\|feed"
```

Events to watch for:

| Event | Level | Meaning |
|-------|-------|---------|
| `regulatory_monitor_started` | INFO | Monitor cycle began |
| `regulatory_monitor_completed` | INFO | Cycle finished (includes article/signal counts) |
| `feed_fetch_failed` | WARNING | One feed source failed (others still processed) |
| `signal_extraction_failed` | WARNING | LLM returned invalid JSON for a batch |
| `regulatory_monitor_skipped` | INFO | Another worker already holds the lock |
| `regulatory_monitor_error` | ERROR | Unexpected error in the monitor cycle |

### LLM Cost Management

Each cycle processes ~20-30 articles in batches of 5 per LLM call. Estimated costs at 2-hour intervals:

| Model | Per Cycle | Per Day | Per Month |
|-------|-----------|---------|-----------|
| Claude Haiku | ~$0.01 | ~$0.12 | ~$3-5 |
| Claude Sonnet | ~$0.05 | ~$0.60 | ~$15-20 |

To reduce costs:
- Set `REGULATORY_MONITOR_EXTRACTION_MODEL` to a smaller model for extraction
- Increase `REGULATORY_MONITOR_INTERVAL_HOURS` to check less frequently
- Disable the monitor entirely when not needed

### Database Maintenance

Regulatory signals accumulate over time. To manage table size:

```bash
# Check signal count
docker compose -f docker-compose.prod.yml exec postgres \
  psql -U clearance -d clearance_engine -c "SELECT count(*) FROM regulatory_signals;"

# Check signals by status
docker compose -f docker-compose.prod.yml exec postgres \
  psql -U clearance -d clearance_engine -c \
  "SELECT status, count(*) FROM regulatory_signals GROUP BY status;"

# Archive old signals (older than 6 months)
docker compose -f docker-compose.prod.yml exec postgres \
  psql -U clearance -d clearance_engine -c \
  "DELETE FROM regulatory_signals WHERE created_at < NOW() - INTERVAL '6 months';"
```

### Deduplication

The monitor uses three tiers of deduplication to avoid storing duplicate signals:

1. **URL match** — Skips articles with `source_url` already in the database
2. **Hash match** — SHA-256 of `lower(title)|url` stored in the `source_hash` column (unique index)
3. **Semantic dedup** — For candidates with overlapping HS codes/countries vs. recent signals (30 days), asks the LLM to confirm if they describe the same policy change

---

## CSL Ingestion (Denied Party Screening)

The Consolidated Screening List (CSL) ingestion pipeline fetches restricted party data from the US Department of Commerce's trade.gov API and loads it into the `compliance.restricted_parties` database table. The DPS Screener (E3) uses this data for live denied entity matching against 14 US federal screening lists.

### Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `CSL_INGESTION_ENABLED` | `true` | Enable/disable CSL ingestion |
| `CSL_INGESTION_INTERVAL_HOURS` | `6.0` | Hours between fetch-and-refresh cycles |

### How It Works

1. **Fetch** — HTTP GET to `api.trade.gov/consolidated_screening_list/v2/search` (paginated, ~12,000 records)
2. **Parse** — JSON response normalized to `RestrictedParty` records with entity name, aliases, list source, country, addresses, programs, identification numbers
3. **Upsert** — `INSERT ... ON CONFLICT (source_list_id, list_source) DO UPDATE` into `compliance.restricted_parties`
4. **Stale cleanup** — Records from known CSL list sources not in the latest fetch are deleted
5. **Event** — `ScreeningListUpdatedEvent` emitted on NATS for downstream consumers

### Lists Ingested (14 US Federal)

| List Source | Agency | Description |
|-------------|--------|-------------|
| OFAC SDN | Treasury | Specially Designated Nationals and Blocked Persons |
| BIS Entity List | Commerce | End-use/user concern entities |
| BIS Denied Persons | Commerce | Denied export privileges |
| BIS Unverified | Commerce | Unable to verify end-use |
| BIS Military End-User | Commerce | Military end-user/use entities |
| AECA Debarred | State/DDTC | Arms export debarred parties |
| ISN Nonproliferation | State | WMD/missile proliferation sanctions |
| OFAC FSE | Treasury | Foreign Sanctions Evaders |
| OFAC Sectoral | Treasury | Sectoral sanctions (SSI) |
| OFAC CAPTA | Treasury | Palestinian terrorism designations |
| OFAC NS-MBS | Treasury | Non-SDN Menu-Based Sanctions |
| OFAC NS-CMIC | Treasury | Non-SDN Chinese Military-Industrial Complex |
| OFAC PLC | Treasury | Palestinian Legislative Council |
| UFLPA Entity List | DHS | Uyghur Forced Labor Prevention Act entities |

### Monitoring Logs

```bash
# Dev
docker compose logs -f backend | grep -i "csl"

# Production
docker compose -f docker-compose.prod.yml logs -f backend | grep -i "csl"
```

| Event | Level | Meaning |
|-------|-------|---------|
| `csl_ingestion_started` | INFO | Pipeline registered with scheduler |
| `csl_ingestion_complete` | INFO | Fetch + parse + upsert cycle finished |
| `csl_parsed` | INFO | Records parsed from API (includes count) |
| `csl_upsert_complete` | INFO | Records written to DB |
| `csl_ingestion_failed` | ERROR | Pipeline error (network, parse, or DB) |
| `csl_ingestion_disabled` | INFO | Pipeline not started (CSL_INGESTION_ENABLED=false) |

### Database Maintenance

```bash
# Check restricted party count
docker compose exec postgres \
  psql -U clearance -d clearance_engine -c \
  "SELECT list_source, count(*) FROM compliance.restricted_parties GROUP BY list_source ORDER BY count DESC;"

# Check latest ingestion time
docker compose exec postgres \
  psql -U clearance -d clearance_engine -c \
  "SELECT max(ingested_at) FROM compliance.restricted_parties;"
```

### DPS Screener Integration

The DPSScreener (E3B) lazily loads restricted party data from the database on its first `screen()` call after startup. If the database is empty (CSL not yet ingested), it falls back to in-memory seed data (~50 representative entities). Once CSL data is loaded, the screener operates against the full corpus.

---

## Analysis Watchdog

The analysis watchdog is a core platform service that automatically ensures every shipment has up-to-date analysis (E1 Classification, E2 Tariff, E3 Compliance). It detects stale analysis by comparing `shipments.updated_at` against `analysis->>'analyzed_at'` — any change to a shipment row (or related documents/entry_filings via PostgreSQL triggers) automatically triggers re-analysis.

### Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `ANALYSIS_WATCHDOG_ENABLED` | `true` | Enable/disable the watchdog |
| `ANALYSIS_WATCHDOG_INTERVAL_SECONDS` | `15` | Polling interval between cycles |
| `ANALYSIS_WATCHDOG_BATCH_SIZE` | `5` | Max shipments analyzed per cycle |

### Monitoring Logs

```bash
# Dev
docker compose logs -f backend | grep -i "analysis_watchdog"

# Production
docker compose -f docker-compose.prod.yml logs -f backend | grep -i "analysis_watchdog"
```

Events to watch for:

| Event | Level | Meaning |
|-------|-------|---------|
| `analysis_watchdog_started` | INFO | Watchdog is running |
| `analysis_watchdog_analyzed` | INFO | A shipment was successfully analyzed |
| `analysis_watchdog_cycle_complete` | INFO | Cycle finished (includes processed/error counts) |
| `analysis_watchdog_error` | WARNING | Analysis failed for a specific shipment |
| `analysis_watchdog_skipped` | DEBUG | Another worker holds the lock |

### Disabling the Watchdog

```bash
# Dev: Edit docker-compose.yml, set ANALYSIS_WATCHDOG_ENABLED: "false"
docker compose restart backend

# Production:
sed -i 's/ANALYSIS_WATCHDOG_ENABLED=true/ANALYSIS_WATCHDOG_ENABLED=false/' .env.prod
docker compose -f docker-compose.prod.yml restart backend
```

### Per-Shipment Locking

The watchdog uses per-shipment Redis locks (`analysis:shipment:{id}`, TTL 5 min) to prevent concurrent analysis of the same shipment. The global lock (`analysis_watchdog:lock`, TTL 2 min) ensures only one watchdog instance runs per cycle.

---

## Troubleshooting Runbook

### Container Won't Start

```bash
# Check container logs
docker compose -f docker-compose.prod.yml logs backend

# Check Docker events
docker events --since 10m

# Check if the container is restarting
docker compose -f docker-compose.prod.yml ps
# Look for "Restarting" in STATUS
```

### Backend Can't Connect to Database

```bash
# Verify postgres is healthy
docker compose -f docker-compose.prod.yml exec postgres pg_isready -U clearance

# Verify the password matches
docker compose -f docker-compose.prod.yml exec backend env | grep DATABASE_URL

# Test connection manually
docker compose -f docker-compose.prod.yml exec postgres \
  psql -U clearance -d clearance_engine -c "SELECT 1;"
```

### Caddy Returns 502 Bad Gateway

This means Caddy can't reach the backend or frontend:

```bash
# Check if backend is running
docker compose -f docker-compose.prod.yml ps backend

# Check backend logs
docker compose -f docker-compose.prod.yml logs --tail 50 backend

# Verify backend is listening
docker compose -f docker-compose.prod.yml exec backend curl -sf http://localhost:4000/health
```

### TLS Certificate Not Working

```bash
# Check Caddy logs for ACME errors
docker compose -f docker-compose.prod.yml logs caddy | grep -i "acme\|tls\|certificate"

# Verify DNS is pointing correctly
dig +short clearance.yourdomain.com
# Should return the Elastic IP
```

Common causes:
- DNS not propagated yet (wait and retry)
- Port 80 not open (Caddy needs this for ACME HTTP challenge)
- Domain set to `localhost` in `.env.prod`

### Disk Space Running Low

```bash
# Check disk usage
df -h

# Docker is usually the culprit
docker system df

# Prune unused images, containers, and volumes
docker system prune -a

# Prune only dangling images (safer)
docker image prune
```

### Migration Fails

```bash
# Check current migration state
docker compose -f docker-compose.prod.yml exec backend uv run alembic current

# Check migration history
docker compose -f docker-compose.prod.yml exec backend uv run alembic history

# If stuck, manually stamp the database to a known state
docker compose -f docker-compose.prod.yml exec backend uv run alembic stamp <revision>
```

### SSE Streaming Not Working

If the chat/classification/analysis streaming endpoints hang or batch responses:

1. Verify Caddy's `flush_interval -1` is in the Caddyfile
2. Check that requests go through `/api/*` (not directly to the backend)
3. Restart Caddy: `docker compose -f docker-compose.prod.yml restart caddy`

### Regulatory Monitor Not Running

```bash
# Check if the monitor is enabled
docker compose -f docker-compose.prod.yml exec backend env | grep REGULATORY_MONITOR

# Check monitor status via API
curl -u operator:password https://clearance.yourdomain.com/api/regulatory/monitor/status

# Check backend logs for scheduler errors
docker compose -f docker-compose.prod.yml logs --tail 100 backend | grep -i "scheduler\|monitor\|apscheduler"

# Verify Redis is available (needed for distributed lock)
docker compose -f docker-compose.prod.yml exec redis redis-cli ping
```

Common causes:
- `REGULATORY_MONITOR_ENABLED` not set to `true` in `.env.prod`
- Redis is down (monitor needs Redis for the distributed lock)
- No LLM API key configured (extraction requires at least one provider)
- APScheduler failed to start (check backend startup logs)

### Regulatory Monitor Producing No New Signals

```bash
# Trigger manual refresh and check logs
curl -u operator:password -X POST https://clearance.yourdomain.com/api/regulatory/refresh
docker compose -f docker-compose.prod.yml logs -f --tail 50 backend
```

Common causes:
- All feeds are returning errors (network/firewall issue — check `feed_fetch_failed` logs)
- LLM extraction returning invalid JSON (check `signal_extraction_failed` logs)
- Deduplication filtering all signals (expected if feeds haven't changed since last run)
- Redis lock stuck from a crashed worker (lock auto-expires after 30 min; or manually: `redis-cli DEL regulatory_monitor:lock`)

### Out of Memory

```bash
# Check memory usage
free -h
docker stats --no-stream

# Identify the memory hog
docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}"
```

If PostgreSQL is using too much memory, consider adding resource limits in `docker-compose.prod.yml`:

```yaml
postgres:
  deploy:
    resources:
      limits:
        memory: 2G
```

---

---

## Kubernetes Deployment (Alternative)

A full set of Kubernetes manifests exists in `clearance-engine/k8s/` as an alternative to the Docker Compose + EC2 deployment. These are managed via Kustomize.

### Applying the Stack

```bash
kubectl apply -k clearance-engine/k8s/
```

This creates all resources in the `clearance-engine` namespace:

| Resource                 | Kind          | Notes                                      |
|--------------------------|---------------|--------------------------------------------|
| `clearance-engine`       | Namespace     | Isolates all resources                     |
| `clearance-engine-config`| ConfigMap     | Non-sensitive environment variables        |
| `clearance-engine-secrets`| Secret       | Base64-encoded API keys and passwords      |
| PostgreSQL               | StatefulSet   | 1 replica, 5Gi PVC                         |
| Redis                    | Deployment    | 1 replica, 200MB maxmemory                 |
| Qdrant                   | StatefulSet   | 1 replica, 5Gi PVC                         |
| Backend API              | Deployment    | 2 replicas, rolling updates                |
| Frontend                 | Deployment    | 2 replicas, rolling updates                |
| Ingress                  | Ingress       | nginx-ingress, host: `clearance.local`     |

### Key Operational Notes

- **Secrets:** Edit `k8s/secrets.yaml` and base64-encode values before applying. Do not commit plaintext secrets.
- **Ingress host:** Change `clearance.local` in `k8s/ingress.yaml` to your actual domain.
- **Backend init container:** Waits for PostgreSQL to be ready before starting the API.
- **Health probes:** Backend has liveness (30s initial delay) and readiness (10s initial delay) probes on `/health`.
- **Resource limits:** All pods have CPU and memory requests/limits defined.

### Checking Status

```bash
kubectl -n clearance-engine get pods
kubectl -n clearance-engine get svc
kubectl -n clearance-engine logs -f deployment/backend
```

**Note:** The K8s manifests and Docker Compose deployments are independent paths. The primary supported production deployment is Docker Compose on EC2 (see [deployment-production.md](deployment-production.md)). The K8s manifests are provided for teams that prefer container orchestration.

---

## Periodic Maintenance Checklist

### Weekly

- [ ] Check `/health` endpoint returns all services healthy
- [ ] Review container logs for recurring errors: `docker compose logs --since 168h | grep -i error`
- [ ] Check disk usage: `df -h` and `docker system df`
- [ ] Check regulatory monitor status: `GET /api/regulatory/monitor/status` — verify `error_count` is low and `last_run` is recent

### Monthly

- [ ] Pull updated base images: `docker compose pull`
- [ ] Update system packages: `sudo dnf update -y`
- [ ] Review and rotate API keys if needed
- [ ] Create a database backup
- [ ] Check for Python/npm dependency security advisories
- [ ] Review regulatory signal count and archive old signals (>6 months) if table is growing large
- [ ] Spot-check recent regulatory signals for accuracy (verify LLM extraction quality)

### Quarterly

- [ ] Audit security group rules (SSH CIDR, open ports)
- [ ] Review and update rate limit thresholds if usage patterns changed
- [ ] Test disaster recovery: restore a database backup on a test instance
- [ ] Update Docker Compose version if a new release is available
- [ ] Review EBS volume size and instance type against actual usage
- [ ] Review regulatory monitor LLM costs and adjust extraction model or interval if needed
