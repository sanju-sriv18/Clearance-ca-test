# Broker Surface — Domain Expert Analysis

**Analyst Perspective:** Licensed Customs Broker, 20 years experience
**Date:** 2026-02-07
**Scope:** BrokerDashboard, BrokerQueue, EntryDetail, CBPResponses, Communications screens
**Goal:** Would a domain expert investor advisor flag this platform as credible or "obviously built by non-brokers"?

---

## Executive Summary

**Overall Assessment: Credible demo, but a seasoned broker would identify gaps within 5 minutes.**

The platform demonstrates genuine understanding of the CBP entry lifecycle — the CF-28/CF-29 workflow, bond verification, PGA flags, UFLPA screening, and GO deadlines are all present and correctly named. The filing status state machine (draft → pending_broker_approval → submitted → cf28/cf29_pending → released) is directionally correct. The AI-assisted work plan with time estimates and financial impact tracking shows real operational thinking.

However, a licensed customs broker or domain expert advisor would flag several categories of issues:
1. **Missing legally required CF-7501 data fields** that every real entry contains
2. **Oversimplified duty/fee calculations** that would never pass CBP validation
3. **No ISF/10+2 workflow** — a glaring omission for any import platform
4. **No liquidation lifecycle** — entries show "released" as terminal state
5. **Entry number format is fabricated**, not ACE-compliant
6. **No multi-line tariff treatment** — each entry appears to be single-product

The good news: the architectural bones are solid. The data model (Shipment → EntryFiling with JSONB analysis/codes/financials) can accommodate all missing fields. The intelligence service demonstrates the right patterns. Fixing these issues is a data model expansion + UI addition, not a rearchitecture.

---

## Screen-by-Screen Analysis

### 1. Broker Dashboard (BrokerDashboard.tsx)

**Screenshot:** `01-dashboard.png`

#### What Works Well
- **Morning briefing with sequenced work plan** — This is genuinely innovative. Real brokers work from prioritized task lists. The time estimates, deadline cascading, and capacity utilization percentage show operational depth that would impress a domain expert.
- **GO countdown integration** — Showing General Order deadlines with days remaining is critical. The 15-day window before goods go to General Order is one of the highest-urgency items in a brokerage.
- **Financial impact on alerts** — Storage accrued, rate per day, duty exposure. Real brokers think in dollars.
- **Stuck shipment alerts** — "Held shipment without entry" with severity-based color coding. This is exactly what a broker needs to see.
- **Unassigned Work table** — The "Claim" workflow for unassigned shipments is realistic. Brokerages route work this way.

#### Issues

| # | Finding | Severity | Code Reference |
|---|---------|----------|----------------|
| D1 | **Stats tiles don't show "Liquidated" or "Protest Filed" stages** — The pipeline shows Draft → Filed → CBP Response → Released but the real lifecycle continues: Released → Liquidated (314 days later) → Protest window. Every broker tracks post-release. The "Released" status appears terminal. | P1 | `BrokerDashboard.tsx:528-534` (pipeline stages), `broker.py:528-534` |
| D2 | **No ISF (Importer Security Filing) status anywhere** — ISF 10+2 must be filed 24 hours before vessel lading for ocean shipments. This is legally required (19 CFR 149). Any platform without ISF tracking is incomplete for ocean freight. The `transport_mode` field exists but ISF obligations aren't surfaced. | P0 | Missing entirely from dashboard and all screens |
| D3 | **"Held" stat shows `stats.held` but maps to `rejected` count in backend** — `broker.py:523` maps `held` to `status_counts.get("rejected", 0)`. These are fundamentally different states. "Held" means CBP is examining/detaining; "Rejected" means the entry was denied. | P2 | `broker.py:523`, `BrokerDashboard.tsx:612-614` |
| D4 | **License number display format** — Shows "CHB CHB-4892" but CBP license numbers are 5 digits (e.g., "12345") issued by district. The "CHB-" prefix with 4-digit number isn't a real format. Minor but a domain expert would notice. | P3 | `BrokerDashboard.tsx:314` |
| D5 | **No distinction between Type 01 (Consumption) and Type 06 (Bonded Warehouse) entries** — All entries appear to be Type 01. Real brokerages handle multiple entry types daily (01, 06, 11 FTZ, 21 Mail, 46 Warehouse). | P2 | `broker.py:1115` hardcodes `entry_type="01"` |
| D6 | **Capacity utilization uses 8-hour day (480 min)** but doesn't account for overhead — Real brokers spend ~40% of their day on non-filing tasks (calls, research, compliance reviews). A 480-minute capacity overestimates throughput. | P3 | `broker_intelligence.py:89` |

---

### 2. My Queue (BrokerQueue.tsx)

**Screenshot:** `02-queue.png`

#### What Works Well
- **Card-based layout with status badges** — Clear visual hierarchy: entry number, product, importer, route, value.
- **Checklist progress bar** — Showing completion percentage gives brokers quick triage ability.
- **Inline action buttons** — "Approve", "Respond", "Req Docs" directly on cards saves clicks.
- **Filter by status and priority** — Essential for a queue with 50+ entries.
- **Pagination** — PAGE_SIZE of 50 with offset/limit is appropriate.

#### Issues

| # | Finding | Severity | Code Reference |
|---|---------|----------|----------------|
| Q1 | **No sort by deadline/urgency** — Queue sorts by `created_at.desc()` in backend. A broker needs to sort by: CF-28 deadline remaining, GO deadline, days since arrival, value at risk. Created date is nearly useless for prioritization. | P1 | `broker.py:951` |
| Q2 | **No filter for "Released" or "Released today"** — Status filter options stop at "Exam Scheduled". Released entries aren't in the dropdown. Brokers need to review recently released entries for post-release compliance. | P2 | `BrokerQueue.tsx:271-279` |
| Q3 | **Queue card doesn't show HTS code** — The HS/HTS code is the single most important data point for a customs broker. It determines duty rate, PGA requirements, ADD/CVD applicability, and quotas. Not showing it on the queue card forces unnecessary drill-down. | P1 | `BrokerQueue.tsx:152-159` |
| Q4 | **No ADD/CVD (Antidumping/Countervailing Duty) indicator** — ADD/CVD orders dramatically change duty rates (sometimes 200%+). There's no field or flag for this anywhere in the data model. Any shipment from CN of steel, aluminum, solar panels, etc. would need this. | P0 | Missing from data model entirely |
| Q5 | **Entry number format is fabricated** — `broker.py:1103-1107` generates random `{3digits}-{7digits}-{1digit}`. Real CBP entry numbers follow the format: Filer Code (3 chars) + Entry Number (7 digits) + Check Digit (1 digit), where the filer code is the broker's assigned code, not random. | P2 | `broker.py:1103-1107` |
| Q6 | **No quantity, weight, or container info** — Queue cards show value but not physical characteristics. Brokers need to know if it's 1 pallet or 40 containers — it affects exam risk, port fees, and urgency. | P2 | `BrokerQueue.tsx:152-159` |

---

### 3. Entry Detail (EntryDetail.tsx)

**Screenshot:** `03-entry-detail-top.png` (showing error state — "Invalid entry ID format")

**Note:** The screenshot shows an error state, which itself is a UX issue, but I'll analyze the code for the full Entry Detail.

#### What Works Well
- **Pre-Filing Verification Checklist** — The 8-item computed checklist (commercial invoice, packing list, B/L, HS classification, valuation, origin cert, PGA docs, bond) is a solid starting framework. The status-driven verification (verified/uploaded/issues/missing/not_applicable) with expandable validation checks is genuinely useful.
- **Bond type selection** — Offering single transaction vs. continuous vs. no bond with inline UI is correct workflow.
- **FTA-aware origin cert** — The `FTA_PARTNERS_US` check to mark origin cert as not_applicable for non-FTA countries is smart and correct.
- **Analysis summary with confidence sections** — Showing classification confidence (HIGH/MEDIUM/LOW) with improvement steps is valuable decision support.
- **Risk flags with screening matches table** — Entity screening against SDN/BIS/UFLPA lists with match scores is real compliance work. The financial exposure breakdown (storage + duty) is operationally useful.
- **Contextual assistant actions** — "Request documents from shipper", "Set bond type", "Draft CBP response" based on entry state is intelligent UI.
- **Communications tracker** — Showing sent/received/no-response communications inline on the entry is excellent for audit trail.

#### Issues

| # | Finding | Severity | Code Reference |
|---|---------|----------|----------------|
| E1 | **Missing critical CF-7501 fields** — A real entry summary (7501) requires: Importer of Record Number (EIN/IRS#/CBP assigned), Ultimate Consignee, Manufacturer ID (MID), Country of Origin per line item, Entered Value per line item, Relationship indicator (related parties), SPI (Special Program Indicator for FTAs), Gross Weight, Net Weight, IT number, and Port of Unlading. The `EntrySummaryData` schema has some of these but the EntryDetail UI displays almost none of them. | P0 | `broker.py:379-401` (schema partial), `EntryDetail.tsx:766-848` (display omits) |
| E2 | **Importer number is fabricated** — `broker.py:1614` generates `XX-{company[:4]}-12345`. Real importer numbers are IRS EIN (XX-XXXXXXX), CBP assigned number, or Social Security Number. This format doesn't match any real CBP identifier format. | P1 | `broker.py:1614` |
| E3 | **Duty rate calculation is placeholder** — `broker.py:1596-1602` uses a static lookup by 2-digit HS chapter with hardcoded rates (chapter 84→2%, 85→3%, etc.). Real HTS rates are at the 8-digit or 10-digit level and vary from 0% to 48.5%+ (excluding ADD/CVD). This calculation would be wrong for most entries. | P1 | `broker.py:1596-1602` |
| E4 | **MPF formula uses wrong rate** — Backend uses `declared_value * 0.003464` but the current MPF ad valorem rate is 0.3464% = 0.003464, min $31.67, max $614.35 (adjusted annually). The code uses `min=29.66, max=575.35` which are outdated 2022 figures. As of FY2026 these would be wrong. | P2 | `broker.py:1605`, `EntryDetail.tsx:1228-1229` |
| E5 | **HMF only charged on ocean** — `broker.py:1606` only applies HMF for `transport_mode == "ocean"`. This is correct — Harbor Maintenance Fee is 0.125% for ocean and doesn't apply to air. Good. | OK | `broker.py:1606` |
| E6 | **No "Entered Value" vs "Declared Value" distinction** — Customs law distinguishes between declared/invoice value and entered/appraised value. The entered value is what CBP uses for duty, which may differ from the declared value after assists, royalties, packing, etc. The UI only shows "Declared Value". | P1 | `EntryDetail.tsx:797` |
| E7 | **Port of entry defaults to 2704 (LA/LB)** — `broker.py:1116` hardcodes port 2704. While LA/Long Beach is the highest-volume port, real entries must match the actual port of arrival. No mechanism exists to change port on the entry. | P2 | `broker.py:1116` |
| E8 | **No "Other Government Agency" (OGA/PGA) detail** — The checklist has "PGA Documentation" as a single item, but real PGA requirements are per-agency: FDA (food, drugs, cosmetics, medical devices), EPA (vehicles, engines), FCC (electronics), CPSC (consumer products), APHIS (agriculture), TTB (alcohol/tobacco). Each has different forms and filing requirements. | P1 | `broker.py:355-377` |
| E9 | **Approve & Sign lacks regulatory weight** — The attestation text is generic: "I attest that this entry is true and accurate." A real broker attestation under 19 USC 1641 carries legal liability. The signature should reference the broker's license number, the power of attorney from the importer, and the specific legal obligations. No power of attorney verification exists. | P1 | `broker.py:1473-1476` |
| E10 | **No ISF status on entry detail** — For ocean shipments, ISF filing status should appear on the entry. ISF is a separate filing from the entry but they're tightly linked. | P1 | Missing from EntryDetail.tsx |
| E11 | **Line items table lacks required fields** — Shows product, HS code, quantity, value, origin. Missing: unit of measure (required per HTS), gross/net weight, ADCVD case number, SPI, manufacturer ID. | P1 | `EntryDetail.tsx:826-832` |
| E12 | **Single "Approve & Submit" flow** — Real workflow has more states: Draft → Pre-liminary Review → Classification Complete → Broker Review → Broker Approval → Submission → Accepted/Rejected by CBP → Conditionally Released → Finally Liquidated. The current flow compresses several distinct regulatory steps. | P2 | `EntryDetail.tsx:702-707` |
| E13 | **No ACE (Automated Commercial Environment) integration concept** — The submit action goes to an internal state. Real entries are filed via ACE and responses come back as ACE status messages. The platform should at least conceptually reference ACE as the filing target. | P2 | `broker.py:1539-1541` |

---

### 4. CBP Responses (CBPResponses.tsx)

**Screenshot:** `05-cbp-responses.png`

#### What Works Well
- **CF-28 vs CF-29 distinction** — Correctly identifies CF-28 as "Request for Information" and CF-29 as "Notice of Action". This is fundamental and they got it right.
- **Deadline countdown** — "29d remaining" with urgency coloring. CF-28 has a 30-day response window; this tracking is essential.
- **Category-aware guidance** — The classification/valuation/origin branching with specific recommended attachments shows domain knowledge. "GRI-based analysis" for classification is the correct approach.
- **AI Draft for CF-28** — Offering an AI-drafted response with supporting points is high-value. This is where AI can genuinely help — drafting the initial response framework.
- **Accept vs. Protest for CF-29** — The binary choice is correct. CF-29 gives the importer the option to accept the proposed action or file a protest (19 USC 1514).
- **Proactive guidance panel** — Per-category guidance with recommended attachments (e.g., "Certificate of origin, Manufacturing records, Bill of materials, Factory audit report" for an origin inquiry) demonstrates real customs knowledge.

#### Issues

| # | Finding | Severity | Code Reference |
|---|---------|----------|----------------|
| C1 | **CF-28 deadline is 30 days, not always** — The default of 30 days is common but CF-28s can specify different timeframes. The backend should track the specific deadline from the CF-28 document, not assume 30. | P2 | `CBPResponses.tsx:27` defaults to 30 |
| C2 | **No CF-29 protest timeline** — A protest under 19 USC 1514 must be filed within 180 days of liquidation (not 30 days from the CF-29). The CF-29 response window is 20 days to accept; protest is a separate longer timeline. The UI conflates these. | P1 | `broker.py:1708` uses `cbp.get("deadline_days") or cbp.get("protest_deadline_days")` — mixing windows |
| C3 | **No "Request for Information" vs "Notice of Action" detail types** — CF-28s can be about classification, value, origin, marking, country of origin marking, NAFTA/USMCA preference claims, or other. The category field exists but the UI doesn't show all real categories. | P2 | `broker.py:1710-1733` only handles classification/valuation/origin |
| C4 | **No attachment upload on CF-28 response** — The response modal has a text area but no file upload capability. Real CF-28 responses almost always require documentary evidence (invoices, lab reports, specs). The schema has `attachments: list[str]` but the UI doesn't expose it. | P1 | `CBPResponses.tsx:291-303` — text only, no upload |
| C5 | **No protest form fields** — Filing a protest (CBP Form 19) requires specific fields: protest number, protest category, nature of protest, description of merchandise, protest basis (legal authority). The current "protest reason" textarea is far too simplistic. | P1 | `CBPResponses.tsx:259-264` |
| C6 | **Exam response is "Awaiting Exam" with no actions** — When CBP schedules an exam, the broker needs to: coordinate with CES (Container Examination Station), arrange for cargo availability, pay exam fees, and potentially attend. The "Awaiting Exam" button with no actions misses real workflow. | P2 | `CBPResponses.tsx:146-155` |
| C7 | **No Ruling Request capability** — When classification is disputed, brokers frequently request binding rulings from CBP (19 CFR Part 177). This is a standard workflow that's absent. | P3 | Missing entirely |

---

### 5. Communications (Communications.tsx)

**Screenshot:** `06-communications.png`

#### What Works Well
- **Party type categorization** — Shipper, Importer, Carrier, CBP. These are the correct parties in a customs transaction.
- **Channel options** — Email, Phone, Portal. Realistic communication channels.
- **AI-drafted communications** — The ability to auto-draft missing document requests, classification inquiries, and value clarifications is genuinely useful.
- **Purpose-driven drafts** — "Request Missing Documents", "Classification Inquiry", "Value Clarification" as dropdown purposes map to real broker communications.

#### Issues

| # | Finding | Severity | Code Reference |
|---|---------|----------|----------------|
| M1 | **Missing party types** — No "Freight Forwarder", "Consignee", "Notify Party", "Foreign Supplier/Manufacturer". These are all parties a broker regularly communicates with. | P2 | `Communications.tsx:173-183` |
| M2 | **No message threading by entry/shipment** — Messages exist in a flat list with no grouping. A broker handling 20 entries needs to see all communications for a specific entry grouped together. | P2 | `Communications.tsx:377-381` |
| M3 | **"Start a conversation with a shipper or carrier"** — The empty state text is misleading. Brokers don't "start conversations" — they request documents, send compliance notices, coordinate exam attendance, and issue Powers of Attorney. The terminology should be more professional. | P3 | `Communications.tsx:374` |
| M4 | **No document request templates** — Real brokerages have standard letter templates for: Power of Attorney, Missing Document Request, Classification Questionnaire, ISF Data Request, and Customs Bond Application. The AI draft helps but templates are faster. | P3 | Missing from Communications.tsx |
| M5 | **No Power of Attorney (POA) management** — Before a broker can file for any importer, they need a valid POA. This is legally required (19 CFR 141.46). There's no POA tracking anywhere in the platform. | P1 | Missing entirely from all screens |

---

## Cross-Cutting Issues

### Missing Regulatory Workflows (P0)

| # | Missing Workflow | Impact | Real-World Consequence |
|---|-----------------|--------|----------------------|
| X1 | **ISF (Importer Security Filing) / 10+2** | Every ocean shipment requires ISF 24h before lading. $5,000 penalty per violation. No ISF workflow exists anywhere. | CBP penalties, holds, denied loading |
| X2 | **ADD/CVD (Antidumping/Countervailing Duty)** | No data field for AD/CVD case numbers, cash deposit rates, or special duty orders. Wrong duties = post-liquidation demand letters. | Massive duty underpayment, broker liability |
| X3 | **Post-Release / Liquidation tracking** | Entries liquidate ~314 days after release. The platform treats "released" as final. No liquidation date tracking, no protest deadline monitoring. | Missed protest deadlines, lost refund opportunities |
| X4 | **Reconciliation (Entry Type 09)** | For importers who use estimated values at entry and reconcile later. No reconciliation workflow. | Non-compliance with CBP reconciliation program |

### Missing Data Fields on CF-7501

The following fields are legally required on CBP Form 7501 (Entry Summary) and are not tracked or displayed:

| Field | CF-7501 Block | Status in Platform |
|-------|--------------|-------------------|
| Entry Number (with filer code) | Block 1 | Fabricated format |
| Entry Type | Block 2 | Present (hardcoded "01") |
| Entry Date | Block 3 | Present |
| Port Code | Block 4 | Present (hardcoded "2704") |
| Bond Type/Number | Block 5 | Partially present |
| Surety Code | Block 5a | **Missing** |
| Importer Number (EIN) | Block 6 | Fabricated |
| Importing Carrier | Block 7 | Present |
| Bill of Lading/AWB | Block 8 | Present |
| Manufacturer ID (MID) | Block 9 | **Missing** |
| Country of Origin | Block 10 | Present |
| Export Date | Block 11 | **Missing** |
| IT Number | Block 12 | **Missing** |
| IT Date | Block 13 | **Missing** |
| Missing Docs indicator | Block 14 | **Missing** |
| Country of Export | Block 15 | Present (same as origin) |
| Description of Merchandise | Block 27 | Present |
| HTS Number (10-digit) | Block 28 | Present (6-digit only) |
| AD/CVD Case Number | Block 29 | **Missing** |
| Rate of Duty | Block 30 | Placeholder |
| Relationship indicator | Block 31 | **Missing** |
| SPI (Special Program Indicator) | Block 32 | **Missing** |
| Net Quantity (reporting units) | Block 33 | **Missing** |
| Entered Value | Block 34 | Uses declared value |
| Duty/Tax | Block 35 | Placeholder calculation |
| Total Entered Value | Block 36 | Present |
| Other Fees | Block 39 | Partial (MPF/HMF) |
| Total Duty/Tax/Fees | Block 40 | Present (approximate) |

**Of 25 key fields, 12 are missing or fabricated.**

### Terminology Issues

| # | Current Term | Correct Term | Location |
|---|-------------|-------------|----------|
| T1 | "Approve" button on queue | "Review for Filing" or "Sign Entry" | `BrokerQueue.tsx:87-99` |
| T2 | "Submit to CBP" | "File via ACE" or "Transmit Entry" | `EntryDetail.tsx:1155-1157` |
| T3 | "Released" | Should distinguish "Conditionally Released" vs "Finally Released" | `BrokerDashboard.tsx:43` |
| T4 | "Rejected" | CBP doesn't "reject" entries — they "refuse" or "deny entry". Rejected implies the filing format was wrong. | `BrokerDashboard.tsx:44` |
| T5 | "Exam" (generic) | Should specify: VACIS (X-ray), Tailgate (partial unload), Intensive (full devanning), Stratified | `BrokerDashboard.tsx:42` |
| T6 | "Bond Verification" | Should specify "Customs Bond (STB/Continuous)" with surety info | `broker.py:104` |

---

## What a Real Broker Expects vs. What They See

### Immediate "This is fake" signals:
1. Entry numbers with random digits instead of the broker's filer code
2. No ISF anywhere — ocean brokers file ISF daily
3. No MID (Manufacturer ID) — this is one of the first things you enter
4. Duty rates that are obviously wrong (5% flat rate for everything)
5. No importer EIN/number management
6. No Power of Attorney tracking

### What Would Impress Them:
1. The intelligent work plan with deadline cascading and financial impact
2. UFLPA screening with supply chain risk assessment
3. CF-28 AI drafting with category-aware guidance
4. GO deadline tracking with storage cost accrual
5. PGA requirement detection from HS classification
6. Bond type selection workflow
7. Real-time analysis status tracking (pending → stale → complete)

---

## Prioritized Recommendations

### P0 — Must Fix for Credibility (Investor Demo Blockers)

1. **Add ISF filing status** — At minimum, a status indicator on ocean entries showing ISF filed/pending/late. Even a mock status is better than omission.
2. **Add ADD/CVD flag** — A boolean or case number field on entries. When an HTS code is subject to an AD/CVD order, the entry must show the AD/CVD deposit rate.
3. **Fix entry number format** — Use the broker's filer code prefix instead of random digits.
4. **Add key CF-7501 fields to Entry Detail** — MID, EIN, SPI, export date, gross weight at minimum.

### P1 — Should Fix Before Investor Review

5. **Fix duty rate calculations** — Either use real HTS lookup or clearly label as "estimated" with disclaimers.
6. **Add attachment upload to CF-28 responses** — The backend schema supports it; just need the UI.
7. **Add Power of Attorney tracking** — Even a simple "POA on file: Yes/No" per importer.
8. **Add post-release status** — Liquidation date estimate, protest deadline.
9. **Expand PGA to per-agency breakdown** — FDA, EPA, CPSC, APHIS at minimum.
10. **Add "Entered Value" distinct from "Declared Value"** — With adjustments for assists, royalties, etc.

### P2 — Good to Have for Depth

11. **Support entry types beyond 01** — At least 06 (Warehouse) and 11 (FTZ).
12. **Add queue sorting by urgency/deadline** — Not just created date.
13. **Show HTS code on queue cards** — Most important data point for brokers.
14. **Fix "Held" vs "Rejected" conflation** — These are different states.
15. **Update MPF min/max to current figures** — Annually adjusted.

### P3 — Nice to Have

16. **Broker license number format** — Use real 5-digit format.
17. **Adjust capacity model** — Account for overhead time.
18. **Add message templates** — Standard document request letters.
19. **Ruling request workflow** — 19 CFR Part 177 binding ruling requests.

---

## Verdict for Investor Domain Expert

**Would pass initial scrutiny** — The platform clearly demonstrates understanding of the CBP customs clearance process, uses correct terminology for major workflows (CF-28/CF-29, bond verification, PGA flags, entity screening), and the AI-assisted work planning shows genuine operational innovation.

**Would fail detailed review** — An advisor with customs brokerage experience would flag the missing ISF workflow, fabricated entry numbers, placeholder duty calculations, and absence of ADD/CVD handling as signs the platform hasn't been tested with real entry data. The missing CF-7501 fields would particularly concern someone who knows that CBP ACE validation would reject these entries.

**Fix estimate:** The P0 items (ISF status, ADD/CVD flag, entry number fix, key 7501 fields) are primarily data model additions and UI fields — likely 2-3 days of work. The architectural foundation is sound and can support all required expansions.
