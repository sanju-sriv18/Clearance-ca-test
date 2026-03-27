# Shipper & Buyer Surface: Domain Expert Analysis
## Import/Export Operations Manager Perspective (15 Years Experience)

**Analyst Role**: Senior Import/Export Operations Manager
**Date**: 2026-02-07
**Surfaces Reviewed**: Shipper (Shipping Center), Buyer (GlobalShop)
**Method**: Screenshot review + full frontend/backend code audit

---

## Executive Summary

The Shipper surface has genuinely impressive bones: real tariff engine integration, real compliance screening (PGA, UFLPA, DPS), deterministic document requirements with legal citations, and a multi-product order workflow. This is far ahead of most demo platforms. However, a domain expert investor would immediately flag **critical gaps in the product data model** (no weight, no Incoterms, no importer-of-record), **a completely fake Buyer duty calculation**, and **missing valuation methodology** that would make real customs entries impossible. The Order-to-Shipment pipeline works but skips fundamental CBP entry requirements.

---

## PART 1: SHIPPER SURFACE ANALYSIS

### 1. Product Creation (Add Product)

**File**: `frontend/src/surfaces/shipper/screens/AddProduct.tsx`

#### What's Good
- Description quality assessment with follow-up questions (lines 27-57) -- excellent for HS classification accuracy
- Multiple source locations with facility types (factory/warehouse/subsidiary) -- shows real supply chain thinking
- HS code field accepts comma-separated codes (line 80-81) -- correct, products can have multiple classifications

#### P0 -- CRITICAL: Missing Regulatory Data Fields

A real importer MUST capture the following at the product level, none of which exist:

| Missing Field | Why Required | Regulatory Basis |
|---|---|---|
| **Weight per unit (kg/lbs)** | Required for duty calculation on weight-based tariffs (e.g., steel, chemicals), for Customs bond calculation, and for carrier booking | 19 CFR 141.61 |
| **Unit of Measure** | CBP requires specific UOM (each, dozen, kg, m2) matching HTS statistical suffix | HTS General Notes |
| **Country of Origin determination method** | Not just "where it ships from" -- must track substantial transformation, regional value content | 19 CFR 134 |
| **Manufacturer ID (MID)** | Required on CBP entry summary, derived from manufacturer name + address + country | 19 CFR 102.12 |
| **Product material composition** | Drives HS classification for textiles (fiber content %), metals (alloy %), chemicals | HTS chapters 50-63, 72-83 |
| **Intended end use** | Determines classification in many chapters (automotive vs. general use changes the HS code) | HTS Chapter/Section Notes |

**Code reference**: `AddProduct.tsx:78-86` -- the `createProduct` payload has no weight, no UOM, no material composition.

#### P1 -- HIGH: HS Code Entry is Manual and Unvalidated

- Users type HS codes as free text (line 197-202) with no format validation
- No check that the code matches the 10-digit HTS format (e.g., `8507.60.0020`)
- No integration with the classification engine during product creation -- the AI classifier only runs during route analysis
- A real platform would: (a) auto-classify on product creation, (b) validate format, (c) show the HTS description to confirm accuracy

#### P1 -- HIGH: No Valuation Method Selection

- Only captures "Value per Unit" with no context on valuation method
- CBP requires declaration of which of the 6 WTO valuation methods applies:
  1. Transaction Value (most common)
  2. Transaction Value of Identical Goods
  3. Transaction Value of Similar Goods
  4. Deductive Value
  5. Computed Value
  6. Fallback Method
- Missing: Incoterms (FOB/CIF/EXW/DDP) -- determines which costs are in the declared value
- Missing: Whether the declared value includes or excludes freight/insurance (critical for CIF vs. FOB)
- **Impact**: Without Incoterms, duty calculation is unreliable. A $10,000 FOB value vs. $10,000 CIF value produces different duty amounts.

**Code reference**: `AddProduct.tsx:206-215` -- just a bare number input with no Incoterms context.

#### P2 -- MEDIUM: Source Location Model is Incomplete

- Only captures: country code (2-3 chars), facility name, facility type (factory/warehouse/subsidiary)
- Missing: full address (needed for MID code), city, postal code, contact
- Missing: relationship to seller (related party transactions affect valuation rules per 19 CFR 152.103)
- Missing: facility certifications (ISO, CTPAT enrollment, AEO status)
- `maxLength={3}` on country code input (line 252) allows 3 characters, but ISO country codes are 2 characters

#### P2 -- MEDIUM: Currency List is Incomplete

- Only supports: USD, EUR, GBP, JPY, CNY (line 224-230)
- Missing major trade currencies: KRW, TWD, INR, MXN, BRL, CAD, AUD, THB, VND, SGD
- A shipment from Vietnam or Taiwan would have no currency option

---

### 2. Order Creation (Create Order)

**File**: `frontend/src/surfaces/shipper/screens/CreateOrder.tsx`

#### P0 -- CRITICAL: No Per-Line-Item Quantity Unit or Weight

- Line items only capture: product, source, quantity (integer) (lines 9-13)
- No weight per line (gross/net/tare) -- required for:
  - Customs bond amount calculation (19 CFR 113.13)
  - Weight-based duty items (e.g., steel at X cents per kg)
  - Carrier booking (air cargo charged by kg or dimensional weight)
- No unit of measure -- `quantity: 1` of what? 1 each? 1 pallet? 1 container?

#### P1 -- HIGH: Destination List is Hardcoded to 10 Countries

- `DESTINATIONS` array (lines 15-26) has only 10 countries
- Missing: most of the world including China (a major re-export destination), Singapore, UAE, Netherlands, etc.
- Should be a searchable country selector with all ISO 3166-1 countries
- Same issue in `ShipperProductDetail.tsx:29-38` (`COMMON_DESTINATIONS`)

#### P1 -- HIGH: No Incoterms Selection on Order

- No field for terms of sale (FOB, CIF, EXW, DDP, etc.)
- Incoterms determine: (a) who pays freight/insurance, (b) risk transfer point, (c) what's included in customs value
- Without this, the declared value for duty purposes is ambiguous

#### P1 -- HIGH: No Importer of Record (IOR) Information

- No field for who the importer of record is in the destination country
- CBP requires IOR name, address, and EIN/SSN/CBP assigned number
- No consignee information
- No customs broker designation

#### P2 -- MEDIUM: Order Summary Miscalculates with Mixed Currencies

- `totalValue` sums all line items assuming same currency (line 61-65)
- But products can have different currencies (USD, EUR, JPY, etc.)
- `formatCurrency(totalValue, "USD")` at line 239 always displays as USD
- If you mix a EUR product and a JPY product, the total is nonsensical

---

### 3. Order Analysis (Order Detail)

**File**: `frontend/src/surfaces/shipper/screens/OrderDetail.tsx`
**Backend**: `backend/app/api/routes/orders.py:320-506`

#### What's Good (Genuinely Impressive)
- Real tariff engine integration (`e2_tariff.calculate_tariff`) per line item
- Real compliance screening (`e3_compliance.screen_compliance`) per line item
- PGA flag detection (FDA, BIS, EPA, etc.)
- UFLPA risk screening
- Denied Party Screening
- Per-item tariff breakdown with categories (DUTY, SURCHARGE, TAX, FEE)
- Legal citations on tariff line items
- FTA eligibility checking (E4 engine)
- Narrative summary generation

#### P1 -- HIGH: Effective Rate Display May Confuse Users

- Screenshot shows "Rate: 530.0%" for the DE->US bracket order
- This likely means $530% displayed incorrectly -- should be the actual percentage
- `orders.py:489` calculates `effective_rate = round((total_charges / total_value * 100) if total_value else 0.0, 2)` -- this is correct math but the display at `OrderCard.tsx` needs validation

#### P1 -- HIGH: Document Requirements Only Use First Line Item's HS Code

- `AnalysisResultsPanel.tsx:345-346`: `{lineItems[0]?.hsCode && ...`
- Multi-product orders may have different HS codes requiring different documents
- E.g., an order with food products AND electronics needs BOTH FDA Prior Notice AND potentially BIS End-Use Certificate
- The document requirements component receives only the first item's HS code

#### P2 -- MEDIUM: No ADD/CVD (Antidumping/Countervailing Duty) Visibility

- The tariff engine may calculate these, but they're not called out separately in the UI
- ADD/CVD duties can be 100-400%+ and are a major cost factor
- A domain expert would expect: explicit ADD/CVD line items, case numbers, and the deposit rate
- Example: Chinese steel products face 25% Section 301 + up to 200%+ ADD

#### P2 -- MEDIUM: No Section 301/201/232 Tariff Visibility

- These are the "trade war" tariffs and they're enormous (25% on most Chinese goods)
- The tariff engine likely includes them, but the UI doesn't distinguish them from base duties
- An investor's expert would ask: "Where do I see Section 301 impact?"

---

### 4. Shipment Tracking (Shipment History + Detail)

**File**: `frontend/src/surfaces/shipper/screens/ShipmentHistory.tsx`
**File**: `frontend/src/surfaces/shipper/screens/ShipmentDetail.tsx`

#### What's Good
- Status categories: booked, in_transit, at_customs, inspection, held, cleared, delivered
- Hold/inspection alert banners with reason text
- Pre-clearance screening banner for booked status
- Waypoint tracking
- Predicted vs. Actual financials with variance calculation
- Transport mode display (air/ocean/ground)
- HAWB/HBL/Pro# terminology changes based on transport mode (lines 288-299)
- Chat-with-AI feature for shipment questions

#### P1 -- HIGH: No ETA (Estimated Time of Arrival)

- Nowhere in the shipment detail or list is there an ETA
- This is the #1 thing every importer checks every day
- Missing: estimated departure, estimated arrival at customs, estimated delivery
- No ISF (Importer Security Filing) deadline tracking (must be filed 24 hours before vessel departure for ocean)

#### P1 -- HIGH: No Entry Number or Entry Type

- No CBP entry number (11-digit format: PPP-NNNNNNN-C)
- No entry type indicator (01=consumption, 06=warehouse, 11=informal, 21=warehouse withdrawal)
- No entry date or liquidation date
- These are how importers track their shipments through customs

#### P1 -- HIGH: No Bond Information

- No continuous bond vs. single entry bond indication
- No bond sufficiency check (bond must be >= highest duty/tax/fee amount)
- Bond is required for ALL commercial imports over $2,500

#### P2 -- MEDIUM: Hardcoded Shipper Name in History

- `ShipmentHistory.tsx:49`: `getShipments(undefined, undefined, undefined, "Apex Trading Co.")`
- The shipper name is hardcoded -- this would need to be dynamic per user/company

#### P2 -- MEDIUM: No ISF/AMS Filing Status

- Ocean imports require ISF (10+2) filing at least 24 hours before departure
- Air imports require AMS (Automated Manifest System) filing
- Neither filing status is tracked or shown

#### P3 -- LOW: Missing Carrier-Specific Tracking

- No link to carrier tracking (FedEx/DHL/Maersk tracking page)
- No container/AWB number prominently displayed
- No real-time GPS or vessel tracking integration

---

### 5. Resolution Center

**File**: `frontend/src/surfaces/shipper/screens/ResolutionCenter.tsx`

#### What's Good
- Three-tab design (Product Lookup, Document Upload, Stuck Shipment) is practical
- AI-powered classification from description
- Document upload with field extraction
- Stuck shipment resolution suggestions with priority levels

#### P2 -- MEDIUM: Product Lookup Doesn't Save to Catalog

- The lookup classifies products but doesn't offer to save the result to the product catalog
- An importer would classify, then want to add it to their catalog for ordering
- No connection between resolution results and the product creation workflow

#### P2 -- MEDIUM: Document Upload Limited to PDF Only

- `handleFile` checks `file.name.endsWith(".pdf")` (line 211)
- Real trade documents come in many formats: Excel (packing lists), Word, XML (EDI), image scans
- Missing: multi-page document support, batch upload

---

## PART 2: BUYER SURFACE ANALYSIS

### 6. Storefront & Product Page

**File**: `frontend/src/surfaces/buyer/screens/StoreFront.tsx`
**File**: `frontend/src/surfaces/buyer/screens/ProductPage.tsx`
**Data**: `frontend/src/data/products.ts`

#### P0 -- CRITICAL: Buyer Duty Calculation is Completely Fake

This is the single biggest credibility risk for an investor demo.

**`ProductPage.tsx:51`**: `const taxes = Math.round(product.declaredValue * 0.05);`

- Import duties are hardcoded at **flat 5% of declared value** across ALL products
- This is completely wrong. Real duty rates vary from 0% to 400%+:
  - EV Batteries from China: 25% Section 301 + 3.4% base = ~28.4%
  - Cotton sweaters from China: 25% Section 301 + 19.7% base + potential ADD
  - Aluminum brackets from Germany: 0% (EU has no Section 301) + 5.7% base
  - Semiconductors from Taiwan: 0% base duty
  - Green tea from Japan: 0% base duty (some tea HS codes are duty-free)

**`ProductPage.tsx:49`**: `const shippingEstimate = 350;` -- hardcoded $350 shipping for all products regardless of weight, volume, origin, or mode.

**`products.ts`**: The `totalDuty` field in the catalog IS based on real data (e.g., prod-1 has $4,800 duty on $12,500 value = 38.4% effective rate for CN->US batteries). But `ProductPage.tsx` ignores this cached analysis and recalculates with `0.05`.

The breakdown shown to buyers:
```
Product base:     $12,500  (correct)
Import duties:    $625     (WRONG -- should be $4,800)
Taxes:            $625     (invented 5%)
Shipping:         $350     (hardcoded)
Total:            $14,100  (vs. real ~$17,650)
```

**Impact**: The buyer surface claims "No surprise fees at delivery" and "Duties & taxes included" -- but the included amount is wrong by potentially thousands of dollars.

#### P0 -- CRITICAL: Product Data is Static Client-Side, Not From Tariff Engine

- `StoreFront.tsx:8`: `import { CATALOG_PRODUCTS } from "@/data/products"`
- All 5 buyer products are hardcoded in `frontend/src/data/products.ts`
- `buyerPrice` is pre-computed and static -- it never calls the analysis pipeline
- The `totalDuty` values in the static data are plausible but frozen -- they don't update when tariff schedules change
- The buyer surface has ZERO connection to the real tariff engine that the shipper surface uses

#### P1 -- HIGH: No ADD/CVD Implications Shown

- Cotton Knit Sweater (prod-2) is from Xinjiang, China
- The product data correctly flags UFLPA risk (`readyLight: "red"`, `issues: ["UFLPA: Xinjiang cotton..."]`)
- But the buyer surface shows NONE of this -- the buyer just sees "$2,880" with "Duties & taxes included"
- A buyer purchasing Xinjiang cotton products would receive a DETAINED shipment at port
- The buyer surface should either: (a) not list UFLPA-flagged products, or (b) show a prominent warning

#### P1 -- HIGH: Missing Fee Components in Landed Cost

The buyer breakdown (ProductPage.tsx:158-168) shows:
- Product base
- Import duties (fake 5%)
- Taxes (fake 5%)
- Shipping ($350)

Missing real fee components:
- **Merchandise Processing Fee (MPF)**: 0.3464% of value, min $31.67, max $614.35
- **Harbor Maintenance Fee (HMF)**: 0.125% of value (ocean only)
- **Customs broker fees**: typically $150-$400 per entry
- **ISF filing fee**: typically $25-$50
- **Freight insurance**: typically 0.5-2% of value
- **Customs exam fees**: if selected for inspection ($300-$1,000+)
- **Drayage/last-mile**: domestic delivery from port to warehouse

#### P2 -- MEDIUM: No Country-Specific Buyer Pricing

- All products show a single price regardless of buyer's country
- Duty rates are destination-specific (US rates ≠ EU rates ≠ Japanese rates)
- The buyer surface assumes US destination for all
- No destination country selection in the buyer flow

---

### 7. Checkout

**File**: `frontend/src/surfaces/buyer/screens/CheckoutPrice.tsx`

#### P1 -- HIGH: Checkout Creates Pipeline Entries But Uses Fake Prices

- `handlePurchase` (lines 24-43) correctly calls `analyze()` for each cart item
- This creates entries in the pipeline with the correct analysis
- BUT the price the buyer paid was based on the fake 5% calculation
- There's a pricing disconnect between what the buyer pays and what the analysis actually costs

#### P2 -- MEDIUM: No Real Payment Processing or DDP Logic

- The "Complete Purchase" button is decorative -- no payment gateway
- For a DDP (Delivered Duty Paid) model (which "duties included" implies), the platform needs:
  - Real-time duty calculation at checkout (not hardcoded)
  - A margin buffer for duty variance risk
  - Reconciliation when actual duties differ from estimate
  - Drawback calculation if goods are re-exported

#### P3 -- LOW: Decorative Address/Payment Form

- Hardcoded address "Sarah Chen, 1234 Oak Street" (line 117-125)
- This is fine for demo purposes but should be acknowledged

---

## PART 3: CROSS-CUTTING CONCERNS

### P0 -- CRITICAL: No Importer of Record Architecture

Neither surface captures or manages:
- IOR EIN/tax identification number
- Customs bond number and type
- Power of Attorney relationship (who's authorized to file)
- CBP Form 5106 (Importer Identity)

This is the single biggest structural gap. Without an IOR, you cannot file a customs entry.

### P1 -- HIGH: Trade Agreement Benefits Not Surfaced to Shipper

- The backend has FTA eligibility checking (E4 engine) and the document system knows about FTA partners (`FTA_PARTNERS_US` in `documents.py:208-229`)
- But the shipper surface doesn't prominently show: "You could save $X by providing a Certificate of Origin under USMCA"
- FTA savings can be 5-25% of goods value -- this is a major selling point

### P1 -- HIGH: No Multi-Currency Handling in Orders

- Order creation sums line items across currencies without conversion (CreateOrder.tsx:61-65)
- No exchange rate management
- CBP requires all values declared in USD or converted at official exchange rates

### P2 -- MEDIUM: Document System is Excellent But Incomplete

The document requirements engine (`documents.py`) is one of the strongest parts of the platform:
- Deterministic, rule-based (no LLM dependency for requirements determination)
- Proper legal citations (19 CFR, 21 CFR, etc.)
- FTA-conditional Certificate of Origin logic
- PGA-conditional FDA/BIS documents
- UFLPA supply chain traceability when flagged
- Full field schemas for each document type

But missing documents:
- **CBP Form 7501 (Entry Summary)** -- the primary customs filing document
- **ISF (Importer Security Filing)** -- required 24hrs before ocean vessel departure
- **AES/EEI (Automated Export System)** -- required for exports over $2,500
- **SED (Shipper's Export Declaration)** -- certain exports require this
- **Phytosanitary Certificate** -- required for plant/wood products
- **TSCA Certification** -- required for chemical substances
- **FCC Declaration** -- required for electronic devices
- **DOT/NHTSA Compliance** -- required for motor vehicles and parts

### P2 -- MEDIUM: No Audit Trail / Entry History

- No record of past duty payments
- No liquidation tracking (CBP can reliquidate entries up to 314 days after entry)
- No drawback eligibility tracking
- No reconciliation workflow for estimated vs. actual duties

---

## FINDINGS SUMMARY

### P0 (Show-Stopper -- Investor Expert Would Flag Immediately)

| # | Finding | Location | Impact |
|---|---|---|---|
| 1 | Buyer duty calculation is hardcoded 5%, not from tariff engine | `ProductPage.tsx:51` | Completely undermines "duties included" value prop |
| 2 | Missing critical product data fields (weight, UOM, material, MID) | `AddProduct.tsx:78-86` | Cannot file real customs entries |
| 3 | No Importer of Record architecture anywhere | Cross-cutting | Fundamental regulatory requirement for ALL imports |
| 4 | Buyer product data is static/hardcoded, not from analysis engine | `data/products.ts`, `StoreFront.tsx:8` | Buyer surface disconnected from real engine |

### P1 (High -- Would Need Resolution Before Production)

| # | Finding | Location | Impact |
|---|---|---|---|
| 5 | No Incoterms/valuation method on products or orders | `AddProduct.tsx`, `CreateOrder.tsx` | Declared value ambiguous for duty calculation |
| 6 | No ETA or ISF deadline tracking on shipments | `ShipmentDetail.tsx` | #1 importer daily operational need missing |
| 7 | No entry number, entry type, or bond information | `ShipmentDetail.tsx` | Cannot track shipments through customs |
| 8 | FTA savings not surfaced to shipper | Cross-cutting | Missing major cost optimization opportunity |
| 9 | Hardcoded 10-country destination list | `CreateOrder.tsx:15-26` | Cannot ship to most of the world |
| 10 | Document requirements only use first line item's HS code | `AnalysisResultsPanel.tsx:345` | Multi-product orders get incomplete doc lists |
| 11 | No ADD/CVD or Section 301 visibility in buyer surface | `ProductPage.tsx` | Buyer unaware of massive cost implications |
| 12 | No multi-currency conversion in order totals | `CreateOrder.tsx:61-65` | Nonsensical totals for mixed-currency orders |
| 13 | No IOR/consignee information on orders | `CreateOrder.tsx` | Cannot complete CBP entry filing |
| 14 | Missing landed cost components (MPF, HMF, broker fees) in buyer | `ProductPage.tsx:158-168` | Understated total cost to buyer |

### P2 (Medium -- Important for Credibility)

| # | Finding | Location | Impact |
|---|---|---|---|
| 15 | Source location model incomplete (no full address, no MID) | `AddProduct.tsx:239-280` | Cannot derive Manufacturer ID |
| 16 | HS code entry is unvalidated free text | `AddProduct.tsx:197-202` | Invalid codes can be entered |
| 17 | Missing document types (7501, ISF, AES, Phyto, TSCA, FCC) | `documents.py` | Incomplete regulatory coverage |
| 18 | No audit trail or liquidation tracking | Cross-cutting | No historical compliance record |
| 19 | Incomplete currency list (missing KRW, TWD, INR, etc.) | `AddProduct.tsx:224-230` | Cannot price products from major origins |
| 20 | Resolution Center lookup doesn't save to catalog | `ResolutionCenter.tsx` | Classified products lost |
| 21 | Document upload limited to PDF only | `ResolutionCenter.tsx:211` | Excludes Excel, Word, XML documents |
| 22 | ADD/CVD not called out separately in tariff display | `AnalysisResultsPanel.tsx` | Major cost factor hidden in aggregate |
| 23 | No country-specific buyer pricing | `StoreFront.tsx` | Single price assumes US destination |
| 24 | Hardcoded shipper name in shipment history | `ShipmentHistory.tsx:49` | Not multi-tenant ready |

### P3 (Low -- Nice to Have)

| # | Finding | Location | Impact |
|---|---|---|---|
| 25 | No carrier-specific tracking links | `ShipmentDetail.tsx` | Users can't link to FedEx/DHL tracking |
| 26 | Decorative checkout form | `CheckoutPrice.tsx:117-125` | Fine for demo, noted for completeness |
| 27 | No product images in shipper catalog | `ShipperCatalog.tsx` | Minor UX enhancement |

---

## What an Investor's Domain Expert Would Say

**Positive (and these matter):**
> "The tariff engine integration is real. The compliance screening with PGA flags, DPS, and UFLPA is real. The document requirement system with legal citations is genuinely impressive. This team understands trade compliance at a deeper level than most startups I've seen. The shipment lifecycle with holds, inspections, and pre-clearance screening shows they understand the operational reality."

**Critical feedback:**
> "But the buyer surface is a liability -- that 5% hardcoded duty rate is the kind of thing that destroys credibility in a demo. If I'm a potential customer and I see the buyer surface first, I'd dismiss the entire platform. Fix this by connecting the buyer to the real tariff engine, even if it means making the page load a second slower."

> "The product data model is too thin for real customs work. You need weight, Incoterms, material composition, and manufacturer details at minimum. Without these, you can't file a real CBP entry. This isn't a feature request -- it's a regulatory requirement."

> "Where's the IOR? Where's the bond? Where's the entry number? These are the fundamental building blocks of a customs entry. You have the analysis engine but you're missing the entity management layer that makes it usable for real import operations."

**Recommended priority for next sprint:**
1. Connect buyer surface to real tariff engine (kill the 5% hardcode)
2. Add weight, Incoterms, and material composition to product model
3. Add IOR/entity management
4. Expand country/currency lists
5. Add ETA and entry tracking to shipments
