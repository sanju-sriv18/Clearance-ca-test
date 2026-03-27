# WP-7: Authentication Middleware Report

**Finding addressed:** #3 -- "Unauthenticated High-Impact APIs -- `/api/admin/seed*`, `/api/chat/stream`, `/api/session/*`, `/api/financial/demurrage`, `/api/adjudication/*`, etc., accept arbitrary requests with no auth or CSRF protection while allowing destructive operations (data wipes, LLM tool execution, shipment mutations)."

**Branch:** `wp-7-auth-middleware`

## Changes Summary

### 1. Auth Module Created

**Directory:** `clearance_platform/shared/auth/`

| File | Purpose |
|------|---------|
| `__init__.py` | Public API: `get_current_user`, `require_auth`, `require_role`, `User`, `AuthToken` |
| `config.py` | JWT configuration: secret, algorithm, expiry, admin users (all env-configurable) |
| `models.py` | Pydantic models: `User`, `AuthToken`, `LoginRequest` |
| `dependencies.py` | FastAPI dependencies: `get_current_user` (Bearer + cookie), `require_auth`, `require_role(role)` |
| `middleware.py` | `AuthMiddleware` -- enforces JWT on all routes except health/auth/docs |
| `routes.py` | Auth endpoints: `POST /api/auth/login`, `POST /api/auth/logout`, `GET /api/auth/me` |

### 2. Auth Middleware Added

**Files modified:**
- `app/main.py` -- `AuthMiddleware` added to middleware stack
- `clearance_platform/platform_service/main.py` -- same middleware + auth router registered

**How it works:**
- Intercepts all requests before they reach route handlers
- Extracts JWT from `Authorization: Bearer <token>` header or `access_token` cookie
- On success: sets `request.state.user` with claims (`sub`, `role`, `email`)
- On failure: returns 401 JSON response
- OPTIONS requests always pass through (CORS preflight)

**Excluded paths (no auth required):**
- `/health` and any path ending in `/health` (domain service health checks)
- `/api/auth/login`, `/api/auth/logout`, `/api/auth/*`
- `/`, `/docs`, `/redoc`, `/openapi.json`

### 3. Role-Based Access Control

**File modified:** `clearance_platform/gateway/admin_routes.py`

- Added `dependencies=[Depends(require_role("admin"))]` to the admin router
- All `/api/admin/*` endpoints now require `admin` role
- Viewers and brokers get 403 Forbidden

### 4. Auth Router Registered

**File modified:** `clearance_platform/gateway/router_registry.py`

- Auth router registered first (before domain routers) so `/api/auth/*` endpoints are available

### 5. Frontend Auth Integration

**Files modified:**
- `clearance-engine/frontend/src/api/client.ts`:
  - Added `getAccessToken()`, `setAccessToken()`, `clearAccessToken()` helpers
  - Added `login()`, `logout()`, `getCurrentUser()` API functions
  - `fetchJSON()` now includes `Authorization` header and `credentials: "include"`
  - `fetchJSON()` now handles 401 by redirecting to `/login`
  - All 5 raw `fetch()` calls updated with auth headers and credentials
- `clearance-engine/frontend/src/hooks/useDashboardStream.ts`:
  - Polling fallback includes auth headers

**Files created:**
- `clearance-engine/frontend/src/surfaces/auth/LoginPage.tsx` -- Login form with error handling
- `clearance-engine/frontend/src/router.tsx` -- Added `/login` route

### 6. JWT Configuration

| Setting | Default | Env Var |
|---------|---------|---------|
| Secret | `dev-secret-change-in-production` | `JWT_SECRET` |
| Algorithm | `HS256` | -- |
| Expiry | 24 hours | `JWT_EXPIRY_HOURS` |
| Admin users | `admin` | `ADMIN_USERS` (comma-separated) |
| Secure cookie | `false` | `COOKIE_SECURE` |

### 7. Dev User Credentials

| Username | Password | Role |
|----------|----------|------|
| admin | admin | admin |
| broker | broker | broker |
| viewer | viewer | viewer |

### 8. Tests Added

**File:** `tests/unit/test_auth_middleware.py` -- 25 tests

- `TestAuthMiddleware` (9 tests): health no auth, domain health no auth, login no auth, unauthenticated 401, valid bearer, expired token, invalid token, cookie fallback, OPTIONS preflight
- `TestAuthDependencies` (4 tests): require_auth, admin role viewer 403, admin role admin passes, admin role broker 403
- `TestLoginLogout` (7 tests): valid login, httpOnly cookie, invalid credentials, unknown user, logout, get_me with token, get_me without token 401
- `TestAuthConfig` (3 tests): default secret, env override secret, env override expiry, env override admin users
- `TestEndToEndAuthFlow` (1 test): full login -> access -> logout flow

## Verification

```bash
# All WP-7 tests pass
uv run python -m pytest tests/unit/test_auth_middleware.py -v
# 25 passed

# Full unit suite (excluding pre-existing e2_tariff failures)
uv run python -m pytest tests/unit/ -q --ignore=tests/unit/engines/test_e2_tariff.py
# 1329 passed (or 1412 total with e2_tariff: 2 pre-existing failures)
```

## Residual Risks

1. **Dev user store is in-memory.** The `DEV_USERS` dict in `auth/routes.py` is for development only. Production deployments must integrate with an external identity provider (OIDC/SAML) or a database-backed user store.

2. **JWT secret must be changed for production.** The default `dev-secret-change-in-production` is intentionally insecure. Production must set `JWT_SECRET` to a cryptographically random string of at least 32 bytes.

3. **EventSource SSE connections** (dashboard stream) rely on cookie-based auth since the `EventSource` API does not support custom headers. The `access_token` httpOnly cookie handles this automatically for same-origin requests.

4. **No token refresh mechanism.** Tokens expire after 24 hours (configurable). Users must re-login after expiry. A refresh token flow can be added in a future iteration.

5. **CSRF protection** is partially addressed by the `SameSite=Strict` cookie policy, but dedicated CSRF tokens are not implemented. The `X-CSRF-Token` header is accepted by CORS but not validated server-side.
