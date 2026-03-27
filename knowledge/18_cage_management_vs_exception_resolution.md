# Cage Management vs. Exception Resolution: Gap Analysis

> **Clearance Intelligence Engine -- Domain Analysis**
> Last Updated: February 2026

---

## Executive Summary

A clearance expert repeatedly referenced "cage management" as a missing capability. This document defines cage management in detail, compares it to our existing Exception Resolution feature, identifies where they overlap and where they diverge, and recommends how to address any gaps.

**Bottom line:** Cage management and exception resolution overlap significantly in *intent* (resolve holds and get goods moving), but cage management encompasses a layer of **physical logistics coordination, facility-level inventory tracking, and cost accumulation** that our current exception resolution feature does not address. Our feature handles the *regulatory/compliance resolution* side well but is largely blind to the *physical cargo management* side.

---

## 1. What Is Cage Management?

### 1.1 Definition

"Cage management" refers to the end-to-end operational process of managing **physical cargo that is held under customs bond** -- from the moment a shipment is placed on hold through its physical handling, examination, and eventual release or disposition.

The term "cage" comes from the **literal, physical caged/fenced areas** within carrier facilities, Container Freight Stations (CFS), Centralized Examination Stations (CES), airline cargo terminals, and bonded warehouses where held cargo is stored under restricted access. These are wire-mesh or fenced enclosures within larger warehouse facilities, designed to segregate customs-held goods from released/general cargo.

### 1.2 Where Cages Exist

| Facility Type | Context | Who Operates |
|---|---|---|
| **Carrier facility cage** | Express carriers (FedEx, UPS, DHL) have caged areas at gateway hubs (Memphis, Indianapolis, Newark, LAX, Miami, JFK) for held parcels/freight | Carrier |
| **CFS (Container Freight Station)** | Bonded warehouse at port where LCL shipments are deconsolidated; has caged/bonded sections for held cargo | Private CFS operator |
| **CES (Centralized Examination Station)** | Private facility authorized by CBP specifically for physical cargo examination; entire facility is essentially a "cage" | Private CES operator (e.g., Global CFS, STG Logistics) |
| **Airline cargo terminal** | Secure partitioned areas within air cargo warehouses where held freight is stored by airline/handler | Airline or ground handler |
| **Bonded warehouse (Class 1-5)** | Government or privately owned bonded storage; Class 1 = government-owned for seized/examined goods | Warehouse proprietor |
| **General Order (GO) warehouse** | Bonded warehouse for cargo unclaimed/uncleared after 15 days; last stop before auction/destruction at 6 months | Private GO warehouse operator |

### 1.3 The Full Cage Management Lifecycle

```
HOLD ISSUED ──► CAGE INTAKE ──► CAGE STORAGE ──► ACTION/EXAM ──► RELEASE/DISPOSITION
    │                │               │                │                │
    │                │               │                │                ├─ Released → back to carrier flow
    │                │               │                │                ├─ Transferred → CES for exam
    │                │               │                │                ├─ GO → general order warehouse (day 15+)
    │                │               │                │                └─ Seized → CBP takes custody
    │                │               │                │
    │                │               │                ├─ Tailgate exam at carrier facility
    │                │               │                ├─ Intensive exam at CES (drayage required)
    │                │               │                ├─ VACIS/NII x-ray scan
    │                │               │                ├─ PGA lab sampling
    │                │               │                └─ Document review (no physical action)
    │                │               │
    │                │               ├─ Inventory tracking (location, dwell time, status)
    │                │               ├─ Storage cost accrual ($50+/day after free time)
    │                │               ├─ Demurrage/detention clock running
    │                │               └─ Cage capacity monitoring
    │                │
    │                ├─ Physical move to cage area
    │                ├─ Scan/log into cage inventory system
    │                ├─ Assign cage location (rack, bin, floor)
    │                └─ Chain of custody documentation
    │
    ├─ CBP hold (selectivity, targeting, manifest discrepancy)
    ├─ PGA hold (FDA, USDA, EPA, CPSC, NHTSA)
    ├─ Exam order (tailgate, intensive, VACIS, CET)
    ├─ UFLPA detention
    ├─ DPS/sanctions flag
    └─ Classification/valuation dispute
```

### 1.4 Key Cage Management Activities

**Physical Operations:**
- Receiving held cargo into the cage (intake logging, location assignment)
- Staging cargo for CBP/PGA examination (pulling from cage, presenting at exam area)
- Restaging after exam (repacking, returning to cage or releasing to carrier flow)
- Transferring to CES (coordinating bonded drayage when exam requires offsite facility)
- Transferring to GO warehouse (mandatory at day 15 if no entry filed)

**Inventory & Tracking:**
- Real-time cage inventory (what's in the cage, where, how long)
- Dwell time tracking per shipment (critical metric -- drives costs and GO risk)
- Cage capacity utilization monitoring (typically designed for 2-5% of daily volume)
- Status tracking per shipment (awaiting exam, exam scheduled, exam in progress, awaiting results, pending release)
- General Order countdown tracking (15-day clock from arrival)

**Financial Tracking:**
- Storage fee accrual (typically free for 2 days post-exam, then ~$50/day)
- Demurrage charges (carrier container/equipment charges for extended holds)
- Drayage costs (bonded trucking to/from CES, $300-2,500+ depending on exam type)
- Exam fees (VACIS ~$300, tailgate ~$500-1,000, intensive ~$1,000-2,500+)
- All costs billed to importer -- broker must track and pass through

**Coordination:**
- Scheduling exams with CBP (FIFO queue; C-TPAT gets priority)
- Coordinating bonded drayage carriers for CES transfers
- Communicating cage status to broker/importer/consignee
- Managing CBP Form 6043 (delivery tickets for exam transfers)
- Processing release notifications and restaging for delivery

### 1.5 Key Metrics

| Metric | Typical Value | Why It Matters |
|---|---|---|
| Cage utilization at major gateways | 80-120% (frequently over capacity) | Overflow requires costly off-site bonded storage |
| Routine hold dwell time | 1-3 days (historically); 5-10 days post-de minimis changes | Direct cost and delivery delay |
| PGA hold dwell time | 5-15 days | FDA/USDA/CPSC review cycles |
| UFLPA detention dwell time | 30-90+ days | Evidence compilation window is 30 days |
| Exam hold dwell time | 3-7 days | Scheduling + physical handling + results |
| CBP legal hold limit | 35 days from arrival | Must release or seize after 35 days |
| GO transfer deadline | 15 days from arrival | Automatic by law if no entry filed |
| GO auction deadline | 6 months from GO transfer | Importer forfeits ownership |
| Physical exam rate | < 1% of entries | Low rate but high cost per occurrence |
| CBP hold rate (express) | 3-8% | Volume makes even small rates material |
| PGA hold rate | 2-5% | Often data-driven, not product-driven |

---

## 2. What Our Exception Resolution Feature Covers

Our Exception Resolution feature (documented in detail in the codebase) provides:

### 2.1 Exception Detection & Triage
- Surfaces entries/shipments with statuses: DETAINED, FLAGGED, HOLD, EXCEPTION
- Sources from both ClearanceEntry pipeline and Shipment database (PostgreSQL + Redis)
- Sorts by severity: critical (DETAINED/FLAGGED) > high (HOLD) > medium/low (EXCEPTION)
- Hold types tracked: `uflpa`, `entity_screening`, `cf28_cf29`, `pga`, `inspection`, `inspection_failure`

### 2.2 Exception Context & Intelligence
- Detailed view per exception: product, HS code, origin/destination, carrier, compliance flags, declared value
- Route information including customs port and estimated delivery
- Document status: submitted vs. missing/required documents
- Issue analysis with type-specific explanation
- Recommended resolution steps (type-specific, AI-generated)
- Financial impact display (declared value, estimated duty)

### 2.3 Resolution Tools
- **Type-specific action buttons:** UFLPA (traceability docs, supply chain audit, exemption request), PGA (agency certification, compliance checklist), Classification (CROSS ruling search, protest filing, binding ruling), DPS (enhanced due diligence, OFAC license application)
- **E5 Exception Engine:** CROSS ruling search via vector store, CBP response letter drafting with validated citations
- **AI Operator Assistant integration:** Pre-filled prompts for each exception type
- **Shipper-facing Resolution Center:** Product lookup, document upload/analysis, stuck shipment resolution steps

### 2.4 Backend Resolution APIs
- `/api/resolution/lookup` — HS classification + compliance readiness assessment
- `/api/resolution/upload` — Document extraction and issue identification
- `/api/resolution/suggest` — AI-generated resolution steps per shipment

---

## 3. Gap Analysis: What Cage Management Adds Beyond Exception Resolution

### 3.1 Overlap (What We Already Cover)

| Cage Management Aspect | Our Coverage | Notes |
|---|---|---|
| **Hold detection** | Strong | We detect holds from multiple sources and categorize by type |
| **Hold type classification** | Strong | UFLPA, DPS, PGA, classification, valuation, CF-28/29 |
| **Severity triage** | Strong | DETAINED > FLAGGED > HOLD > EXCEPTION priority system |
| **Regulatory resolution guidance** | Strong | Type-specific recommended actions, CROSS ruling search, response drafting |
| **Document management** | Moderate | Shows submitted/missing docs, document upload + extraction |
| **AI-assisted resolution** | Strong | LLM-powered analysis, resolution steps, response drafting |
| **Shipper communication** | Moderate | Shipper-facing Resolution Center with self-service tools |

### 3.2 Gaps (What Cage Management Requires That We Don't Have)

| Cage Management Aspect | Our Coverage | Gap Severity | Description |
|---|---|---|---|
| **Physical location tracking** | None | Medium | No concept of *where* the cargo physically is (cage, CES, GO warehouse, in transit between facilities). Brokers need to know: "Is this still at the carrier's cage at LAX, or has it been transferred to the CES?" |
| **Dwell time tracking** | Minimal | High | We show `holdImpactDays` but don't track actual cage intake timestamp, cumulative dwell time, or countdown to GO transfer (15-day clock) or CBP's 35-day legal limit. |
| **Cage capacity / utilization** | None | Low | Less relevant for a broker-focused tool (carriers manage cage capacity), but awareness of cage congestion at a gateway could be useful context. |
| **Cost accrual tracking** | None | High | No tracking of accumulating storage fees, demurrage, detention, drayage costs, or exam fees. Brokers need to report total hold cost to importers and ensure fees are passed through on invoices. |
| **Exam scheduling & coordination** | None | High | No workflow for: exam order received → schedule with CES → coordinate bonded drayage → present cargo → track exam status → process results. This is a multi-party coordination workflow that brokers manage daily. |
| **CES/facility management** | None | Medium | No concept of CES facilities, their locations, which exams they handle, scheduling availability, or contact information. Brokers need this for coordination. |
| **Bonded drayage coordination** | None | Medium | No tracking of the physical transport of held cargo between facilities (carrier → CES, CES → carrier, carrier → GO warehouse). Requires bonded carrier, CBP Form 6043. |
| **General Order countdown** | None | High | No tracking of the 15-day clock. If cargo hits 15 days without entry, it's automatically transferred to a GO warehouse -- significantly increasing costs and complexity. This is a critical deadline brokers must track. |
| **CBP Form 6043 / delivery tickets** | None | Medium | The exam transfer process requires signed delivery tickets. No document management for this workflow. |
| **Exam type workflow differences** | Minimal | Medium | We show "inspection" as a hold type, but don't distinguish between tailgate (at carrier facility, 1-2 days), intensive (CES transfer, 5-7 days, full devanning), and VACIS (x-ray, often same-day). Each has different operational requirements. |
| **Multi-party status coordination** | Minimal | High | Cage management requires coordinating status across CBP (ACE), PGA systems, carrier systems, CES systems, and the broker's own tracking. No integration or unified status view across these parties. |
| **Release processing workflow** | Minimal | Medium | After CBP releases, there's a physical restaging process. CES notifies broker via email, broker coordinates pickup, cargo re-enters carrier flow. We show release status but don't manage the post-release logistics. |
| **Financial impact / cost reporting** | Minimal | High | We show declared value and estimated duty, but not the *exception-related costs* (storage, demurrage, drayage, exam fees) that directly result from the hold. Importers need this for cost visibility and invoice reconciliation. |
| **FIFO queue position** | None | Low | CES exams run FIFO (C-TPAT gets priority). Knowing approximate queue position would help set expectations. |

### 3.3 Summary: The Two Halves of Exception Management

Our Exception Resolution feature handles **the regulatory/compliance half**:
- *What* is the hold about? → Hold type classification
- *Why* was it triggered? → Compliance flag analysis
- *How* do we resolve it? → Resolution guidance, CROSS rulings, response drafting
- *What documents* are needed? → Document management and extraction

Cage management adds **the physical/logistics/financial half**:
- *Where* is the cargo? → Physical location tracking
- *How long* has it been held? → Dwell time with deadline countdowns
- *How much* is it costing? → Cost accrual (storage + demurrage + drayage + exam fees)
- *What's the exam workflow?* → Scheduling, drayage, staging, results tracking
- *Who needs to do what next?* → Multi-party coordination (CBP, CES, carrier, drayage, broker)

---

## 4. Express Consignment vs. Formal Entry Context

The clearance expert's emphasis on cage management makes additional sense when you consider the operational context:

### 4.1 Express Consignment (FedEx, UPS, DHL)

In express operations, cage management is **primarily a carrier responsibility**:
- The carrier operates the gateway facility and cage
- The carrier's own operations team handles physical cargo management
- The broker (often the carrier's in-house brokerage) focuses on regulatory resolution
- Systems like FedEx's internal clearance platform handle cage tracking natively

Our broker operations knowledge base notes: *"Hold resolution: Carrier-managed cage operations"* for express consignment.

### 4.2 Formal Entry (Commercial Brokerage)

In formal entry operations, cage management is **primarily the broker's responsibility** in coordination with carriers and facilities:
- The broker must coordinate with multiple parties (carrier, CES, drayage company, CBP, PGA)
- The broker must track physical location and status across facilities they don't operate
- The broker must manage the financial pass-through (tracking costs to invoice the importer)
- There is no single system that provides end-to-end visibility

Our broker operations knowledge base notes: *"Hold resolution: Broker-managed, often with client collaboration"* for formal entry.

### 4.3 Implication for Our Product

If our product targets **independent customs brokers handling formal entries**, cage management becomes a significant operational workflow that our Exception Resolution feature does not currently support. These brokers are managing cage operations manually via spreadsheets, emails, phone calls, and disparate systems -- exactly the kind of fragmented workflow our platform could unify.

---

## 5. Recommendations

### 5.1 High-Priority Additions (Would Meaningfully Address the "Cage Management" Gap)

**1. Dwell Time Tracking with Deadline Countdowns**
- Track cage intake timestamp per held shipment
- Display running dwell time prominently in the exception queue
- Show countdown to critical deadlines:
  - **15-day General Order deadline** (with escalating urgency at day 10, 12, 14)
  - **35-day CBP legal hold limit**
  - **30-day UFLPA evidence submission window**
  - **Carrier free-time expiration** (typically 2-3 days)

**2. Exception Cost Tracker**
- Track and display accumulating costs per held shipment:
  - Storage fees (configurable rate, typically ~$50/day after free time)
  - Demurrage/detention charges
  - Estimated exam fees by type (VACIS ~$300, tailgate ~$500-1,000, intensive ~$1,000-2,500)
  - Drayage costs for CES transfers
- Running total visible in exception detail and summary views
- Exportable for importer invoicing

**3. Exam Workflow Management**
- Track exam type (document review, tailgate, intensive, VACIS/NII)
- Status progression: `exam_ordered` → `scheduling` → `drayage_arranged` → `staged_for_exam` → `exam_in_progress` → `awaiting_results` → `results_received` → `released` or `continued_hold`
- Different workflows per exam type (tailgate = at carrier facility; intensive = CES transfer required)
- Integration points for scheduling notes, CES contact info, and drayage carrier details

**4. Multi-Party Coordination View**
- Show who needs to act next (broker, importer, CBP, PGA, CES, carrier)
- Action items per party with status tracking
- Communication log (when was importer notified? when did CES confirm scheduling?)

### 5.2 Medium-Priority Additions

**5. Physical Location Tracking**
- Track cargo location: carrier facility → cage → CES → GO warehouse
- Simple status-based (not GPS) -- broker updates location as events occur
- Helpful for answering the #1 importer question: "Where is my cargo?"

**6. Exam Type Differentiation**
- Distinguish between exam types in the UI with different workflows and expected timelines
- Tailgate: 1-2 days, at carrier facility, minimal cost
- Intensive: 5-7 days, CES transfer, $1,000-2,500+ in fees
- VACIS: Often same-day, ~$300, minimal handling
- Document review: Remote resolution, no physical handling needed

**7. General Order Management**
- Dedicated GO tracking for shipments approaching or past 15 days
- GO warehouse transfer coordination
- 6-month auction/destruction countdown
- Recovery workflow (correct paperwork → pay fees → coordinate release)

### 5.3 Lower-Priority / Future Considerations

**8. CES/Facility Directory**
- Database of CES facilities by port with contact info, exam types handled, hours, and typical queue times

**9. Cage Capacity Awareness**
- Gateway-level cage congestion indicators (could be sourced from carrier APIs or manual input)

**10. Financial Reporting**
- Exception cost reports by importer, commodity, trade lane, hold type
- Trend analysis (are hold rates/costs increasing for a particular client?)

---

## 6. Conclusion

The clearance expert was correct to flag cage management as a gap. While our Exception Resolution feature provides strong **regulatory resolution intelligence** (what's wrong, how to fix it, what documents are needed, AI-assisted response drafting), it does not address the **physical logistics coordination and financial tracking** that constitute the other half of managing held cargo.

The most impactful additions would be:
1. **Dwell time tracking with deadline countdowns** -- prevents GO transfers and sets proper expectations
2. **Exception cost accumulation** -- critical for importer communication and invoicing
3. **Exam workflow management** -- the multi-step, multi-party coordination that brokers spend significant time managing
4. **Multi-party coordination view** -- clarifying who needs to act next

These additions would transform our Exception Resolution feature from a "compliance resolution assistant" into a "complete hold management platform" that covers both the regulatory and physical dimensions of cage management -- which is what the expert was likely looking for.

---

## Sources

- [Flexport: CES (Centralized Examination Station)](https://www.flexport.com/glossary/centralized-examination-station/)
- [DHL: CES Definition](https://lot.dhl.com/glossary/centralized-examination-station-ces/)
- [Shapiro: CES Resources](https://www.shapiro.com/resources/centralized-examination-station-ces/)
- [On Time Delivery: Cleveland CES Experts on Customs Exams & Holds](https://otdw.net/2022/08/17/cleveland-centralized-examination-station-experts-explain-customs-exams-holds/)
- [CBP: First Airside CES at DFW](https://www.cbp.gov/newsroom/local-media-release/cbp-dfw-dnata-cargo-usa-unveil-nation-s-first-airside-centralized)
- [CBP: Cargo Examination](https://www.cbp.gov/border-security/ports-entry/cargo-security/examination)
- [CBP: Appendix D -- CES Operational & Facility Requirements](https://content.govdelivery.com/attachments/USDHSCBP/2025/06/05/file_attachments/3282792/Appendix%20D%20%E2%80%93%20Operational%20and%20Facility%20Characteristic%20and%20Minimum%20Requirements.pdf)
- [Flexport: List of Customs Holds and Exams](https://www.flexport.com/help/299-customs-holds-and-exams/)
- [USA Customs Clearance: Guide to Cargo Examinations](https://usacustomsclearance.com/process/cargo-examinations/)
- [Ship4WD: Customs Holds & Exams Types](https://ship4wd.com/import-guides/customs-holds-customs-exams)
- [Custom Goods Logistics: Examination Processes](https://www.custom-goods.com/services/examination)
- [CustomsCity: LAX CES Procedures](https://customscity.com/procedures-for-the-transport-of-air-cargo-to-the-centralized-examination-station-at-lax/)
- [Flexport: General Order](https://www.flexport.com/glossary/general-order/)
- [FreightAmigo: General Order for Importers](https://www.freightamigo.com/en/blog/logistics/understanding-general-order-what-importers-need-to-know-about-customs-delays/)
- [CargoWise: Customs and Compliance Software](https://www.cargowise.com/solutions/cargowise-customs/)
- [Magaya: Customs Filing Software](https://www.magaya.com/customs-filing-software-customs-brokers/)
- [19 CFR Part 19: Customs Warehouses, Container Stations](https://www.ecfr.gov/current/title-19/chapter-I/part-19)
- Internal knowledge: `04_clearance_process_today.md` Section 6 (Holds, Exams, Caging Operations)
- Internal knowledge: `08_customs_broker_operations.md` Section 13.3 (Express vs. Formal Entry)
