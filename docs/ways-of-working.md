# Ways of Working -- Agent Team Operations Manual

**Status:** Living Document (Single Source of Truth)
**Date:** 2026-02-09
**Author:** Simon Kissler (simon.kissler@accenture.com)
**Co-Author:** Claude Code (Anthropic)
**Scope:** All Claude Code agent teams working on the Clearance Platform
**Related:** `docs/development-standards.md`, `docs/architecture-clearance-platform.md`

---

## Table of Contents

1. [Team Structure and Roles](#1-team-structure-and-roles)
2. [Task Management](#2-task-management)
3. [File Ownership and Coordination](#3-file-ownership-and-coordination)
4. [Coding Standards](#4-coding-standards)
5. [API Contract Documentation](#5-api-contract-documentation)
6. [Testing Requirements](#6-testing-requirements)
7. [Git and Version Control](#7-git-and-version-control)
8. [Communication Protocols](#8-communication-protocols)
9. [Architecture Compliance](#9-architecture-compliance)
10. [Review and Quality Gates](#10-review-and-quality-gates)
11. [Rituals and Processes](#11-rituals-and-processes)
12. [Anti-Patterns to Avoid](#12-anti-patterns-to-avoid)

---

## 1. Team Structure and Roles

### 1.1 Leader Agent Responsibilities

The leader agent coordinates work and never writes code directly. Enable delegate mode (Shift+Tab) to enforce this. The leader:

- Decomposes sprint backlog items into discrete, parallelizable tasks
- Creates the shared task list with explicit dependencies between tasks
- Assigns file ownership boundaries before any agent starts coding
- Assigns tasks to teammates based on specialization and current workload
- Monitors progress across all agents, intervening when an agent is stuck for more than two task cycles
- Reviews completed work against acceptance criteria before marking tasks done
- Synthesizes results and communicates status to the user
- Manages agent lifecycle: spawning, redirecting, and shutting down teammates
- Resolves file ownership conflicts by reassigning work or sequencing tasks
- Runs final verification (build + full test suite) before declaring a sprint complete

The leader does NOT:
- Write code, edit files, or run tests directly
- Approve its own work or bypass quality gates
- Start implementing if teammates are slow (instruct them instead)

### 1.2 Specialist Agent Types

| Agent Type | When to Use | Typical File Ownership |
|---|---|---|
| **backend-developer** | New API endpoints, domain service logic, database migrations, Pydantic schemas | `backend/app/api/routes/`, `backend/app/api/schemas/`, `backend/app/services/`, `backend/app/engines/` |
| **frontend-developer** | React components, hooks, state management, UI styling | `frontend/src/components/`, `frontend/src/hooks/`, `frontend/src/store/`, `frontend/src/pages/` |
| **fullstack-developer** | Cross-layer features requiring coordinated frontend + backend changes | Assigned specific backend route files AND specific frontend component files (never overlapping with other agents) |
| **test-automator** | E2E test creation and maintenance, test infrastructure, test data strategies | `frontend/e2e/`, `frontend/src/test/`, `backend/tests/` |
| **simulation-developer** | Simulation actors, event bus logic, routing engine, clock management | `backend/app/simulation/` |
| **intelligence-developer** | LLM engine logic, prompt engineering, Qdrant knowledge stores, classification/compliance/tariff engines | `backend/app/engines/`, `backend/app/knowledge/` |
| **infrastructure-developer** | Docker, Caddy, deployment scripts, Terraform, CI/CD, database ops | `clearance-engine/docker-compose*.yml`, `Caddyfile`, `deploy.sh`, `infra/`, `k8s/` |
| **reviewer** | Code review, architecture review, security audit, performance analysis | Read-only; produces reports, does not edit code |
| **technical-documenter** | Platform documentation, event catalogs, API reference docs, architecture decision records, domain model documentation. Keeps living docs in sync with code reality. | `docs/`, architecture doc, event schemas, API docs. Does NOT write application code. |
| **ux-designer** | Screen layouts, interaction flows, component hierarchy, accessibility, responsive design. Reviews all UI changes for coherence and consistency before they ship. Prevents duplicative UX and ensures intelligent user experience across surfaces. | Read-only on code; produces design specs, wireframes, component inventories. Has **veto authority** on UI changes that degrade user experience. |
| **business-analyst** | Functional design adherence, acceptance criteria validation, cross-surface workflow verification, data integrity checks. Verifies that what was built matches the functional specification and that end-to-end workflows actually work from the user's perspective. **De facto product owner — accountable for the successful outcome.** | Read-only on code; produces test scenarios, acceptance reports, gap analyses. Has **authority to reject** a task as incomplete if it does not meet functional requirements, even if the code compiles and tests pass. |
| **architecture-steward** | Documents architecture decisions (ADRs), reviews all changes against `docs/architecture-clearance-platform.md`, validates domain boundaries, event contracts, and CQRS discipline. **Has authority to make architecture decisions** within the bounds of the approved architecture, and to **reject code** that violates architectural constraints. Escalates to user when a decision would change the approved architecture. | `docs/architecture-*.md`, ADR files. Edits architecture docs; does not write application code. |

### 1.3 Governance Roles (Non-Coding, Accountable)

These roles do not write application code but have authority over quality and correctness. They are essential for preventing the failure modes that plague multi-agent development: duplicative UX, architecture drift, functionality that "works" but doesn't match the functional design, and documentation that falls out of sync.

**Business Analyst (Product Owner)**
- Writes acceptance criteria for every sprint backlog item before work begins
- Reviews completed features against the functional specification — not just "does it compile" but "does it do what the user needs"
- Validates end-to-end workflows across surfaces (e.g., shipper creates order → broker sees it in queue → customs processes it → buyer sees delivery update)
- Maintains the unified product backlog and ensures feature prioritization reflects business value
- **Accountable for successful outcome**: if the sprint delivers something that doesn't match what was asked, this role failed
- Has authority to send work back for rework even if it passes code review and tests
- Reports to user when scope conflicts, ambiguous requirements, or trade-offs need resolution

**UX Designer**
- Owns the user experience across all surfaces (broker, shipper, buyer, platform)
- Reviews every UI change before it ships — not just "does it render" but "does it make sense to the user"
- Maintains a component inventory to prevent duplicative UX (e.g., two different date pickers, three different table implementations, inconsistent navigation patterns)
- Designs new screens and interaction flows when sprint backlog requires them
- **Veto authority on UI changes** that degrade user experience, introduce inconsistency, or create UX debt
- Ensures accessibility (WCAG 2.1 AA) and responsive behavior
- Produces wireframes/specs before frontend agents code, not after

**Technical Documenter**
- Keeps `docs/` in sync with code reality after every sprint
- Documents new NATS events, API endpoints, domain model changes, and configuration changes
- Maintains the event catalog as a living reference (not just the architecture doc's summary)
- Writes developer onboarding content: "how does X work" guides for each domain
- Updates `docs/development-standards.md` when patterns evolve
- Runs a documentation sync check at sprint completion: "what changed in code that is not yet reflected in docs?"

**Architecture Steward**
- Reviews all code changes against `docs/architecture-clearance-platform.md`
- Documents Architecture Decision Records (ADRs) for any decision that deviates from or extends the architecture
- Validates domain boundaries: no cross-domain database queries, correct event subjects, proper CQRS discipline
- **Has authority to make architecture decisions** within the bounds of the approved architecture (e.g., "this new endpoint should use Redis cache, not direct DB query")
- **Escalates to user** when a proposed change would alter the approved architecture (e.g., adding a new domain, changing event contracts, introducing a new infrastructure component)
- Validates that event schemas follow state-transfer pattern with full snapshots
- Ensures new code respects the jurisdiction adapter pattern for multi-jurisdiction concerns

### 1.4 Team Sizing Guide

| Work Scope | Team Size | Rationale |
|---|---|---|
| Single-surface bug fix (e.g., fix broker queue sort) | 1 agent (no team needed) | Coordination overhead exceeds benefit |
| Single-layer feature (e.g., new backend endpoint) | 1-2 agents | One implements, one reviews or writes tests |
| Cross-layer feature (e.g., new entry detail tab with API) | 2-3 agents | Backend + frontend + test-automator |
| Full sprint (multiple features across surfaces) | 5-8 agents | Implementation agents (one per surface/layer) + governance roles (BA, UX, test-automator) |
| Architecture migration or large refactor | 6-9 agents | Specialized by domain, governance roles, architecture steward, with strict file ownership |
| Research/investigation (no code changes) | 2-4 agents | Parallel hypothesis testing converges faster |

**Rule of thumb:** Add a coding agent only when it can work on a genuinely independent work stream. If tasks are sequential or touch the same files, a single agent is more effective than two agents waiting on each other.

**Governance roles are not optional for sprints.** For any sprint with 3+ coding agents, the business analyst and test-automator are mandatory. UX designer is mandatory when UI changes are involved. Architecture steward is mandatory when new domains, events, or infrastructure are introduced. Technical documenter is mandatory when the sprint adds new APIs, events, or configuration.

**Cost awareness:** Each teammate consumes its own token context window. A 5-agent team uses roughly 5x the tokens of a single session. Reserve larger teams for work that genuinely benefits from parallelism. Governance roles (BA, UX) can often share investigation time and run their reviews sequentially at sprint completion rather than consuming tokens throughout.

### 1.5 Naming Conventions

**Team names:** `{project}-{sprint-or-initiative}`
```
clearance-sprint-4
clearance-perf-audit
clearance-brazil-jurisdiction
```

**Agent names:** `{role}-{surface-or-domain}`
```
backend-broker
frontend-shipper
test-automator
fullstack-entry-detail
sim-routing
intel-classification
reviewer-security
```

Names must be descriptive enough that any agent reading a message knows the sender's specialty and scope.

---

## 2. Task Management

### 2.1 Task Decomposition

Every sprint backlog item must be decomposed into tasks before any agent starts coding. Tasks must be:

- **Self-contained:** Produces a clear deliverable (a function, a component, a test file, an API endpoint)
- **File-bounded:** Lists the exact files the agent will create or modify
- **Acceptance-criteria-driven:** Defines what "done" looks like in testable terms
- **Right-sized:** 15-60 minutes of agent work. Too small means coordination overhead dominates. Too large means too much context drift before check-in.

**Target:** 5-6 tasks per agent per sprint. This keeps agents productive and gives the leader room to reassign work if someone gets stuck.

**Decomposition example:**

Bad (too vague):
```
Task: "Add ISF filing support"
```

Good (specific and bounded):
```
Task 1: Backend ISF schema and route
  Files: backend/app/api/schemas/isf.py, backend/app/api/routes/isf.py
  Acceptance: POST /broker/isf/{shipment_id}/file returns 201 with ISF transaction number
  Depends on: nothing

Task 2: Backend ISF validation logic
  Files: backend/app/services/isf_validation.py
  Acceptance: Validates 10+2 data fields, returns structured errors for missing fields
  Depends on: Task 1

Task 3: Frontend ISF filing panel
  Files: frontend/src/components/broker/ISFPanel.tsx, frontend/src/hooks/useISF.ts
  Acceptance: Panel renders ISF form, submits via API, displays validation errors
  Depends on: Task 1 (needs schema)

Task 4: E2E test for ISF filing flow
  Files: frontend/e2e/broker/isf.spec.ts
  Acceptance: Test queries for ocean shipment, opens ISF panel, files ISF, verifies success
  Depends on: Task 2, Task 3
```

### 2.2 Task Lifecycle

Tasks flow through three states in the shared task list:

```
pending --> in_progress --> completed
```

- **pending:** Created by the leader, not yet claimed. If a task has unresolved dependencies, it cannot be claimed.
- **in_progress:** Claimed by a specific agent. Only one agent works on a task at a time.
- **completed:** Agent has finished, code compiles, and unit/component tests pass for the changed files.

**Rules:**
- An agent must mark a task `completed` when done. Failing to do so blocks dependent tasks.
- If a task is blocked for more than 10 minutes, the agent must message the leader with the blocker.
- The leader can reassign a task if the assigned agent is stuck or overloaded.

### 2.3 Dependency Management

Dependencies are declared at task creation time. The shared task list enforces them: an agent cannot claim a task whose dependencies are not completed.

**Dependency patterns:**
- **Schema first:** Backend schemas before frontend types or tests
- **Route first:** Backend routes before frontend API calls
- **Data first:** Test data setup before test execution tasks
- **Independent surfaces:** Broker frontend and shipper frontend can proceed in parallel since they touch different files

**When tasks share a dependency but not files:**
Both tasks depend on the schema task. Once the schema task completes, both can proceed in parallel because they own different files.

### 2.4 Sub-tasks vs. Inline Work

Create a sub-task when:
- The work requires touching a different set of files than the parent task owns
- The work would benefit from a different agent's specialization
- The work is independently verifiable and could be reviewed separately

Handle inline when:
- It is a small fix within the files already owned by the current task
- It takes less than 5 minutes
- It does not affect any other agent's work

---

## 3. File Ownership and Coordination

### 3.1 The Cardinal Rule

**Two agents must never edit the same file at the same time.** This is the single most important coordination rule. Violations cause silent overwrites, merge conflicts, and wasted work. There are no exceptions.

### 3.2 File Ownership Assignment

Before any agent starts coding, the leader must produce a **File Ownership Map** in the task list. Every file that will be modified in the sprint must be assigned to exactly one agent.

Example:
```
backend-broker owns:
  backend/app/api/routes/broker.py
  backend/app/api/schemas/broker.py
  backend/app/services/broker_intelligence.py

frontend-broker owns:
  frontend/src/components/broker/QueuePanel.tsx
  frontend/src/components/broker/EntryDetail.tsx
  frontend/src/hooks/useBrokerQueue.ts
  frontend/src/store/brokerStore.ts

test-automator owns:
  frontend/e2e/broker/queue.spec.ts
  frontend/e2e/broker/entry-detail.spec.ts
  backend/tests/test_broker.py

fullstack-shipper owns:
  backend/app/api/routes/orders.py
  frontend/src/components/shipper/OrderPanel.tsx
  frontend/src/hooks/useOrders.ts
```

### 3.3 Shared File Protocols

Some files are naturally shared: `client.ts`, `types.ts`, `main.py`, router registrations, store files. Handle these with one of three strategies:

**Strategy 1: Single owner (preferred).** Assign the shared file to one agent. Other agents who need changes message that agent with the specific edit needed. The owner integrates the change.

**Strategy 2: Sequential access.** Tasks that touch the shared file are sequenced with explicit dependencies. Agent A completes their edits, then Agent B's task unblocks and they make their edits.

**Strategy 3: Additive-only convention.** For files that are append-only in nature (like route registrations or type definitions), agents may add to the end of specific sections. This ONLY works when agents are adding new items to a list, never modifying existing items. The leader must explicitly approve this convention per file.

**For this project, the following files require Strategy 1 or Strategy 2:**
- `frontend/src/api/client.ts` -- All API client functions and transforms (single owner per sprint)
- `frontend/src/types.ts` -- All shared TypeScript types (single owner per sprint)
- `backend/app/api/routes/__init__.py` -- Route registration (single owner)
- `backend/app/main.py` -- Application bootstrap (single owner)
- `frontend/src/App.tsx` -- Route and layout registration (single owner)

### 3.4 Conflict Resolution

If two tasks unexpectedly need to touch the same file:
1. The first agent to identify the conflict messages the leader immediately
2. The leader decides: either (a) reassign one task to the file owner, (b) sequence the tasks, or (c) split the file so each agent owns a different piece
3. No agent proceeds with the conflicting edit until the leader resolves it
4. If the leader is unresponsive, the later agent (by task creation order) waits

### 3.5 Merge Strategy When Work Converges

At sprint completion, the leader verifies all changes integrate cleanly:
1. All agents commit to the same branch (main, unless otherwise directed)
2. The test-automator runs the full suite after all agents complete
3. If integration issues arise (e.g., type mismatches at API boundaries), the leader assigns fix tasks with clear file ownership
4. No one pushes until the full suite passes

---

## 4. Coding Standards

### 4.1 Python Backend Standards

**Framework:** FastAPI with async everywhere. SQLAlchemy for ORM. Pydantic v2 for schemas.

**File organization:**
```
backend/app/
  api/
    routes/{domain}.py       # Route handlers (thin -- delegate to services)
    schemas/{domain}.py      # Pydantic request/response models
  services/{domain}.py       # Business logic
  engines/{engine_name}/     # Intelligence engines (LLM, computation)
  simulation/                # Simulation platform (separate concern)
  knowledge/                 # Reference data models and repositories
```

**Async discipline:**
- All route handlers are `async def`
- All database operations use `async` sessions
- All HTTP calls use `httpx.AsyncClient`, never `requests`
- All Redis operations use `aioredis` / async Redis client
- CPU-bound work (heavy computation) uses `asyncio.to_thread()` or a process pool

**Type hints:** Mandatory on all function signatures. No `Any` without a comment explaining why.

```python
# CORRECT
async def get_entry_detail(entry_id: UUID, db: AsyncSession) -> EntryDetailResponse:
    ...

# WRONG
async def get_entry_detail(entry_id, db):
    ...
```

**Pydantic models:**
- All API request bodies use Pydantic `BaseModel` subclasses
- All API responses use Pydantic `BaseModel` subclasses
- Validators use `@field_validator` or `@model_validator`
- Use `model_config = ConfigDict(from_attributes=True)` for ORM integration
- snake_case field names always (camelCase transformation happens in the frontend)

```python
class EntryDetailResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    entry_number: str
    filing_status: FilingStatus
    checklist_state: dict[str, Any]
    summary_data: dict[str, Any] | None = None
```

**Error handling:**
- Use FastAPI's `HTTPException` for expected errors (400, 404, 409, 422)
- Use custom exception classes for domain-specific errors
- Never catch `Exception` broadly and return 200 with an error message
- Always include a meaningful `detail` string

```python
# CORRECT
raise HTTPException(status_code=404, detail=f"Entry {entry_id} not found")

# WRONG
return {"error": "not found", "status": "fail"}
```

**Route handler pattern:** Handlers are thin. They validate input, call a service, and return the response. Business logic lives in services.

```python
@router.post("/broker/entries/{entry_id}/approve")
async def approve_entry(
    entry_id: UUID,
    request: ApproveEntryRequest,
    db: AsyncSession = Depends(get_db),
) -> EntryDetailResponse:
    entry = await broker_service.approve_entry(db, entry_id, request)
    return EntryDetailResponse.model_validate(entry)
```

**Import ordering:**
```python
# 1. Standard library
import asyncio
from datetime import datetime, timezone
from uuid import UUID

# 2. Third-party
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession

# 3. Local application
from app.api.schemas.broker import EntryDetailResponse
from app.services.broker import BrokerService
from app.services.database import get_db
```

### 4.2 TypeScript / React Frontend Standards

**Framework:** Vite + React 18+ with TypeScript strict mode. Zustand for state management.

**File organization:**
```
frontend/src/
  api/client.ts              # ALL API calls and snake_case->camelCase transforms
  types.ts                   # ALL shared TypeScript interfaces
  components/{surface}/      # Components organized by user surface (broker/, shipper/, buyer/, platform/)
  hooks/                     # Custom React hooks
  store/                     # Zustand stores
  pages/                     # Page-level components (route targets)
  lib/                       # Pure utility functions (no React, no side effects)
  config/                    # Configuration constants
```

**API URL pattern (mandatory, no exceptions):**
```typescript
const API_BASE = import.meta.env.VITE_API_URL || "/api";

// Paths never include /api/ -- it is already in the base URL
fetch(`${API_BASE}/broker/dashboard`);
```

**snake_case to camelCase transformation:** All transformations happen in `client.ts`. Components never see snake_case.

```typescript
// In client.ts -- CORRECT
function transformBrokerEntry(raw: Record<string, unknown>): BrokerEntry {
  return {
    entryNumber: raw.entry_number as string,
    filingStatus: raw.filing_status as string,
    checklistState: raw.checklist_state as ChecklistState,
  };
}

// In a component -- WRONG
const data = await res.json();
setEntry({ entryNumber: data.entry_number }); // Don't transform in components
```

**Component patterns:**
- Functional components only (no class components)
- Props interfaces defined inline or in `types.ts` (never `any`)
- `useEffect` must have correct dependency arrays (no eslint-disable for exhaustive-deps)
- Error boundaries wrap major page sections
- Loading states handled explicitly (not implicit null checks)

```tsx
// CORRECT -- explicit loading and error states
function EntryDetail({ entryId }: { entryId: string }) {
  const { data, isLoading, error } = useEntryDetail(entryId);

  if (isLoading) return <LoadingSkeleton />;
  if (error) return <ErrorCard message={error.message} />;
  if (!data) return <EmptyState />;

  return <EntryDetailContent entry={data} />;
}
```

**State management (Zustand):**
- One store per domain surface (brokerStore, shipperStore, cartStore)
- Stores own data fetching and caching logic
- Components subscribe to specific slices, not the whole store
- No derived state in stores that can be computed in components

**Toast notifications:** Use the project's `useToast` hook for user-facing messages. Never use `alert()` or `console.log` for user feedback.

### 4.3 Naming Conventions

| Entity | Convention | Example |
|---|---|---|
| Python files | snake_case | `broker_intelligence.py` |
| Python classes | PascalCase | `EntryDetailResponse` |
| Python functions | snake_case | `get_entry_detail` |
| Python constants | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| TypeScript files | camelCase | `brokerStore.ts` |
| React components | PascalCase (file and component) | `EntryDetail.tsx` / `EntryDetail` |
| TypeScript interfaces | PascalCase | `BrokerEntry` |
| TypeScript functions | camelCase | `fetchBrokerQueue` |
| CSS classes | kebab-case | `entry-detail-panel` |
| API routes | kebab-case | `/broker/entries/{id}/checklist-override` |
| NATS subjects | dot-separated kebab-case | `clearance.declaration.checklist-updated` |
| Redis keys | colon-separated | `broker:queue:{broker_id}` |
| Database tables | snake_case, plural | `entry_filings` |
| Database columns | snake_case | `filing_status` |
| Environment variables | UPPER_SNAKE_CASE | `VITE_API_URL` |
| E2E test files | kebab-case by surface/function | `broker-queue.spec.ts` |

### 4.4 Cross-Language Naming at the API Boundary

Python uses `snake_case` (PEP 8) and TypeScript uses `camelCase` (community convention). These conventions differ, and the mismatch at the API boundary is a known source of bugs. The project addresses this with a **single transformation layer** in `client.ts` — but this only works if everyone follows the rule strictly.

**The contract:**

```
PostgreSQL columns:  snake_case    (filing_status, entry_number, checklist_state)
         ↓
Python/Pydantic:     snake_case    (filing_status, entry_number, checklist_state)
         ↓
JSON over HTTP:      snake_case    (filing_status, entry_number, checklist_state)
         ↓
client.ts transform: snake → camel (filingStatus, entryNumber, checklistState)
         ↓
React components:    camelCase     (filingStatus, entryNumber, checklistState)
```

**Rules:**
1. **Backend always emits `snake_case` in JSON.** Pydantic does this by default. Never override with `alias` or `by_alias` to produce camelCase from the backend.
2. **The transformation happens in ONE place: `client.ts`.** Every API response goes through a transform function that maps snake_case keys to camelCase. Every API request transform maps camelCase back to snake_case.
3. **React components NEVER see snake_case.** If a component references `entry.filing_status`, that is a bug — it should be `entry.filingStatus`.
4. **TypeScript interfaces use camelCase.** The TypeScript `BrokerEntry` interface has `filingStatus: string`, not `filing_status: string`.
5. **NATS event subjects use `kebab-case`** (`clearance.declaration.checklist-updated`), which is a third convention — but these are not field names, they are topic identifiers.
6. **Redis keys use `colon-separated snake_case`** (`broker:queue:{broker_id}`).

**Why not just use camelCase everywhere?** Because Python's ecosystem (Pydantic, SQLAlchemy, FastAPI, pytest) is built around snake_case. Fighting the language convention creates more bugs than the transformation layer. The same is true for TypeScript — using snake_case in React components would look foreign and confuse TypeScript developers.

**Common mistakes and how to catch them:**
- Backend returns `entryNumber` (camelCase) instead of `entry_number` → the transform layer breaks because it expects snake_case
- Frontend directly accesses `response.data.entry_number` without going through the transform → works by accident but is fragile
- New endpoint skips the transform function in `client.ts` → snake_case leaks into components
- **Automated check:** the test-automator should verify that TypeScript interfaces have no underscore-containing field names (except when intentionally passing raw data to the backend)

### 4.5 Error Handling Patterns

**Backend -- structured error responses:**
```python
# Domain-specific exceptions
class EntryNotFoundError(Exception):
    def __init__(self, entry_id: UUID):
        self.entry_id = entry_id
        super().__init__(f"Entry {entry_id} not found")

# Exception handler registered at app level
@app.exception_handler(EntryNotFoundError)
async def entry_not_found_handler(request: Request, exc: EntryNotFoundError):
    return JSONResponse(status_code=404, content={"detail": str(exc)})
```

**Frontend -- error boundaries + toast:**
```tsx
// Page-level error boundary
<ErrorBoundary fallback={<ErrorPage />}>
  <BrokerDashboard />
</ErrorBoundary>

// API call error handling
try {
  const result = await approveBrokerEntry(entryId);
  toast.success("Entry approved");
} catch (err) {
  toast.error(err instanceof ApiError ? err.message : "Failed to approve entry");
}
```

**Never silently swallow errors:**
```typescript
// WRONG -- silent failure
try { await submitEntry(id); } catch { /* ignore */ }

// CORRECT -- handle or propagate
try {
  await submitEntry(id);
} catch (err) {
  console.error("Entry submission failed:", err);
  toast.error("Submission failed. Please try again.");
}
```

### 4.6 When to Use JSONB vs. Separate Tables

**Use JSONB when:**
- The data is a configuration blob or metadata that varies by record (e.g., `checklist_state`, `jurisdiction_config`)
- The data is queried as a whole, not filtered or joined by individual fields
- The schema varies by jurisdiction or context and cannot be normalized efficiently
- The data is a non-authoritative convenience copy from another domain

**Use a separate table when:**
- The data grows over time (append-only patterns) -- JSONB rewrites the entire column on each update
- The data needs indexing or filtering on individual fields
- The data has its own lifecycle (created, updated, deleted independently)
- The data is referenced by foreign keys from other tables

**Project examples:**
- `checklist_state` on `entry_filings` -- JSONB (varies by jurisdiction, queried as whole)
- `hu_transit_events` -- separate table (append-only movement history, extracted from JSONB per architecture v5)
- `cage_status` on `handling_units` -- JSONB (physical detention detail, queried as whole)
- `compliance_results` -- separate table (has own lifecycle, queried independently)

### 4.7 Event Schema Standards

Every NATS event must include standard metadata:

```python
class EventMetadata(BaseModel):
    event_id: UUID = Field(default_factory=uuid4)
    event_type: str                    # e.g., "ClassificationProduced"
    timestamp: datetime                # ISO 8601
    version: int = 1                   # Schema version
    source_domain: str                 # e.g., "trade_intelligence"
    correlation_id: UUID | None = None # Trace across domain boundaries

class ClassificationProducedEvent(BaseModel):
    metadata: EventMetadata
    payload: ClassificationPayload     # Full state snapshot (state-transfer pattern)
```

**Rules:**
- Events carry **full state snapshots**, not deltas. Consumers never need to query the producer.
- New optional fields are backward compatible. Old consumers ignore unknown fields.
- Removing or renaming fields is a breaking change requiring a version bump and downstream updates.
- Event schemas are defined in each domain's `events.py` and exported via `__init__.py` (public API).

---

## 5. API Contract Documentation

### 5.1 Mandatory Documentation for Every Endpoint

Every new API endpoint must have the following documented before the implementation is considered complete:

| Element | Location | Example |
|---|---|---|
| **Pydantic request model** | `backend/app/api/schemas/{domain}.py` | `class ApproveEntryRequest(BaseModel)` |
| **Pydantic response model** | `backend/app/api/schemas/{domain}.py` | `class EntryDetailResponse(BaseModel)` |
| **Error codes and conditions** | Docstring on route handler | `Raises: 404 if entry not found, 409 if already approved` |
| **Rate limit** | Comment on route decorator | `# Rate: 50000/min (default)` — infrastructure present, limits non-restrictive per policy |
| **Authentication** | Route dependency | `Depends(get_current_user)` or noted as public |
| **Frontend type** | `frontend/src/types.ts` | `interface BrokerEntry { ... }` |
| **Client function** | `frontend/src/api/client.ts` | `export async function approveBrokerEntry(...)` |
| **Transform function** | `frontend/src/api/client.ts` | `function transformBrokerEntry(raw): BrokerEntry` |

### 5.2 OpenAPI Annotations

FastAPI generates OpenAPI docs automatically from Pydantic models. Ensure:
- Route handlers have a descriptive docstring (becomes the endpoint description)
- Response models use `response_model` parameter on the decorator
- Path parameters have `Path(description="...")` annotations
- Query parameters have `Query(description="...", example=...)` annotations
- Tags group endpoints by domain

```python
@router.get(
    "/broker/entries/{entry_id}",
    response_model=EntryDetailResponse,
    tags=["broker"],
    summary="Get entry filing detail",
)
async def get_entry_detail(
    entry_id: UUID = Path(description="The entry filing ID"),
    db: AsyncSession = Depends(get_db),
) -> EntryDetailResponse:
    """Retrieve full detail for a broker's entry filing, including checklist state,
    classification data, tariff breakdown, and compliance flags."""
    ...
```

### 5.3 Frontend-Backend Contract Alignment Process

When an API change is needed:

1. **Backend agent** updates the Pydantic schemas first and communicates the new shape to the frontend agent
2. **Frontend agent** updates `types.ts` and the transform function in `client.ts` to match
3. **Test-automator** verifies the contract with an integration test
4. The leader verifies that the Pydantic model fields and the TypeScript interface fields match (modulo camelCase transformation)

### 5.4 Breaking Change Policy

A breaking change is any modification that would cause existing frontend code to fail:
- Removing a response field
- Changing a field type
- Renaming a field
- Changing an endpoint path
- Changing required vs. optional status of a request field

**Breaking changes require:**
1. Explicit approval from the user (project owner)
2. Simultaneous update of backend, frontend types, client transforms, and tests
3. A single agent owns the full vertical slice (or tasks are sequenced with dependencies)

### 5.5 Versioning Strategy

This project currently uses a single API version (no `/v1/` prefix). If versioning becomes necessary:
- New versions get a path prefix: `/v2/broker/queue`
- Old versions remain operational until all consumers migrate
- The frontend always targets the latest version
- Deprecation is communicated via response headers: `Deprecation: true`

---

## 6. Testing Requirements

### 6.1 Zero Tolerance for Failing Tests

**ALL tests must pass. No exceptions. No "unrelated" failures.**

- If you run tests and something fails, you fix it, even if you did not cause it
- Never dismiss a failure as "pre-existing" or "unrelated"
- Never merge or push with failing tests
- If a test is flaky, fix the test to be deterministic or remove it with leader approval
- Tests that depend on LLM output must accept reasonable variation (e.g., HIGH or MEDIUM confidence)

This is non-negotiable. We do not ship broken code.

### 6.2 Unit Test Requirements

**Backend (pytest):**
- Every new service function must have unit tests
- Every new engine must have unit tests for core logic paths
- Tests use fixtures, not real database/Redis connections (unless marked as integration)
- Async tests use `pytest-asyncio`
- Minimum coverage: test the happy path, one error path, and one edge case per function

**Frontend (Vitest):**
- Every new hook must have unit tests
- Transform functions in `client.ts` must have tests verifying snake_case to camelCase mapping
- Component tests use React Testing Library, never Enzyme
- Mock API calls with MSW (Mock Service Worker), already configured in `frontend/src/test/mocks/`

### 6.3 E2E Test Organization

**E2E tests are organized by surface and function, NEVER by sprint or batch.**

```
frontend/e2e/
  broker/
    queue.spec.ts           # Broker queue listing, filtering, claiming
    entry-detail.spec.ts    # Entry detail view, checklist, approval
    isf.spec.ts             # ISF filing flow
    cbp-responses.spec.ts   # CF-28/CF-29 response workflow
  shipper/
    orders.spec.ts          # Order creation and management
    products.spec.ts        # Product catalog management
    tracking.spec.ts        # Shipment tracking
  buyer/
    checkout.spec.ts        # Buyer checkout and tariff preview
  platform/
    simulation.spec.ts      # Simulation controls and dashboard
    consolidation.spec.ts   # Consolidation management
  cross-surface.spec.ts     # Flows spanning multiple surfaces
```

When a sprint adds or changes a feature, tests go into the appropriate surface/function file. Existing tests must be updated when behavior changes. **Never create a file named after a sprint** (e.g., `sprint3-batch2.spec.ts`).

### 6.4 E2E Test Data Strategy

Since the platform runs a live simulation, E2E tests must query the API to find records in the required state before testing. **Never assume the first record matches.**

```typescript
// CORRECT -- query for a record in the right state
test("broker can approve a pending entry", async ({ page }) => {
  // Find an entry in 'pending_broker_approval' state
  const response = await page.request.get(`${API_BASE}/broker/queue`);
  const queue = await response.json();
  const pendingEntry = queue.entries.find(
    (e: any) => e.filing_status === "pending_broker_approval"
  );
  expect(pendingEntry).toBeTruthy();

  // Now test the approval flow with this specific entry
  await page.goto(`/broker/entries/${pendingEntry.id}`);
  ...
});

// WRONG -- assuming the first entry works
test("broker can approve entry", async ({ page }) => {
  await page.goto("/broker/queue");
  await page.click("tr:first-child"); // Might not be in the right state
  ...
});
```

**Prefer records assigned to the human broker persona** -- the simulation broker has a known license number and predictable behavior.

### 6.5 The Single Test-Runner Agent Rule

Never have multiple agents editing test files simultaneously. The workflow:

1. **One dedicated test-automator agent** runs the full suite and analyzes failures (root cause, affected component, suggested fix)
2. The test-automator reports analyzed failures to the agents responsible for the functionality
3. The responsible agents fix their own code/components (not the test files)
4. The test-automator does the final clean run to verify all fixes
5. No agent other than the test-automator touches E2E test files

**Exception:** Backend unit tests (`backend/tests/`) can be owned by the backend agent who wrote the code being tested, since those tests are tightly coupled to the implementation and unlikely to conflict with E2E tests.

### 6.6 Performance Testing

**No endpoint may take more than 5 seconds except LLM inference calls.**

- If an endpoint consistently takes more than 5 seconds, that is a performance bug requiring root cause analysis
- Common causes: slow queries, connection pool exhaustion, lock contention, N+1 queries, missing indexes
- **Increasing timeouts is NOT a fix.** It masks the problem. Find and fix the root cause.
- LLM inference calls (classification, analysis, chat) are exempt because latency is inherently variable
- The E2E test suite should be checked periodically for steps taking more than 5 seconds (see `/tmp/e2e-full-run.txt`)

### 6.7 Test-Before-Commit Gate

Before any commit:
1. `npm run build` (frontend) -- TypeScript must compile without errors
2. `pytest` (backend) -- All backend tests must pass
3. `npm run test` (frontend unit) -- All frontend unit tests must pass
4. E2E tests must pass (run by the test-automator before sprint completion)

An agent that cannot get all tests passing must not commit. It must report the failure to the leader and work with the responsible agent to resolve it.

---

## 7. Git and Version Control

### 7.1 Branch Strategy

This project works on `main` by default. The user will direct branching when needed.

When branches are used:
```
main                         # Production-ready code
feature/{short-description}  # Feature branches (e.g., feature/isf-filing)
fix/{short-description}      # Bug fix branches (e.g., fix/broker-queue-sort)
```

### 7.2 Commit Message Format

```
{type}: {concise description}

{optional body explaining what changed and why}
```

Types:
- `feat` -- New feature or capability
- `fix` -- Bug fix
- `refactor` -- Code restructuring without behavior change
- `test` -- Adding or updating tests
- `docs` -- Documentation only
- `perf` -- Performance improvement
- `chore` -- Build, tooling, dependency updates

Examples:
```
feat: Add ISF filing endpoint and validation

fix: Broker queue sort order reversed on status column

test: Add E2E tests for entry detail checklist override

refactor: Extract tariff calculation into separate service module
```

### 7.3 Commit Timing

- Commit when the user explicitly asks
- Commit when a logical unit of work is complete and all tests pass
- Never commit partial work that breaks the build or tests
- Never commit with the intention of "fixing it in the next commit"

### 7.4 Push Rules

- **Never push with failing tests.** Zero tolerance.
- **Never push with skipped tests.** A skipped test is a bug that needs fixing or explicit removal.
- **Never force push to main.** Ever.
- The leader verifies test status before authorizing a push

---

## 8. Communication Protocols

### 8.1 When to Use SendMessage vs. Broadcast

| Scenario | Method | Rationale |
|---|---|---|
| Reporting task completion | SendMessage to leader | Only the leader needs to know |
| Reporting a blocker | SendMessage to leader | Leader decides how to resolve |
| Requesting a change from another agent's owned file | SendMessage to that specific agent | Direct request to the file owner |
| Sharing API schema that multiple agents depend on | Broadcast | All agents need the updated contract |
| Sprint completion announcement | Broadcast | Everyone should know |
| Asking for clarification on a task | SendMessage to leader | Task assignment question |

**Broadcast sparingly.** Each broadcast costs tokens across all agents. Only broadcast information that genuinely affects everyone.

### 8.2 Status Update Format

When reporting to the leader, use this structure:

```
STATUS: {task-name}
State: {in_progress | completed | blocked}
Files changed: {list of files modified}
Tests: {passing | failing (details)}
Blocker: {none | description of blocker}
Next: {what I will work on next}
```

### 8.3 Error and Blocker Reporting

When blocked, report to the leader immediately with:

```
BLOCKED: {task-name}
Issue: {what is wrong}
Root cause: {what I think caused it, or "unknown"}
Files involved: {which files are affected}
Attempted: {what I already tried}
Need: {what I need to unblock -- a file edit from another agent, a schema decision, user input}
```

### 8.4 When to Ask the User vs. Decide Autonomously

**Ask the user when:**
- The requirement is ambiguous and multiple valid interpretations exist
- A breaking change to an existing API is needed
- A design decision affects the product's user experience
- The scope of the task needs to expand beyond what was requested
- A third-party dependency or infrastructure change is required
- You need to delete or remove existing functionality

**Decide autonomously when:**
- The implementation approach is a standard pattern documented in this file
- The choice is between two equivalent technical approaches with no user-visible difference
- The fix is clearly a bug with an obvious correct solution
- The code follows an existing pattern already established in the codebase

### 8.5 Shutdown Protocol

When the user or leader sends a shutdown request:
1. **Honor it immediately.** Do not finish "just one more thing."
2. Save or commit any completed work if the leader directs it
3. Report current status (what was done, what remains)
4. Exit cleanly

---

## 9. Architecture Compliance

The architecture document at `docs/architecture-clearance-platform.md` is the definitive reference. These rules enforce it.

### 9.1 Domain Boundary Enforcement

**Import rules (Python backend):**

| From | To | Allowed? |
|---|---|---|
| Domain A internals | Domain A internals | Yes |
| Domain A | Domain B's `__init__.py` (events only) | Yes |
| Domain A | Domain B's internal modules | **NO** |
| Domain A | `shared/` (infrastructure) | Yes |
| Domain A | Domain B's database tables | **NO, never** |

```python
# CORRECT -- importing an event schema from another domain
from domains.trade_intelligence import ClassificationProducedEvent

# WRONG -- importing internal service logic from another domain
from domains.trade_intelligence.service import classify_product

# WRONG -- querying another domain's table directly
db.query(TradeIntelligence.classifications).filter(...)
```

**Enforcement:** Use `import-linter` rules in `pyproject.toml`. The CI pipeline (or pre-commit hook) fails on violations.

### 9.2 Event Schema Standards

All events follow the state-transfer pattern defined in the architecture:

1. **Events carry full state snapshots.** A `ClassificationProduced` event contains the complete `Classification` object, not just the changed fields.
2. **Events include standard metadata:** `event_id`, `event_type`, `timestamp`, `version`, `source_domain`, `correlation_id`.
3. **Backward compatibility is mandatory.** Adding new optional fields is safe. Removing, renaming, or changing the type of existing fields is a breaking change.
4. **Version field is incremented** on breaking changes. Consumers check the version and handle gracefully.
5. **NATS subject follows the hierarchy** defined in architecture Section 4.1. New subjects must fit the existing pattern: `clearance.{domain}.{event-type}`.

### 9.3 CQRS Discipline

The architecture uses a strict CQRS split:

- **Writes** go through domain services, which apply business logic, persist to PostgreSQL, and emit events
- **Reads** come from Redis state cache, which is a pre-computed read model maintained by the cache projector subscribing to NATS events
- **UI** subscribes to NATS (via SSE) for real-time updates

**Rules:**
- GET endpoints query Redis, never PostgreSQL directly (except for admin/debug endpoints explicitly marked as such)
- POST/PUT/DELETE endpoints go through domain services, never write to Redis directly
- The cache projector is the only component that writes to Redis based on events
- If Redis is stale, the fix is in the projector, not in adding a PostgreSQL fallback to the GET handler

### 9.4 No Cross-Domain Database Queries

Domains access each other's data through exactly three mechanisms:

1. **Events** carry full state snapshots. Subscribe to get another domain's data.
2. **Redis cache** has denormalized views. UI queries Redis for the full picture.
3. **API calls** for on-demand needs. Never direct cross-domain DB queries.

```python
# WRONG -- cross-domain database query
from domains.trade_intelligence.models import Classification
classification = await db.execute(
    select(Classification).where(Classification.product_id == product_id)
)

# CORRECT -- read from Redis cache (populated by events)
classification_data = await redis.hgetall(f"classification:{product_id}")

# CORRECT -- subscribe to events and store locally
@event_handler("clearance.trade-intel.classification.produced")
async def on_classification_produced(event: ClassificationProducedEvent):
    # Store locally in declaration_management schema
    await update_declaration_checklist(event.payload)
```

### 9.5 State-Transfer Event Format

Every state-transfer event includes the complete state of the changed entity. Consumers can reconstruct the entity from any single event without history.

```python
# CORRECT -- full state snapshot
class HUStatusChangedEvent(BaseModel):
    metadata: EventMetadata
    payload: HandlingUnitState  # Complete HU state, not just the changed field

# WRONG -- delta event
class HUStatusChangedEvent(BaseModel):
    metadata: EventMetadata
    hu_id: UUID
    old_status: str
    new_status: str  # Consumer would need to query for the rest of the state
```

---

## 10. Review and Quality Gates

### 10.1 Code Review Checklist

The leader (or designated reviewer agent) checks every completed task against:

**General:**
- [ ] Code compiles without errors or warnings
- [ ] All existing tests still pass (not just the new ones)
- [ ] No hardcoded URLs, ports, or localhost references
- [ ] No new environment variables invented (use existing `VITE_API_URL`)
- [ ] Follows naming conventions from Section 4.3
- [ ] Error handling is explicit (no silent catch blocks)
- [ ] No `console.log` left in production code (use proper logging)
- [ ] No `# TODO` or `// FIXME` without a linked task

**Backend:**
- [ ] Route handlers are thin (business logic in services)
- [ ] Pydantic models defined for all request/response bodies
- [ ] Async used consistently (no sync calls in async context)
- [ ] Type hints on all function signatures
- [ ] No cross-domain imports beyond `__init__.py` event schemas
- [ ] No direct cross-domain database queries
- [ ] Rate limit decorators present on LLM endpoints (limits set high — do NOT lower without user approval)

**Frontend:**
- [ ] API calls use `VITE_API_URL || "/api"` pattern exclusively
- [ ] snake_case to camelCase transforms in `client.ts`, not in components
- [ ] TypeScript types match backend Pydantic models
- [ ] Loading and error states handled explicitly
- [ ] No `any` types without justification comment
- [ ] Components have `data-testid` attributes for E2E targeting

**Tests:**
- [ ] New functionality has corresponding tests
- [ ] E2E tests organized by surface/function
- [ ] E2E tests query for state-appropriate records (no hardcoded IDs)
- [ ] No increased timeouts without root cause justification

### 10.2 Architecture Review Triggers

The following changes require architecture review by the **architecture steward** (and escalation to the user for changes that alter the approved architecture) before implementation:

- Adding a new domain
- Adding a new NATS event subject
- Changing an existing event schema (breaking change)
- Adding a new database table
- Introducing a cross-domain dependency
- Changing the CQRS read/write split for an endpoint
- Adding a new infrastructure component (container, service, external dependency)
- Modifying rate limits
- Changing authentication or authorization logic

### 10.3 Pre-Commit Checklist for Agents

Before marking a task as completed, every agent must verify:

1. `npm run build` passes (if frontend files were changed)
2. `pytest` passes (if backend files were changed)
3. `npm run test` passes (if frontend test files were changed)
4. No lint errors in changed files
5. No `console.log` or `print()` debug statements left in code
6. All acceptance criteria for the task are met
7. File ownership was respected (only files assigned to this agent were modified)

### 10.4 Definition of Done

A task is "done" when:

1. All acceptance criteria are satisfied
2. Code compiles and all tests pass
3. Code follows the standards in this document
4. The leader has reviewed and approved (or the task is low-risk and pre-approved)
5. No regressions introduced in existing functionality

A sprint is "done" when:

1. All tasks are completed
2. The full E2E suite passes (run by the test-automator)
3. Build succeeds for both frontend and backend
4. **Business analyst has validated** all features against acceptance criteria
5. **UX designer has approved** all UI changes (when UI work was done)
6. **Architecture steward has verified** domain boundaries and event contracts (when architecture-touching work was done)
7. **Technical documenter has synced** all documentation with code changes
8. The leader has verified integration across all agents' work
9. No known failing tests, skipped tests, or timeout workarounds

---

## 11. Rituals and Processes

### 11.1 Sprint Planning

When the user provides a sprint backlog:

1. **Leader reads the backlog** and identifies themes, surfaces, and dependencies
2. **Leader decomposes** each backlog item into tasks per Section 2.1
3. **Leader assigns file ownership** per Section 3.2
4. **Leader creates the shared task list** with dependencies between tasks
5. **Leader spawns teammates** with appropriate roles per Section 1.2
6. **Leader communicates the plan** to all agents via broadcast, including:
   - Each agent's assigned tasks
   - File ownership boundaries
   - Key dependencies and sequencing constraints
   - Definition of done for the sprint
7. **Leader enables delegate mode** (Shift+Tab) to focus on coordination

### 11.2 Mid-Sprint Coordination

The leader monitors progress and intervenes when:

- An agent has been stuck on a single task for more than 20 minutes without progress
- A dependency is not being completed on time, blocking other agents
- An agent reports a blocker that requires file ownership reassignment
- Test failures indicate integration issues between agents' work
- Scope creep is detected (an agent is doing more than the task requires)

**Rebalancing triggers:**
- One agent has finished all tasks while others are still working: assign unfinished tasks
- A task turns out to be much larger than estimated: split it and redistribute
- A new blocker is discovered that requires a different specialization

### 11.3 Sprint Completion

1. **All agents report completion** of their tasks
2. **Test-automator runs the full suite:** `pytest` + `npm run test` + `npm run test:e2e`
3. **Business analyst validates** that completed features match acceptance criteria and end-to-end workflows work correctly — not just "code works" but "feature works as designed"
4. **UX designer reviews** all UI changes for consistency, coherence, and absence of duplicative patterns
5. **Architecture steward verifies** domain boundaries, event contracts, and CQRS discipline in new code
6. **Technical documenter syncs** all docs with code changes (new events, endpoints, configuration)
7. **Leader reviews** integration points (API contracts, type alignment, state management)
8. **If issues found:** leader assigns targeted fix tasks with clear file ownership
9. **Fix cycle:** fix -> retest -> verify (maximum 2 rounds before escalating to user)
10. **Leader reports to user:** what was delivered, what tests pass, any remaining items
11. **Cleanup:** leader shuts down teammates, cleans up team resources

### 11.4 Post-Sprint Retrospective Inputs

After each sprint, the leader captures:

- **What went well:** patterns and practices that worked
- **What went wrong:** conflicts, rework, unexpected issues
- **File ownership conflicts:** which files caused coordination problems
- **Task sizing accuracy:** which tasks were over/under-estimated
- **Test coverage gaps:** what broke that tests did not catch
- **Performance observations:** any endpoints trending slow

These observations should be added to CLAUDE.md or this document to improve future sprints.

---

## 12. Anti-Patterns to Avoid

### 12.1 Over-Engineering

**Anti-pattern:** Agent builds an abstraction layer, factory pattern, or plugin system when a simple function would suffice.

**Rule:** Solve the problem at hand. Do not build frameworks. If a pattern is needed, it will be requested explicitly. Three concrete implementations before abstracting.

### 12.2 Breaking Other Tests

**Anti-pattern:** Agent modifies a shared component and breaks 15 tests owned by the test-automator, then says "I only changed the component, the tests need updating."

**Rule:** If your change breaks existing tests, either (a) your change is wrong and you need a different approach, or (b) you need to coordinate with the test-automator to update the tests. You do not push breaking changes and leave cleanup to others.

### 12.3 Ignoring Existing Patterns

**Anti-pattern:** Agent creates a new API endpoint using a completely different pattern from the 30 existing endpoints (different error handling, different response format, different URL structure).

**Rule:** Before writing new code, read 2-3 existing examples of the same type. Follow the established pattern. If you think the existing pattern is wrong, raise it with the leader for discussion, not unilateral change.

### 12.4 The "Just Increase the Timeout" Trap

**Anti-pattern:** E2E test times out waiting for API response. Agent increases timeout from 5s to 30s and calls it fixed.

**Rule:** Increasing timeouts is NEVER a fix. Any API endpoint taking more than 5 seconds is a performance bug. Investigate: slow queries, missing indexes, connection pool exhaustion, lock contention, N+1 queries, unnecessary computation. The only exception is LLM inference calls where latency is inherently variable.

### 12.5 Creating New Files When Editing Existing Ones Suffices

**Anti-pattern:** Agent creates `brokerUtilsNew.ts` instead of adding to the existing `brokerStore.ts`, because they are not sure about modifying the existing file.

**Rule:** Always prefer editing existing files. New files are created only when (a) the existing file would become too large (500+ lines), (b) the new code belongs to a genuinely different domain/concern, or (c) the task explicitly requires a new file. When in doubt, ask the leader.

### 12.6 Adding Features Beyond What Was Asked

**Anti-pattern:** Task says "add a sort button to the broker queue." Agent adds sort button, filter dropdown, search bar, column resize, and pagination.

**Rule:** Do exactly what the task asks. If you see obvious improvements, report them to the leader as potential future tasks. Do not implement them without approval. Scope creep from agents is the number one cause of file conflicts and integration issues.

### 12.7 Silent Error Swallowing

**Anti-pattern:** Agent adds a try/catch that returns a default value on any error, hiding bugs that would otherwise be caught.

```python
# ANTI-PATTERN
try:
    result = await complex_computation(data)
except Exception:
    result = {}  # Silently return empty, hiding the real error
```

**Rule:** Errors must be either (a) handled with specific recovery logic and logged, or (b) propagated to the caller. Never catch broad exceptions and return defaults.

### 12.8 Hardcoding Values That Should Be Configurable

**Anti-pattern:** Agent hardcodes `"http://localhost:4000"` in a fetch call, or hardcodes a rate limit of `10` inline instead of using a constant.

**Rule:** Use existing environment variables and configuration patterns. If a new configurable value is needed, follow the process in `docs/development-standards.md` (document in `vite-env.d.ts`, add to docker-compose, add to Dockerfile.prod, document in standards file).

### 12.9 File Conflict Resolution Failures

**Anti-pattern:** Two agents both edit `client.ts`. The second agent's save overwrites the first agent's changes. Nobody notices until tests fail 30 minutes later.

**Rule:** The leader assigns file ownership upfront. If ownership was not assigned, agents must check with the leader before editing any shared file. "I didn't know another agent was editing it" is not an acceptable explanation -- it means the leader failed to assign ownership, and the agent failed to check.

### 12.10 Inventing New Environment Variables

**Anti-pattern:** Agent needs a backend URL and creates `VITE_API_BASE`, `VITE_BACKEND_URL`, or `REACT_APP_API_URL` instead of using the existing `VITE_API_URL`.

**Rule:** Use `VITE_API_URL || "/api"` for all frontend API calls. This is the ONE pattern. See `docs/development-standards.md`. If you genuinely need a new environment variable (rare), follow the documented process: type in `vite-env.d.ts`, value in `docker-compose.yml`, default in `Dockerfile.prod`, documented in standards file.

### 12.11 Committing Debug Artifacts

**Anti-pattern:** Agent commits `console.log("DEBUG HERE")`, `print(f"XXX {variable}")`, or commented-out code blocks.

**Rule:** Remove all debug statements before committing. If logging is needed for production, use the proper logging framework with appropriate log levels.

### 12.12 Ignoring the Architecture Document

**Anti-pattern:** Agent creates a new endpoint that reads from PostgreSQL directly for a GET request, bypassing the Redis cache that the architecture mandates.

**Rule:** Before implementing any endpoint or event handler, check the architecture document (`docs/architecture-clearance-platform.md`) for the relevant domain. Follow the CQRS pattern (GET reads from Redis, POST/PUT/DELETE writes through domain services and emits events). If the architecture does not cover your case, ask the leader.

---

## Appendix A: Quick Reference Card

### Agent Spawn Template

When the leader spawns a teammate, include this context:

```
You are {agent-name}, a {role} on the Clearance Platform team.

Your file ownership:
{list of files this agent may create or modify}

Your tasks:
{list of task names and IDs}

Key rules:
- ONLY modify files in your ownership list
- Follow patterns in docs/development-standards.md and docs/ways-of-working.md
- Report blockers immediately via SendMessage to the leader
- All tests must pass before marking a task complete
- Never increase timeouts to fix slow endpoints
- Transform snake_case->camelCase in client.ts, not in components
- API calls use VITE_API_URL || "/api" exclusively
```

### Pre-Sprint Checklist (Leader)

- [ ] Read and understand the sprint backlog
- [ ] Decompose each item into tasks with acceptance criteria
- [ ] Identify all files that will be modified
- [ ] Assign file ownership (no overlaps)
- [ ] Map task dependencies
- [ ] Determine team size and roles
- [ ] Create shared task list
- [ ] Spawn teammates with context
- [ ] Enable delegate mode
- [ ] Broadcast sprint plan to all agents

### Pre-Commit Checklist (Agent)

- [ ] Only my owned files were modified
- [ ] `npm run build` passes (frontend changes)
- [ ] `pytest` passes (backend changes)
- [ ] `npm run test` passes (frontend test changes)
- [ ] No debug statements (`console.log`, `print`)
- [ ] No hardcoded URLs or ports
- [ ] Error handling is explicit
- [ ] Acceptance criteria met
- [ ] Status reported to leader

### Rate Limits Reference

**Current policy: Rate limiting infrastructure is in place but limits are set high enough to be non-restrictive.** The functionality exists and can be tightened for production. Do not remove rate limit decorators — just keep limits generous.

| Endpoint Category | Current Limit | Notes |
|---|---|---|
| Default API endpoints | 50,000/min | Effectively unlimited for development/testing |
| LLM inference endpoints (classify, analyze, chat, description-quality, resolution) | 50,000/min | Same — no artificial throttling during development |

**Do NOT lower these limits** without explicit user approval. The rate limit infrastructure (slowapi middleware, per-endpoint decorators) must remain in place for future production hardening, but current limits must not interfere with development, testing, or simulation load.

---

## Appendix B: Project File Map

Key files agents need to know about:

| File | Purpose | Ownership Rule |
|---|---|---|
| `frontend/src/api/client.ts` | All API calls and transforms | Single owner per sprint |
| `frontend/src/types.ts` | Shared TypeScript interfaces | Single owner per sprint |
| `frontend/src/App.tsx` | Route and layout registration | Single owner |
| `backend/app/api/routes/__init__.py` | Route registration | Single owner |
| `backend/app/main.py` | Application bootstrap | Single owner |
| `docs/development-standards.md` | Mandatory coding patterns | Leader only |
| `docs/architecture-clearance-platform.md` | Definitive architecture reference | Requires user approval to change |
| `docs/ways-of-working.md` | This document | Leader only |
| `clearance-engine/docker-compose.yml` | Dev environment | Infrastructure agent only |
| `clearance-engine/docker-compose.prod.yml` | Production environment | Infrastructure agent only |
