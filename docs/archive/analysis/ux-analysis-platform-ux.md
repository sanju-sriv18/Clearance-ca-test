# Platform Surface UX Analysis
**Analyst**: Senior Product/UX Designer
**Date**: 2026-02-07
**Surface**: Platform (Ops/Admin Control Center)
**Screens analyzed**: 12 screens, 20+ components

---

## Executive Summary

The Platform surface is a strong ops dashboard with real, data-driven KPIs, well-structured exception management, and functional AI-powered tools. The Control Tower is the standout screen: it presents the right information hierarchy for a 6 AM ops review. However, critical gaps emerge around **actionability** (most workflows terminate at the AI assistant rather than direct tool invocation), **data authenticity** (two screens use hardcoded demo data), and **missing capabilities** (no simulation control panel, no scenario builder, no bulk operations).

The platform's information architecture follows a clear three-tier model: Command Center (dashboards) > Intelligence (analysis) > Tools (utilities). This is sound, but the screens within "Intelligence" and "Tools" operate as isolated silos with no cross-linking between related workflows.

---

## Screen-by-Screen Analysis

---

### 1. Control Tower (`ControlTower.tsx`)

**Screenshot**: `01-control-tower.png`, `11-compliance-dashboard.png` (two variants visible)

#### What works well
- **KPI bar is excellent** (`PlatformPulse.tsx:26-72`): 8 metrics in a horizontal strip with trend indicators. Entries, Cleared, STP Rate, Avg Clear Time, Hold Rate, Duty Collected, FTA Savings, Active Alerts. This is exactly what a trade ops director scans first.
- **Live ticker** (`PlatformPulse.tsx:74-121`): Synthesized from real entry data. GO deadlines, LFD alerts, exam statuses, UFLPA detentions. Not fake - computed from `usePipelineStore` entries.
- **Exception Queue** (`ExceptionQueue.tsx:51-103`): Expandable rows with transport references, cage location, dwell time, storage costs, GO deadlines, LFD alerts. Rich operational detail in compact space.
- **Risk Summary cards** (`RiskSummary.tsx:19-99`): Clickable UFLPA/DPS/PGA/Classification cards navigate to filtered shipment views. True drill-down.
- **Ops Activity Feed** (`OpsActivityFeed.tsx:44-241`): Synthesized timeline from entry data. Severity-sorted, reference-tagged, relative timestamps.
- **Corridor click drills down** (`ControlTower.tsx:44-46`): Trade lane rows navigate to filtered Active Shipments.

#### Issues found

| # | Issue | Severity | File:Line | Details |
|---|-------|----------|-----------|---------|
| P1-1 | KPI values not clickable | P2 | `PlatformPulse.tsx:130-163` | KPIs show numbers but none are clickable to drill into the underlying data. "47.8K Entries" should link to the entries list; "3.8% Hold Rate" should link to held entries. |
| P1-2 | Trend values are hardcoded | P2 | `PlatformPulse.tsx:33,38,43,48,53` | "+8%", "+1.2%", "-0.3", "+0.4%" are static strings, not computed from historical data. Misleading in a live dashboard. |
| P1-3 | Industry context footer is static | P3 | `ControlTower.tsx:69-72` | "$2.1T at risk... 72% entry errors" is static marketing copy baked into the dashboard. Should either be removed or computed from real data. |
| P1-4 | No time range selector | P2 | `ControlTower.tsx:33-75` | Dashboard shows "month" data but no way to change to weekly, quarterly, or custom date range. |
| P1-5 | Exception "Contact Shipper" button is a no-op | P1 | `ExceptionQueue.tsx:379-384` | The "Contact Shipper" button's `onClick` only calls `e.stopPropagation()`. It literally does nothing. |
| P1-6 | ExceptionQueue hardcoded portfolio name | P3 | `ExceptionQueue.tsx:64` | "AutoParts Global portfolio" is hardcoded. Should be dynamic per account/portfolio. |
| P1-7 | ExceptionQueue hardcoded evidence timelines | P3 | `ExceptionQueue.tsx:401-405` | Hardcoded `daysMap` for specific entry IDs: `"CLR-2025-00044": "evidence due 22d"`. Should be computed from entry data. |

#### Ideal experience
- Every KPI card is clickable, drilling to the filtered list
- Time range selector (Today / Week / Month / Quarter)
- Trend values computed from actual period-over-period data
- Exception "Contact Shipper" generates a pre-filled email template with entry details

---

### 2. Active Shipments (`ActiveShipments.tsx`)

**Screenshot**: `02-active-shipments.png`

#### What works well
- **Status tabs with counts**: Booked, In Transit, At Customs, Inspection, Held, Cleared, Delivered
- **Transport mode filter pills**: Air/Ocean/Ground with counts
- **Corridor and risk filter chips**: Deep-linkable via URL params. Removable chips.
- **Clickable rows** navigate to shipment detail (`ActiveShipments.tsx:365`)
- **Error state with retry** (`ActiveShipments.tsx:300-318`)

#### Issues found

| # | Issue | Severity | File:Line | Details |
|---|-------|----------|-----------|---------|
| P2-1 | Empty state shows 0 shipments with skeleton placeholders | P1 | `02-active-shipments.png` | Screenshot shows "0 shipments across all corridors" with gray skeleton rectangles. No explanation of why there are 0 shipments, no guidance to seed data or run simulation. |
| P2-2 | No search/filter by product, company, or tracking # | P2 | `ActiveShipments.tsx:80-423` | Only status and mode filters. No text search, no company filter, no date range. For 100+ shipments this becomes unmanageable. |
| P2-3 | No bulk actions | P2 | `ActiveShipments.tsx:320-416` | No way to select multiple shipments for bulk release, escalation, or export. |
| P2-4 | No sort controls on table headers | P3 | `ActiveShipments.tsx:330-354` | Table headers are not sortable. Can't sort by duty amount, date, or status. |
| P2-5 | No pagination | P2 | `ActiveShipments.tsx:357-413` | All shipments rendered at once. With simulation generating hundreds, this will cause performance issues. |

#### Ideal experience
- Full-text search across product, company, tracking number
- Sortable table columns
- Bulk select with actions (export, escalate, assign)
- Pagination or virtual scrolling for large datasets
- Empty state with a "Run simulation to generate shipments" CTA

---

### 3. Exception Resolution (`ExceptionResolution.tsx`)

**Screenshot**: `03-exception-resolution.png`

#### What works well
- **Two-panel layout**: Left queue, right detail. Classic master-detail pattern.
- **Unified exception list** (`ExceptionResolution.tsx:392-404`): Merges pipeline entries AND shipment exceptions, deduplicates, sorts by severity. Excellent data unification.
- **Severity-based visual hierarchy**: Red dots for critical, amber for high, with priority labels.
- **Deep-link support** (`ExceptionResolution.tsx:412-421`): `?select=` query param auto-selects exception.
- **Contextual action buttons** (`ExceptionActions.tsx:44-175`): Type-aware actions for UFLPA, PGA, Classification, DPS, and General exceptions. 4 actions per type.
- **Recommended next steps** (`ExceptionResolution.tsx:832-854`): Numbered action list with contextual content based on compliance flags.
- **Documentation status** (`ExceptionResolution.tsx:737-830`): Shows submitted vs. missing documents, intelligently derived from compliance flags when explicit lists aren't available.

#### Issues found

| # | Issue | Severity | File:Line | Details |
|---|-------|----------|-----------|---------|
| P3-1 | ALL action buttons route to chat assistant | P1 | `ExceptionActions.tsx:198-201` | Every action button calls `onOpenAssistant(action.prompt(...))`. "Request Traceability Docs", "Search CROSS Rulings", "File Protest" -- they ALL just open the chat panel with a pre-filled prompt. None invoke backend tools directly. The backend HAS these tools (screening, classification, resolution APIs). |
| P3-2 | Timeline is fake/static | P2 | `ExceptionResolution.tsx:857-901` | Timeline only shows two events, both with the same timestamp (`exception.createdAt`). No real timeline data from the backend. |
| P3-3 | No status transitions | P1 | `ExceptionResolution.tsx:352-603` | No way to change exception status (resolve, escalate, dismiss, assign). It's read-only. An exception resolution screen that can't resolve exceptions. |
| P3-4 | No assignment/ownership model | P2 | `ExceptionResolution.tsx:352-603` | No concept of who is working on each exception. No "Assign to me", no "Assign to broker" capability. |
| P3-5 | No filtering/search in exception list | P2 | `ExceptionResolution.tsx:461-523` | Left panel has no filters. Can't filter by type (UFLPA only), severity, port, or date. |
| P3-6 | "Open in Assistant" button duplicates ExceptionActions | P3 | `ExceptionResolution.tsx:580-592` | Full-width "Open in Assistant" button at the bottom does the same thing as the action buttons above it. Redundant. |

#### Ideal experience
- Action buttons invoke backend APIs directly (screen entity, search CROSS, submit document request)
- Status transition buttons: Resolve / Escalate / Assign / Dismiss with confirmation dialog
- Assignment system with owner avatar/name per exception
- Filter panel: by type, severity, port, date range
- Real timeline from shipment events backend

---

### 4. Product Analysis (`ProductAnalysis.tsx`)

**Screenshot**: `04-product-analysis.png`

#### What works well
- **Live AI-powered analysis**: Uses `useAnalysis` hook with SSE streaming. Real classification, tariff calculation, compliance screening.
- **Progressive disclosure**: Form fills left panel; results expand right panel with animation.
- **Description quality check** (`ProductAnalysis.tsx:111-118`): Pre-flight check on product description quality.
- **Reasoning chain** (`ReasoningChain.tsx`): Shows AI's classification reasoning steps.
- **Tariff stack** (`TariffStack.tsx`): Layered tariff visualization.
- **Add to Catalog CTA** (`ProductAnalysis.tsx:196-227`): After analysis, prompts to save product to platform catalog. Good workflow continuity.
- **Demo product chips** (visible in screenshot): Quick-fill buttons for testing.

#### Issues found

| # | Issue | Severity | File:Line | Details |
|---|-------|----------|-----------|---------|
| P4-1 | No history of previous analyses | P2 | `ProductAnalysis.tsx:15-248` | Each analysis is ephemeral. No list of recent analyses, no save/recall. The "Add to Catalog" partially addresses this but it's easy to lose results. |
| P4-2 | No comparison mode | P3 | `ProductAnalysis.tsx:15-248` | Can't analyze two products side by side or compare classification results. |
| P4-3 | Results not linked to exception resolution | P2 | `ProductAnalysis.tsx:15-248` | If I re-classify a product here, there's no way to apply that result to a classification dispute exception. The tools are siloed. |
| P4-4 | "Add to Platform" error silently ignored | P2 | `ProductAnalysis.tsx:59` | `catch { /* ignore */ }` -- catalog save errors are silently swallowed. User gets no feedback if save fails. |

#### Ideal experience
- Analysis history sidebar with recall
- "Apply to exception" button when classification differs from the disputed entry
- Error feedback on catalog save failure
- Optional: side-by-side comparison mode for alternative classifications

---

### 5. Regulatory Intelligence (`RegulatoryIntel.tsx`)

**Screenshot**: `05-regulatory-intel.png`

#### What works well
- **Scenario modeling** (`RegulatoryIntel.tsx:129-160`): `calculateRealScenario` function computes impact on REAL portfolio data. Current duty vs. projected duty, delta, affected count. This is genuinely useful.
- **Affected entries drill-down** (`RegulatoryIntel.tsx:340-480`): After modeling, shows clickable list of impacted entries with color-coded status. Separated into "Impacted" vs "Already Cleared".
- **Signal detail panel**: Source, date, status, impact assessment, affected HTS codes.
- **Deep link support** (`RegulatoryIntel.tsx:178-186`): `?signal=` param for direct linking.

#### Issues found

| # | Issue | Severity | File:Line | Details |
|---|-------|----------|-----------|---------|
| P5-1 | ALL 6 regulatory signals are hardcoded | P0 | `RegulatoryIntel.tsx:16-89` | `const DEMO_SIGNALS: RegulatorySignal[]` with 6 static entries. No API call. No way to add, edit, or remove signals. This is the #1 credibility problem for the entire platform. The screen says "Live regulatory signals" but they are frozen demo data. |
| P5-2 | Rate impacts are hardcoded assumptions | P2 | `RegulatoryIntel.tsx:96-103` | `SIGNAL_RATE_IMPACTS` maps signal IDs to fixed percentages. Not computed from actual tariff data. |
| P5-3 | "Model Impact" delay is fake | P3 | `RegulatoryIntel.tsx:196-201` | `setTimeout(() => { ... }, 600)` -- artificial 600ms delay to simulate computation. The actual calculation is synchronous and instant. |
| P5-4 | No signal creation/management UI | P1 | `RegulatoryIntel.tsx:166-500` | Platform operators cannot add new regulatory signals, mark signals as superseded, or update status. Read-only on hardcoded data. |
| P5-5 | No notification/alert when new signals arrive | P2 | `RegulatoryIntel.tsx:166-500` | No subscription mechanism. No push notifications. No "new since last visit" indicator. |
| P5-6 | Affected entries only shown after clicking "Model Impact" | P3 | `RegulatoryIntel.tsx:340` | `scenario !== null` guards the affected entries list. You have to click the button to see which entries are affected, even though this information is available immediately. |

#### Ideal experience
- Signals fetched from backend API (even if seeded with defaults)
- Admin can add/edit/archive signals
- Affected entries shown immediately on signal selection (no click required)
- Notification badge when new signals are added
- Rate impacts computed from actual tariff schedule data

---

### 6. Entity Screening (`EntityScreening.tsx`)

**Screenshot**: `06-entity-screening.png`

#### What works well
- **Clean, focused form**: Entity name, entity type (Company/Individual/Vessel/Aircraft), submit.
- **Real API integration** (`EntityScreening.tsx:26`): `screenEntity()` calls real backend.
- **Quick-test entities**: "Huawei Technologies", "SMIC", "Xinjiang Cotton Corp", "Acme Trading Co" for testing.
- **ScreeningResult component**: Renders actual API results.

#### Issues found

| # | Issue | Severity | File:Line | Details |
|---|-------|----------|-----------|---------|
| P6-1 | No screening history | P2 | `EntityScreening.tsx:9-150` | Every screening is ephemeral. No log of past screenings. Critical for audit compliance - operators need to demonstrate they screened parties. |
| P6-2 | No batch screening | P2 | `EntityScreening.tsx:9-150` | Can only screen one entity at a time. For an order with 5 parties (shipper, consignee, manufacturer, notify, ultimate consignee), need to run 5 separate screens. |
| P6-3 | No link from exception queue | P2 | `EntityScreening.tsx:9-150` | DPS/SDN exceptions in the exception queue don't link to this screen pre-filled. You'd have to manually type the entity name. |
| P6-4 | Massive empty space below form | P3 | `06-entity-screening.png` | 60% of the screen is empty white space when no result is showing. Waste of real estate. |

#### Ideal experience
- Screening history log with timestamps and results
- Batch screening: paste or upload a list of entities
- Pre-fill from exception context when navigating from DPS exceptions
- Use empty space to show recent screenings or screening statistics

---

### 7. Trade Lane Comparison (`TradeLaneComparison.tsx`)

**Screenshot**: `07-trade-lanes.png`

#### What works well
- **Multi-destination comparison**: Up to 5 destinations with add/remove chips.
- **Pre-filled sensible defaults**: "Lithium-ion battery pack for electric vehicles", CN origin, US/DE/JP destinations, $12,500 value.
- **Real API call** (`TradeLaneComparison.tsx:70`): `getTradeLanes()` calls backend.
- **Lowest-cost highlighting** (`TradeLaneComparison.tsx:84-86`): Identifies and highlights the lowest landed cost option.
- **Origin selector with 14 countries** (`TradeLaneComparison.tsx:22-35`)

#### Issues found

| # | Issue | Severity | File:Line | Details |
|---|-------|----------|-----------|---------|
| P7-1 | Dropdown destination add uses CSS hover trick | P2 | `TradeLaneComparison.tsx:166-184` | `hidden group-hover:block` for dropdown. Not accessible, no keyboard navigation, disappears on mouse-out. |
| P7-2 | No connection to Product Analysis | P2 | `TradeLaneComparison.tsx:44-232` | Can't pre-fill from a product that was just analyzed. Must manually re-enter product description. |
| P7-3 | Results disappear on re-submission | P3 | `TradeLaneComparison.tsx:66` | `setResults([])` on new submission. Previous results are lost. No comparison history. |
| P7-4 | No export/share capability | P3 | `TradeLaneComparison.tsx:44-232` | Can't export comparison results as CSV, PDF, or shareable link. |

#### Ideal experience
- Proper accessible dropdown component for destination selection
- "Compare this product" button in Product Analysis that navigates here pre-filled
- Export results as CSV/PDF
- Save comparison scenarios

---

### 8. Orders & Entries (`PlatformOrders.tsx`)

**Screenshot**: `08-orders.png`

#### What works well
- **Clean table layout**: Order ID, Status, Items, Origin, Destination, Total Value, Total Duty, Created.
- **Real API data** (`PlatformOrders.tsx:71`): `getOrders()` fetches from backend.
- **Clickable rows** navigate to order detail (`PlatformOrders.tsx:187`).
- **Status badges**: Draft, Analyzed, Ready, Shipped with appropriate colors.

#### Issues found

| # | Issue | Severity | File:Line | Details |
|---|-------|----------|-----------|---------|
| P8-1 | No filters or search | P2 | `PlatformOrders.tsx:62-237` | 1,958 orders with no search, no status filter, no date filter, no origin/destination filter. Just a raw list. |
| P8-2 | No pagination | P1 | `PlatformOrders.tsx:173-226` | All 1,958 orders rendered at once. Performance problem. |
| P8-3 | No bulk operations | P2 | `PlatformOrders.tsx:62-237` | Can't select multiple orders for bulk analyze, export, or status change. |
| P8-4 | All orders show "Draft" status with "--" duty | P3 | `08-orders.png` | Every visible order is "Draft" with no duty calculated. No visual indication of which orders need attention. |
| P8-5 | No "Create Order" button | P2 | `PlatformOrders.tsx:82-97` | Platform operators can view orders but can't create them from this screen. Order creation seems limited to buyer/shipper surfaces. |

#### Ideal experience
- Search and filter controls
- Server-side pagination
- Bulk operations toolbar
- Status badges with color coding that draws attention to actionable states
- "Create order" button for platform operators

---

### 9. Education Center (`EducationCenter.tsx`)

**Screenshot**: `10-education-center.png`

#### What works well
- **Well-organized topic taxonomy**: Tariff Programs (5), Trade Agreements (2), Compliance Requirements (4), Country Guides (4). 15 total topics.
- **AI-powered "Learn More"** (`EducationCenter.tsx:150-163`): Opens AI assistant with context-aware prompt. Genuinely useful for on-demand learning.
- **Clean card layout**: Grid of topic cards with title, summary, action link.

#### Issues found

| # | Issue | Severity | File:Line | Details |
|---|-------|----------|-----------|---------|
| P9-1 | All content is AI-generated on demand, no authored content | P2 | `EducationCenter.tsx:146-163` | "Learn More" just opens the chat assistant. No pre-written guides, no diagrams, no decision trees. Quality depends entirely on the LLM response quality. |
| P9-2 | No search | P3 | `EducationCenter.tsx:146-235` | Can't search across topics. Must browse visually. |
| P9-3 | No progress tracking | P3 | `EducationCenter.tsx:146-235` | No indication of which topics have been explored, no bookmarks, no certification/quiz. |
| P9-4 | Topics are hardcoded | P3 | `EducationCenter.tsx:34-139` | Topic list is a static array. Can't be managed by admins. |

#### Ideal experience
- Blend of authored content (diagrams, decision trees, checklists) with AI Q&A
- Search across all topics
- Bookmarks and "recently viewed" section
- Quizzes or knowledge checks for training purposes

---

### 10. Compliance Dashboard (`ComplianceDashboard.tsx`)

**Screenshot**: `11-compliance-dashboard.png`

#### Issues found

| # | Issue | Severity | File:Line | Details |
|---|-------|----------|-----------|---------|
| P10-1 | ENTIRELY hardcoded demo data | P0 | `ComplianceDashboard.tsx:12-81` | `generateDemoEntries()` creates 47 fake entries with random values. `generateDemoStats()` computes stats from fake data. Nothing real. |
| P10-2 | Duplicates Control Tower functionality | P2 | `ComplianceDashboard.tsx:99-192` | Shows KPIs (total entries, value, duty, hold rate, compliance rate) + entry table with search/filter. The Control Tower already does this with REAL data. |
| P10-3 | "Pre-loaded" label is misleading | P3 | `ComplianceDashboard.tsx:141` | "47 entries pre-loaded. Real-time compliance monitoring" -- there's nothing real-time about hardcoded random data. |

#### Ideal experience
- Either merge with Control Tower (remove this screen) or make it show REAL compliance metrics
- If kept separate, differentiate by focusing on compliance-specific views (audit history, rule adherence scores, compliance officer workflow)

---

## Cross-Cutting Issues

### Information Architecture

| # | Issue | Severity | Details |
|---|-------|----------|---------|
| IA-1 | No cross-screen workflow linking | P1 | Exception Resolution doesn't link to Entity Screening (for DPS issues), Product Analysis (for classification disputes), or Regulatory Intel (for tariff-related holds). Each tool is an island. |
| IA-2 | No simulation control panel | P1 | The backend has a full simulation API. No UI to start/stop/configure simulations, seed scenarios, or control data generation. The "Ask Assistant" button is the only entry point. |
| IA-3 | Two screens show entirely fake data | P0 | Regulatory Intel (hardcoded `DEMO_SIGNALS`) and Compliance Dashboard (generated random entries). These undermine the credibility of the entire platform. |
| IA-4 | Operator Assistant is overloaded as the universal action handler | P1 | Nearly every action across the platform opens the chat assistant. Exception actions, education topics, regulatory analysis - all route to chat. This creates a bottleneck and hides the fact that direct-action APIs exist. |
| IA-5 | No global search | P2 | No way to search across entries, shipments, orders, and products from the nav. Must navigate to individual screens. |

### Navigation & Layout

| # | Issue | Severity | Details |
|---|-------|----------|---------|
| NL-1 | Nav sections are well-organized | N/A | Command Center > Intelligence > Tools hierarchy is logical and scannable. |
| NL-2 | Surface switcher is hidden behind hamburger menu | P3 | `PlatformLayout.tsx:27-37`: Menu icon in top-right, slide-out panel with surface switcher. Discovery problem for new users. |
| NL-3 | No breadcrumbs in detail screens | P3 | Shipment Detail, Entry Detail, Order Detail have no breadcrumb back to the parent list. Browser back is the only option. |

### Operator Assistant (`OperatorAssistant.tsx`)

| # | Issue | Severity | Details |
|---|-------|----------|---------|
| OA-1 | Assistant works well as a chat panel | N/A | Real SSE streaming, tool event indicators, markdown rendering, analysis cards, action buttons. Solid implementation. |
| OA-2 | Context is surface-aware | N/A | `setContext()` passes screen, entry/shipment IDs, exception type. Good contextual awareness. |
| OA-3 | No conversation persistence | P2 | `OperatorAssistant.tsx:214-575`: Messages are in Zustand store but not persisted. Refresh = lose everything. |
| OA-4 | Single conversation thread | P3 | Can't have multiple concurrent conversations or switch between topics. |

---

## Severity Distribution

| Severity | Count | Description |
|----------|-------|-------------|
| **P0** | 3 | Hardcoded demo data masquerading as live data (Regulatory Intel, Compliance Dashboard), no simulation UI |
| **P1** | 6 | Non-functional buttons (Contact Shipper), all exception actions route to chat, no status transitions, no cross-screen linking, missing pagination on 1958 orders |
| **P2** | 18 | Missing search/filter, no history/persistence, no bulk operations, no screening audit log, no assignment model |
| **P3** | 14 | Hardcoded strings, accessibility issues, empty space, missing breadcrumbs |

---

## Priority Recommendations

### Immediate (P0 - Fix before any demo)
1. **Replace hardcoded Regulatory Signals** with backend API + admin CRUD
2. **Remove or overhaul Compliance Dashboard** - either merge into Control Tower or connect to real data
3. **Add Simulation Control Panel** - start/stop, scenario selection, speed control, data reset

### High (P1 - Fix for production readiness)
4. **Make Exception Actions invoke backend APIs directly** instead of routing everything through chat
5. **Add status transitions to Exception Resolution** - Resolve/Escalate/Assign/Dismiss
6. **Fix "Contact Shipper" button** - generate email template or open communication workflow
7. **Add pagination to Orders** - 1,958 rows rendered at once is a performance problem
8. **Add cross-screen navigation** - DPS exception links to Entity Screening pre-filled, classification dispute links to Product Analysis pre-filled

### Medium (P2 - Next sprint)
9. Add search/filter controls to Active Shipments, Orders, Exception Queue
10. Add entity screening history/audit log
11. Make KPI cards clickable for drill-down
12. Compute trend values from real historical data
13. Add bulk operations (multi-select + batch actions)
14. Persist assistant conversations

### Lower (P3)
15. Accessible dropdown components (Trade Lane Comparison)
16. Breadcrumbs in detail screens
17. Education Center: authored content, search, progress tracking
18. Remove hardcoded portfolio name from Exception Queue

---

## What the Ideal Platform Experience Looks Like

**6 AM scenario**: A trade ops director opens the Control Tower.

1. **Instant situational awareness**: KPIs show overnight changes with real deltas. The ticker scrolls urgent alerts. One red flash: "GO deadline 1d: Polysilicon Cells at APM Newark."

2. **One-click action**: They click the GO alert. It opens Exception Resolution with the entry pre-selected. Status: DETAINED/UFLPA. They click "Request Traceability Docs" and the system **directly generates and emails a document request** to the shipper with entry details, required documents, and deadline. No chat needed.

3. **Cross-functional workflow**: A new regulatory signal appears (badge on Regulatory Intel nav). They see it: "Section 301 List 4A Rate Increase to 50%." They click "Model Impact" and instantly see 23 in-flight entries affected, $2.3M additional duty exposure. They click an affected entry and are taken to its detail view.

4. **Bulk resolution**: From the filtered view, they select 5 entries that need classification review, click "Assign to Broker Team B", and the broker surface shows them in their queue.

5. **Audit trail**: Every action is logged. Every screening result is persisted. Every status change has a timestamp and actor. Compliance officer can pull a full audit report.

That's the gap between where we are and where we need to be: **from a dashboard that shows data and routes to chat, to a control center that enables direct action with full traceability.**
