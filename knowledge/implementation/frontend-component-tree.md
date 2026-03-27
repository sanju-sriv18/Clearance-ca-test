# Frontend Component Tree

Implementation reference for the Clearance Intelligence Engine frontend component hierarchy. This document maps every component to its props, store dependencies, API calls, and rendering patterns.

## Full Component Hierarchy

```
ReactDOM.createRoot(#root)
  React.StrictMode
    App
      BrowserRouter
        AppRoutes
          AnimatePresence mode="wait"
            useRoutes(routes) --> one of:
              |
              +-- PlatformLayout
              |     Store: useOperatorStore (panelOpen)
              |     +-- Header (trailingContent: menu button)
              |     +-- SceneNav (sidebar)
              |     +-- SurfaceSwitcher (slide-out overlay)
              |     +-- ErrorBoundary
              |     |     +-- Outlet --> [13 Platform Screens]
              |     +-- OperatorAssistant (conditional, AnimatePresence)
              |
              +-- ShipperLayout
              |     +-- header (amber-branded)
              |     +-- NavLink sidebar (4 links)
              |     +-- SurfaceSwitcher (slide-out overlay)
              |     +-- ErrorBoundary
              |           +-- Outlet --> [9 Shipper Screens]
              |
              +-- BuyerLayout
                    Store: useCartStore (items.length)
                    +-- header (green-branded + cart badge)
                    +-- SurfaceSwitcher (slide-out overlay)
                    +-- ErrorBoundary
                          +-- Outlet --> [3 Buyer Screens]
```

## Shared Components

### Header

**File:** `components/Header.tsx`

| Prop | Type | Purpose |
|------|------|---------|
| `trailingContent` | `ReactNode` | Slot for surface-specific controls (menu button) |

**Store deps:** None directly (each layout passes its own trailing content).
**API calls:** None.

### SurfaceSwitcher

**File:** `components/SurfaceSwitcher.tsx`

**Store deps:** `useSessionStore` (currentSurface, setSurface).
**Renders:** Navigation links to `/platform`, `/shipper`, `/buyer` with active state highlighting.

### SceneNav

**File:** `components/SceneNav.tsx`

**Store deps:** `useOperatorStore` (togglePanel).
**Renders:** `NavLink` elements for all platform routes. Includes the operator assistant toggle button at the bottom.

### SystemsPanel

**File:** `components/SystemsPanel.tsx`

**Store deps:** None.
**API calls:** Uses `useSimulation()` hook internally.
**Renders:** Backend system health indicators and simulation start/stop/speed controls.

### ConfidenceBadge

**File:** `components/ConfidenceBadge.tsx`

| Prop | Type | Purpose |
|------|------|---------|
| `level` | `ConfidenceLevel` | "HIGH", "MEDIUM", or "LOW" |

**Renders:** Colored badge. GREEN for HIGH, AMBER for MEDIUM, RED for LOW.

### ErrorBoundary

**File:** `components/ErrorBoundary.tsx`

| Prop | Type | Purpose |
|------|------|---------|
| `children` | `ReactNode` | Content to wrap |
| `fallback` | `ReactNode` (optional) | Custom error UI |

**Class component.** Uses `getDerivedStateFromError` to catch render errors. Default fallback shows an AlertTriangle icon, error message, and "Try Again" button that resets the error state.

### LoadingSpinner

**File:** `components/LoadingSpinner.tsx`

**Renders:** Centered spinner with animation. No props.

## Platform Surface Components

### Screen Components

#### ControlTower (Dashboard)

**File:** `surfaces/platform/screens/ControlTower.tsx`
**Route:** `/platform/dashboard`

| Store Dep | Usage |
|-----------|-------|
| `usePipelineStore` | entries, aggregates, getByStatus, getExceptionQueue |

| Child Component | Purpose |
|----------------|---------|
| `DashboardStats` | KPI summary cards |
| `CorridorMatrix` | Trade corridor grid |
| `DutyStackBar` | Duty by program chart |
| `PlatformPulse` | Real-time pipeline health |
| `EntryTable` | Recent entries table |
| `ExceptionQueue` | Priority exception list |

Optionally uses `useDashboardStream(enabled)` for live simulation data.

#### ProductAnalysis

**File:** `surfaces/platform/screens/ProductAnalysis.tsx`
**Route:** `/platform/analysis`

| Store Dep | Usage |
|-----------|-------|
| `useAnalysisStore` | Full analysis state |
| `usePipelineStore` | Creates pipeline entries |

| Hook | Purpose |
|------|---------|
| `useAnalysis()` | Drives SSE stream |
| `usePipelineAnalysis()` | Dual-store dispatch |

| Child Component | Purpose |
|----------------|---------|
| `ProductInputForm` | Product description + origin/destination form |
| `ReasoningChain` | Step-by-step classification display |
| `TariffStack` | Tariff program breakdown |
| `CompliancePanel` | PGA + DPS + UFLPA display |
| `FTASummary` | FTA savings card |

#### EntryDetail

**File:** `surfaces/platform/screens/EntryDetail.tsx`
**Route:** `/platform/entries/:id`

| Store Dep | Usage |
|-----------|-------|
| `usePipelineStore` | getEntry(id) |

Renders the full analysis result for a specific clearance entry. Uses `useParams()` to extract the entry ID.

#### TradeLaneComparison

**File:** `surfaces/platform/screens/TradeLaneComparison.tsx`
**Route:** `/platform/trade-lanes`

| API Call | Purpose |
|----------|---------|
| `getTradeLanes()` | Multi-destination comparison |

| Child Component | Purpose |
|----------------|---------|
| `TradeLaneCard` | Per-destination card with tariff + compliance |
| `ScenarioComparison` | Side-by-side duty comparison |

#### RegulatoryIntel

**File:** `surfaces/platform/screens/RegulatoryIntel.tsx`
**Route:** `/platform/regulatory`

| API Call | Purpose |
|----------|---------|
| `getRegulatorySignals()` | Signal feed |
| `modelScenario()` | What-if scenario modeling |

| Child Component | Purpose |
|----------------|---------|
| `SignalBoard` | Signal list with status badges |
| `RegulatoryFeed` | Scrollable signal details |
| `ScenarioComparison` | Before/after modeling display |

#### ExceptionResolution

**File:** `surfaces/platform/screens/ExceptionResolution.tsx`
**Route:** `/platform/exceptions`

| Store Dep | Usage |
|-----------|-------|
| `usePipelineStore` | getExceptionQueue() |
| `useOperatorStore` | setContext for assistant |

| Child Component | Purpose |
|----------------|---------|
| `ExceptionQueue` | Sorted exception list |
| `ExceptionPanel` | Selected exception detail |
| `ExceptionActions` | Resolution action buttons |

#### EntityScreening

**File:** `surfaces/platform/screens/EntityScreening.tsx`
**Route:** `/platform/screening`

| API Call | Purpose |
|----------|---------|
| `screenEntity()` | DPS entity screening |

| Child Component | Purpose |
|----------------|---------|
| `ScreeningResult` | Match/clear result card |

#### ActiveShipments

**File:** `surfaces/platform/screens/ActiveShipments.tsx`
**Route:** `/platform/shipments`

| API Call | Purpose |
|----------|---------|
| `getShipments()` | Filtered shipment list |

#### PlatformShipmentDetail

**File:** `surfaces/platform/screens/PlatformShipmentDetail.tsx`
**Route:** `/platform/shipments/:id`

| API Call | Purpose |
|----------|---------|
| `getShipmentDetail()` | Full shipment detail |
| `analyzeShipment()` | Trigger analysis |

#### PlatformOrders / PlatformOrderDetail

**Routes:** `/platform/orders`, `/platform/orders/:id`

| API Call | Purpose |
|----------|---------|
| `getOrders()` | Order list |
| `getOrder()` | Single order |
| `analyzeOrder()` | Trigger order analysis |

#### EducationCenter

**File:** `surfaces/platform/screens/EducationCenter.tsx`
**Route:** `/platform/education`

Static content. No store or API dependencies. Renders compliance training materials and reference links.

#### ComplianceDashboard

**File:** `surfaces/platform/screens/ComplianceDashboard.tsx`

Uses static demo data. Renders `DashboardStatsView` and `EntryTable` with pre-generated compliance entries.

### Platform-Specific Components

#### OperatorAssistant (Detailed Breakdown)

**File:** `surfaces/platform/components/OperatorAssistant.tsx`
**Store deps:** `useOperatorStore` (panelOpen, messages, streaming, currentContext, closePanel, addMessage, setStreaming)
**API calls:** `streamChat`

This is the most complex component in the application. It implements an AI chat panel with rich content rendering.

**Internal types:**

```typescript
type Segment =
  | { type: "text"; content: string }
  | { type: "card"; data: AnalysisCardData }
  | { type: "action"; label: string; route: string };

interface AnalysisCardData {
  title?: string;
  items: Array<{
    product: string;
    hsCode: string;
    value: number;
    duty: number;
    effectiveRate: number;
    compliance: string;
  }>;
  totals?: { totalDuty: number; effectiveRate: number; landedCost: number };
}
```

**Content parsing:** The `parseContent()` function splits assistant message content into segments:

1. **Text segments** -- Plain text rendered in gray chat bubbles.
2. **Analysis cards** -- JSON between `[ANALYSIS_CARD]...[/ANALYSIS_CARD]` markers, rendered as structured data cards with product/HS code/value/duty/compliance grids.
3. **Action buttons** -- `[ACTION:label:route]` markers rendered as navigation buttons that call `navigate(route)`.

**Tool progress indicators:** During streaming, the component tracks `activeTools` state. When the backend sends tool events (`{ tool, status: "running", label }`), they appear as animated progress pills. When `status: "complete"` arrives, the tool is removed from the active list.

**Conversation history:** For multi-turn conversations, the component builds a full conversation history string prefixed with "Previous conversation:" and sends it as part of the message to maintain context.

**Suggested prompts:** When the message list is empty, four suggested prompts are displayed as clickable buttons:

1. "I need to distribute products from China and Vietnam to the US"
2. "Help me resolve the UFLPA detention on a shipment"
3. "What's the impact of the new Section 301 tariff changes?"
4. "Walk me through customs clearance for electronics from Taiwan"

**Keyboard shortcuts:** Escape key closes the panel. Input auto-focuses when the panel opens.

**Auto-trigger:** When the panel opens and the last message is from the user (e.g., a screen set context and added a user message before opening), the assistant automatically triggers a response.

**Sub-component: RenderedAnalysisCard**

Renders a structured card with:
- Gradient header with title.
- Per-item rows: product name, HS code, compliance badge (color-coded), value/duty/rate grid.
- Totals footer: total duty, effective rate, landed cost.

#### ExceptionQueue

**File:** `surfaces/platform/components/ExceptionQueue.tsx`
**Store deps:** `usePipelineStore` (getExceptionQueue)

Renders the prioritized exception list. Entries are color-coded by status:
- DETAINED: Red background
- FLAGGED: Red-orange background
- HOLD: Amber background
- EXCEPTION: Gray background

#### EntryTable

**File:** `surfaces/platform/components/EntryTable.tsx`

| Prop | Type | Purpose |
|------|------|---------|
| `entries` | `DashboardEntry[]` or `ClearanceEntry[]` | Rows to display |

Sortable, filterable table with status badges and click-through to entry detail.

#### DashboardStats

**File:** `surfaces/platform/components/DashboardStats.tsx`

Renders KPI cards from `PlatformAggregates`: total entries, total duty, clearance rate, average clearance hours, STP rate, hold rate.

#### CorridorMatrix

**File:** `surfaces/platform/components/CorridorMatrix.tsx`

Renders a grid of trade corridors with volume, average duty rate, hold rate, FTA utilization, and trend indicators.

#### ProductInputForm

**File:** `surfaces/platform/components/ProductInputForm.tsx`

Form component with fields for product description, origin country, destination country, declared value, and currency. Calls `onSubmit` with a `ProductInput` object.

#### ReasoningChain

**File:** `surfaces/platform/components/ReasoningChain.tsx`

Renders `ClassificationStep[]` as a vertical chain of numbered steps with titles and content, revealing progressively during streaming.

## Shipper Surface Components

### Screen Components

#### OrderList (Default)

**Route:** `/shipper/orders`
**API calls:** `getOrders()`
**Renders:** List of `OrderCard` components with status filtering.

#### OrderDetail

**Route:** `/shipper/orders/:id`
**API calls:** `getOrder(id)`, `analyzeOrder(id)`, `shipOrder(id)`, document endpoints
**Child components:** `AnalysisOverlay`, `AnalysisResultsPanel`, `DutyBreakdownCard`, `PlainLanguageSummary`, `DocumentRequirements`, `DocumentViewer`, `AttachedDocuments`

#### CreateOrder

**Route:** `/shipper/orders/new`
**API calls:** `getProducts()`, `createOrder(data)`
**Renders:** Multi-step form: select products, set quantities, choose destination, review.

#### ShipperCatalog

**Route:** `/shipper/products`
**API calls:** `getProducts()`
**Renders:** Grid of `ShipProductCard` components with `StatusLight` readiness indicators.

#### ShipperProductDetail

**Route:** `/shipper/products/:id`
**API calls:** `getProduct(id)`
**Child components:** `PlainLanguageSummary`, `ReadinessCard`

#### AddProduct

**Route:** `/shipper/products/new`
**API calls:** `createProduct(product)`
**Renders:** Product registration form with source location management.

#### ShipmentHistory

**Route:** `/shipper/shipments`
**API calls:** `getShipments()`
**Renders:** List of `ShipmentCard` components with status filtering.

#### ShipmentDetailScreen

**Route:** `/shipper/shipments/:id`
**API calls:** `getShipmentDetail(id)`, `getShipmentDocuments(id)`
**Child components:** `ShipmentTimeline`, `ShipmentWaypoints`, `ShipmentFinancials`, `ShipmentChat`, `AttachedDocuments`

#### ResolutionCenter

**Route:** `/shipper/resolve`
**API calls:** `resolutionLookup()`, `resolutionUpload()`, `resolutionSuggest()`
**Renders:** Self-service resolution tools: product lookup, document upload, action suggestions.

### Shipper-Specific Components

#### StatusLight

| Prop | Type | Purpose |
|------|------|---------|
| `status` | `"green" \| "yellow" \| "red"` | Traffic light color |

Renders a colored dot indicator for ship-readiness.

#### AnalysisOverlay

Full-screen overlay displayed during order analysis. Shows streaming progress with step indicators and a cancel button.

**Store deps:** None (receives streaming state via props).

#### AnalysisResultsPanel

Expandable panel showing analysis results after completion. Contains duty breakdown, compliance summary, and document requirements.

#### PlainLanguageSummary

Uses `lib/plainLanguage.ts` to generate human-readable summaries of compliance and tariff data. Avoids jargon and uses simple sentences.

#### ShipmentTimeline

Renders `ShipmentEvent[]` as a vertical timeline with actor icons, timestamps, and event descriptions. Tool-use events show the tool name and result.

#### ShipmentWaypoints

Renders `ClearanceWaypoint[]` as a geographic progression bar. Waypoints are color-coded by status: green (completed), blue (current), gray (pending).

#### ShipmentFinancials

Renders predicted vs. actual duty and landed cost comparison with variance percentage and color-coded delta indicators.

#### ShipmentChat

In-shipment messaging interface. Uses the `streamChat` API with shipment context.

#### DocumentRequirements / DocumentViewer / AttachedDocuments

Document management components for viewing required documents, previewing generated PDFs, and listing uploaded attachments.

## Buyer Surface Components

### Screen Components

#### StoreFront

**Route:** `/buyer/shop`
**Data source:** Static product catalog from `data/products.ts`
**Store deps:** `useCartStore` (addItem)
**Renders:** Product grid with `PriceDisplay` and `DutiesBadge` per card.

#### ProductPage

**Route:** `/buyer/product/:id`
**Data source:** Static product catalog
**Store deps:** `useCartStore` (addItem)
**Child components:** `PriceDisplay`, `DutiesBadge`, `BreakdownToggle`

#### CheckoutPrice

**Route:** `/buyer/checkout`
**Store deps:** `useCartStore` (items, getTotal, getItemCount, removeItem, updateQuantity, clearCart)
**Renders:** Cart item list with quantity controls, duty-inclusive totals, and `BreakdownToggle` for line-item detail.

### Buyer-Specific Components

#### PriceDisplay

Renders the duty-inclusive landed cost price. Formats in the appropriate currency.

#### DutiesBadge

Small green badge indicating "Duties Included" on product cards.

#### BreakdownToggle

Expandable section that reveals the duty/tax/shipping breakdown for a product or cart total. Collapsed by default to keep the checkout experience clean.

## Key Rendering Patterns

### ErrorBoundary Wrapping

Every surface layout wraps its `<Outlet />` in an `ErrorBoundary`:

```tsx
<main className="flex-1 overflow-hidden">
  <ErrorBoundary>
    <Outlet />
  </ErrorBoundary>
</main>
```

This ensures navigation and layout chrome remain functional even if a screen crashes.

### AnimatePresence for Panels

Conditional panels use `AnimatePresence` to enable exit animations:

```tsx
<AnimatePresence>
  {panelOpen && (
    <motion.div
      initial={{ width: 0, opacity: 0 }}
      animate={{ width: 384, opacity: 1 }}
      exit={{ width: 0, opacity: 0 }}
      transition={{ type: "spring", bounce: 0.1, duration: 0.35 }}
    >
      <OperatorAssistant />
    </motion.div>
  )}
</AnimatePresence>
```

Without `AnimatePresence`, the component would unmount immediately when `panelOpen` becomes false, skipping the exit animation.

### Conditional Store Updates

The `usePipelineAnalysis` hook uses an `updateAnalysisStore` flag to conditionally dispatch to the analysis store:

```typescript
if (options.updateAnalysisStore) {
  analysisStore.updateClassification(result);
}
```

This allows the same hook to be used from both the `ProductAnalysis` screen (needs both stores) and other contexts (only needs pipeline store).

### Direct State Reads During SSE

Components that receive rapid SSE updates read state directly from the store to avoid stale closures:

```typescript
// Inside SSE callback (captured in closure):
const s = useOperatorStore.getState();
const msgs = [...s.messages];
msgs[msgs.length - 1] = { ...msgs[msgs.length - 1], content: last.content + chunk };
useOperatorStore.setState({ messages: msgs });
```

This pattern is used in `OperatorAssistant` for chat streaming and avoids the "stale state in callback" problem that occurs with React hooks.
