# Clearance Vibe — Unified Product Backlog

**Synthesized from**: 6 analyst reports (3 UX + 3 Domain) across Broker, Platform, and Shipper/Buyer surfaces
**Date**: 2026-02-07
**Backlog items**: 72

---

## 1. Executive Summary

### The Formula 1 Engine Powering a Sedan's Dashboard

Clearance Vibe has built one of the most sophisticated customs clearance intelligence backends in the startup space. Under the hood sits:

- **8 intelligence engines**: Classification (E1), Tariff (E2), Compliance (E3), FTA Eligibility (E4), Description Quality, Document Requirements, Routing, and Consolidation
- **6 AI agent personas**: Broker Agent, Platform Operator, Shipper Assistant, Compliance Advisor, Resolution Specialist, and Classification Expert
- **22 composable tools**: From `verify_classification` to `screen_entity` to `resolve_shipment_hold` — each callable, contextual, and powerful
- **19 simulation actors**: Generating realistic customs clearance scenarios with correct regulatory timing, PGA agency behavior, demurrage accrual, and CBP authority responses
- **A deterministic document engine**: Rule-based requirement identification with legal citations, AI-powered field population, and PDF generation — `DocumentRequirements.tsx` is the gold standard

**The problem is not the engine. It's the dashboard.**

The frontend surfaces this intelligence through templates, placeholders, and passive information displays. The result:

| Dimension | Backend Reality | Frontend Experience |
|---|---|---|
| **Tools** | 22 agent tools with structured APIs | 16 of 22 accessible only through chat |
| **Classification** | GRI-based analysis, confidence scoring, alternative codes | "Improve classification confidence" button → opens chat |
| **Duty Calculation** | Full tariff stack: MFN + 301 + 232 + IEEPA + AD/CVD | 5% hardcoded flat rate (Buyer), crude chapter-level lookup (Broker) |
| **Compliance** | PGA per-agency, UFLPA, DPS/SDN screening, entity matching | Single "PGA Documentation" checklist item, 3 hardcoded entity names |
| **Resolution** | Step-by-step workflows with tool invocations | Numbered text lists with no clickable actions |
| **Regulatory Intel** | Real scenario modeling against portfolio | 6 hardcoded signals from 2025, no admin CRUD |
| **Simulation** | 19 actors, 7.5% exception injection, carrier D&D modeling | No control panel, no scenario builder, Compliance Dashboard shows fake data |
| **Documents** | 8 document types with field schemas, AI generation, PDF pipeline | Upload-only on Broker surface; buyer surface has zero document awareness |

**Three user stories that capture the gap:**

1. **The empty contact message**: A broker clicks "Request Documents from Shipper" and gets `"Dear {company},\n\n[Message body]\n\nBest regards,\n[Broker Name]"` — a placeholder where the system has enough context to generate a complete, specific document request.

2. **The upload-only documents**: Missing documents show an "Upload" button but no "Generate" button — even for documents the system can auto-generate (entry summary, duty calculation worksheet). The `DocumentRequirements.tsx` pattern proves this works; it just isn't replicated.

3. **The passive improvement suggestions**: Analysis results show "Upload Certificate of Origin" as a bullet point that does nothing. It should scroll to the relevant checklist item and offer a "Generate" or "Request from Shipper" action.

**The strategic fix**: Transform every piece of displayed information into an actionable workflow. The backend capabilities exist. The frontend needs to surface them as buttons, not text.

---

## 2. Capability Gap Matrix

### Backend Intelligence vs. Frontend Surfacing

| Backend Capability | Code Location | Broker Surface | Platform Surface | Shipper Surface | Buyer Surface |
|---|---|---|---|---|---|
| `verify_classification` — HS code verification with GRI analysis | `tools.py:492-509` | Chat only | Chat only | Via analysis flow | Not available |
| `calculate_entry_fees` — Full tariff stack with 301/232/AD/CVD | `tools.py:511-527` | Chat only; sidebar shows 5% fallback | Via Product Analysis | Via analysis flow | **Hardcoded 5%** |
| `check_compliance` — PGA + DPS + UFLPA screening | `tools.py:158-179` | Chat only | Chat only | Via analysis flow | Not available |
| `screen_entity` — Denied party screening | `tools.py:82` | Chat only | Standalone screen, no link from exceptions | Not available | Not available |
| `check_entry_readiness` — Pre-submission gate | `tools.py:474` | Chat only | Not available | Not available | Not available |
| `get_regulatory_signals` — Regulatory change alerts | `tools.py:213` | Chat only | **Hardcoded 6 signals** | Not available | Not available |
| `identify_documents` — Deterministic doc requirements | `tools.py:181-212` | Chat only | Not available | **Fully surfaced** (gold standard) | Not available |
| `draft_communication` — Contextual message drafting | `broker.py:2461` | Has UI but placeholder fallback | "Contact Shipper" is a no-op | Not available | Not available |
| `resolve_shipment_hold` — Hold resolution workflow | `tools.py:328` | Chat only | Chat only | Not available | Not available |
| `search_by_reference` — Cross-field search | `tools.py:651-668` | Chat only | Not available | Not available | Not available |
| `get_cage_status` — Warehouse cage tracking | `tools.py:633` | Chat only | Not available | Not available | Not available |
| `assess_description_quality` — AI quality scoring | `tools.py:240` | Chat only | Not available | **Fully surfaced** (AddProduct) | Not available |
| Entry Summary `/entries/{id}/summary` | `broker.py:1566` | Not available | Used internally | Not available | Not available |
| Document Generation + PDF pipeline | `documents.py` | Upload only | Not available | **Generate + Upload** | Not available |
| FTA Eligibility (E4 engine) | `fta.py` | Not available | Not available | In analysis results | Not available |
| Simulation API (start/stop/configure) | `simulation/` | Not available | **No UI** | Not available | Not available |
| Routing Engine (multi-modal waypoints) | `routing/` | Not available | Not available | Waypoints shown | Not available |

**Key insight**: The Shipper surface's `DocumentRequirements.tsx` and `AddProduct.tsx` description quality panel prove the frontend CAN surface backend intelligence beautifully. These patterns need to be replicated across all surfaces.

---

## 3. Prioritized Backlog

### P0 — Template/Placeholder/Hardcoded Content Where Real Intelligence Should Operate

| ID | Title | What's Wrong | Why It Matters | What It Should Look Like | Backend Available | Changes | Effort | Perspective |
|---|---|---|---|---|---|---|---|---|
| BL-001 | **Buyer duty calculation hardcoded at 5%** | `ProductPage.tsx:51`: `const taxes = Math.round(product.declaredValue * 0.05)` — flat 5% regardless of product, HS code, origin, or tariff program. Real rates range 0-400%+. | **Both agreed (UX: B5, Domain: #1)**: UX says "buyers see wildly inaccurate pricing"; Domain says "completely undermines the 'duties included' value prop." EV batteries should show ~38.4%, not 5%. $4,175 understatement on the battery product alone. | Product page calls `/api/analyze/stream` with buyer's country, product HS code, and origin. Displays real itemized breakdown: base duty + Section 301 + MPF + HMF. | `/api/analyze/stream`, `e2_tariff.calculate_tariff()` | Frontend: replace hardcoded calc with API call; add loading state | M | Both agreed |
| BL-002 | **Buyer product catalog is entirely hardcoded** | `StoreFront.tsx:8` imports from `data/products.ts` — 5 static products with pre-computed prices. No API call. Cannot scale. | **Both agreed (UX: B1/B8, Domain: #4)**: UX says "buyer surface cannot scale past 5 demo products"; Domain says "buyer surface disconnected from real engine." The `totalDuty` values in static data are frozen and won't reflect tariff changes. | Storefront fetches products from backend API. Prices computed dynamically from tariff engine. Products managed through admin or shipper catalog. | Product catalog API exists; tariff engine exists | Backend: product listing endpoint for buyer; Frontend: API integration | L | Both agreed |
| BL-003 | **Broker resolution steps are display-only text** | `BrokerDashboard.tsx:474-482`, `EntryDetail.tsx:456-475` — Resolution steps render as `<li>` elements with no click handlers. "Verify entity identity against BIS/OFAC lists" is passive text. | **Both agreed (UX: P0 #1, Domain: implicit)**: UX identifies this as the #1 issue: "Brokers can't act on AI recommendations." Each step maps to an existing tool (`screen_entity`, `draft_communication`, `resolve_shipment_hold`). | Each resolution step is a button. "Verify entity identity" triggers `screen_entity` with pre-filled entity name. "Request documentation" triggers `draft_communication` with pre-filled context. Progress tracks completion. | `screen_entity`, `draft_communication`, `resolve_shipment_hold`, `request_shipper_documents` — all in `tools.py` | Frontend: convert `<li>` to action buttons with tool invocation | M | Both agreed |
| BL-004 | **`verify_classification` not surfaced as button** | `EntryDetail.tsx:1016-1024` — "Improve classification confidence" opens chat panel instead of calling the tool directly. The tool returns current vs. suggested HS code, GRI analysis, and confidence. | **UX-only (P0 #2)**: "Most critical broker tool hidden behind chat." This is the single most frequent broker action — verifying an HS code is correct before filing. | "Verify Classification" button in EntryDetail sidebar. Calls backend directly. Shows inline panel: current code vs. suggested, match indicator, confidence, GRI reasoning, "Reclassify" action if mismatch. | `verify_classification` tool in `tools.py:492-509` | Frontend: new inline panel component | M | UX-only |
| BL-005 | **`calculate_entry_fees` not surfaced; crude estimates shown** | `EntryDetail.tsx:1209-1239` — Falls back to `entry.declaredValue * 0.05`. The real tool calls `e2_tariff.calculate_tariff()` with Section 301, 232, AD/CVD. | **Both agreed (UX: P0 #3, Domain: E3)**: UX says "inaccurate financial data for decision-making"; Domain says "duty rate calculation is placeholder" with hardcoded chapter-level rates. | "Calculate Actual Fees" button next to estimated duty. Shows full breakdown: MFN base + Section 301 + IEEPA + AD/CVD + MPF + HMF. Also add "Generate Entry Summary" button using `/entries/{id}/summary`. | `calculate_entry_fees` tool, `/entries/{id}/summary` endpoint | Frontend: fee breakdown panel + entry summary button | M | Both agreed |
| BL-006 | **`draft_communication` has `[Message body]` placeholder** | `broker.py:2543` — Fallback for unrecognized purposes: `"Dear {company},\n\n[Message body]\n\nBest regards,\n[Broker Name]"`. | **UX-only (P0 #4)**: "Broken UX for catch-all communication purpose." The system has full shipment context (product, tracking number, missing docs) but outputs a template with a placeholder. | Fallback generates contextual content using shipment data. "Dear {company}, I'm reaching out regarding your shipment of {product} ({tracking_number}). We require the following documents..." LLM fallback for any purpose. | Shipment context is available in the endpoint; needs LLM integration for fallback | Backend: LLM-powered fallback generation | S | UX-only |
| BL-007 | **Regulatory Intel signals are all hardcoded** | `RegulatoryIntel.tsx:16-89` — `const DEMO_SIGNALS` with 6 static entries. No API call. No CRUD. Dates from 2025. | **Both agreed (UX: P5-1, Domain: S-12)**: UX says "#1 credibility problem for the entire platform"; Domain says "static regulatory data doesn't track current policy changes." | Signals fetched from backend API. Admin can add/edit/archive. Rate impacts computed from tariff data. Badge notification for new signals. | `get_regulatory_signals` tool exists; needs persistence layer | Backend: signal CRUD API; Frontend: API integration + admin UI | L | Both agreed |
| BL-008 | **Compliance Dashboard uses 100% fake data** | `ComplianceDashboard.tsx:12-81` — `generateDemoEntries()` creates 47 random entries client-side. "Real-time compliance monitoring" label on fake data. | **Both agreed (UX: P10-1, Domain: S-2)**: UX says "entirely hardcoded demo data"; Domain says "most critical domain accuracy issue." Real simulation data exists in DB. | Either merge into Control Tower or connect to real data via API. If kept, focus on compliance-specific views (audit history, rule adherence, compliance officer workflow). | Dashboard aggregator API exists; simulation data in DB | Frontend: replace `generateDemoEntries()` with API call | M | Both agreed |
| BL-009 | **Missing document "Generate" capability on Broker** | `EntryDetail.tsx:304-318` — Missing documents show "Upload" only. No "Generate" or "Request from AI" despite backend having document generation capability. | **UX-only (P0 #3)**: The `DocumentRequirements.tsx` pattern on Shipper surface proves Generate + Upload works. Broker surface only has Upload. | Replicate `DocumentRequirements.tsx` pattern: "Generate & Attach" (AI + PDF), "Upload" (manual), "Request from Shipper" (triggers `draft_communication`). | Document generation pipeline exists in `documents.py`; PDF generation works on Shipper | Frontend: port DocumentRequirements pattern to Broker | M | UX-only |
| BL-010 | **Missing ISF (Importer Security Filing) workflow** | No ISF status indicator anywhere despite being legally required for all ocean shipments (19 CFR 149). $5,000 penalty per violation. | **Domain-only (Broker: D2/X1, Shipper: #6)**: "Any platform without ISF tracking is incomplete for ocean freight." ISF must be filed 24h before vessel lading. The simulation's ISFActor generates ISF events but they're not surfaced. | ISF status badge on ocean entries. Filing deadline countdown. Late filing warnings. ISFActor events surfaced in shipment timeline. | ISFActor exists in simulation; events in DB | Backend: ISF status endpoint; Frontend: ISF badges + timeline | L | Domain-only |
| BL-011 | **Missing ADD/CVD (Antidumping/Countervailing Duty) data** | No data field for AD/CVD case numbers or deposit rates anywhere in the data model. AD/CVD can add 100-400%+ to duty. | **Domain-only (Broker: Q4/X2)**: "Wrong duties = post-liquidation demand letters." Chinese steel, aluminum, solar panels all subject. CustomsActor HAS ADD/CVD case matching but it's not surfaced in UI. | ADD/CVD flag on entries. Case number display. Deposit rate shown separately from base duty. Warning badge on queue cards for affected products. | `CustomsActor` has ADD/CVD matching with real case numbers | Backend: ADD/CVD data model; Frontend: flags + display | L | Domain-only |

### P1 — Backend Capabilities Not Surfaced At All

| ID | Title | What's Wrong | Why It Matters | What It Should Look Like | Backend Available | Changes | Effort | Perspective |
|---|---|---|---|---|---|---|---|---|
| BL-012 | **Exception actions all route to chat** | `ExceptionActions.tsx:198-201` — Every action button calls `onOpenAssistant(action.prompt(...))`. "Request Traceability Docs", "Search CROSS Rulings", "File Protest" — all just open chat. | **Both agreed (UX: P3-1, Domain: implied)**: UX says "none invoke backend tools directly"; backend HAS screening, classification, resolution APIs. The exception resolution screen can't resolve exceptions. | Action buttons invoke backend APIs directly. "Request Traceability Docs" generates and sends email. "Screen Entity" calls `screen_entity` and shows results inline. Status transitions: Resolve/Escalate/Assign/Dismiss. | `screen_entity`, `draft_communication`, `resolve_shipment_hold` tools | Frontend: direct API calls instead of chat routing | M | Both agreed |
| BL-013 | **No exception status transitions** | `ExceptionResolution.tsx:352-603` — No way to change exception status. It's entirely read-only. An exception resolution screen that can't resolve. | **UX-only (P3-3)**: "No way to change exception status (resolve, escalate, dismiss, assign)." | Status buttons: Resolve (with confirmation + notes), Escalate (with reason), Assign (to broker/team), Dismiss (with justification). Full audit trail per transition. | Resolution endpoints exist | Frontend: status transition UI; Backend: status update API if needed | M | UX-only |
| BL-014 | **No search in Broker queue** | `BrokerQueue.tsx:308-338` — Only status and priority dropdowns. No text search by entry number, company, product, or shipment ID. | **Both agreed (UX: P1 #5, Domain: Q1)**: UX says "critical when a broker gets a call about a specific entry"; Domain says "sort by deadline/urgency, not just created date." | Search input querying across entry number, company name, product, reference numbers. Sort by: days waiting, value, priority, deadline. | `search_by_reference` tool in `tools.py:651-668` | Frontend: search bar + sort controls | S | Both agreed |
| BL-015 | **"Contact Shipper" button is a no-op** | `ExceptionQueue.tsx:379-384` — `onClick` only calls `e.stopPropagation()`. Button does literally nothing. | **UX-only (P1-5)**: A dead button in the primary ops dashboard undermines trust. | Button generates pre-filled email template with entry details, required documents, and deadline. Uses `draft_communication` backend. | `draft_communication` endpoint | Frontend: modal with draft communication flow | S | UX-only |
| BL-016 | **No simulation control panel UI** | Backend has full simulation API (start/stop/pause/resume/reset, actor config, clock control). No frontend for any of it. "Ask Assistant" is the only entry point. | **Both agreed (UX: IA-2, Domain: Part 4 #6)**: UX says "no UI to start/stop/configure simulations"; Domain provides detailed actor-by-actor assessment showing rich configurable simulation. | Simulation Control Panel: start/stop/pause/resume, scenario seed selection, actor parameter sliders, speed control (1x/5x/10x), data reset. Real-time event stream. | Full simulation API at `/api/simulation/start|stop|pause|resume|reset` | Frontend: new Simulation Control Panel screen | L | Both agreed |
| BL-017 | **Compose modal loses entry context** | `Communications.tsx:99-286` — When opened from "New Message" button, neither `defaultShipmentId` nor `defaultPurpose` is provided. AI Draft disabled without both. | **UX-only (P1 #6)**: "AI Draft is always disabled when opening from Communications page directly." | When no shipment context, show shipment selector dropdown (from broker's queue). Selecting a shipment enables AI Draft. | Broker queue data is available | Frontend: shipment selector in compose modal | S | UX-only |
| BL-018 | **Risk flag resolution steps passive in EntryDetail** | `EntryDetail.tsx:456-475` — Same issue as BL-003 but in entry context. Resolution steps are numbered text with no action buttons. | **UX-only (P1 #7)**: Same pattern as dashboard alerts — passive text where actions should be. | Same as BL-003: each step becomes a button invoking the appropriate tool. | Same tools as BL-003 | Frontend: action buttons on risk flag cards | S | UX-only |
| BL-019 | **No AI draft for CF-29 protests** | `CBPResponses.tsx:276-289` — "AI Draft" button only shown when `!isCF29`. CF-29 protests require careful legal drafting. | **Both agreed (UX: P1 #8, Domain: C5)**: UX says "complex legal writing with no AI assistance"; Domain says "protest form needs specific fields: protest number, category, legal authority." | AI Draft enabled for CF-29 with protest-specific template. Protest form includes required CBP Form 19 fields. | `draft_cf28_response` tool can be extended | Backend: CF-29 draft capability; Frontend: enable button + form fields | M | Both agreed |
| BL-020 | **Missing docs sidebar is display-only** | `EntryDetail.tsx:1290-1303` — "Missing Documents" bullet points with no actions. | **UX-only (P1 #6)**: "No action pathway from identified gap to resolution." | Per-doc: (1) click to scroll to checklist item, (2) "Upload" button, (3) "Generate" if system-generatable. "Request All" button to draft combined request. | Document generation pipeline | Frontend: action buttons + scroll-to-item | S | UX-only |
| BL-021 | **Held shipments show no hold context** | `BrokerDashboard.tsx:758-810` — Held shipments in unassigned list show red badge but only "Claim" action. No hold type, no cage status preview. | **UX-only (P1 #10)**: "Broker needs to know WHY it's held before claiming." | Hold type badge (entity_screening, UFLPA, PGA). One-line cage status. Hover/expand for resolution path preview. | `get_cage_status` tool, hold_type data in backend | Frontend: preview data on held entries | S | UX-only |
| BL-022 | **No cross-screen workflow linking** | Exception Resolution doesn't link to Entity Screening (for DPS), Product Analysis (for classification), or Regulatory Intel (for tariff holds). Each tool is an island. | **Both agreed (UX: IA-1, Domain: implied)**: Ops director's ideal flow: click GO alert → Exception Resolution → "Screen Entity" → Entity Screening pre-filled → result feeds back to exception. | Deep links between screens: DPS exception → Entity Screening pre-filled. Classification dispute → Product Analysis pre-filled. Tariff hold → Regulatory Intel with affected signal highlighted. | All screens exist; need URL parameter passing | Frontend: deep-link navigation with context passing | M | Both agreed |
| BL-023 | **Orders & Entries missing pagination** | `PlatformOrders.tsx:173-226` — All 1,958 orders rendered at once. No pagination, no filters, no search. | **Both agreed (UX: P8-2, Domain: S-6)**: UX says "performance problem"; Domain says "orders stuck in Draft with no duty data." | Server-side pagination, search, status/date/origin filters. Fix the order-to-duty pipeline so orders progress past Draft. | Pagination patterns exist in BrokerQueue | Frontend: pagination + filters; Backend: paginated endpoint | M | Both agreed |
| BL-024 | **Buyer shipping cost hardcoded $350** | `ProductPage.tsx:49`: `const shippingEstimate = 350` for all products. A 500kg tea shipment and a single sweater cost the same. | **Both agreed (UX: B6, Domain: #14)**: UX says "pricing fiction extends to logistics"; Domain lists missing MPF, HMF, broker fees, insurance, drayage. | Shipping estimate based on product weight, origin, destination, and transport mode. Include MPF, HMF, broker fees in the breakdown. | Weight data needed on products; shipping calc can use routing engine | Backend: shipping estimate endpoint; Frontend: dynamic display | M | Both agreed |
| BL-025 | **No Incoterms / valuation method on products or orders** | No field for terms of sale (FOB/CIF/EXW/DDP). Determines what's in declared value, who pays freight/insurance, risk transfer point. | **Domain-only (Shipper: #5)**: "Without Incoterms, duty calculation is unreliable. $10,000 FOB vs. $10,000 CIF produces different duty amounts." Critical for real customs entries. | Incoterms dropdown on product and order creation. Auto-adjusts declared value calculation based on selected term. | Tariff engine would use this for valuation | Backend: Incoterms field on data model; Frontend: dropdown + logic | M | Domain-only |
| BL-026 | **No Importer of Record (IOR) architecture** | Neither surface captures IOR EIN/tax ID, customs bond, Power of Attorney, or CBP Form 5106. IOR is required for ALL imports. | **Domain-only (Shipper: P0 #3, Broker: E9/M5)**: "Without an IOR, you cannot file a customs entry." Broker domain also flags missing POA tracking as P1. | IOR entity management: EIN/tax ID, bond number and type, POA status per importer. IOR selector on orders and entries. | Entity model needed | Backend: IOR entity management; Frontend: IOR UI across surfaces | XL | Domain-only |
| BL-027 | **Country code is free-text, not validated** | `AddProduct.tsx:246-252` — Country code field accepts any string. "CHN" instead of "CN" silently breaks routing. `maxLength={3}` allows 3 chars but ISO codes are 2. | **Both agreed (UX: S7, Domain: implied)**: UX says "typos silently break routing analysis." Affects AddProduct, CreateOrder, ShipperProductDetail, buyer destination. | Searchable ISO-2 country picker with flag icons across all surfaces. Validate on input, reject invalid codes. | Country list available in backend | Frontend: country picker component, used globally | S | Both agreed |
| BL-028 | **Error handling silently swallowed across Shipper** | `ShipperCatalog.tsx:21` `.catch(() => {})`, `AddProduct.tsx:88-90` catch does nothing, `CreateOrder.tsx:83-85` silent, `ShipperProductDetail.tsx:101-104` sets flag but no message. | **UX-only (S2/S8/S22)**: "User sees blank/frozen UI on failure with no recovery path." | Global toast/notification system. Network errors show retry button. Form submission errors show specific message. Single implementation fixes 5+ screens. | N/A | Frontend: global error handling + toast system | S | UX-only |
| BL-029 | **CF-28 responses have no document attachment** | `CBPResponses.tsx:210-341` — Modal has text area but no file upload. Guidance suggests "Certificate of origin, Manufacturing records" but can't attach them. | **Both agreed (UX: P2, Domain: C4 P1)**: Domain says "Real CF-28 responses almost always require documentary evidence." Backend schema has `attachments: list[str]` but UI doesn't expose it. | Document attachment area below textarea. Pre-populate with suggested docs from guidance. Allow upload or link to already-uploaded shipment documents. | Backend schema supports attachments | Frontend: file upload in response modal | S | Both agreed |
| BL-030 | **Missing critical CF-7501 fields** | Of 25 key fields on CBP Form 7501, 12 are missing or fabricated: MID, surety code, export date, IT number, AD/CVD case number, SPI, relationship indicator, net quantity, entered value, and more. | **Domain-only (Broker: E1)**: "An advisor with customs brokerage experience would flag these as signs the platform hasn't been tested with real entry data." | Add missing fields to EntryDetail display. Prioritize: MID, Entered Value, SPI, export date, gross weight. Use real filer code prefix for entry numbers instead of random digits. | Data model can accommodate via JSONB | Backend: data model expansion; Frontend: field display | L | Domain-only |
| BL-031 | **Analysis improvement steps are passive text** | `EntryDetail.tsx:881-890` — `improvementSteps` renders as bullet points. "Upload Certificate of Origin" should scroll to checklist item. "Request lab results from shipper" should trigger document request. | **UX-only (P1)**: Every piece of displayed information should connect to an action. These steps are the system telling the broker what to do but giving no way to do it. | Each step is clickable: "Upload Certificate of Origin" scrolls to checklist item + highlights. "Request lab results" opens draft communication. | Document checklist exists in same screen | Frontend: click handlers + scroll-to + action triggers | S | UX-only |

### P2 — Passive Information Without Actionable Next Steps

| ID | Title | What's Wrong | Why It Matters | What It Should Look Like | Backend Available | Changes | Effort | Perspective |
|---|---|---|---|---|---|---|---|---|
| BL-032 | **Buyer price breakdown math doesn't add up** | `ProductPage.tsx:49-51` — `buyerPrice` is a separate hardcoded number. Breakdown components (base + duty + tax + shipping) don't sum to it. | **UX-only (B7)**: "Financial inconsistency destroys trust." When buyer sees breakdown, the numbers should be verifiable. | Price = base + computed duty + computed tax + computed shipping. No separate hardcoded total. Math must be verifiable. | Tariff engine provides real numbers | Frontend: compute total from components | S (with BL-001) | UX-only |
| BL-033 | **No "Run Compliance Check" button** | `check_compliance` tool (tools.py:158-179) runs PGA requirements, DPS screening, UFLPA assessment. Not surfaced as button anywhere in EntryDetail. | **UX-only**: A "Run Compliance Pre-Check" before submission would catch issues early. Currently brokers can only access via chat. | "Run Compliance Pre-Check" button in EntryDetail. Shows results inline: PGA agencies triggered, DPS matches, UFLPA flags. Gates the "Submit to CBP" button. | `check_compliance` tool | Frontend: compliance check panel + button | M | UX-only |
| BL-034 | **Sidebar insights use crude duty estimates** | `EntryDetail.tsx:1214-1219` — `insights?.estimatedDuty ?? entry.declaredValue * 0.05` — 5% fallback misleading for 301 (25%) or AD/CVD entries. | **UX-only (P2)**: "Misleading for entries with Section 301 or AD/CVD duties." | Always call `calculate_entry_fees` on load. Show accurate breakdown. Remove 5% fallback. | `calculate_entry_fees` tool | Frontend: API call on load | S (with BL-005) | UX-only |
| BL-035 | **Hold details banner has no action button** | `EntryDetail.tsx:1262-1288` — Shows storage costs and GO deadline but no action. | **UX-only (P2)**: "Should have a 'Begin Resolution' button." | "Begin Resolution" button opens resolution wizard or navigates to first resolution step. Shows resolution path inline. | `BrokerIntelligenceService.get_resolution_path()` | Frontend: resolution button + navigation | S | UX-only |
| BL-036 | **KPI values not clickable in Control Tower** | `PlatformPulse.tsx:130-163` — Numbers are display-only. "47.8K Entries" should link to entries list. "3.8% Hold Rate" should link to held entries. | **UX-only (P1-1)**: Dashboard should be a navigation hub, not just a display. | Each KPI card clickable, navigating to filtered view of the underlying data. | Views exist for all metrics | Frontend: click handlers + navigation | S | UX-only |
| BL-037 | **Trend values in KPIs are hardcoded** | `PlatformPulse.tsx:33,38,43,48,53` — "+8%", "+1.2%", etc. are static strings, not computed from historical data. | **Both agreed (UX: P1-2, Domain: implied)**: "Misleading in a live dashboard." | Compute trends from period-over-period data (current vs. previous month/week). | Historical data available in DB | Backend: trend computation endpoint | M | Both agreed |
| BL-038 | **Exception Queue missing cargo exceptions** | ExceptionsActor generates shortage (1.5%), overage (0.5%), damage (2%), misdeclared weight (3%) — but Exception Resolution only shows compliance holds. | **Domain-only (S-7)**: "Cargo exceptions are the #1 daily workflow for ops teams." 7.5% of all shipments generate cargo exceptions that are invisible. | Add cargo exception categories to Exception Resolution queue. Show alongside compliance holds with appropriate priority. | Events exist in DB from ExceptionsActor | Frontend: expand exception type filter; Backend: include cargo events in query | M | Domain-only |
| BL-039 | **No search/filter on Active Shipments** | `ActiveShipments.tsx:80-423` — Only status and mode filters. No text search, no company filter, no date range. | **UX-only (P2-2)**: "For 100+ shipments this becomes unmanageable." | Full-text search across product, company, tracking number. Date range picker. Company filter dropdown. | Data available via API | Frontend: search + filter controls | S | UX-only |
| BL-040 | **No assignment model for exceptions** | `ExceptionResolution.tsx:352-603` — No concept of who is working on each exception. No "Assign to me" or "Assign to broker." | **UX-only (P3-4)**: Ops teams need ownership to prevent duplicate work. | Assignment system: "Assign to me", "Assign to [broker]", owner avatar per exception. Filter by assigned/unassigned. | Needs assignment field on exception data | Backend: owner field; Frontend: assignment UI | M | UX-only |
| BL-041 | **No entity screening history/audit log** | `EntityScreening.tsx:9-150` — Every screening is ephemeral. No log of past screenings. Critical for compliance audits. | **Both agreed (UX: P6-1, Domain: S-10)**: "Operators need to demonstrate they screened parties." Compliance audits require evidence of screening at booking time. | Screening history log: timestamp, entity, result, screened by. Searchable. Exportable for audit. | Needs persistence layer | Backend: screening history table; Frontend: history view | M | Both agreed |
| BL-042 | **No batch entity screening** | `EntityScreening.tsx:9-150` — One entity at a time. Order with 5 parties (shipper, consignee, manufacturer, notify, ultimate consignee) requires 5 separate screens. | **UX-only (P6-2)**: Multi-party screening is the norm, not the exception. | Batch screening: paste list or upload CSV. Pre-fill from exception context when navigating from DPS exceptions. | `screen_entity` tool handles one at a time; batch wrapper needed | Backend: batch endpoint; Frontend: batch UI | M | UX-only |
| BL-043 | **"Req Docs" button loses entry context** | `BrokerQueue.tsx:121-130` — Navigates to `/broker/messages` without context. Broker starts from scratch. | **UX-only (P2)**: "The broker loses the entry context." The `draft_communication` endpoint accepts `shipment_id` and `purpose`. | "Req Docs" opens pre-filled draft modal using `draft_communication` with entry's `shipment_id` and `purpose: "request_missing_documents"`. | `draft_communication` endpoint at `broker.py:2461` | Frontend: modal with context passing | S | UX-only |
| BL-044 | **No deadline/urgency info on queue cards** | `BrokerQueue.tsx:180-185` — Only "days since arrival". No GO deadline, no storage costs, no cascade warnings. | **UX-only (P2)**: Dashboard work plan has this data but queue cards don't. Broker can't prioritize from queue view. | GO deadline countdown badge. Storage cost indicator. CF-28 deadline if applicable. | Data available from backend | Frontend: additional badges on queue cards | S | UX-only |
| BL-045 | **Work plan items navigate away instead of inline** | `BrokerDashboard.tsx:155-175` — "Approve" navigates to `/broker/entries/{id}`. For simple actions, should act from dashboard. | **UX-only (P2)**: "Approve" already exists inline in My Work table (line 241). Work plan should match. | Work plan "approve" triggers inline confirmation micro-modal. "Respond CF-28" opens response modal directly. | Approval endpoint exists | Frontend: inline action modals | S | UX-only |
| BL-046 | **Buyer checkout has no destination country** | `CheckoutPrice.tsx:24-42` — Uses hardcoded US destination. No country selector. Tariff analysis meaningless without knowing destination. | **Both agreed (UX: B14, Domain: #23)**: UX says "shipped 'nowhere'"; Domain says "duty rates are destination-specific." | Checkout includes country selector. Duty recalculates on country change. Shows accurate per-country breakdown. | Analysis endpoint accepts destination | Frontend: country selector + recalculation | S (with BL-001) | Both agreed |
| BL-047 | **Tariff analysis runs after purchase, not before** | `CheckoutPrice.tsx:24-42` — `handlePurchase()` calls `analyze()` for each cart item AFTER buyer commits. Price buyer paid was based on fake 5% calc. | **Both agreed (UX: B15, Domain: #1)**: "The analysis that should inform pricing happens too late." Should run at browse/cart time. | Analysis runs when product is viewed or added to cart. Buyer price computed from real analysis. Cart shows real duty breakdown per item. | Analysis endpoint exists | Frontend: move analysis to product view/cart add time | M (with BL-001) | Both agreed |
| BL-048 | **No ETA or ISF deadline on Shipper shipments** | No estimated departure, arrival at customs, or delivery date anywhere in shipment detail or list. ISF deadline (24h before vessel departure) not tracked. | **Domain-only (Shipper: #6, #7)**: "ETA is the #1 thing every importer checks every day." Also missing entry number and entry type. | ETA displayed prominently on shipment cards and detail. ISF filing deadline countdown for ocean. Entry number and type shown when assigned by broker. | Routing engine has transit times; ISFActor tracks deadlines | Frontend: ETA display; Backend: ETA computation from route | M | Domain-only |
| BL-049 | **Product Analysis results not linked to exceptions** | `ProductAnalysis.tsx:15-248` — Re-classification results can't be applied to classification dispute exceptions. Tools are siloed. | **UX-only (P4-3)**: "If I re-classify a product here, there's no way to apply that result to a classification dispute exception." | "Apply to exception" button when classification differs from disputed entry. Cross-links to Exception Resolution. Analysis history sidebar with recall. | Classification data available | Frontend: cross-linking + history | M | UX-only |
| BL-050 | **No product count, search, or readiness indicator on Shipper catalog** | `ShipperCatalog.tsx` — No search, no filter, no compliance readiness per product card. | **UX-only (S1/S3/S5)**: "Can't scan for products needing attention." Catalog grows but discovery doesn't scale. | Each card shows readiness dot (green/yellow/red). Search by name/HS code. Product count in header. "Add Product" button in header, not footer. | Last analysis data available per product | Frontend: readiness indicator + search | S | UX-only |
| BL-051 | **No duty estimate on Create Order page** | `CreateOrder.tsx:220-243` — Order summary only shows product value. No duty preview before submission. | **Both agreed (UX: S21, Domain: implied)**: "Shippers make decisions based on cost. Orders created without cost visibility." | "Quick Estimate" button or inline duty preview as line items are added. Show approximate total duty alongside product total. | Analysis endpoint exists | Frontend: inline estimate + API call | M | UX-only |
| BL-052 | **CBP Responses not linked to Entry Detail** | `CBPResponses.tsx:76-84` — Entry number shown but not clickable. Can't navigate to entry for context review before drafting response. | **UX-only (P2)**: "Brokers need to review the full entry before drafting a response." | Entry number is clickable link to `/broker/entries/{entryId}`. "View Entry" button on response card. | Navigation only | Frontend: link + button | S | UX-only |
| BL-053 | **No message threading in Communications** | `Communications.tsx:46-97` — Flat card list. 10 messages about one shipment mixed with others. | **Both agreed (UX: P2, Domain: M2)**: "Broker handling 20 entries needs to see all communications for a specific entry grouped together." | Group by shipment/thread. Conversation count per thread. Expand to see full exchange. | Needs threading model | Backend: thread grouping; Frontend: threaded UI | M | Both agreed |
| BL-054 | **Entry number format is fabricated** | `broker.py:1103-1107` — Random 3-7-1 digits. Real format: Filer Code (3 chars, broker's assigned code) + Entry Number (7 digits) + Check Digit (computed). | **Domain-only (Broker: Q5)**: "Random digits instead of the broker's filer code" is an immediate "this is fake" signal. | Use broker's filer code prefix. Compute check digit per CBP algorithm. Reference ACE formatting. | Algorithm is public | Backend: entry number generation fix | S | Domain-only |
| BL-055 | **"Held" maps to "Rejected" in backend** | `broker.py:523` maps `held` to `status_counts.get("rejected", 0)`. These are fundamentally different states. | **Domain-only (Broker: D3)**: "Held means CBP is examining/detaining; Rejected means the entry was denied." | Separate held and rejected states. Add specific status for CBP examination vs. denied entry. | Status field in data model | Backend: status distinction | S | Domain-only |
| BL-056 | **Duplicate HS code display in ShipmentDetail** | `ShipmentDetail.tsx:385-463` vs `578-632` — Classification shown in "Analysis Results" AND again in separate "Classification" section. Same data rendered twice. | **UX-only (S30)**: "Confusing redundancy." | Consolidate into single classification section. Remove duplicate. | N/A | Frontend: remove duplicate section | S | UX-only |

### P3 — Missing Proactive Intelligence

| ID | Title | What's Wrong | Why It Matters | What It Should Look Like | Backend Available | Changes | Effort | Perspective |
|---|---|---|---|---|---|---|---|---|
| BL-057 | **No "entry readiness gate" before submission** | Submit button appears based on status alone, not actual completeness. No `check_entry_readiness` pre-flight. | **UX-only**: "Before showing 'Submit to CBP', run `check_entry_readiness` and show a pre-flight checklist." | Pre-flight modal before submission: document completeness, classification confidence, fee calculation status, compliance check status. Block submission if critical items incomplete. | `check_entry_readiness` tool | Frontend: pre-flight gate | M | UX-only |
| BL-058 | **No time range selector on Control Tower** | Dashboard shows "month" data but no way to change to weekly, quarterly, or custom. | **UX-only (P1-4)**: Ops director needs different views for different decisions. | Time range selector: Today / Week / Month / Quarter / Custom. All metrics recalculate. | Historical data in DB | Frontend: selector + API params | M | UX-only |
| BL-059 | **No bulk actions anywhere** | No bulk claim (BrokerDashboard), no bulk approve (BrokerQueue), no bulk select (Active Shipments, Orders, Exception Queue). | **UX-only (multiple)**: "5 separate API calls with 5 loading spinners" for claiming 5 shipments. | Multi-select checkboxes with batch action toolbar: Claim, Approve, Export, Assign, Escalate. | Batch endpoints may need creation | Frontend: multi-select + batch toolbar; Backend: batch endpoints | M | UX-only |
| BL-060 | **No sort options on Broker queue** | `BrokerQueue.tsx:356-365` — Queue in server-provided order only. No client-side sort. | **Both agreed (UX: P1, Domain: Q1)**: "Brokers need to sort by days waiting, declared value, priority, status, company name." | Sortable column headers. Default sort by urgency (GO deadline proximity + value at risk). | Data available | Frontend: sort controls | S | Both agreed |
| BL-061 | **HTS code not shown on Broker queue cards** | `BrokerQueue.tsx:152-159` — Cards show product, company, route, value — but not the HS/HTS code. | **Domain-only (Broker: Q3 P1)**: "The HS/HTS code is the single most important data point for a customs broker." | Show HTS code in monospace on queue card. Add classification confidence indicator (HIGH/MEDIUM/LOW). | Data available from entry | Frontend: add HTS display | S | Domain-only |
| BL-062 | **No post-release / liquidation tracking** | Platform treats "Released" as terminal state. Real lifecycle: Released → Liquidated (~314 days) → Protest window. | **Domain-only (Broker: D1/X3)**: "Missed protest deadlines = lost refund opportunities." Every broker tracks post-release. | Add pipeline stages: Released → Liquidated → Protest Window. Liquidation date estimate. Protest deadline countdown. | Needs status expansion | Backend: status additions; Frontend: pipeline stage display | L | Domain-only |
| BL-063 | **Resolution steps fallback is useless** | `ResolutionCenter.tsx:373` — When AI fails, shows only "Contact customs broker." Zero actionable value. | **UX-only (S33)**: Fallback should at least show common troubleshooting steps per hold type. | Per-hold-type fallback steps: UFLPA → "Gather supply chain documentation, Identify alternative sourcing." PGA → "Check agency-specific requirements." Etc. | Static content | Frontend: improved fallback content | S | UX-only |
| BL-064 | **No link from Resolution Center to shipment** | `ResolutionCenter.tsx:411-438` — Shows resolution steps but no "Go to Shipment" to act on them. | **UX-only (S34)**: "Can't take action on suggestions." | "View Shipment" button linking to `/shipper/shipments/{id}`. | Navigation only | Frontend: link button | S | UX-only |
| BL-065 | **ShipmentChat only on ShipmentDetail** | `ShipmentChat.tsx` — Not on OrderDetail, ProductDetail, or Resolution Center. | **UX-only (S38)**: "Useful tool locked to one screen." | Promote to global floating assistant across entire shipper surface. Context-aware based on current screen. | Chat API works with any context | Frontend: lift ShipmentChat to layout level | M | UX-only |
| BL-066 | **No conversation persistence in assistants** | Messages stored in Zustand but not persisted across page refreshes. Broker, Platform, and Shipper assistants all lose context. | **UX-only (multiple)**: "Conversation is lost" on refresh or navigation. | Persist conversations to backend or localStorage. Restore on return. Multiple conversation threads. | Needs persistence endpoint or localStorage | Frontend: persistence layer | M | UX-only |
| BL-067 | **PGA shown as single item, not per-agency** | Broker checklist has "PGA Documentation" as one item. Real PGA = per-agency: FDA, EPA, FCC, CPSC, APHIS, TTB. Each has different forms. | **Domain-only (Broker: E8)**: "Missing other government agency detail." An electronic device needs FCC + EPA, not just generic "PGA." | Expand PGA to per-agency items: FDA Prior Notice, EPA TSCA cert, FCC Declaration, CPSC cert, APHIS phytosanitary, TTB permit. Each with agency-specific forms. | PGAActor has per-agency data | Frontend: agency-specific checklist items | M | Domain-only |
| BL-068 | **ShipmentDetail is 12+ sections in one scroll** | `ShipmentDetail.tsx:137-681` — Hold banner, header, analysis, cargo, classification, tariff, compliance, waypoints, financials, HS codes, docs, timeline. Overwhelming. | **UX-only (S29)**: "Information overload, hard to find specific data." | Tab layout or accordion: Overview | Analysis | Documents | Timeline | Financials. Or sticky section nav sidebar. | N/A | Frontend: layout restructure | M | UX-only |
| BL-069 | **No signal creation/management for Regulatory Intel** | `RegulatoryIntel.tsx:166-500` — Read-only on hardcoded data. No admin CRUD. | **UX-only (P5-4)**: Goes with BL-007. Admin needs to add new signals, mark as superseded, update status. | Admin CRUD: create signal with affected HTS codes, legal authority, effective date. Edit status (proposed → confirmed → effective). Archive superseded. | Needs backend API | Backend: CRUD endpoints; Frontend: admin forms | M (with BL-007) | UX-only |
| BL-070 | **No quantity selector on buyer product page** | `ProductPage.tsx:43-46` — "Add to Cart" adds exactly 1 unit. | **UX-only (B9)**: "For B2B products like 500 aluminum brackets, this is wrong." | Quantity input with +/- buttons. Price recalculates with quantity. Weight-based shipping adjusts. | N/A | Frontend: quantity selector | S | UX-only |

### P4 — Polish, Consistency, Advanced Features

| ID | Title | What's Wrong | Why It Matters | What It Should Look Like | Backend Available | Changes | Effort | Perspective |
|---|---|---|---|---|---|---|---|---|
| BL-071 | **No keyboard shortcuts in EntryDetail** | No shortcuts for Approve, Submit, Open Assistant. Broker's primary workspace used 50+ times/day. | **UX-only (P3)**: Speed matters for high-volume brokers. | Ctrl+A: Approve, Ctrl+Enter: Submit, Ctrl+/: Open Assistant. Shortcut hints on buttons. | N/A | Frontend: keyboard handlers | S | UX-only |
| BL-072 | **Broker license number format wrong** | Shows "CHB CHB-4892" — CBP license numbers are 5 digits (e.g., "12345") issued by district. | **Domain-only (Broker: D4)**: Minor but a domain expert would notice. | Use real 5-digit format with district. | Static data fix | Frontend: format fix | S | Domain-only |

---

## 4. Simulation Realism

### Overall Score: 6.5/10

The simulation has **exceptional coverage** (19 actors modeling every major customs clearance workflow) but **critical disconnect** between probabilistic shortcuts and causal intelligence.

### Actor-by-Actor Assessment

| Actor | Score | Key Issue |
|---|---|---|
| **ShipperActor** | 7/10 | Tariff uses flat corridor rates (38.4% for ALL CN→US), not HS-code-aware |
| **CustomsActor** | 5/10 | Decisions are 100% dice rolls — `misclassified` flag never read, same STP probability for clean and bad shipments |
| **ComplianceEngineActor** | 3/10 | `run_real_engines` defaults False; classification confidence is `random.choice(["HIGH","HIGH","HIGH","MEDIUM"])` |
| **ResolutionActor** | 4/10 | Wait 1-5 days + roll 75% resolved — hold type has NO influence. UFLPA should be 6-12 months / 15% release |
| **FinancialActor** | 8/10 | MPF, HMF, broker fees, bond types, exam fees all correct |
| **CBPAuthorityActor** | 8/10 | Risk-based response probabilities, correct CF-28/CF-29 handling, protest modeling |
| **BrokerSimActor** | 8/10 | 5 realistic profiles, specialization matching, progressive workflow |
| **PGAActor** | 8/10 | Per-agency timelines, FDA Prior Notice, APHIS ISPM-15, TTB permits |
| **DemurrageActor** | 8/10 | Carrier-specific D&D, container type rates, appointment miss fees |
| **CarrierActor** | 7/10 | Mode-aware transit, delay injection, consolidation lifecycle |
| **ExceptionsActor** | 7/10 | Good exception types but flat rates regardless of product fragility |
| **DocumentsActor** | 8/10 | 7 discrepancy types, realistic rates, CF-28 trigger association |
| **ISFActor** | 8/10 | Ocean-only, $5K penalty, 3 mismatch types, amendment tracking |
| **TerminalActor** | 7/10 | Berth delays, chassis shortage with real pool names |

### Priority Simulation Fixes

**S-1: Replace corridor flat-rate duties with HS-code-aware calculation** (P1)
- `shipper.py:198`: `predicted_duty = round(declared_value * corridor.avg_duty_rate, 2)`
- All CN→US products get 38.4% regardless of HS code
- Fix: Chapter-level rate table at minimum (Chapter 85 from CN = 25% §301 + 20% IEEPA = 45%; Chapter 61 from MX = 0% USMCA-eligible)
- Impact: All financial aggregates across the entire platform become more realistic

**S-2: Make CustomsActor read the `misclassified` flag** (P1)
- `customs.py:258`: ShipperActor injects `misclassified` flag but CustomsActor ignores it
- Fix: Reduce STP probability for misclassified shipments (30% instead of 87%)
- Impact: Error injection becomes meaningful — currently defeated by random dice rolls

**S-3: Differentiate resolution timelines by hold type** (P1)
- `resolution.py:102-168`: All holds resolve in 1-5 days at 75% success
- Fix: UFLPA 30-180 days / 15% release. CF-28 5-30 days / 80% release. PGA per-agency (already partially modeled). Inspection 3-10 days / 60% release
- Impact: Operational reality reflected in dashboard metrics

**S-4: Wire ComplianceDashboard to simulation data** (P0 — same as BL-008)
- Replace `generateDemoEntries()` with API call to existing endpoints
- Impact: Entire screen becomes real instead of fake

**S-5: Surface cargo exceptions in Exception Queue** (P1 — same as BL-038)
- ExceptionsActor generates damage/shortage/overage/misdeclared events
- These are invisible in the Exception Resolution queue
- Impact: 7.5% of shipments' exceptions become visible

**S-6: Enable `run_real_engines` path** (P1)
- `ComplianceEngineActor` config defaults `run_real_engines=False`
- The real engine delegation path has dead imports
- Fix: Connect to actual classification/tariff/compliance engines OR make synthetic data follow causal rules (high-risk product → lower confidence)
- Impact: Analysis data becomes meaningful instead of random

**S-7: Two tariff engines produce inconsistent results** (P2)
- Product Analysis uses LLM tariff engine; simulation uses hardcoded corridor rates
- Same product analyzed in both paths gives different duty amounts
- Fix: Unify tariff calculation path or validate consistency

---

## 5. Implementation Roadmap

### Sprint 1: "Kill the Placeholders" (2 weeks)
**Theme**: Replace hardcoded/fake data with real intelligence. Highest credibility impact.

| ID | Item | Effort |
|---|---|---|
| BL-001 | Connect Buyer duty to real tariff engine | M |
| BL-002 | Buyer catalog from API, not static file | L |
| BL-006 | Fix draft_communication placeholder | S |
| BL-008 | Wire ComplianceDashboard to real data | M |
| BL-028 | Global error handling + toast system | S |
| BL-032 | Fix buyer price breakdown math | S |
| BL-046 | Buyer checkout destination selector | S |
| BL-047 | Move tariff analysis before purchase | M |
| BL-054 | Fix entry number format | S |
| BL-055 | Separate held vs. rejected status | S |

**Outcome**: Buyer surface shows real prices. Compliance Dashboard shows real data. No more placeholders in communications. Error handling works globally.

---

### Sprint 2: "Information → Action" (2 weeks)
**Theme**: Convert passive text into clickable actions. Highest workflow impact.

| ID | Item | Effort |
|---|---|---|
| BL-003 | Resolution steps become action buttons | M |
| BL-004 | verify_classification inline panel | M |
| BL-005 | calculate_entry_fees button + panel | M |
| BL-009 | Port DocumentRequirements to Broker | M |
| BL-012 | Exception actions invoke APIs directly | M |
| BL-013 | Exception status transitions | M |
| BL-015 | Fix Contact Shipper button | S |
| BL-018 | Risk flag action buttons | S |
| BL-020 | Missing docs sidebar actions | S |
| BL-031 | Improvement steps become clickable | S |

**Outcome**: Brokers and ops can act directly on every recommendation. Exception resolution actually resolves. 16 tools no longer hidden behind chat.

---

### Sprint 3: "Operational Depth" (2 weeks)
**Theme**: Add missing search/filter/sort, bulk operations, and cross-screen workflows.

| ID | Item | Effort |
|---|---|---|
| BL-014 | Broker queue search + sort | S |
| BL-016 | Simulation Control Panel | L |
| BL-022 | Cross-screen workflow linking | M |
| BL-023 | Orders pagination + filters | M |
| BL-027 | Country code picker (global) | S |
| BL-029 | CF-28 attachment upload | S |
| BL-036 | Clickable KPI cards | S |
| BL-039 | Active Shipments search/filter | S |
| BL-041 | Entity screening audit log | M |
| BL-059 | Bulk actions (claim, approve, export) | M |

**Outcome**: Platform usable for 100+ entries. Simulation controllable from UI. Tools connected across screens.

---

### Sprint 4: "Domain Credibility" (2 weeks)
**Theme**: Add missing regulatory data fields and domain workflows.

| ID | Item | Effort |
|---|---|---|
| BL-010 | ISF workflow + status indicators | L |
| BL-011 | ADD/CVD data fields + display | L |
| BL-019 | CF-29 AI draft + protest form | M |
| BL-025 | Incoterms on products/orders | M |
| BL-030 | Missing CF-7501 fields | L |
| BL-038 | Cargo exceptions in queue | M |
| BL-048 | ETA + ISF deadline on shipments | M |
| BL-053 | Message threading | M |
| BL-062 | Post-release / liquidation tracking | L |
| BL-067 | PGA per-agency breakdown | M |

**Outcome**: Platform passes domain expert scrutiny. ISF, ADD/CVD, and liquidation tracking present. CF-7501 fields substantively complete.

---

### Sprint 5: "Intelligence Layer" (2 weeks)
**Theme**: Simulation realism, proactive intelligence, polish.

| ID | Item | Effort |
|---|---|---|
| BL-007 | Regulatory Intel from API + admin CRUD | L |
| BL-026 | Importer of Record architecture | XL |
| BL-033 | Compliance check button + pre-flight gate | M |
| BL-037 | Computed KPI trend values | M |
| BL-057 | Entry readiness gate before submission | M |
| BL-065 | Global floating assistant | M |
| BL-066 | Conversation persistence | M |
| S-1 | HS-code-aware tariff in simulation | L |
| S-2 | CustomsActor reads misclassified flag | S |
| S-3 | Hold-type-aware resolution timelines | M |

**Outcome**: Simulation produces causal (not just random) outcomes. Regulatory intelligence is live. IOR architecture enables real entry filing. Assistants persist context.

---

## Appendix: Verification Notes

### Analyst Agreement Matrix

| Topic | UX Analyst | Domain Analyst | Agreement |
|---|---|---|---|
| Buyer 5% duty | P0 (B5) | P0 (#1) | **Full agreement** |
| Buyer static catalog | P0 (B1/B8) | P0 (#4) | **Full agreement** |
| Resolution steps passive | P0 (Dashboard + Entry) | Not flagged as P0 | **UX stronger** |
| Missing ISF workflow | Not flagged | P0 (D2/X1) | **Domain only** |
| Missing ADD/CVD | Not flagged | P0 (Q4/X2) | **Domain only** |
| Regulatory signals hardcoded | P0 (P5-1) | P2 (S-12) | **UX rates higher** |
| Compliance Dashboard fake | P0 (P10-1) | P0 (S-2) | **Full agreement** |
| Exception actions → chat | P1 (P3-1) | Not flagged | **UX only** |
| CF-28 no attachments | P2 | P1 (C4) | **Domain rates higher** |
| Entry number format | Not flagged | P2 (Q5) | **Domain only** |
| No IOR architecture | Not flagged | P0 (#3) | **Domain only** |
| DocumentRequirements pattern | Highlighted as gold standard | Praised as "strongest part" | **Full agreement** |
| Error handling swallowed | P1 (S2/S8/S22) | Not flagged | **UX only** |
| No Incoterms | Not flagged | P1 (#5) | **Domain only** |

### Backend Capability Verification

All referenced backend capabilities verified to exist in code:
- `verify_classification`: `tools.py:492-509` — returns `current_hs_code`, `ai_suggested_code`, `codes_match`, `confidence`, `gri_analysis`
- `calculate_entry_fees`: `tools.py:511-527` — calls `e2_tariff.calculate_tariff()`
- `check_compliance`: `tools.py:158-179` — runs PGA + DPS + UFLPA
- `screen_entity`: `tools.py:82` — denied party screening
- `draft_communication`: `broker.py:2461` — accepts `shipment_id` and `purpose`
- `check_entry_readiness`: `tools.py:474` — pre-submission validation
- `get_regulatory_signals`: `tools.py:213` — regulatory change alerts
- `identify_documents`: `tools.py:181-212` — deterministic document requirements
- `search_by_reference`: `tools.py:651-668` — cross-field search
- `resolve_shipment_hold`: `tools.py:328` — hold resolution workflow
- `get_cage_status`: `tools.py:633` — warehouse cage tracking
- `/entries/{id}/summary`: `broker.py:1566` — 7501-equivalent entry summary
- Simulation API: start/stop/pause/resume/reset endpoints exist
- Document generation pipeline: `documents.py` with 8 document types + PDF generation
