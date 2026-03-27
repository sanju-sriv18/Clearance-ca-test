# Global Gap Analysis: Unified Multi-Jurisdiction Broker UX

> **Design Mandate**: "One simple, intelligent, and intuitive experience allowing a broker to work any import/export."
>
> Every jurisdiction is first-class. No "US mode" vs "EU mode." One adaptive interface.

---

## Table of Contents

1. [Jurisdiction Abstraction Layer](#1-jurisdiction-abstraction-layer)
2. [Smart Context Detection](#2-smart-context-detection)
3. [Universal Data Model Design](#3-universal-data-model-design)
4. [Document Framework](#4-document-framework)
5. [Regulatory Intelligence Panel](#5-regulatory-intelligence-panel)
6. [Agent/Assistant Intelligence](#6-agentassistant-intelligence)
7. [Dashboard/Queue Universality](#7-dashboardqueue-universality)
8. [Navigation Architecture](#8-navigation-architecture)
9. [Terminology Strategy](#9-terminology-strategy)
10. [Implementation Strategy](#10-implementation-strategy)

---

## Current State Audit

### What exists today (US-only hardcoding)

| Layer | US-Hardcoded Element | File(s) |
|-------|---------------------|---------|
| **Router** | `/broker/cbp-responses` route name | `router.tsx` |
| **EntryDetail** | "Submit to CBP" button, CF-28/CF-29 modal, MPF/HMF labels, bond verification | `EntryDetail.tsx` |
| **Dashboard** | "CBP Response" stat tile, CF-28/CF-29 status badges | `BrokerDashboard.tsx` |
| **Queue** | "CF-28 Pending" / "CF-29 Pending" status filters | `BrokerQueue.tsx` |
| **Checklist** | `FTA_PARTNERS_US`, `US_PORT_CODES`, bond type options | `broker.py` |
| **Agent Tools** | `draft_cf28_response`, `calculate_entry_fees` (MPF/HMF/Section 301) | `tools.py` |
| **Data Model** | `EntryFiling.entry_type` defaults to "01" (US consumption entry), `filing_status` includes `cf28_pending`/`cf29_pending` | `operational.py` |
| **Tariff Engine** | Destination defaults to `"US"`, regime dispatch already multi-jurisdiction | `engine.py` |

### What exists today (jurisdiction-aware -- good patterns)

| Component | Pattern | Why it's good |
|-----------|---------|---------------|
| **TariffEngine** | `REGIME_MAP` dispatch by destination country | Already supports US/EU/CN/BR/IN with regime-specific tax stacks |
| **DocumentRequirements** | Takes `origin`, `destination`, `complianceFlags` as props | Already parameterized -- does not assume US |
| **Shipment model** | Has `origin`, `destination`, `transport_mode` | Universal fields already in place |
| **Multi-modal routing** | Routing engine handles transshipment hubs | Regulatory touchpoints already modeled |

---

## 1. Jurisdiction Abstraction Layer

### Universal vs. Jurisdiction-Specific Concepts

```
UNIVERSAL (every declaration)          JURISDICTION-SPECIFIC (varies)
─────────────────────────────          ──────────────────────────────
HS Code (harmonized to 6 digits)       Extended tariff code (HTS-10, CN-10, NCM-10, ITC-8)
Declared value                         Filing form (CF-7501, SAD/CDS, Bill of Entry, DI/DUIMP)
Origin country                         Tax components (duty + MPF + HMF vs VAT vs IGST + SWS vs ICMS cascade)
Destination country                    Customs authority (CBP, HMRC, EU Customs, ICEGATE, Receita Federal)
Weight / quantity                      Regulatory bodies (FDA vs FSSAI vs ANVISA vs CE marking)
Parties (importer, exporter, broker)   Document types (CF-28 vs BTI ruling vs advance ruling)
Transport mode                         Filing channels (ACE vs ICS2 vs ICEGATE EDI vs SISCOMEX)
Documents (invoice, packing list, BoL) Importer ID format (EIN vs EORI vs IEC vs CNPJ/RADAR)
                                       Bond/guarantee requirements (continuous bond vs bank guarantee)
                                       Currency/valuation method
                                       Status vocabulary (released vs "libre circulación" vs "desembaraçado")
```

### How EntryDetail Renders Differently Per Jurisdiction

The key insight: EntryDetail should be a **single component** that renders a **jurisdiction configuration object** -- not four separate screens.

#### Jurisdiction Configuration Schema

```typescript
interface JurisdictionConfig {
  // Identity
  code: string;                    // "US", "EU", "IN", "BR", "CN"
  name: string;                    // "United States", "European Union", etc.
  customsAuthority: string;        // "CBP", "EU Customs", "ICEGATE", "Receita Federal"
  flag: string;                    // 🇺🇸, 🇪🇺, 🇮🇳, 🇧🇷, 🇨🇳

  // Filing
  declarationType: string;         // "Entry", "Customs Declaration", "Bill of Entry", "DI"
  declarationFormCode: string;     // "CF-7501", "SAD/CDS", "BoE", "DI/DUIMP"
  importerIdLabel: string;         // "EIN", "EORI", "IEC", "CNPJ"
  portLabel: string;               // "Port of Entry", "Customs Office", "Port", "Recinto"

  // Tax breakdown labels
  taxComponents: TaxComponentDef[];
  // e.g. US: [{label:"Duty"}, {label:"MPF"}, {label:"HMF"}]
  // e.g. EU: [{label:"MFN Duty"}, {label:"VAT", suffix:"%"}]
  // e.g. IN: [{label:"BCD"}, {label:"SWS"}, {label:"IGST"}]
  // e.g. BR: [{label:"II"}, {label:"IPI"}, {label:"PIS"}, {label:"COFINS"}, {label:"ICMS"}]

  // Regulatory inquiry vocabulary
  inquiryTypes: InquiryTypeDef[];
  // US: [{code:"cf28", label:"Request for Information"}, {code:"cf29", label:"Notice of Action"}]
  // EU: [{code:"bti", label:"Binding Tariff Information"}, {code:"pre_arrival", label:"Pre-arrival Declaration"}]
  // IN: [{code:"advance_ruling", label:"Advance Ruling"}, {code:"dri_query", label:"DRI Query"}]
  // BR: [{code:"parametrizacao", label:"Parametrização Channel"}, {code:"exigencia", label:"Exigência Fiscal"}]

  // Checklist items (ordered)
  checklistItems: ChecklistDef[];

  // Status lifecycle
  statusFlow: StatusFlowDef;

  // Regulatory agencies
  agencies: AgencyDef[];

  // Bond/guarantee
  guaranteeConfig: GuaranteeConfig;
}
```

#### Concrete Examples: Same Screen, Four Jurisdictions

**US Import (CN → US)**

```
┌─────────────────────────────────────────────────────────┐
│  🇺🇸 Entry  #ENT-2025-0142    [Draft]                    │
│  Apex Trading Co. · CN → US                             │
│                                                         │
│  Shipment Details                                       │
│  Entry Type: 01 — Consumption Entry                     │
│  Port of Entry: 2704 — Los Angeles, CA                  │
│  Importer EIN: 12-3456789                               │
│                                                         │
│  Tax Summary                                            │
│  ├─ Duty (6.5%)              $1,560.00                  │
│  ├─ MPF (0.3464%, min $29.66, max $575.35)    $83.14    │
│  ├─ HMF (0.125%)            $30.00                      │
│  └─ Sec 301 (25.0%)         $6,000.00                   │
│  Total                       $7,673.14                  │
│                                                         │
│  Pre-Filing Verification                                │
│  ✅ Commercial Invoice                                   │
│  ✅ Packing List                                         │
│  ✅ Bill of Lading                                       │
│  ✅ HS Classification (HIGH confidence)                  │
│  ⚠️ Bond Verification — No bond type set                │
│  ✅ Customs Valuation                                    │
│  ─  Certificate of Origin (not applicable — no FTA)     │
│  ─  PGA Documentation (not applicable)                  │
│                                                         │
│  [Approve & Sign]   [Submit to CBP]                     │
└─────────────────────────────────────────────────────────┘
```

**EU Import to Germany (CN → DE)**

```
┌─────────────────────────────────────────────────────────┐
│  🇪🇺 Declaration  #DE-2025-A-0087    [Entwurf]          │
│  EuroTech GmbH · CN → DE                               │
│                                                         │
│  Shipment Details                                       │
│  Declaration Type: H1 — Free Circulation                │
│  Customs Office: DE003102 — Hamburg Hafen                │
│  Importer EORI: DE123456789000                          │
│                                                         │
│  Tax Summary                                            │
│  ├─ MFN Duty (6.5%)         €1,430.00                   │
│  ├─ Anti-dumping Duty        €0.00                      │
│  └─ VAT (19% DE)            €4,561.70                   │
│  Total                       €5,991.70                  │
│                                                         │
│  Pre-Filing Verification                                │
│  ✅ Commercial Invoice                                   │
│  ✅ Packing List                                         │
│  ✅ Transport Document (Bill of Lading)                  │
│  ✅ CN Code Classification                               │
│  ✅ Customs Valuation (D.V.1)                            │
│  ✅ EORI Verification                                    │
│  ⚠️ CE Marking Declaration — required for Chapter 85    │
│  ⚠️ REACH Compliance — substance pre-registration       │
│  ─  EUR.1 / REX (not applicable — CN is not PEM)       │
│                                                         │
│  [Validate & Sign]   [Submit to ICS2]                   │
└─────────────────────────────────────────────────────────┘
```

**India Import (CN → IN)**

```
┌─────────────────────────────────────────────────────────┐
│  🇮🇳 Bill of Entry  #BOE/2025/MUM/0532    [Draft]       │
│  TechImports Pvt Ltd · CN → IN                          │
│                                                         │
│  Shipment Details                                       │
│  Entry Type: Home Consumption                           │
│  Custom House: Mumbai (Nhava Sheva)                     │
│  Importer IEC: 0305012345                               │
│                                                         │
│  Tax Summary                                            │
│  ├─ BCD (Basic Customs Duty, 15%)     ₹2,52,000        │
│  ├─ SWS (Social Welfare Surcharge, 10% of BCD) ₹25,200 │
│  └─ IGST (Integrated GST, 18%)       ₹4,98,960        │
│  Total                                ₹7,76,160        │
│                                                         │
│  Pre-Filing Verification                                │
│  ✅ Commercial Invoice                                   │
│  ✅ Packing List                                         │
│  ✅ Bill of Lading                                       │
│  ✅ ITC-HS Classification                                │
│  ✅ Customs Valuation                                    │
│  ⚠️ BIS Certificate — required for electronics (Ch. 85)│
│  ❌ FSSAI Import License — required for food contact    │
│  ─  Certificate of Origin (not applicable)              │
│                                                         │
│  [Approve & Sign]   [File via ICEGATE]                  │
└─────────────────────────────────────────────────────────┘
```

**Brazil Import (CN → BR)**

```
┌─────────────────────────────────────────────────────────┐
│  🇧🇷 DI  #25/0142398-7    [Rascunho]                    │
│  Comércio Global Ltda · CN → BR                         │
│                                                         │
│  Shipment Details                                       │
│  Declaration Type: Importação Definitiva                │
│  Recinto: Santos — Porto de Santos                      │
│  Importer CNPJ: 12.345.678/0001-90                      │
│  RADAR Status: ✅ Habilitado (Ilimitado)                │
│                                                         │
│  Tax Summary (cascading)                                │
│  ├─ II (Imposto de Importação, 14%)     R$7.560        │
│  ├─ IPI (10%)                            R$6.156        │
│  ├─ PIS-Importação (2.1%)               R$1.134        │
│  ├─ COFINS-Importação (9.65%)           R$5.211        │
│  └─ ICMS (SP 18%, grossup)             R$17.574        │
│  Total                                  R$37.635        │
│                                                         │
│  Pre-Filing Verification                                │
│  ✅ Commercial Invoice (Fatura Comercial)                │
│  ✅ Packing List (Romaneio)                              │
│  ✅ Bill of Lading (Conhecimento de Embarque)            │
│  ✅ NCM Classification                                   │
│  ✅ Customs Valuation                                    │
│  ⚠️ RADAR Verification — expiry approaching             │
│  ⚠️ LI (Import License) — required for NCM 8518.xx     │
│  ❌ NF-e Template — required for nacionalização         │
│                                                         │
│  Parametrização Channel: [Awaiting assignment]          │
│  [Validate & Sign]   [Register via SISCOMEX]            │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Smart Context Detection

### Trade Lane Inference

The system should automatically detect the customs jurisdiction from the destination country. This is already partially implemented in the tariff engine's `REGIME_MAP`.

```
Trade Lane             Customs Authority      Regime
─────────────────────  ─────────────────────  ──────
CN → US                U.S. CBP               US
CN → DE                EU Customs (via DE)    EU
CN → FR                EU Customs (via FR)    EU
VN → IN                Indian Customs         IN
CN → BR                Receita Federal        BR
DE → US                U.S. CBP               US
CN → SG → DE           EU Customs (via DE)    EU    ← Singapore is transit only
CN → CDG → US          U.S. CBP               US    ← CDG is transit only
```

### Detection Cascade

```
destination_country
  │
  ├─ Is it an EU member? ──→ jurisdiction = "EU", member_state = destination
  │    (27 countries: AT, BE, BG, ... already in EU_MEMBERS frozenset)
  │
  ├─ Direct match in REGIME_MAP? ──→ jurisdiction = destination
  │    (US, CN, BR, IN)
  │
  └─ Unknown? ──→ Show "Unsupported jurisdiction" banner with fallback to generic view
```

### Transshipment Handling

The routing engine already models transshipment hubs with `regulatory_touchpoints`. The rule:

- **Customs jurisdiction = final destination**, NOT transit points
- Transit points may require transit declarations (e.g., EU T1 transit doc, Singapore transshipment permit)
- The UI should show transit stops in the shipment timeline but apply customs rules from the destination

```
CN → SG → DE:
  - Singapore: transit permit only (no customs declaration)
  - Germany: full EU customs declaration (SAD/CDS)
  - UI shows: 🇪🇺 Declaration, with Singapore transit leg in the shipment timeline

CN → CDG → US:
  - CDG (France): transit — ISF advance manifest for US
  - US: full CBP entry (CF-7501)
  - UI shows: 🇺🇸 Entry, with CDG transit leg in timeline
```

### Dual Declarations

Some trade lanes require declarations at BOTH origin and destination:

```
CN → US (export + import):
  - China export: 报关单 (customs declaration) at origin
  - US import: CF-7501 entry at destination
  - The broker surface focuses on the IMPORT side
  - Export declarations shown as "counterpart filings" in a collapsed section
```

This is a Phase 2 concern. Initially, the broker surface focuses on the import declaration.

---

## 3. Universal Data Model Design

### Approach: Core + Jurisdiction Extensions (JSONB)

The recommended approach is **core universal fields + jurisdiction-specific JSONB extensions**. This avoids schema changes per jurisdiction while maintaining type safety at the application layer.

### Current → Proposed Model Changes

#### EntryFiling → Declaration

```sql
-- Rename table: entry_filings → declarations (migration alias approach)
ALTER TABLE entry_filings RENAME TO declarations;

-- Add new columns
ALTER TABLE declarations ADD COLUMN jurisdiction VARCHAR(5) NOT NULL DEFAULT 'US';
ALTER TABLE declarations ADD COLUMN jurisdiction_config JSONB;
-- jurisdiction_config stores:
-- {
--   "declaration_form": "CF-7501" | "SAD" | "BoE" | "DI",
--   "customs_authority": "CBP" | "EU_CUSTOMS" | "ICEGATE" | "RECEITA",
--   "importer_id_type": "EIN" | "EORI" | "IEC" | "CNPJ",
--   "importer_id_value": "12-3456789",
--   "filing_channel": "ACE" | "ICS2" | "ICEGATE_EDI" | "SISCOMEX",
--   "currency": "USD" | "EUR" | "INR" | "BRL",
--   "member_state": "DE" | null,      -- EU only
--   "radar_status": "habilitado" | null, -- BR only
--   "parametrizacao_channel": "green" | "yellow" | "red" | null, -- BR only
-- }

-- Generalize filing_status values
-- Keep existing: draft, pending_broker_approval, submitted, released, rejected, exam_scheduled
-- Rename: cf28_pending → inquiry_pending, cf29_pending → notice_pending
-- Add: validated (EU pre-check), channeled (BR parametrização assigned)
ALTER TABLE declarations ADD COLUMN inquiry_type VARCHAR(30);
-- "cf28" | "cf29" | "bti" | "advance_ruling" | "dri_query" | "exigencia" | "parametrizacao"
```

#### Tax Breakdown (already in shipment.financials JSONB)

The tariff engine already returns jurisdiction-specific tax breakdowns via `TariffResult.line_items`. The frontend just needs to read and render the line items array rather than hardcoding MPF/HMF labels:

```typescript
// Current (hardcoded):
<span>MPF</span> <span>${insights.estimatedMpf}</span>
<span>HMF</span> <span>${insights.estimatedHmf}</span>

// Proposed (driven by data):
{taxLineItems.map(item => (
  <div key={item.code}>
    <span>{item.label}</span>
    <span>{formatCurrency(item.amount, currency)}</span>
  </div>
))}
```

#### Checklist Items (generalized)

Instead of hardcoding 8 US-specific checklist items, the checklist should be driven by a jurisdiction configuration:

```typescript
// Universal checklist items (all jurisdictions):
const UNIVERSAL_CHECKLIST = [
  "commercial_invoice",
  "packing_list",
  "transport_document",  // renamed from "bill_of_lading"
  "classification",
  "valuation",
];

// Jurisdiction-specific additions:
const JURISDICTION_CHECKLIST: Record<string, string[]> = {
  US: ["origin_cert", "pga_docs", "bond_verification"],
  EU: ["eori_verification", "ce_marking", "reach_compliance", "eur1_rex"],
  IN: ["iec_verification", "bis_certificate", "fssai_license", "advance_license"],
  BR: ["radar_verification", "import_license_li", "nfe_template", "siscomex_registration"],
  CN: ["ccc_certificate", "ciq_inspection", "import_license"],
};
```

---

## 4. Document Framework

### Extending DocumentRequirements.tsx

`DocumentRequirements.tsx` is the best existing pattern -- it already takes `origin`, `destination`, `hsCode`, and `complianceFlags` as props and dynamically loads requirements from the backend. It needs minimal changes:

#### Current (good)
```typescript
getDocumentRequirements({ hsCode, origin, destination, complianceFlags })
```

#### Enhancement: Jurisdiction-Aware Labels

Instead of hardcoded document names, the backend should return jurisdiction-localized labels:

```json
// US response:
{ "document_type": "Certificate of Origin", "citation": "19 CFR 10.1" }

// EU response:
{ "document_type": "EUR.1 Movement Certificate", "citation": "UCC Art. 64" }

// India response:
{ "document_type": "Certificate of Origin (Non-Preferential)", "citation": "Customs Act 1962, Section 25" }
```

The frontend doesn't change -- it already renders whatever `document_type` string comes back.

### Regulatory Inquiry Response Generalization

The current `CBPResponses` screen and `ResponseModal` are tightly coupled to CF-28/CF-29. The generalization:

```
Current (US-only)                    Proposed (universal)
──────────────────                   ────────────────────
CBPResponses screen                  RegulatoryInquiries screen
respondCF28()                        respondToInquiry(entryId, inquiryType, response)
respondCF29()                        respondToNotice(entryId, noticeType, action)
"CF-28 Request for Information"      "[Authority] Request for Information"
"CF-29 Notice of Action"             "[Authority] Notice of Action"
draft_cf28_response tool             draft_inquiry_response tool
```

The UI adapts the header, guidance text, and attachment recommendations based on inquiry type:

| Inquiry Type | Authority | UI Label | Guidance |
|-------------|-----------|----------|----------|
| `cf28` | CBP | "CBP Request for Information" | Classification/valuation/origin evidence |
| `cf29` | CBP | "CBP Notice of Action" | Accept/protest within deadline |
| `bti` | EU Customs | "Binding Tariff Information" | Classification ruling application |
| `advance_ruling` | ICEGATE | "Advance Ruling Request" | Classification/valuation pre-import ruling |
| `exigencia` | Receita Federal | "Exigência Fiscal" | Additional documentation/clarification |
| `parametrizacao` | Receita Federal | "Parametrização Channel" | Channel assignment (green/yellow/red/grey) |

### Document Generation Templates

The existing `generateDocument()` + `generateDocumentPDF()` pipeline works universally. The jurisdiction config determines which templates are available:

- US: Commercial Invoice (US format), CF-7501 summary, Power of Attorney
- EU: EUR.1, D.V.1 (customs value declaration), Intrastat declaration
- India: Bill of Entry format, ICEGATE upload manifest
- Brazil: Fatura Comercial, DI extract, NF-e skeleton

---

## 5. Regulatory Intelligence Panel

### Universal Component: `<RegulatoryRequirements>`

A single component that renders jurisdiction-aware regulatory agencies and their requirements:

```
┌─ Regulatory Requirements ──────────────────────────┐
│                                                     │
│  US Import (CN → US)           │  EU Import (CN → DE)
│  ────────────────              │  ────────────────
│  🏛 FDA — food/drug/cosmetic   │  🏛 CE Marking — safety
│  🏛 EPA — environmental        │  🏛 REACH — chemicals
│  🏛 CPSC — consumer safety     │  🏛 ICS2 — advance cargo info
│  🏛 FCC — electronics          │  🏛 RAPEX — product safety
│  🏛 UFLPA — forced labor       │
│                                │
│  India Import (CN → IN)        │  Brazil Import (CN → BR)
│  ────────────────              │  ────────────────
│  🏛 BIS — standards cert       │  🏛 ANVISA — health/pharma
│  🏛 FSSAI — food safety        │  🏛 MAPA — agriculture
│  🏛 CDSCO — drugs/devices      │  🏛 INMETRO — metrology/safety
│  🏛 WPC — wireless equipment   │  🏛 IBAMA — environmental
│  🏛 AERB — atomic energy       │  🏛 ANATEL — telecom
└─────────────────────────────────────────────────────┘
```

### Implementation Pattern

```typescript
interface RegulatoryAgency {
  code: string;          // "FDA", "BIS", "ANVISA"
  name: string;          // Full name
  jurisdiction: string;  // "US", "IN", "BR"
  triggerHsCodes: string[]; // HS prefixes that trigger this agency
  requiredDocuments: string[];
  website: string;
  filingPortal?: string;
}

// The compliance engine (E3) already returns PGA flags for US.
// Extend it to return equivalent flags for other jurisdictions.
```

The backend `check_compliance` tool currently checks "PGA requirements, denied party screening, and UFLPA forced labor risk." This should be extended per jurisdiction:

- US: PGA + UFLPA + DPS screening (already done)
- EU: CE marking + REACH + ICS2 pre-arrival + dual-use export controls
- India: BIS certification + FSSAI license + WPC approval + legal metrology
- Brazil: ANVISA + MAPA + INMETRO + IBAMA + Exército (controlled goods)
- China: CCC (compulsory certification) + CIQ inspection + import license system

---

## 6. Agent/Assistant Intelligence

### Jurisdiction-Aware Tool Dispatch

The AI assistant should automatically adapt its tool selection based on the jurisdiction context. The `tools.py` tool schemas need jurisdiction awareness:

#### Current Tools → Generalized

| Current Tool | Generalized Approach |
|-------------|---------------------|
| `draft_cf28_response` | → `draft_inquiry_response` with `inquiry_type` parameter |
| `calculate_entry_fees` | → Already works (tariff engine dispatches by destination) |
| `check_compliance` | → Extend to include jurisdiction-specific checks |
| `check_entry_readiness` | → Checklist items vary by jurisdiction (use config) |
| `verify_classification` | → Same logic, different code systems (HTS vs CN vs ITC-HS vs NCM) |

#### Context-Aware Prompting

When the assistant receives a declaration context, it should automatically adjust:

```
User: "Draft a response"
  │
  ├─ US jurisdiction, cf28_pending
  │   → "I'll draft a CF-28 response addressing CBP's classification inquiry..."
  │
  ├─ EU jurisdiction, bti_pending
  │   → "I'll prepare a BTI (Binding Tariff Information) application for EU customs..."
  │
  ├─ India jurisdiction, advance_ruling_pending
  │   → "I'll draft an advance ruling request for Indian customs under Section 28H..."
  │
  └─ Brazil jurisdiction, exigencia_pending
      → "I'll prepare the exigência response for Receita Federal..."
```

#### New Tool: `get_jurisdiction_context`

```python
{
    "name": "get_jurisdiction_context",
    "description": (
        "Get the customs jurisdiction context for a declaration or shipment. "
        "Returns the applicable customs authority, required regulatory agencies, "
        "filing requirements, and jurisdiction-specific terminology."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "destination_country": {"type": "string"},
            "origin_country": {"type": "string"},
            "hs_code": {"type": "string"},
        },
        "required": ["destination_country"],
    },
}
```

---

## 7. Dashboard/Queue Universality

### Unified Broker Dashboard

The dashboard should show ALL declarations across jurisdictions, unified by default, with smart filtering:

#### Stats Summary (generalized)

```
Current (US-only):
  Pre-Filing | Filed | CBP Response | Exam | Released | Held | Needs Entry

Proposed (universal):
  Pre-Filing | Filed | Authority Response | Under Exam | Released | Held | Unassigned

  + Jurisdiction breakdown as a subtle secondary row:
  🇺🇸 12  🇪🇺 8  🇮🇳 3  🇧🇷 2
```

#### Status Badges (generalized)

| Current (US-only) | Proposed (universal) |
|-------------------|---------------------|
| `cf28_pending` → "CF-28" | `inquiry_pending` → Shows jurisdiction-specific label |
| `cf29_pending` → "CF-29" | `notice_pending` → Shows jurisdiction-specific label |
| "Submit to CBP" | "Submit to [Authority]" |
| "Awaiting CBP response" | "Awaiting [Authority] response" |

```typescript
function StatusBadge({ status, jurisdiction }: { status: string; jurisdiction: string }) {
  if (status === "inquiry_pending") {
    // Show jurisdiction-specific inquiry type
    const label = JURISDICTION_CONFIGS[jurisdiction]?.inquiryTypes[0]?.label || "Inquiry";
    return <Badge color="orange">{label}</Badge>;
  }
  // ... generic statuses work the same
}
```

#### KPIs with Per-Jurisdiction Benchmarks

```
Average Clearance Time
├─ 🇺🇸 US: 2.3 days (benchmark: 2.5 days)
├─ 🇪🇺 EU: 1.8 days (benchmark: 2.0 days)
├─ 🇮🇳 India: 5.2 days (benchmark: 6.0 days)
└─ 🇧🇷 Brazil: 7.1 days (benchmark: 8.0 days)
```

#### Queue Filters

```
Current:                              Proposed:
  Status: [All | Draft | CF-28...]      Status: [All | Draft | Inquiry | Notice...]
  Priority: [All | Urgent...]           Priority: [All | Urgent...]
                                        Jurisdiction: [All | 🇺🇸 US | 🇪🇺 EU | 🇮🇳 India | 🇧🇷 Brazil]
                                        Trade Lane: [All | CN→US | CN→DE | ...]
```

### Work Plan Integration

The morning briefing work plan already works generically (it's driven by urgency + deadlines). The only change needed is replacing US-specific action labels:

```
Current: "Respond to CF-28 — Apex Trading"
Proposed: "Respond to inquiry — Apex Trading (🇺🇸 CF-28)"
```

---

## 8. Navigation Architecture

### Principle: Jurisdiction is a Property, Not a Mode

**No separate surfaces.** No `/broker/us/entries` vs `/broker/eu/entries`. Jurisdiction is a property of each declaration, just like status or priority.

#### Route Structure (no changes needed)

```
/broker
  /dashboard           ← shows ALL jurisdictions
  /queue               ← filterable by jurisdiction
  /declarations/:id    ← (rename from /entries/:id) adapts to jurisdiction
  /regulatory-inquiries ← (rename from /cbp-responses) shows all inquiry types
  /messages            ← unchanged
```

The route rename from `/entries/:id` to `/declarations/:id` is optional and could be done with a redirect. See Terminology Strategy below.

#### Jurisdiction Indicator

Each declaration in the queue should show a jurisdiction flag:

```
🇺🇸 ENT-2025-0142  |  Ceramic Tableware  |  Apex Trading  |  [Draft]  |  2.1d
🇪🇺 DE-2025-A-0087  |  LED Panels        |  EuroTech GmbH |  [Filed]  |  1.8d
🇮🇳 BOE/2025/0532   |  Speakers           |  TechImports   |  [Inquiry] | 5.2d
```

Flag icons are small, universally understood, and avoid the need for text labels. They can be accompanied by a tooltip showing the full jurisdiction name.

---

## 9. Terminology Strategy

### Recommendation: **Adaptive Terminology with User Preference**

**Option (b) — adapt term to jurisdiction** is the recommended approach, with a global neutral fallback.

#### Rationale

- Brokers are domain experts. A US broker expects "entry," an EU broker expects "declaration," an Indian broker expects "Bill of Entry."
- Using jurisdiction-incorrect terminology would undermine credibility.
- The system is already jurisdiction-aware from the trade lane, so it can pick the right term.

#### Implementation

```typescript
const TERMINOLOGY: Record<string, Record<string, string>> = {
  US: {
    declaration: "Entry",
    declarationType: "Entry Type",
    customsAuthority: "CBP",
    submitAction: "Submit to CBP",
    inquiryType: "CF-28/CF-29",
    portLabel: "Port of Entry",
    importerId: "EIN",
  },
  EU: {
    declaration: "Declaration",
    declarationType: "Declaration Type",
    customsAuthority: "EU Customs",
    submitAction: "Submit to Customs",
    inquiryType: "BTI/Inquiry",
    portLabel: "Customs Office",
    importerId: "EORI",
  },
  IN: {
    declaration: "Bill of Entry",
    declarationType: "Entry Type",
    customsAuthority: "Indian Customs",
    submitAction: "File via ICEGATE",
    inquiryType: "Advance Ruling",
    portLabel: "Custom House",
    importerId: "IEC",
  },
  BR: {
    declaration: "DI",
    declarationType: "Tipo de Declaração",
    customsAuthority: "Receita Federal",
    submitAction: "Register via SISCOMEX",
    inquiryType: "Exigência",
    portLabel: "Recinto Aduaneiro",
    importerId: "CNPJ",
  },
};

// In the UI:
const terms = TERMINOLOGY[jurisdiction] || TERMINOLOGY.US;
<h1>{terms.declaration} #{declarationNumber}</h1>
<button>{terms.submitAction}</button>
```

#### URL Slugs

URLs use the neutral term `declarations` (not jurisdiction-specific):
```
/broker/declarations/:id    ← always this, regardless of jurisdiction
```

But the page title, breadcrumbs, and button labels adapt:
```
🇺🇸 context: "Entry ENT-2025-0142"
🇪🇺 context: "Declaration DE-2025-A-0087"
```

---

## 10. Implementation Strategy

### Phase 0: Foundation (frontend-only, no backend changes)

**Goal**: Abstract away US-specific labels and prepare the component architecture.

1. **Create `JurisdictionConfig` type and registry** — TypeScript interfaces + a config map keyed by country code. Initially only US is populated with real data; others are stubs.

2. **Extract hardcoded labels** — Replace hardcoded "CBP", "CF-28", "MPF", "HMF" in `EntryDetail.tsx`, `BrokerDashboard.tsx`, `BrokerQueue.tsx` with config lookups. These components already receive `entry.destination` or similar — thread it through.

3. **Generalize `StatusBadge`** — Currently has hardcoded `cf28_pending` / `cf29_pending` labels. Make these driven by a jurisdiction + status → label mapping.

4. **Rename internal references** — `CBPResponses` screen → `RegulatoryInquiries` (with redirect for bookmarks). `entry` terminology → `declaration` where it appears in UI text.

5. **Add jurisdiction filter to queue** — A dropdown in `BrokerQueue` that filters by `destination` country. No backend change needed — the filter can be applied client-side or added as a query param.

**Estimated effort**: 2-3 days. Zero backend changes. All existing tests pass unchanged.

### Phase 1: Data Model Generalization (backend)

**Goal**: Make the data model jurisdiction-aware.

1. **Add `jurisdiction` + `jurisdiction_config` columns to `EntryFiling`** — Populate on creation based on shipment destination. Existing entries default to `"US"`.

2. **Generalize `filing_status`** — Add `inquiry_pending` and `notice_pending` as aliases/replacements for `cf28_pending`/`cf29_pending`. Keep the old values as valid for backward compatibility during migration.

3. **Generalize checklist computation** — `_compute_checklist_state()` in `broker.py` currently hardcodes 8 items. Refactor to load checklist items from a jurisdiction config. US config reproduces the existing 8 items exactly.

4. **Extend `check_compliance` response** — Include jurisdiction-specific agency flags (not just PGA). The E3 engine already returns structured data; add `regulatory_agencies` to the response.

5. **Add jurisdiction-aware entry number generation** — Currently generates `ENT-YYYY-NNNN`. Add formats for other jurisdictions.

**Estimated effort**: 4-5 days. Full backward compatibility.

### Phase 2: Per-Jurisdiction Filing Workflows

**Goal**: Support actual filing differences.

1. **EU filing workflow** — ICS2 pre-arrival, SAD/CDS fields, EORI validation, CE marking requirements, VAT calculation with member-state rates.

2. **India filing workflow** — ICEGATE EDI fields, BIS/FSSAI/WPC agency checks, ITC-HS code validation, Bill of Entry format.

3. **Brazil filing workflow** — SISCOMEX integration fields, RADAR status check, parametrização channel assignment, cascading tax display (the tariff engine already computes this), NF-e linkage.

4. **Generalized inquiry/response system** — Replace `respondCF28`/`respondCF29` API endpoints with `respondToInquiry(entryId, inquiryType, response)`. Backend dispatches to jurisdiction-specific handlers.

**Estimated effort**: 2-3 weeks per jurisdiction.

### Phase 3: Simulation Actor Generalization

**Goal**: Make the simulation generate multi-jurisdiction scenarios.

Currently, simulation actors generate US-only scenarios (CBP status transitions, CF-28 issuance, etc.). Generalize:

1. **Customs authority actor** — Parameterized by jurisdiction. Generates jurisdiction-appropriate status transitions, inquiry issuance, and exam scheduling.

2. **Multi-jurisdiction scenario generator** — Creates a mix of trade lanes (e.g., 60% US, 20% EU, 10% India, 10% Brazil) with jurisdiction-appropriate companies, products, and regulatory situations.

3. **Jurisdiction-specific timing** — Average clearance times vary: US ~2.5 days, EU ~2 days, India ~6 days, Brazil ~8 days.

**Estimated effort**: 1 week.

### Migration Path Summary

```
Phase 0 (frontend)    │  Phase 1 (backend)     │  Phase 2 (workflows)    │  Phase 3 (simulation)
──────────────────    │  ─────────────────────  │  ────────────────────   │  ─────────────────────
Config registry        │  jurisdiction column    │  EU ICS2 workflow       │  Multi-jurisdiction
Label extraction       │  Generalized status     │  India ICEGATE          │  scenario generator
StatusBadge refactor   │  Checklist config       │  Brazil SISCOMEX        │  Customs authority
Jurisdiction filter    │  Compliance extension   │  Inquiry/response API   │  actor per jurisdiction
Route rename           │  Entry number formats   │  Per-jurisdiction agent │  Timing parameters
                       │                         │  tools                  │
──────────────────    │  ─────────────────────  │  ────────────────────   │  ─────────────────────
~3 days                │  ~5 days                │  ~6 weeks total         │  ~1 week
```

### What Requires NO Changes

- **Tariff engine** — Already multi-jurisdiction (`REGIME_MAP` dispatches correctly)
- **DocumentRequirements component** — Already parameterized by origin/destination
- **Shipment model** — Already has origin/destination/transport_mode
- **Multi-modal routing** — Already handles transshipment hubs
- **Core data flow** — Order → Shipment → Declaration pipeline works universally

---

## Appendix: Jurisdiction Config Reference

### US (United States)

```yaml
code: US
customs_authority: CBP (Customs and Border Protection)
filing_system: ACE (Automated Commercial Environment)
declaration_form: CF-7501 (Entry Summary)
importer_id: EIN (Employer Identification Number)
entry_types: ["01-Consumption", "11-Informal", "06-FTZ"]
tax_components: [Duty, MPF, HMF, Sec301, Sec232, AD/CVD]
regulatory_bodies: [FDA, EPA, CPSC, FCC, APHIS, TTB, ATF]
forced_labor: UFLPA (Uyghur Forced Labor Prevention Act)
bond_requirement: Continuous or Single Transaction
key_inquiries: [CF-28 (RFI), CF-29 (Notice of Action)]
screening: OFAC SDN, BIS Entity List, DPL
```

### EU (European Union)

```yaml
code: EU
customs_authority: DG TAXUD / National Customs Administrations
filing_system: ICS2 (Import Control System 2) + national systems
declaration_form: SAD (Single Administrative Document) or CDS
importer_id: EORI (Economic Operators Registration and Identification)
entry_types: ["H1-Free Circulation", "H2-Customs Warehousing", "H3-Temporary Admission"]
tax_components: [MFN Duty, Anti-dumping, VAT (member-state rate)]
regulatory_bodies: [CE Marking, REACH, RoHS, RAPEX, ICS2]
forced_labor: EU Due Diligence Regulation (proposed)
guarantee_requirement: Comprehensive guarantee (bank/insurance)
key_inquiries: [BTI (Binding Tariff Information), BOI (Binding Origin Information)]
screening: EU Sanctions Lists, Dual-Use Regulation
```

### India (IN)

```yaml
code: IN
customs_authority: CBIC (Central Board of Indirect Taxes and Customs)
filing_system: ICEGATE (Indian Customs EDI Gateway)
declaration_form: Bill of Entry
importer_id: IEC (Importer Exporter Code)
entry_types: ["Home Consumption", "Warehousing", "Ex-Bond"]
tax_components: [BCD, SWS (10% of BCD), IGST (18%/28%)]
regulatory_bodies: [BIS, FSSAI, CDSCO, WPC, AERB, DGFT, Plant Quarantine]
forced_labor: None (no specific legislation)
guarantee_requirement: Bank guarantee for specific scenarios
key_inquiries: [Advance Ruling (Section 28H), DRI Query, Show Cause Notice]
screening: DGFT ITC-HS restrictions, Strategic goods list
```

### Brazil (BR)

```yaml
code: BR
customs_authority: Receita Federal do Brasil
filing_system: SISCOMEX (Sistema Integrado de Comércio Exterior)
declaration_form: DI (Declaração de Importação) → transitioning to DUIMP
importer_id: CNPJ + RADAR (Registro e Rastreamento da Atuação dos Intervenientes Aduaneiros)
entry_types: ["Importação Definitiva", "Admissão Temporária", "Drawback"]
tax_components: [II, IPI, PIS-Importação, COFINS-Importação, ICMS (cascading)]
regulatory_bodies: [ANVISA, MAPA, INMETRO, IBAMA, ANATEL, Exército]
forced_labor: None (no specific legislation)
guarantee_requirement: RADAR habilitação (registration) + bank guarantee for yellow/red channel
key_inquiries: [Exigência Fiscal, Parametrização Channel Assignment]
screening: Lista de impedidos, CNPJ situação cadastral
special: Parametrização channels (Green/Yellow/Red/Grey), NF-e requirement for nacionalização
```

### China (CN)

```yaml
code: CN
customs_authority: GACC (General Administration of Customs of China)
filing_system: Single Window (国际贸易单一窗口)
declaration_form: 报关单 (Customs Declaration Form)
importer_id: USCC (Unified Social Credit Code) + Customs Registration
entry_types: ["一般贸易 (General Trade)", "加工贸易 (Processing Trade)", "保税 (Bonded)"]
tax_components: [Import Duty, Consumption Tax (grossup), VAT (13%)]
regulatory_bodies: [CCC (China Compulsory Certification), CIQ, AQSIQ, MOFCOM]
forced_labor: None
guarantee_requirement: Customs guarantee deposit for specific scenarios
key_inquiries: [Price Verification, Classification Review, Administrative Ruling]
screening: Unreliable Entity List, Export Control Law
```
