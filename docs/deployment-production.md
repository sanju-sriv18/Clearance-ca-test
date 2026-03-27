# Production Deployment (AWS)

> **Author:** Simon Kissler (simon.kissler@accenture.com) | **Co-Author:** Claude Code (Anthropic)

This guide covers provisioning infrastructure on AWS with Terraform and deploying the Clearance Intelligence Engine to a production EC2 instance.

## Architecture Overview

Production runs six containers on a single EC2 instance behind a Caddy reverse proxy:

```
Internet
  │
  ├── :80  ──► Caddy (HTTP → HTTPS redirect)
  └── :443 ──► Caddy (TLS termination + basic auth)
                 │
                 ├── /api/*  ──► Backend :4000  (FastAPI, 2 workers)
                 └── /*      ──► Frontend :4001 (nginx serving static build)

                 Backend connects to:
                   ├── PostgreSQL :5432  (not exposed externally)
                   ├── Redis :6379       (not exposed externally)
                   └── Qdrant :6333      (not exposed externally)
```

Key differences from dev:

| Aspect                | Dev                            | Production                              |
|-----------------------|--------------------------------|-----------------------------------------|
| Reverse proxy         | None                           | Caddy (HTTPS, basic auth, security headers) |
| Frontend build        | Vite dev server                | Multi-stage: npm build → nginx static   |
| Backend workers       | 1 (with `--reload`)           | 2 (no reload)                           |
| Database ports        | Exposed on host (5440)        | Internal only (container network)       |
| Redis ports           | Exposed on host (6381)        | Internal only                           |
| CORS                  | `*`                           | Domain-specific                         |
| Qdrant ports          | Exposed on host (6336/6337)   | Internal only                           |
| Backend/Frontend ports| Exposed on host (4000/4001)   | Internal only (Caddy proxies)           |
| Restart policy        | None                          | `unless-stopped`                        |
| Secrets               | `.env` with dev defaults      | `.env.prod` with real credentials       |

## Prerequisites

- **Terraform >= 1.5** installed locally
- **AWS CLI** configured with credentials that can create EC2, EIP, SG, and Key Pair resources
- An **SSH key pair** (ed25519 recommended)
- A **domain name** (optional but recommended for real TLS certs)
- LLM API keys (Anthropic and/or OpenAI)

## Initial Setup

### 1. Configure Terraform Variables

```bash
cd infra
cp terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars`:

```hcl
# Required
ssh_public_key    = "ssh-ed25519 AAAA... you@host"
auth_pass         = "strong-password-here"       # Caddy basic auth password
postgres_password = "generate-with-openssl"       # DB password

# Recommended
domain            = "clearance.yourdomain.com"    # Or "localhost" for IP-only access
anthropic_api_key = "sk-ant-..."
openai_api_key    = "sk-..."

# Optional overrides
# aws_region      = "us-east-2"                  # Default region
# instance_type   = "t4g.xlarge"                 # 4 vCPU, 16GB RAM, ARM64
# volume_size     = 30                           # Root EBS in GB
# ssh_cidr_blocks = ["203.0.113.42/32"]          # Restrict SSH access
# auth_user       = "operator"                   # Basic auth username
# allowed_origin  = "https://clearance.yourdomain.com"  # Auto-set from domain if blank
# github_repo     = "https://github.com/warum26/clearance_vibe.git"  # Default repo
# github_pat      = "ghp_..."                    # GitHub PAT (required for private repos)
```

Generate secure passwords:

```bash
openssl rand -base64 24   # For postgres_password
openssl rand -base64 24   # For auth_pass
```

**Password note:** Avoid single quotes in `auth_pass` — the value is passed through shell interpolation during Caddy hash generation.

### 2. Provision Infrastructure

```bash
cd infra

terraform init
terraform plan          # Review what will be created
terraform apply         # Confirm with 'yes'
```

This creates:

- **EC2 instance** (t4g.xlarge, Amazon Linux 2023 ARM64, 30GB gp3)
- **Elastic IP** (static public IP that survives instance stop/start)
- **Security group** allowing SSH (22), HTTP (80), HTTPS (443)
- **Key pair** from your SSH public key

### 3. Set Up DNS (If Using a Domain)

After `terraform apply`, note the Elastic IP from the output:

```bash
terraform output elastic_ip
```

Create an **A record** in your DNS provider pointing your domain to this IP:

```
clearance.yourdomain.com  →  A  →  <elastic-ip>
```

Caddy will automatically obtain and renew a Let's Encrypt TLS certificate once DNS propagates.

If using `domain = "localhost"` (no custom domain), Caddy generates a self-signed certificate and you'll access the app via `https://<elastic-ip>` (browser will show a certificate warning).

### 4. Wait for Bootstrap

The EC2 user-data script runs automatically on first boot. It:

1. Updates system packages
2. Installs Docker and Docker Compose v2
3. Clones the repository
4. Generates the Caddy password hash
5. Creates `.env.prod` with all secrets
6. Runs `deploy.sh` (builds containers, runs migrations, health check)

Monitor bootstrap progress:

```bash
# SSH into the instance
ssh -i <your-private-key> ec2-user@<elastic-ip>

# Watch the init log
tail -f /var/log/clearance-init.log

# Or check cloud-init status
cloud-init status --wait
```

Bootstrap is complete when you see `==> Bootstrap complete.` in the log.

### 5. Verify

```bash
# From your local machine (replace with your domain or IP):
curl -u operator:yourpassword https://clearance.yourdomain.com/api/health
```

Or visit `https://clearance.yourdomain.com` in a browser. You'll be prompted for basic auth credentials.

## Terraform Outputs Reference

After `terraform apply`, these outputs are available:

| Output              | Description                                  |
|---------------------|----------------------------------------------|
| `elastic_ip`        | Public IP address                            |
| `ssh_command`       | SSH command template                         |
| `app_url`           | Application URL (https)                      |
| `deploy_command`    | One-liner to redeploy from local machine     |
| `dns_instructions`  | DNS record instructions                      |

View all outputs:

```bash
cd infra && terraform output
```

## Subsequent Deployments

After the initial provisioning, deploy code updates by SSHing into the instance and running the deploy script:

```bash
# Option 1: One-liner from local machine
ssh ec2-user@<elastic-ip> 'cd ~/clearance_vibe/clearance-engine && bash deploy.sh'

# Option 2: Use the Terraform output
$(cd infra && terraform output -raw deploy_command)

# Option 3: SSH in and run manually
ssh -i <key> ec2-user@<elastic-ip>
cd ~/clearance_vibe/clearance-engine
bash deploy.sh
```

The deploy script (`deploy.sh`) performs these steps:

1. `git pull origin main` — fetch latest code
2. `docker compose -f docker-compose.prod.yml up --build -d` — rebuild and restart containers
3. `docker compose -f docker-compose.prod.yml exec backend uv run alembic upgrade head` — run pending migrations
4. Sleep 5 seconds to let services stabilize
5. Health check — runs `curl` inside the backend container against `http://localhost:4000/health`. **Note:** the health check is non-fatal; a failure prints a warning but does not abort the script
6. Print container status

## Production Docker Compose Details

### Container Network

All services communicate over an internal Docker network. Only the Caddy container exposes ports to the host:

- `:80` — HTTP (redirects to HTTPS)
- `:443` — HTTPS (TLS termination)

PostgreSQL, Redis, and Qdrant have **no exposed ports** in production.

### Caddy Configuration

The `Caddyfile` provides:

- **Automatic HTTPS** via Let's Encrypt (when using a real domain)
- **Basic authentication** on all routes
- **Reverse proxy** routing (`/api/*` → backend, `/*` → frontend)
- **SSE streaming support** via `flush_interval -1` on the backend proxy
- **Security headers**: HSTS, X-Content-Type-Options, X-Frame-Options, Referrer-Policy, Server header removal

### Frontend Build

The production frontend uses a multi-stage Docker build (`Dockerfile.prod`):

1. **Build stage** (node:20-alpine): `npm ci` → `npm run build` with `VITE_API_URL=/api`
2. **Serve stage** (nginx:alpine): Serves the static build with SPA routing, gzip compression, and security headers

In production the frontend makes API calls to `/api/*` (relative path), which Caddy proxies to the backend. There is no CORS issue because everything is served from the same origin.

### Environment Files

Production uses two env files:

- **`.env.prod`** — loaded by the backend container via `env_file`; contains API keys, auth hash, database password, etc.
- **`.env`** — symlinked to `.env.prod`; used by Docker Compose for variable substitution in the compose file itself (e.g., `${POSTGRES_PASSWORD}`, `${DOMAIN}`, `${AUTH_PASS_HASH}`)

The bcrypt hash in `AUTH_PASS_HASH` has its `$` characters escaped to `$$` for Docker Compose compatibility.

## Regulatory Monitor Configuration

The backend includes an automated regulatory feed monitor that watches government sources and trade news for policy changes. It uses the LLM to extract structured signals and stores them in the `regulatory_signals` database table.

### Enabling in Production

Add these variables to `.env.prod`:

| Variable | Recommended | Description |
|----------|-------------|-------------|
| `REGULATORY_MONITOR_ENABLED` | `true` | Enable the background monitor |
| `REGULATORY_MONITOR_INTERVAL_HOURS` | `2` | How often feeds are checked (hours) |
| `REGULATORY_MONITOR_EXTRACTION_MODEL` | *(empty)* | Override the LLM model for signal extraction (empty = use primary model) |
| `ANALYSIS_WATCHDOG_ENABLED` | `true` | Enable background analysis watchdog (recommended) |
| `ANALYSIS_WATCHDOG_INTERVAL_SECONDS` | `15` | Watchdog polling interval |
| `ANALYSIS_WATCHDOG_BATCH_SIZE` | `5` | Max shipments analyzed per watchdog cycle |

The monitor starts automatically with the backend when enabled. It uses APScheduler to run on the configured interval. A Redis distributed lock ensures only one worker runs the feed check per cycle, so it is safe with multiple uvicorn workers.

### Feed Sources

The monitor fetches from four sources in each cycle:

1. **Federal Register API** — CBP, ITC, and Commerce Department rules/notices
2. **CBP CSMS RSS** — Cargo Systems Messaging Service updates
3. **USTR Press Releases** — Trade policy announcements
4. **Google News RSS** — Global trade/tariff/sanctions news

### LLM Cost Estimate

Each monitor cycle processes ~20-30 articles in batches of 5 per LLM call. Approximate cost at 2-hour intervals:

- ~16K tokens per cycle, ~192K tokens per day
- **Claude Haiku**: ~$3-5/month
- **Claude Sonnet**: ~$15-20/month

To reduce cost, set `REGULATORY_MONITOR_EXTRACTION_MODEL` to a smaller model (e.g., `claude-haiku-3-20240307`) while keeping the primary model for user-facing endpoints.

### Verifying the Monitor

After deploy:

```bash
# Check monitor status
curl -u operator:password https://clearance.yourdomain.com/api/regulatory/monitor/status

# Trigger a manual refresh
curl -u operator:password -X POST https://clearance.yourdomain.com/api/regulatory/refresh

# View extracted signals
curl -u operator:password https://clearance.yourdomain.com/api/regulatory
```

---

## Security Considerations

### Network

- Restrict `ssh_cidr_blocks` to your IP/office CIDR instead of `0.0.0.0/0`
- PostgreSQL, Redis, and Qdrant are never exposed outside the Docker network
- All external traffic goes through Caddy with TLS and basic auth

### Secrets

- All sensitive Terraform variables are marked `sensitive = true` (won't appear in plan output)
- API keys, passwords, and auth hashes live in `.env.prod` on the instance
- The GitHub PAT (if used for private repos) is embedded in the `git clone` URL during bootstrap and is **logged to `/var/log/clearance-init.log`** because the user-data script runs with `set -x`. After initial setup, delete this log: `sudo rm /var/log/clearance-init.log`
- The PAT is also accessible via the EC2 instance metadata service (user-data). Consider enforcing IMDSv2 with a hop limit to restrict metadata access

### Encryption & Volume Security

- The Terraform config does not enable EBS encryption by default. For production workloads handling sensitive customs data, add `encrypted = true` to the `root_block_device` in `main.tf`
- Database, Redis, and Qdrant data are stored in Docker volumes on the root EBS volume — encrypting the root volume covers all of them

### Headers

Caddy adds these security headers to all responses:

- `Strict-Transport-Security: max-age=31536000; includeSubDomains`
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `Referrer-Policy: strict-origin-when-cross-origin`
- `Server` header is stripped

### Application

- Rate limiting via slowapi: 60 req/min default, 10 req/min on LLM-powered endpoints
- Swagger/OpenAPI docs are disabled in production (`DEBUG=false`)
- Backend runs as non-root `appuser` inside the container

## Destroying Infrastructure

```bash
cd infra
terraform destroy    # Confirm with 'yes'
```

This removes the EC2 instance, Elastic IP, security group, and key pair. **All data on the instance (including the database volume) is permanently lost.**
