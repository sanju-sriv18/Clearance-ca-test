# Broker Expert Review — Clearance Platform Architecture (v4)

**Reviewer:** Licensed Customs Broker / Operations Director (10 years express carrier + freight forwarding)
**Date:** 2026-02-08
**Document Reviewed:** `docs/architecture-clearance-platform.md` (v4)

---

## Executive Summary

This is one of the most thoughtful customs technology architectures I've reviewed. The author clearly understands international trade logistics — the separation of Shipment (commercial abstraction) from Handling Unit (physical reality) and Consolidation (transport grouping) is exactly right and is something most vendors get wrong. The HU→Consolidation nesting model, the movement event cascade, and the broker intelligence flow all reflect real operational patterns.

That said, there are meaningful gaps that would cause problems in production. Some are domain model omissions (missing entry types, missing party roles, missing operational states). Others are regulatory blind spots (Section 321/Type 86 changes, ACAS vs ISF distinction, IEEPA tariff requirements). And some are architectural assumptions that don't match how express carriers actually operate.

I'll be specific throughout. Section and line references point to the architecture document.

---

## 1. Domain Model Accuracy

### 1.1 What's Right

**Shipment as "shipper convenience abstraction" (Section 3.5, line 514):** This is exactly correct. The shipment is NOT a customs entity. Customs cares about physical units and transport documents. Most platforms make the mistake of attaching customs processes to the shipment. This architecture correctly attaches them to HUs and consolidations. Well done.

**The `border_crossing_mode` concept (Section 3.7, lines 802-805):** The insight that a FedEx Air shipment from Shanghai crosses the border by air, but a J.B. Hunt ground shipment from Shanghai crosses by ocean — this is operationally correct and crucial for determining document requirements and filing obligations. Most systems get this wrong by equating "carrier mode" with "border-crossing mode."

**Dual customs clearance fields on HU (line 657-659):** Having separate `export_clearance` and `import_clearance` states is correct. These are independent processes governed by different authorities in different jurisdictions.

**The cage_status model (lines 667-680):** The cage/detention tracking with facility types, storage rates, GO deadlines, and exam types reflects real operations. The distinction between carrier_cage, CFS, CES, border_facility, and bonded_warehouse is important and correct.

### 1.2 What's Wrong or Missing

**Entry type codes are too narrow (line 939).** The doc lists `"01" (consumption), "11" (informal), "06" (FTZ)`. This covers maybe 40% of real entry types. Missing critical types:

| Code | Type | When Used |
|------|------|-----------|
| 02 | Consumption - Quota/Visa | Textiles, certain agricultural products |
| 03 | Consumption - AD/CVD | Antidumping/countervailing duty entries |
| 21 | Warehouse | Goods entering bonded warehouse |
| 22 | Re-warehouse | Transferring between warehouses |
| 23 | TIB (Temporary Importation Bond) | Samples, exhibits, equipment for repair |
| 26 | FTZ Admission (warehouse) | FTZ warehouse admission (different from 06) |
| 31 | Warehouse Withdrawal - Consumption | Withdrawing from warehouse for consumption |
| 46 | Drawback entry | Duty drawback claims |
| 51/52 | Government entries | Defense/dutiable government |
| 86 | Section 321 / De Minimis | Express low-value (NOTE: suspended as of Aug 2025) |

The absence of entry types 02, 03, 21-23, 31, and especially 86 is a significant gap. AD/CVD entries (type 03) are particularly important because they carry separate cash deposit requirements that interact with the bond system differently from ordinary duties.

**Section 321 / De Minimis is completely absent.** This is a critical gap for an express-focused platform. As of August 29, 2025, the Section 321 de minimis exemption has been **suspended for all countries**, and Entry Type 86 is no longer accepted by CBP. Before that suspension, approximately 4 million packages PER DAY entered the US under Section 321. Any platform handling express clearance MUST model this — both the pre-suspension workflow (Type 86 filing) and the post-suspension reality (formal entry required for everything). The architecture makes zero mention of de minimis.

**HU status machine is missing key states (line 640).** The current states are:
```
created | at_origin | picked_up | in_transit | at_customs | inspection | held | cleared | out_for_delivery | delivered
```

Missing states:
- **`in_bond`** — Cargo moving under CBP bond (IT, IE, or T&E) between ports of entry. This is extremely common for ocean cargo arriving at LA/LB destined for inland ports. In-bond movements can take days and have their own tracking requirements.
- **`exam_ordered`** — Between hold and inspection. CBP orders the exam; the broker arranges transport to the CES/exam site; the exam hasn't started yet. This can be hours or days.
- **`general_order`** — After 15 days unclaimed. This is a distinct legal status, not just a deadline. Cargo under GO is transferred to a GO warehouse and the importer's rights change.
- **`seized`** — CBP has seized the merchandise (IPR violations, contraband, UFLPA forced labor determinations). This is a terminal state that triggers a completely different legal process.
- **`refused`** — Importer refused delivery. Triggers return/re-export/destruction options.
- **`abandoned`** — Importer has formally abandoned the goods. Different from GO (which is involuntary).

The `in_bond` omission is the most operationally significant. At major gateway ports (USLAX, USLGB), a huge percentage of cargo moves in-bond to inland destinations. In-bond tracking is a separate CBP workflow with its own timeline requirements.

**Shipment status machine is missing `held` (line 594).** If all HUs in a shipment are held, the shipment aggregate should reflect that. The current machine goes `in_transit → partially_cleared → cleared → delivered` with no path for "everything is held and nothing is moving." The shipper needs to see this.

**The Product model is missing key trade attributes (lines 174-203):**
- **Country of origin vs. country of export:** These are different. A Chinese product warehoused in Vietnam is exported from VN but origin is CN. This distinction is critical for duty determination, AD/CVD, UFLPA, and FTA.
- **Manufacturer identification (MID):** Required on CBP entry summaries. Format is specific: country code + first two characters of city + first three characters of manufacturer name + street address digits.
- **Material composition:** Critical for textile/apparel classification (which chapter 50-63 headings depend entirely on fiber content) and for restricted materials screening.
- **Unit of measure:** CBP requires specific units per tariff heading (dozens, pairs, square meters, kilograms). Getting this wrong causes CF-28s.

---

## 2. Handling Unit / Consolidation Lifecycle

### 2.1 What's Right

**The `consolidated_into` immediate-parent model (line 651-654):** This is correct. An HU only knows its immediate parent consolidation. A consolidation only knows its immediate parent consolidation. Position cascades down via events. This matches real hub operations where you don't track the full nesting chain at every level — you track what container/ULD/manifest your cargo is on RIGHT NOW.

**The movement event cascade (Section 3.7.1, lines 875-916):** This is well designed. The cascade from top-level consolidation → nested consolidation → HU → shipment projection is how real cargo tracking works. The `derived_from` field for tracing cascade origin is a good design choice.

**The `can_deconsolidate` guard (line 762):** Correct. You cannot break down a MAWB/container if any HAWB/HBL inside has an active customs hold. This prevents the nightmare scenario of releasing held cargo by deconsolidating around it. In real operations, a single held HAWB in a MAWB of 200 HAWBs can block the entire MAWB deconsolidation — though in practice, carriers move held cargo to a cage and deconsolidate the rest. The model should support **partial deconsolidation**: deconsolidate everything except held items.

### 2.2 What's Wrong or Missing

**No partial deconsolidation.** The architecture has `deconsolidating → deconsolidated` as binary states. In reality, at a hub like Memphis or Leipzig, when a MAWB arrives with 200 HAWBs and 3 are held, the carrier:
1. Breaks down the ULD/container
2. Sorts all HAWBs
3. Moves 197 to the sort facility for onward routing
4. Moves 3 to the cage
5. The MAWB is "deconsolidated" for 197 HAWBs but the consolidation record can't close until all 3 held ones are resolved

The architecture needs a `partially_deconsolidated` status or a per-HU deconsolidation flag.

**No reconsolidation for onward movement.** At transshipment hubs, cargo is deconsolidated from one MAWB/container and reconsolidated into another for the next leg. A package from Shanghai→CDG→Memphis→Dallas touches THREE different MAWB consolidations. The architecture models `consolidated_into` as a single reference. It should support a history of consolidation relationships, or the ability to deconsolidate from one and consolidate into another cleanly.

**No ground consolidation model for last-mile.** After an air MAWB is deconsolidated at Memphis, HAWBs going to Dallas are loaded onto a ground trailer with a manifest. This is a new consolidation (ground manifest type) that happens after the air consolidation. The architecture supports this through nesting, but the `consolidation_type` list (line 743: `mawb | mbl | manifest | container | uld`) is missing `trailer` and `pallet_build` which are common in express operations.

**ULD types are incomplete (line 768).** Lists `LD3 / LD7 / LD11 / PMC`. Missing: LD1 (regional), LD2 (wide-body lower deck), LD6 (half-width), LD8 (contoured), LD9 (rectangular main deck), PGA (half-pallet), PAG (20ft), PAJ (30ft). Express carriers have very specific ULD management needs — FedEx uses LD3s primarily on 777Fs and A300s, while DHL Leipzig hub uses LD7/LD11 on their 747-400Fs. ULD type determines capacity, sort facility handling, and dwell time calculations.

**Container types are incomplete.** The financial model mentions `"20GP", "40GP", "40HC", "45HC"` (line 1376). Missing: 20RF (reefer), 40RF (reefer), 40OT (open top), 40FR (flat rack), 20TK (tank). Reefer containers have different demurrage/detention rates and additional power charges. Open tops and flat racks are common for oversized/project cargo.

---

## 3. Customs Process Fidelity

### 3.1 What's Right

**Declaration Management as broker's domain, Adjudication as authority's domain (Sections 3.8-3.9):** This separation is correct and important. The broker prepares and submits; the authority receives and decides. They are different systems owned by different parties.

**Dynamic checklist model (lines 951-962):** This is exactly right. The broker's filing checklist varies by jurisdiction, transport mode, commodity, and compliance flags. Making it data-driven rather than hardcoded is the correct approach.

**ISF workflow (lines 926-994):** The ISF model is solid. The 24-hour deadline before vessel loading for ocean, the data elements, amendment tracking, late filing penalty assessment — all correct.

**CF-28/CF-29 workflow (lines 1047-1059):** The back-and-forth between broker and authority for information requests (CF-28) and penalty notices (CF-29) is well modeled. The intelligence service for drafting CF-28 responses from precedents is a genuinely useful feature.

### 3.2 What's Wrong or Missing

**ISF is ocean-only; ACAS is air. The architecture conflates them.** ISF 10+2 applies ONLY to ocean cargo. Air cargo has ACAS (Air Cargo Advance Screening), which is a completely different program with different data elements, different filing parties (carrier files, not importer), and different timelines ("as early as practicable, no later than prior to loading"). The architecture mentions ACAS in the Declaration Management checklist (line 1039: "US entries need ACAS confirmation") but doesn't model it as a separate filing. ACAS is a carrier obligation, not a broker obligation. It belongs in the Consolidation domain or as a carrier interface, not in Declaration Management.

**No entry summary (CBP 7501) vs. entry/release (CBP 3461) distinction.** In US customs, there are two steps:
1. **Entry (CBP 3461)** — Filed to get cargo released. This is the "release me from customs" request. Can be filed up to 15 days before arrival.
2. **Entry Summary (CBP 7501)** — Filed within 10 working days of release. This is the "here's what I owe" declaration with full tariff detail.

The architecture treats these as one thing (`EntryFiling`). They are separate documents with separate timelines, and the entry can be filed and cargo released while the entry summary is still being prepared. This is how pre-clearance actually works: file the entry (3461) to get release, then complete the entry summary (7501) with full duty computation. The `filing_status` machine should model this two-step process.

**No Remote Location Filing (RLF).** CBP allows brokers to file entries at ANY port, regardless of where the cargo physically arrives. A Memphis-based broker can file entries for cargo arriving at LA/LB. The architecture's `port_of_entry` field exists (line 941) but RLF is unmentioned. In express operations, RLF is standard — FedEx's brokerage team in Memphis files for all US ports.

**No ABI/ACS transaction types.** The architecture doesn't model the specific CBP electronic transaction types:
- Entry/Immediate Delivery (01/ID)
- Entry Summary (01/ES)
- Release (01/RL)
- Cargo Release Status Notifications (from CBP to broker)
- Liquidation notices
- Census warnings

These are the actual electronic messages exchanged with CBP via ABI. Without modeling them, the Declaration→Adjudication event flow is an abstraction that won't map to real EDI integration.

**No IEEPA tariff program.** As of 2025-2026, IEEPA (International Emergency Economic Powers Act) tariffs are the single largest tariff program affecting US imports. The TariffCalculation model (line 276) lists `programs_applied: [string] # Section 301, 232, IEEPA, ADCVD, etc.` — IEEPA is mentioned but its unique requirements are not modeled:
- IEEPA has country-specific rates (10% baseline, with 145% for China, 25% for Canada/Mexico, varying reciprocal rates)
- IEEPA tariffs apply ON TOP of regular duties (they stack)
- IEEPA has specific exemptions (USMCA-qualifying goods, certain sectors)
- IEEPA requires specific HTS reporting (additional 99-series HTS codes)
- Transit rules for IEEPA are complex — goods loaded before certain dates may be exempt

**No protest timeline tracking.** The architecture models protests (line 1059, line 1178) but doesn't model the 180-day protest window after liquidation, the 2-year reliquidation window, or the Court of International Trade escalation path. Protests are time-critical — missing the window means the importer permanently loses the right to contest.

---

## 4. Carrier-Broker Interaction

### 4.1 The Fundamental Problem

**The architecture doesn't model the "carrier IS the broker" scenario (Express integrator model).** In express operations (FedEx, UPS, DHL Express):

- FedEx Trade Networks is a licensed customs broker that IS part of FedEx
- The carrier already has all the data (shipper, consignee, weight, value, description) because it's on the air waybill
- There is no "broker claims shipment" step — the carrier's brokerage division automatically files for all shipments unless the importer designates a different broker (self-file or third-party)
- The carrier has a continuous bond covering all their brokered entries
- Pre-clearance happens automatically because the carrier has the data from booking

**The current flow (Section 7, lines 2096-2143):** `ShipmentCreated → intelligence cascade → broker reviews → approve → submit` assumes a human broker deliberately claims and reviews each entry. In express operations:

1. Shipper books with FedEx → data captured at booking
2. FedEx's automated brokerage system files entry automatically during transit
3. 87% of entries go through STP (Simplified Transaction Processing) — no human review
4. Only flagged entries get human broker attention (the 13%)
5. The "broker queue" is a **triage queue for exceptions**, not a workflow for every shipment

**The architecture needs a broker assignment model that supports:**
- **Auto-brokerage (carrier-integrated):** Carrier files automatically; no claim step; no human review for STP-eligible entries
- **Designated broker (third-party):** Importer designates their own broker via Power of Attorney; broker claims from queue
- **Self-file (importer files directly):** Importer has their own ABI access; files without a broker
- **Split brokerage (different broker per port):** Some importers use different brokers at different ports

The Broker entity (lines 996-1007) doesn't capture the carrier affiliation. Add:
```
carrier_affiliation: string?     # "FedEx", "UPS", "DHL" — null for independent brokers
is_integrated_carrier_broker: boolean
auto_file_eligible: boolean      # Can this broker auto-file without human review?
```

### 4.2 Power of Attorney

The architecture is missing the Power of Attorney (POA) model entirely. In US customs, a broker CANNOT file on behalf of an importer without a POA on file. POAs can be:
- **Per-entry** (one-time, rare)
- **Continuous** (standard — "you are authorized to act as my broker for all entries")
- **Revocable** (importer can switch brokers by revoking POA and issuing a new one)

The Document Management domain lists `power_of_attorney` as a document type (line 1612) but there's no model for the POA relationship between importer and broker. This is a gating requirement for filing — no POA, no filing.

---

## 5. Multi-Jurisdiction Reality

### 5.1 What's Right

**Jurisdiction-specific adjudication events (lines 1153-1160):** The architecture correctly models that US (CBP), EU (UCC), BR (Siscomex), IN (ICEGATE), and CN (GACC) have fundamentally different adjudication processes. Brazil's parametrization system (green/yellow/red/grey channels) is correctly identified.

### 5.2 What's Missing Per Jurisdiction

**US — Missing:**
- **ACE portal interactions.** CBP doesn't just accept/reject. There are specific ACE transaction statuses: "conditional release," "intensive exam ordered," "hold for FDA," "hold for CPSC," etc.
- **Reconciliation program.** The architecture mentions reconciliation (line 1446) but doesn't model the CBP Reconciliation program (entry type 09), which allows importers to file entries with estimated values and reconcile later. This is critical for transfer pricing, assists/sells, and complex value determinations.
- **CPSC eFiling.** As of 2025, CPSC requires electronic filing for consumer products. This is separate from PGA flagging.
- **Lacey Act declarations.** For wood/plant products, a separate declaration is required.

**EU — Substantially undermodeled:**
- **No ENS (Entry Summary Declaration) for ICS2.** The EU's Import Control System 2 (ICS2) requires pre-loading advance cargo information (PLACI) for ALL goods entering or transiting the EU. This is the EU equivalent of ISF+ACAS combined. As of February 2026, ICS2 Release 3 is fully operational for all transport modes. The architecture doesn't model ENS filing at all.
- **No T1/T2 transit procedures.** Goods moving through the EU under customs transit use T1 (non-EU goods) or T2 (EU goods) documents. This is how goods move from one EU port to an inland customs office for clearance. The EU equivalent of US in-bond.
- **No EORI requirement.** All parties in an EU customs transaction need an EORI (Economic Operators Registration and Identification) number. The architecture doesn't model this.
- **No AEO status.** Authorized Economic Operator (EU equivalent of C-TPAT) affects clearance speed and exam rates.

**UK — Completely absent.** Post-Brexit, the UK has its own customs system (CHIEF → CDS migration). UK imports from the EU now require full customs declarations. The architecture lists US, EU, UK, IN, BR, CN but doesn't model any UK-specific requirements.

**India — Missing:**
- No IGST (Integrated GST) modeling — India's customs duty is Customs Duty + IGST + Social Welfare Surcharge
- No BoE (Bill of Entry) vs. Shipping Bill distinction
- No ICEGATE electronic filing specifics

**Brazil — Missing:**
- No Siscomex Importacao vs. Siscomex Exportacao
- No NF-e (Nota Fiscal Eletronica) requirement — every goods movement in Brazil requires a fiscal document
- No RADAR registration for importers

**China — Missing:**
- No CIQ (China Inspection and Quarantine) pre-clearance requirements
- No GACC registration for overseas manufacturers (required since 2022 for food/cosmetics)
- No Single Window filing specifics

### 5.3 Recommendation

The `jurisdiction_config: {}?` JSONB field on EntryFiling (line 944) is the right extensibility mechanism, but the architecture should define the **minimum required schema** for each jurisdiction. A flat JSONB blob invites inconsistency.

---

## 6. Physical Operations Gaps

### 6.1 In-Bond Movements

The architecture has NO model for in-bond movements. In US customs, cargo frequently moves under bond from the port of arrival to an inland destination:

- **IT (Immediate Transportation):** Most common. Cargo arrives at USLAX, moves in-bond to Dallas for clearance.
- **IE (Immediate Exportation):** Cargo arrives at a US port and is immediately re-exported without entering US commerce.
- **T&E (Transportation and Exportation):** Cargo enters the US at one port and exits at another.

In-bond movements:
- Require a CBP 7512 or electronic in-bond filing in ACE
- Have specific transit time limits (30 days for IT)
- Must be tracked with arrival notifications at the destination port
- Can be extended with proper notification
- Have their own penalties for late arrival or diversion

For ocean cargo arriving at West Coast ports and clearing at inland locations, in-bond is the standard path. This is a major operational flow that the architecture completely misses. It would belong in the Cargo & Handling Units domain as a status (`in_bond`) with an associated `in_bond_detail` tracking object.

### 6.2 Bonded Warehouse / FTZ Operations

The architecture mentions FTZ as entry type 06 (line 939) and `bonded_warehouse` as a facility type (line 644), but there's no model for:
- **FTZ admission/withdrawal lifecycle.** Goods enter an FTZ under entry type 06/26, can be manipulated/manufactured in the zone, and are withdrawn under entry type 06 (consumption) with duty paid on either the admitted article or the finished product (whichever has a lower duty rate — "inverted tariff" benefit).
- **Bonded warehouse withdrawal schedules.** Goods can stay in a bonded warehouse for up to 5 years before being withdrawn for consumption, exported, or destroyed.
- **Privileged vs. non-privileged foreign status.** FTZ goods can be classified at admission (privileged) or withdrawal (non-privileged), which affects duty rates when tariffs change.

### 6.3 Split Shipments and Partial Deliveries

The Order→Shipment relationship is 1:1 with a note that it should evolve to many-to-many (line 480). But split shipment handling needs more than just a join table:
- Different parts of an order may ship on different dates, via different modes, and clear customs under different entries
- Partial shipment creates partial invoicing, partial duty payment, and partial delivery tracking
- The shipper needs to see "your order is 60% delivered, 40% still in transit"

### 6.4 Returns / RMA / Re-Export

No model for returned goods. In international trade:
- **Drawback on re-export:** Imported goods that are re-exported can get up to 99% duty refund (modeled in DutyDrawback, good)
- **Returned US goods (entry type 01 with 9801/9802):** US-origin goods returning to the US get duty-free treatment under HTSUS 9801/9802. This requires proving US origin — a common pain point.
- **Refused shipments:** Importer refuses delivery. Carrier must hold, return to origin, or destroy. Duty implications depend on whether entry was filed.

---

## 7. Financial Realism

### 7.1 What's Right

**The four-phase financial lifecycle (lines 1462-1488):** `ESTIMATED → PROVISIONAL → LIQUIDATED → RECONCILED` is correct and well-articulated. The distinction between estimated duty (before filing), provisional duty (duty deposit at filing), liquidated duty (CBP's final assessment), and reconciled (post-entry adjustments) reflects the real lifecycle.

**314-day liquidation timeline (line 1479):** Correct for US entries.

**The D&D model (lines 1366-1399):** The distinction between ocean D&D (carrier-specific per-container-type daily rates) and air storage (per-kg rates with tiered pricing) is correct and detailed. The carrier-specific free days model is right — Maersk, MSC, and COSCO do have different free time policies.

**Bond model (lines 1419-1431):** The continuous vs. single entry bond, surety code, utilization tracking, and sufficiency threshold are all correct.

### 7.2 What's Wrong or Missing

**PMS (Periodic Monthly Statement) not modeled.** The `payment_method` field (line 1406) lists `ach | pms | statement | cash_deposit`. PMS is listed but not modeled. PMS is how most large importers pay duties — all entries in a calendar month are consolidated into a single payment due on the 15th working day of the following month. This is a separate CBP program with enrollment requirements, and the payment timeline differs significantly from per-entry payment.

**AD/CVD cash deposits are not separately tracked.** Antidumping and countervailing duties have a separate cash deposit requirement at entry that is distinct from regular duties. The cash deposit rate is set annually by Commerce Department administrative reviews. If the final assessed rate differs from the deposit rate, the importer gets a refund or owes additional duty — potentially years after the original entry. The financial model treats all duties as a single `duty_total`, but AD/CVD cash deposits should be tracked separately because:
- They have different liquidation timelines (tied to Commerce annual review, not CBP standard liquidation)
- They can be bonded differently (AD/CVD bonds are separate from customs bonds)
- The deposit rate changes annually and applies retroactively to unliquidated entries

**Liquidation extensions not modeled.** CBP can extend liquidation up to 3 times, each for 1 year (so up to 4 years total). Entries under AD/CVD are often extended pending Commerce administrative reviews. Importers need to track which entries are extended, when they're expected to liquidate, and what the financial exposure is. The `liquidated_at` field (line 1416) is present but there's no model for extensions, suspensions, or the "deemed liquidated" 4-year statutory deadline.

**Customs bonds — missing the $50K standard.** The bond model has `amount: decimal` with a note about "typically 10% of annual duties" (line 1425). The standard CBP continuous bond is the greater of $50,000 or 10% of duties, taxes, and fees paid in the prior 12 months. For AD/CVD entries, there's a separate single transaction bond requirement at 100% of the antidumping duty margin. These specific thresholds matter for bond sufficiency calculations.

**Harbor Maintenance Fee scope.** HMF (line 1351) is described as "0.125%, ocean only." This is correct — HMF applies only to ocean-arriving cargo and is calculated on the value of the goods. But it's also important that HMF does NOT apply to FTZ withdrawals for export, which is one of the FTZ program's benefits. The financial model should account for FTZ HMF exemptions.

**MPF min/max figures.** The MPF values listed (line 1350: `$31.67 min, $614.35 max`) — note these amounts are adjusted annually by the CBP. They should be configuration values, not hardcoded. The rate of 0.3464% is correct for formal entries; informal entries have a different fee structure ($2, $6, or $9 depending on type).

---

## 8. Express vs. Freight Forwarding

### 8.1 The Architecture Leans Freight Forwarding

The architecture models the freight forwarding workflow well (separate shipper, carrier, broker, and authority roles with handoffs between them) but underserves the express integrator model.

**Express integrator differences the architecture doesn't capture:**

| Aspect | Freight Forwarding | Express Integrator |
|--------|-------------------|-------------------|
| **Broker relationship** | Importer selects broker, broker claims shipment | Carrier's brokerage files automatically |
| **Data availability** | Broker may receive data days before arrival | Carrier has data from booking |
| **Filing timing** | Often filed when cargo at customs | Filed during transit (pre-clearance) |
| **STP rate** | Lower (complex commodity mix) | Higher (~87% for express) |
| **Volume** | Tens of entries per day | Thousands per day per hub |
| **Entry type mix** | Mostly Type 01 (formal) | Mix of 01, 11, and (historically) 86 |
| **Exam handling** | CES/CFS exam at third-party | Carrier's own cage facility at hub |
| **D&D** | Charged by terminal/carrier separately | Integrated into carrier billing |
| **Bond** | Per-importer continuous bond or single entry | Carrier's customs bond covers brokered entries |

The simulation should model BOTH patterns. A ShipperBot→CarrierBot→BrokerBot flow works for freight forwarding. For express, the CarrierBot and BrokerBot should be the SAME actor (or the CarrierBot should have integrated brokerage capability).

### 8.2 Section 321 / De Minimis / Express Clearance

The suspension of Section 321 de minimis (August 2025) is the single biggest change to express clearance in a decade. Before suspension:
- Shipments under $800: duty-free, minimal data filing (Entry Type 86)
- ~4 million packages/day entered under 321
- Express carriers processed these with automated Type 86 filing

After suspension:
- ALL shipments require formal or informal entry regardless of value
- Massive increase in formal entry filings
- Express carriers scrambling to file formal entries for what were previously de minimis
- Significant increase in duty collection and processing burden

The architecture MUST model:
1. The current post-suspension reality (all entries require formal filing)
2. Value-based entry type determination (under $2,500 → informal Type 11; over $2,500 → formal Type 01)
3. The express carrier's automated filing workflow for high-volume low-value shipments
4. The possibility that Section 321 may be reinstated with modifications (the platform should be configurable)

---

## 9. Simulation Realism

### 9.1 What's Good

**The simulation actor → platform API mapping (Section 9.2, lines 2239-2251):** The separation of simulation actors using platform APIs (not internal events) is the right design. The ConsolidatorActor and DemurrageActor as separate actors are good — these reflect distinct operational roles.

**The CustomsAuthorityBot's configurable parameters (line 1140):** STP rates, corridor-based scrutiny multipliers, and configurable exam pass rates are realistic.

### 9.2 What's Missing

**No express clearance bot.** The simulation needs a bot that models the express integrator's automated brokerage workflow — high-volume, low-touch, STP-based. The BrokerBot as described models a human broker deliberating over each entry. Express brokerage processes thousands of entries per hour with automated classification, automated filing, and human review only for exceptions.

**CarrierBot doesn't model carrier billing.** Real carriers don't just move cargo — they bill for it. Carrier billing (freight charges, fuel surcharges, security surcharges) is separate from customs duties but affects the importer's total landed cost. The CarrierBot should generate freight invoices that feed into financial settlement.

**No USDA/APHIS/FDA bot.** PGA reviews are modeled as part of the CustomsAuthorityBot, but in reality, FDA, USDA/APHIS, CPSC, and EPA are SEPARATE agencies with SEPARATE review processes, timelines, and dispositions. An FDA hold and a CBP hold have completely different resolution paths. The simulation should have separate PGA bots (or at minimum, the PGABot mentioned at line 2246 should be fleshed out with per-agency behavior).

**No BuyerBot interaction with customs.** In reality, buyers (importers of record) interact with customs through their broker. The buyer provides commercial documents, answers broker questions, makes payment decisions, and approves filing. The BuyerBot should model this interaction, not just "receive delivery."

**Missing disruption scenarios:** The DisruptionInjector should model:
- Port congestion (Long Beach backup scenarios — add days to vessel berthing)
- Labor actions (port strikes, airline ground crew strikes)
- Vessel rerouting (Suez Canal disruptions → Cape of Good Hope → weeks of delay)
- Government shutdowns (CBP staffing reductions → longer processing times)
- System outages (ACE downtime — which actually happens and forces manual processing)

---

## 10. Critical Gaps — "Would Break in Production"

### 10.1 Showstopper: No In-Bond Model

Cargo arriving at US ports frequently doesn't clear at the port of arrival. It moves in-bond to an inland destination for clearance. Without in-bond modeling, the platform cannot track cargo between arrival and customs presentation at the destination port. The HU status machine jumps from `in_transit` to `at_customs` with no way to represent "arrived at port A, moving under bond to port B."

**Fix:** Add `in_bond` status to HU, with an `in_bond_detail` object tracking: origin port, destination port, bond type (IT/IE/T&E), transit time limit, and arrival notification status.

### 10.2 Showstopper: No Section 321 / De Minimis Handling

Since August 2025, all imports require formal or informal entry. The platform has no model for entry type determination based on value, no informal entry workflow (Type 11), and no model for the post-Section-321 express clearance reality.

**Fix:** Add entry type determination logic, informal entry support, and value-based routing.

### 10.3 Showstopper: No Entry/Release vs. Entry Summary Split

US customs operates on a two-step process: entry for release (3461) and entry summary for duty assessment (7501). The architecture models these as one thing. In practice, cargo can be released (physically moving) while the entry summary is still being prepared. This is fundamental to how pre-clearance works.

**Fix:** Split `EntryFiling` into two coordinated objects with separate status machines, or add explicit status states for the release/summary split.

### 10.4 High Risk: Express Carrier Auto-Brokerage Not Modeled

The dominant use case for express clearance — carrier files automatically, STP processes instantly, no human touch — isn't supported. The architecture assumes a human broker claims, reviews, and approves every entry.

**Fix:** Add auto-brokerage broker assignment model with STP decision engine.

### 10.5 High Risk: EU ICS2/ENS Filing Absent

For any shipment entering or transiting the EU, ICS2 pre-loading advance cargo information is MANDATORY as of 2026. The architecture doesn't model ENS filing, and this would be a legal compliance failure for EU-bound cargo.

**Fix:** Add ENS/ICS2 filing to Declaration Management (or as a separate pre-arrival filing, parallel to ISF for ocean).

### 10.6 Medium Risk: No ACAS Model for Air Cargo

ACAS is mandatory for air cargo into the US. It's a carrier obligation (not broker), with different data elements than ISF. The architecture conflates ACAS with the broker's filing workflow.

**Fix:** Model ACAS as a carrier-side pre-loading filing, separate from the broker's entry filing.

### 10.7 Medium Risk: No Importer-of-Record (IOR) Entity

The architecture has `company_name` on Shipment (line 537) and `importer_id` on Bond (line 1421), but there's no first-class Importer of Record (IOR) entity. The IOR is the legal entity responsible for customs compliance, duty payment, and post-entry obligations. The IOR may differ from the buyer, the consignee, and the shipper. Express carriers often serve as IOR for convenience (DHL Express's "DHL as IOR" service). The IOR relationship to entries, bonds, and POAs needs its own entity.

### 10.8 Low Risk: Exam Fee Numbers

The exam fees listed (line 1352: `VACIS $350, tailgate $425, intensive $850`) are in a reasonable range but vary significantly by CES and port. These should be configuration values per facility, not fixed values. Some CES facilities charge >$1,000 for intensive exams. Also missing: the exam reservation/appointment fee and the drayage cost to transport cargo from the terminal to the CES for exam.

---

## 11. What's Genuinely Good

I want to be clear: despite the gaps above, this architecture gets a lot of foundational things right that most platforms get wrong:

1. **Physical entities as first-class customs objects.** Customs processes attach to HUs and consolidations, not shipments. This is correct.

2. **Event-driven intelligence accumulation.** Intelligence flowing to the broker via events rather than API calls is the right pattern for pre-clearance. The broker doesn't go fetch data — data comes to them.

3. **The readiness model (Section 7.1).** Tracking filing readiness as a score that accumulates from intelligence events is a good design. It gives the broker immediate visual feedback on what's ready and what's blocking.

4. **CQRS with Redis as read cache.** For a customs platform where read patterns (broker checking status, shipper tracking shipment) are very different from write patterns (filing entry, recording movement), CQRS is the right call.

5. **The movement cascade.** This is genuinely well designed and reflects real cargo tracking at hubs.

6. **D&D modeling with carrier-specific policies.** Most platforms treat demurrage/detention as a flat daily rate. Modeling per-carrier policies with different free days, tiered rates, and container-type-specific pricing is operationally accurate.

7. **The cage management subprocess.** The GO deadline tracking, exam type handling, and storage cost accrual model reflect real cage operations at express hubs and CES facilities.

8. **Separation of commercial docs (shipment) from regulatory docs (HU/consolidation).** Commercial invoice and packing list attach to the shipment (shipper's unit). Regulatory filings (entry, ISF, ENS) attach to the physical entity. This is correct.

---

## 12. Prioritized Recommendations

| Priority | Gap | Effort | Impact |
|----------|-----|--------|--------|
| P0 | Add in-bond movement model | Medium | Blocks all inland clearance flows |
| P0 | Add entry/release vs. entry summary split | Medium | Core US filing workflow is wrong without this |
| P0 | Add Section 321/de minimis handling + entry type determination | Medium | Express clearance doesn't work without this |
| P1 | Model express carrier auto-brokerage | Medium | 87% of express entries would fail current workflow |
| P1 | Add EU ICS2/ENS filing | High | Legal compliance requirement for EU cargo |
| P1 | Add ACAS as carrier-side filing (air) | Low | Separate from ISF (ocean) |
| P1 | Add IOR as first-class entity with POA model | Medium | Gating requirement for all broker filing |
| P2 | Expand entry type codes (02, 03, 21-23, 31, 86) | Low | Currently only handles 3 of 15+ entry types |
| P2 | Add partial deconsolidation | Medium | Current model blocks entire MAWB for single hold |
| P2 | Add AD/CVD separate tracking in financial model | Medium | Wrong duty calculations for AD/CVD merchandise |
| P2 | Add PMS payment workflow | Low | How most large importers actually pay duties |
| P3 | Expand ULD/container type lists | Low | Operational accuracy |
| P3 | Add liquidation extension/suspension tracking | Low | Financial risk management |
| P3 | Per-jurisdiction minimum schema for jurisdiction_config | Medium | Prevents inconsistent JSONB blobs |
| P3 | Model reconsolidation for onward movement | Medium | Hub operations accuracy |

---

*Review completed 2026-02-08. Happy to discuss any of these findings in detail.*
