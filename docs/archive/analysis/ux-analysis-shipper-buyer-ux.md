# UX Analysis: Shipper & Buyer Surfaces

**Analyst**: Senior Product/UX Designer
**Date**: 2026-02-07
**Scope**: Shipper surface (product catalog, orders, shipments, resolution center) + Buyer surface (storefront, product page, checkout)

---

## Executive Summary

The Shipper surface is surprisingly mature for an early product — the order-to-shipment flow is coherent, the DocumentRequirements component is a standout pattern, and the ShipmentDetail screen packs genuine intelligence. The Buyer surface, by contrast, is a polished-looking facade over hardcoded data — prices are static, duty calculations are fake, and the "duties included" promise is empty.

The platform's biggest UX risk is the gap between what the Shipper surface *can* do (real tariff engine, real compliance screening, AI document generation) and what the Buyer surface *pretends* to do (hardcoded 5% tax, static product catalog, no API calls for pricing). Closing this gap is the highest-leverage product work available.

---

## SHIPPER SURFACE

### Screen 1: Product Catalog (`ShipperCatalog.tsx`)

**Screenshot**: `shipper/01-catalog.png`

**What works well:**
- Clean card layout with essential info at a glance (name, category, price, source locations, HS codes)
- Subtle `MapPin` icons with facility names and country codes give quick origin visibility
- HS code shown in monospace — a nice detail for trade professionals
- Staggered animation on card load (framer-motion) adds polish without being distracting

**Issues:**

| # | Issue | Severity | File:Line |
|---|-------|----------|-----------|
| S1 | **No "Add Product" button in the header** — the CTA is a dashed-border footer card that may be below the fold. Primary action should be prominent. | P2 | `ShipperCatalog.tsx:55-61` |
| S2 | **Empty error state is swallowed** — `.catch(() => {})` means network failures show nothing, not even a retry button. | P1 | `ShipperCatalog.tsx:21` |
| S3 | **No search or filter** — Unlike OrderList and ShipmentHistory which have search bars, the catalog has none. For catalogs >10 products, this becomes a problem. | P3 | `ShipperCatalog.tsx:25-64` |
| S4 | **No product count** — Header says "Your product catalog" but doesn't show how many. Small but useful orientation. | P4 | `ShipperCatalog.tsx:34-36` |
| S5 | **No indication of compliance readiness per product** — The product detail screen runs analysis and shows green/yellow/red, but the catalog cards don't surface this. A shipper scanning the list can't tell which products need attention. | P2 | `ShipperCatalog.tsx:67-110` |

**Ideal experience**: Each catalog card shows a small readiness indicator (green/yellow/red dot) based on the last route analysis. The "+ Add Product" button is in the header next to the title. Search/filter is present. Error states show a retry prompt.

---

### Screen 2: Add Product (`AddProduct.tsx`)

**Screenshot**: `shipper/02-add-product.png`

**What works well:**
- **Description Quality Panel is excellent** — live AI assessment as you type, with follow-up questions to improve classification accuracy (`AddProduct.tsx:27-57`). This is a genuinely intelligent feature.
- Debounced quality check (1500ms) avoids spamming the API
- Follow-up questions expand inline with smooth animation
- Source locations are multi-entry with facility type selector
- Good placeholder text guides users ("e.g. EV Battery Pack 75kWh")

**Issues:**

| # | Issue | Severity | File:Line |
|---|-------|----------|-----------|
| S6 | **No inline validation before submit** — Category, HS codes, and value are all optional with no guidance on what's needed for analysis to work. If you skip HS codes, does classification fail? No feedback until you try to analyze. | P2 | `AddProduct.tsx:106-301` |
| S7 | **Country code is a free-text field** — The source location country code field says "CC" but accepts any string. Should be a dropdown or validated ISO-2 country picker. A typo here ("CHN" instead of "CN") silently breaks routing. | P1 | `AddProduct.tsx:246-252` |
| S8 | **No error feedback on create failure** — `catch` block does nothing except stop the spinner. User sees the form freeze and has no idea what went wrong. | P1 | `AddProduct.tsx:88-90` |
| S9 | **Currency limited to 5 options** — Only USD, EUR, GBP, JPY, CNY. Missing common trade currencies (KRW, TWD, MXN, CAD, AUD, INR, VND, THB). | P3 | `AddProduct.tsx:224-230` |
| S10 | **Description quality check errors silently swallowed** — If the AI quality endpoint fails, user sees no indication. The loading spinner just stops. | P3 | `AddProduct.tsx:33` |

**Ideal experience**: Country code is a searchable dropdown with flag icons. Required fields (for analysis to work) are clearly marked. Create failures show a toast with the error message. The description quality panel is the star feature and should be promoted more — perhaps showing a classification preview before product creation.

---

### Screen 3: Product Detail (`ShipperProductDetail.tsx`)

**What works well:**
- **Auto-analyzes on first load** with default route (`ShipperProductDetail.tsx:110-115`) — immediately shows value without requiring user action
- Source location selector with facility type icons (Factory, Warehouse, Subsidiary)
- Destination selector with 8 common countries
- **PlainLanguageSummary component** (`PlainLanguageSummary.tsx`) is outstanding — categorized duty breakdown (Duties, Surcharges, Taxes, Fees) with expandable rows showing "Why" and "Legal basis" citations. Proportion bars give intuitive visual weight.
- **DocumentRequirements** appears here too — consistent pattern across surfaces
- Green/yellow/red readiness indicator with clear status message and actionable PGA flag details

**Issues:**

| # | Issue | Severity | File:Line |
|---|-------|----------|-----------|
| S11 | **Destination list is hardcoded to 8 countries** — Missing many common trade corridors (IN, VN, TH, SG, BR). Should ideally be a searchable list of all supported destinations. | P2 | `ShipperProductDetail.tsx:29-38` |
| S12 | **No "Create Order from this product" CTA** — After analyzing a product route, the natural next step is to create an order. There's no direct path to do that. | P2 | `ShipperProductDetail.tsx:150-398` |
| S13 | **Analysis streaming errors show nothing** — `onError` just sets `analyzing=false` and `analysisRun=true` but displays no error message. | P2 | `ShipperProductDetail.tsx:101-104` |
| S14 | **Re-analysis doesn't clear previous results before streaming begins** — If you change source location, old results are visible while new results stream in. Confusing intermediate state. | P3 | `ShipperProductDetail.tsx:82-84` |

**Ideal experience**: After analysis, a prominent "Create Order" CTA appears. The destination field is a searchable country picker. On error, a clear message with retry option is shown.

---

### Screen 4: Order List (`OrderList.tsx`)

**Screenshot**: `shipper/03-order-list.png`

**What works well:**
- Smart status grouping (Draft → Ready to Ship → Shipped)
- Search that covers both order ID and product names
- OrderCard shows duty estimates and compliance action indicators ("CLEAR", "REVIEW")
- "+ New Order" button is prominent in the header
- Count per section ("DRAFT (1)", "READY TO SHIP (2)")

**Issues:**

| # | Issue | Severity | File:Line |
|---|-------|----------|-----------|
| S15 | **Rate 530.0% shown on Ready order** — The order list card for "CNC Aluminum Bracket, CNC Servo Motor" shows "Rate: 530.0%" which appears to be a display bug (likely total duty / single item value instead of effective rate on total). | P1 | Visible in screenshot `shipper/03-order-list.png` |
| S16 | **No pagination** — All orders loaded in one fetch. Will degrade with volume. | P3 | `OrderList.tsx:15-19` |
| S17 | **Empty state text doesn't explain value** — "No orders yet" → Should explain what orders are for: "Orders let you pre-analyze customs costs before shipping." | P4 | `OrderList.tsx:94-105` |

---

### Screen 5: Create Order (`CreateOrder.tsx`)

**Screenshot**: `shipper/04-create-order.png`

**What works well:**
- Source location selection per line item — shows available origins from the product's source locations
- Quantity input with per-unit pricing shown
- Live order summary with running total in amber card
- Hardcoded destination list but covers 10 major countries

**Issues:**

| # | Issue | Severity | File:Line |
|---|-------|----------|-----------|
| S18 | **Product selector is a basic `<select>` dropdown** — For catalogs with >20 products, this is unusable. Should be a searchable combobox showing product details (category, price, origin). | P2 | `CreateOrder.tsx:124-140` |
| S19 | **No quantity validation guidance** — No indication of MOQ, max quantity, or weight/volume implications. | P3 | `CreateOrder.tsx:171-181` |
| S20 | **Currency mismatch not handled** — If products have different currencies (e.g., EUR and JPY), the total shows in USD without conversion. Total is incorrect for mixed-currency orders. | P1 | `CreateOrder.tsx:61-65`, `CreateOrder.tsx:239` |
| S21 | **No estimated duties shown before submission** — The order summary only shows product value. A "Run Quick Estimate" button or inline duty preview would help shippers make informed decisions before committing. | P2 | `CreateOrder.tsx:220-243` |
| S22 | **Error on create is completely silent** — No error toast, no message. | P1 | `CreateOrder.tsx:83-85` |

**Ideal experience**: The create order form shows an inline duty estimate (even approximate) before submission. Product selection uses a searchable combobox. Mixed currencies are either prevented or handled with conversion.

---

### Screen 6: Order Detail (`OrderDetail.tsx`)

**What works well:**
- Clear lifecycle: Draft → Analyze → Ship flow with contextual action buttons
- Inline country editing (click origin/destination to modify)
- **AnalysisOverlay** provides a full-screen immersive view of analysis results
- **AnalysisResultsPanel** shows per-line-item HS codes, duty amounts, and compliance statuses
- **Document readiness check before shipping** — warns about missing documents with modal, allows "Ship Anyway" override (`OrderDetail.tsx:68-85, 342-396`)
- **AttachedDocuments** component lists generated/uploaded documents
- Transit point display when multi-leg routes are present

**Issues:**

| # | Issue | Severity | File:Line |
|---|-------|----------|-----------|
| S23 | **Analysis button is blue, Ship button is amber** — inconsistent with the rest of the app where amber is the primary action color. The blue "Run Analysis" button looks like a secondary action. | P3 | `OrderDetail.tsx:282-293` |
| S24 | **Cannot edit line items after creation** — Once an order is created, you can't add/remove products, change quantities, or swap source locations. The only editable fields are origin/destination country codes. | P2 | `OrderDetail.tsx:87-95` |
| S25 | **Ship button always shows even when analysis is stale** — If you change the origin/destination, the "Ship" button is technically not shown (only `canShip` when analyzed/ready), but the analysis invalidation UX could be clearer. | P3 | `OrderDetail.tsx:124-125` |

---

### Screen 7: Shipment History (`ShipmentHistory.tsx`)

**Screenshot**: `shipper/06-shipment-history.png`

**What works well:**
- Three-tier grouping: Needs Attention (red) → Active (blue) → Completed (green)
- Transport mode icons (Plane/Ship/Truck) with carrier name
- "From Order" badge links shipments back to their source orders
- Duty amount shown per shipment
- Search by product name or shipment ID

**Issues:**

| # | Issue | Severity | File:Line |
|---|-------|----------|-----------|
| S26 | **Hardcoded shipper filter** — `getShipments(undefined, undefined, undefined, "Apex Trading Co.")` filters to a single shipper. This makes sense for a demo but would break with real multi-shipper environments. | P3 | `ShipmentHistory.tsx:49` |
| S27 | **No filter by status, mode, or date range** — Power users tracking many shipments need filtering beyond text search. | P3 | `ShipmentHistory.tsx:42-62` |
| S28 | **COMPLETED (890) with no pagination** — Loading 890 completed shipments in one go will cause performance issues. | P2 | Visible in screenshot `shipper/06-shipment-history.png` |

---

### Screen 8: Shipment Detail (`ShipmentDetail.tsx`)

**What works well:**
- **This is the crown jewel of the shipper surface.** Dense, information-rich screen that packs real intelligence:
  - Pre-clearance screening banner for booked shipments with animated spinner
  - Compliance notice alerts from pre-clearance division
  - Hold/inspection banners with specific hold reasons
  - Editable product description with re-analysis trigger
  - Transport mode badges (Air/Ocean/Ground)
  - Multi-item classification results with per-item HS codes and confidence
  - Full tariff breakdown with effective rate, landed cost, and line items
  - Compliance status with PGA flags and UFLPA risk levels
  - **ShipmentWaypoints** component for multi-leg routing visualization
  - **ShipmentFinancials** with predicted vs actual comparison
  - **ShipmentTimeline** with event history
  - **DocumentRequirements** (same pattern!) with AI generation + upload
  - **AttachedDocuments** for viewing stored docs
  - **ShipmentChat** (floating action button) for contextual AI assistance
- "View Source Order" link when shipment originated from an order

**Issues:**

| # | Issue | Severity | File:Line |
|---|-------|----------|-----------|
| S29 | **Very long vertical scroll** — The screen has ~12 sections stacked vertically. For a held shipment with full analysis, you'd scroll through hold banner, header, analysis button, cargo, classification, tariff, compliance, waypoints, financials, HS codes (duplicate of classification!), attached docs, document requirements, and timeline. Consider tabs or an accordion layout. | P2 | `ShipmentDetail.tsx:137-681` |
| S30 | **Duplicate HS code display** — Classification is shown in the "Analysis Results" section AND again in a separate "Classification" section below. Same data rendered twice. | P2 | `ShipmentDetail.tsx:385-463` vs `ShipmentDetail.tsx:578-632` |
| S31 | **Analyze button always visible even when analysis exists** — Shows "Re-Analyze Shipment" but it's positioned prominently as if it's a required action. Should be demoted to a secondary/text link. | P3 | `ShipmentDetail.tsx:319-337` |
| S32 | **Chat FAB overlaps content at bottom of page** — Fixed position bottom-right can overlap with the last timeline entries or document cards. | P3 | `ShipmentDetail.tsx:676-679` |

---

### Screen 9: Resolution Center (`ResolutionCenter.tsx`)

**Screenshot**: `shipper/08-resolution-center.png`

**What works well:**
- Three-tab design covers main support scenarios: Product Lookup, Document Upload, Stuck Shipment
- Product Lookup uses the real classification API with confidence levels and readiness status
- Document Upload with drag-and-drop, AI field extraction, and issue/suggestion display
- Stuck Shipment tab auto-loads held/inspection shipments and generates AI resolution steps with priority levels

**Issues:**

| # | Issue | Severity | File:Line |
|---|-------|----------|-----------|
| S33 | **Resolution steps fallback is a hardcoded single step** — When AI fails, user gets "Contact customs broker" which provides zero value. Should at least show common troubleshooting steps. | P2 | `ResolutionCenter.tsx:373` |
| S34 | **No link from resolution to actual shipment** — After seeing resolution steps for a stuck shipment, there's no "Go to Shipment" button to take action. | P2 | `ResolutionCenter.tsx:411-438` |
| S35 | **Document upload limited to PDF only** — Real trade documents come in many formats (images/scans, Word docs, Excel). Only PDF accepted. | P3 | `ResolutionCenter.tsx:211` |
| S36 | **Resolution steps have no "execute" action** — Steps show what to do but provide no way to actually do it (e.g., "Upload missing Certificate of Origin" but no upload button inline). | P2 | `ResolutionCenter.tsx:449-475` |

---

### Component Highlight: DocumentRequirements (POSITIVE PATTERN)

**File**: `shipper/components/DocumentRequirements.tsx`

This component deserves special recognition as a **model pattern** that should be replicated across the platform:

**Why it's exceptional:**
1. **Deterministic requirements lookup** — Calls `/api/documents/requirements` which is rule-based (no LLM needed), fast, and reliable (`DocumentRequirements.tsx:90`)
2. **Three action modes per document**: Generate & Attach (AI-powered), Review Fields (preview), Upload (manual), Mark Complete (`DocumentRequirements.tsx:458-513`)
3. **Progress tracking** — Progress bar showing X of Y documents complete with "All Complete" celebration state (`DocumentRequirements.tsx:234-256`)
4. **PDF generation pipeline** — "Generate & Attach" calls the AI-powered document generation endpoint, then auto-generates a PDF via `pdfGenerator`, then uploads it to the backend — all in one click (`DocumentRequirements.tsx:126-166`)
5. **Document viewer modal** for reviewing/editing AI-generated fields before finalizing
6. **Bidirectional sync** — Works with both orders (via `orderId`) and shipments (via `shipmentId`), syncs status back to the parent
7. **Expandable card design** — Each document type collapses to one line, expands to show reason, citation, field count, and action buttons
8. **Required vs Conditional separation** — Required documents shown first, conditional (like Customs Power of Attorney) shown in a separate section
9. **Backend is equally solid** — The `documents.py` backend has full field schemas for 8 document types, deterministic auto-fill with LLM fallback, and proper upsert logic for document storage

**Where this pattern should be replicated:**
- Buyer surface should use this for showing required import documentation
- Broker surface should have a document queue that uses the same status tracking
- Platform Control Tower should aggregate document readiness across all shipments

---

### Component: ShipmentChat (`ShipmentChat.tsx`)

**What works well:**
- Floating action button (FAB) design doesn't interrupt the main content
- Suggested questions ("Why is this shipment being held?", etc.) reduce blank-page syndrome
- Real streaming from `/api/chat/stream` with SSE — not template responses
- Markdown rendering in assistant responses
- Context-aware: passes `shipmentId`, `productId`, and `hsCode` to the backend

**Issues:**

| # | Issue | Severity | File:Line |
|---|-------|----------|-----------|
| S37 | **No conversation persistence** — Messages are component-local state. Closing and reopening the chat loses all history. | P2 | `ShipmentChat.tsx:32` |
| S38 | **Only available on ShipmentDetail** — Not surfaced on OrderDetail, ProductDetail, or Resolution Center where it would be equally useful. | P2 | `ShipmentChat.tsx:29` |
| S39 | **No "action" integration** — Chat can explain what to do but can't execute actions (e.g., "generate the missing Certificate of Origin" → should trigger DocumentRequirements). | P3 | `ShipmentChat.tsx:43-81` |
| S40 | **No loading state in FAB** — While streaming, the FAB doesn't indicate activity. User might open the chat during streaming and see a blank last message. | P4 | `ShipmentChat.tsx:86-97` |

---

## BUYER SURFACE

### Screen 10: Storefront (`StoreFront.tsx`)

**Screenshot**: `buyer/01-storefront.png`

**What works well:**
- Clean e-commerce grid layout with gradient placeholder images
- Hero banner with clear value proposition: "Shop with confidence — All prices include import duties and taxes"
- "Duties & taxes included" badge and "Free delivery" on each card
- Green color scheme clearly differentiates buyer surface from shipper's amber

**Issues:**

| # | Issue | Severity | File:Line |
|---|-------|----------|-----------|
| B1 | **CRITICAL: Entire product catalog is hardcoded** — Products come from `@/data/products.ts`, a static file with 5 products. No API call. No real inventory. | P0 | `StoreFront.tsx:8`, `data/products.ts:32-143` |
| B2 | **No search or category filtering** — Five products is fine, but this surface has zero infrastructure for product discovery. | P3 | `StoreFront.tsx:39-92` |
| B3 | **"No surprise fees" promise is misleading** — The prices ARE the surprise fees, because they're hardcoded estimates, not real tariff engine calculations. | P0 | `StoreFront.tsx:24-26` |
| B4 | **No product images** — Gradient placeholders with a single letter. Functional for demo but makes the surface feel unfinished. | P4 | `StoreFront.tsx:53-56` |

---

### Screen 11: Product Page (`ProductPage.tsx`)

**What works well:**
- Two-column layout (image left, info right) follows e-commerce conventions
- "What's in this price?" expandable breakdown is a great transparency feature
- Add to Cart with satisfying green flash confirmation
- Delivery estimate ("5-7 business days") sets expectations

**Issues:**

| # | Issue | Severity | File:Line |
|---|-------|----------|-----------|
| B5 | **CRITICAL: Duty calculation is hardcoded 5%** — `const taxes = Math.round(product.declaredValue * 0.05)` — this is a flat 5% on declared value regardless of product type, HS code, origin, or destination. The actual tariff engine calculates 38.4% for EV batteries (Section 301 + AD/CVD), 12.5% for aluminum brackets, etc. The buyer sees a fantasy number. | P0 | `ProductPage.tsx:51` |
| B6 | **Shipping estimate is hardcoded $350** — `const shippingEstimate = 350` for every product regardless of weight, mode, or distance. A 500kg tea shipment and a single sweater cost the same to ship. | P1 | `ProductPage.tsx:49` |
| B7 | **Price breakdown doesn't match buyerPrice** — The breakdown shows `productBase + totalDuty + taxes + shipping` but `buyerPrice` is a separate hardcoded number. The math doesn't add up in all cases because buyerPrice was manually set. | P1 | `ProductPage.tsx:49-51`, `data/products.ts:50-51` |
| B8 | **Product data from static import, not API** — Uses `import { getProduct } from "@/data/products"` — no API call to the backend that has real tariff data. | P0 | `ProductPage.tsx:18` |
| B9 | **No quantity selector** — "Add to Cart" adds exactly 1 unit. For B2B products like 500 aluminum brackets, this is wrong. | P2 | `ProductPage.tsx:43-46` |
| B10 | **No origin/destination awareness** — The page doesn't know where the buyer is. Duties change dramatically by destination country, but the buyer surface assumes US. | P1 | `data/products.ts` (all products hardcoded to US destination) |

**Ideal experience**: Product page calls the tariff engine API with the buyer's detected/selected country, gets real duty calculations, and displays an accurate, itemized breakdown. The buyerPrice is computed dynamically as `declaredValue + realDuty + realTaxes + realShipping`.

---

### Screen 12: Checkout (`CheckoutPrice.tsx`)

**Screenshot**: `buyer/03-checkout-pricing.png`

**What works well:**
- Standard two-column checkout layout (shipping/payment left, order summary right)
- "Duties & taxes included" badge reinforces the value proposition
- **Purchase creates real pipeline entries** — `handlePurchase()` calls `analyze()` for each cart item, creating actual entries in the pipeline that appear in the Control Tower (`CheckoutPrice.tsx:24-42`)
- Order confirmation tells user to check Platform surface — good cross-surface guidance

**Issues:**

| # | Issue | Severity | File:Line |
|---|-------|----------|-----------|
| B11 | **Decorative checkout form** — Shipping address and payment fields are completely fake. Hardcoded to "Sarah Chen" at "1234 Oak Street". Not functional. | P2 | `CheckoutPrice.tsx:116-138` |
| B12 | **No duty itemization in checkout summary** — Order summary shows subtotal, shipping (Free), and "Duties & taxes included" badge but no actual duty breakdown per item. The buyer can't see what they're paying for duties. | P2 | `CheckoutPrice.tsx:174-193` |
| B13 | **Cart total uses hardcoded buyerPrice** — `getTotal()` sums the static `buyerPrice` values from the data file. Zero connection to the tariff engine. | P0 | `CheckoutPrice.tsx:22` |
| B14 | **No destination country selection** — The checkout doesn't ask where to ship. Uses hardcoded US destination from product data. | P1 | `CheckoutPrice.tsx:24-42` |
| B15 | **Pipeline analysis at purchase time is wasteful** — The tariff analysis runs at checkout completion, not at browse/cart time. This means the buyer has already "paid" a price that wasn't informed by real analysis. The analysis should happen before pricing, not after. | P1 | `CheckoutPrice.tsx:24-42` |

---

## CROSS-SURFACE ANALYSIS

### Flow Analysis: Can a shipper complete the full order-to-shipment flow?

**Verdict: Yes, with minor friction.**

1. Products → Add Product (with AI description quality) ✅
2. Products → Product Detail → Analyze Route ✅ (auto-analyzes)
3. Orders → Create Order (from product catalog) ✅
4. Order Detail → Run Analysis (tariff + compliance) ✅
5. Order Detail → Document Requirements → Generate/Upload docs ✅
6. Order Detail → Ship Order (with doc readiness check) ✅
7. Shipment History → Shipment Detail → Full tracking ✅
8. Resolution Center → Stuck shipment triage ✅

**Missing link**: No direct path from Product Detail → Create Order. User must navigate away and manually select the same product.

### Does DocumentRequirements pattern get replicated?

**Verdict: Partially.** Used in ShipperProductDetail and ShipmentDetail. NOT used in:
- OrderDetail (has AnalysisResultsPanel instead, which has its own doc status UI)
- Buyer surface (no document awareness at all)
- Resolution Center (has document upload but different pattern)

### Is the chat helpful or basic Q&A?

**Verdict: Real AI streaming, context-aware, not template.** The `streamChat` API call passes shipment context (ID, HS code). It's genuinely useful but limited to ShipmentDetail only.

### Does the buyer see real duty calculations?

**Verdict: No.** The buyer surface is completely disconnected from the tariff engine:
- Storefront: Static prices from `data/products.ts`
- ProductPage: Hardcoded 5% tax rate (`ProductPage.tsx:51`)
- Checkout: Sums static buyerPrice values
- Analysis only runs AFTER purchase (`CheckoutPrice.tsx:24-42`)

### Are there proactive suggestions?

**Verdict: Yes, in shipper surface; No, in buyer surface.**
- Shipper gets: AI description quality with follow-up questions, AI document generation, AI resolution steps, compliance PGA flags
- Buyer gets: Zero proactive intelligence

---

## PRIORITIZED FINDINGS

### P0 — Must Fix (Fundamentally Broken)

| ID | Finding | Surface | Impact |
|----|---------|---------|--------|
| B1 | Buyer product catalog is hardcoded (no API) | Buyer | Buyer surface cannot scale past 5 demo products |
| B3 | "No surprise fees" promise is false — prices are static | Buyer | Trust violation — core value proposition is a lie |
| B5 | Duty calculation is hardcoded 5% — real rates are 8-38%+ | Buyer | Buyers see wildly inaccurate pricing |
| B8 | Product data from static import, not tariff engine API | Buyer | Buyer surface has zero connection to the real clearance engine |
| B13 | Cart total uses hardcoded prices, not computed values | Buyer | All financial figures in checkout are fiction |

### P1 — High Priority (Significant UX Impact)

| ID | Finding | Surface | Impact |
|----|---------|---------|--------|
| S2 | Network errors silently swallowed across all screens | Shipper | User sees blank/frozen UI on failure with no recovery path |
| S7 | Country code is free-text, not validated picker | Shipper | Typos silently break routing analysis |
| S8 | Order creation errors show nothing | Shipper | User can't recover from server errors |
| S15 | 530% rate displayed on order card | Shipper | Display bug undermines data credibility |
| S20 | Mixed-currency orders show incorrect totals | Shipper | Financial data is wrong for multi-currency orders |
| S22 | Create order error is silent | Shipper | Same as S8 |
| B6 | Shipping cost hardcoded to $350 | Buyer | Pricing fiction extends to logistics costs |
| B7 | Price breakdown math doesn't add up | Buyer | Breakdown components don't sum to displayed total |
| B10 | No destination country selection for buyer | Buyer | Duty estimates are meaningless without knowing destination |
| B14 | Checkout has no destination field | Buyer | Shipped "nowhere" — analysis uses hardcoded US |
| B15 | Tariff analysis runs after purchase, not before | Buyer | The analysis that should inform pricing happens too late |

### P2 — Medium Priority (Meaningful Improvement)

| ID | Finding | Surface | Impact |
|----|---------|---------|--------|
| S1 | Add Product CTA below the fold | Shipper | Primary action hidden |
| S5 | No readiness indicator on catalog cards | Shipper | Can't scan for products needing attention |
| S6 | No inline validation in Add Product form | Shipper | Garbage-in, garbage-out for classification |
| S11 | Only 8 destination countries available | Shipper | Limits real-world utility |
| S12 | No "Create Order" from Product Detail | Shipper | Broken workflow continuity |
| S18 | Basic `<select>` for product selection | Shipper | Unusable for larger catalogs |
| S21 | No duty estimate on Create Order page | Shipper | Orders created without cost visibility |
| S24 | Cannot edit order line items after creation | Shipper | Must delete and recreate to fix mistakes |
| S28 | 890 completed shipments loaded without pagination | Shipper | Performance cliff |
| S29 | ShipmentDetail has 12+ sections in one scroll | Shipper | Information overload, hard to find specific data |
| S30 | Duplicate HS code display in ShipmentDetail | Shipper | Confusing redundancy |
| S33 | Resolution steps fallback is useless | Shipper | Fallback provides zero value |
| S34 | No link from resolution to actual shipment | Shipper | Can't take action on suggestions |
| S36 | Resolution steps are read-only, not actionable | Shipper | Shows what to do but can't execute |
| S37 | Chat has no conversation persistence | Shipper | Context lost on close |
| S38 | Chat only on ShipmentDetail | Shipper | Useful tool locked to one screen |
| B9 | No quantity selector on product page | Buyer | Can't order bulk quantities |
| B11 | Decorative checkout form (not functional) | Buyer | Feels like a prototype, not a product |
| B12 | No duty itemization in checkout | Buyer | Buyer can't see duty breakdown |

### P3 — Low Priority (Nice to Have)

| ID | Finding | Surface | Impact |
|----|---------|---------|--------|
| S3 | No search on product catalog | Shipper | Minor until catalog grows |
| S9 | Only 5 currencies supported | Shipper | Missing common trade currencies |
| S10 | Description quality errors silently swallowed | Shipper | Minor — feature degrades gracefully |
| S13 | Analysis streaming errors show nothing | Shipper | User confused but can retry |
| S14 | Old results visible during re-analysis | Shipper | Confusing but not harmful |
| S16 | No order list pagination | Shipper | Performance concern at scale |
| S19 | No quantity guidance | Shipper | Missing context |
| S23 | Blue analyze button doesn't match amber theme | Shipper | Visual inconsistency |
| S25 | Stale analysis UX could be clearer | Shipper | Minor confusion |
| S26 | Hardcoded shipper filter | Shipper | Demo limitation |
| S27 | No shipment filters beyond search | Shipper | Power user limitation |
| S31 | Re-analyze button too prominent | Shipper | Visual weight issue |
| S32 | Chat FAB overlaps content | Shipper | Layout issue |
| S35 | Document upload limited to PDF | Shipper | Real docs come in many formats |
| S39 | Chat can't execute actions | Shipper | Feature enhancement |
| B2 | No search/filter on storefront | Buyer | Minor with 5 products |

### P4 — Cosmetic

| ID | Finding | Surface | Impact |
|----|---------|---------|--------|
| S4 | No product count in catalog header | Shipper | Orientation detail |
| S17 | Empty order state text isn't helpful | Shipper | Onboarding detail |
| S40 | No activity indicator on chat FAB | Shipper | Polish item |
| B4 | Gradient placeholders instead of product images | Buyer | Visual polish |

---

## STRATEGIC RECOMMENDATIONS

### 1. Connect Buyer Surface to Tariff Engine (P0)
The highest-leverage change is replacing hardcoded buyer prices with real tariff engine calls. The backend already has everything needed: `/api/analyze/stream` returns classification, tariff breakdown, and compliance screening. The buyer surface just needs to call it.

### 2. Replicate DocumentRequirements Pattern
This component demonstrates what an AI-native clearance workflow looks like. The pattern (deterministic requirements → AI-generated fields → PDF generation → upload) should be the template for every document interaction in the platform.

### 3. Add Global Error Handling
Nearly every shipper screen silently swallows errors. A global toast/notification system would fix S2, S8, S10, S13, S22 in one shot.

### 4. Country Code Standardization
Replace free-text country inputs with ISO-2 validated dropdown/combobox across all surfaces. Addresses S7, S11, B10, B14.

### 5. Promote ShipmentChat to Global Assistant
The chat component works well but is locked to one screen. Making it available across the entire shipper surface (and eventually buyer surface) would multiply its value.
