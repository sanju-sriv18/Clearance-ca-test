# Sprint 1: Foundation — Domain Package Structure & Shared Infrastructure

**Status:** In Progress
**Goal:** Create the `clearance_platform/` package tree with 15 domain packages, shared infrastructure modules, event base classes, and the gateway routing module. Wire `app/main.py` to import from the new structure. No behavior changes — pure restructure.

## Agent Allocation

| Agent | Role | File Ownership |
|-------|------|----------------|
| `backend-foundation` | backend-developer | All `clearance_platform/` files, `app/main.py` modifications |
| `test-validator` | test-automator | `tests/unit/test_domain_structure.py`, runs full test suite |

## Tasks

### Task 1: Create domain package skeleton (backend-foundation)
**Files to create:** 15 domain packages + shared packages under `backend/clearance_platform/`
**Acceptance:** All 15 domain packages importable, 756 tests pass

### Task 2: Create shared infrastructure modules (backend-foundation)
**Files to create:** `clearance_platform/shared/` — event_bus, cache, database, llm, types
**Depends on:** Task 1
**Acceptance:** `DomainEventBus` publishable, all modules importable

### Task 3: Create event base classes (backend-foundation)
**Files to create:** `clearance_platform/shared/events/` — base.py, registry.py
**Depends on:** Task 1
**Acceptance:** BaseEvent round-trips JSON, EventMetadata auto-generates fields

### Task 4: Create domain module templates (backend-foundation)
**Files to create:** routes.py, service.py, events.py, models.py per domain (60 files)
**Depends on:** Tasks 1, 3
**Acceptance:** All domain modules importable

### Task 5: Create gateway + rewire main.py (backend-foundation)
**Files to create:** `clearance_platform/gateway/` — router_registry.py
**Files to modify:** `backend/app/main.py`
**Depends on:** Tasks 1, 4
**Acceptance:** Same routes respond, all tests pass, frontend works

### Task 6: Write structural validation tests (test-validator)
**Files to create:** `backend/tests/unit/test_domain_structure.py`
**Depends on:** Tasks 1-5
**Acceptance:** All structural tests pass, total test count >= 760

## Definition of Done
- All 756+ tests pass
- Frontend `npm run build` passes
- `curl http://localhost:4000/health` returns all OK
- All 15 domain packages importable with standard module files
- `app/main.py` uses gateway for router registration
