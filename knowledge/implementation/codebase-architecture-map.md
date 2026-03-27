# Clearance Intelligence Engine -- Codebase Architecture Map

Annotated file tree covering every module in the repository with its purpose, dependencies, and role in the system.

Last updated: 2026-02-03

---

## Repository Root

```
clearance-engine/
  docker-compose.yml          5-service compose stack (postgres, redis, qdrant, backend, frontend)
  Makefile                    Developer convenience targets: up, down, migrate, seed-*, test-*, lint, format, shell-*
  .env                        Environment variables (API keys, DB creds) -- git-ignored
  backend/                    Python 3.12 FastAPI application
  frontend/                   React 18 TypeScript application
  k8s/                        Kubernetes deployment manifests
  doc/                        Technical documentation
```

---

## Backend (`backend/`)

### Entry Points and Configuration

```
backend/
  Dockerfile                  Multi-stage build: python:3.12-slim, pip install, uvicorn entrypoint
  pyproject.toml              Project metadata, dependencies, tool config (ruff, pytest)
  alembic.ini                 Alembic migration configuration (points to alembic/ directory)
  uv.lock                     Dependency lock file (uv package manager)

  app/
    __init__.py               Package marker
    main.py                   ** KEY ENTRY POINT **
                              - create_app() factory: FastAPI instance, CORS, timing middleware
                              - lifespan(): async context manager for startup/shutdown
                                - Creates SQLAlchemy async engine (pool_size=20, max_overflow=10)
                                - Connects Redis (max_connections=50)
                                - Connects Qdrant (async client)
                                - Creates SimulationCoordinator (optional auto-start)
                              - Registers 18 route modules
                              - /health endpoint (checks DB + Redis + Qdrant)
                              - Module-level: app = create_app() for `uvicorn app.main:app`

    config.py                 ** KEY ENTRY POINT **
                              - Settings(BaseSettings): pydantic-settings with .env loading
                              - DATABASE_URL (postgresql+asyncpg://...localhost:5440)
                              - DATABASE_SYNC_URL (postgresql://...localhost:5440)
                              - REDIS_URL (redis://localhost:6381/0)
                              - QDRANT_URL (http://localhost:6336)
                              - ANTHROPIC_API_KEY, OPENAI_API_KEY
                              - LLM_PRIMARY_PROVIDER="anthropic", LLM_PRIMARY_MODEL="claude-sonnet-4-20250514"
                              - LLM_FALLBACK_PROVIDER="openai", LLM_FALLBACK_MODEL="gpt-4o"
                              - LLM_TEMPERATURE=0.1, LLM_MAX_TOKENS=4096
                              - SIM_AUTO_START, SIM_TIME_RATIO=10.0, SIM_RANDOM_SEED
                              - get_settings() -> cached singleton via @lru_cache
```

### API Layer (`app/api/`)

```
  app/api/
    __init__.py               Package marker
    streaming.py              SSE utilities:
                              - Event type constants (CLASSIFICATION_STEP, TARIFF_COMPLETE, etc.)
                              - sse_event(event_type, data) -> formatted SSE frame string
                              - sse_response(generator) -> StreamingResponse with event-stream headers

    schemas/
      __init__.py             Package marker
      common.py               Shared Pydantic models:
                              - ConfidenceLevel enum (HIGH/MEDIUM/LOW)
                              - EngineResult base model (engine_id, status, processing_time_ms)
                              - ErrorResponse model (error, detail, request_id, timestamp, context)
                              - SessionInfo model (session_id, created_at, company_id)
      trade_lanes.py          TradeLaneRequest/Response schemas
      regulatory.py           RegulatorySignal/ScenarioRequest schemas

    routes/
      __init__.py             Package marker
      analysis.py             POST /analyze/stream - Full pipeline (SSE streaming + non-streaming)
                              Instantiates AnalysisPipeline, calls .stream() or .execute()
      classify.py             POST /classify/stream - E1 classification only (SSE)
      tariff.py               POST /tariff - E2 tariff calculation
      compliance.py           POST /compliance - E3 compliance screening
      fta.py                  POST /fta - E4 FTA eligibility check
      exception.py            POST /exception/search - E5 CROSS ruling semantic search
      regulatory.py           GET /regulatory, POST /regulatory/scenario - E6 signal feed + modeling
      screening.py            POST /screen - DPS entity screening
      dashboard.py            GET /dashboard/demo - Dashboard aggregates (demo data)
      trade_lanes.py          POST /trade-lanes - Multi-destination tariff/compliance comparison
      session.py              GET /session - Session management
      products.py             GET/POST/PUT /products - Product catalog CRUD (UUID keys)
      orders.py               GET/POST/PATCH /orders - Order lifecycle (draft->analyzed->shipped)
                              POST /orders/:id/analyze - Run E1+E2+E3 per line item
                              POST /orders/:id/ship - Create shipment from order
      shipments.py            GET/POST /shipments - Shipment list and creation
                              GET /shipments/:id - Shipment detail with events/waypoints/financials
                              POST /shipments/:id/analyze - Run analysis on existing shipment
                              Contains _seed_shipments() for demo data initialization
      documents.py            POST /documents/requirements - E7 document requirements
                              POST /documents/generate - E7 document field generation
                              POST /documents/analyze - Document validation
                              POST /documents/orders/:id/upload - Upload PDF to order
                              POST /documents/shipments/:id/upload - Upload PDF to shipment
                              GET /documents/:id/content - Serve stored PDF
      chat.py                 POST /chat/stream - AI assistant (SSE)
                              Agent selection, agentic tool-use loop (max 10 iterations)
                              Streams text chunks and tool progress events
      resolution.py           POST /resolution/lookup - Product classification lookup
                              POST /resolution/upload - Document upload and extraction
                              POST /resolution/suggest - Resolution step suggestions
      simulation.py           POST /simulation/start|stop|reset|pause|resume
                              GET /simulation/status - Current state + actor configs
                              PATCH /simulation/clock - Update time ratio
                              PATCH /simulation/actors/:id - Hot-update actor config
                              GET /simulation/dashboard/stream - Dashboard SSE (periodic aggregates)

    agent/
      __init__.py             Package marker ("Screen-specific agentic agents")
      tools.py                11 tool schemas (Anthropic tool-use format) + execute_tool() dispatcher:
                              READ (8):  lookup_shipment, search_shipments, screen_entity,
                                         classify_product, calculate_tariff, check_compliance,
                                         identify_documents, get_regulatory_signals
                              WRITE (3): add_shipment_event, request_shipper_documents,
                                         resolve_shipment_hold
                              Helper: _shipment_to_dict(), _shipment_summary(), _serialize()
                              TOOL_LABELS dict for SSE progress display
      prompts.py              4 AgentConfig dataclasses with screen-specific system prompts:
                              - Exception Resolution Specialist (exception screen)
                              - Shipment Analysis Agent (shipment detail screen)
                              - Compliance Advisor (compliance/screening screens)
                              - General Trade Expert (default fallback)
                              Shared _SHARED_INSTRUCTIONS preamble enforcing tool-use-first behavior
```

### Engine Layer (`app/engines/`)

```
  app/engines/
    __init__.py               Package marker
    base.py                   ** CORE ABSTRACTION **
                              - EngineOutput @dataclass(slots=True):
                                engine_id, status, data, processing_time_ms, errors, warnings, metadata
                              - BaseEngine ABC:
                                Requires: engine_id, engine_name (class attrs), execute() (async method)
                                Provides: _start_timer(), _elapsed_ms()

    e0_preclearance/
      __init__.py             Package marker + convenience imports
      engine.py               PreClearanceEngine: LLM agentic loop with 5 tools
                              Runs during simulation for risk assessment on booked shipments
                              Deterministic fallback if LLM unavailable
      tools.py                5 pre-clearance-specific tool definitions
      prompts.py              System prompt for pre-clearance agent reasoning

    e1_classification/
      __init__.py             Package marker + classify() convenience function
      engine.py               ClassificationEngine(BaseEngine):
                              Two-phase LLM classification using GRI (General Rules of Interpretation)
                              Phase 1: Reasoning (generate analysis)
                              Phase 2: HS code extraction + validation
                              stream_classify() async generator for SSE streaming
      prompts.py              System/user prompt templates for classification LLM calls
      validator.py            HS code format validation (length, digit patterns)
      confidence.py           Confidence scoring logic (HIGH/MEDIUM/LOW based on match quality)

    e2_tariff/
      __init__.py             Package marker + calculate_tariff() convenience function
      engine.py               TariffEngine(BaseEngine):
                              Multi-jurisdiction tariff calculation
                              Queries htsus_headings, section_301, section_232, ieepa_rates, adcvd_orders
                              Applies fee_schedule (MPF, HMF)
                              Delegates to tax_regime for non-US jurisdictions
      landed_cost.py          Landed cost assembly: base duty + surcharges + taxes + fees
      tax_regime.py           TaxRegimeEngine: loads TaxRegimeTemplate from DB
                              Processes ordered computation steps (additive vs cascading)
      programs/
        __init__.py           Package marker for duty program modules
      regimes/
        __init__.py           Package marker for jurisdiction-specific regime logic
        eu.py                 EU tariff regime (TARIC-based)
        cn.py                 China tariff regime (MFN + VAT)
        br.py                 Brazil regime (II + IPI + PIS + COFINS + ICMS cascading)
        india.py              India regime (BCD + SWS + IGST)

    e3_compliance/
      __init__.py             Package marker + screen_compliance() convenience function
      engine.py               ComplianceEngine(BaseEngine):
                              Orchestrates PGA check + DPS screening + UFLPA analysis
                              Returns overall_risk: HIGH/MEDIUM/LOW
      pga.py                  PGA flag lookup: queries pga_mappings table by HS code range
      dps.py                  DPSScreener: fuzzy match against restricted_parties using pg_trgm
      fuzzy_match.py          pg_trgm similarity helpers (SQL functions, threshold tuning)
      uflpa.py                UFLPA forced labor risk assessment
                              Region-based (Xinjiang) + entity-based + commodity-based scoring

    e4_fta/
      __init__.py             Package marker
      engine.py               FTAEngine(BaseEngine):
                              Queries fta_rules table by agreement + chapter range
                              Returns eligibility, rule_type, tariff_shift_requirement, savings estimate

    e5_exception/
      __init__.py             Package marker
      engine.py               ExceptionEngine(BaseEngine):
                              Calls EmbeddingService.search_rulings() for semantic CROSS search
                              LLM-generates exception response drafts from matched rulings

    e6_regulatory/
      __init__.py             Package marker
      engine.py               RegulatoryEngine(BaseEngine):
                              get_signals(): filtered query against regulatory_signals table
                              model_scenario(): calculates duty delta from proposed signal changes

    e7_documents/
      __init__.py             Package marker
      engine.py               DocumentEngine(BaseEngine):
                              Determines required documents based on HS code, origin, dest, value
                              Generates document field suggestions via LLM
```

### Orchestration Layer (`app/orchestration/`)

```
  app/orchestration/
    __init__.py               Package marker
    pipeline.py               ** CORE ORCHESTRATION **
                              AnalysisPipeline class:
                                __init__: creates E1, E2, E3, E4 instances
                                execute(): Phase 1 (E1 sequential) -> Phase 2 (E2+E3+E4 parallel)
                                stream(): Same but yields SSE event dicts incrementally
                              _safe_output(): converts exceptions to error EngineOutput
                              _assemble(): merges 4 EngineOutputs into unified result dict
    parallel.py               run_parallel(*coroutines, timeout=30.0):
                              asyncio.gather with shared timeout
                              On timeout: returns error EngineOutputs for all coroutines
    error_handling.py         handle_engine_error(): exception -> EngineOutput converter
                              is_critical_failure(): E1 or E2 error = critical, E3/E4 = non-critical
```

### Knowledge Layer (`app/knowledge/`)

```
  app/knowledge/
    __init__.py               Package marker
    models/
      base.py                 Base (DeclarativeBase) + TimestampMixin (created_at, updated_at)
      tariff.py               6 ORM models:
                              - HTSUSChapter: chapter-level metadata (99 chapters)
                              - HTSUSHeading: tariff lines (52K+ rows, multi-jurisdiction)
                              - Section301List: 301 tariff lists (China)
                              - Section232Scope: steel/aluminum scope
                              - IEEPARate: per-country reciprocal rates
                              - ADCVDOrder: antidumping/CVD orders
      compliance.py           3 ORM models:
                              - RestrictedParty: DPS entries with pg_trgm GIN index
                              - PGAMapping: agency requirements by HS code range
                              - FTARule: USMCA rules of origin
      reference.py            5 ORM models:
                              - CrossRuling: CROSS binding rulings (links to Qdrant via vector_id)
                              - TaxRate: multi-jurisdiction taxes
                              - FeeSchedule: CBP fees (MPF, HMF)
                              - ExchangeRate: FX snapshots
                              - TaxRegimeTemplate: per-jurisdiction computation templates (JSONB)
      operational.py          7 ORM models:
                              - RegulatorySignal: regulatory intelligence feed
                              - DemoShipment: demo customs entries
                              - Product: product catalog (UUID PK)
                              - Order: clearance orders (UUID PK, FK -> Shipment)
                              - OrderLineItem: line items (FK -> Order)
                              - Shipment: physical shipment tracking (UUID PK, JSONB events/waypoints/codes/financials)
                              - Document: stored PDFs (FK -> Order, FK -> Shipment)
    repositories/
      __init__.py             Package marker (repository pattern placeholder)
```

### Services Layer (`app/services/`)

```
  app/services/
    __init__.py               Package marker
    llm.py                    ** CORE SERVICE **
                              LLMService class:
                              - __init__: creates AsyncAnthropic + AsyncOpenAI clients
                              - complete(): non-streaming, auto-failover between providers
                              - stream(): streaming, yields text chunks, auto-failover
                              - complete_with_tools(): tool-use calls, Anthropic-native or OpenAI-translated
                              - OpenAI responses wrapped in SimpleNamespace for Anthropic compatibility
    database.py               DatabaseService class:
                              - Async engine (asyncpg) + sync engine (for Alembic)
                              - session() context manager with auto-rollback on exception
                              - get_session(), get_sync_session() for unmanaged usage
                              - health_check(), close()
    cache.py                  CacheService class:
                              - Fail-open: all methods catch exceptions, return None on failure
                              - get/set/delete/delete_pattern with JSON serialization
                              - Static key builders: tariff_key(), classification_key(), screening_key(), ruling_search_key()
    embedding.py              EmbeddingService class:
                              - Qdrant async client + OpenAI embedding generation (httpx)
                              - Collection: cross_rulings, 1536-dim cosine
                              - embed_text(), embed_texts() (batch), search_rulings(), upsert_ruling()
                              - HS code filter: exact match OR same 4-digit heading
    currency.py               CurrencyService: FX conversion using exchange_rates table
    dashboard.py              Dashboard metrics aggregation service
```

### Simulation Layer (`app/simulation/`)

```
  app/simulation/
    config.py                 Pydantic configuration models:
                              - SimulationConfig (top-level): time_ratio, random_seed, auto_start, dashboard_interval
                              - ShipperConfig: tick_sim_minutes=60, base_rate=30/day, misclassification_rate=0.07
                              - CarrierConfig: tick=120min, delay_chance=0.08
                              - CustomsConfig: tick=30min, stp_rate=0.87, hold_rate=0.038, corridor_scrutiny
                              - PGAConfig: tick=60min, FDA/EPA/CPSC review times and approval rates
                              - PreClearanceConfig: tick=15min
                              - ComplianceConfig: tick=15min, run_real_engines flag
                              - ResolutionConfig: tick=240min, resolve_rate=0.75

    coordinator.py            ** SIMULATION ENTRY POINT **
                              SimulationCoordinator class:
                              - start(): creates EventBus + 7 actors, starts dashboard broadcast loop
                              - stop(): stops all actors + event bus + dashboard task
                              - reset(): stops, clears shipments, re-seeds demo data, resets clock
                              - pause()/resume(): delegates to SimulationClock
                              - subscribe_sse()/unsubscribe_sse(): manage dashboard SSE subscribers
                              - _dashboard_loop(): periodic aggregate computation + broadcast

    clock.py                  SimulationClock class:
                              - Compressed time: 1 real second = time_ratio simulated minutes (default 10)
                              - now() -> datetime, now_iso(), elapsed_sim_minutes(), sim_day(), sim_hour()
                              - is_business_hours() (08:00-18:00)
                              - pause()/resume() with accumulated pause tracking
                              - sleep_sim_minutes(): converts sim time to real seconds

    event_bus.py              EventBus class:
                              - Redis pub/sub on channel "sim:events"
                              - subscribe(event_type, handler), publish(event_type, data)
                              - _listen(): background task reading Redis pub/sub
                              - In-memory fallback: direct dispatch if Redis unavailable

    state_machine.py          Shipment state transitions:
                              - TRANSITIONS dict: {from_status: {to_status: {allowed_actors}}}
                              - States: booked, in_transit, at_customs, inspection, held, cleared, delivered
                              - Terminal: delivered
                              - transition(): guarded status change + event appended to shipment.events
                              - TransitionError exception for invalid moves

    dashboard_aggregator.py   get_cached_or_compute(): builds dashboard snapshot
                              Aggregates shipment counts, duty totals, status distribution
                              Caches in Redis with short TTL

    actors/
      __init__.py             Package marker
      base.py                 BaseActor ABC: start()/stop(), tick loop on configurable interval
                              Each actor runs as an asyncio.Task calling tick() repeatedly
      shipper.py              ShipperActor: creates new bookings at configurable rate
                              Business hours weighted (2.5x daytime, 0.3x overnight)
                              Random product descriptions, origins, values
                              Occasional misclassification and value errors
      preclearance.py         PreClearanceAdapter: wraps E0 engine for simulation
                              Runs entity screening on each new booking
                              Can hold shipments before carrier pickup
      carrier.py              CarrierActor: simulates transit and delivery
                              Picks up booked shipments -> in_transit -> at_customs
                              Delay simulation (8% chance, 6-48 hour range)
                              Post-clearance: cleared -> delivered
      customs.py              CustomsActor: primary clearance decision maker
                              STP (straight-through processing): 87% pass rate
                              Manual review, physical inspection (4%), holds (3.8%)
                              Corridor scrutiny multipliers (CN: 1.3x)
                              HS reclassification (5% rate)
      pga.py                  PGAActor: Partner Government Agency reviews
                              FDA (1-5 day review, 92% approval)
                              EPA (1-3 days, 95%), CPSC (1-2 days, 97%)
      compliance_engine.py    ComplianceEngineActor: runs real E1-E3 engines or simulated results
                              Attaches classification/tariff/compliance analysis to shipments
      resolution.py           ResolutionActor: resolves held shipments
                              1-5 day review cycle
                              Outcomes: resolve (75%), escalate (10%), pending (15%)
```

### Database Migrations (`backend/alembic/`)

```
  alembic/
    env.py                    Alembic environment: loads Settings, configures async engine
                              Imports all ORM models for autogenerate support
    versions/
      001_initial_schema.py   Initial migration: creates all 18 tables with indexes
```

### Data Ingestion (`backend/data/`)

```
  data/
    __init__.py               Package marker
    init.sql                  PostgreSQL init script (mounted to docker-entrypoint-initdb.d)
                              Creates pg_trgm extension, sets up initial schema if needed

    validation/
      __init__.py             Package marker for data validation tests

    ingestion/
      __init__.py             Package marker
      base.py                 Base ingestion utilities (session creation, upsert helpers)
      seed_all.py             ** MASTER SEED SCRIPT ** -- runs all seeders in dependency order
      seed_htsus_chapters.py  Seeds 99 HTSUS chapter records
      seed_htsus.py           Seeds US HTSUS heading records (thousands of tariff lines)
      seed_eu_tariff.py       Seeds EU TARIC tariff schedule
      seed_cn_tariff.py       Seeds China MFN + VAT tariff data
      seed_br_tariff.py       Seeds Brazil tariff + cascading tax data
      seed_in_tariff.py       Seeds India BCD + IGST + SWS tariff data
      seed_section301.py      Seeds Section 301 lists (Lists 1-4A, exclusions)
      seed_section232.py      Seeds Section 232 steel/aluminum scope
      seed_ieepa.py           Seeds IEEPA per-country reciprocal tariff rates
      seed_adcvd.py           Seeds AD/CVD orders
      seed_fees.py            Seeds CBP fee schedule (MPF, HMF)
      seed_taxes.py           Seeds multi-jurisdiction tax rates
      seed_tax_regimes.py     Seeds per-jurisdiction computation templates (JSONB)
      seed_pga.py             Seeds PGA agency mappings by HS code range
      seed_fta_rules.py       Seeds USMCA rules of origin
      seed_restricted_parties.py  Seeds consolidated DPS list (OFAC SDN, BIS Entity, UFLPA)
      seed_cross_rulings.py   Seeds CROSS rulings + Qdrant vector upserts
      seed_regulatory_signals.py  Seeds regulatory intelligence signals
      seed_demo_shipments.py  Seeds AutoParts Global demo entries
      seed_exchange_rates.py  Seeds FX rate snapshots
```

### Tests (`backend/tests/`)

```
  tests/
    __init__.py               Package marker
    conftest.py               Shared pytest fixtures (async engine, session, test client)

    unit/
      __init__.py             Package marker
      test_agent.py           Unit tests for agent tool dispatch and prompts
      engines/
        __init__.py           Package marker
        test_e1_classification.py   E1 classification engine tests (mocked LLM)
        test_e2_tariff.py           E2 tariff calculation tests (deterministic)
        test_e3_compliance.py       E3 compliance screening tests

    integration/
      __init__.py             Package marker
      test_pipeline.py        Full pipeline integration tests (E1->E2->E3->E4)

    data_validation/
      __init__.py             Package marker
      test_rate_correctness.py  Validates seeded tariff rates against known values

    e2e/
      __init__.py             Package marker
      test_golden_path.py     End-to-end golden path: classify -> tariff -> compliance
```

---

## Frontend (`frontend/`)

### Root Configuration

```
frontend/
  Dockerfile                  Multi-stage build: node:20, npm install, vite build
  package.json                Dependencies: react, react-router-dom, zustand, framer-motion, tailwindcss
  tsconfig.json               TypeScript strict config, path alias @/ -> src/
  vite.config.ts              Vite config: path aliases, dev server proxy
  tailwind.config.js          TailwindCSS configuration
  postcss.config.js           PostCSS with Tailwind plugin
  index.html                  SPA entry HTML
  eslint.config.js            ESLint configuration
  vitest.config.ts            Vitest configuration for unit tests
```

### Application Entry (`frontend/src/`)

```
  src/
    main.tsx                  ** KEY ENTRY POINT ** -- ReactDOM.createRoot, renders <App />
    App.tsx                   BrowserRouter + AnimatePresence + useRoutes(routes)
    router.tsx                ** ROUTE DEFINITIONS **
                              3 surface groups:
                              /platform/* (13 routes) -> PlatformLayout
                              /shipper/* (10 routes) -> ShipperLayout
                              /buyer/* (4 routes) -> BuyerLayout
                              Root / redirects to /platform
    types.ts                  Shared TypeScript interfaces:
                              ProductInput, ClassificationResult, ClassificationStep,
                              TariffResult, ComplianceResult, DashboardEntry, DashboardStats,
                              DPSResult, RegulatorySignal, ScenarioResult, TradeLaneResult,
                              CatalogProductNew, Order, OrderAnalysis, ShipmentSummary,
                              ShipmentDetail, DocumentRequirement, GeneratedDocument, AttachedDocument
    vite-env.d.ts             Vite environment type declarations
```

### API Client (`frontend/src/api/`)

```
  src/api/
    client.ts                 ** CORE API LAYER **
                              fetchJSON<T>() generic helper with APIError class
                              Response transformers (snake_case -> camelCase):
                                transformClassification(), transformTariff(),
                                transformCompliance(), transformProduct(), transformOrder(),
                                transformShipmentSummary(), transformShipmentDetail()
                              SSE streaming:
                                streamAnalysis() -- pipeline SSE with typed callbacks
                                streamChat() -- AI assistant SSE with tool events
                              REST functions (23 exports):
                                classifyProduct, getTariff, getCompliance,
                                analyzeProduct, screenEntity, getTradeLanes,
                                getDashboard, getRegulatorySignals, modelScenario,
                                getProducts, getProduct, createProduct, updateProduct,
                                getOrders, getOrder, createOrder, updateOrder,
                                analyzeOrder, shipOrder, updateDocumentStatus,
                                getShipments, getShipmentDetail, createShipment,
                                analyzeShipment, resolutionLookup, resolutionUpload,
                                resolutionSuggest
                              Document functions:
                                getDocumentRequirements, generateDocument, analyzeDocument,
                                uploadOrderDocument, getOrderDocuments,
                                uploadShipmentDocument, getShipmentDocuments
```

### State Management (`frontend/src/store/`)

```
  src/store/
    pipelineStore.ts          ** PRIMARY STORE **
                              ClearanceEntry interface (40+ fields for full entry tracking)
                              PlatformAggregates (monthly stats, corridors, duty programs)
                              Actions: createEntry, updateEntryStatus, updateEntryAnalysis
                              Queries: getEntry, getByStatus, getBySource, getByCorridor, getExceptionQueue
                              Pre-loaded with 49 demo entries spanning 10 corridors

    analysisStore.ts          Product analysis screen state:
                              Current analysis results (classification, tariff, compliance)
                              Streaming step tracking

    operatorStore.ts          Operator assistant state:
                              Chat history, tool events, agent context

    sessionStore.ts           Session management:
                              Session ID, company context, last activity

    cartStore.ts              Buyer cart/checkout state:
                              Selected products, quantities, duty estimates
```

### Hooks (`frontend/src/hooks/`)

```
  src/hooks/
    useAnalysis.ts            Wraps streamAnalysis() for the ProductAnalysis screen
                              Manages loading state, abort controller, error handling

    useClassification.ts      Standalone classification hook (non-pipeline usage)

    usePipelineAnalysis.ts    ** KEY HOOK **
                              Bridges streamAnalysis() with both pipelineStore and analysisStore
                              Creates pipeline entry at RECEIVED, updates through
                              CLASSIFYING -> CALCULATING -> SCREENING -> CLEARED
                              Tracks abort refs for cleanup

    useTradeLanes.ts          Trade lane comparison hook (multi-destination)

    useDashboardStream.ts     SSE subscription hook for simulation dashboard
                              Manages EventSource lifecycle and reconnection

    useSimulation.ts          Simulation control hook (start/stop/pause/reset/configure)
```

### Shared Components (`frontend/src/components/`)

```
  src/components/
    Header.tsx                Application header with navigation, surface indicator
    SurfaceSwitcher.tsx       Dropdown to switch between Platform/Shipper/Buyer surfaces
    SceneNav.tsx              Side navigation within a surface
    ConfidenceBadge.tsx       Visual badge for HIGH/MEDIUM/LOW confidence levels
    SystemsPanel.tsx          System status panel (service health indicators)
    LoadingSpinner.tsx        Reusable loading spinner component
    ErrorBoundary.tsx         React error boundary with fallback UI
```

### Platform Surface (`frontend/src/surfaces/platform/`)

```
  src/surfaces/platform/
    PlatformLayout.tsx        Layout wrapper: sidebar nav + main content + OperatorAssistant

    screens/
      ControlTower.tsx        ** MAIN DASHBOARD **
                              Aggregated stats, corridor matrix, duty distribution,
                              status breakdown, recent entries, simulation controls
      ProductAnalysis.tsx     Product analysis screen: input form + streaming results
      EntryDetail.tsx         Single entry detail with classification, tariff, compliance tabs
      TradeLaneComparison.tsx Multi-destination comparison with side-by-side tariff/compliance
      RegulatoryIntel.tsx     Regulatory signal feed + scenario modeling
      ExceptionResolution.tsx Exception queue with hold/detention management
      EntityScreening.tsx     DPS entity screening interface
      ActiveShipments.tsx     Shipment list with status filters, corridor grouping
      PlatformShipmentDetail.tsx  Full shipment detail: timeline, financials, analysis, documents
      PlatformOrders.tsx      Order list with status management
      PlatformOrderDetail.tsx Order detail with line items, analysis, document checklist
      EducationCenter.tsx     Trade compliance educational content
      ComplianceDashboard.tsx Compliance metrics and risk overview

    components/
      ProductInputForm.tsx    Reusable product analysis input form
      ReasoningChain.tsx      Animated classification reasoning step display
      CompliancePanel.tsx     PGA flags, DPS screening, UFLPA risk display
      TariffStack.tsx         Duty program breakdown visualization (stacked bar)
      DutyStackBar.tsx        Horizontal stacked bar for duty composition
      FTASummary.tsx          FTA eligibility summary card
      SignalBoard.tsx         Regulatory signal feed display
      EntryTable.tsx          Sortable/filterable entry table
      ExceptionPanel.tsx      Exception detail and resolution actions
      ExceptionQueue.tsx      Prioritized exception queue (DETAINED > FLAGGED > HOLD > EXCEPTION)
      ExceptionActions.tsx    Action buttons for exception resolution
      ScreeningResult.tsx     Entity screening results display
      DashboardStats.tsx      KPI stat cards (total entries, duty, clearance rate)
      CorridorMatrix.tsx      Trade corridor heatmap/matrix
      TradeLaneCard.tsx       Individual trade lane comparison card
      ScenarioComparison.tsx  Before/after scenario duty comparison
      RegulatoryFeed.tsx      Regulatory signal timeline feed
      RiskSummary.tsx         Aggregated risk summary panel
      PlatformPulse.tsx       Real-time simulation pulse/heartbeat display
```

### Shipper Surface (`frontend/src/surfaces/shipper/`)

```
  src/surfaces/shipper/
    ShipperLayout.tsx         Layout wrapper: sidebar nav + main content + ShipmentChat

    screens/
      ShipperCatalog.tsx      Product catalog grid with search and filter
      ShipperProductDetail.tsx Product detail with analysis results and source locations
      AddProduct.tsx          Add new product form
      OrderList.tsx           Order list with status badges
      CreateOrder.tsx         Multi-step order creation (select products, quantities, destination)
      OrderDetail.tsx         Order detail: line items, analysis, documents, ship action
      ShipmentHistory.tsx     Shipment history with timeline view
      ShipmentDetail.tsx      Shipment detail with events, waypoints, financials
      ResolutionCenter.tsx    Self-service resolution for held shipments

    components/
      ShipProductCard.tsx     Product card for catalog display
      OrderCard.tsx           Order summary card
      StatusLight.tsx         Animated status indicator (green/yellow/red)
      ShipmentCard.tsx        Shipment summary card
      ShipmentTimeline.tsx    Visual event timeline
      ShipmentWaypoints.tsx   Geographic waypoint visualization
      ShipmentFinancials.tsx  Predicted vs actual duty/cost comparison
      ShipmentChat.tsx        In-context AI chat for shipment questions
      ReadinessCard.tsx       Ship-readiness indicator card
      DutyBreakdownCard.tsx   Detailed duty line-item breakdown
      PlainLanguageSummary.tsx Plain-English compliance summary for non-experts
      AnalysisOverlay.tsx     Full-screen analysis overlay during order processing
      AnalysisResultsPanel.tsx Analysis results panel within order detail
      AttachedDocuments.tsx   Document list and upload interface
      DocumentViewer.tsx      PDF document viewer
      DocumentRequirements.tsx Document checklist with completion tracking
      NotificationCard.tsx    Notification/alert card component
```

### Buyer Surface (`frontend/src/surfaces/buyer/`)

```
  src/surfaces/buyer/
    BuyerLayout.tsx           Layout wrapper: minimal nav + content

    screens/
      StoreFront.tsx          Product browsing grid with duty-inclusive pricing
      ProductPage.tsx         Product detail with landed cost breakdown
      CheckoutPrice.tsx       Checkout with duty/tax transparency

    components/
      PriceDisplay.tsx        Price with duty overlay
      DutiesBadge.tsx         Compact duty percentage badge
      BreakdownToggle.tsx     Expandable duty breakdown accordion
```

### Utility Libraries (`frontend/src/lib/`)

```
  src/lib/
    currencyFormat.ts         Currency formatting utilities (USD, EUR, CNY, etc.)
    plainLanguage.ts          Converts technical compliance data to plain English
    pdfGenerator.ts           Client-side PDF generation for documents
    tradeKnowledge.ts         Static trade knowledge data (country names, HS chapter titles)
```

### Static Data (`frontend/src/data/`)

```
  src/data/
    products.ts               Static product catalog for buyer surface demo
    systemInfo.ts             System information constants (version, build)
```

### Tests (`frontend/src/test/`)

```
  src/test/
    setup.ts                  Vitest setup: MSW server initialization
    mocks/
      handlers.ts             MSW request handlers (mock API responses)
      server.ts               MSW server configuration
    regression/
      no-crash.test.tsx       Smoke tests: all screens render without crash
      connected-flows.test.tsx Flow tests: cross-screen data consistency
      agent-tools.test.tsx    Agent tool integration regression tests
    integration/
      api-live.test.ts        Live API integration tests (requires running backend)

  e2e/
    agent-assistant.spec.ts   Playwright E2E tests for operator assistant
```

---

## Kubernetes (`k8s/`)

```
k8s/
  kustomization.yaml          Kustomize base: lists all resources
  namespace.yaml              clearance-engine namespace definition
  configmap.yaml              Non-secret config: API URLs, feature flags, LLM models
  secrets.yaml                Base64-encoded secrets: API keys, DB password
  postgres-statefulset.yaml   StatefulSet (1 replica) + PVC (10Gi) + ClusterIP Service
  redis-deployment.yaml       Deployment (1 replica) + ClusterIP Service
  qdrant-statefulset.yaml     StatefulSet (1 replica) + PVC (5Gi) + ClusterIP Service
  api-deployment.yaml         Deployment (2+ replicas) + ClusterIP Service
                              Resource limits, health/readiness probes, env from configmap/secret
  frontend-deployment.yaml    Deployment (2+ replicas) + ClusterIP Service
  ingress.yaml                Ingress: TLS termination, path-based routing (/ -> frontend, /api -> backend)
```

---

## Key Entry Point Summary

| File | Role |
|------|------|
| `backend/app/main.py` | FastAPI app factory, lifespan, router registration |
| `backend/app/config.py` | All configuration (DB, Redis, Qdrant, LLM, Sim) |
| `backend/app/orchestration/pipeline.py` | Core analysis pipeline (E1 -> parallel E2+E3+E4) |
| `backend/app/services/llm.py` | Dual-provider LLM with failover |
| `backend/app/simulation/coordinator.py` | Simulation lifecycle and actor management |
| `backend/app/api/agent/tools.py` | 11 agent tools (8 read + 3 write) |
| `backend/app/api/routes/chat.py` | AI assistant agentic loop |
| `backend/data/ingestion/seed_all.py` | Master data seeding script |
| `frontend/src/main.tsx` | React app mount point |
| `frontend/src/router.tsx` | All route definitions (3 surfaces, 26 routes) |
| `frontend/src/api/client.ts` | API client with SSE streaming + transforms |
| `frontend/src/store/pipelineStore.ts` | Central clearance entry state |
| `docker-compose.yml` | 5-service local dev stack |
| `Makefile` | Developer workflow targets |
