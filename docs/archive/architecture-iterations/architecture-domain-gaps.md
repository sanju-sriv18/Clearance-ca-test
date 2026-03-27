# Architecture Domain Model Gap Analysis

**Purpose:** Identify every gap between the domain entity models in `architecture-clearance-platform.md` and what's actually implemented in the codebase and required by the business.

**Method:** Line-by-line comparison of each architecture domain model against:
- `backend/app/knowledge/models/operational.py` — existing SQLAlchemy models
- `backend/app/simulation/routing.py` — multi-modal routing engine
- `backend/app/simulation/actors/consolidator.py` — consolidation logic
- `backend/app/simulation/actors/cage.py` — cage management
- `backend/app/simulation/reference_data.py` — CarrierService, Corridor, lifecycle events, regulatory data
- `backend/app/simulation/intermediate_events.py` — transit event generation
- `docs/multi-modal-clearance-vision.md` — business requirements
- All 8 engines (e0-e7) and their test suites

**Verdict:** The architecture's structural decisions (11 domains, NATS, CQRS, simulation separation) are sound. The **entity models within each domain** are academically simplified to the point of being incorrect. They would require a complete rearchitecture if implemented as written.

---

## 1. Shipment Lifecycle Domain — CRITICAL GAPS

### 1.1 Transport Mode: Single Field vs Multi-Modal Reality

**Architecture says:**
```
transport_mode: air | ocean | ground
```

**Reality (implemented in routing.py):**

Every international shipment is multi-modal. A "FedEx Air" shipment has ground legs at origin and destination. An "ocean" shipment has truck-to-port, ocean, and rail-to-inland legs. The `transport_mode` field represents the **border-crossing mode** (which determines the customs document hierarchy), NOT how goods physically move.

The routing engine (`routing.py:130-176`) already produces multi-leg routes with 6 different routing strategies:
1. Express integrator air (through hub network with possible EU transit)
2. Non-integrator air freight (direct origin→destination)
3. Ocean (with optional transshipment)
4. Ground direct (NAFTA corridors)
5. Ground on transoceanic corridor (multi-modal with ocean leg)
6. Ocean carrier on ground corridor (defaults to ground)

**What the model needs:**
```
border_crossing_mode: air | ocean | ground    # regulatory mode (determines doc hierarchy)
legs: [RouteLeg]                              # planned multi-modal itinerary
  - origin, destination
  - mode: ground | air | ocean | rail
  - carrier_segment: string
  - regulatory_zone: string
  - clearance_type: export | transit | import | null
  - is_international: boolean
```

**Gap severity:** CRITICAL. Without legs, the system cannot represent how a shipment physically moves, which carriers operate which segments, or where regulatory events occur.

### 1.2 Routing: `current_leg: int` vs Real Position Tracking

**Architecture says:**
```
routing: {legs: [{origin, dest, mode, carrier, hub_type}], current_leg: int}
```

**Reality:** Position is not a leg index. A shipment at `current_leg: 2` tells you nothing about WHERE the cargo physically is. Real position is:
- A **facility** (FedEx CDG Gateway Cage, Long Beach CFS, Laredo Border Facility)
- Or a **vehicle** (flight FX4521, vessel MAERSK SEALAND V.2603)
- Potentially **inside a consolidation** (MAWB 023-12345678, container MAEU1234567)

The cage actor (`cage.py`) already tracks this with `cage_status`:
```json
{
  "facility_type": "carrier_cage",
  "facility_name": "FedEx Memphis Gateway Cage",
  "cage_location": "Bay A, Rack 3",
  "intake_time": "...",
  "dwell_days": 3.2,
  "storage_cost_accrued": 160.00,
  "go_deadline": "..."
}
```

**What the model needs:**
```
current_position: {
  position_type: in_transit | at_facility | at_hub | at_port
  location: string                    # facility name or vehicle ID
  leg_index: int                      # which leg we're on
  consolidation_context: {            # if inside a consolidation
    consolidation_id: UUID
    master_number: string
    container_or_uld: string
  }
}
```

### 1.3 Carrier: Flat String vs Structured CarrierService

**Architecture says:**
```
carrier: string
```

**Reality (reference_data.py:120-155):** The codebase already has `CarrierService`:
```python
@dataclass
class CarrierService:
    carrier: str          # "FedEx", "DHL Express", "Maersk"
    service_type: str     # "International Priority", "Express Worldwide"
    prefix: str           # tracking number prefix
    mode: str             # primary mode: "air", "ocean", "ground"
    is_integrator: bool   # True = owns chain + customs brokerage
    airline_code: str     # 3-digit IATA for MAWB generation
    scac: str             # Standard Carrier Alpha Code
```

The architecture model doesn't distinguish:
- **FedEx (integrator)** vs **Cargolux (non-integrator air freight forwarder)** — fundamentally different operating models
- **DHL Express** vs **DHL Global Forwarding** — different business units, different routing
- **Service level** (International Priority vs Economy) — affects transit time, not carrier identity

**What the model needs:** The Shipment should store a structured carrier reference, or at minimum the carrier_service_id that resolves to the full CarrierService. The current flat `carrier: string` loses the integrator flag, mode, and service type.

### 1.4 Consolidation Hierarchy: Missing Entirely from Shipment Model

**Architecture model has:** `consolidation_id: UUID?` — a bare FK.

**Reality (operational.py:252-311, consolidator.py):** The codebase has a full Consolidation model:
- **Transport mode-specific fields:**
  - Air: `flight_number`, `uld_id`, `uld_type` (LD3/LD7/LD11/PMC)
  - Ocean: `vessel_name`, `voyage_number`, `container_number`, `seal_number`
- **Master numbers:** MAWB (airline prefix + 8 digits), MBL (SCAC + sequence), manifest
- **Consolidation lifecycle:** booked → closed → in_transit → arrived → deconsolidating → deconsolidated
- **Cargo totals:** total_pieces, total_weight_kg aggregated from children
- **Events JSONB:** consolidation-level events (created, closed, etc.)

The Shipment itself carries:
- `house_number`: HAWB / HBL / Pro#
- `consolidation_id`: FK to parent consolidation
- `references`: JSONB map with `master_number`, `flight_number`, `uld_id`, `container_number`, `seal_number`, `vessel`, `voyage`, etc.
- `cargo_detail`: JSONB with `pieces`, `weight_kg`, `handling_units`, `shipping_marks`

**Gap:** The architecture model mentions none of this detail. A Consolidation model is listed in the Shipment Lifecycle domain table list but has no entity definition. The relationship between HAWB→MAWB→ULD→flight (air) or HBL→MBL→container→vessel (ocean) is the **fundamental document hierarchy** of international trade and it's completely absent from the architecture model.

### 1.5 References Map: Missing

**Architecture model has:** Individual fields for `tracking_number`, `hs_code`.

**Reality (operational.py:368-370):** The codebase uses a `references: JSONB` map that acts as a cross-reference index:
```json
{
  "house_number": "FX12345678",
  "master_number": "023-87654321",
  "entry_number": "ABC-1234567-0",
  "booking_reference": "BK-2026-001234",
  "acas_reference": "ACAS-2026-005678",
  "isf_reference": "ISF-2026-000142",
  "flight_number": "FX4521",
  "uld_id": "AKE12345FX",
  "container_number": "MAEU1234567",
  "seal_number": "SL123456",
  "vessel": "MAERSK SEALAND",
  "voyage": "V.2603",
  "paps_reference": "...",
  "regulatory_touchpoints": [...]
}
```

This is critical because: (1) different stakeholders use different reference numbers to identify the same shipment, (2) the references map is used by all event templates for variable interpolation, (3) regulatory touchpoints are stored here.

### 1.6 Activity Timeline: Not Modeled

**Architecture model:** Only mentions `waypoints: [{location, timestamp, event_type}]`.

**Reality:** The codebase has THREE layers of activity tracking:

1. **Mode-specific lifecycle events** (reference_data.py:413-484): 13 air events (hawb_issued → delivered), 13 ocean events (hbl_issued → delivered), 9 ground events (bol_issued → delivered). Each has event_type, description template with variable interpolation, actor, and reference keys.

2. **Intermediate/transit events** (intermediate_events.py): 8 types including ics2_filed, placi_filed, transit_screening, transit_cleared, transit_held, hub_sort, transshipment_arrived, transshipment_departed. Generated from regulatory touchpoints.

3. **Domain activity events** (from consolidator, cage, customs, etc.): consolidated, cage_intake, cage_released, go_warning, plus all CBP/authority response events.

All three are merged into the `events: JSONB` array on Shipment. The architecture model doesn't describe this aggregation at all.

### 1.7 Regulatory Touchpoints: Missing

**Architecture model:** Not mentioned.

**Reality (routing.py:42-49, 484-516):**
```python
@dataclass
class RegulatoryTouchpoint:
    territory: str          # "EU", "US", "SG"
    location: str           # "Paris CDG, FR"
    touchpoint_type: str    # "transit", "import", "export"
    filings: list[str]      # ["ICS2/ENS", "PLACI"]
    risks: list[str]        # ["sanctions_screening", "dual_use_controls"]
    status: str             # "pending", "cleared", "held"
```

Derived from route legs, stored in `references["regulatory_touchpoints"]`. Each territory the shipment enters creates a touchpoint with required filings and risk factors. This is THE core insight of the multi-modal vision: every territory is a clearance gate.

### 1.8 Cage Status: Wrong Domain

**Architecture model:** Puts cage in Exception Management domain as a nested object.

**Reality (cage.py, operational.py:374-376):** Cage status lives directly ON the Shipment as `cage_status: JSONB`:
```json
{
  "facility_type": "carrier_cage",
  "facility_name": "FedEx Memphis Gateway Cage",
  "cage_location": "Bay A, Rack 3",
  "intake_time": "...",
  "dwell_days": 3.2,
  "storage_cost_accrued": 160.00,
  "storage_rate_per_day": 50.00,
  "go_deadline": "...",
  "go_days_remaining": 11.8,
  "exam_type": "vacis",
  "exam_description": "VACIS/NII — non-intrusive x-ray scan",
  "exam_scheduled": "...",
  "exam_duration_hours": 6.5
}
```

In the NATS architecture, Exception Management can own the lifecycle, but the DENORMALIZED shipment view (Redis) must include this data directly. The architecture's cage model in Exception is too abstract — it doesn't include facility_type, storage rate, exam details, or the physical location within the facility.

---

## 2. Shipment Lifecycle Domain — Additional Entity Gaps

### 2.1 Consolidation Entity: Not Defined

The architecture lists Consolidation in the domain summary but provides NO entity model. The real model (operational.py:252-311) has 14+ fields including mode-specific ones (air: flight/ULD, ocean: vessel/container/seal).

### 2.2 Status Machine: Oversimplified

**Architecture says:**
```
booked → in_transit → at_customs → cleared → delivered
```

**Reality (state_machine.py, referenced in tests):** The codebase has jurisdiction-specific state machines with additional states: `inspection`, `held`, and jurisdiction-specific sub-states (EU: `under_control`, `customs_released`; BR: parameterization channels; IN: `duty_assessed`, `out_of_charge`). Plus consolidation has its own 5-state machine.

---

## 3. Trade Intelligence Domain — Gaps

### 3.1 Classification Model: Incomplete

**Architecture model** is reasonable but misses:
- `hs_codes: JSONB` — multi-jurisdiction classification (the Product model stores per-jurisdiction codes)
- Classification version tracking tied to product version
- The relationship between product `hs_code` (shipper-declared) vs classification `hs_code` (system-determined)

### 3.2 Tariff Model: Missing Jurisdiction Regime Detail

The architecture's `TariffCalculation` is adequate for the US but doesn't capture the full multi-jurisdiction regime system. The codebase has 6 jurisdiction-specific regime engines (US, EU, CN, BR, IN, UK) each with different program structures. The tariff model should carry jurisdiction-specific metadata.

### 3.3 Missing: Landed Cost as a Composite

The architecture mentions "Landed Cost" as an engine output but doesn't model it. Landed cost = duty + tax + fees + freight + insurance + D&D. It's a composite that spans Trade Intelligence AND Financial Settlement.

---

## 4. Order Management Domain — Gaps

### 4.1 Order → Shipment: 1:1 vs Many-to-Many

**Architecture says:** `Order { ... }` with implicit 1:1 to shipment.

**Reality:** The codebase currently has bidirectional 1:1 (`Order.shipment_id` FK + `Shipment.order_id` FK with unique=True). But the business reality (from vision doc and real trade) is many-to-many:
- One order can spawn multiple shipments (partial shipments, split shipments)
- One consolidation groups multiple shipments from potentially different orders
- Line items from different orders can be in the same container

The architecture should model the join table (`order_shipment_lines`) that maps order_line_items → shipment_id, even if current implementation is 1:1.

### 4.2 Order Line Items: Missing Compliance State

**Architecture says:** `line_items: [{product_id, quantity, unit_price, line_total, source_location}]`

**Reality (operational.py:208-245):** OrderLineItem has:
- `hs_code`: per-line HS classification
- `duty_amount`: per-line duty
- `compliance_status`: per-line compliance result
- `tariff_breakdown: JSONB`: per-line tariff detail
- `source_country` + `source_facility`: separate origin tracking per line

The architecture model treats line items as simple quantity × price records, missing that each line item is independently classified, tariffed, and compliance-screened.

---

## 5. Declaration Management Domain — Gaps

### 5.1 EntryFiling: Missing Jurisdiction Config

**Architecture model:** Defines `Declaration` without jurisdiction-specific configuration.

**Reality (operational.py:552-614):** `EntryFiling` has:
- `jurisdiction: str` — "US", "EU", "IN", "BR", "CN"
- `jurisdiction_config: JSONB` — jurisdiction-specific filing configuration
- `checklist_state: JSONB` — dynamic checklist that varies by jurisdiction
- `authority_response: JSONB` — CBP/customs authority response data
- `summary_data: JSONB` — filing summary
- `broker_approval: JSONB` — broker sign-off data

### 5.2 Broker Messages: Missing from Architecture

The architecture mentions communications but doesn't model `BrokerMessage` (operational.py:617-671):
- `direction`: inbound/outbound
- `channel`: email/phone/portal/edi
- `party_type`: shipper/carrier/cbp/importer
- `thread_id`: conversation threading
- `attachments: JSONB`
- `read_at`: read tracking

---

## 6. Compliance & Screening Domain — Gaps

### 6.1 ComplianceResult: Stored on Shipment, Not Separately

**Architecture says:** `ComplianceResult` is a standalone entity in `compliance_screening.compliance_results`.

**Reality:** Compliance results are currently stored as `shipment.analysis: JSONB` (among other analysis results). In the new architecture, they should indeed be separate entities — but the architecture model for ComplianceResult misses the per-line-item compliance that exists on `OrderLineItem.compliance_status`.

### 6.2 Missing: Pre-Clearance Screening Results

The E0 pre-clearance engine produces a detailed screening result with multiple checks (entity screening, HS validation, value reasonableness, document requirements, origin risk). This multi-check result isn't modeled in the architecture.

---

## 7. Exception Management Domain — Gaps

### 7.1 Cage Model: Missing Operational Detail

**Architecture says:**
```
cage_status: {
  facility, intake_at, exam_type, exam_scheduled_at,
  go_deadline, dwell_days, storage_cost, released_at
}
```

**Reality (cage.py):** Missing from architecture:
- `facility_type`: carrier_cage / cfs / ces / border_facility / bonded_warehouse
- `cage_location`: physical slot ("Bay A, Rack 3")
- `storage_rate_per_day`: rate varies by facility type ($40-$100/day)
- `exam_description`: human-readable exam type description
- `exam_duration_hours`: expected duration
- `_last_go_warning`: threshold tracking for warning dedup
- GO warning events at specific thresholds (10, 5, 3, 1 days)

### 7.2 Transit Holds: New Exception Type

The architecture lists `transit_hold` as a hold_type but doesn't model the territory-specific nature:
- EU sanctions hold requires EU customs engagement
- EU dual-use export control requires different documentation
- Singapore/Korea port authority inspections have different procedures
- Each territory hold has different resolution paths, contacts, and timelines

---

## 8. Financial Settlement Domain — Gaps

### 8.1 Demurrage/Detention: Missing Carrier-Specific Policies

**Architecture model** has a generic demurrage structure. **Reality (reference_data.py:1117-1169):** Each carrier has specific D&D policies:
- Free days vary by carrier (Maersk: 4 import / COSCO: 7 import)
- Rates vary by container size (20GP, 40GP, 40HC, 45HC)
- Combined vs separate free time rules
- Air cargo has per-kg storage rates with minimum daily charges

### 8.2 Missing: Fee Reference Data

The architecture doesn't model the fee reference data that drives calculations:
- MPF ($29.66 min, $614.35 max, 0.3464%)
- HMF (0.125%, ocean only)
- Broker fee tiers (simple/standard/complex/critical: $150-$650)
- Exam fees (VACIS $350, tailgate $425, intensive $850)
- Bond structures (continuous 0.5% annual duty, single entry 1% shipment value)

---

## 9. Regulatory Intelligence Domain — Minor Gaps

### 9.1 RegulatorySignal: Mostly Correct

The architecture model is close to the implementation. Minor gaps:
- Missing `rate_change: JSONB` — specific rate change data for tariff scenario modeling
- Missing `source_hash: str` — deduplication hash

---

## 10. Product Catalog Domain — Gaps

### 10.1 Product: Missing Multi-Jurisdiction HS Codes

**Architecture says:** Single `country_of_origin`.

**Reality (operational.py:129-155):**
- `hs_codes: JSONB` — per-jurisdiction HS codes (same product classified differently in US, EU, CN)
- `source_locations: JSONB` — multiple sourcing locations (not just one country)
- `cached_analysis: JSONB` — cached analysis results for performance

---

## 11. Document Management Domain — Gaps

### 11.1 Document: Missing Validation State

**Architecture model** includes `validation_result` but the codebase stores documents with `content_base64` (actual PDF content). The architecture model should clarify that documents can be:
- Required but not yet uploaded (from Document Engine's requirements determination)
- Uploaded but not validated
- Validated with discrepancies
- Validated clean

### 11.2 Missing: Mode-Specific Document Requirements

The codebase has `MODE_DOCUMENTS` (reference_data.py:493-499) defining required documents per transport mode:
- Air: HAWB, MAWB, Commercial Invoice, Packing List, ACAS Filing, Entry Summary (7501)
- Ocean: HBL, MBL, Commercial Invoice, Packing List, ISF 10+2, Arrival Notice, Entry Summary
- Ground: BOL, Commercial Invoice, Packing List, PAPS/PARS, USMCA Certificate, Entry Summary

---

## 12. Cross-Cutting: State-Transfer Event Subscriptions

### 12.1 Missing: What Each Domain Stores Locally

The architecture lists event subscriptions per domain but doesn't specify what **local data structures** each domain creates from those events. For state-transfer to work correctly:

| Domain | Subscribes To | Must Store Locally |
|--------|--------------|-------------------|
| Trade Intelligence | ProductCreated/Updated | Local copy of product description, origin, material |
| Compliance | ShipmentCreated, ClassificationProduced | Local copy of shipment origin, HS code, company name, carrier |
| Declaration Management | ShipmentCreated, ClassificationProduced, TariffCalculated, ComplianceScreened | Local copies of ALL intelligence results to populate checklist |
| Financial Settlement | TariffCalculated, ShipmentCreated/Cleared/Delivered, CageIntake/Released | Local copy of tariff amounts, shipment timeline, cage costs |
| Exception Management | AdjudicationDecision, ComplianceScreened(HOLD), ShipmentArrived | Local copy of hold details, shipment position, declaration status |

---

## 13. Redis Cache: Denormalized Shipment View Incomplete

### 13.1 Architecture's Denormalized View Missing Critical Data

**Architecture shows:**
```json
{
  "classification": { "hs_code", "confidence" },
  "tariff": { "total_duty", "effective_rate", "programs" },
  "compliance": { "overall_status", "pga_flags" },
  "declaration": { "filing_status", "broker" },
  "financial": { "total_exposure", "demurrage_accruing" },
  "exceptions": [],
  "documents": { "required", "uploaded", "validated" }
}
```

**What's missing from the Redis view:**
```json
{
  "routing": {
    "border_crossing_mode": "air",
    "legs": [...],                        // Full leg sequence
    "current_leg_index": 2,
    "regulatory_touchpoints": [...]       // With status per territory
  },
  "consolidation": {
    "consolidation_id": "...",
    "master_number": "023-87654321",
    "consolidation_type": "mawb",
    "flight_number": "FX4521",            // or vessel, container for ocean
    "status": "in_transit"
  },
  "carrier_service": {
    "carrier": "FedEx",
    "service_type": "International Priority",
    "is_integrator": true,
    "mode": "air"
  },
  "references": {                          // All cross-reference IDs
    "house_number": "FX12345678",
    "master_number": "023-87654321",
    "entry_number": "ABC-1234567-0",
    ...
  },
  "cargo": {
    "pieces": 4,
    "weight_kg": 120.5,
    "handling_units": [{"type": "pallet", "count": 2}]
  },
  "cage_status": {                         // If held/inspection
    "facility_name": "FedEx Memphis Gateway Cage",
    "cage_location": "Bay A, Rack 3",
    "dwell_days": 3.2,
    "storage_cost_accrued": 160.00,
    "go_deadline": "...",
    "go_days_remaining": 11.8,
    "exam_type": "vacis"
  },
  "activity_timeline": [                   // Merged from ALL sources
    // Mode-specific lifecycle events
    // Intermediate/transit events
    // Domain events (consolidation, cage, customs)
  ],
  "current_position": {
    "position_type": "at_facility",
    "location": "FedEx Memphis Gateway Cage",
    "consolidation_context": { ... }
  }
}
```

Without this, rendering the shipment detail screen requires calling 7+ domain services — defeating the entire purpose of CQRS.

---

## Summary: Gap Count by Domain

| Domain | Critical Gaps | Moderate Gaps | Minor Gaps |
|--------|:------------:|:-------------:|:----------:|
| **Shipment Lifecycle** | 8 | 3 | 1 |
| **Order Management** | 2 | 1 | 0 |
| **Declaration Management** | 1 | 2 | 0 |
| **Trade Intelligence** | 0 | 3 | 1 |
| **Compliance & Screening** | 1 | 1 | 0 |
| **Exception Management** | 2 | 1 | 0 |
| **Financial Settlement** | 1 | 2 | 0 |
| **Regulatory Intelligence** | 0 | 0 | 2 |
| **Product Catalog** | 0 | 1 | 1 |
| **Document Management** | 0 | 2 | 0 |
| **Customs Adjudication** | 0 | 0 | 0 |
| **Redis Cache** | 1 | 0 | 0 |
| **Cross-cutting (state-transfer)** | 1 | 0 | 0 |
| **TOTAL** | **17** | **16** | **5** |

The Shipment Lifecycle domain accounts for nearly half of all critical gaps — which makes sense, as it's the core domain of the platform.
