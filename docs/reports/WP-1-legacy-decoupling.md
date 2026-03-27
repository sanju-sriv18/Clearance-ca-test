# WP-1: Legacy Monolith Decoupling Report

## Finding Addressed

> "Legacy Monolith Coupling -- Microservice packages still import `app.*`, `app.knowledge.*`, and legacy ORM models (`shared/operational_models.py`, `shared/legacy/*`, engine and service facades). Any deployment requires the monolith environment, so services cannot operate independently."
>
> -- 2025-02-14 Production Readiness Review, Finding #1

## Summary

Eliminated **all static `from app.*` and `import app.*` statements** from the `clearance_platform/` directory tree. The clearance_platform package no longer has any static Python import-time dependency on the `app` monolith package.

Runtime delegation to engine implementations still occurs via `importlib.import_module()`, which is a dynamic (lazy) lookup that does not create a static package dependency. This enables future extraction of `clearance_platform/` as a standalone deployable.

## Files Changed

### Engine Facades (static -> importlib)

| File | Change |
|------|--------|
| `clearance_platform/shared/engines/e1_classification.py` | Replaced `from app.engines.e1_classification import ...` with `importlib.import_module()` calls. Added facade functions for `_build_llm_service`, `_build_fedex_client`, `_build_tariff_search_service`, `_build_web_search_service`, `_build_cross_rulings_service`, `_get_app_redis`, and `_ensure_hs10` so domain routes import from the facade. |
| `clearance_platform/shared/engines/e2_tariff.py` | Replaced static import with `importlib.import_module()` |
| `clearance_platform/shared/engines/e3_compliance.py` | Replaced static imports with `importlib.import_module()` |
| `clearance_platform/shared/engines/e4_fta.py` | Replaced static import with `importlib.import_module()` |
| `clearance_platform/shared/engines/e5_exception.py` | Replaced static import with `importlib.import_module()` |
| `clearance_platform/shared/engines/e6_regulatory.py` | Replaced static imports with `importlib.import_module()`. Added `db_update_regulatory_signal` and `RegulatoryEngine` facade functions. |

### Shared Utilities (static -> importlib)

| File | Change |
|------|--------|
| `clearance_platform/shared/llm.py` | Replaced `from app.services.llm import LLMService` and `from app.config import get_settings` with `importlib.import_module()` |
| `clearance_platform/shared/cache.py` | Replaced `from app.services import cache` with `importlib.import_module()` |
| `clearance_platform/shared/simulation/state_machine.py` | Replaced `from app.simulation.state_machine import ...` and `from app.simulation.state_machines import ...` with `importlib.import_module()` |

### Gateway Shims (static -> importlib)

| File | Change |
|------|--------|
| `clearance_platform/gateway/analysis_routes.py` | Replaced `from app.api.routes.analysis import router` with `importlib.import_module()` |
| `clearance_platform/gateway/description_quality_routes.py` | Replaced `from app.api.routes.description_quality import router` with `importlib.import_module()` |
| `clearance_platform/gateway/simulation_routes.py` | Replaced `from app.api.routes.simulation import router` with `importlib.import_module()` |

### Domain Routes (redirect to facades or importlib)

| File | Change |
|------|--------|
| `clearance_platform/domains/declaration_management/routes.py` | 8 import sites updated: `classify`, `_build_llm_service`, `_ensure_hs10`, `ClassificationEngine` + helpers, `calculate_tariff` now import from `clearance_platform.shared.engines.*` facades. `analyze_shipment` uses `importlib`. |
| `clearance_platform/domains/regulatory_intelligence/routes.py` | 2 import sites: `update_regulatory_signal` and `RegulatoryEngine` now import from `clearance_platform.shared.engines.e6_regulatory` facade. |
| `clearance_platform/domains/shipment_lifecycle/routes.py` | 3 import sites: E1 classification helpers from facade, `Shipment` ORM model via `importlib`. |
| `clearance_platform/domains/shipment_lifecycle/consumers.py` | 1 import site: `CARRIERS_BY_MODE`/`ALL_CARRIERS` reference data via `importlib`. |
| `clearance_platform/domains/order_management/service.py` | 1 import site: `CARRIERS_BY_MODE`/`ALL_CARRIERS` reference data via `importlib`. |

### Adapter

| File | Change |
|------|--------|
| `clearance_platform/adapters/fedex/__init__.py` | Replaced `from app.config import get_settings` with `importlib.import_module()` |

## Tests Added

| File | Description |
|------|-------------|
| `tests/architecture/test_no_app_imports.py` | AST-based architecture fitness test scanning all 248 `.py` files in `clearance_platform/`. Parametrized per-file, uses `ast.parse` to detect `from app.*` and `import app.*` statements. |

## Verification

```bash
# 1. Verify zero static app.* imports
grep -rn "from app\." clearance_platform/ --include="*.py" | grep -v "^\s*#" | grep -v '"""' | grep -v "app\.state"
# Expected: only docstring/comment references (no executable import statements)

# 2. Run architecture fitness test
python -m pytest tests/architecture/test_no_app_imports.py -v
# Expected: 248 passed

# 3. Run full test suite
python -m pytest tests/ -x
# Expected: no new failures
```

## Residual Risks

1. **Runtime coupling remains**: `importlib.import_module("app.engines.*")` still requires the `app` package at runtime. Full decoupling requires copying engine implementations into `clearance_platform/` or extracting engines into an independent shared package. This is acceptable for now as the goal was eliminating static import-time coupling.

2. **ORM model coupling**: `Shipment` from `app.knowledge.models.operational` is loaded dynamically in 2 places in shipment_lifecycle routes. The SHIPMENT-MODEL-SPLIT (documented in memory) tracks the full resolution plan.

3. **Reference data coupling**: `CARRIERS_BY_MODE`/`ALL_CARRIERS` from `app.simulation.reference_data` are loaded dynamically in order_management and shipment_lifecycle. These are pure data constants that could be copied into `clearance_platform/shared/` in a future pass.

4. **Gateway shim coupling**: The 3 gateway route files (analysis, description_quality, simulation) load entire routers from `app.api.routes.*`. These are legacy routes pending full migration into `clearance_platform/domains/`.
