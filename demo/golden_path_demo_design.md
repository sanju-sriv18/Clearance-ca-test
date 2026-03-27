# The Golden Path: A Working Clearance Intelligence System

## Scoped Demo with Real Analytical Capability

---

> **What this is:** A working system that actually analyzes products, computes duties, screens compliance, models regulatory change, and resolves exceptions — live, against real trade data, across the major global trade lanes. A stakeholder can describe their product, name their origin and destination, and watch the system think. It is not playing back pre-recorded answers. It is reasoning about their input.

> **What this is not:** A full platform. The scope is tiered — US import at full depth, EU and China import at meaningful analytical depth, other jurisdictions at framework level. Within scope, it works. Outside scope, it says so. The honesty about boundaries is itself a feature: it shows the team knows where the hard problems are.

> **The shift from theater to proof:** The previous design said "we are not proving technology works." This design says the opposite. We prove that the core analytical problems — classification, duty stacking, compliance screening, exception analysis — are solvable with current AI and structured trade data. A stakeholder who walks in skeptical walks out having seen the system correctly analyze *their* product.

---

## Table of Contents

1. [The System: What Actually Works](#the-system)
2. [Scope Definition: What's Live vs. What's Scaffolded](#scope)
3. [The Six Engines](#engines)
4. [The Golden Path Experience](#golden-path)
   - [Opening: The Burning Platform](#opening)
   - [Scene 1: Describe a Product, Get a Clearance Plan](#scene-1)
   - [Scene 2: The Compliance Dashboard](#scene-2)
   - [Scene 3: Trade Lane Comparison and Consumer Landed Cost](#scene-3)
   - [Scene 4: Regulatory Change Intelligence](#scene-4)
   - [Scene 5: Exception Resolution](#scene-5)
   - [Scene 6: Bring Your Own Product](#scene-6)
5. [Design Principles](#design-principles)
6. [Data Infrastructure Required](#data-infrastructure)
7. [What the System Can Be Wrong About (and Why That's Good)](#wrong)
8. [Visual and Interaction Design](#design)
9. [Presenter Strategy](#presenter)

---

<a name="the-system"></a>
## The System: What Actually Works

This is not a demo in the traditional sense. It is a scoped, working clearance intelligence system. Six analytical engines operate against real trade data across the major global trade lanes. The golden path narrative (Maria, David, the consumer, the exception, the regulatory shift) still exists, but it runs *through* the system — the system produces the answers live, not from a script.

The critical difference: **when a stakeholder says "try my product," the system actually tries it.** It classifies the product. It computes the tariff stack for any supported destination. It identifies compliance requirements. It screens restricted parties. It models the impact of pending regulatory changes. It does this in real time, using the same logic it used for the scripted examples.

This transforms the demo from a performance into a conversation. The audience stops watching and starts interacting. The question shifts from "do I believe this future is worth building?" to "how far along are you already?"

**The global dimension is not a feature — it's the architecture.** The same engines, the same analytical approach, the same tax regime patterns work across jurisdictions. Adding a country is a knowledge-loading exercise, not an engine-building exercise. The demo proves this by computing landed costs for the same product entering different markets.

**The magic is in the transparency.** The system presents three fundamentally different application surfaces — the Platform Intelligence command center for compliance teams and analysts, the Shipper experience embedded in logistics workflows, and the Buyer experience embedded in checkout. The same six engines power all three. The demo's most powerful moments come from switching between surfaces: the audience watches a screen full of analytical complexity collapse into "Ready to ship, $56.81" or a single checkout price. The complexity is absorbed. The experience is simple. That compression — from intelligence to simplicity — is the product.

---

<a name="scope"></a>
## Scope Definition: What's Live vs. What's Scaffolded

Intellectual honesty about scope is what separates this from vaporware. The system must be explicit about what it can do and what it cannot do yet. This honesty builds credibility rather than undermining it.

### Global Scope Tiers

The system operates at three tiers of depth, reflecting the phased investment in knowledge infrastructure:

| Tier | Jurisdictions | What Works |
|---|---|---|
| **Tier 1 — Full Depth** | US import | All 6 engines at full analytical capability. Complete tariff stack, compliance screening, FTA qualification, exception analysis, regulatory change intelligence. |
| **Tier 2 — Meaningful Analytical Capability** | EU import, China import | Classification (HS 6-digit global + major national subheadings). Tariff computation (MFN + surcharges + trade remedies + VAT/import taxes). Major compliance frameworks (CE/REACH for EU, CCC for China). Regulatory change scenario modeling. |
| **Tier 3 — Framework Demonstration** | Brazil, India, others | Classification (HS 6-digit). MFN rate lookup. Tax structure awareness (including Brazil's cascading tax pattern). Narrative-level compliance. Enough to demonstrate global scalability. |

### What's Live (Real Analysis, Real Data, Real Reasoning)

| Capability | What It Does | Jurisdiction Scope |
|---|---|---|
| **Product Classification** | Takes a product description and determines the HS code with reasoning chain. Classifies to HS 6-digit (globally valid) then extends to national subheadings. | All tiers — HS 6-digit is universal. National extension at Tier 1 (HTSUS 10-digit) and Tier 2 (CN 8-digit for EU, China 8-digit). |
| **Global Tariff & Landed Cost** | Given HS code + origin + destination + declared value, computes the full tariff stack and landed cost using the destination country's tax regime template. | Tier 1: full stack (MFN + 301 + 232 + IEEPA + AD/CVD + MPF + HMF + state tax). Tier 2: full stack (EU: MFN + trade remedies + VAT. China: MFN + retaliatory + consumption tax + VAT). Tier 3: MFN + primary tax. |
| **Compliance Screening** | Maps HS codes to regulatory requirements (PGA for US, CE/REACH for EU, CCC for China). Screens entities against restricted party lists. | Tier 1: full PGA mapping + UFLPA. Tier 2: major frameworks. |
| **Denied Party Screening** | Given entity name, screens against restricted party lists with fuzzy matching. | All tiers — lists are global (OFAC SDN, BIS Entity List, UFLPA Entity List, EU sanctions lists). |
| **Regulatory Program Matching** | Given HS code + origin + destination, identifies applicable special tariff programs and surcharges. | Per-jurisdiction: Section 301/232/IEEPA for US, trade remedies for EU, retaliatory tariffs for China. |
| **Regulatory Change Intelligence** | Monitors pending and enacted regulatory changes. Models scenario impact on portfolio. | US, EU, China signals. Cross-jurisdiction scenario modeling. |
| **Trade Lane Comparison** | Same product, same origin, multiple destinations — side-by-side landed cost computed by different tax regime templates. | All loaded jurisdictions. |

### What's Scaffolded (Demonstrated with Realistic Logic but Bounded Data)

| Capability | What It Shows | What's Behind It |
|---|---|---|
| **FTA Qualification Analysis** | For USMCA (and optionally one EU FTA): given HS code + origin + simplified BOM, assesses whether product meets rules of origin | USMCA rules of origin for ~50 product categories. EU FTA at framework level. Not the full annex — but enough to demonstrate the analytical approach. |
| **Supply Chain Risk Mapping** | Shows tier-2 supplier risk when a listed entity appears in the supply chain | Pre-structured supply chain trees for 3-4 fictional companies. The *screening* against restricted party lists is live; the *supply chain data* is illustrative. |
| **Product Intelligence Memory** | Shows the system remembering prior shipments of the same product | A pre-loaded shipment history for the demo personas. Demonstrates the concept of institutional memory without requiring a real shipment database. |
| **Exception Response Generation** | Given a customs hold type and product details, generates an evidence package and draft response | LLM-powered reasoning against CBP CROSS rulings database. Works within the scope of common hold types (classification questions, PGA data gaps, valuation questions). US-focused for the demo; the analytical approach extends to other jurisdictions. |

### What the System Explicitly Cannot Do (and Says So)

- Classification for products requiring physical testing (e.g., tensile strength for textiles, alcohol content for beverages)
- AD/CVD rate calculation beyond confirming an active order exists (rates require case-specific manufacturer analysis)
- Real-time customs filing in any jurisdiction (the system computes what *would* be filed, not actually files)
- Supply chain traceability beyond tier-1 (this is honest about the fungibility problem documented in the knowledge base)
- Full compliance screening for Tier 3 jurisdictions (framework only — the system says what it covers and what it doesn't)
- Guaranteed accuracy of pending regulatory signals (the system distinguishes fuzzy/discussed from confirmed/enacted)

**The presenter's line when something is outside scope:** "That's outside what we've built for this scope. Here's what it would take to cover it — and here's why we're confident the same analytical approach extends. In fact, let me show you — [switches destination country] — the same architecture already works for [EU/China/etc.]."

---

<a name="engines"></a>
## The Six Engines

Each engine is a distinct analytical capability. They operate independently but compose together across jurisdictions. The golden path shows them in combination; Scene 6 ("Bring Your Own Product") lets the audience use them individually. Every engine is jurisdiction-aware — the same classification engine works globally because HS 6-digit is universal; the same tariff engine works globally because tax regime templates are configurable per destination.

### Engine 1: Classification Intelligence

**What it does:** Takes a natural-language product description and determines the HTSUS classification.

**How it actually works:**
- The full HTSUS schedule (Chapters 1-99, including General Notes, Section Notes, Chapter Notes, and legal text) is structured as the system's knowledge base
- An LLM receives the product description and reasons through the classification hierarchy: which Section, which Chapter, which Heading, which Subheading, which statistical suffix
- The reasoning chain is visible — not just the answer, but *why* that answer
- For each classification, the system shows:
  - The GRI (General Rules of Interpretation) it applied
  - The Chapter Notes or Section Notes that guided the decision
  - Similar products and how they were classified (from a reference set of common examples)
  - A confidence assessment based on specificity of the description

**What makes it real vs. a lookup table:**
- A lookup table maps "ceramic mug" → 6912.00.48. It fails on "handmade glazed drinking vessel made from fired clay." The classification engine handles both because it reasons about materials, function, and construction — the same way a human classifier does.
- The engine can handle ambiguity: "Is this a mug or a vase?" becomes a visible question the system flags rather than silently guessing
- The engine can be wrong and explain *why* it might be wrong — "If this product contains more than 50% porcelain by weight, the classification shifts from 6912 to 6911. The description does not specify material composition."

**Interaction in the demo:**
- The user types or speaks a product description
- The system shows its reasoning chain in real time — section selection, chapter selection, heading selection, subheading determination
- The final classification appears with a confidence score and the reasoning basis
- Clicking any step in the chain shows the HTSUS text it referenced

**Scope limitation:** Works across all 99 HTSUS chapters. Classification quality is highest for consumer goods, electronics, auto parts, textiles, food products, and chemicals — the categories with the most structured training examples. Exotic products (e.g., "radioactive isotopes for medical imaging") will get a reasonable chapter-level classification but may not nail the 10-digit code.

### Engine 2: Global Tariff & Landed Cost Engine

**What it does:** Given an HS code, country of origin, destination country, and declared value, computes the total effective duty rate and landed cost by applying the destination country's tax regime template.

**The architectural insight: Tax Regime Templates.** Every destination country has a tax regime template that defines the ordered sequence of tax layers, their rate sources, their base computation formulas, and their stacking behavior. The engine doesn't have hardcoded US logic or EU logic — it dispatches to the appropriate template. Two patterns cover the world:
- **Pattern A (Additive):** Each tax layer computes independently on a defined base. US, EU, China, India, and most of the world follow this pattern.
- **Pattern B (Self-referencing/Cascading):** Some taxes include themselves in their own base, requiring algebraic grossup. Brazil's ICMS is the canonical example.

**How it works for US imports (Tier 1 — full depth):**
- MFN rate: looked up from the HTSUS General Rate of Duty column
- Section 301: cross-referenced against the published 301 product lists (Lists 1-4, with sub-lists 4a/4b) and their rate schedules
- Section 232: checked against the product scope (steel and aluminum products defined by Chapter 72, 73, and 76 HS codes) and country exemptions
- IEEPA Reciprocal: matched against the country-specific rate schedule published under the IEEPA executive orders
- AD/CVD: checked against the ITC's active antidumping and countervailing duty orders database. Shows range of rates with all-others rate as default.
- FTA preferences: for USMCA, flagged as "available if rules of origin are met"
- Entry type determination: formal (>$2,500, MPF 0.3464%) vs. informal (≤$2,500, MPF $2.69 flat)
- State/use tax: applied per destination state from K13

**How it works for EU imports (Tier 2):**
- MFN rate: looked up from TARIC Combined Nomenclature
- EU trade remedies: anti-dumping duties, countervailing duties, safeguard measures from TARIC additional duties
- VAT: computed on (CIF value + all duties) at the member state's standard or reduced rate
- No entry type complexity (EU uses a single declaration process)

**How it works for China imports (Tier 2):**
- MFN rate: from China Customs Tariff Schedule
- Retaliatory tariffs: China's measures on US/EU goods from MOFCOM
- Consumption tax: applicable for specific product categories
- VAT: computed on (CIF + duty + consumption tax) at 13% standard or 9% reduced

**What makes it real:**
- This is not 10 pre-loaded HS codes. The tariff data covers full schedules for loaded jurisdictions. Every rate is sourced.
- The same product analyzed for different destinations produces different stacks from different templates — demonstrating the architecture.
- Rate stacking math is transparent — every program is shown, every rate is sourced, the arithmetic is visible
- The system handles edge cases that trip up humans: a product on Section 301 List 3 increased to 25%, plus IEEPA at 60%, for a total of 85% on top of the MFN rate

**Interaction in the demo:**
- HS code is either entered directly or populated from Engine 1's classification
- Country of origin and destination country selected from dropdowns
- Declared value entered
- The tariff stack builds visibly: each program checks and either applies or shows "N/A" with reason
- The total effective rate and dollar amount appear at the bottom
- Changing destination country recomputes through a different tax regime template

### Engine 3: Compliance Screening

**What it does:** Given product details (HS code, origin, destination, entities involved), screens for regulatory requirements and restricted parties.

**Three sub-capabilities:**

**3A: PGA Jurisdiction.** Maps HS codes to Partner Government Agencies. For example:
- HS 3304 (cosmetics) → FDA
- HS 9503 (toys) → CPSC
- HS 0802 (nuts) → USDA/APHIS
- HS 2208 (spirits) → TTB
- HS 8541 (semiconductor devices) → Commerce/BIS export control

For each PGA hit, the system shows what data is required: FDA Prior Notice, CPSC certificate of compliance, USDA phytosanitary certificate, etc.

**3B: Denied Party Screening.** Takes an entity name and screens against:
- OFAC Specially Designated Nationals (SDN) list
- OFAC Consolidated Sanctions List
- BIS Entity List
- BIS Denied Persons List
- UFLPA Entity List

Uses fuzzy name matching (Levenshtein distance, transliteration variants for CJK names, known aliases). Shows match confidence and which list triggered the match.

**3C: UFLPA Risk Assessment.** For China-origin goods, cross-references product type against UFLPA enforcement priorities (cotton/tomatoes/polysilicon/PVC) and checks entities against the UFLPA Entity List. Shows risk level and what documentation would be required.

**What makes it real:**
- The restricted party lists are real, publicly available, and current. The screening produces real results.
- The PGA mapping covers the most common HS-to-agency relationships. A stakeholder who imports food products will see the correct FDA requirements; one who imports children's products will see the correct CPSC requirements.
- The UFLPA screening uses the actual entity list maintained by DHS.

### Engine 4: FTA Qualification (USMCA Scope)

**What it does:** For products originating in Mexico or Canada, assesses whether USMCA preferential rates are available and what rules of origin must be met.

**How it actually works:**
- Given an HS code, looks up the USMCA product-specific rule of origin (tariff shift rule, regional value content requirement, or both)
- If a simplified BOM is provided (components with HS codes and origins), evaluates whether the tariff shift test is met
- Computes RVC using the transaction value method or net cost method
- Shows the result: qualifies, does not qualify, or "may qualify pending additional data"

**Scope:** Pre-loaded rules of origin for approximately 50 product categories spanning auto parts (Chapter 87), electronics (Chapter 85), textiles (Chapters 50-63), food/agriculture (Chapters 1-24), plastics (Chapter 39), and steel/metals (Chapters 72-73). Enough to demonstrate the analytical approach and handle the most common stakeholder products.

### Engine 5: Exception Analysis

**What it does:** Given a customs hold scenario (hold type, product details, CBP question), generates an evidence package and draft response.

**How it actually works:**
- Classifies the hold type (classification question, PGA data gap, valuation question, UFLPA detention)
- For classification questions: searches the CBP CROSS rulings database for relevant precedents, assembles evidence from the product description, images, and prior entry history
- For PGA holds: identifies the specific data gap and generates the required data elements
- For UFLPA detentions: outlines the evidence required under the UFLPA enforcement strategy and maps available documentation against requirements
- Generates a draft response with sourced evidence

**What makes it real:**
- The CROSS rulings database is publicly available from CBP (rulings.cbp.gov). The system can search and cite real rulings.
- The response generation uses LLM reasoning against the actual regulatory framework — it's producing real analysis, not templated text
- The system shows its confidence and flags where human review is most needed

### Engine 6: Regulatory Change Intelligence

**What it does:** Monitors, classifies, and models the impact of regulatory changes across jurisdictions. This is the engine that addresses the anxiety of rapid change — the most acute pain point in today's trade environment.

**Three modes of operation:**

**6A: Signal Intelligence.** The system maintains awareness of pending and enacted regulatory changes. Every signal has a status:
- **Fuzzy** — discussed, proposed, rumored. Source: news reports, government statements, draft proposals. The system flags these but never presents them as obligations.
- **Confirmed** — enacted, published in official source, has an effective date. Source: Federal Register (US), Official Journal of the EU, MOFCOM announcements (China).

**6B: Scenario Modeling.** Given a pending signal (fuzzy or confirmed), the system recomputes landed cost for affected products and trade lanes. "If the proposed 100% tariff on EU steel is enacted, here's the impact on your portfolio." This lets users prepare for changes that haven't happened yet.

**6C: Automated Impact Analysis.** When a signal transitions from fuzzy to confirmed, the system automatically identifies affected SKUs and trade lanes in the portfolio, quantifies the cost impact, and generates a summary. This is the "it happened — here's what changed" mode.

**What makes it real:**
- The signals are sourced from real regulatory announcements, with attribution and dates
- The scenario modeling uses the same tariff engine (Engine 2) that computed the current costs — it's not a separate estimate, it's a recomputation with modified parameters
- The system distinguishes clearly between "proposed" and "enacted" — it never presents a scenario as a current obligation
- Cross-jurisdiction modeling is the unique power: "China retaliates on US goods → here's the impact on your China-bound shipments AND here's how the same goods entering the EU are unaffected"

**Interaction in the demo:**
- The Regulatory Change Intelligence screen shows a timeline of active signals across US, EU, and China
- The presenter selects a signal and clicks "Model Impact"
- The system recomputes affected trade lanes and shows current vs. projected side-by-side
- For a confirmed change with a future effective date, the system shows a countdown and the portfolio impact

**Why this engine matters for the pitch:** Rate volatility is the most visceral pain point. Every person in the room has been blindsided by a tariff change. Engine 6 transforms that anxiety into preparedness. It's the engine that turns the platform from "a better calculator" into "a strategic advantage."

---

<a name="golden-path"></a>
## The Golden Path Experience

The golden path is still a narrative — Maria, David, the consumer, the regulatory shift, the exception. But now each scene is *produced by the system*, not pre-rendered. The engines run live. The data populates in real time. The presenter doesn't advance through a script — they operate the system.

The experience culminates in Scene 6: the audience uses the system themselves.

**Critical design principle: Resequenceability.** Every scene is independently accessible. The presenter can navigate to any scene from any other scene. The golden path below is the *recommended* order, but if an audience member asks "can you show me the impact of a tariff change?" in the middle of Scene 1, the presenter can jump to Scene 4, show it, and return. The system doesn't care about scene order — every engine is independently callable. This is not a presentation tool — it's a working analytical system with a recommended tour.

**Total experience: 55-70 minutes plus Q&A**

```
Opening (5 min)     → DISCOMFORT    "This is broken, getting worse, and changing faster than anyone can track."
Scene 1 (10 min)    → CLARITY       "Watch the system analyze a real product for any destination."
Scene 2 (10 min)    → STRATEGY      "This is compliance that creates value — across trade lanes."
Scene 3 (8 min)     → COMPARISON    "Same product, three destinations. Three different cost structures."
Scene 4 (8 min)     → PREPAREDNESS  "A tariff change is coming. The system already knows."
Scene 5 (7 min)     → CONFIDENCE    "Even when things go wrong, the system knows what to do."
Scene 6 (12 min)    → CONVICTION    "It just analyzed YOUR product, for YOUR market."
Q&A (15+ min)       → COMMITMENT    "What would you need to see next?"
```

---

<a name="opening"></a>
### Opening: The Burning Platform (5 Minutes)

**Five full-screen data cards — updated for global scope and regulatory velocity:**

1. **4,000,000** parcels/day now requiring formal entry (US de minimis elimination, August 2025)
2. **Tariff stacking:** consumer electronics from China → US, 73.5% effective rate. Same product → EU, different stack entirely. Same product → back into China from the US, retaliatory tariffs apply. Three markets. Three tax regimes. One product.
3. **Velocity of change:** A scrolling timeline of tariff actions in the past 12 months — US IEEPA modifications, EU trade remedy reviews, China retaliatory measures. Each one affects thousands of SKUs. Each one requires recalculation. The timeline should feel relentless.
4. **47 open holds** on one compliance officer's dashboard. 30-90+ days to resolve a UFLPA detention.
5. **"This is global trade in 2026. The rules change faster than any team can track. Here's what replaces the spreadsheet."**

Cards 1-4 are static. They set the emotional context — not just the *magnitude* of the problem (how many parcels, how high the rates) but the *velocity* (how fast things change, how many jurisdictions are moving simultaneously). Card 3 is new and critical: it makes the audience feel the pace of change before the system shows them the antidote.

After Card 5, the presenter says: **"Everything you're about to see is a working system. It operates across the US, the EU, and China — the trade lanes that matter most right now. It's scoped, and we'll be honest about what it covers and what it doesn't. But within that scope, it's real. It's analyzing real trade data from real tariff schedules. And when we get to the end, I'm going to ask you to give it something to analyze — any product, any market."**

That last line reframes the entire experience. The audience is now watching with a purpose: they're evaluating a system they will use themselves. And the "any market" addition signals that this is not a US-only tool.

---

<a name="scene-1"></a>
### Scene 1: Describe a Product, Get a Clearance Plan (10 Minutes)

**Emotional target:** "The system genuinely understands products and trade rules. This isn't a lookup table."

**The screen:** A clean interface with a single large text input area on the left ("Describe your product") and a results panel on the right ("Clearance Intelligence"). Below the text input: dropdowns for Country of Origin and Destination Country. A customs value field. A "Calculate" button.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  ✦ Clearance Intelligence Engine                                        │
├────────────────────────────┬─────────────────────────────────────────────┤
│                            │                                             │
│  Describe the product      │                                             │
│  ┌──────────────────────┐  │     [Results appear here after analysis]    │
│  │                      │  │                                             │
│  │  [text input area]   │  │                                             │
│  │                      │  │                                             │
│  └──────────────────────┘  │                                             │
│                            │                                             │
│  Country of Origin:          │                                             │
│    [Portugal (PT) ▼]         │                                             │
│  Destination: [US ▼]         │                                             │
│  Customs Value: [$38.00]     │                                             │
│                            │                                             │
│  [Analyze →]               │                                             │
│                            │                                             │
└────────────────────────────┴─────────────────────────────────────────────┘
```

**The narrative:** The presenter sets up Maria's story as before — ceramics studio in Lisbon, sells on Shopify, never heard of the Harmonized Tariff Schedule. But then, instead of showing a pre-filled form:

**Presenter types the product description live:**
"Handcrafted ceramic mug, glazed stoneware, 450ml, food-safe, made in Lisbon, Portugal"

Selects Portugal as country of origin. **US** as destination. $38.00 customs value. Clicks Analyze.

**What the system does (visible to the audience):**

The right panel comes alive. Not a loading spinner — a visible reasoning chain:

```
┌─────────────────────────────────────────────────────────────┐
│  ✦ Classification Analysis                          [E1→K1] │
│                                                              │
│  Parsing description...                                      │
│  Material: ceramic, stoneware, glazed                        │
│  Function: drinking vessel (mug), tableware                  │
│  Construction: handcrafted                                   │
│                                                              │
│  ► Section XIII: Articles of stone, plaster, cement,         │
│    ceramic, glass                                            │
│  ► Chapter 69: Ceramic products                              │
│    Chapter Note 1: This chapter covers ceramic articles      │
│    obtained by firing...                                     │
│  ► Heading 6912: Ceramic tableware, kitchenware, other       │
│    household articles, and toilet articles, other than of    │
│    porcelain or china                                        │
│  ► Subheading 6912.00: [single subheading]                   │
│  ► Statistical suffix: .48 (other)                           │
│                                                              │
│  CLASSIFICATION: HTSUS 6912.00.4800                          │
│  Confidence: High                                            │
│  Basis: Material (stoneware/ceramic) + function (tableware)  │
│         + not porcelain → 6912 per GRI 1                     │
│                                                              │
│  ⚠ Note: If glaze contains lead above FDA action levels,    │
│    FDA may test under CPG Sec. 545.450.                      │
│    Description states "food-safe" — assumed compliant.       │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  ✦ Tariff Stack                                     [E2→K2] │
│                                                              │
│  MFN Duty (General):     9.8% ad valorem                     │
│    Source: HTSUS 6912.00.48, General Rate column             │
│  Section 301:            N/A — Portugal is not China          │
│  Section 232:            N/A — ceramic, not steel/aluminum    │
│  IEEPA Reciprocal:       [current EU rate, sourced from K5]  │
│  AD/CVD:                 No active orders for 6912/Portugal   │
│  FTA:                    No US-EU FTA in effect               │
│                                                              │
│  Effective Duty Rate:    [computed sum of applicable rates]   │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  ✦ Landed Cost                              [E2→K2,K13,K16] │
│                                                              │
│  Entry Type:             Informal (Type 11) — value ≤ $2,500 │
│  Customs Value:          $38.00                [user input]   │
│  Duty (9.8%):            $3.72              [E2: rate × val]  │
│  MPF:                    $2.69      [K16: informal flat fee]  │
│  HMF:                    N/A          [air, not ocean cargo]  │
│  Estimated Shipping:     $12.40          [estimate — stub]    │
│  IL State/Use Tax:       [K13: rate × customs value]          │
│  ────────────────────────────────────                         │
│  Import Landed Cost:     [computed sum, excl. state tax]      │
│  Total with State Tax:   [computed sum, incl. state tax]      │
│  Note: State/use tax is a domestic obligation, not an import  │
│  duty. The Shipper and Buyer surfaces show import landed cost │
│  ($56.81). The state tax line appears here for completeness.  │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  ✦ Compliance Screening                         [E3→K7-K10] │
│                                                              │
│  PGA Jurisdiction:       None required for HS 6912           │
│                          (see lead/glaze note above)         │
│  Restricted Party Check: No matches found                    │
│    Screened: OFAC SDN, BIS Entity List, UFLPA Entity List    │
│    As of: [K7/K8/K9 load date]                               │
│  UFLPA Assessment:       N/A (Portugal origin)               │
│  Special Requirements:   None identified                     │
│                                                              │
│  Status: ✓ ENTRY ELIGIBLE — No compliance holds identified   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

> **Value Source Map — Scene 1 Wireframe (US Destination)**
>
> Every value displayed in the wireframe above is produced at runtime by the engines operating against the knowledge layer. No values are hardcoded in the system. The wireframe shows example output for a **US destination** scenario; K-numbers below refer to US knowledge collections (K1-US, K2-US, etc. per the build specification). For EU or China destinations, the engines dispatch to the corresponding jurisdiction's knowledge collections (K1-EU/K2-EU or K1-CN/K2-CN) and tax regime template.
>
> | Value | Source | Knowledge Collection | Notes |
> |---|---|---|---|
> | Classification (6912.00.4800) | Engine 1 (LLM reasoning) | K1 (HTSUS) | Validated against K1 before display |
> | MFN rate (9.8%) | Engine 2 (lookup) | K2 (HTSUS rates) | Deterministic lookup, must be 100% accurate |
> | 301/232/IEEPA/AD rates | Engine 2 (lookup) | K3/K4/K5/K6 | Each program checked independently |
> | Entry type (Informal) | Engine 2 (logic) | K16 + value threshold | Based on declared value vs. $2,500 |
> | MPF ($2.69) | Engine 2 (lookup) | K16 (CBP fee schedule) | Varies by entry type and fiscal year |
> | State tax | Engine 2 (computation) | K13 (state rates + base rules) | Rate × base per state rules. Domestic obligation — excluded from the $56.81 import landed cost shown on Shipper/Buyer surfaces. |
> | PGA jurisdiction | Engine 3A (lookup) | K10 (PGA-HS mapping) | Includes regulatory authority citation |
> | Restricted party result | Engine 3B (screening) | K7/K8/K9 (lists) | Names all lists screened + data date |
> | UFLPA assessment | Engine 3C (logic) | K9 + origin + sector mapping | Rules-based on origin country |
> | Shipping estimate | Stub | N/A | Labeled as estimate; production integrates carrier API |
> | FDA/lead caveat | Engine 1 (LLM) + K10 | K10 (PGA regulatory citations) | Citation must come from K10, not LLM memory |

**What the audience sees that's different from theater:**
1. The reasoning chain is visible. They can see the system moving through HTSUS sections, chapters, headings. It's thinking, not regurgitating.
2. The lead/glaze note — the system flagged a potential FDA issue *unprompted*. It noticed the product is ceramic tableware and raised the lead content question. This is intelligence, not a script.
3. Every tariff program is checked and resolved with a reason, not just "N/A."

**The presenter's move after the analysis completes:**

"That took [X] seconds. Let me show you what a different product looks like."

The presenter clears the form and types a new description: **"Wireless Bluetooth speaker, lithium-ion battery, plastic housing, manufactured in Shenzhen, China"**

Selects China as country of origin. US. $45.00 customs value. Clicks Analyze.

Now the system returns a dramatically different result:
- Classification to 8518.21 (single loudspeaker, mounted in its enclosure) or 8527.92 (radio reception apparatus) — and it shows the *ambiguity*: "Description matches both 8518 (loudspeaker) and 8527 (radio reception). If the primary function is amplifying sound from an external source via Bluetooth, 8518.21 applies. If it includes FM radio reception, 8527.92 applies. Classified as 8518.21 based on 'speaker' description — confirm if radio functionality exists."
- Tariff stack: MFN 0% (free) — but Section 301 + IEEPA stack on top for China origin, creating a dramatic total tariff that's entirely surcharge-driven
- PGA: FCC equipment authorization may be required (electronic device)
- Compliance: Lithium battery → DOT/PHMSA hazmat shipping requirements

**The contrast is the point.** The Portuguese mug was simple — clear classification, moderate duty, no compliance issues. The Chinese Bluetooth speaker is complex — classification ambiguity, tariff stacking from surcharges alone (MFN is actually free), regulatory requirements. The system handled both, and it handled the complexity by being *explicit about the complexity*.

**Presenter says:** "Same system. Two very different products, two very different results. The mug was simple — about ten percent duty, no complications. The speaker is fascinating — the base duty is actually zero, but Section 301 and IEEPA surcharges push the total to sixty-plus percent. The system surfaced that the tariff drama comes entirely from the surcharge programs, not the base rate. That's insight you can't get from a lookup table."

**The global pivot (optional, if time and audience warrant):** The presenter changes the destination for the Bluetooth speaker from US to **EU**. Clicks Analyze. The tariff stack changes entirely — no Section 301, no IEEPA, but EU MFN duty applies, EU trade remedies may apply, and EU VAT computes on a different base. "Same product, same origin, different destination. The entire cost structure changes. The system used a different tax regime template — this is the EU's tariff logic, not the US's. That's the architectural insight: adding a country is loading knowledge into a template, not building a new engine."

**The Shipper Surface Reveal (the first switching moment):**

After both product analyses are complete, the presenter pauses. "You've just seen what the platform's intelligence layer looks like — classification reasoning, tariff stacking, compliance screening. This is what a compliance officer or trade analyst sees. But Maria — the ceramics artist in Lisbon — she doesn't see any of this."

The presenter navigates to the **Shipper Surface.** The screen changes fundamentally — not a collapsed version of the Platform Intelligence view, but a different application entirely:

```
┌──────────────────────────────────────────────────────────────┐
│  ✦ Ship to: Chicago, IL (US)                                  │
│                                                                │
│  ┌────────────────────────────────────────────────────────┐   │
│  │  Handcrafted ceramic mug, glazed stoneware, 450ml       │   │
│  │  From: Lisbon, Portugal                                  │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                                │
│  Status: ✓ Ready to ship                                       │
│                                                                │
│  Total landed cost:    $56.81                                  │
│    Product             $38.00                                  │
│    Shipping            $12.40                                  │
│    Import duties        $3.72                                  │
│    Processing fee       $2.69                                  │
│                                                                │
│  Documents needed:                                             │
│    ☑ Commercial invoice (auto-generated)                       │
│    ☐ Packing list                                              │
│                                                                │
│  Estimated delivery: Feb 12-15                                 │
│                                                                │
│  No restrictions. No special requirements.                     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

No HTSUS codes. No tariff program names. No GRI citations. No confidence scores. No reasoning chains. Just: ready to ship, here's the cost, here's what you need. The commercial invoice is already auto-generated from the order data.

**Presenter says:** "This is what Maria sees. In production, this panel lives inside her Shopify dashboard. She never leaves her workflow. She never sees a tariff code. She typed a product description and a destination — the system did everything else. All of that" — gestures back toward where the Platform Intelligence analysis was — "became this."

The audience should feel the contrast physically. A screen of analytical complexity just became five lines of actionable simplicity. That contrast is the platform's value proposition, and they just saw it happen.

**Then the setup for Scene 6:** "We're going to come back to this interface at the end. When we do, I want you to have a product in mind — and a market."

---

<a name="scene-2"></a>
### Scene 2: The Compliance Dashboard (10 Minutes)

**Emotional target:** "This is what strategic compliance looks like when you have real analytical engines behind it — across trade lanes."

This scene shifts from product-level analysis to portfolio-level intelligence. The dashboard is pre-loaded with data for a fictional company (AutoParts Global) that imports across multiple trade lanes (Mexico→US, China→US, Germany→US, with some US→EU and US→China export exposure). The analytical engines are real — the duty rates, compliance flags, and FTA assessments are computed by the same engines that ran in Scene 1.

**The dashboard is pre-populated but live.** The overnight entries were "processed" by the classification and tariff engines. The volume must be realistic for the company profile — ~47 entries overnight from multiple origins, with 3-5 exceptions and 1-2 strategic alerts. A new element: **regulatory change alerts** from Engine 6, showing pending changes that affect the portfolio.

**Screen layout:** Four summary cards (overnight entries, exceptions, strategic alerts, **regulatory change signals**), exception breakdown by type, strategic alerts with drill-down. The regulatory change signal card shows a count badge: "3 active signals affecting your portfolio" with severity indicators (fuzzy vs. confirmed).

**The key difference from the original design:** When the presenter clicks into the USMCA savings alert, the FTA qualification analysis is not a pre-rendered screen — the engine is showing its actual assessment. The RVC calculation runs against the structured USMCA rules. The component-by-component analysis applies real tariff shift logic.

**The drill-downs are where the engines show:**

**USMCA Savings Alert drill-down:**
- The presenter clicks into the alert
- The system shows: "HS 8708.99 — Motor vehicle parts. Currently paying MFN [X]% + IEEPA [Y]%. USMCA preferential rate: 0%."
- Below that, the FTA engine's analysis: "USMCA Rule of Origin for 8708.99: Regional Value Content minimum 75% (net cost method) OR Tariff Shift at Chapter level from any non-USMCA country."
- The system shows the BOM analysis with each component, its origin, and whether it satisfies the tariff shift or RVC test
- Result: "RVC: 87.4% — Exceeds 75% minimum. Product QUALIFIES for USMCA preferential treatment."
- The documentation section shows what would need to be filed: the 9 minimum data elements for a USMCA certification of origin (as specified in USMCA Chapter 5, Article 5.2) — note: USMCA does not require a specific form; the certification can appear on any document

**UFLPA Risk Alert drill-down:**
- This is where the denied party screening engine works live
- The presenter can type any company name into the entity search and the system screens it against the actual UFLPA Entity List
- The pre-loaded scenario shows a tier-2 supplier match, but the live screening capability means the presenter can demonstrate: "Let me check another entity" and type a name — and the system screens it in real time
- If the presenter types a name from the actual UFLPA Entity List, the system finds it. If they type a random name, the system shows no match. It's real.

**Presenter script evolves from the original:** Instead of narrating what's on screen, the presenter *operates* the system and explains what the engines are doing. "The FTA engine just checked each component against the USMCA rules of origin for Chapter 87. It's applying the net cost RVC method — that's the math you see here. 87.4% against a 75% threshold. This isn't a number we put in a spreadsheet. The engine computed it from the bill of materials."

---

<a name="scene-3"></a>
### Scene 3: Trade Lane Comparison and Consumer Landed Cost (8 Minutes)

**Emotional target:** "Same product, three markets, three completely different cost structures. This is why global clearance intelligence matters."

This scene has two beats. The first is new — it demonstrates the global architecture directly. The second is the original consumer landed cost concept, enhanced.

**Beat 1: Trade Lane Comparison (5 minutes)**

The presenter uses a new screen: Trade Lane Comparison. Same product — Maria's ceramic mug from Portugal — but three destination columns side by side:

```
┌─────────────────┬─────────────────┬─────────────────┐
│  → US            │  → Germany (EU)  │  → China         │
│                  │                  │                  │
│  HS: 6912.00.48  │  CN: 6912 00 50  │  HS: 6912.0090   │
│  MFN: 9.8%       │  MFN: 6.0%       │  MFN: 12.0%      │
│  301: N/A         │  Trade remedy:   │  Retaliatory:    │
│  232: N/A         │    N/A           │    N/A (PT orig) │
│  IEEPA: [rate]    │  VAT: 19% (DE)   │  VAT: 13%        │
│  MPF: $2.69       │  No MPF equiv    │  Consumption: N/A│
│  State tax: [IL]  │                  │                  │
│  ─────────────── │  ─────────────── │  ─────────────── │
│  Landed: [total]  │  Landed: [total] │  Landed: [total] │
│  in USD           │  in EUR (USD)    │  in CNY (USD)    │
└─────────────────┴─────────────────┴─────────────────┘
```

Each column is computed by Engine 2 using a different tax regime template. The HS code at 6 digits (6912.00) is the same across all three — proving the global meta key. The national extensions differ. The tax structures differ. The totals differ.

**Presenter says:** "Same mug, same origin, three destinations. Three different tariff regimes, three different tax structures, three different landed costs. The US stack includes surcharge programs — Section 301, IEEPA. The EU stack is simpler but includes 19% VAT. China has its own duty rate and 13% VAT. The system computed all three from different tax regime templates — but the same analytical engine."

This is one of the most powerful moments in the demo because it makes the global architecture tangible. The audience sees that this is not a US customs tool — it's a global trade intelligence platform.

**The Buyer Surface Reveal (the second switching moment):**

The presenter pauses after the trade lane comparison. "You've seen three tariff regimes side by side — the analytical view. Now let me show you what this looks like to the person buying Maria's mug."

The presenter navigates to the **Buyer Surface** — a checkout mockup. Three destination variants, each showing what the consumer sees:

```
┌───────────────────┬───────────────────┬───────────────────┐
│  🛒 Ship to: US    │  🛒 Ship to: DE    │  🛒 Ship to: CN    │
│                    │                    │                    │
│  Ceramic Mug       │  Ceramic Mug       │  Ceramic Mug       │
│  handcrafted,      │  handcrafted,      │  handcrafted,      │
│  Lisbon            │  Lisbon            │  Lisbon            │
│                    │                    │                    │
│  $56.81            │  €52.30            │  ¥389              │
│  delivered          │  delivered          │  delivered          │
│                    │                    │                    │
│  ✓ Duties included │  ✓ Duties included │  ✓ Duties included │
│                    │                    │                    │
│  Est. Feb 12-15    │  Est. Feb 14-18    │  Est. Feb 18-22    │
│                    │                    │                    │
│  [Add to Cart]     │  [Add to Cart]     │  [Add to Cart]     │
│                    │                    │                    │
│  ▸ What's in       │  ▸ What's in       │  ▸ What's in       │
│    this price?     │    this price?     │    this price?     │
└───────────────────┴───────────────────┴───────────────────┘
```

The "What's in this price?" expandable, if opened, shows a three-line breakdown: product, shipping, duties/taxes. No program names. No rate percentages. Just amounts in the buyer's currency.

**Presenter says:** "Three destinations, three prices. That's it. The consumer in Chicago sees $56.81 delivered. The consumer in Berlin sees €52.30. The consumer in Shanghai sees ¥389. 'Duties included.' No surprise charges. No customs hold notification two weeks later. No 'additional fees may apply.' Behind each of those prices, the system computed a full tariff stack under a different country's tax regime. The consumer never sees any of it. They see a price, a delivery date, and a green checkmark."

The audience has just watched three columns of tariff stacks, surcharge programs, and VAT computations in the Platform Intelligence view. Now they see three prices. That compression — from analytical complexity to a checkout line — is the platform's thesis made visible.

**The before/after timeline:**

The presenter briefly shows the contrast between today's experience and the platform-enabled experience:
- **TODAY:** Consumer sees "$38.00 + shipping" at checkout. Weeks later: customs hold, surprise duty bill, "additional charges: $18.47." Customer files a complaint. Seller issues a refund. Everyone loses.
- **TOMORROW:** Consumer sees "$56.81 delivered, duties included" at checkout. Mug arrives on schedule. No surprises. Everyone wins.

The before/after timeline (7 days vs. same-day clearance) is illustrative, not engine-driven, and the presenter frames it as such: "The clearance timeline improvement is what happens when all the data is right *before* the goods ship."

---

<a name="scene-4"></a>
### Scene 4: Regulatory Change Intelligence (8 Minutes)

**Emotional target:** "A tariff change is coming. The system already knows what it means for your business."

This is the new scene that addresses the most acute pain point: the velocity and unpredictability of regulatory change. It uses Engine 6 in all three modes.

**Beat 1: The Signal Board (2 minutes)**

The presenter navigates to the Regulatory Change Intelligence screen. It shows a timeline of active signals across jurisdictions:

```
┌──────────────────────────────────────────────────────────────┐
│  ✦ Regulatory Change Intelligence                             │
│                                                                │
│  Active Signals: 7                                             │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ CONFIRMED  US IEEPA rate increase on China:             │  │
│  │            [product scope] effective [date]              │  │
│  │            Source: Federal Register [citation]           │  │
│  │            Portfolio impact: 12 SKUs, +$47K/month        │  │
│  │            [Model Impact →]                              │  │
│  ├─────────────────────────────────────────────────────────┤  │
│  │ PROPOSED   EU anti-dumping review on Chinese steel:      │  │
│  │            [product scope] — initiation notice published │  │
│  │            Source: Official Journal [citation]           │  │
│  │            Status: Investigation phase                   │  │
│  │            [Model Scenario →]                            │  │
│  ├─────────────────────────────────────────────────────────┤  │
│  │ DISCUSSED  China considering tariff reduction on         │  │
│  │            EU agricultural products                      │  │
│  │            Source: [news source, government statement]   │  │
│  │            Confidence: Low (pre-proposal)                │  │
│  │            [Model Scenario →]                            │  │
│  └─────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

**Presenter says:** "This is what the system is tracking right now. Seven active signals across three jurisdictions. Notice the status levels — CONFIRMED means it's enacted, we have a citation, and the system has already computed the impact. PROPOSED means there's an official proceeding but no final decision. DISCUSSED means someone in a position of authority has mentioned it but it's still speculative. The system never presents a fuzzy signal as a confirmed obligation."

**Beat 2: Scenario Modeling (3 minutes)**

The presenter clicks "Model Scenario" on the proposed EU anti-dumping review. The system:
1. Identifies the HS code range affected
2. Scans the portfolio (K15 simulated data) for matching products and trade lanes
3. Recomputes landed cost for affected products at the current rate AND at a hypothetical anti-dumping duty rate
4. Shows a side-by-side comparison: "If this measure is enacted at the preliminary rate, here's the impact"

```
┌──────────────────────────────────────────────────────────────┐
│  Scenario: EU Anti-Dumping on Chinese Steel                   │
│                                                                │
│  Affected trade lane: China → EU                               │
│  Affected products: 4 SKUs (HS 7208-7210 range)              │
│                                                                │
│  ┌──────────────────────┬──────────────────────┐              │
│  │  CURRENT              │  IF ENACTED            │              │
│  │  MFN: 0%              │  MFN: 0%                │              │
│  │  AD Duty: N/A         │  AD Duty: 22.5%         │              │
│  │  VAT: 19%             │  VAT: 19%               │              │
│  │  Landed: €14,200      │  Landed: €17,400         │              │
│  │                       │  Δ: +€3,200 (+22.5%)     │              │
│  └──────────────────────┴──────────────────────┘              │
│                                                                │
│  Monthly portfolio impact: +€12,800                            │
│  Annual projection: +€153,600                                   │
│                                                                │
│  ⚠ Note: Preliminary rate used. Final rate may differ.        │
│    Investigation timeline: 12-15 months from initiation.       │
└──────────────────────────────────────────────────────────────┘
```

**Presenter says:** "The system just recomputed the tariff stack for these products using a hypothetical anti-dumping rate. It's using the same Engine 2 that computed Maria's mug costs — but with a modified parameter. The monthly impact is €12,800. That's the number your supply chain team needs to start contingency planning. And this is for a *proposed* measure — the system lets you prepare before it's enacted."

**Beat 3: Cross-Jurisdiction Impact (3 minutes)**

The presenter selects the confirmed US IEEPA rate increase. The system shows the automated impact analysis: which SKUs are affected, the cost delta, and — critically — the cross-jurisdiction perspective: "These same products entering the EU are NOT affected by this change. Trade lane shift opportunity identified."

**Presenter says:** "This is where global intelligence creates strategic advantage. The US just increased the tariff on these products from China. But the same products entering the EU face no change. For a company with dual-market presence, the system just identified a trade lane optimization opportunity that would take a human analyst days to map out."

**The Shipper Surface Reveal (the third switching moment):**

After the cross-jurisdiction impact analysis, the presenter pauses. "That's what the compliance team sees. The signal board, the scenario modeling, the portfolio impact — strategic intelligence for people whose job is to manage trade risk. But what about Maria? A tariff changed. What does she need to do?"

The presenter navigates to the **Shipper Surface.** The regulatory change becomes a notification:

```
┌──────────────────────────────────────────────────────────────┐
│  ✦ Ship to: Chicago, IL (US)                                  │
│                                                                │
│  📋 Update — Jan 28, 2026                                      │
│  ┌────────────────────────────────────────────────────────┐   │
│  │  Shipping costs to the US have changed.                  │   │
│  │                                                          │   │
│  │  Your ceramic mug now costs $4.20 more per unit to       │   │
│  │  ship to the US due to an import duty increase that      │   │
│  │  took effect January 15.                                 │   │
│  │                                                          │   │
│  │  New total landed cost: $61.01  (was $56.81)             │   │
│  │                                                          │   │
│  │  Your product pricing has been updated automatically.    │   │
│  │                                                          │   │
│  │  No action required.                                     │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                                │
│  Status: ✓ Ready to ship                                       │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

**Presenter says:** "Maria sees this. One notification. Plain language. 'Your costs went up by $4.20. Pricing updated. No action required.' She doesn't know what IEEPA stands for. She doesn't need to know. She doesn't know that the system modeled this scenario three months before it was enacted, that the compliance team already adjusted sourcing strategy, that the system updated her checkout pricing automatically on the effective date. She sees: the price changed, it's handled, keep shipping."

The audience should feel the sheer scale of what was just compressed. A seven-signal regulatory intelligence dashboard with multi-jurisdiction scenario modeling and portfolio-level impact analysis just became "your costs went up by $4.20, no action required."

**Why this scene matters:** Everyone in the room has been blindsided by a tariff change. This scene transforms that shared anxiety into a competitive advantage narrative — at the strategic level (Platform Intelligence) AND at the human level (Shipper). The system doesn't just calculate what is — it models what might be, alerts when things change, and translates the impact into plain language for every actor in the chain.

---

<a name="scene-5"></a>
### Scene 5: Exception Resolution (7 Minutes)

**Emotional target:** "The system doesn't just prevent problems. When problems happen, it knows what to do."

This scene uses Engine 5 (Exception Analysis) live. The presenter presents a hold scenario and the system generates a response.

**The scenario:** A CBP classification question on a cable assembly (HS 8544.42 vs 8544.49 — "fitted with connectors" or not).

**What the system does live:**
1. The hold details are entered: HS code, product description, CBP's question
2. The exception engine searches the CBP CROSS database for relevant rulings
3. It finds actual rulings (or rulings from its loaded reference set) that address the "fitted with connectors" distinction
4. It analyzes the product description against the ruling language
5. It generates a draft response citing the ruling(s), the product evidence, and the classification basis
6. It shows its confidence and flags where a human should review

**What makes this different from the original design:**
- The CROSS rulings search is real — the system is querying actual CBP rulings data
- The response generation uses the actual regulatory language, not templated text
- The presenter can modify the scenario: "What if it's not connectors but terminals?" The system re-analyzes with different reasoning

**A second scenario (optional, time permitting):** The presenter shows a UFLPA detention scenario. The system outlines the evidence requirements from the UFLPA enforcement strategy, maps available documentation against requirements, and identifies gaps. This shows the system understanding the most complex compliance challenge in current trade.

---

<a name="scene-6"></a>
### Scene 6: Bring Your Own Product (12 Minutes)

**Emotional target:** "This is real. It just analyzed my product, for my market. Correctly."

**This is the scene that doesn't exist in any competitor's demo. This is the scene that creates conviction.**

The presenter returns to the Scene 1 interface — the product description input, the origin/destination dropdowns, the Analyze button. And then:

**"I'd like someone in the room to describe a product they actually import — and tell me where it's going."**

The addition of "where it's going" is critical. It signals that the system is global. The audience member names a product AND a destination. The system analyzes it for that market.

This is a controlled risk. The system has a defined scope. But within that scope, it works. And the audience doesn't know the scope — they just see the system thinking.

**How to manage this moment:**

1. **The presenter pre-briefs 1-2 friendly audience members** before the session. They know to suggest products that are within scope (common consumer goods, auto parts, electronics, textiles, food products) and destinations that are loaded (US, EU, China). This guarantees at least one successful analysis.

2. **The system handles out-of-scope gracefully.** If someone names a destination not yet loaded (e.g., "import into Indonesia"), the system says: "Indonesia is not yet in the active knowledge base. The system can classify the product at HS 6-digit level (globally valid) but does not have Indonesia's tariff schedule loaded. Here's the result for [US/EU/China] — the same analytical approach extends to any WCO member country once the tariff schedule is loaded." This is honest, demonstrates understanding of the architecture, and pivots to a live demonstration.

3. **Run 2-3 audience products.** Each one takes 2-3 minutes. If possible, analyze at least one product for two different destinations — the side-by-side comparison is the strongest proof of the global architecture. The presenter narrates what the engines are doing as they work.

**The "holy shit" moment:** When the system correctly classifies a product that an audience member assumed was obscure — and then shows a tariff stack they didn't know about, or flags a compliance requirement they hadn't considered, or identifies that the same product is 40% cheaper to clear into the EU than the US because of surcharge programs — that is the moment conviction happens. The global comparison is a new vector for this moment: the audience sees not just what duties are, but how they differ across markets. That's strategic intelligence.

**The Triple Surface Reveal (the closing image of the demo):**

After the last audience product is analyzed in Platform Intelligence, the presenter delivers the closing sequence. This is the most important switching moment — the one that leaves the final impression.

**Presenter says:** "You've just seen the system analyze your product. Let me show you what happens to that analysis as it moves through the supply chain."

The presenter stays on the Platform Intelligence view for a beat. "This is what your compliance team sees. The full analytical depth — classification reasoning, tariff stack by program, compliance screening, regulatory change exposure. Every source cited. Every limitation acknowledged. This is the command center."

Switch to the **Shipper Surface.**

```
┌──────────────────────────────────────────────────────────────┐
│  ✦ Ship to: [audience member's destination]                    │
│                                                                │
│  [Audience member's product description]                       │
│  From: [origin]                                                │
│                                                                │
│  Status: ✓ Ready to ship                                       │
│                                                                │
│  Total landed cost:    [computed total]                         │
│    Product             [declared value]                         │
│    Shipping            [estimate]                               │
│    Import duties       [computed]                               │
│    Taxes & fees        [computed]                               │
│                                                                │
│  Documents needed:                                             │
│    ☑ Commercial invoice (auto-generated)                       │
│    ☐ [any required documents]                                  │
│                                                                │
│  No action required.                                           │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

"This is what your logistics coordinator sees. Ready to ship, total cost, documents needed. They never saw a tariff code."

Switch to the **Buyer Surface.**

```
┌──────────────────────────────────────────────────────────────┐
│                                                                │
│  [Product name]                                                │
│                                                                │
│  [Total landed price in destination currency]                  │
│  delivered — duties and taxes included                         │
│                                                                │
│  ✓ Duties included                                             │
│                                                                │
│  Estimated delivery: [date range]                              │
│                                                                │
│  [Add to Cart]                                                 │
│                                                                │
│  ▸ What's in this price?                                       │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

"And this is what your customer sees. One price. Delivered. No surprises."

**Pause.**

"Three surfaces. One system. Six engines. The same analytical intelligence that just classified your product, computed duties under three different countries' tax regimes, screened restricted party lists, and checked regulatory requirements — all of that became a checkmark and a price."

"That's what we're building."

This is the image the audience takes home. Not the complexity of the engines. Not the depth of the knowledge layer. The fact that all of it becomes invisible to the people who ship and buy. The complexity is absorbed. The experience is simple. That's the product.

**Fallback if something goes wrong:** The system is wrong. It misclassifies. The tariff rate looks off. This is actually fine, IF the presenter handles it: "The system classified this as [X]. You're telling me it's actually [Y]. That's exactly the kind of edge case that requires the classification engine to learn from — and that feedback loop is built into the platform design. The engine got us to the right chapter and heading; the subheading distinction between [X] and [Y] requires domain expertise that the engine would learn from with more training data on this specific product category." This is honest, demonstrates understanding, and frames the error as a feature of the learning architecture.

---

<a name="design-principles"></a>
## Design Principles

### Resequenceability

Every scene is independently accessible. The golden path above is a recommended narrative arc, but the system supports any scene order. This is critical for several reasons:

1. **Audience-driven navigation.** If an audience member asks "can you show me the regulatory change impact?" during Scene 1, the presenter can jump to Scene 4 without breaking anything. Each engine is independently callable. Each screen initializes from its own state, not from a prior screen's output.

2. **Tailored presentations.** For a supply chain audience, start with Scene 4 (Regulatory Change) and Scene 2 (Dashboard). For a consumer-focused audience, start with Scene 3 (Trade Lane Comparison). For a technical audience, start with Scene 1 and go deep on classification. The system accommodates all approaches.

3. **Recovery from live demo issues.** If an engine takes too long or produces an unexpected result in one scene, the presenter can pivot to another scene without the audience feeling like something broke. The scenes are independent.

**Implementation requirement:** No scene stores state that another scene needs. The product analysis (Scene 1) doesn't populate data that the dashboard (Scene 2) requires. The dashboard doesn't prepare data that the regulatory change screen (Scene 4) requires. Each scene reads from its own knowledge and engine calls.

### Global Architecture Visibility

The demo should make the global architecture visible, not just talk about it. This means:
- The destination country dropdown is always visible and always changeable
- Switching destinations triggers a visible recomputation — the audience sees the tax regime template change
- Currency display switches with destination
- The Trade Lane Comparison screen (Scene 3) is the most explicit demonstration of this principle

### Transparency Over Performance

The system shows its work. Every calculation is sourced, every reasoning step is visible, every limitation is acknowledged. This applies across jurisdictions — when the system computes EU VAT, it shows the base it used and why. When it applies China's consumption tax, it explains which product categories trigger it. Transparency is not a US-specific value — it's the platform's identity.

### Three Application Surfaces

This is the most important design principle in the system. The six engines and the knowledge layer are shared infrastructure. What changes is the surface — and the surface must meet each actor in their natural context, not ask them to come to ours.

The system presents three fundamentally different application surfaces. These are not "views" of the same screen. They are different applications — different layouts, different language, different interaction models — powered by the same analytical core.

#### Surface 1: Platform Intelligence (The Command Center)

**Who uses it:** Compliance officers, trade analysts, customs brokers, strategic sourcing managers. People whose job is to understand, decide, and act on trade complexity.

**Where it lives:** This is a destination application. These users come to it. It is their primary work environment for clearance intelligence — the way a Bloomberg terminal is the primary environment for a trader.

**Design philosophy: Smart surfacing with progressive disclosure.**

The command center does NOT show everything at once. It surfaces the most decision-relevant information first — exceptions, regulatory changes, savings opportunities, risk flags — with the ability to drill into any dimension to its full depth.

- **Summary layer (default view):** Every analysis opens with a decision-oriented summary. "HS 6912.00.48 — MFN 9.8% + IEEPA [rate] — No compliance holds — Entry eligible." One line. The answer.
- **Reasoning layer (one click):** Expand any summary to see the full analytical chain. Classification reasoning with GRI citations. Tariff stack program-by-program with source annotations. Compliance screening with list-by-list results.
- **Evidence layer (two clicks):** From any reasoning step, drill to the underlying data. The HTSUS legal text. The specific Federal Register citation. The CROSS ruling that supports the classification. The restricted party list entry with match score.
- **Source annotations are discoverable, not decorative.** The [E1→K1] engine-to-knowledge annotations are available on hover or in an information panel — not competing with the analytical output for visual attention.

**What makes it brilliant, not just comprehensive:** The dashboard doesn't present 47 entries with equal weight. It presents: "3 entries need your attention (1 UFLPA risk, 1 classification question, 1 FTA savings opportunity). 44 entries cleared normally." The system has already triaged. The analyst's job starts at the exceptions, not at entry #1.

**Cross-jurisdiction intelligence is native.** The command center shows portfolio exposure across all trade lanes simultaneously. A regulatory change in the US is immediately contextualized against EU and China trade lanes — "these products are affected on the US lane; the EU lane is unaffected; trade lane shift opportunity identified." The analyst sees the global picture without switching between country-specific views.

#### Surface 2: Shipper Experience (Workflow-Embedded)

**Who uses it:** Maria the artisan. Logistics coordinators. E-commerce sellers. Fulfillment teams. Third-party logistics providers. Anyone whose job is to get goods from origin to destination.

**Where it lives:** This is NOT a standalone application. The shipper never opens "Clearance Intelligence Platform." The clearance intelligence is embedded in the tools they already use — their Shopify dashboard, their shipping platform, their ERP, their fulfillment center screen. It appears as a panel, a widget, an API response within their existing workflow.

**Design philosophy: Invisible complexity. Three questions answered.**

The shipper surface answers exactly three questions:
1. **Can I ship this?** (Yes / Yes with conditions / No — and why)
2. **What will it cost?** (One number: total landed cost including all duties, taxes, and fees)
3. **What do I need to include?** (Plain-language document checklist)

That's it. No HTSUS codes. No tariff program names. No GRI citations. No confidence scores. No reasoning chains. The shipper doesn't need to know that the system traversed the HTSUS hierarchy, checked six surcharge programs, screened three restricted party lists, and computed VAT on a customs-duty-inclusive base. The shipper needs to know: **$56.81, ready to ship, include a commercial invoice.**

**Key characteristics:**
- **Language is plain English (or the shipper's language).** Not "Section 301 surcharge applicable" but "Additional import duty applies for products from China." Not "PGA jurisdiction: FDA — 21 CFR 177" but "Food contact declaration required."
- **Status is traffic-light simple.** Green: ready to ship. Yellow: action needed (with a specific, plain-language action). Red: cannot ship (with the reason in one sentence).
- **Cost is one number with an optional breakdown.** The total landed cost is prominent. If the shipper wants to see what's in it, they can expand to a simplified breakdown: "Product: $38.00. Shipping: $12.40. Import duties: $3.72. Processing fee: $2.69. Total: $56.81." No program names. No rate percentages. Just dollar amounts.
- **Regulatory changes arrive as notifications, not signals.** The shipper doesn't see a signal board with CONFIRMED/PROPOSED/DISCUSSED status levels. They see: "📋 Heads up: Shipping costs to the US will increase by approximately $4.20 per unit starting March 1. No action needed — your pricing will update automatically." The anxiety is absorbed. The action is clear (or absent — "no action needed" is the ideal state).
- **Document requirements are a checklist.** Not a regulatory mapping — a checklist. "☐ Commercial invoice ☐ Packing list ☐ Food contact declaration (template attached)." If the platform can pre-populate or auto-generate any of these, it does so silently.

**What the demo shows:** A clean panel that represents the shipper integration. The presenter doesn't pretend it's embedded in Shopify — they show the panel and say: "This is what Maria sees. In production, this panel lives inside her shipping platform. She never leaves Shopify. She never sees a tariff code. She sees this." The contrast with the Platform Intelligence view is the point.

#### Surface 3: Buyer Experience (Checkout-Embedded)

**Who uses it:** The end consumer. The person buying Maria's mug from Lisbon.

**Where it lives:** The checkout page of the e-commerce site. It is a line item, a badge, or a price — not a screen, not a panel, not an application.

**Design philosophy: One number. Total transparency in one line.**

The buyer sees: **"$56.81 delivered — duties and taxes included."**

If they're curious, one expandable line: "Includes $3.72 import duty and $2.69 customs processing." If they're not curious, the expandable never opens. The purchase completes. The mug arrives. No surprise charges. No customs hold notification. No "additional fees may apply" disclaimer.

**Key characteristics:**
- **"Duties included" is a trust badge, not a disclaimer.** It appears next to the price the way "Free shipping" appears — as a positive signal, not a caveat.
- **The breakdown is optional and simple.** Product price, shipping, duties/taxes. Three lines. The buyer doesn't need to know there are six surcharge programs and a Merchandise Processing Fee.
- **Currency is the buyer's currency.** If the buyer is in Berlin, they see euros. If they're in Shanghai, they see yuan. The conversion happened upstream. The buyer never sees "USD equivalent."
- **Delivery estimate is clean.** "Estimated delivery: February 12-15." The buyer doesn't need to know that clearance timing depends on PGA jurisdiction or that the system pre-cleared the shipment.

**What the demo shows:** A checkout mockup — almost comically simple compared to what's behind it. The audience has just watched the Platform Intelligence view compute a 12-line tariff stack with regulatory citations and compliance screening. Now they see: "$56.81 delivered." That visual contrast — a screenful of analytical complexity becoming a single price line — IS the pitch.

#### The Switching Moment: The Demo's Thesis Made Visible

The act of switching between surfaces is the single most powerful recurring motif in the demo. Each switch is a visual proof of the platform's value proposition: **the system absorbs complexity so that the people who ship and buy never have to.**

The switching is not a UI toggle on a shared screen. It is a navigation to a fundamentally different surface — different layout, different information density, different language. The presenter treats it as a reveal: "Now let me show you what Maria sees." The audience watches a screen full of tariff analysis dissolve into "Ready to ship. $56.81."

The switching moments are mapped to specific scenes (see Scene descriptions above). The recommended switching points are:

| Scene | Switch | What the Audience Sees |
|---|---|---|
| Scene 1 (after full product analysis) | Platform Intelligence → Shipper | Full classification + tariff stack + compliance → "✓ Ready to ship. $56.81. Commercial invoice required." |
| Scene 3 (after trade lane comparison) | Platform Intelligence → Buyer | Three columns of tariff stacks across jurisdictions → Three checkout prices. |
| Scene 4 (after regulatory change modeling) | Platform Intelligence → Shipper | Signal board + scenario modeling + portfolio impact → "Shipping costs to the US will increase by $4.20 on March 1. No action needed." |
| Scene 6 (Bring Your Own Product) | Platform Intelligence → Shipper → Buyer | Full analysis → "What your logistics team sees" → "What your customer sees at checkout." The triple reveal is the closing image. |

---

<a name="data-infrastructure"></a>
## Data Infrastructure Required

This is the real work behind the demo. The engines are only as good as their data. The global scope increases the data preparation effort substantially, but the structure is the same for every jurisdiction.

### Data Sources by Jurisdiction

**All data is publicly available.** Tariffs are law, and laws are public. The challenge is not access — it's parsing, structuring, and cross-referencing.

**US (Tier 1 — Full Depth):**

| Source | Content | Format | Language |
|---|---|---|---|
| **HTSUS** (usitc.gov) | Full tariff schedule — chapters, headings, subheadings, rates, notes | Structured text/PDF → needs parsing | English |
| **Section 301 Lists** (USTR) | Product lists with HS codes and rates | Federal Register notices | English |
| **Section 232 Scope** (Commerce) | Steel/aluminum HS codes, country exemptions | Published proclamations | English |
| **IEEPA Rate Schedule** (White House/CBP) | Country-specific reciprocal rates | Executive orders + CBP guidance | English |
| **AD/CVD Orders** (ITC) | Active orders with HS codes and rates | ITC database | English |
| **OFAC SDN List** (Treasury) | Designated nationals and blocked persons | XML, CSV | English |
| **BIS Entity List** (Commerce) | Export-restricted entities | Federal Register | English |
| **UFLPA Entity List** (DHS) | Forced labor entities | DHS FLETF website | English |
| **CBP PGA Reference** (CBP) | HS-to-agency mappings | ACE PGA message set docs | English |
| **USMCA Rules of Origin** (USTR) | Product-specific rules | Treaty text (Annex 4-B) | English |
| **CBP CROSS Rulings** (CBP) | Classification rulings | rulings.cbp.gov | English |
| **State Tax Rates** (various) | State + local sales/use tax | Aggregated from state DORs | English |
| **CBP Fee Schedule** | MPF, HMF rates by fiscal year | CBP published schedule | English |

**EU (Tier 2 — Meaningful Depth):**

| Source | Content | Format | Language |
|---|---|---|---|
| **TARIC** (European Commission) | EU Combined Nomenclature + duty rates + trade remedies + suspensions | Machine-readable database | English |
| **EU VAT Rates** (European Commission) | Standard and reduced rates by member state | VIES database | English |
| **EU Sanctions Lists** (European Commission) | Restrictive measures — persons and entities | Downloadable XML | English |
| **CE/REACH Requirements** | Product safety and chemical regulation | EUR-Lex / ECHA | English |

**China (Tier 2 — Meaningful Depth):**

| Source | Content | Format | Language |
|---|---|---|---|
| **China Customs Tariff** (GACC) | 8-digit tariff schedule | Published schedule | Chinese (English via WTO at 6-digit) |
| **MOFCOM Tariff Measures** | Retaliatory tariffs on US/EU goods | MOFCOM announcements | Chinese (English translations available) |
| **China VAT/Consumption Tax** | Rates by product category | State Taxation Administration | Chinese |
| **CCC Requirements** (CNCA) | Compulsory certification by product | Published catalogs | Chinese (English available) |

**Brazil (Tier 3 — Framework):**

| Source | Content | Format | Language |
|---|---|---|---|
| **NCM / TEC** (Receita Federal) | Mercosur common tariff | Published schedule | Portuguese |
| **ICMS/IPI/PIS/COFINS** (Receita Federal) | Tax rates by state and product | Tax authority databases | Portuguese |

### Data Preparation Work

**The most significant build effort is not the engines — it's structuring the data. This is true for every jurisdiction.**

**US Knowledge (Phase 1-3):**
1. **HTSUS Parsing:** Full tariff schedule into structured, queryable format. One-time but substantial.
2. **301/232/IEEPA Cross-Referencing:** Separate documents cross-referenced at HS code level.
3. **CBP CROSS Rulings Indexing:** Thousands of rulings indexed by HS code and topic.
4. **PGA Mapping:** HS code → agency requirements structured into lookup.
5. **USMCA Rules of Origin Structuring:** Legal text into executable logic for ~50 categories.

**EU Knowledge (Phase 4A):**
6. **TARIC Integration:** EU's tariff database is the most machine-friendly in the world. Parsing effort is moderate. The challenge is mapping TARIC's additional code system (trade remedies, quotas, suspensions) to the tax regime template structure.
7. **EU VAT Rate Table:** Standard and reduced rates by member state. Straightforward.

**China Knowledge (Phase 4B):**
8. **China Tariff Schedule Parsing:** The primary challenge is language. The Chinese schedule at 8-digit level requires Chinese-language processing. Alternative: use WTO Tariff Download Facility for 6-digit English rates (free, immediate) and supplement with commercial providers for 8-digit detail.
9. **MOFCOM Retaliatory Tariff Cross-Referencing:** Chinese-language announcements with HS code lists.

**Regulatory Change Signals (Phase 5):**
10. **Signal Database Seeding:** 20-30 real signals across US, EU, and China. This is research and curation work, not data engineering. Each signal needs: source, citation, affected HS range, status (fuzzy/confirmed), effective date (if confirmed).

---

<a name="wrong"></a>
## What the System Can Be Wrong About (and Why That's Good)

A system that is always right is a system that is hiding its mistakes. A system that shows its reasoning and admits uncertainty is a system that builds trust.

**Classification ambiguity:** The system will encounter products where two HS codes are plausible. Rather than guessing, it shows both options, explains the distinction, and identifies what additional information would resolve the ambiguity. "This could be 8518.21 (single loudspeaker in enclosure) or 8527.92 (radio apparatus). The distinction depends on whether the device includes independent radio reception capability." This is more valuable than a confident wrong answer.

**Rate currency:** Tariff rates change with executive orders, sometimes with days of notice. The system should show the "as of" date for its rate data in every jurisdiction. "IEEPA reciprocal rate for China: 60% (as of January 15, 2026). Note: IEEPA rates have been modified 4 times in the past 12 months." This is honest and shows the system understands the volatility problem. Engine 6 (Regulatory Change Intelligence) is the structural response to this uncertainty — it doesn't just show current rates, it shows what's coming.

**Cross-jurisdiction classification divergence:** HS 6-digit is globally harmonized, but national subheadings can differ. A product classified as 8518.21.00.00 in the US may map to a different 8-digit code in the EU's Combined Nomenclature. The system should note when national extensions differ: "HS 6-digit classification (8518.21) applies globally. US HTSUS extension: 8518.21.0000. EU CN extension: 8518 21 00. These are equivalent for this product." Occasionally they are not equivalent, and the system should flag this honestly.

**FTA qualification uncertainty:** Rules of origin analysis often requires data the system doesn't have (exact BOM weights, manufacturing process details, cost allocation). The system should say: "Based on the simplified BOM provided, RVC appears to meet the 75% threshold. A definitive determination requires a complete cost analysis per USMCA Article 5.4." This is what a competent trade advisor would say.

**Why errors build conviction (when handled well):**
- They prove the system is actually analyzing, not playing back recordings
- They demonstrate that the team understands the problem's complexity
- They create opportunities for the presenter to show domain expertise: "Great catch — the system assumed this is a loudspeaker, but if it has FM radio, the classification shifts. That's exactly the kind of feedback the platform learns from."
- They make the successes more credible: if the system can be wrong, then when it's right, it earned that answer

---

<a name="design"></a>
## Visual and Interaction Design

### The Fundamental Design Principle

**The system is showing its work.** Every screen should make the analytical process visible. The audience should be able to follow the reasoning — not just see the answer. This means:

- Reasoning chains are first-class UI elements, not hidden in expandable sections
- Source data is cited inline ("MFN rate: 9.8% — HTSUS 6912.00.48, General Rate column")
- Confidence levels and uncertainty are visible, not hidden
- The system's limitations are stated, not suppressed

### Layout

Two-panel layout throughout:
- **Left panel:** Input (product description, parameters, controls)
- **Right panel:** Analysis (results, reasoning, evidence)

The right panel is the star. It should feel like watching an expert think — structured, sourced, deliberate.

### Color Palette

Same as original design (FedEx Purple accent, near-white background, semantic colors for success/warning/error). One addition:

| Use | Color | Hex | Notes |
|---|---|---|---|
| Reasoning/process | Light blue-gray | #EFF6FF | Background for reasoning chain steps — distinguishes "how the system thought" from "what the system concluded" |
| Source citation | Muted blue | #3B82F6 | Inline citations linking to HTSUS text, rulings, regulations |

### Typography

- **Reasoning chains:** JetBrains Mono or SF Mono, 14px, with left-border color-coding by engine (purple for classification, amber for tariff, green for compliance)
- **Results/conclusions:** Inter Bold, 18px
- **Source citations:** Inter Regular, 13px, in muted blue, clickable

### Animation

- **Reasoning chains build sequentially** — each step appears 200-400ms after the last, creating the feel of the system thinking through the problem
- **No fake delays.** If the LLM responds in 2 seconds, the reasoning appears in 2 seconds. If it takes 5, it takes 5. The realness of the timing is part of the credibility.
- **Tariff stack builds program by program** — each check appears, resolves (applies or N/A), and the running total updates. 300ms per program.

### Responsiveness to Input

This is the most important design quality: **the system reacts to what you type.** When the presenter changes the origin country, the tariff stack recalculates. When they modify the product description, the classification re-runs. The system feels alive because it *is* alive — it's processing real inputs against real data.

---

<a name="presenter"></a>
## Presenter Strategy

### The Role Shifts

In the theater version, the presenter was an actor performing a script. In this version, the presenter is a **domain expert operating a global system.** The presenter must:

1. **Know trade — globally.** The presenter should understand not just US tariff programs (301, 232, IEEPA) but the equivalent concepts for EU (trade remedies, VAT structure) and China (retaliatory tariffs, consumption tax, VAT). They don't need to be an expert in every jurisdiction, but they need to explain what the system is doing when it switches tax regime templates. "The EU doesn't have Section 301 — but it has its own trade remedy framework. Watch what the system does differently."

2. **Be comfortable with live systems.** Things will take a moment to compute. The classification engine may hesitate on a complex product. The presenter fills these moments with explanation, not apology: "The system is working through the HTSUS hierarchy — this product touches multiple chapters and it's resolving which one takes precedence under the General Rules of Interpretation."

3. **Handle errors with confidence.** If the system gets something wrong, the presenter treats it as a teaching moment. "The system classified this as 6911 — porcelain. But we know this product is stoneware, not porcelain. The distinction is in the material composition, and the description didn't specify it clearly enough. In production, this is where the confidence score triggers human review."

4. **Navigate non-linearly.** The presenter must be comfortable jumping between scenes based on audience interest. "You're asking about rate changes — let me show you something." Jump to Scene 4. Show the regulatory change intelligence. Return to wherever you were. The system supports this; the presenter must practice it.

5. **Master the surface switches.** The act of switching between Platform Intelligence, Shipper, and Buyer surfaces is the demo's most powerful recurring motif. The presenter must treat each switch as a reveal — a deliberate, paced moment, not a quick tab change. Practice the pause before and after. Practice the gesture ("All of that became this."). The contrast between the analytical depth and the shipper/buyer simplicity is the platform's thesis. The audience needs a beat to absorb it.

6. **Know which surface you're on.** After a switch to the Shipper or Buyer surface, the presenter should name it: "This is the shipper's view." "This is what appears at checkout." When returning to Platform Intelligence: "Back to the command center." This labeling is important — it teaches the audience that these are fundamentally different applications, not tabs on a dashboard.

### The Arc of the Presentation

| Phase | Presenter Stance | Audience State | Surface |
|---|---|---|---|
| Opening (cards) | Storyteller — creating urgency about velocity and scale | Receiving information, feeling the pace | Static cards |
| Scene 1 (products) | Demonstrator — showing the system works live, globally | Observing, assessing | Platform Intelligence → **Shipper reveal** |
| Scene 2 (dashboard) | Strategist — showing portfolio intelligence across trade lanes | Connecting to their world | Platform Intelligence |
| Scene 3 (trade lanes) | Architect — showing the global platform structure | Understanding the ambition | Platform Intelligence → **Buyer reveal** |
| Scene 4 (regulatory change) | Advisor — showing preparedness for volatility | Feeling relieved, engaged | Platform Intelligence → **Shipper reveal** |
| Scene 5 (exception) | Problem-solver — showing resilience | Gaining confidence | Platform Intelligence |
| Scene 6 (bring your own) | Facilitator — the audience operates, globally | Active participation, conviction | Platform Intelligence → **Shipper → Buyer** (triple reveal) |

### Pre-Session Preparation

1. **Test 20+ product descriptions** through the classification engine for at least two destinations (US and EU). Know which ones the system handles well and which ones it struggles with. Have these in your back pocket for Scene 1 if you need to add examples.
2. **Brief 1-2 audience members** on what products to suggest in Scene 6. Not scripted — just "think about a product you actually import, where it's coming from, and where it's going. Be ready to describe it."
3. **Know the current tariff rates across jurisdictions.** IEEPA rates change. EU trade remedy measures change. China retaliatory tariffs change. Verify the data is current the day before the demo. Check that the K17 regulatory signals are still accurate.
4. **Prepare "what if" pivots** including cross-jurisdiction pivots: "what about shipping to the EU instead?" or "what does this look like for China?" These are now strengths, not deflections — the system can show it.
5. **Practice non-linear navigation.** Rehearse jumping from any scene to any other scene. Practice recovering the narrative arc after a detour. The resequenceability is a design feature, but it only works if the presenter is fluent in it.
6. **Know the regulatory change signals.** Read all K17 signals before the demo. Be prepared to explain each one — why it matters, what the timeline is, what the impact would be. Engine 6 shows the data; the presenter provides the geopolitical context.

### What to Do If the System Is Down

If the LLM is unresponsive or the data service is unavailable, the presenter falls back to pre-computed results for the golden path products. This is the safety net. The pre-computed results must cover all three primary jurisdictions (US, EU, China). But it should never be the plan.

---

## Appendix: What This Proves That Theater Cannot

| Question the Audience Has | Theater Answers | Working System Answers |
|---|---|---|
| "Can you actually classify products?" | "Here's a pre-loaded example" | "Describe a product. Watch." |
| "Does the tariff math work?" | "Here are correct numbers on screen" | "Change the origin country. See the math change." |
| "Does it work outside the US?" | "We have a global vision" | "Switch the destination to the EU. Watch it recompute with a different tax regime." |
| "What about my products?" | "Imagine this for your business" | "Type your product. Name your market. Here's the result." |
| "How do you handle tariff changes?" | "The platform will track changes" | "Here are 7 active signals the system is tracking right now. This one affects 12 of your SKUs. Here's the cost impact." |
| "How do you handle errors?" | "Here's a designed error scenario" | "The system just got one wrong. Here's how it handles it." |
| "Is this real or a mockup?" | "It's a designed experience" | "It's a working system with a defined scope across US, EU, and China. Ask it anything within scope." |
| "How far along are you?" | "We have a vision and a demo" | "We have working engines across three jurisdictions. Classification, duty computation, compliance screening, regulatory change intelligence, exception analysis. Today." |
| "Who uses this?" | "Compliance teams and trade analysts" | "Let me show you three surfaces. This is what your compliance team sees. This is what your shipper sees. This is what your customer sees at checkout. Same engines. Three different experiences." |
| "What does it look like for a shipper?" | "We envision a simpler experience" | "Here — ready to ship, $56.81, one document needed. That's it. They never see a tariff code." |

The theater version creates desire: "I want this to exist." The working system creates belief: "This already exists — across markets — and it just needs to scale." The three surfaces create understanding: "I see who this is for, I see what each person experiences, and I see that the complexity is genuinely absorbed."

---

*This design specification is part of the FedEx Global Clearance Knowledge Base. It should be read alongside:*
- *[Vision: Clearance of the Future](../vision/clearance_of_the_future_vision.md)*
- *[Vision: FedEx-Accenture Partnership](../vision/fedex_accenture_clearance_platform_vision.md)*
- *[Conviction Strategy](conviction_strategy.md) — The broader experience portfolio this demo sits within*
- *[Build Specification](build_specification.md) — Detailed requirements, knowledge architecture, and build sequencing*
- *[Authenticity Audit](authenticity_audit.md) — Skeptical review of all data points and terminology*
- *[Presenter Notes](presenter_notes.md) — Voiceover guidance for areas of inherent complexity*
