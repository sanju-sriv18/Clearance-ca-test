# Frontend State Patterns

Implementation reference for state management patterns in the Clearance Intelligence Engine. Covers full store definitions, data flow with examples, SSE streaming patterns, response transformer implementations, and hook composition.

## Store Definitions with Full TypeScript Interfaces

### Pipeline Store

```typescript
// store/pipelineStore.ts

import { create } from "zustand";
import type {
  ClassificationResult,
  ClassificationStep,
  TariffResult,
  ComplianceResult,
} from "@/types";

// --- Status Types ---

type EntryStatus =
  | "RECEIVED"
  | "CLASSIFYING"
  | "CALCULATING"
  | "SCREENING"
  | "CLEARED"
  | "PENDING"
  | "HOLD"
  | "EXCEPTION"
  | "DETAINED"
  | "FLAGGED";

type EntrySource = "platform" | "shipper" | "buyer" | "demo";

// --- Entry Interface ---

interface ClearanceEntry {
  id: string;
  product: string;
  productName: string;
  origin: string;
  destination: string;
  declaredValue: number;
  currency: string;
  hsCode?: string;
  shipper?: string;
  status: EntryStatus;
  source: EntrySource;
  createdAt: string;
  classification?: ClassificationResult;
  tariff?: TariffResult;
  compliance?: ComplianceResult;
  streamingSteps?: ClassificationStep[];
  totalDuty?: number;
  effectiveRate?: number;
  landedCost?: number;
  complianceStatus?: "CLEAR" | "REVIEW" | "HOLD";
  complianceFlags?: string[];
  corridor?: string;
  carrier?: string;
  trackingNumber?: string;
  customsPort?: string;
  documentsSubmitted?: string[];
  documentsMissing?: string[];
  estimatedDeliveryDate?: string;
  holdImpactDays?: number;
  recommendedActions?: string[];
}

// --- Aggregate Interfaces ---

interface CorridorMetric {
  corridor: string;
  origin: string;
  destination: string;
  volume: number;
  avgDutyRate: number;
  holdRate: number;
  ftaUtilization: number | null;
  trend: number;
}

interface ProgramMetric {
  program: string;
  amount: number;
  percentage: number;
}

interface StatusMetric {
  status: string;
  count: number;
  percentage: number;
}

interface PlatformAggregates {
  totalEntriesMonth: number;
  totalEntriesWeek: number;
  totalValueMonth: number;
  totalDutyMonth: number;
  clearanceRate: number;
  avgClearanceHours: number;
  stpRate: number;
  holdRate: number;
  exceptionRate: number;
  ftaSavingsMonth: number;
  corridors: CorridorMetric[];
  dutyByProgram: ProgramMetric[];
  statusDistribution: StatusMetric[];
}

// --- Store Interface ---

interface PipelineState {
  entries: ClearanceEntry[];
  aggregates: PlatformAggregates;
  nextId: number;
  createEntry: (entry: Omit<ClearanceEntry, "id" | "createdAt">) => string;
  updateEntryStatus: (id: string, status: EntryStatus) => void;
  updateEntryAnalysis: (id: string, updates: Partial<ClearanceEntry>) => void;
  setAggregates: (aggregates: PlatformAggregates) => void;
  getEntry: (id: string) => ClearanceEntry | undefined;
  getByStatus: (...statuses: EntryStatus[]) => ClearanceEntry[];
  getBySource: (source: EntrySource) => ClearanceEntry[];
  getByCorridor: (corridor: string) => ClearanceEntry[];
  getExceptionQueue: () => ClearanceEntry[];
}
```

**ID generation pattern:**

```typescript
createEntry: (entry) => {
  const state = get();
  const id = `CLR-2025-${String(state.nextId).padStart(5, "0")}`;
  // ...
  set((s) => ({
    entries: [newEntry, ...s.entries],    // Prepend (newest first)
    nextId: s.nextId + 1,
    aggregates: {
      ...s.aggregates,
      totalEntriesMonth: s.aggregates.totalEntriesMonth + 1,
      totalEntriesWeek: s.aggregates.totalEntriesWeek + 1,
      totalValueMonth: s.aggregates.totalValueMonth + (entry.declaredValue || 0),
      totalDutyMonth: s.aggregates.totalDutyMonth + (entry.totalDuty || 0),
    },
  }));
  return id;
},
```

**Exception queue sort logic:**

```typescript
getExceptionQueue: () => {
  return get().entries
    .filter((e) => ["DETAINED", "FLAGGED", "HOLD", "EXCEPTION"].includes(e.status))
    .sort((a, b) => {
      const priority: Record<string, number> = {
        DETAINED: 0,
        FLAGGED: 1,
        HOLD: 2,
        EXCEPTION: 3,
      };
      return (priority[a.status] ?? 4) - (priority[b.status] ?? 4);
    });
},
```

### Analysis Store

```typescript
// store/analysisStore.ts

interface AnalysisState {
  selectedProduct: string;
  selectedOrigin: string;
  selectedDestination: string;
  declaredValue: number;
  currency: string;
  isStreaming: boolean;
  streamingStep: number;
  streamingSteps: ClassificationStep[];
  classification: ClassificationResult | null;
  tariff: TariffResult | null;
  compliance: ComplianceResult | null;
  error: string | null;
  setProduct: (input: Partial<ProductInput>) => void;
  startAnalysis: () => void;
  addStreamingStep: (step: ClassificationStep) => void;
  updateClassification: (result: ClassificationResult) => void;
  updateTariff: (result: TariffResult) => void;
  updateCompliance: (result: ComplianceResult) => void;
  setError: (error: string) => void;
  reset: () => void;
}

// Selectors (exported separately)
const selectIsAnalysisComplete = (state: AnalysisState) =>
  state.classification !== null &&
  state.tariff !== null &&
  state.compliance !== null;

const selectOverallConfidence = (state: AnalysisState): ConfidenceLevel | null =>
  state.classification?.confidence ?? null;
```

**Key implementation detail:** `updateCompliance` sets `isStreaming: false` because compliance is the final engine in the pipeline:

```typescript
updateCompliance: (result) =>
  set({ compliance: result, isStreaming: false }),
```

### Operator Store

```typescript
// store/operatorStore.ts

interface OperatorMessage {
  role: "user" | "assistant";
  content: string;
  timestamp: string;
}

interface OperatorState {
  panelOpen: boolean;
  messages: OperatorMessage[];
  streaming: boolean;
  currentContext: {
    screen?: string;
    shipmentId?: string;
    entryId?: string;
    exceptionType?: string;
    signalId?: string;
    signalTitle?: string;
  };
  togglePanel: () => void;
  openPanel: () => void;
  closePanel: () => void;
  setContext: (ctx: Partial<OperatorState["currentContext"]>) => void;
  addMessage: (msg: OperatorMessage) => void;
  updateLastMessage: (content: string) => void;
  setStreaming: (s: boolean) => void;
  clearMessages: () => void;
}
```

**updateLastMessage implementation:** Only modifies the last message if it is an assistant message:

```typescript
updateLastMessage: (content) =>
  set((s) => {
    const msgs = [...s.messages];
    if (msgs.length > 0 && msgs[msgs.length - 1].role === "assistant") {
      msgs[msgs.length - 1] = { ...msgs[msgs.length - 1], content };
    }
    return { messages: msgs };
  }),
```

### Session Store

```typescript
// store/sessionStore.ts

interface SessionState {
  currentSurface: Surface;               // "platform" | "shipper" | "buyer"
  sessionId: string;                     // Auto-generated
  userId: string | null;
  sidebarOpen: boolean;                  // Default: true
  theme: "light" | "dark";              // Default: "light"
  setSurface: (surface: Surface) => void;
  setSessionId: (id: string) => void;
  setUserId: (id: string | null) => void;
  toggleSidebar: () => void;
  setTheme: (theme: "light" | "dark") => void;
}

function generateSessionId(): string {
  return `session_${Date.now()}_${Math.random().toString(36).slice(2, 9)}`;
}
```

### Cart Store

```typescript
// store/cartStore.ts

interface CartItem {
  product: CatalogProduct;    // From data/products.ts
  quantity: number;
}

interface CartState {
  items: CartItem[];
  addItem: (product: CatalogProduct) => void;
  removeItem: (productId: string) => void;
  updateQuantity: (productId: string, quantity: number) => void;
  clearCart: () => void;
  getTotal: () => number;
  getItemCount: () => number;
}
```

**addItem deduplication:**

```typescript
addItem: (product) => {
  set((s) => {
    const existing = s.items.find((i) => i.product.id === product.id);
    if (existing) {
      return {
        items: s.items.map((i) =>
          i.product.id === product.id
            ? { ...i, quantity: i.quantity + 1 }
            : i,
        ),
      };
    }
    return { items: [...s.items, { product, quantity: 1 }] };
  });
},
```

**updateQuantity auto-remove:**

```typescript
updateQuantity: (productId, quantity) => {
  if (quantity <= 0) {
    get().removeItem(productId);
    return;
  }
  // ... update quantity
},
```

## API to Store to Component Data Flow Examples

### Example 1: Product Analysis (Single Product)

```
User clicks "Analyze" on ProductAnalysis screen
  |
  v
useAnalysis().startStream(input)
  |
  +-- store.setProduct(input)         --> analysisStore: selectedProduct = "EV battery..."
  +-- store.startAnalysis()           --> analysisStore: isStreaming = true, results = null
  |
  +-- streamAnalysis(input, callbacks) --> POST /analyze/stream
      |
      v
  SSE: event: classification_step
  data: {"step":"reasoning","index":0,"content":"Examining battery composition..."}
      |
      v
  callback: onClassificationStep({ step: 1, title: "Reasoning Step 1", content: "..." })
      |
      v
  store.addStreamingStep(step)
      |
      v
  analysisStore: streamingSteps = [{ step: 1, ... }], streamingStep = 1
      |
      v
  ReasoningChain component re-renders, shows step 1
      |
      v
  [... more steps arrive ...]
      |
      v
  SSE: event: classification_complete
  data: {"hs_code":"8507.60","confidence":"HIGH",...}
      |
      v
  transformClassification(payload) --> { htsCode: "8507.60", confidence: "HIGH", ... }
      |
      v
  store.updateClassification(result)
      |
      v
  analysisStore: classification = { htsCode: "8507.60", ... }
      |
      v
  Classification panel re-renders with result
      |
      v
  SSE: event: tariff_complete
  data: {"total_duty":4800,"effective_rate":0.384,...}
      |
      v
  transformTariff(payload) --> { totalDuty: 4800, effectiveRate: 0.384, ... }
      |
      v
  store.updateTariff(result)
      |
      v
  TariffStack component re-renders
      |
      v
  SSE: event: analysis_complete
  data: {"compliance":{"pga_flags":[],"overall_risk":"LOW",...}}
      |
      v
  transformCompliance(payload.compliance) --> { overallStatus: "CLEAR", ... }
      |
      v
  store.updateCompliance(result)   --> also sets isStreaming = false
      |
      v
  CompliancePanel re-renders, streaming indicator disappears
```

### Example 2: Pipeline Entry (Dual-Store)

```
usePipelineAnalysis().analyze(input, { source: "platform", updateAnalysisStore: true })
  |
  +-- pipelineStore.createEntry({...})
  |     --> entries: [{ id: "CLR-2025-00050", status: "RECEIVED", ... }, ...existing]
  |     --> aggregates: { totalEntriesMonth: 47833, ... }
  |
  +-- analysisStore.startAnalysis()
  |     --> isStreaming: true
  |
  +-- setTimeout: pipelineStore.updateEntryStatus("CLR-2025-00050", "CLASSIFYING")  [300ms]
  |
  +-- streamAnalysis(input, callbacks)
      |
      v
  onClassificationComplete:
    pipelineStore.updateEntryAnalysis("CLR-2025-00050", {
      classification: result,
      hsCode: "8507.60",
      status: "CALCULATING"           // Pipeline advances
    })
    analysisStore.updateClassification(result)  // Mirror to analysis store
      |
      v
  ControlTower dashboard: entry row updates to "CALCULATING"
  ProductAnalysis: classification panel populates
      |
      v
  onTariffComplete:
    pipelineStore.updateEntryAnalysis("CLR-2025-00050", {
      tariff: result,
      totalDuty: 4800,
      effectiveRate: 0.384,
      landedCost: 17300,
      status: "SCREENING"
    })
    analysisStore.updateTariff(result)
      |
      v
  onComplianceComplete:
    // Determine final status from compliance result
    finalStatus = result.overallStatus === "HOLD" ? "HOLD"
                : result.overallStatus === "REVIEW" ? "PENDING"
                : "CLEARED"

    // Extract compliance flags
    flags = []
    if (result.uflpaRisk.level === "HIGH") flags.push("UFLPA")
    if (result.dpsScreening.matched) flags.push("DPS")
    result.pgaFlags.forEach(f => {
      if (f.status === "REQUIRED") flags.push(`PGA:${f.agency}`)
    })

    pipelineStore.updateEntryAnalysis("CLR-2025-00050", {
      compliance: result,
      complianceStatus: "CLEAR",
      complianceFlags: [],
      status: "CLEARED"
    })
    analysisStore.updateCompliance(result)   // Sets isStreaming: false
```

### Example 3: Buyer Cart Flow

```
StoreFront: user clicks "Add to Cart"
  |
  v
useCartStore.getState().addItem(product)
  |
  v
cartStore: items = [{ product: { id: "p1", buyerPrice: 149.99 }, quantity: 1 }]
  |
  v
BuyerLayout: cartCount = useCartStore(s => s.items.length) --> re-renders badge "1"
  |
  v
User navigates to /buyer/checkout
  |
  v
CheckoutPrice: reads items, getTotal(), getItemCount()
  |
  v
Renders: cart list, BreakdownToggle per item, total with duties included
```

## SSE Streaming Patterns

### Buffer-Based Line Parsing

Both `streamAnalysis` and `streamChat` use the same buffer pattern for SSE parsing:

```typescript
const decoder = new TextDecoder();
let buffer = "";

while (true) {
  const { done, value } = await reader.read();
  if (done) {
    callbacks.onDone?.();
    break;
  }

  // Append new bytes to buffer
  buffer += decoder.decode(value, { stream: true });

  // Split on newlines
  const lines = buffer.split("\n");

  // The last element may be an incomplete line -- keep it in the buffer
  buffer = lines.pop() || "";

  // Process complete lines
  for (const line of lines) {
    if (line.startsWith("event: ")) {
      currentEventType = line.slice(7).trim();
    } else if (line.startsWith("data: ")) {
      const data = line.slice(6).trim();
      if (data === "[DONE]") {
        callbacks.onDone?.();
        return;
      }
      try {
        const payload = JSON.parse(data);
        // Dispatch based on currentEventType
      } catch {
        // Skip malformed JSON
      }
    }
  }
}
```

**Why manual parsing instead of EventSource?** The `streamAnalysis` endpoint requires POST with a JSON body. The browser's native `EventSource` API only supports GET requests. The `streamChat` endpoint similarly requires POST. Only `useDashboardStream` uses native `EventSource` because its endpoint is GET.

### Deduplication Guards

The SSE stream may send both individual events (e.g., `tariff_complete`) and a combined `analysis_complete` event that contains the same data. Guards prevent duplicate dispatches:

```typescript
let seenClassificationComplete = false;
let seenTariffComplete = false;

case "classification_complete":
  if (!seenClassificationComplete) {
    seenClassificationComplete = true;
    callbacks.onClassificationComplete?.(transformClassification(payload));
  }
  break;

case "analysis_complete":
  // Only extract tariff if we haven't seen tariff_complete
  if (!seenTariffComplete && payload.tariff) {
    callbacks.onTariffComplete?.(transformTariff(payload.tariff));
  }
  // Compliance always comes from analysis_complete
  if (payload.compliance) {
    callbacks.onComplianceComplete?.(transformCompliance(payload.compliance));
  }
  break;
```

### Chat Streaming with Tool Events

The `streamChat` function dispatches three types of SSE data payloads:

```typescript
for (const line of lines) {
  if (line.startsWith("data: ")) {
    const data = line.slice(6).trim();
    if (data === "[DONE]") { onDone(); return; }
    try {
      const payload = JSON.parse(data);
      if (payload.error) {
        onError(payload.error);
      } else if (payload.tool && payload.status) {
        // Tool event: { tool: "classify", status: "running", label: "..." }
        onToolEvent?.({
          tool: payload.tool,
          status: payload.status,
          label: payload.label,
        });
      } else if (payload.text) {
        // Text chunk
        onChunk(payload.text);
      }
    } catch { /* skip */ }
  }
}
```

### Dashboard SSE with Polling Fallback

The `useDashboardStream` hook uses native `EventSource` with automatic fallback:

```typescript
// Try SSE first
const es = new EventSource(`${BASE_URL}/simulation/dashboard/stream`);

es.addEventListener("dashboard", (event: MessageEvent) => {
  setData(JSON.parse(event.data) as PlatformAggregates);
  setConnected(true);
});

es.onerror = () => {
  es.close();
  setConnected(false);
  // Fall back to polling
  if (!pollRef.current) {
    pollLatest();
    pollRef.current = setInterval(pollLatest, 5000);
  }
};

es.onopen = () => {
  setConnected(true);
  // Stop polling if SSE reconnects
  if (pollRef.current) {
    clearInterval(pollRef.current);
    pollRef.current = null;
  }
};
```

## Response Transformer Implementations

### Classification Transformer

```typescript
function transformClassification(raw: Record<string, unknown>): ClassificationResult {
  if (!raw) return {
    htsCode: "", description: "", confidence: "LOW",
    reasoning: [], chapter: "", section: "", gri: []
  };

  const steps = Array.isArray(raw.reasoning_steps)
    ? (raw.reasoning_steps as string[])
    : [];

  return {
    htsCode: (raw.hs_code as string) ?? "",
    description: (raw.hs_description as string) ?? "",
    confidence: (raw.confidence as "HIGH" | "MEDIUM" | "LOW") ?? "LOW",
    reasoning: steps.map((s, i) => ({
      step: i + 1,
      title: `Step ${i + 1}`,
      content: typeof s === "string" ? s : JSON.stringify(s),
    })),
    chapter: (raw.chapter as string) ?? "",
    section: (raw.heading as string) ?? "",
    gri: Array.isArray(raw.alternative_codes)
      ? (raw.alternative_codes as unknown[]).map((c: unknown) =>
          typeof c === "string" ? c : ((c as Record<string, unknown>)?.code as string) ?? ""
        )
      : [],
  };
}
```

**Key behaviors:**
- Returns safe defaults (empty strings, empty arrays) if input is null.
- Maps `reasoning_steps` (string array) to `ClassificationStep[]` with auto-incrementing step numbers.
- Handles `alternative_codes` that may be strings or objects with a `code` field.

### Tariff Transformer

```typescript
function transformTariffLineItem(item: Record<string, unknown>) {
  return {
    program: (item.program as string) ?? "",
    rate: typeof item.rate === "string"
      ? parseFloat(item.rate) / 100 || 0
      : (item.rate as number) ?? 0,
    rateType: "ad_valorem" as const,
    amount: (item.amount as number) ?? 0,
    citation: (item.citation as string) ?? "",
    description: (item.description as string) ?? "",
    category: ((item.category as string) ?? "DUTY") as "DUTY" | "SURCHARGE" | "TAX" | "FEE",
    ratePct: (item.rate_pct as number) ?? 0,
    baseValue: (item.base_value as number) ?? 0,
    cumulative: (item.cumulative as number) ?? 0,
  };
}

function transformTariff(raw: Record<string, unknown>): TariffResult {
  if (!raw) return {
    htsCode: "", declaredValue: 0, currency: "USD",
    totalDuty: 0, effectiveRate: 0, programs: []
  };

  const lineItems = Array.isArray(raw.line_items)
    ? (raw.line_items as Record<string, unknown>[])
    : [];

  return {
    htsCode: (raw.hs_code as string) ?? "",
    declaredValue: (raw.declared_value as number) ?? 0,
    currency: (raw.currency as string) ?? "USD",
    totalDuty: (raw.total_duty as number) ?? 0,
    effectiveRate: (raw.effective_rate as number) ?? 0,
    programs: lineItems.map(transformTariffLineItem),
  };
}
```

**Key behaviors:**
- String rates (e.g., `"25"`) are parsed to float and divided by 100 to produce decimal rates.
- Defaults `rateType` to `"ad_valorem"` since the backend does not always provide this.

### Compliance Transformer

```typescript
function transformCompliance(raw: Record<string, unknown>): ComplianceResult {
  if (!raw) return {
    pgaFlags: [],
    dpsScreening: { entity: "", matched: false, lists: [], score: 0 },
    uflpaRisk: { level: "NONE", factors: [] },
    overallStatus: "CLEAR"
  };

  const pgaFlags = Array.isArray(raw.pga_flags)
    ? (raw.pga_flags as Record<string, unknown>[])
    : [];
  const dpsResults = Array.isArray(raw.dps_results)
    ? (raw.dps_results as Record<string, unknown>[])
    : [];
  const uflpa = (raw.uflpa_risk as Record<string, unknown>) ?? {};
  const firstDps = dpsResults[0];

  return {
    pgaFlags: pgaFlags.map((f) => ({
      agency: (f.agency as string) ?? "",
      requirement: (f.requirement as string) ?? "",
      status: ((f.mandatory as boolean) !== false ? "REQUIRED" : "CONDITIONAL"),
      details: (f.form_number as string) ?? "",
    })),
    dpsScreening: firstDps
      ? {
          entity: (firstDps.entity_name as string) ?? "",
          matched: (firstDps.match_found as boolean) ?? false,
          lists: ((firstDps.matches as Record<string, unknown>[]) ?? [])
            .map((m) => (m.list_source as string) ?? ""),
          score: (firstDps.score as number) ?? 0,
        }
      : { entity: "", matched: false, lists: [], score: 0 },
    uflpaRisk: {
      level: ((uflpa.risk_level as string) ?? "NONE") as "HIGH" | "MEDIUM" | "LOW" | "NONE",
      factors: (uflpa.flags as string[]) ?? [],
    },
    overallStatus: mapRiskToStatus((raw.overall_risk as string) ?? "LOW"),
  };
}

function mapRiskToStatus(risk: string): "CLEAR" | "REVIEW" | "HOLD" {
  switch (risk) {
    case "HIGH": return "HOLD";
    case "MEDIUM": return "REVIEW";
    default: return "CLEAR";
  }
}
```

**Key behaviors:**
- Only takes the first DPS result (the primary entity screening).
- Maps `mandatory: true/false` to `"REQUIRED"/"CONDITIONAL"` status.
- Maps overall risk level to terminal status via `mapRiskToStatus`.

## Hook Implementations and Composition

### useAnalysis -- Single-Store SSE Hook

```typescript
export function useAnalysis() {
  const store = useAnalysisStore();
  const abortRef = useRef<(() => void) | null>(null);

  const startStream = useCallback(async (input: ProductInput) => {
    abortRef.current?.();          // Cancel previous stream
    store.setProduct(input);
    store.startAnalysis();

    const abort = await streamAnalysis(input, {
      onClassificationStep: (step) => store.addStreamingStep({ ... }),
      onClassificationComplete: (result) => store.updateClassification(result),
      onTariffComplete: (result) => store.updateTariff(result),
      onComplianceComplete: (result) => store.updateCompliance(result),
      onError: (error) => store.setError(error),
      onDone: () => { /* stream completed */ },
    });

    abortRef.current = abort;
  }, [store]);

  const cancel = useCallback(() => {
    abortRef.current?.();
    abortRef.current = null;
  }, []);

  return {
    startStream,
    cancel,
    isStreaming: store.isStreaming,
    streamingSteps: store.streamingSteps,
    classification: store.classification,
    tariff: store.tariff,
    compliance: store.compliance,
    error: store.error,
  };
}
```

**Pattern:** Wraps store + API, exposes a combined API with both control functions and reactive state.

### usePipelineAnalysis -- Dual-Store SSE Hook

```typescript
export function usePipelineAnalysis() {
  const pipeline = usePipelineStore();
  const analysisStore = useAnalysisStore();
  const abortRef = useRef<(() => void) | null>(null);

  const analyze = useCallback(
    async (input: ProductInput, options: PipelineAnalysisOptions) => {
      abortRef.current?.();

      const corridor = `${input.origin}\u2192${input.destination}`;

      // 1. Create pipeline entry
      const entryId = pipeline.createEntry({
        product: input.description,
        productName: options.productName,
        origin: input.origin,
        destination: input.destination,
        declaredValue: input.declaredValue,
        currency: input.currency || "USD",
        shipper: options.shipper,
        status: "RECEIVED",
        source: options.source,
        corridor,
      });

      // 2. Optionally initialize analysis store
      if (options.updateAnalysisStore) {
        analysisStore.setProduct(input);
        analysisStore.startAnalysis();
      }

      // 3. Advance to CLASSIFYING after 300ms delay
      setTimeout(() => pipeline.updateEntryStatus(entryId, "CLASSIFYING"), 300);

      // 4. Start SSE stream with dual-dispatch callbacks
      const abort = await streamAnalysis(input, {
        onClassificationStep: (step) => {
          pipeline.updateEntryAnalysis(entryId, {
            streamingSteps: [
              ...(pipeline.getEntry(entryId)?.streamingSteps || []),
              { step: step.step, title: step.title, content: step.content },
            ],
          });
          if (options.updateAnalysisStore) {
            analysisStore.addStreamingStep({ ... });
          }
        },
        onClassificationComplete: (result) => {
          pipeline.updateEntryAnalysis(entryId, {
            classification: result,
            hsCode: result.htsCode,
            status: "CALCULATING",
          });
          if (options.updateAnalysisStore) {
            analysisStore.updateClassification(result);
          }
        },
        onTariffComplete: (result) => {
          pipeline.updateEntryAnalysis(entryId, {
            tariff: result,
            totalDuty: result.totalDuty,
            effectiveRate: result.effectiveRate,
            landedCost: result.declaredValue + result.totalDuty,
            status: "SCREENING",
          });
          if (options.updateAnalysisStore) {
            analysisStore.updateTariff(result);
          }
        },
        onComplianceComplete: (result) => {
          // Determine final status
          const finalStatus = result.overallStatus === "HOLD" ? "HOLD"
            : result.overallStatus === "REVIEW" ? "PENDING"
            : "CLEARED";

          // Extract compliance flags
          const flags: string[] = [];
          if (result.uflpaRisk.level === "HIGH") flags.push("UFLPA");
          if (result.dpsScreening.matched) flags.push("DPS");
          result.pgaFlags.forEach((f) => {
            if (f.status === "REQUIRED") flags.push(`PGA:${f.agency}`);
          });

          pipeline.updateEntryAnalysis(entryId, {
            compliance: result,
            complianceStatus: result.overallStatus,
            complianceFlags: flags,
            status: finalStatus,
          });
          if (options.updateAnalysisStore) {
            analysisStore.updateCompliance(result);
          }
        },
        onError: (error) => {
          pipeline.updateEntryAnalysis(entryId, {
            status: "EXCEPTION",
            complianceFlags: [`Error: ${error}`],
          });
          if (options.updateAnalysisStore) {
            analysisStore.setError(error);
          }
        },
      }).catch(() => {
        pipeline.updateEntryStatus(entryId, "EXCEPTION");
        return () => {};
      });

      abortRef.current = abort;
      return entryId;
    },
    [pipeline, analysisStore],
  );

  return { analyze, cancel };
}
```

**Key patterns:**
- Creates a pipeline entry synchronously, then starts the async SSE stream.
- Returns the entry ID so callers can track/navigate to the entry.
- Uses `setTimeout` for the RECEIVED-to-CLASSIFYING transition to give the UI a brief moment to show the initial state.
- Computes `landedCost` as `declaredValue + totalDuty` on the client side.
- Derives compliance flags from the structured compliance result for display purposes.
- Error fallback: if `streamAnalysis` throws, the entry is moved to EXCEPTION.

### useClassification -- Simple REST Hook

```typescript
export function useClassification() {
  const [result, setResult] = useState<ClassificationResult | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const classify = useCallback(async (input: ProductInput) => {
    setLoading(true);
    setError(null);
    setResult(null);
    try {
      const data = await classifyProduct(input);
      setResult(data);
    } catch (err) {
      setError(err instanceof Error ? err.message : "Classification failed");
    } finally {
      setLoading(false);
    }
  }, []);

  const reset = useCallback(() => {
    setResult(null);
    setError(null);
    setLoading(false);
  }, []);

  return { result, loading, error, classify, reset };
}
```

**Pattern:** Uses local React state instead of Zustand. Suitable for standalone classification requests that do not need to be shared across components.

### useSimulation -- Polling Control Hook

```typescript
export function useSimulation() {
  const [status, setStatus] = useState<SimulationStatus | null>(null);
  const [loading, setLoading] = useState(false);
  const intervalRef = useRef<ReturnType<typeof setInterval> | null>(null);

  const fetchStatus = useCallback(async () => {
    const res = await fetch(`${BASE_URL}/simulation/status`);
    if (res.ok) setStatus(await res.json());
  }, []);

  // Poll every 2 seconds
  useEffect(() => {
    fetchStatus();
    intervalRef.current = setInterval(fetchStatus, 2000);
    return () => { if (intervalRef.current) clearInterval(intervalRef.current); };
  }, [fetchStatus]);

  const start = useCallback(async () => {
    setLoading(true);
    try { await fetch(`${BASE_URL}/simulation/start`, { method: "POST" }); await fetchStatus(); }
    finally { setLoading(false); }
  }, [fetchStatus]);

  // stop, reset, setSpeed follow the same pattern

  return { status, loading, isRunning: status?.running ?? false, clock: status?.clock ?? null,
           start, stop, reset, setSpeed };
}
```

**Pattern:** Combines polling for status with imperative control actions. Each action triggers an immediate status refresh after the POST completes.

## Store Interaction Summary

```
+-------------------+     +------------------+     +-------------------+
|  pipelineStore    |     |  analysisStore   |     |  operatorStore    |
|                   |     |                  |     |                   |
| entries[]         |     | classification   |     | messages[]        |
| aggregates        |     | tariff           |     | streaming         |
| nextId            |     | compliance       |     | currentContext    |
|                   |     | streamingSteps   |     | panelOpen         |
+--------+----------+     +--------+---------+     +--------+----------+
         |                          |                        |
         |    usePipelineAnalysis   |                        |
         +----------+--------------+                        |
                    |                                        |
         +----------+----------+                  +----------+----------+
         |                     |                  |                     |
    ControlTower        ProductAnalysis     OperatorAssistant    Platform Screens
    EntryDetail                            (streamChat)       (setContext)
    ExceptionResolution
    ActiveShipments

+-------------------+     +-------------------+
|  sessionStore     |     |  cartStore        |
|                   |     |                   |
| currentSurface    |     | items[]           |
| sessionId         |     |                   |
| sidebarOpen       |     +--------+----------+
| theme             |              |
+--------+----------+     +--------+----------+
         |                |                    |
    SurfaceSwitcher   StoreFront          CheckoutPrice
    All Layouts       ProductPage         BuyerLayout
```
