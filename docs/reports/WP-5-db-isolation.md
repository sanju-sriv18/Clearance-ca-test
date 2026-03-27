# WP-5: Per-Domain PostgreSQL Schema Isolation with RBAC

**Finding:** #2 -- All domain containers connect to the same PostgreSQL database using schema prefixes only; no role-based access control enforcement.

**Status:** Complete

**Date:** 2026-02-27

---

## Summary

Enforced per-domain PostgreSQL schema isolation by ensuring all 15 domain ORM models specify their target schema via `__table_args__`, creating per-domain service roles with RBAC that restricts each role to its own schema plus the shared `public` schema, and adding cross-schema revocation to prevent any domain service from accessing another domain's data via direct SQL.

## Audit Findings (Pre-existing State)

Before this work package, the codebase already had:

1. **Schema creation in `init.sql`** -- All 15 domain schemas are created (`CREATE SCHEMA IF NOT EXISTS`) on fresh database initialization.
2. **Per-domain service roles in `init.sql`** -- 15 `svc_{schema}` roles with login credentials and grants on their own schema.
3. **Public schema access** -- All roles granted CRUD on the `public` schema for shared tables (alembic_version, processed_events).
4. **ORM model schema annotations** -- All 15 domain model files (47 model classes total) already had correct `{"schema": "<domain_schema>"}` in `__table_args__`.

### Gap Identified

**Cross-schema access was NOT revoked.** While each role was granted access to its own schema, no `REVOKE` statements prevented a role from accessing other domain schemas. PostgreSQL's default `public` USAGE grant means newly created schemas may be accessible to any authenticated role. This is the core isolation gap.

## Changes Made

### 1. `clearance_platform/shared/database.py` -- Per-Domain Session Factories

Added three new functions for microservices-mode database access:

- **`get_domain_database_url(domain)`** -- Returns a database URL with per-domain service role credentials when `MICROSERVICES_MODE=true`. Supports production password override via `SVC_{SCHEMA}_PASSWORD` env vars.
- **`create_domain_engine(domain)`** -- Creates a cached async SQLAlchemy engine with `search_path` set to `{schema},public`, ensuring unqualified table names resolve correctly.
- **`create_domain_session_factory(domain)`** -- Creates an `async_sessionmaker` bound to the domain's engine, ready for use in microservice containers.

### 2. `alembic/versions/029_enforce_domain_schema_isolation.py` -- Migration

Idempotent migration that:
- Creates all 15 domain schemas (`CREATE SCHEMA IF NOT EXISTS`)
- Creates per-domain service roles (`svc_{schema}`) if they don't exist
- Grants each role USAGE + CRUD on its own schema
- Grants all roles access to `public` schema (shared tables)
- **Revokes cross-schema access** -- Each `svc_{schema}` role is explicitly denied access to all other domain schemas

This migration brings existing databases to the same state as a fresh `init.sql` initialization.

### 3. `data/init.sql` -- Cross-Schema Revocation Block

Added a new PL/pgSQL block after the public-schema grants that iterates all 15 domain schemas and revokes each role's access to every schema except its own. This closes the isolation gap for fresh database initializations.

### 4. Integration Tests -- `tests/integration/test_db_isolation.py`

96 tests organized in three test classes:

| Class | Tests | Coverage |
|-------|-------|----------|
| `TestDomainSchemaMapping` | 17 | DOMAIN_SCHEMAS completeness, uniqueness, correctness, `get_schema_name()` |
| `TestDomainModelSchemaAlignment` | 16 | Every ORM class has `__table_args__` with correct schema |
| `TestPerDomainSessionFactory` | 63 | URL generation in monolith/microservices mode, custom passwords, session factory |

**Results:** 95 passed, 1 skipped (asyncpg driver not installed in unit test environment).

## Schema-to-Role Mapping

| Domain | Schema | Role |
|--------|--------|------|
| cargo_handling_units | cargo | svc_cargo |
| compliance | compliance | svc_compliance |
| consolidation | consolidation | svc_consolidation |
| customs_adjudication | adjudication | svc_adjudication |
| declaration_management | declaration | svc_declaration |
| document_management | document | svc_document |
| exception_management | exception | svc_exception |
| financial_settlement | financial | svc_financial |
| order_management | order_mgmt | svc_order_mgmt |
| party_management | party | svc_party |
| product_catalog | product | svc_product |
| regulatory_intelligence | regulatory | svc_regulatory |
| shipment_lifecycle | shipment | svc_shipment |
| supply_chain_disruptions | disruption | svc_disruption |
| trade_intelligence | trade_intel | svc_trade_intel |

Additionally: `svc_platform` role for cross-cutting `public` schema access.

## Files Changed

| File | Change |
|------|--------|
| `clearance_platform/shared/database.py` | Added `get_domain_database_url()`, `create_domain_engine()`, `create_domain_session_factory()` |
| `alembic/versions/029_enforce_domain_schema_isolation.py` | New migration: schema creation, role RBAC, cross-schema revocation |
| `data/init.sql` | Added cross-schema revocation PL/pgSQL block |
| `tests/integration/test_db_isolation.py` | New: 96 integration tests for schema isolation |
| `docs/reports/WP-5-db-isolation.md` | This report |

## Production Notes

- **Passwords:** Dev-only default `svc_dev_password` is used in `init.sql` and migrations. Production MUST rotate passwords via the secrets manager (see `docs/operations.md`). Per-domain password overrides use `SVC_{SCHEMA}_PASSWORD` environment variables.
- **Monolith vs Microservices:** The `MICROSERVICES_MODE` env var controls whether per-domain credentials are used. In monolith mode (current default), all domains use the shared `clearance` superuser.
- **Legacy tables:** Tables in `public` schema are untouched. The migration does NOT move existing data between schemas. Domain models already create new tables in the correct schema via `__table_args__`.
- **Downgrade:** Migration 029 downgrade restores cross-schema USAGE grants, reverting to pre-isolation state.
