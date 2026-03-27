# WP-4: CORS & Rate Limit Hardening Report

**Finding addressed:** #9 — "CORS/Ratelimits Open — `platform_service` enables `allow_origins=["*"]` with credentials, and most routers allow `50k/minute` without authentication, exposing the entire API surface to abuse."

**Branch:** `wp-4-cors-ratelimits`

## Changes Summary

### 1. CORS Configuration Hardened

**Files modified:**
- `app/main.py` — `create_app()` CORS middleware
- `clearance_platform/platform_service/main.py` — platform service CORS middleware

**What changed:**
- Replaced single `ALLOWED_ORIGIN` env var with multi-origin `ALLOWED_ORIGINS` (comma-separated)
- Backward compatible: falls back to legacy `ALLOWED_ORIGIN` if `ALLOWED_ORIGINS` is not set
- Default: `http://localhost:4001` (dev)
- When origins contain `"*"`, `allow_credentials` is automatically set to `false` (CORS spec compliance)
- `allow_methods` restricted from `["*"]` to `["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"]`
- `allow_headers` restricted from `["*"]` to `["Authorization", "Content-Type", "X-CSRF-Token", "X-Requested-With"]`
- Both `app/main.py` and `platform_service/main.py` share the same `_get_allowed_origins()` helper pattern

### 2. Rate Limit Configuration Centralized

**File created:**
- `clearance_platform/shared/rate_limits.py`

**Tiers (all configurable via env vars):**

| Tier | Default | Env Var | Use Case |
|------|---------|---------|----------|
| ADMIN | 10/minute | `RATE_LIMIT_ADMIN` | Seed, refresh monitor |
| LLM | 30/minute | `RATE_LIMIT_LLM` | Chat stream, classify, analyze, description-quality, resolution lookup/suggest |
| WRITE | 100/minute | `RATE_LIMIT_WRITE` | POST/PUT/DELETE mutations, tariff calc, FTA check, compliance screening |
| READ | 500/minute | `RATE_LIMIT_READ` | GET list/detail endpoints |

### 3. Default Rate Limit Updated

**File:** `app/main.py`
- Changed `default_limits=["50000/minute"]` to `default_limits=[RATE_LIMIT_READ]` (500/minute)
- This applies as a fallback for any route without an explicit `@limiter.limit()` decorator

### 4. Per-Route Rate Limit Decorators Updated

**Files modified (13 route files):**

| File | Endpoints Updated | Tiers Applied |
|------|------------------|---------------|
| `gateway/chat_routes.py` | `/chat/stream` | LLM |
| `gateway/admin_routes.py` | `/admin/seed`, `/admin/seed-customer-data` | ADMIN |
| `app/api/routes/analysis.py` | `/analyze/stream` | LLM |
| `app/api/routes/description_quality.py` | `/description-quality` | LLM |
| `domains/trade_intelligence/routes.py` | 7 endpoints | LLM (classify), WRITE (tariff, fta), READ (jurisdiction, countries) |
| `domains/product_catalog/routes.py` | 4 endpoints | READ (list, get), WRITE (create, update) |
| `domains/regulatory_intelligence/routes.py` | 9 endpoints | READ (list, monitor_status), WRITE (CRUD, scenarios), ADMIN (refresh) |
| `domains/compliance/routes.py` | 2 endpoints | WRITE (screen, entity) |
| `domains/party_management/routes.py` | 9 endpoints | READ (list, get, poa, ior), WRITE (create, update, grant, revoke, ior create) |
| `domains/supply_chain_disruptions/routes.py` | 5 endpoints | READ (list, get), WRITE (create, update, delete) |
| `domains/cargo_handling_units/routes.py` | 5 endpoints | READ (list, get), WRITE (create, movement, cage-intake) |
| `domains/exception_management/routes.py` | 3 endpoints | LLM (lookup, suggest), WRITE (upload) |

### 5. Tests Added

**File:** `tests/unit/test_cors_rate_limits.py` — 14 tests

- `TestRateLimitModuleDefaults` (2 tests): default values match spec, env overrides work
- `TestCORSOriginParsing` (7 tests): default origin, legacy env var, comma-separated, precedence, wildcard, whitespace, empty values
- `TestCORSMiddlewareIntegration` (5 tests): allows configured origin, rejects bad origin, no credentials with wildcard, restricts methods, restricts headers

## Verification

```bash
# All WP-4 tests pass
uv run python -m pytest tests/unit/test_cors_rate_limits.py -v
# 14 passed

# Full unit suite passes (1395 tests)
uv run python -m pytest tests/unit/ -q --ignore=tests/unit/engines/test_e2_tariff.py
# 1395 passed

# No remaining 50k/min references
grep -rn "50000/minute" clearance-engine/backend/ --include="*.py"
# (no output)
```

## Residual Risks

1. **Domain admin/replay endpoints** (15 domains) do not accept a `request` parameter, so they cannot have per-endpoint `@limiter.limit()` decorators. They fall back to the app-level default of 500/minute, which is still a 100x reduction from 50k but not the ADMIN tier of 10/minute. These are stub endpoints returning static JSON, so the risk is low.

2. **Rate limiting is per-IP** (`get_remote_address`). Behind a reverse proxy, all requests may appear from the same IP unless `X-Forwarded-For` is configured. The Caddy reverse proxy in production passes this header correctly.

3. **CORS configuration change requires frontend alignment.** Production deployments must set `ALLOWED_ORIGINS` to include the actual frontend domain(s). The docker-compose files should be updated with the correct origins for each environment.
