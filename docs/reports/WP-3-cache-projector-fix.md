# WP-3: Cache Projector Fix

**Finding:** #7 — Cache Projector & Redis Usage Unsustainable
**Agent:** cache-eng
**Branch:** `wp-3-cache-projector`
**Date:** 2026-02-27

## Problem Statement

The production readiness review identified three critical issues with the cache projector:

1. **No TTLs** — All Redis writes were permanent (no `ex=` parameter), causing unbounded memory growth.
2. **No key partitioning** — Cache keys lacked domain prefixes, making it impossible to selectively flush or monitor per-domain cache usage.
3. **Cache hits discarded** — Route handlers called `get_cached()` but ignored the return value, always falling through to PostgreSQL queries, making the cache decorative.
4. **Full-stream replay** — Rebuilds replayed ALL messages from ALL JetStream streams from sequence 1, causing O(total_events) startup time.

## Changes Made

### 1. BaseProjector TTL Support (`clearance_platform/shared/projector/base.py`)

- Added `CACHE_TTL_SECONDS` (default 300s), `QUEUE_CACHE_TTL_SECONDS` (60s), `AGG_CACHE_TTL_SECONDS` (900s) — all configurable via environment variables.
- Added `ttl` constructor parameter to `BaseProjector`.
- All write methods (`_set_json`, `_set_list`, `_set_json_versioned`) now include `ex=self.ttl`.
- Added `cache_set(key, value)` — standard TTL write.
- Added `cache_set_permanent(key, value)` — for index pointers that must persist until explicit deletion.
- All 15 domain projectors inherit TTL automatically through `_set_json()`.

### 2. Domain-Partitioned Keys (`clearance_platform/cache_projector/projector.py`)

All Redis keys now follow the pattern:

| Key Type | Pattern | Example |
|----------|---------|---------|
| Entity snapshot | `{domain}:entity:{type}:{id}` | `shipment_lifecycle:entity:shipment_lifecycle:abc123` |
| Secondary index | `{domain}:idx:{type}:by-{attr}:{value}` | `shipment_lifecycle:idx:shipment:by-status:abc123` |
| Aggregate counter | `{domain}:agg:{name}` | `product_catalog:agg:event_count` |
| Broker queue | `{domain}:queue:{name}:{key}` | `declaration_management:queue:broker:filed` |
| Global aggregate | `global:agg:{name}` | `global:agg:event_count_total` |

### 3. TTLs on All Write Operations

| Write Type | TTL | Env Var |
|------------|-----|---------|
| Entity snapshots | 300s (5 min) | `CACHE_TTL_SECONDS` |
| Secondary indexes | 300s (5 min) | `CACHE_TTL_SECONDS` |
| Broker queue sets | 60s (1 min) | `QUEUE_CACHE_TTL_SECONDS` |
| Aggregate counters | 900s (15 min) | `AGG_CACHE_TTL_SECONDS` |
| Sequence pointer | No TTL | (system metadata) |

### 4. Cache-First Route Handlers

Fixed 4 route handlers to use cache-first pattern (return cached data on hit, query DB on miss, cache result):

- `shipment_lifecycle/routes.py` — `get_shipment()`, `list_companies()`
- `order_management/routes.py` — `get_order()`
- `cargo_handling_units/routes.py` — `get_handling_unit()`

Created shared utility (`clearance_platform/shared/cache_utils.py`):
- `get_cached_or_none(request, domain, key_type, entity_type, entity_id)` — reads domain-partitioned key, returns deserialized JSON or None.
- `set_cache(request, domain, key_type, entity_type, entity_id, data, ttl)` — writes with TTL.

### 5. Incremental Rebuild

- Added `LAST_SEQ_KEY = "cache_projector:last_seq"` tracking in Redis.
- `project_event()` now accepts optional `seq` parameter and stores it.
- `rebuild()` reads last_seq and replays only from `last_seq + 1`.
- `/admin/rebuild` endpoint accepts `?full=true` query param to force full replay.
- `/health` endpoint now reports `last_seq` for monitoring.
- Reconciliation task also uses incremental replay.

### 6. Reconciliation Service Update (`clearance_platform/shared/reconciliation.py`)

- Updated key scan pattern from `entity:*` to `{domain}:entity:*` for each of the 15 domains.

## Files Changed

| File | Change |
|------|--------|
| `clearance_platform/shared/projector/base.py` | TTL on all writes, `cache_set()`/`cache_set_permanent()`, configurable TTL |
| `clearance_platform/shared/projector/__init__.py` | Export TTL constants |
| `clearance_platform/cache_projector/projector.py` | Domain-prefixed keys, TTLs, seq tracking, incremental rebuild |
| `clearance_platform/shared/reconciliation.py` | Scan domain-prefixed keys |
| `clearance_platform/shared/cache_utils.py` | **New** — shared cache read/write utilities |
| `clearance_platform/domains/shipment_lifecycle/routes.py` | Cache-first `get_shipment()`, `list_companies()` |
| `clearance_platform/domains/order_management/routes.py` | Cache-first `get_order()` |
| `clearance_platform/domains/cargo_handling_units/routes.py` | Cache-first `get_handling_unit()` |
| `tests/unit/test_cache_projector_ttl.py` | **New** — 21 tests covering TTL, keys, cache-first, sequence |

## Tests

- **21 new tests** in `tests/unit/test_cache_projector_ttl.py` — all passing
- **13 existing tests** in `tests/unit/test_projector.py` — all still passing
- **Full suite**: No regressions

### Test Coverage

| Test Class | Tests | Status |
|------------|-------|--------|
| `TestBaseProjectorTTL` | 7 | All pass |
| `TestCacheKeyPartitioning` | 3 | All pass |
| `TestProjectEventTTL` | 1 | All pass |
| `TestIncrementalRebuild` | 2 | All pass |
| `TestCacheUtils` | 4 | All pass |
| `TestDomainProjectorTTL` | 3 | All pass |
| `TestQueueIndexTTL` | 1 | All pass |

## Verification Steps

```bash
# Run WP-3 specific tests
cd clearance-engine/backend
uv run python -m pytest tests/unit/test_cache_projector_ttl.py -v

# Run existing projector tests (regression check)
uv run python -m pytest tests/unit/test_projector.py -v

# Run full suite
uv run python -m pytest tests/ -x
```

## Residual Risks

1. **Remaining route handlers** — Only 4 of ~40 route handlers use cache-first pattern. The remaining handlers still call `get_cached()` but ignore the result. These are functional (they just bypass cache), but represent missed optimization opportunities. The `get_cached` helpers in those files have been updated to return deserialized JSON, so the infrastructure is ready when those handlers are updated.

2. **Key migration** — Existing Redis data uses old key format (`entity:{domain}:{id}`). On first deploy, the cache will be cold until events repopulate with new keys. This is expected and safe since the DB fallback works correctly.

3. **TTL tuning** — The 300s default is conservative. Production monitoring should inform whether entity TTLs should be longer (e.g., 600s for shipment snapshots) or shorter (e.g., 60s for dashboard aggregates).
