# Sprint 2: Database Schema Separation & New Entity Tables

**Status:** In Progress
**Goal:** Create per-domain PostgreSQL schemas (logical separation within same database). Add all missing entity tables from the architecture doc. Migrate existing tables into domain schemas. All reads/writes continue to work.

## Agent Allocation

| Agent | Role | File Ownership |
|-------|------|----------------|
| `db-architect` | backend-developer | All Alembic migrations (016-022), `clearance_platform/shared/database.py`, domain `models.py` files |
| `test-validator` | test-automator | `tests/unit/test_database_schemas.py`, runs full test suite |

## Tasks

### Task 1: Create per-domain PostgreSQL schemas (db-architect)
**Files to create:** `backend/alembic/versions/016_create_domain_schemas.py`
**Acceptance:** 14 schemas created, existing tables untouched, all tests pass

### Task 2: Add Trade Intelligence entity tables (db-architect)
**Files to create:** `backend/alembic/versions/017_trade_intel_entity_tables.py`
**Tables:** `trade_intel.classifications`, `trade_intel.tariff_calculations`, `trade_intel.fta_determinations`
**Depends on:** Task 1
**Acceptance:** Tables created with proper indexes, cross-schema FK to products works

### Task 3: Add Compliance entity tables (db-architect)
**Files to create:** `backend/alembic/versions/018_compliance_entity_tables.py`
**Tables:** `compliance.compliance_results`
**Depends on:** Task 1

### Task 4: Add Adjudication, Exception, Financial entity tables (db-architect)
**Files to create:** `backend/alembic/versions/019_adjudication_exception_financial_tables.py`
**Tables:** 6 tables across 3 domains
**Depends on:** Task 1

### Task 5: Add Party Management entity tables (db-architect)
**Files to create:** `backend/alembic/versions/020_party_management_tables.py`
**Tables:** `party.parties`, `party.power_of_attorney`, `party.importers_of_record`
**Depends on:** Task 1

### Task 6: Add Declaration & Transit entity tables (db-architect)
**Files to create:** `backend/alembic/versions/021_declaration_and_transit_tables.py`
**Tables:** 5 new tables
**Depends on:** Task 1

### Task 7: Migrate existing tables into domain schemas (db-architect)
**Files to create:** `backend/alembic/versions/022_migrate_tables_to_domain_schemas.py`
**Depends on:** Tasks 1-6
**Critical:** Create public.* views for backward compatibility

### Task 8: Update shared database module (db-architect)
**Files to modify:** `clearance_platform/shared/database.py`, domain `models.py` files
**Depends on:** Task 7

## Definition of Done
- All 857+ tests pass
- All new entity tables exist in correct domain schemas
- Existing tables moved to domain schemas with public.* views
- `schema_name()` returns correct domain schema names
