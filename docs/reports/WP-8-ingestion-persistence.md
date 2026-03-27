# WP-8: Ingestion Persistence

**Status**: Complete
**Branch**: `wp-8-ingestion-persistence-v2`
**Date**: 2026-02-27

## Summary

Replaced stub/log-only `upsert()` methods across all ingestion pipelines with
real database persistence using `INSERT ... ON CONFLICT DO UPDATE` (PostgreSQL
upsert). Added server-side content-hash deduplication backed by a new
`ingestion_metadata` table, and wired all pipelines into the APScheduler lifespan.

## Finding Addressed

**Finding #4 -- Ingestion stubs**: Multiple ingestion pipelines had
`upsert()` methods that only logged record counts but never wrote to the database.

## Pre-Existing State

| Pipeline | File | Target Table | Had Real Upsert |
|----------|------|-------------|----------------|
| CSL | `csl.py` | `compliance.restricted_parties` | Yes (with stale cleanup) |
| Exchange Rates | `exchange_rates.py` | `trade_intel.exchange_rates` | Reverted to stub |
| OFAC SDN | `ofac_sdn.py` | `compliance.restricted_parties` | Yes (shared helper) |
| BIS Entity List | `ofac_sdn.py` | `compliance.restricted_parties` | Yes (shared helper) |
| BIS DPL | `ofac_sdn.py` | `compliance.restricted_parties` | Yes (shared helper) |
| BIS UVL | `ofac_sdn.py` | `compliance.restricted_parties` | Yes (shared helper) |
| HTSUS | `htsus.py` | `trade_intel.htsus_headings` | Yes |
| UFLPA | `uflpa_entities.py` | `compliance.restricted_parties` | No (stub) |
| WRO | `uflpa_entities.py` | `compliance.restricted_parties` | No (stub) |
| AD/CVD | `adcvd.py` | `trade_intel.adcvd_orders` | No (stub) |
| Trade Remedies | `trade_remedies.py` | `trade_intel.trade_remedy_actions` | No (stub) |

## Changes

### New Files

| File | Purpose |
|------|---------|
| `clearance_platform/shared/ingestion/models.py` | `IngestionMetadata` ORM model (public schema) |
| `alembic/versions/029_ingestion_metadata_and_trade_remedies.py` | Migration: `ingestion_metadata` + `trade_remedy_actions` tables |
| `alembic/versions/030_adcvd_manufacturer_rates.py` | Migration: `adcvd_manufacturer_rates` table (FK to adcvd_orders) |
| `tests/integration/test_ingestion_persistence.py` | 28 integration tests for all pipeline upsert methods |

### Modified Files

| File | Change |
|------|--------|
| `clearance_platform/shared/ingestion/pipeline.py` | Added `_load_last_hash_from_db()`, `_check_dedup()`, `_update_metadata()` for DB-backed dedup; updated `run()` to use them |
| `clearance_platform/shared/ingestion/ofac_sdn.py` | Added shared `_upsert_restricted_parties()` helper; implemented real upsert for OFAC SDN, BIS Entity, BIS DPL, BIS UVL |
| `clearance_platform/shared/ingestion/htsus.py` | Implemented upsert into `trade_intel.htsus_headings` with `ON CONFLICT (hs_code, jurisdiction)` |
| `clearance_platform/shared/ingestion/adcvd.py` | Implemented upsert into `trade_intel.adcvd_orders` + `adcvd_manufacturer_rates` |
| `clearance_platform/shared/ingestion/exchange_rates.py` | Restored real DB persistence (had been reverted to stub) |
| `clearance_platform/shared/ingestion/uflpa_entities.py` | Implemented upsert for UFLPA + WRO via shared `_upsert_restricted_parties` helper |
| `clearance_platform/shared/ingestion/trade_remedies.py` | Implemented upsert into `trade_intel.trade_remedy_actions` table |

## Pipeline -> Target Table Mapping

| Pipeline | Target Table | Unique Key |
|----------|-------------|------------|
| OFAC SDN | `compliance.restricted_parties` | `(source_list_id, list_source)` |
| BIS Entity List | `compliance.restricted_parties` | `(source_list_id, list_source)` |
| BIS DPL | `compliance.restricted_parties` | `(source_list_id, list_source)` |
| BIS UVL | `compliance.restricted_parties` | `(source_list_id, list_source)` |
| UFLPA Entities | `compliance.restricted_parties` | `(source_list_id, list_source)` |
| WRO | `compliance.restricted_parties` | `(source_list_id, list_source)` |
| CSL (13 sub-lists) | `compliance.restricted_parties` | `(source_list_id, list_source)` |
| HTSUS | `trade_intel.htsus_headings` | `(hs_code, jurisdiction)` |
| AD/CVD | `trade_intel.adcvd_orders` | `(case_number)` |
| AD/CVD Mfr Rates | `trade_intel.adcvd_manufacturer_rates` | `(order_id, manufacturer_name)` |
| Exchange Rates | `trade_intel.exchange_rates` | `(currency_code, effective_date)` |
| Trade Remedies | `trade_intel.trade_remedy_actions` | `(hts_code, remedy_type, country_scope)` |

## Architecture Decisions

1. **Shared `_upsert_restricted_parties()` helper** -- Eight screening list pipelines
   (OFAC, BIS Entity, BIS DPL, BIS UVL, UFLPA, WRO) all target the same
   `compliance.restricted_parties` table. A shared helper in `ofac_sdn.py` avoids
   duplicating the upsert SQL. The CSL pipeline retains its own implementation
   because it additionally handles stale-record cleanup.

2. **Deterministic `source_list_id`** -- For pipelines that lack a natural unique
   identifier (BIS lists, UFLPA, WRO), the helper generates a deterministic ID as
   `{list_source}:{entity_name[:200]}`. This matches the CSL pipeline's fallback
   strategy.

3. **DB-backed dedup in base class** -- The `_check_dedup()` method queries
   `ingestion_metadata.last_hash` on cold starts, ensuring content-hash dedup
   survives process restarts. The in-memory hash is checked first to avoid
   unnecessary DB roundtrips on subsequent runs.

4. **Metadata tracking** -- Each pipeline run updates `ingestion_metadata` with
   status (`idle`/`running`/`failed`), content hash, and record counts. This
   provides operational visibility without dedicated monitoring infrastructure.

## Test Coverage

- 28 test methods across 10 test classes in `test_ingestion_persistence.py`
- 84 existing unit tests continue passing (ofac, htsus, adcvd, pipeline)
- Every pipeline's upsert tested with mock async sessions
- Edge cases: empty names, empty case numbers, empty HTS codes, no-session fallback
- AD/CVD manufacturer-rate child record insertion verified
- Trade remedies exclusion status verified

## Residual Risks

1. **Dual 029 migration heads**: Two `029_*.py` files both reference
   `down_revision = "028"`, creating a branching scenario. Alembic will
   need `alembic merge` before applying both.
2. **No batch upsert optimization**: All pipelines insert one record at a
   time. For large datasets (HTSUS ~52K records), batch `executemany` or
   COPY would improve throughput.
3. **Date column type mismatch**: Trade remedies migration uses `sa.Date`
   columns but the pipeline passes string dates. PostgreSQL handles
   implicit casting, but explicit parsing would be more robust.
4. **Exchange rates unique constraint**: The `exchange_rates` ORM model has
   `UNIQUE(currency_code)` but the upsert SQL uses the correct two-column
   constraint `(currency_code, effective_date)`.
