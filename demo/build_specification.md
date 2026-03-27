# Clearance Intelligence System: Build Specification

## Feature Requirements, Knowledge Architecture, Quality Framework, and Build Sequencing

---

> **Purpose of this document:** To articulate — at the level of a well-described feature and capability — every piece that must be built, what knowledge it requires, how good it needs to be, how we handle what we can't yet do, and in what order we build it. This document serves two audiences: the team that will build it, and the stakeholders who need to understand what it takes to build the future.

> **What this is not:** A code plan. There are no file names, no framework choices, no API schemas. Those decisions come after this document is approved and understood.

---

## Table of Contents

1. [System Anatomy](#anatomy)
2. [Engine 1: Classification Intelligence — Detailed Requirements](#engine-1)
3. [Engine 2: Global Tariff & Landed Cost Engine — Detailed Requirements](#engine-2)
4. [Engine 3: Compliance Screening — Detailed Requirements](#engine-3)
5. [Engine 4: FTA Qualification — Detailed Requirements](#engine-4)
6. [Engine 5: Exception Analysis — Detailed Requirements](#engine-5)
7. [Engine 6: Regulatory Change Intelligence — Detailed Requirements](#engine-6)
8. [Orchestration Layer — How Engines Compose](#orchestration)
9. [Experience Layer — What the User Sees and Touches](#experience)
10. [Knowledge Collections — What We Must Assemble](#knowledge)
11. [Quality Framework — How Good Is Good Enough](#quality)
12. [Guardrails — What the System Must Never Do](#guardrails)
13. [Stub Strategy — The Line Between Authentic and Realistic](#stubs)
14. [Build Sequencing — What to Build First and Why](#sequencing)

---

<a name="anatomy"></a>
## 1. System Anatomy

The system has four layers. Every feature lives in one of them. The experience layer presents three fundamentally different application surfaces — each meeting a different actor in their natural context.

```
┌─────────────────────────────────────────────────────────┐
│  EXPERIENCE LAYER — Three Application Surfaces           │
│                                                          │
│  ┌─────────────┐ ┌──────────────┐ ┌──────────────┐     │
│  │  PLATFORM    │ │  SHIPPER     │ │  BUYER       │     │
│  │  INTELLIGENCE│ │  EXPERIENCE  │ │  EXPERIENCE  │     │
│  │             │ │              │ │              │     │
│  │  Command     │ │  Workflow-   │ │  Checkout-   │     │
│  │  center for  │ │  embedded    │ │  embedded    │     │
│  │  compliance, │ │  for sellers,│ │  for end     │     │
│  │  analysts,   │ │  logistics,  │ │  consumers.  │     │
│  │  brokers.    │ │  fulfillment.│ │  One price.  │     │
│  │  Full depth. │ │  3 questions.│ │  Duties incl.│     │
│  └─────────────┘ └──────────────┘ └──────────────┘     │
│                                                          │
│  Same engines. Same knowledge. Three experiences.        │
├─────────────────────────────────────────────────────────┤
│  ORCHESTRATION LAYER                                     │
│  How engines compose. Sequencing, error handling,        │
│  progressive streaming, session management.              │
│  All capabilities are modular and resequenceable.        │
├─────────────────────────────────────────────────────────┤
│  ENGINE LAYER                                            │
│  Six analytical engines. Each independently callable.    │
│  Each jurisdiction-aware. Each with its own knowledge    │
│  requirements and output patterns.                       │
├─────────────────────────────────────────────────────────┤
│  KNOWLEDGE LAYER                                         │
│  Structured data collections organized by jurisdiction.  │
│  Tariff schedules, surcharge programs, restricted party  │
│  lists, regulatory mappings, rulings, rules of origin,   │
│  tax regime templates. The HS code at 6 digits is the    │
│  global meta key linking knowledge across jurisdictions. │
└─────────────────────────────────────────────────────────┘
```

**Critical principle:** The knowledge layer is the foundation. An engine with perfect logic but wrong data produces wrong answers with confidence — the worst possible outcome. The knowledge layer must be correct before the engine layer is trusted.

**Critical principle:** The experience layer is not one application with three views. It is three different applications — different layouts, different language, different information density, different interaction models — sharing one analytical core. The Shipper never sees the Platform Intelligence interface. The Buyer never sees the Shipper interface. Each surface is designed for its actor's natural context and cognitive needs.

### 1.1 Global Scope

The system operates across the major global trade lanes. The scope is tiered:

| Tier | Jurisdictions | Depth | What This Means |
|---|---|---|---|
| **Tier 1 — Full depth** | US import | All 6 engines at full analytical capability | Complete tariff stack, compliance screening, FTA qualification, exception analysis |
| **Tier 2 — Meaningful analytical capability** | EU import, China import | Classification (6-digit global + major national subheadings), tariff computation (MFN + surcharges + trade remedies + VAT/import taxes), major compliance frameworks, key FTAs | Enough to compute meaningful landed costs, compare trade lanes, and model regulatory change scenarios |
| **Tier 3 — Framework demonstration** | Brazil, India, others | Classification (6-digit), MFN rate lookup, tax structure awareness, narrative coverage | Enough to show the architecture scales and to discuss with regional stakeholders |

**The architectural principle that enables this:** The HS (Harmonized System) code at 6 digits is universal across all WCO member countries (~180+ countries). This is the global meta key. National tariff schedules diverge at 7-10 digits, but the classification intelligence (Engine 1), the tariff lookup pattern, and the compliance screening approach are structurally identical across jurisdictions. Adding a new country is primarily a *knowledge loading* task, not an *engine development* task.

### 1.2 Currency

The system must handle multi-currency display. All internal computations use a base currency (USD). Landed costs are displayed in:
- The destination country's currency (EUR for EU, CNY for China, BRL for Brazil)
- USD equivalent for comparability
- Conversion rate and date are always shown

For the demo, exchange rates are loaded as a static snapshot with an "as of" date. In production, the system would integrate a live exchange rate feed.

---

<a name="engine-1"></a>
## 2. Engine 1: Classification Intelligence

### 2.1 Core Requirement

Given a natural-language product description, determine the most likely HTSUS classification (up to 10-digit statistical suffix level) and produce a visible reasoning chain that explains the classification path.

### 2.2 Inputs

| Input | Required | Format | Notes |
|---|---|---|---|
| Product description | Yes | Free text, 5-500 words | The primary signal. Must handle informal language ("coffee cup"), technical language ("borosilicate glass vessel"), and mixed ("travel mug with silicone lid, double-walled stainless steel") |
| Product images | No | JPEG/PNG | Secondary signal. Used to confirm material, shape, construction. Not required for classification but improves confidence when available. |
| Country of origin | No | ISO country code | Not used for classification itself (HS codes are origin-independent up to 6 digits) but used downstream for tariff computation. However, origin CAN affect classification at the 8-10 digit level in some cases (e.g., "product of a developing country" special provisions). |
| Declared value | No | USD amount | Not used for classification. Passed through to tariff calculation. |
| Additional context | No | Free text | "This is used in automotive manufacturing" or "this is a children's toy" — functional context that may resolve ambiguity. |

### 2.3 Outputs

| Output | Format | Notes |
|---|---|---|
| HTSUS code | 10-digit string (XXXX.XX.XX.XX) | Must be a valid code that exists in the current HTSUS. The system must NEVER output a code that doesn't exist in the schedule. |
| Reasoning chain | Structured steps | Each step: the level of the hierarchy (Section → Chapter → Heading → Subheading → Statistical suffix), the decision made, and the HTSUS text that supports it. |
| Confidence assessment | Categorical: High / Medium / Low | Not a percentage — percentages imply statistical precision we don't have. Categorical with clear definitions (see 2.7). |
| Ambiguity flags | List of alternative classifications | When confidence is Medium or Low, the system must show what other codes were considered and what would need to be true for each alternative. |
| Caveats | Free text | Conditions that could change the classification: "If this product contains >50% porcelain by weight, classification shifts to 6911." |

### 2.4 The Reasoning Chain — Detailed Requirements

The reasoning chain is not a summary. It is a step-by-step trace of how the classification was determined. It must follow the actual HTSUS classification methodology:

**Step 1: Material and Function Extraction.** Parse the product description to identify:
- Primary material (ceramic, steel, plastic, cotton, etc.)
- Function/use (tableware, automotive part, clothing, tool, etc.)
- Construction method (handcrafted, injection-molded, woven, etc.)
- Any special characteristics (food-safe, flame-retardant, wireless, etc.)

**Step 2: Section Determination.** The HTSUS has 22 Sections. The system must identify which Section the product falls into, citing the Section description. This is the broadest classification step.

**Step 3: Chapter Determination.** Within the Section, identify the Chapter (2-digit). The system must check the Chapter Notes — many Chapter Notes contain exclusions ("This chapter does not cover...") that redirect products elsewhere. The system must show that it checked relevant exclusions.

**Step 4: Heading Determination (4-digit).** Within the Chapter, identify the Heading. This is where the General Rules of Interpretation (GRIs) become critical:
- GRI 1: Classification by the terms of the headings and Section/Chapter Notes
- GRI 2(a): Incomplete or unfinished articles classified as complete articles if they have the essential character
- GRI 2(b): Mixtures and combinations — composite goods
- GRI 3: When goods appear classifiable under two or more headings — most specific description, essential character, or last in numerical order
- GRI 4: Most akin (fallback)
- GRI 5: Cases, containers, packing materials
- GRI 6: Sub-classification within a heading follows the same GRI logic

The system must cite which GRI it applied and why. For most products, GRI 1 is sufficient. But when ambiguity exists (a product with both metal and plastic components, a set containing multiple items), the GRI logic becomes critical and the system must work through it visibly.

**Step 5: Subheading Determination (6-digit, then 8-digit).** Narrow within the Heading. The 6-digit level is internationally harmonized (same worldwide). The 8-digit level is US-specific. The system must navigate the indented subheading tree correctly — the HTSUS uses a hierarchical indentation system where sub-subheadings are scoped by their parent.

**Step 6: Statistical Suffix (10-digit).** The final two digits are statistical only (don't affect duty rates) but are required for filing. These are often based on specific characteristics (size, material composition, intended use). The system should attempt the statistical suffix but flag it as lower-confidence than the 8-digit classification.

### 2.5 Knowledge Required

**Primary: The full HTSUS**
- All 99 chapters with legal heading text
- All Section Notes (22 sections)
- All Chapter Notes (some chapters have extensive notes — Chapter 39 (plastics) has 11 notes with sub-notes; Chapter 61-62 (apparel) have complex fiber composition rules)
- All General Notes (including GRIs, which are in General Note 1)
- Rate columns: General (MFN), Special (FTA/preference programs), Column 2 (non-NTR)
- The hierarchical indentation structure must be preserved — this is how sub-classification works

**Secondary: Classification reference examples**
- A curated set of 500-1,000 product → HS code mappings for common products
- Sourced from CBP CROSS rulings (verified classifications)
- Organized by chapter for retrieval
- Purpose: Give the LLM concrete examples to anchor its reasoning, not just the abstract legal text

**Tertiary: Chapter-specific guidance**
- Explanatory Notes from the World Customs Organization (WCO) — these are the internationally recognized interpretation guides for HS codes
- Not legally binding in the US but widely used by CBP and trade professionals
- Available for purchase from the WCO (not freely available — licensing consideration)
- If WCO Explanatory Notes are not available, CBP's Informed Compliance Publications (ICPs) serve a similar purpose and are free

### 2.6 Edge Cases and Nuances

**Composite goods (GRI 3).** A product made of two materials — say a mug with a ceramic body and a stainless steel handle. Which material determines classification? GRI 3(b) says "essential character." For a mug, the body is the essential character (it holds liquid), so the ceramic classification takes precedence. The system must identify composite goods and apply GRI 3 reasoning. This is one of the hardest classification problems and the system should flag it explicitly.

**Sets put up for retail sale (GRI 3(b)).** A "coffee set" containing a mug, a saucer, and a spoon — three different products in one package. Classified by the component that gives the set its essential character. The system must recognize sets and reason about essential character.

**Parts and accessories.** "A replacement screen for a laptop" — is this classified as a screen (Chapter 90: optical elements) or as a laptop part (Chapter 84: computer parts)? Section XVI Note 2 has specific rules for parts of machines. The system must know these rules.

**"Other" classifications.** Many HTSUS subheadings end with "Other" as a catch-all. The system should only classify into "Other" if it has exhausted the named subheadings and confirmed the product doesn't fit any of them. Falling into "Other" too readily is a sign of lazy classification.

**Products that genuinely span chapters.** A smartwatch with health monitoring: Chapter 85 (electronic apparatus), Chapter 91 (clocks and watches), or Chapter 90 (medical instruments)? There are CBP rulings on this specific question. The system should flag the ambiguity and ideally cite relevant precedent.

**Products requiring physical testing.** Some classifications depend on measurable characteristics:
- Textiles: fiber composition by weight (>50% cotton = Chapter 52 vs. >50% polyester = Chapter 54)
- Metals: tensile strength, carbon content
- Chemicals: molecular formula, CAS number
- Beverages: alcohol content by volume

The system cannot perform physical tests. When a classification depends on a measurable characteristic, the system must ask: "Classification depends on [characteristic]. If [X], the code is [A]. If [Y], the code is [B]. Please provide the [characteristic] or select the applicable classification."

**Textile complexity.** Chapters 50-63 are notoriously difficult. Classification depends on fiber composition (by weight), construction (woven vs. knitted vs. nonwoven), gender (men's vs. women's — with different chapters), and end use. The system should flag textile classifications as requiring additional specificity and ask clarifying questions rather than guessing.

### 2.7 Confidence Assessment

Confidence is not a number — it is a categorical assessment with defined criteria.

| Level | Criteria | What the System Shows | What It Means for the Demo |
|---|---|---|---|
| **High** | Product clearly matches one heading/subheading. No ambiguity in material, function, or construction. Section/Chapter Notes do not create exceptions. | "Classification: HS 6912.00.48 — Confidence: High" | The presenter can state this classification confidently. |
| **Medium** | Product matches one heading but subheading selection involves interpretation. OR the description is missing information that could change the classification. | "Classification: HS 8518.21 — Confidence: Medium. Note: If this device includes FM radio reception, classification shifts to 8527.92." | The presenter explains the nuance: "The system gives this Medium confidence because the description doesn't clarify whether it has radio functionality." |
| **Low** | Product could plausibly fall in multiple headings under different GRI interpretations. OR the description is too vague. OR physical characteristics are needed that aren't provided. | "Possible classifications: 8518.21 (loudspeaker), 8527.92 (radio), 8517.62 (telecom apparatus). Additional information needed." | The presenter uses this as a feature: "This is a hard classification. The system identified three possibilities and told you what information would resolve it. That's better than guessing." |

### 2.8 Performance Requirements

| Metric | Target | Rationale |
|---|---|---|
| Time to first token of reasoning chain | < 2 seconds | The audience must see the system start thinking immediately. Dead time feels like loading. |
| Time to complete classification | < 8 seconds typical, < 15 seconds maximum | Longer is acceptable for complex products IF the reasoning chain is streaming and the audience can see progress. |
| Time for a simple product (clear chapter/heading) | < 5 seconds | Most demo products should be in this range. |

**Streaming is mandatory.** The reasoning chain must render progressively — section determination first, then chapter, then heading, etc. The audience sees the system working step by step. This transforms wait time into information.

### 2.9 Guardrails

1. **No hallucinated HS codes.** Every HS code output must be validated against the loaded HTSUS. If the LLM generates a code that doesn't exist (e.g., 6912.00.99 when only .48 exists), the system must catch this and either correct it or flag it as unresolvable. This is a hard requirement — displaying a nonexistent HS code to a trade professional is an instant credibility kill.

2. **No fabricated rulings or legal citations.** If the system cites a CBP ruling, that ruling must exist in the loaded CROSS database. No invented ruling numbers. No paraphrased ruling text presented as a quote. If the system needs to reference a ruling it doesn't have, it says "CBP rulings on this product type should be consulted" without fabricating a specific citation.

3. **No confident wrong answers.** If the system is uncertain, it must say so. A High confidence classification that turns out to be wrong is far more damaging than a Medium confidence classification with the correct answer as an alternative. The confidence calibration must be conservative.

4. **No classification without reasoning.** Every output must include the reasoning chain. The system never outputs a bare HS code without showing why. This is both a quality measure (forces the LLM to reason) and a trust measure (the audience can verify the logic).

5. **Textile and chemical caveats.** For Chapters 28-29 (chemicals), 39 (plastics), and 50-63 (textiles), the system must include a standing caveat: "Classification in this chapter may depend on physical/chemical characteristics not determinable from the description alone. Verify [specific characteristic] before filing." This is not a weakness — it is professional practice.

---

<a name="engine-2"></a>
## 3. Engine 2: Global Tariff & Landed Cost Engine

### 3.1 Core Requirement

Given an HS code, country of origin, and destination country, compute the total effective duty rate and landed cost by identifying and stacking all applicable tariff programs, import taxes, and fees for the destination jurisdiction. Show each component's contribution transparently.

The engine is jurisdiction-aware: it reads a **Tax Regime Template** for the destination country that defines which tax layers apply, in what order, with what base formula. This means the engine's computation logic is generic — adding a new country is a knowledge/configuration task, not an engine logic change (with one exception: self-referencing/cascading tax regimes require an algebraic solver — see Section 3.13).

### 3.1.1 Tax Regime Pattern Architecture

Every country's import tax regime follows one of two computational patterns:

**Pattern A: Additive Stacking** (US, EU, China, India, UK, Japan, South Korea, most of the world)

Each tax layer computes on a defined base. Layers are independent — no tax includes itself in its own base.

```
Layer_1 = Rate_1 × Base_1     (e.g., MFN duty = 9.8% × customs_value)
Layer_2 = Rate_2 × Base_2     (e.g., Section 301 = 25% × customs_value)
...
Layer_N = Rate_N × Base_N     (e.g., VAT = 19% × (customs_value + duty))
Total = customs_value + Σ(all layers) + fees
```

**Pattern B: Self-Referencing/Cascading** (Brazil, some other Latin American jurisdictions)

Some tax layers are computed on a base that includes themselves. This creates a circular reference that requires algebraic grossup.

```
ICMS = Rate × (customs_value + duty + IPI + PIS + COFINS + ICMS)
Solving: ICMS = Rate × (all_other_components) / (1 - Rate)
```

The engine handles both patterns. Pattern A is simple sequential computation. Pattern B triggers the grossup solver. The Tax Regime Template for each country specifies which pattern each layer uses.

### 3.1.2 Tax Regime Templates

Each destination country is defined by a template: an ordered list of tax layers with rate source, base formula, and stacking type.

**US Template:**

| Order | Layer | Rate Source | Base | Stacking | Notes |
|---|---|---|---|---|---|
| 1 | MFN Duty | K2-US (HTSUS rates) by HS code | customs_value | additive | Ad valorem, specific, or compound |
| 2 | Section 301 | K3-US (301 lists) if China origin | customs_value | additive | List-specific rate |
| 3 | Section 232 | K4-US (232 scope) if steel/aluminum | customs_value | additive | Country exemptions apply |
| 4 | IEEPA Reciprocal | K5-US (IEEPA schedule) by country | customs_value | additive | Volatile — show "as of" date |
| 5 | AD/CVD | K6-US (active orders) by HS + country | customs_value | additive | Manufacturer-specific rates |
| 6 | MPF | K16-US (fee schedule) by entry type | customs_value | additive (fee) | Formal vs. informal per §3.9 |
| 7 | HMF | 0.125% if ocean transport | customs_value | additive (fee) | Air exempt |
| 8 | State/Use Tax | K13-US (state rates) by state | per state rules (§3.11) | additive | |

**EU Template:**

| Order | Layer | Rate Source | Base | Stacking | Notes |
|---|---|---|---|---|---|
| 1 | Third Country Duty (MFN) | K2-EU (TARIC) by CN code | customs_value | additive | EU Combined Nomenclature 8-digit |
| 2 | Additional Duties (retaliation) | K3-EU (EU surcharges) if applicable origin | customs_value | additive | e.g., retaliation on US steel/aluminum |
| 3 | Safeguard Duties | K4-EU (safeguard scope) if steel | customs_value | additive | Quota-based — may not apply within quota |
| 4 | AD/CVD | K6-EU (EU AD/CVD orders) by CN + country | customs_value | additive | EC investigation-based |
| 5 | VAT | K13-EU (VAT rates) by member state + product | customs_value + all duties | additive | Standard rate (19-27%) or reduced. Reclaimable for B2B — note this in output. |

**China Template:**

| Order | Layer | Rate Source | Base | Stacking | Notes |
|---|---|---|---|---|---|
| 1 | MFN Duty | K2-CN (China tariff) by HS code | customs_value | additive | Published by Customs Tariff Commission |
| 2 | Retaliatory Tariff | K3-CN (retaliation lists) if US/other origin | customs_value | additive | Multiple rounds; verify current rates |
| 3 | AD/CVD | K6-CN (MOFCOM orders) by HS + country | customs_value | additive | |
| 4 | Consumption Tax | K13-CN (consumption tax rates) if luxury/sin goods | customs_value + duty | additive | Alcohol, tobacco, cosmetics, autos, jewelry |
| 5 | VAT | K13-CN (VAT rates): 13% standard / 9% reduced | customs_value + duty + consumption_tax | additive | |

**Brazil Template (Tier 3 — demonstrates cascading pattern):**

| Order | Layer | Rate Source | Base | Stacking | Notes |
|---|---|---|---|---|---|
| 1 | Import Duty (II) | K2-BR (NCM rates) by NCM code | customs_value | additive | Mercosur CET base rates |
| 2 | IPI | K3-BR (IPI table) by NCM code | customs_value + import_duty | additive | |
| 3 | PIS | 2.1% (standard imports) | all components including self | **self-referencing** | Requires grossup solver |
| 4 | COFINS | 9.65% (standard imports) | all components including self | **self-referencing** | Requires grossup solver |
| 5 | ICMS | K4-BR (state ICMS) by state | all components including self | **self-referencing** | 17-25% depending on state and product |

**Why this architecture matters:** Adding Japan takes roughly two days of data work — define the template (MFN + AD + consumption tax + VAT), load the tariff schedule, and the engine computes. No code changes. The template IS the specification of how that country's import taxes work. This is how a global platform scales to 50+ jurisdictions without 50× the engineering.

### 3.2 Inputs

| Input | Required | Format | Notes |
|---|---|---|---|
| HTSUS code | Yes | 8 or 10 digit | Can come from Engine 1 or be entered directly. Must be a valid code. |
| Country of origin | Yes | ISO country code | The country where the goods were manufactured (not transshipped from). |
| Customs value (declared value) | Yes | USD amount | The transaction value of the goods per 19 USC 1401a. For the demo scope, this is the invoice/retail price. See Section 3.10 for customs valuation scope. The UI label should be "Customs Value" or "Declared Value" — not "Product Value." |
| Shipping cost | No | USD amount | Used for landed cost calculation. If not provided, the system should either estimate or omit shipping from the total. |
| Destination state | No | US state code | Used for state/local tax in landed cost. If not provided, exclude tax or show a representative rate. See Section 3.11 for state tax base rules. |
| Mode of transport | No | Air / Ocean / Ground | Affects HMF applicability (ocean only). Default to Air (express carrier context). |
| Quantity and unit | No | Numeric + unit type | Required for specific duties (e.g., "6.3¢ per liter"). If not provided and the duty is specific, the system displays the rate without computing a dollar amount. |

### 3.3 Outputs

| Output | Format | Notes |
|---|---|---|
| MFN duty rate | Percentage or specific rate | Some duties are ad valorem (percentage), some are specific (per unit — e.g., "6.3¢ per liter"), and some are compound ("5% + 2¢/kg"). The system must handle all three types. |
| Section 301 rate and list | Percentage + list identifier | "25% (List 1)" or "7.5% (List 4a)" or "N/A — product not on any 301 list" or "N/A — non-China origin" |
| Section 232 rate | Percentage or N/A | "25% (steel)" or "10% (aluminum)" or "N/A — product not covered" or "N/A — [country] exempt" |
| IEEPA reciprocal rate | Percentage | Country-specific rate. "60% (China)" or "20% (EU, current rate)" or "0% (USMCA partner, exempt)" |
| AD/CVD status | Active/None + rate range | "AD order active: all-others rate 24.5%" or "No active AD/CVD orders" |
| FTA availability | Available/Not available | "USMCA rate available: 0% (if rules of origin met)" — this is a flag, not a determination. Determination requires Engine 4. |
| Total effective rate | Percentage | Sum of all applicable ad valorem components. For specific/compound duties, show both the rate and the dollar amount. |
| Entry type | Formal / Informal | Determined by declared value relative to the $2,500 threshold. See Section 3.9. |
| MPF | Dollar amount | Merchandise Processing Fee. Amount depends on entry type — see Section 3.9. The system must compute MPF from the entry type and current CBP fee schedule (K16), not from hardcoded values. |
| HMF | Dollar amount or N/A | Harbor Maintenance Fee: 0.125% of declared value for ocean shipments. Not applicable for air. |
| State/local tax | Dollar amount or N/A | If destination state is provided. Computed per Section 3.11 rules. |
| Total landed cost | Dollar amount | Customs value + duty + fees + shipping + tax. The one number. |

### 3.4 Duty Rate Types — Nuances

**Ad valorem duties** are straightforward: a percentage of declared value. Most duties are ad valorem.

**Specific duties** are per-unit charges: "6.3¢ per liter" for wine, "3.2¢ per kg" for certain chemicals. The system must ask for quantity/unit if the duty is specific and only a value was provided. If quantity is not available, the system should show the rate description without computing a dollar amount: "Duty: 6.3¢/liter — provide quantity to compute amount."

**Compound duties** combine ad valorem and specific: "5% ad valorem + 2¢/kg." Both components must be shown.

**Tariff-rate quotas (TRQs).** Some products have a lower duty rate up to a quota volume, and a higher rate above the quota. Example: certain dairy products, sugar. The system should note when a TRQ exists: "Within-quota rate: 5%. Over-quota rate: 25%. TRQ status: [if available] or 'check with CBP for current quota fill.'" For the demo, acknowledging TRQs exist is sufficient; real-time quota fill status is out of scope.

**Temporary duty suspensions and modifications.** The Miscellaneous Tariff Bill (MTB) temporarily suspends or reduces duties on specific products. These are published as temporary modifications to Chapter 99 of the HTSUS. The system should check Chapter 99 provisions if loaded — but this is a lower-priority data set due to its specificity and temporary nature.

### 3.5 Tariff Program Stacking Logic — Detailed

The programs stack. This is the core intellectual problem. The system must get the stacking order and applicability right.

**Does Section 301 apply?**
- Is the country of origin China? If no → N/A
- Is the HS code on any 301 list (Lists 1, 2, 3, 4a, 4b)? If no → N/A
- If yes → which list and what rate? Note: rates have been modified by subsequent proclamations. The system must use the *current* rate for each list, not the originally published rate.
- 301 rates apply to BOTH the MFN duty and are additive (they are separate line items, not modifications of MFN).

**Does Section 232 apply?**
- Is the product in scope? 232 covers steel (primarily Chapters 72, 73) and aluminum (Chapter 76) and derivative products. The product scope is defined by specific HS codes in the Presidential proclamation.
- Is the country exempt? Some countries have negotiated exemptions or alternative quotas. The system must check the country against 232 exemptions.
- 232 rate: 25% for steel, 10% for aluminum (these are the base rates — modifications may apply by country).
- 232 and 301 are independently applicable. A Chinese steel product could face MFN + 232 + 301 + IEEPA.

**Does IEEPA reciprocal apply?**
- The IEEPA tariffs are country-specific. Each country has a rate published in the executive order.
- USMCA partners (Mexico, Canada) may be exempt or subject to a different rate. The IEEPA-USMCA interaction has changed multiple times. Use the currently effective treatment.
- IEEPA rates are additive on top of MFN + 301 + 232.

**Is there an AD/CVD order?**
- AD/CVD is product-specific AND country-specific AND often manufacturer-specific.
- The ITC maintains a list of active orders. Each order covers specific HS codes from specific countries.
- Rates are company-specific: the exporter may have an individually determined rate, or they may fall under the "all-others" rate.
- For the demo, we show: "AD/CVD order active for [product] from [country]. All-others rate: [X]%. Manufacturer-specific rates may differ."
- AD/CVD stacks on top of everything else. A Chinese product can face: MFN + 301 + IEEPA + AD.

**Are FTA preferences available?**
- If the origin country has an FTA with the US (USMCA for Mexico/Canada, KORUS for South Korea, US-Australia FTA, etc.), preferential rates may be available.
- The system flags this as "available" — Engine 4 determines if the product qualifies.
- For USMCA: the preferential rate is typically 0% but not always. Some products have phased reductions. Check the USMCA rate column in the HTSUS.

### 3.6 Edge Cases

**Zero MFN + tariff surcharges.** Many electronics have 0% MFN duty (e.g., laptops, routers, smartphones). But they may still face 301 + IEEPA. The system must show this: "MFN: 0.0% — but total effective rate: 85% due to 301 + IEEPA." This is one of the most eye-opening results for the audience and it must be calculated correctly.

**De minimis value thresholds.** With the suspension of Section 321, all shipments now require entry regardless of value — but not necessarily *formal* entry. Shipments valued at ≤$2,500 generally use *informal* entry (Type 11) with a flat MPF ($2.69 automated, FY2026), while shipments above $2,500 use formal entry (Type 01) with percentage-based MPF. The system must correctly determine the entry type and apply the corresponding fee structure. See Section 3.9 for full entry type logic.

**Country of origin vs. country of export.** The system asks for country of origin, not country of export. Goods manufactured in China but shipped via Hong Kong are still China-origin for tariff purposes. The system should not have a "Hong Kong" option that implies circumvention. For the demo, this is a knowledge point the presenter can make if asked.

**Transshipment and origin determination.** The system does not determine origin — it accepts it as an input. Origin determination (substantial transformation, marking rules) is a separate analytical problem and is explicitly out of scope. The system can note: "Origin as provided. Origin determination under 19 CFR 134 is not performed by this engine."

### 3.7 Performance Requirements

| Metric | Target | Rationale |
|---|---|---|
| Time to compute full tariff stack | < 1 second | This is a lookup + math operation. There is no LLM in the loop. It should be near-instant. |
| Time to recalculate on input change | < 500ms | When the presenter changes the origin country, the tariff stack should update immediately. |

This engine has no streaming requirement. It's fast enough to appear all at once.

### 3.8 Accuracy Requirement

**This engine must be 100% accurate for loaded data.** There is no tolerance for tariff rate errors. A wrong MFN rate, a missing 301 list, or incorrect IEEPA applicability will be caught by any trade professional in the room. This is different from Engine 1 (classification), where some uncertainty is inherent and expected.

Accuracy depends entirely on the knowledge layer. If the data is loaded correctly, the computation is deterministic. The risk is in data preparation, not in engine logic.

**Validation protocol:** Before any demo, a trade compliance SME must run 50+ product/country combinations through the engine and verify every rate against the current HTSUS and current tariff programs. This validation must be repeated whenever tariff data is updated.

### 3.9 Entry Type Determination

**This is a critical capability that was missing from the initial specification.** The entry type determines how a shipment is filed with CBP and directly affects the MPF calculation.

#### Entry Types in Scope

| Entry Type | Code | When Used | MPF |
|---|---|---|---|
| **Formal Entry** | Type 01 | Declared value > $2,500, or certain regulated goods regardless of value | 0.3464% of declared value. Minimum $33.58, maximum $651.50 (FY2026). Adjusted annually for inflation. |
| **Informal Entry** | Type 11 | Declared value ≤ $2,500 (most low-value consumer parcels post-de minimis) | Flat fee: $2.69 if processed via automated system; $12.09 if CBP-prepared document (FY2026). |

**Note on deprecated entry types:** Type 86 (Section 321 + advance data) was used for de minimis shipments. With Section 321 suspended, Type 86 is deprecated. Low-value parcels that formerly entered duty-free under Type 86 now require either informal or formal entry with full data.

#### Determination Logic

```
IF declared_value > $2,500:
    entry_type = FORMAL
ELSE IF product has AD/CVD order OR requires quota/visa:
    entry_type = FORMAL (regardless of value)
ELSE:
    entry_type = INFORMAL
```

**Why this matters for the demo:**
- Maria's $38 ceramic mug → Informal entry (Type 11), MPF = $2.69
- David's $50,000 auto parts shipment → Formal entry (Type 01), MPF = 0.3464% × $50,000 = $173.20
- A $38 parcel that happens to be subject to an AD/CVD order → Formal entry despite low value

The system must show the entry type in the output and explain why that type applies. This is one of the most common questions from importers in the post-de minimis world.

#### Related Concepts the Presenter Must Understand

**Importer of Record (IOR).** The system does not determine the IOR — this is a business/legal decision between the shipper, buyer, and broker. But the presenter should be prepared for the question. For the Maria scenario (Shopify DTC seller shipping to a US consumer), the IOR is typically the buyer (consumer) unless the seller operates DDP (Delivered Duty Paid). The system should note the assumed Incoterm if one is provided, but IOR determination is explicitly out of scope.

**MPF fee schedule currency.** The MPF minimums and maximums are adjusted annually by CBP for inflation, effective October 1 each fiscal year. The system must source MPF amounts from the knowledge layer (K16) with an effective date, not from hardcoded values. The "as of" date must be displayed.

### 3.10 Customs Valuation Scope

**What "Declared Value" means in this system:** For the demo, declared value equals transaction value — the price actually paid or payable for the goods when sold for export to the US (19 USC 1401a). In practice, this is the invoice price.

**What the demo does NOT handle (and should say so):**
- Assists (tools, dies, molds, engineering provided by the buyer to the seller) — these must be added to transaction value
- Royalties and license fees related to the imported goods
- Proceeds of subsequent resale accruing to the seller
- Buying commissions vs. selling commissions (different treatment)
- Related-party transactions (transfer pricing adjustments)

**For the demo scope:** The user enters a price. The system uses it as transaction value. No adjustments. This is correct for:
- DTC consumer purchases (Maria's mug for $38 — the Shopify price IS the transaction value)
- Simple commercial transactions where the invoice reflects the full price

**For the presenter:** If asked "does this handle assists/royalties?", the answer is: "In production, the system would capture valuation adjustments per 19 USC 1401a(b). For this demo, we use invoice price as transaction value — which is correct for the vast majority of simple commercial transactions."

### 3.11 State/Local Tax Base Specification

**The problem:** State sales/use tax on imports varies by jurisdiction in both rate and taxable base. There is no single national rule.

**For the demo, the system must define the taxable base explicitly:**

| Base Component | Included? | Rationale |
|---|---|---|
| Customs value (product price) | Yes — always | All states tax the sale price |
| Import duty | Varies by state | Most states do NOT include import duty in the taxable base. The demo should EXCLUDE duty from the base unless the destination state specifically includes it. |
| Shipping/delivery charges | Varies by state | Some states tax shipping if it's not separately stated; others exempt it. The demo should follow the specific state's rule if loaded in K13. |
| MPF and other CBP fees | No | Government fees are generally not included in the taxable base |

**For the demo:** The default taxable base is *customs value only* (not duty, not shipping, not fees). This is a conservative and defensible approach. K13 should contain, for each state:
- The combined state + average local rate
- Whether shipping is taxable in that state
- Whether duty is included in the taxable base (rare but some states do this)

**The system must show its work:** "IL State/Use Tax: 7.56% of $38.00 (customs value) = $2.87" — showing exactly what the rate is applied to. Never display a tax amount without showing the base.

**Why this is inherently imperfect:** State/local tax is a genuinely complex area. There are 11,000+ tax jurisdictions in the US. The system uses a representative rate per state (K13), not a real-time tax API. This is sufficient for the demo but the presenter should acknowledge the simplification. See Presenter Notes for voiceover guidance.

### 3.12 Guardrails

1. **No rates from memory.** The engine must look up every rate from its loaded data. It never "remembers" that steel from China is 25% — it checks the 232 product list, confirms the HS code is covered, confirms China is not exempt, and applies the published rate. This ensures accuracy as rates change.

2. **Show "as of" dates.** Every tariff program should show the effective date of the data: "IEEPA rates as of [date]." This manages the volatility problem honestly.

3. **Handle unknown HS codes gracefully.** If the HS code is not in the loaded HTSUS data (either it's invalid or the data is incomplete), the system says: "HS code [X] not found in current schedule. Verify the code or use the classification engine to determine the correct code."

4. **No arithmetic display without showing the work.** The total effective rate must be the visible sum of itemized components. No "black box" totals.

5. **No hardcoded fee amounts.** MPF minimums, maximums, and informal entry fees must be sourced from K16 (CBP Fee Schedule), not hardcoded in the engine logic. These amounts change annually.

6. **Entry type must be displayed.** Every landed cost computation must show whether the calculation is based on formal or informal entry and explain why. The audience will include people who understand the distinction.

---

<a name="engine-3"></a>
## 4. Engine 3: Compliance Screening

### 4.1 Sub-Engine 3A: PGA Jurisdiction Mapping

#### Core Requirement

Given an HTSUS code, identify all Partner Government Agencies that have jurisdiction and what each requires for entry.

#### The Knowledge Problem

There are approximately 48 PGAs that can assert jurisdiction over imports. The most commonly encountered:

| Agency | Jurisdiction | Common HS Ranges | Data Required |
|---|---|---|---|
| FDA | Food, drugs, cosmetics, medical devices, tobacco | Chapters 2-22 (food), 30 (pharmaceuticals), 33 (cosmetics), 38 (various), 90 (medical devices) | Prior Notice (food), product codes, facility registration, drug listing |
| USDA/APHIS | Animals, plants, agricultural products | Chapters 1-14, some 44 (wood), some 06 (live plants) | Phytosanitary certificate, import permit, APHIS inspection |
| CPSC | Consumer products, children's products | Chapters 39 (plastics), 63-65 (textiles/headwear), 94 (furniture), 95 (toys) | Certificate of Compliance, testing to applicable standard (ASTM, CPSIA) |
| EPA | Chemicals, vehicles, engines, pesticides | Chapters 28-29 (chemicals), 38 (pesticides), 84 (engines), 87 (vehicles) | TSCA certification, EPA engine/vehicle import form |
| TTB | Alcohol, tobacco | Chapters 22 (beverages), 24 (tobacco) | TTB import permit, COLA (Certificate of Label Approval) |
| FCC | Electronic devices | Chapters 84-85 (electronics), 90 (instruments) | FCC equipment authorization, FCC ID |
| ATF | Firearms, ammunition, explosives | Chapter 93 (arms), some 36 (explosives) | ATF import permit |
| DOT/PHMSA | Hazardous materials, vehicles | Chapters 27-29 (chemicals), 36 (explosives), 87 (vehicles) | Hazmat declaration, FMVSS compliance |
| NHTSA | Motor vehicles, motor vehicle equipment | Chapter 87 | HS-7 Declaration (DOT form) |
| Fish & Wildlife | Wildlife products, CITES species | Various chapters involving animal or plant products | CITES permit, FWS Declaration |

#### Nuances

**FDA jurisdiction is the most complex and most common.** FDA asserts jurisdiction over an enormous range of products — not just food and drugs. Medical devices (Chapter 90), cosmetics (Chapter 33), radiation-emitting products (certain electronics), and even some general consumer products. The mapping from HS code to FDA product code is not one-to-one; a single HS code can trigger different FDA requirements depending on the specific product.

**Multiple PGA overlaps.** A single product can trigger multiple PGAs. An imported alcoholic beverage: FDA (food), TTB (alcohol), USDA (if containing agricultural products from regulated countries). The system must show all applicable agencies, not just the first match.

**PGA requirements vary by product, not just by agency.** FDA's requirements for food (Prior Notice, facility registration, FSMA compliance) are different from FDA's requirements for cosmetics (facility registration, product listing) and different again for medical devices (premarket notification/approval, establishment registration). The system must map to the specific requirement, not just the agency.

**"May require" vs. "requires."** Some PGA triggers are definite (all food products require FDA Prior Notice) and some are conditional (certain electronic products may require FCC authorization depending on whether they emit radiofrequency energy). The system should distinguish: "Requires: FDA Prior Notice" vs. "May require: FCC equipment authorization (if device emits RF energy above threshold)."

#### Output Format

```
PGA Screening Results for HS [code]:

REQUIRED:
• FDA — [specific requirement]. Data needed: [specific data elements]

CONDITIONAL:
• FCC — Required if device emits RF energy. Confirm with product specifications.

NOT APPLICABLE:
• CPSC, EPA, TTB, ATF, USDA — No jurisdiction for this HS code
```

#### Quality Requirement

The PGA mapping must be correct for the most common agency-HS relationships. A system that tells a food importer "no PGA jurisdiction" for a food product, or that fails to flag FDA for a cosmetic, fails visibly. Target: 95%+ accuracy on the top 200 most commonly imported product categories.

Achieving 100% coverage of all PGA-HS combinations is out of scope — there are thousands of specific cases. The system should err on the side of over-flagging (false positives are better than false negatives in compliance).

### 4.2 Sub-Engine 3B: Denied Party Screening

#### Core Requirement

Given an entity name (person, company, or organization), screen against US government restricted party lists and report any matches with match quality assessment.

#### Lists to Screen Against

| List | Maintained By | Size | Update Frequency | Format |
|---|---|---|---|---|
| SDN List (Specially Designated Nationals) | OFAC (Treasury) | ~12,000 entries | Multiple times per month | XML, CSV — publicly downloadable |
| Consolidated Sanctions List | OFAC | Superset of SDN + other programs | Same | XML, CSV |
| Entity List | BIS (Commerce) | ~700 entries | Periodically | Federal Register |
| Denied Persons List | BIS | ~500 entries | Periodically | Published list |
| Unverified List | BIS | ~200 entries | Periodically | Published list |
| UFLPA Entity List | DHS/FLETF | ~70 entries | Per determination | Published list |
| Non-SDN Chinese Military-Industrial Complex Companies | OFAC | ~60 entries | Periodically | Published list |

#### Matching Logic — This Is Where Quality Lives

Exact string matching is insufficient. The system must handle:

**Name variations:**
- "Huawei Technologies Co., Ltd." = "Huawei Technologies Company Limited" = "HUAWEI TECHNOLOGIES"
- Abbreviations: "Co." = "Company", "Ltd." = "Limited", "Inc." = "Incorporated"
- Transliterations from non-Latin scripts: Chinese, Russian, Arabic names can have multiple romanization variants

**Fuzzy matching thresholds:**
- Exact match (score 100): Report as definite match
- High similarity (score 90-99): Report as probable match — "Entity name closely matches [listed entity]. Verify."
- Moderate similarity (score 75-89): Report as potential match — "Entity name has similarities to [listed entity]. Manual review recommended."
- Below 75: Do not report — too many false positives become noise

**Address matching (secondary):**
- Some list entries include addresses. If the screened entity includes an address, cross-reference against listed addresses for entities with moderate name similarity.
- Address matching is a secondary signal, not a primary one.

**Alias handling:**
- The SDN list includes "also known as" (AKA) entries. The system must screen against all aliases, not just the primary name.

#### Output Format

```
Restricted Party Screening: [entity name]

✓ No matches found on any screened list
  Lists screened: OFAC SDN, OFAC Consolidated, BIS Entity List,
  BIS DPL, BIS Unverified, UFLPA Entity List, NS-CMIC
  As of: [date]
```

or

```
⚠ MATCH FOUND

Entity: [screened name]
Matches: [list entry name] on [list name]
Match confidence: [High/Probable/Potential]
Basis: [name match / alias match / name + address match]
Listed entry details: [listed reason, program, effective date]

Action required: Do not proceed with transaction until match is resolved.
```

#### Performance Requirement

| Metric | Target | Rationale |
|---|---|---|
| Single entity screening | < 2 seconds | Must feel instant in the demo. Pre-indexed lists enable this. |
| Batch screening (dashboard scenario) | < 10 seconds for 100 entities | For the dashboard pre-population. |

#### Guardrails

1. **Always screen against all loaded lists.** Never partial screening. The output must name every list that was checked.
2. **Never report "clear" without stating which lists were checked and the data date.** "No matches found" without context is meaningless.
3. **False positive tolerance.** It is better to flag a potential match that isn't real (user resolves it in 30 seconds) than to miss a real match (legal and regulatory exposure). The matching thresholds should be tuned for sensitivity over specificity in the demo.

### 4.3 Sub-Engine 3C: UFLPA Risk Assessment

#### Core Requirement

For products with a nexus to the Xinjiang Uyghur Autonomous Region (XUAR) or to UFLPA Entity List companies, assess the forced labor risk and outline the evidence required to overcome a rebuttable presumption.

#### Inputs

| Input | Source | Notes |
|---|---|---|
| HS code | Engine 1 or direct input | Used to check against UFLPA enforcement priority sectors |
| Country of origin | User input | Primary flag: China. Secondary: any country with known XUAR supply chain exposure |
| Entities (manufacturer, supplier) | User input or pre-loaded | Screened against UFLPA Entity List (via Engine 3B) |
| Product type | Extracted from description | Matched against enforcement priority sectors |

#### UFLPA Enforcement Priority Sectors

DHS has identified specific sectors for priority enforcement:
- **Cotton and cotton products** — polyester/cotton blends included
- **Tomatoes and downstream products** — sauces, paste, canned tomatoes
- **Polysilicon** — solar panels, semiconductor supply chains
- **PVC (polyvinyl chloride)** — pipe, flooring, packaging

The system must map HS codes to these sectors:
- Cotton: Chapters 52 (cotton), 61-62 (apparel), 63 (made-up textiles) when fiber content includes cotton
- Tomatoes: HS 0702, 2002 (prepared tomatoes)
- Polysilicon: HS 2804.61, 8541 (solar cells)
- PVC: HS 3904.10 (PVC resin), 3917 (tubes/pipes), 3918 (floor coverings)

#### Output

Three risk levels:

**HIGH:** Entity on UFLPA list, OR product is in enforcement priority sector AND origin is China, OR known supply chain link to listed entity.
→ System shows: what evidence is required to overcome the presumption (supply chain mapping, payment records, production records, origin certificates for raw materials), which specific sections of the UFLPA enforcement strategy apply.

**MEDIUM:** Product is in or adjacent to an enforcement priority sector, origin is China, but no specific entity match.
→ System shows: "No specific UFLPA entity matches found, but product category is a DHS enforcement priority. Risk of detention exists. Recommend: document supply chain to tier 2 minimum."

**LOW:** Product is not in enforcement priority sector and no entity matches.
→ System shows: "No UFLPA enforcement priority match. No entity list matches. Standard risk level." But also: "Note: UFLPA applies to ALL goods made wholly or in part in XUAR, not only enforcement priority sectors. CBP may detain any China-origin product if indicators suggest XUAR nexus."

#### Nuance: The "Wholly or in Part" Problem

UFLPA's rebuttable presumption applies to goods made "wholly or in part" in XUAR or by UFLPA-listed entities. "In part" means that even a minor input from XUAR (cotton thread in a garment, polysilicon in a solar panel) triggers the presumption. The system should convey this: "Even if the finished product is manufactured in Guangdong, if any input material originated in XUAR or was processed by a listed entity, the UFLPA presumption applies."

---

<a name="engine-4"></a>
## 5. Engine 4: FTA Qualification (USMCA Scope)

### 5.1 Core Requirement

For products originating in Mexico or Canada, assess whether the product qualifies for USMCA preferential tariff treatment by evaluating the applicable rule of origin.

### 5.2 Why This Is Scaffolded, Not Fully Live

USMCA rules of origin are product-specific. There are thousands of product-specific rules in USMCA Annex 4-B. Each rule specifies one or more of:
- **Tariff shift rule:** The HS code of the finished product must differ from the HS codes of all non-originating inputs at a specified level (Chapter, Heading, or Subheading change)
- **Regional Value Content (RVC):** A minimum percentage of the product's value must be attributable to USMCA-country operations
- **Specific process requirement:** For some products, a specific manufacturing process must occur in a USMCA country

Building the full USMCA rules of origin into executable logic for all products is a major effort. For the demo, we scope to ~50 product categories chosen for demo relevance.

### 5.3 Scoped Product Categories

| Category | HS Chapter(s) | Rule Type | Why Included |
|---|---|---|---|
| Auto parts | 87 (specifically 8708) | RVC (75% net cost for vehicles, 70-75% for parts) + specific rules for core parts | Central to David's dashboard story; USMCA auto rules are the most complex and most impactful |
| Electronics (consumer) | 85 (8518, 8527, 8517) | Tariff shift (typically heading-level change) | Demonstrates the system on high-tariff products where USMCA exemption from IEEPA is relevant |
| Steel and metal products | 72-73 | Tariff shift + RVC options | Shows 232 exemption possibility for USMCA-qualifying steel |
| Textiles and apparel | 61-62 | Yarn-forward rule (specific to USMCA textile chapter) | Demonstrates the most restrictive type of origin rule |
| Agricultural products | 1-24 (selected) | Tariff shift, with specific agricultural exceptions | Common import category |
| Plastics | 39 | Tariff shift (heading or subheading) | Common manufacturing input |

### 5.4 Bill of Materials Input

For RVC and tariff shift analysis, the system needs a simplified Bill of Materials:

```
Component | HS Code | Origin | Value (or % of total)
────────────────────────────────────────────────────
Cast iron rotor | 7325.99 | Mexico | $12.00
Mounting bracket | 7326.90 | Mexico | $4.50
Anti-rust coating | 3210.00 | Mexico | $1.20
Fastener set | 7318.15 | USA | $2.30
Packaging | 4819.10 | Mexico | $0.80
```

The system then:
1. Checks each component's HS code against the finished product's HS code — does a tariff shift occur at the required level?
2. Computes RVC: (total value - non-originating material value) / total value
3. Checks any product-specific requirements (e.g., for auto parts, specific components must themselves be originating)

### 5.5 Output Format

```
USMCA Qualification Assessment for HS 8708.99

Applicable Rule: Regional Value Content ≥ 75% (net cost method)
                 OR Tariff Shift at Chapter level for all non-originating inputs

RVC Analysis:
  Total value: $20.80
  Non-originating material value: $0.00 (all components from USMCA countries)
  RVC: 100% — EXCEEDS 75% minimum ✓

Tariff Shift Analysis:
  Finished product: HS 8708 (Chapter 87)
  Component HS codes: 7325, 7326, 3210, 7318, 4819 (Chapters 73, 32, 73, 48)
  All inputs shift at Chapter level ✓

Assessment: QUALIFIES under both RVC and Tariff Shift tests.

Required Documentation:
  • USMCA certification of origin (9 minimum data elements per USMCA Art. 5.2)
  • Bill of Materials with HS codes and origin for each component
  • RVC calculation worksheet
  • Producer/exporter declaration
```

### 5.6 Nuances and Edge Cases

**The automotive rules are the most complex.** USMCA has specific Labor Value Content requirements for vehicles (16% of value must come from workers earning ≥$16/hour). The system should mention this for Chapter 87 products but is not expected to calculate it (requires wage data we won't have).

**De minimis tolerance.** USMCA allows up to 10% of the product's value to be non-originating and still qualify (for most products). The system should apply this tolerance if RVC is close to the threshold.

**Textiles: yarn-forward rule.** For most textile products, USMCA requires that the yarn used to form the fabric must originate in a USMCA country ("yarn-forward"). This is stricter than tariff shift and requires supply chain data the system may not have. The system should flag: "USMCA textile rule requires yarn-forward compliance. Verify that yarn used in fabric production originates in a USMCA country."

### 5.7 Quality and Guardrails

- The system must never state that a product "qualifies" without showing the analytical basis. Every qualification must show the rule applied, the data evaluated, and the result.
- When data is insufficient for a determination, the system says: "Insufficient data for definitive qualification. Based on available information, product appears to [meet/not meet] RVC threshold. A complete determination requires [missing data elements]."
- The system must correctly identify when a product does NOT qualify and explain why. A system that always says "qualifies" is useless.

---

<a name="engine-5"></a>
## 6. Engine 5: Exception Analysis

### 6.1 Core Requirement

Given a customs hold scenario (hold type, product details, CBP's question or hold reason), generate a structured analysis with relevant precedents and a draft response.

### 6.2 Hold Types Covered

| Hold Type | Frequency | System Response |
|---|---|---|
| **Classification question** | Very common | Search CROSS rulings for precedent. Analyze product against ruling language. Draft response citing rulings. |
| **PGA data gap** | Very common | Identify the specific missing data element. If available from product intelligence, populate it. If not, specify what data the importer needs to provide and in what format. |
| **Valuation question** | Common | Identify the basis for the valuation question (transaction value vs. computed value, assists, royalties). Outline the documentation needed. |
| **UFLPA detention** | Increasing | Map the evidence required under the UFLPA enforcement strategy. Identify which evidence types are available vs. missing. Outline the submission format. |
| **Marking/labeling violation** | Common | Identify the specific marking requirement for the product. Specify corrective action. |

### 6.3 CROSS Rulings Knowledge Base

The CBP CROSS (Customs Rulings Online Search System) database contains thousands of binding rulings. For the demo, we need a curated subset.

**How to select rulings for the demo knowledge base:**

1. **By HS chapter:** For each of the most common chapters (39, 42, 61-62, 64, 69, 73, 84, 85, 87, 90, 94, 95), load the 50-100 most recent classification rulings.
2. **By hold type:** For valuation and marking rulings, load a representative sample (100-200) covering common scenarios.
3. **By relevance to demo products:** Specifically load rulings for:
   - Ceramic tableware (Chapter 69) — Maria's mug
   - Auto parts (Chapter 87) — David's dashboard
   - Consumer electronics (Chapters 84-85) — Bluetooth speaker, cable assemblies
   - Cosmetics/skincare (Chapter 33) — Exception scenario

**Total target: 1,000-2,000 indexed rulings.** This is sufficient to have relevant precedent for the demo scenarios and most "bring your own product" cases in common product categories.

### 6.4 Exception Response Generation

The LLM generates a draft response. This is the most sensitive output in the system because it simulates a legal document (a broker's response to CBP). The quality requirements are strict.

**What the response must include:**
- Statement of the product and the classification/issue under review
- Applicable legal authority (HTSUS heading text, GRI, Section/Chapter Notes)
- Cited precedent (CROSS rulings with ruling number and date)
- Evidence from the product record (description, images, prior entries)
- Clear conclusion and request for release

**What the response must NOT do:**
- Fabricate ruling numbers (see Engine 1 guardrails)
- Make legal arguments beyond the system's scope (e.g., constitutional challenges to tariff authority)
- Guarantee outcomes ("CBP will release upon review of this response" — never guaranteed)
- Include personal or confidential information from pre-loaded demo data in a way that looks like real data

### 6.5 Performance Requirements

| Metric | Target | Rationale |
|---|---|---|
| Time to generate exception analysis | < 15 seconds | This is complex LLM reasoning. The audience expects it to take a moment. Stream the output. |
| Time to search CROSS rulings | < 3 seconds | The search should feel fast; the analysis takes time. Show "Found 7 relevant rulings..." quickly, then stream the analysis. |

### 6.6 Guardrails

1. **Label it as a draft.** The output must always be labeled "Draft Response — Requires Human Review." Never present it as a final filing.
2. **Cite only real rulings.** Every ruling number must map to a real entry in the loaded CROSS database.
3. **Distinguish evidence from inference.** "The product description states 'with connectors' (evidence)" is different from "Based on the description, connectors appear to be factory-attached (inference)." The response must clearly distinguish.
4. **Include a confidence assessment.** "Confidence in this response: High — the ruling precedent is directly on point and recent" vs. "Confidence: Medium — the precedent addresses a similar but not identical product."

---

<a name="engine-6"></a>
## 7. Engine 6: Regulatory Change Intelligence

### 7.1 Core Requirement

Monitor, surface, and model the impact of regulatory and tariff changes across the user's product portfolio. This engine addresses the most acute operational pain point in global trade: the velocity of change.

The capability operates in three modes:

| Mode | Trigger | What the System Does |
|---|---|---|
| **Signal Intelligence** | New regulatory development detected (fuzzy or confirmed) | Surfaces the signal to the user with context: what changed or may change, which products/trade lanes are affected, and what action may be needed |
| **Scenario Modeling** ("What If") | User-initiated | User selects a signal (or creates a hypothetical) and the system computes the portfolio-wide impact: change in duty exposure, affected SKUs, cost impact per product, alternative sourcing implications |
| **Automated Impact Analysis** ("It Happened") | Confirmed rate change loaded into knowledge layer | System automatically recomputes affected entries across the portfolio and shows: what changed, by how much, and what actions are needed |

### 7.2 Signal Intelligence

#### Signal Types

Signals are classified into two levels:

| Level | Definition | Source Types | System Behavior |
|---|---|---|---|
| **Fuzzy** | Discussed, proposed, or threatened. Not enacted. May or may not happen. | Government announcements, proposed rules (Federal Register, Official Journal of the EU, MOFCOM notices), trade representative statements, executive statements, leaked negotiation positions | Surface the signal with clear labeling: "PROPOSED — Not yet enacted." Show affected HS codes and trade lanes if identifiable. Enable scenario modeling. |
| **Confirmed** | Enacted, published, with an effective date. Will happen. | Published law, executive orders, Federal Register final rules, Official Journal of the EU regulations, Presidential/State Council proclamations | Surface with effective date and urgency. Auto-trigger impact analysis against portfolio. Flag entries that need attention before the effective date. |

#### Signal Sources by Jurisdiction

| Jurisdiction | Confirmed Change Sources | Fuzzy Signal Sources | Language | Accessibility |
|---|---|---|---|---|
| **US** | Federal Register, Presidential proclamations, CBP CSMS messages | USTR press releases, White House statements, Congressional testimony | English | High — structured, digital, public |
| **EU** | Official Journal of the EU, Commission Implementing Regulations | EC press releases, trade commissioner statements, proposed regulations | English (+ EU languages) | High — EUR-Lex is well-structured |
| **China** | State Council announcements, MOFCOM orders, Customs Tariff Commission | MOFCOM press conferences, state media, diplomatic statements | Chinese (English summaries available) | Medium — requires monitoring Chinese-language sources |
| **Others** | National gazette/official journal | Government statements, trade press | Varies | Varies |

#### For the Demo: Signal Feed is an Illustrative Stub

The signal feed in the demo is pre-authored based on real-world trade dynamics. The signals are synthetic but realistic — based on actual tariff actions that have occurred or been credibly discussed.

**Recommended demo scenarios (at least two, selected for maximum relevance to the audience):**

**Scenario A — Confirmed change:** "IEEPA reciprocal rate for EU increased from 20% to 50%, effective [date]. Published in Executive Order [number]." This is a confirmed signal that triggers automatic impact analysis.

**Scenario B — Fuzzy signal based on real-world tariff weaponization:** "US Trade Representative announces consideration of 100% tariffs on [specific product category] from EU. EU Commission responds with proposed retaliatory tariffs of 120% on US [product category]. China's Customs Tariff Commission simultaneously announces provisional reduction of tariffs on EU [related inputs]." This is a multi-front fuzzy signal that enables scenario modeling across trade lanes — showing the interconnected nature of global trade policy.

The presenter operates the scenario modeling live against these signals. The signals are pre-authored; the impact computation is authentic (Engine 2 running against portfolio data with modified rates).

**Presenter framing:** "The system monitors official regulatory sources across jurisdictions and surfaces changes that affect your portfolio. For proposed and announced changes, it flags the signal and lets you model the impact before it takes effect. For the demo, we've configured scenarios based on current trade dynamics."

### 7.3 Scenario Modeling (Option B — "What If")

#### Core Capability

Given a hypothetical rate change (or a set of changes), recompute the tariff and landed cost for every affected product in the portfolio and show the aggregate impact.

#### Inputs

| Input | Source | Notes |
|---|---|---|
| Rate change definition | From a signal (pre-authored or user-selected) OR user-defined | "Change IEEPA rate for China from 60% to 145%" or "Add 100% surcharge on HS Chapter 85 from EU" |
| Portfolio data | From K15 (simulated company data) or from the session's product analyses | The products and trade lanes to evaluate |
| Scope | User-selected | "All products" or "Only China-origin" or "Only Chapter 85" |

#### Outputs

| Output | Format | Notes |
|---|---|---|
| Affected product count | Number + list | "47 of 312 SKUs affected" with drill-down to each |
| Total duty exposure change | Dollar amount (+ percentage change) | "Additional annual duty exposure: $2.3M (+34%)" |
| Per-product impact | Table | Each affected product: HS code, current total duty, projected total duty, delta |
| Trade lane summary | Grouped by origin-destination | "China→US: +$1.8M. EU→US: +$0.5M." |
| Alternative sourcing flags | Conditional | If the same HS code is available from a non-affected origin: "HS 8518.21 from Vietnam: current total duty 35% vs. projected China duty 145%. Consider sourcing shift." |
| Timeline urgency | If confirmed signal with effective date | "Effective in 30 days. 12 in-transit shipments will arrive after effective date." |

#### What Makes This Real (Not a Spreadsheet)

The scenario modeling uses Engine 2 to recompute tariff stacks with modified parameters. It's the same engine that computed the original tariff — running the same lookups against the same knowledge layer, with one parameter changed. This means:
- Every surcharge interaction is correctly handled (if you increase IEEPA, it stacks correctly with existing 301 and 232)
- FTA preference flags update (if the new rate makes USMCA qualification more valuable, the system highlights it)
- Landed cost totals cascade correctly through tax regime templates

A spreadsheet can show "duty goes from X to Y." The system shows "duty goes from X to Y, AND here's the interaction with three other tariff programs, AND here's which FTA could offset it, AND here are the top 10 products most affected in your portfolio."

### 7.4 Automated Impact Analysis (Option C — "It Happened")

#### Core Capability

When a confirmed rate change is loaded into the knowledge layer (e.g., K5-US IEEPA rates are updated), the system automatically identifies all portfolio entries affected and presents the impact.

#### How It Works

1. A data administrator updates the relevant knowledge collection (e.g., IEEPA China rate changes from 60% to 145% in K5-US)
2. The system identifies all products in the portfolio with the affected origin-destination-HS combination
3. Engine 2 recomputes the tariff stack for each affected entry
4. The system generates an impact report: what changed, which entries are affected, the total dollar impact, and recommended actions

#### For the Demo

The presenter demonstrates this by updating a rate parameter (a simple input — "change IEEPA China from 60% to 145%") and the dashboard recalculates in real time. The audience sees entries reprice, alerts fire, and the aggregate impact number change.

**Presenter line:** "I just simulated what happens when the IEEPA rate for China increases. Watch the dashboard — 47 entries just repriced. The total duty exposure for this portfolio went up $2.3 million. Three products that were marginally profitable are now underwater. And the system automatically flagged that two of those products qualify for USMCA treatment from the Mexico facility — which would eliminate the surcharge entirely. That's not a spreadsheet. That's strategic intelligence."

### 7.5 Quality Requirements

| Requirement | Standard | Notes |
|---|---|---|
| Scenario computation must use the same engine as live tariff computation | Non-negotiable | The "what if" result must be computed the same way as the "what is" result. No separate calculation paths. |
| Signal labeling must be unambiguous | Non-negotiable | "PROPOSED — Not enacted" vs. "CONFIRMED — Effective [date]" must be visually distinct and never confused |
| Impact numbers must be auditable | Non-negotiable | The user must be able to drill into any aggregate number and see the per-product computation |
| Scenario comparisons must show the delta, not just the new state | Required | "Current: $1.2M duty → Projected: $3.5M duty. Change: +$2.3M (+192%)" |

### 7.6 Guardrails

1. **Never present a fuzzy signal as confirmed.** The system must not imply that a proposed change is enacted. This is legally dangerous — a user who acts on a false "confirmed" signal could make costly decisions.
2. **Never present scenario modeling results as current obligations.** Every scenario output must be clearly labeled "SCENARIO — Based on hypothetical rate change. Not current law."
3. **Always show what changed vs. baseline.** The scenario result is meaningless without comparison to the current state. Both must be visible.
4. **No geopolitical predictions.** The system surfaces signals from official sources. It does not predict whether a proposed tariff will be enacted. It models "if this happens, here's the impact" — not "this will happen."

### 7.7 Performance Requirements

| Metric | Target | Rationale |
|---|---|---|
| Scenario recomputation (full portfolio, 200 entries) | < 5 seconds | Must feel responsive. The audience should see the dashboard "ripple" as entries reprice. |
| Single product scenario | < 1 second | Same as Engine 2 performance — it's the same computation. |
| Signal feed rendering | Instant (pre-loaded) | For the demo, signals are pre-authored and loaded at startup. |

---

<a name="orchestration"></a>
## 8. Orchestration Layer

### 8.1 Core Requirement

The orchestration layer manages how the six engines compose when triggered together and how failures, delays, and partial results are handled.

**Resequenceability principle:** Every engine and every capability must be independently callable. The golden path defines a rehearsed narrative sequence, but real demos don't follow scripts. If the audience asks about regulatory change impact before seeing a product analysis, the system must support that. No capability depends on a prior scene having been completed — each is a self-contained analytical unit that can be invoked in any order.

### 8.2 Composition Sequence

When the user submits a product description with origin, destination, and value, the engines execute in a defined sequence:

```
Step 1: Classification (Engine 1) — MUST complete first
    ↓ HS code determined (6-digit global, refined to national subheading)
Step 2 (parallel):
    ├── Tariff Stack (Engine 2) — uses HS code + origin + destination
    │   (reads Tax Regime Template for destination country)
    ├── PGA/Regulatory Jurisdiction (Engine 3A) — uses HS code + destination
    ├── DPS (Engine 3B) — uses entity names if provided
    └── UFLPA/Forced Labor Assessment (Engine 3C) — uses HS code + origin + entities
    ↓ all complete
Step 3: Landed Cost Assembly — combines tariff + taxes + fees + shipping
Step 4: FTA Flag — if applicable FTA exists for origin-destination pair
Step 5 (optional): Regulatory Change Check — flag if any pending signals
    affect this product-origin-destination combination
```

**Classification is the gating step.** Engines 2-6 cannot begin until Engine 1 produces an HS code. This is why the reasoning chain must stream — the audience needs to see progress during the classification step.

**Steps 2-5 are parallelizable.** Once the HS code exists, all downstream engines can run concurrently. The results populate in the UI as they arrive.

### 8.3 Error Handling

| Failure Mode | System Behavior | UX |
|---|---|---|
| Engine 1 fails to classify (Low confidence, no clear code) | Present the ambiguity; run downstream engines for each alternative code | Show: "Two possible classifications. Showing tariff analysis for both." |
| Engine 2 encounters an HS code not in the tariff database | Show the MFN rate as "not found" and skip the tariff stack | "MFN rate for [code] not in current database. Verify HS code." |
| Engine 3B finds a match | Immediate visual alert, prominently displayed | Red alert banner. This must be unmissable. |
| LLM timeout (Engine 1 or 5) | Show partial results + timeout message | "Analysis timed out after [X] seconds. Partial results shown. Retry?" |
| LLM rate limit | Queue and retry with exponential backoff | "Processing. The system is handling multiple requests." (Only relevant if multiple queries in rapid succession.) |

### 8.4 Session Management

The system should maintain a session so the presenter can:
- Navigate back to previous analyses
- Compare two products side-by-side
- Return to a specific result during Q&A

Session state: all inputs and outputs for the current presentation session, stored client-side. No user authentication. No persistent storage beyond the session.

---

<a name="experience"></a>
## 9. Experience Layer

### 9.0 Three Application Surfaces

The experience layer is not one application with different views. It is **three fundamentally different applications** sharing one analytical core. Each surface meets a different actor in their natural context — different layouts, different language, different information density, different interaction models.

#### Surface 1: Platform Intelligence (The Command Center)

**Actor:** Compliance officers, trade analysts, customs brokers, strategic sourcing managers.

**Context:** A destination application. These users come to it as their primary workspace for clearance intelligence.

**Design philosophy: Smart surfacing with progressive disclosure.** The surface shows the most decision-relevant information first — exceptions, regulatory changes, savings opportunities, risk flags — with the ability to drill into any dimension to its full analytical depth.

**Information architecture:**
- **Summary layer (default):** Decision-oriented summaries. "HS 6912.00.48 — MFN 9.8% + IEEPA [rate] — No compliance holds — Entry eligible." One line. The answer.
- **Reasoning layer (one click):** Full analytical chain. Classification reasoning with GRI citations. Tariff stack program-by-program with source annotations. Compliance screening with list-by-list results.
- **Evidence layer (two clicks):** Underlying data for any reasoning step. HTSUS legal text. Federal Register citations. CROSS rulings. Restricted party list entries with match scores.
- **Source annotations are discoverable, not decorative.** Engine-to-knowledge annotations ([E1→K1]) available on hover or in an information panel — not competing with analytical output for visual attention.

**Intelligence, not information.** The dashboard doesn't present 47 entries with equal weight. It presents: "3 entries need your attention (1 UFLPA risk, 1 classification question, 1 FTA savings opportunity). 44 entries cleared normally." The system has already triaged. The analyst's job starts at the exceptions.

**Cross-jurisdiction intelligence is native.** Portfolio exposure across all trade lanes is visible simultaneously. A regulatory change in the US is immediately contextualized against EU and China trade lanes.

**Screens in this surface:**

| Screen | Purpose | Primary Engines | Pre-loaded Data |
|---|---|---|---|
| **Product Analysis** | Describe a product, get a full clearance plan for any destination | 1, 2, 3, 4 | None — fully live |
| **Compliance Dashboard** | Portfolio-level view across trade lanes | 2, 3, 4, 6 | Simulated multi-jurisdiction shipment history. Alerts and exceptions are engine-generated. |
| **Regulatory Change Intelligence** | Signal feed + scenario modeling + impact analysis | 6, 2 | Signal feed is pre-authored (K17). Impact computation is live. |
| **Trade Lane Comparison** | Same product, different destinations | 2 | None — fully live |
| **Exception Resolution** | Hold scenario analysis and response generation | 5, 1 | Hold scenario is pre-written. Analysis is engine-generated. |
| **Entity Screening** | Standalone denied party search across all jurisdiction lists | 3B | None — fully live |
| **Burning Platform** | Full-screen data cards | None | Static content |

#### Surface 2: Shipper Experience (Workflow-Embedded)

**Actor:** E-commerce sellers, logistics coordinators, fulfillment teams, third-party logistics providers.

**Context:** NOT a standalone application. The shipper never opens "Clearance Intelligence Platform." In production, this surface is embedded in their existing tools — Shopify, shipping platform, ERP, fulfillment system. For the demo, it is a clean panel that represents the integration.

**Design philosophy: Invisible complexity. Three questions answered.**

The shipper surface answers exactly three questions:
1. **Can I ship this?** (Yes / Yes with conditions / No — and why)
2. **What will it cost?** (One number: total landed cost including all duties, taxes, and fees)
3. **What do I need to include?** (Plain-language document checklist)

**Requirements:**
- **Language is plain English.** Not "Section 301 surcharge applicable" but "Additional import duty applies for products from China." Not "PGA jurisdiction: FDA — 21 CFR 177" but "Food contact declaration required."
- **Status is traffic-light simple.** Green: ready to ship. Yellow: action needed (with a specific, plain-language action). Red: cannot ship (with the reason in one sentence).
- **Cost is one number with an optional breakdown.** Total landed cost is prominent. Optional expansion shows product / shipping / duties / fees — no program names, no rate percentages, just amounts.
- **Regulatory changes arrive as notifications.** Not signal boards with CONFIRMED/PROPOSED status levels. Instead: "Shipping costs to the US will increase by $4.20 on March 1. No action needed — pricing updated automatically." Anxiety absorbed. Action clear or absent.
- **Document requirements are a checklist.** Not a regulatory mapping — "☐ Commercial invoice ☐ Packing list ☐ Food contact declaration (template attached)." Auto-generated documents are marked as complete.
- **No HTSUS codes, no tariff program names, no GRI citations, no confidence scores, no reasoning chains.** The shipper sees outcomes, not analysis.

**Screens in this surface:**

| Screen | Purpose | Engines Used (invisible) | Actor Sees |
|---|---|---|---|
| **Ship-Ready Panel** | Clearance status for a single shipment | 1, 2, 3 | Status, total cost, document checklist, delivery estimate |
| **Notifications** | Regulatory changes affecting their shipments | 6 | Plain-language updates with cost impact and required actions (if any) |
| **Shipment History** | Past shipments with clearance outcomes | 2, 3 | Dates, destinations, costs, any issues resolved |

#### Surface 3: Buyer Experience (Checkout-Embedded)

**Actor:** The end consumer purchasing a product.

**Context:** The checkout page of an e-commerce site. Not a screen, not a panel, not an application — a line item, a badge, or a price within the existing purchase flow.

**Design philosophy: One number. Total transparency in one line.**

The buyer sees: **"$56.81 delivered — duties and taxes included."**

**Requirements:**
- **"Duties included" is a trust badge.** Appears next to the price like "Free shipping" — a positive signal, not a caveat.
- **Optional breakdown is three lines.** Product, shipping, duties/taxes. The buyer doesn't need six surcharge programs and a Merchandise Processing Fee.
- **Currency is the buyer's currency.** Berlin buyer sees euros. Shanghai buyer sees yuan. The conversion happened upstream.
- **Delivery estimate is clean.** "Estimated delivery: February 12-15." No mention of clearance timing or PGA jurisdiction.

**Screens in this surface:**

| Screen | Purpose | Engines Used (invisible) | Actor Sees |
|---|---|---|---|
| **Checkout Price** | Landed cost at point of purchase | 2 | Total price in local currency, "duties included" badge |
| **Price Breakdown** | Optional expandable | 2 | Product / shipping / duties-taxes — three lines |
| **Today vs. Tomorrow** | Before/after comparison (demo only) | 2 | Surprise charges vs. transparent pricing |

#### Surface Architecture Principle

The three surfaces share one API layer. The engines compute once; the surfaces present differently. This is not a cosmetic concern — it is an architectural requirement. The engine response includes full analytical depth. Each surface's rendering layer selects what to display and how to express it:

```
Engine Response (full depth)
    │
    ├──→ Platform Intelligence renderer → full analysis with progressive disclosure
    ├──→ Shipper renderer → status + cost + documents + plain-language alerts
    └──→ Buyer renderer → price + badge + optional 3-line breakdown
```

This means the engines do not need to know which surface will display their output. They always return full-depth results. The rendering/presentation layer for each surface extracts and reformats as appropriate. This separation is critical for maintainability and for ensuring the Shipper and Buyer surfaces never accidentally expose analytical internals.

### 9.1 Platform Intelligence — Component Requirements

The following subsections describe the core UI components for the Platform Intelligence surface. The Shipper and Buyer surfaces have simpler rendering requirements (defined in Section 9.0 above) — they consume the same engine output but render only the relevant subset.

### 9.2 Reasoning Chain Renderer

This is the most novel UI component. It must render the step-by-step reasoning from Engine 1 as a structured, readable, progressive display.

**Requirements:**
- Each step appears sequentially (streaming from the LLM)
- Steps are visually grouped by hierarchy level (Section, Chapter, Heading, etc.)
- Each step can be expanded to show the HTSUS text it references
- The current step is visually highlighted while subsequent steps are pending
- When complete, the final classification is promoted to a summary position at the top
- Color-coded left border by engine (see Visual Design in golden path doc)

**This renderer is reused for Engine 5** (exception analysis reasoning). The same component should work for any structured reasoning chain, not just classification.

### 9.3 Tariff Stack Renderer

Displays the tariff program-by-program breakdown. Requirements:
- Each program appears as a row: program name, applicability (Y/N with reason), rate
- Applied programs accumulate into a running total
- Non-applicable programs are shown grayed but visible (important for transparency)
- The total effective rate is emphasized at the bottom
- A plain-English annotation accompanies extreme results ("Duty is 85% of product value — the consumer pays more in duty than for the product itself")

### 9.4 Input Form Design

The product description input must be a genuine text area — not a search box. It should invite natural-language descriptions. Placeholder text: "Describe the product in ordinary language. Include materials, function, and construction if known."

**Do not use a structured form with separate fields for material, function, etc.** The point is that the system handles natural language. Structured input undermines the demo's thesis.

**Do provide structured fields for:** country of origin (dropdown with search), destination country (dropdown — defaulting to US but supporting EU member states and China), customs value (numeric input with currency selector), and optionally: quantity, weight, shipping cost, destination state/region.

**The destination country selector is critical for the global story.** The presenter should be able to change the destination from US to Germany to China and show the tariff stack changing — different tax regime templates, different surcharge programs, different regulatory frameworks. This is one of the most powerful visual demonstrations of the platform's global capability.

### 9.5 Dashboard Design

The compliance dashboard (Scene 2) is pre-populated with simulated data for a fictional company. But the analytical results within the dashboard are engine-generated.

**What's simulated:** The shipment records, SKU numbers, supplier names, entry dates, and shipment volumes. These are authored as realistic data for AutoParts Global, Inc. **The volume must be plausible for the company profile.** A mid-sized auto parts importer with 100 suppliers and 1,000 SKUs receiving shipments 3x/week might process 30-80 entries overnight — not thousands. If a larger volume is desired for visual impact, either scale up the company profile (Fortune 500 auto manufacturer) or frame the dashboard as FedEx's brokerage operations view across multiple customers.

**What's engine-generated:** The duty rates on each entry (Engine 2), the compliance flags (Engine 3), the FTA savings alert (Engine 4), the UFLPA risk alert (Engine 3C). These are computed from the simulated data using the same engines that run in the live product analysis.

**Why this distinction matters:** A stakeholder who asks "is this dashboard real?" gets the honest answer: "The company and shipment data are simulated. The analytical results — the duty calculations, the compliance flags, the FTA savings identification — those are produced by the same engines that just analyzed your product."

### 9.6 Animation and Progressive Rendering

**General principle:** Render results as they become available. Never hold back completed results while waiting for others.

**Specific requirements:**
- Classification reasoning chain: stream token by token or step by step as the LLM generates
- Tariff stack: render all at once (it's fast) or build program by program (300ms each) for dramatic effect — presenter should be able to choose via a setting
- Compliance results: render per sub-engine as each completes (PGA first, then DPS, then UFLPA)
- Exception analysis: stream the analysis as the LLM generates, with rulings citations appearing as "chips" that can be expanded

---

<a name="knowledge"></a>
## 10. Knowledge Collections

Each knowledge collection is a structured data set that must be assembled, validated, and loaded before the engines can function. This is the hardest work in the build — and the most important.

### 10.1 Knowledge Architecture

Knowledge is organized by **jurisdiction** and **function**. Each jurisdiction requires a parallel set of collections that map to the same engine interfaces. The naming convention is: `K[number]-[jurisdiction]` (e.g., K2-US for US tariff rates, K2-EU for EU tariff rates).

The HS code at 6 digits is the global key that links knowledge across jurisdictions. A product classified as HS 6912.00 accesses K2-US (HTSUS rate), K2-EU (TARIC rate), and K2-CN (China tariff rate) using the same key — with national subheading refinement as available.

### 10.2 US Knowledge Collections (Tier 1 — Full Depth)

| ID | Collection | Used By | Volume | Source | Public? | Effort |
|---|---|---|---|---|---|---|
| K1-US | HTSUS Schedule | Engine 1, 2 | ~19,000 headings/subheadings, 99 chapters of notes | USITC (usitc.gov) | Yes — English | **HIGH** — Parsing from PDF/HTML into structured format |
| K2-US | HTSUS Rates | Engine 2 | Rate for every subheading (General, Special, Column 2) | USITC | Yes | Included in K1-US, separately validated |
| K3-US | Section 301 Product Lists | Engine 2 | ~10,000 HS codes across Lists 1-4 with current rates | USTR + Federal Register | Yes | **MEDIUM** — Modified multiple times; must track current rate per code |
| K4-US | Section 232 Product Scope | Engine 2 | ~500 HS codes (steel/aluminum) + country exemptions | Commerce/Presidential Proclamations | Yes | **LOW** |
| K5-US | IEEPA Country Rate Schedule | Engine 2, 6 | Rate per country (~200 countries) | Executive Orders + CBP guidance | Yes | **MEDIUM** — Volatile; must be current |
| K6-US | AD/CVD Active Orders | Engine 2 | ~500 active orders with HS codes, countries, rate ranges | ITC (usitc.gov) | Yes | **MEDIUM** |
| K7-US | OFAC SDN + Consolidated List | Engine 3B | ~12,000 entries with aliases, addresses | OFAC (treasury.gov) | Yes — XML/CSV | **LOW** |
| K8-US | BIS Entity/Denied/Unverified Lists | Engine 3B | ~1,400 combined entries | BIS (commerce.gov) | Yes | **LOW** |
| K9-US | UFLPA Entity List | Engine 3B, 3C | ~70 entries | DHS FLETF | Yes | **LOW** |
| K10-US | PGA-HS Mapping with Regulatory Authority | Engine 3A | Mapping of HS ranges to ~48 PGAs with specific regulatory citations (e.g., FDA ceramics → CPG Sec. 545.450) | CBP ACE PGA documentation + agency regulations | Yes (scattered) | **HIGH** |
| K11-US | USMCA Rules of Origin | Engine 4 | Product-specific rules for ~50 categories | USMCA Annex 4-B | Yes | **MEDIUM** |
| K12-US | CBP CROSS Rulings | Engine 5 | 1,000-2,000 curated rulings | rulings.cbp.gov | Yes | **MEDIUM** |
| K13-US | State Tax Rates and Base Rules | Engine 2 | 50 states + base computation rules | State departments of revenue | Yes | **MEDIUM** |
| K14 | Classification Reference Examples | Engine 1 | 500-1,000 product → HS code mappings | Curated from CROSS rulings + CBP guidance | Yes | **MEDIUM** |
| K15 | Simulated Company Data | Dashboard, Engine 6 | Shipment records across global trade lanes for demo company | Authored for demo | N/A | **LOW** — must include multi-jurisdiction entries |
| K16-US | CBP Fee Schedule | Engine 2 | MPF rates (formal/informal), HMF rate, by fiscal year | CBP fee schedule, Federal Register | Yes | **LOW** |
| K17 | Regulatory Change Signals | Engine 6 | Pre-authored signals for demo scenarios | Authored based on real trade dynamics | N/A | **LOW** — illustrative stub |
| K18 | Exchange Rates | Engine 2 | USD to EUR, CNY, BRL, INR, etc. | Public (ECB, Federal Reserve) | Yes | **LOW** — static snapshot for demo |

### 10.3 EU Knowledge Collections (Tier 2 — Meaningful Depth)

| ID | Collection | Used By | Volume | Source | Public? | Language | Effort |
|---|---|---|---|---|---|---|---|
| K1-EU | EU Combined Nomenclature (CN) Schedule | Engine 1, 2 | ~15,000 CN codes (8-digit) | TARIC (ec.europa.eu/taxation_customs) | Yes — **free, structured, machine-readable** | English | **MODERATE** — TARIC is better structured than HTSUS |
| K2-EU | TARIC Duty Rates | Engine 2 | Third-country (MFN) rate for every CN code | TARIC database | Yes | English | Included in K1-EU |
| K3-EU | EU Surcharge Programs | Engine 2 | Additional duties on US goods (232 retaliation), potential new retaliatory measures | Official Journal of the EU | Yes | English | **LOW-MEDIUM** — Finite list of CN codes |
| K4-EU | EU Safeguard Duties | Engine 2 | Steel safeguard product scope + quotas | Commission Implementing Regulations | Yes | English | **LOW** |
| K6-EU | EU AD/CVD Orders | Engine 2 | Active anti-dumping and countervailing duties | EC TRON database (tron.trade.ec.europa.eu) | Yes | English | **MEDIUM** |
| K7-EU | EU Sanctions Lists | Engine 3B | EU Consolidated Sanctions List | EU Council (data.europa.eu) | Yes — downloadable | English | **LOW** |
| K10-EU | EU Regulatory-HS Mapping | Engine 3A | CE marking categories, REACH flagging, EU product safety | EU Directives, ECHA database | Yes | English | **MEDIUM-HIGH** — CE marking mapping is achievable; full REACH is enormous (flag awareness-level only) |
| K13-EU | VAT Rates by Member State | Engine 2 | Standard + reduced rates per member state + product category mapping | EU Commission, member state revenue authorities | Yes | English | **LOW** — Stable, well-documented |

### 10.4 China Knowledge Collections (Tier 2 — Meaningful Depth)

| ID | Collection | Used By | Volume | Source | Public? | Language | Effort |
|---|---|---|---|---|---|---|---|
| K1-CN | China HS Tariff Schedule | Engine 1, 2 | ~8,500 tariff lines (8-digit) | Customs Tariff Commission / MOFCOM | Yes | **Chinese** (English via WTO at 6-digit or commercial providers at 8-digit) | **MODERATE-HIGH** — Language barrier; recommend WTO 6-digit free data + manual research for demo products + commercial feed for production |
| K2-CN | China MFN Rates | Engine 2 | MFN applied rate for every tariff line | Same as K1-CN | Yes | Chinese | Included in K1-CN |
| K3-CN | China Retaliatory Tariff Lists | Engine 2, 6 | Additional duties on US-origin goods (5-25%, multiple rounds) | State Council / MOFCOM announcements | Yes | **Chinese** (English summaries available from law firms, trade press) | **MODERATE** — Must compile from multiple announcements |
| K6-CN | China AD/CVD Orders | Engine 2 | Active anti-dumping and countervailing duties | MOFCOM | Yes | Chinese | **MODERATE** |
| K7-CN | China Unreliable Entity List | Engine 3B | Small list | MOFCOM | Yes | Chinese + English | **LOW** |
| K10-CN | China Regulatory-HS Mapping | Engine 3A | CCC (China Compulsory Certification) product categories, NMPA flagging | SAMR/CNCA (CCC), NMPA | Yes | Chinese | **MEDIUM** — CCC product list is finite; NMPA is complex |
| K13-CN | China VAT + Consumption Tax | Engine 2 | VAT: 13% standard / 9% reduced. Consumption tax: rates by product category | State Taxation Administration | Yes | English coverage available | **LOW** — Simple rate structure |

### 10.5 Brazil Knowledge Collections (Tier 3 — Framework Demonstration)

| ID | Collection | Used By | Volume | Source | Public? | Language | Effort |
|---|---|---|---|---|---|---|---|
| K1-BR | NCM Tariff Schedule | Engine 1, 2 | ~10,000 NCM codes (8-digit) | Receita Federal | Yes | **Portuguese** | **MODERATE** |
| K2-BR | Mercosur CET + Brazil MFN Rates | Engine 2 | Rate per NCM code | Receita Federal / CAMEX | Yes | Portuguese | Included in K1-BR |
| K3-BR | IPI Rate Table (TIPI) | Engine 2 | IPI rate per NCM code | Receita Federal | Yes | Portuguese | **MODERATE** |
| K4-BR | ICMS State Rates | Engine 2 | Rate per state + product exceptions | State finance secretariats | Yes | Portuguese | **HIGH** — 27 states, product-specific exceptions |
| K13-BR | PIS/COFINS + Tax Regime Parameters | Engine 2 | Standard import rates (2.1% PIS, 9.65% COFINS) + grossup parameters | Receita Federal | Yes | Portuguese | **LOW** — Standard rates; engine needs grossup solver |

### 10.6 Adding Future Jurisdictions

Once US, EU, and China collections are loaded, adding a new jurisdiction follows a repeatable process:

1. **Obtain national tariff schedule** — Parse into HS-based structure (6-digit global key + national refinement)
2. **Map surcharge programs** — Identify retaliatory/safeguard duties with HS code lists and rates
3. **Load AD/CVD orders** — If the jurisdiction has active trade remedy measures
4. **Define Tax Regime Template** — Specify tax layers, rates, bases, and stacking pattern (additive or cascading)
5. **Map major regulatory requirements** — At minimum: which product categories require pre-market approval or import licensing
6. **Load sanctions/restricted party lists** — If the jurisdiction maintains its own lists
7. **Validate** — Run 20+ products through the full pipeline for the new jurisdiction

Estimated effort per additional standard jurisdiction (additive stacking, English data available): **2-3 weeks of data engineering.** For jurisdictions with language barriers or cascading tax structures: **4-6 weeks.**

### 10.7 Knowledge Quality Requirements

| Quality Dimension | Requirement | How to Validate |
|---|---|---|
| **Correctness** | Every fact in the knowledge base must be verifiable against its source document | Spot-check 10% of entries; validate 100% of entries used in golden path scenarios |
| **Currency** | Tariff rates and program lists must reflect the currently effective rules | Compare loaded rates against live USITC/USTR/CBP sources within 1 week of any demo |
| **Completeness** | Within the defined scope, no major gaps | Test each engine against 20+ products in each of its covered categories |
| **Consistency** | No contradictions between knowledge collections | Cross-validate K2 rates against K3/K4/K5 program applicability |
| **Parsability** | The structured format must be reliably readable by the engines | Automated tests that every loaded HS code has a valid rate, every 301 entry maps to a valid HS code, etc. |

### 10.8 Performance Requirements for Knowledge Access

| Access Pattern | Target Latency | Notes |
|---|---|---|
| MFN rate lookup by HS code | < 50ms | Indexed lookup. Simple key-value. |
| 301/232/IEEPA check for an HS code | < 100ms total for all three | Three parallel lookups against indexed lists |
| PGA jurisdiction check | < 100ms | Range-based lookup against HS code ranges |
| Denied party screening (single entity) | < 1 second | Fuzzy matching against ~14,000 entries. Must be pre-indexed with name tokenization. |
| CROSS rulings search | < 2 seconds | Semantic search across 1,000-2,000 rulings. Requires vector index or similar. |
| HTSUS text retrieval (for classification reasoning) | < 200ms per chapter/heading | Chapter Notes, Heading text, etc. must be retrievable by identifier. |

---

<a name="quality"></a>
## 11. Quality Framework

### 11.1 The Quality Question for Each Engine

Each engine has a different quality profile. Accuracy is not uniform.

| Engine | Quality Model | Acceptable Error Profile |
|---|---|---|
| **Classification (1)** | Probabilistic. The LLM reasons; answers vary. | Chapter-level: must be correct >95% of the time. Heading-level (4-digit): >85%. Subheading (8-digit): >75%. Statistical suffix (10-digit): >60%. Errors at deeper levels are expected and should trigger Medium/Low confidence. |
| **Tariff Stack (2)** | Deterministic. Lookups + math. | **Must be 100% correct** for loaded data. Any error is a data quality issue, not an engine logic issue. |
| **PGA Mapping (3A)** | Deterministic. Lookup-based. | >95% for top 200 product categories. False positives (over-flagging) acceptable. False negatives (missing a PGA requirement) not acceptable for common products. |
| **DPS (3B)** | Deterministic + fuzzy. | True positives: must catch exact name matches 100%. Fuzzy matches: measured by recall (catch rate) against known aliases. Precision: some false positives acceptable but not so many that it becomes noise. |
| **UFLPA (3C)** | Rules-based + screening. | Entity screening: same as 3B. Sector mapping: must correctly flag enforcement priority sectors. |
| **FTA (4)** | Rules-based on structured input. | When BOM is complete and accurate, qualification determination must match what a human trade advisor would conclude. When BOM is incomplete, the system must say so. |
| **Exception Analysis (5)** | LLM-generated. Quality varies. | Rulings cited must be real. Legal reasoning must be directionally correct. The draft response should be useful to a broker — not perfect, but a strong starting point. |

### 11.2 How to Test Quality Before the Demo

**Classification testing protocol:**
1. Assemble a test set of 100 products across 20 chapters. Include:
   - 40 "easy" products (clear material + function, unambiguous classification)
   - 40 "medium" products (require Chapter Notes or GRI application)
   - 20 "hard" products (composite goods, products spanning chapters, textiles with fiber composition)
2. Have a trade compliance SME independently classify all 100
3. Run all 100 through Engine 1
4. Compare: chapter match rate, heading match rate, subheading match rate
5. Review all Medium and Low confidence outputs — are the confidence levels calibrated correctly?
6. Review all ambiguity flags — are the alternatives reasonable?

**Tariff testing protocol:**
1. Select 50 product/country combinations spanning all tariff programs
2. Manually compute the correct tariff stack for each (using USITC tariff database)
3. Run all 50 through Engine 2
4. Every result must match exactly. No exceptions.

**End-to-end testing:**
1. Run 20 products through the full orchestration pipeline (description → classification → tariff → compliance → landed cost)
2. Have a trade compliance SME review each output end-to-end
3. Flag any result that would be embarrassing if shown to a trade professional

### 11.3 Quality During the Demo

**The presenter must know the quality boundaries.** Before the demo:
- The presenter reviews the 20+ test products that were run through the system
- The presenter knows which product categories the system handles well and which it struggles with
- The presenter has a mental list of "safe" products to suggest if the audience's suggestions go off scope

**The system's self-reported confidence must be trustworthy.** If the system says "High confidence," the audience must be able to trust that. If it says High confidence and it's wrong, the confidence system is broken. Calibration matters more than average accuracy.

---

<a name="guardrails"></a>
## 12. Guardrails — What the System Must Never Do

### 12.1 Factual Guardrails

| Rule | Applies To | Enforcement |
|---|---|---|
| **Never output an HS code that doesn't exist in the loaded schedule** | Engine 1 | Post-generation validation: every HS code is checked against the loaded tariff schedule for the destination jurisdiction before display |
| **Never cite a ruling that doesn't exist** | Engine 5 | Every ruling number in the output is checked against the loaded CROSS database |
| **Never show a tariff rate that isn't sourced from loaded data** | Engine 2 | All rates come from lookups, not from LLM generation. This applies across all jurisdictions. |
| **Never state "no restrictions" without listing what was checked** | Engine 3 | Every "clear" result includes the list of checks performed |
| **Never present a fuzzy regulatory signal as a confirmed obligation** | Engine 6 | Every signal is labeled with its status (CONFIRMED/PROPOSED/DISCUSSED) and source. Scenario modeling output is always labeled "if enacted." |

### 12.2 Behavioral Guardrails

| Rule | Rationale |
|---|---|
| **Never present a classification without reasoning** | Forces the LLM to show its work. Prevents confident wrong answers. |
| **Never present an FTA qualification without showing the rule and data** | Prevents "it qualifies" without analytical basis. |
| **Never omit a required disclosure** | If something is a draft, label it as a draft. If something is simulated, label it as simulated. |
| **Never hide uncertainty** | Medium and Low confidence must be visible, not suppressed. |
| **Never present simulated data as live data** | The dashboard uses simulated company data. This must be clear if asked. |
| **Never present a scenario model as current obligations** | Scenario modeling shows hypothetical impact. It must never be confused with actual duties owed. |

### 12.2a Surface Integrity Guardrails

| Rule | Applies To | Rationale |
|---|---|---|
| **Never expose analytical internals on the Shipper or Buyer surface** | Shipper, Buyer surfaces | HTSUS codes, tariff program names, GRI citations, confidence scores, reasoning chains, and engine annotations must NEVER appear on the Shipper or Buyer surfaces. These surfaces show outcomes, not analysis. A leaked "[E1→K1]" annotation on the Shipper panel destroys the simplicity thesis. |
| **Never simplify the Platform Intelligence surface to match Shipper/Buyer** | Platform Intelligence | The command center must maintain full analytical depth with progressive disclosure. If it's simplified to match the other surfaces, the contrast that drives the switching moments disappears. |
| **Never present the Shipper surface as a collapsed version of Platform Intelligence** | Shipper surface | The Shipper surface must have its own layout, its own language, its own information hierarchy. If it looks like the Platform Intelligence view with panels hidden, it undermines the "different application" principle. |
| **The plain-language translation must be accurate, not just simple** | Shipper surface | "Additional import duty applies" is correct. "No import duties" when Section 301 applies is wrong. Simplicity cannot sacrifice accuracy. Every plain-language statement must be traceable to the underlying engine output. |

### 12.3 Prompt Engineering Guardrails

For Engines 1, 5, and any other LLM-driven component:

- **System prompt instructs the LLM to cite sources for every claim.** No unsourced assertions about rates, rules, or requirements.
- **System prompt instructs the LLM to express uncertainty.** "Based on the description provided, this appears to be..." rather than "This is..."
- **System prompt prohibits the LLM from inventing data.** If the LLM doesn't know an HS code, it says so. It does not generate a plausible-looking code.
- **Output validation layer sits between the LLM response and the UI.** This layer checks: are all HS codes valid? Are all cited rulings in the database? Are all rates consistent with loaded tariff data? If validation fails, the offending element is flagged or removed before display.

---

<a name="stubs"></a>
## 13. Stub Strategy — The Line Between Authentic and Realistic

This is the most nuanced design question. Some capabilities in the full platform require integrations, data feeds, or operational infrastructure that don't exist in the demo. We must decide, for each: do we simulate it? If so, how? And how transparent are we about the simulation?

### 13.1 The Framework: Four Categories

| Category | Definition | How to Handle | Transparency |
|---|---|---|---|
| **Authentic** | The capability works for real, against real data, producing real answers. | Build it. | "This is live." |
| **Realistic Stub** | The capability is simulated but uses the same logic and data structures the real system would use. The simulation is indistinguishable from reality within its scope. | Build a bounded version. | "This works within our demo scope. The production system would cover [broader scope] with the same analytical approach." |
| **Illustrative Stub** | The capability is represented with pre-authored data to show *what it would look like*. The data is realistic but not computed. | Author the data. Design the UI to display it. | "This data is illustrative. In the production system, this would be computed by [engine]. We've authored realistic examples to show the experience." |
| **Observable Stub** | The capability doesn't exist yet, but we show what the interface and output would look like, clearly labeled as a design concept. | Design the UI. Label it as a concept. | "This is a design concept showing what the production capability would look like." |

### 13.2 Capability-by-Capability Stub Decisions

| Capability | Category | What's Real | What's Simulated | Is It Worth Showing the Stub? |
|---|---|---|---|---|
| **Product Classification** | Authentic | LLM reasoning against full HTSUS | Nothing | — |
| **Tariff Stack Computation** | Authentic (Tier 1/2) | Full rate lookups for US (all HS codes), EU (CN codes via TARIC), China (loaded codes). Tax regime templates compute per jurisdiction. | Tier 3 jurisdictions (Brazil, India) have limited coverage — MFN + primary tax only. | — |
| **Landed Cost Calculation** | Authentic (mostly) | Duty + fees are computed per destination's tax regime template. Multi-currency display via K18 exchange rates. | Shipping cost is estimated (not a live carrier rate query). US state tax uses loaded rate, not real-time API. Exchange rates are loaded, not live-streamed. | Yes — the shipping estimate should be labeled "estimated" and the system should note "production system integrates live carrier rates." Exchange rate "as of" date should be displayed. |
| **PGA Jurisdiction Mapping** | Authentic | Mapping from HS code to PGA | Nothing within loaded mappings. Gaps exist for uncommon HS-PGA combinations. | — |
| **Denied Party Screening** | Authentic | Real screening against real lists | Nothing | — |
| **UFLPA Risk Assessment** | Realistic Stub | Entity screening is live. Sector mapping is live. | Supply chain tier-2 visibility is not live — the system can't actually trace supply chains. When showing the tier-2 risk alert in the dashboard, the supply chain tree is illustrative. | **Yes, but with clear framing.** The supply chain tree is valuable to show because it demonstrates what the production system would surface. Label it: "Supply chain data source: illustrative. In production, integrated from third-party supply chain intelligence provider." The *screening* of the entity against the UFLPA list IS live — and the presenter should highlight that distinction. |
| **FTA Qualification** | Realistic Stub | USMCA rules for ~50 categories are real. Logic is real. Optionally one EU FTA at framework level. | Only covers 50 USMCA categories. EU FTA coverage limited. BOM data in the dashboard scenario is pre-authored. | **Yes.** The analytical logic is genuine — it applies real USMCA rules. The scope limitation is honest and natural. The presenter says: "We've loaded the rules of origin for fifty product categories under USMCA. In production, the full annex is covered. The same analytical approach extends to EU FTAs and any other preferential trade agreement." |
| **Product Intelligence Memory** | Illustrative Stub | Nothing | Pre-loaded shipment history for demo personas | **Yes, but carefully.** This is the concept that the system remembers prior shipments and uses that memory to prevent exceptions. It's powerful conceptually but it's entirely pre-authored data. Show it in the exception resolution scene (Scene 5) where the system "remembers" a prior cosmetics shipment. Label the data as illustrative but the concept as real: "The system would maintain a product intelligence database from every entry it processes. We've seeded it with historical data to show how it prevents this hold." |
| **Exception Response Generation** | Realistic Stub | LLM generates real analysis. CROSS rulings are real. | The hold scenarios are pre-authored (not from a live CBP feed). | **Yes.** The analysis is genuine even though the scenario is authored. The presenter types the hold scenario; the system analyzes it live. The fact that the scenario is authored is like a case study — it doesn't reduce the value of the analysis. |
| **ACE Filing Preview** | Observable Stub | Nothing | N/A — we cannot connect to ACE | **No.** Don't show an ACE filing screen. It adds complexity without adding conviction. The system can note: "Advance filing data staged for ACE" as a status message, but don't render a fake ACE submission. |
| **Advance Data Filing** | Observable Stub | Nothing | N/A | **Mention, don't show.** The concept of pre-clearance via advance data filing is important to the narrative but doesn't need a UI. The presenter explains the concept; the consumer checkout scene illustrates the result (cleared before arrival). |
| **Trade Lane Comparison** | Authentic | Engine 2 computes landed cost per destination using different tax regime templates | Nothing (assumes Tier 2 knowledge is loaded for EU/China) | **Yes — this is Scene 3.** Same product, multiple destinations, side-by-side. Each column is computed by a different tax regime template. This is one of the strongest proof points of the global architecture. |
| **Regulatory Change Intelligence** | Authentic (Tier 1) / Realistic Stub (Tier 2) | Engine 6A: Signal database with real signals, sourced and attributed. Engine 6B: Scenario modeling recomputes via Engine 2. Engine 6C: Impact analysis scans portfolio data. | K17 signal database is curated (20-30 real signals, not a live monitoring feed). Portfolio data (K15) is pre-authored. | **Yes — this is Scene 4.** The signals are real and sourced. The scenario modeling uses real tariff recomputation. The portfolio impact is computed against pre-authored data. The presenter says: "The signals are real — sourced from Federal Register, Official Journal, MOFCOM. The scenario modeling uses the same tariff engine you just saw. In production, the signal database would be populated from automated monitoring. For the demo, we've curated the most impactful signals across three jurisdictions." |
| **Shipper Surface (workflow-embedded)** | Realistic Stub | Ship-Ready Panel is computed live from engine output. Plain-language translation layer is real. Document checklist is real. Regulatory change notifications are real (from Engine 6 signals). | The panel is standalone for the demo — not actually embedded in Shopify/ERP/etc. Auto-generated documents are stubs (template, not populated from real order data). | **Yes — this is the switching moment.** The Shipper Surface appears in Scenes 1, 4, and 6 as the "what Maria sees" reveal. The analytical capability behind it is authentic. The workflow embedding is a design concept. The presenter says: "In production, this panel lives inside the shipper's existing platform — their Shopify dashboard, their ERP, their shipping tool. They never come to us. We go to them." |
| **Buyer Surface (checkout-embedded)** | Realistic Stub | Landed cost price is computed live by Engine 2. Multi-currency conversion uses K18. "Duties included" badge is functional. | The checkout page is a mockup — not embedded in a real e-commerce site. Price breakdown is simplified from engine output, not a production checkout integration. | **Yes — this is the Scene 3 and Scene 6 reveal.** The price is authentic (computed by the same Engine 2 the audience just watched). The checkout context is illustrative. The presenter says: "This price was computed by the same engine that just showed you the twelve-line tariff stack. The consumer sees one number." |
| **Carrier Integration (FedEx tracking/status)** | Not in scope | N/A | N/A | **No.** Don't simulate carrier integration. It introduces questions about FedEx systems access that are outside the demo's thesis. |

### 13.3 The Line Between Authentic and Realistic

The key question: **When does a stub damage credibility vs. enhance understanding?**

**A stub damages credibility when:**
- It's presented as live but a knowledgeable audience member could tell it's scripted
- It claims a capability the system doesn't have (even in bounded form)
- It looks too good — suspiciously perfect data, zero latency, no edge cases

**A stub enhances understanding when:**
- It's labeled honestly ("this data is illustrative; the screening against it is live")
- It shows the shape of the experience without claiming the full capability
- It answers the question "what would this look like in production?" without pretending it's production
- The underlying analytical capability IS real, and the stub just provides the data context

**Practical test:** If a skeptical audience member asks "is this real?" can the presenter give an honest answer that strengthens rather than weakens the impression? If the answer is "the analysis is real, the data context is illustrative," that's fine. If the answer requires deception or evasion, cut the stub.

### 13.4 Stub Observability Design

For illustrative stubs (supply chain tree, product intelligence memory, pre-authored portfolio data), the UI should include a subtle but visible marker:

A small icon or label in the corner of the component: **"Illustrative scenario"** or **"Demo data"**

This is NOT a disclaimer. It's not apologetic. It's factual. The presenter doesn't draw attention to it unless asked. But if someone notices it, the honesty builds trust: "The system is transparent about what's live and what's illustrative. In a real engagement, this distinction disappears because all data is live."

If the team decides this marker is distracting, an alternative: include a "System Information" panel accessible from a settings icon that lists, for each component on screen, whether it's live or illustrative. The presenter never opens this panel proactively, but it's available if asked.

---

<a name="sequencing"></a>
## 14. Build Sequencing

### 14.1 Sequencing Principles

1. **Knowledge before engines.** An engine without data is useless. Data preparation must lead engine development.
2. **Deterministic before probabilistic.** Engine 2 (tariff computation) is deterministic — it either works or it doesn't. Engine 1 (classification) is probabilistic — it works in degrees. Build deterministic engines first; they establish a foundation of trust.
3. **High-value engines before lower-value engines.** The "Bring Your Own Product" moment depends on Engines 1 + 2 + 3. Engine 4 (FTA) and Engine 5 (Exception) are important but not gating.
4. **US depth first, then global breadth.** Establish full Tier 1 depth on US imports before expanding to EU and China. The US implementation proves the architecture; global expansion proves the architecture scales. This also means the demo is always viable — US-only is a credible demo; global is a more powerful one.
5. **Experience layer concurrent with engines.** The UI doesn't wait for perfect engines — it's built against the engine interfaces as they develop.
6. **End-to-end integration early.** Don't perfect each engine in isolation. Get a rough end-to-end flow working early, then improve each engine.
7. **Resequenceability from the start.** Every engine, every scene, every capability must be independently callable. This is not a Phase 8 polish item — it's a Phase 1 architectural requirement. The demo must be reorderable without code changes.
8. **Surfaces are presentation layers, not engine branches.** The three application surfaces (Platform Intelligence, Shipper, Buyer) consume the same engine output through different renderers. Build engine APIs that return full-depth results. Build surface-specific rendering independently. Never fork engine logic for a specific surface.

### 14.2 Build Phases

#### Phase 1: Foundation — US Knowledge Layer and Tariff Engine

**Goal:** Establish the US data foundation, prove the deterministic core works, and lock the architecture for global expansion.

**Deliverables:**
- **K1-US/K2-US: HTSUS parsed and loaded.** The full tariff schedule in a structured, queryable format. Every heading, every rate, every note.
- **K3-US/K4-US/K5-US: Tariff program lists loaded.** 301, 232, IEEPA cross-referenced to HS codes.
- **K7-US/K8-US/K9-US: Restricted party lists loaded.** OFAC SDN, BIS lists, UFLPA list. Pre-indexed for fuzzy matching.
- **K13-US: State tax rates and base rules loaded.** Combined rate per state plus base computation rules.
- **K16-US: CBP fee schedule loaded.** Current fiscal year MPF rates (formal and informal) and HMF rate with effective dates.
- **Tax Regime Template Architecture implemented.** The Pattern A (additive) template for US is the first implementation. The template structure must be designed to accept Pattern B (cascading) and additional jurisdictions from Phase 4 onward.
- **Engine 2: Tariff Stack Calculator operational for US.** Given any HS code + any origin country + declared value, determines entry type, computes the full tariff stack and landed cost correctly using the US tax regime template. Validated against 50+ manual computations including both formal and informal entry scenarios.
- **Engine 3B: Denied Party Screening operational.** Fuzzy matching against all loaded lists.
- **Resequenceable module interfaces defined.** Every engine exposes a standalone callable interface from day one.

**Why this first:** Engine 2 and Engine 3B are the most verifiable, most deterministic engines. They either produce the correct answer or they don't. Building them first establishes a foundation of correctness. Also, the tariff stack is the most viscerally impactful output for the audience — showing that a $45 router has 85% effective duty needs to be right. The tax regime template architecture is implemented here because it's trivial for the US case (Pattern A, additive) but ensures the structure exists when global expansion begins.

**Milestone test:** A trade compliance SME can type an HS code and a country, and the system produces a correct, fully sourced tariff breakdown. A compliance officer can type a company name and get a correct screening result. The tariff computation runs through the tax regime template, not hardcoded US logic.

#### Phase 2: Intelligence — Classification Engine

**Goal:** The system can accept a product description and produce a classification with reasoning. Architecture is global-ready even though knowledge is US-first.

**Deliverables:**
- **K14-US: Classification reference examples curated.** 500-1,000 product → HS code examples from CROSS rulings.
- **Engine 1: Classification Intelligence operational.** LLM-powered classification against the full HTSUS. Produces reasoning chains. Confidence assessment calibrated. Classifies to HS 6-digit (globally valid) and then extends to US 10-digit (HTSUS-specific). This separation is architecturally important — the 6-digit classification is reusable across all jurisdictions.
- **Orchestration: Classification → Tariff pipeline working.** User describes a product → system classifies it → tariff engine computes the stack. End-to-end.
- **Output validation layer.** Every HS code in Engine 1's output is checked against K1-US before display. Every rate in Engine 2's output is verified against K2-US.

**Why second:** This is the hardest engine and the highest risk. It depends on K1-US (HTSUS must already be loaded). It requires prompt engineering, testing, and calibration. Building it after the tariff engine means that when classification produces an HS code, we can immediately show the tariff impact — which is what makes classification tangible ("that HS code means 85% duty").

**Milestone test:** Run the classification testing protocol (100 products, 20 chapters). Achieve the target accuracy at each level. The presenter can type "wireless Bluetooth speaker from China" and get: correct classification, visible reasoning, correct tariff stack, meaningful confidence assessment.

#### Phase 3: Compliance — US Regulatory Screening

**Goal:** The system identifies regulatory requirements beyond duty for US imports.

**Deliverables:**
- **K10-US: PGA-HS mapping loaded.** At minimum, the top 10 PGAs (FDA, USDA, CPSC, EPA, TTB, FCC, ATF, DOT, NHTSA, FWS) mapped to their HS code jurisdiction with specific regulatory citations per agency.
- **Engine 3A: PGA Jurisdiction operational.** Shows which agencies have jurisdiction and what they require.
- **Engine 3C: UFLPA Risk Assessment operational.** Enforcement priority sector mapping + entity screening.
- **Orchestration expanded.** Full US pipeline: classify → tariff → PGA → DPS → UFLPA. All results render in the unified output panel.

**Why third:** PGA and UFLPA add the compliance dimension. After Phase 2, the system classifies and computes duty. After Phase 3, it also tells you who else cares about your product and what they need. This rounds out the US product analysis experience (Scene 1 and Scene 2). US compliance is also the most complex jurisdiction — EU and China compliance frameworks (Phase 4) are structurally simpler.

**Milestone test:** Analyze a cosmetic from Korea (FDA flagged), a children's toy from China (CPSC flagged + UFLPA sector + tariff stacking), and a ceramic mug from Portugal (no PGA). All three should produce correct, complete compliance results.

#### Phase 4: Global Expansion — EU and China Knowledge Layers

**Goal:** The system demonstrates meaningful analytical capability for Tier 2 jurisdictions (EU and China), proving the architecture scales globally. Brazil/India framework knowledge is loaded if audience requires it.

This phase has three sub-phases that can overlap but have a recommended sequence.

##### Phase 4A: EU Knowledge and Engine Globalization

**Goal:** The system computes landed costs for imports into the EU, demonstrating that the tariff engine is truly global.

**Deliverables:**
- **K1-EU: EU Combined Nomenclature (CN) loaded.** 8-digit EU tariff schedule parsed from TARIC. Source: European Commission TARIC database (English, publicly available, well-structured). Cross-linked to HS 6-digit for Engine 1 reuse.
- **K2-EU: EU MFN duty rates loaded.** From TARIC, keyed to CN 8-digit codes.
- **K3-EU: EU trade remedy measures loaded.** Anti-dumping duties, countervailing duties, safeguard measures. Source: TARIC additional duties database.
- **K4-EU: EU safeguard duties loaded.** Steel safeguard product scope and quotas.
- **K6-EU: EU AD/CVD orders loaded.** Active anti-dumping and countervailing duties from EC TRON database.
- **K13-EU: EU VAT rates loaded.** Standard and reduced rates by member state. Source: European Commission VAT information exchange system (VIES).
- **K10-EU: EU compliance framework loaded.** CE marking requirements, REACH chemical regulation (HS-linked), product safety directives mapped to CN codes.
- **EU tax regime template activated.** Pattern A (additive): Customs Duty → Anti-dumping (if applicable) → VAT on (CIF + duty). Template loaded into Engine 2.
- **K18: Exchange rates loaded.** EUR/USD, CNY/USD, BRL/USD from ECB or similar public source. Updated frequency defined.
- **Engine 2 globalized.** The tariff engine accepts destination country as a parameter and dispatches to the appropriate tax regime template. US and EU produce full computations.

**Why EU first in global expansion:** TARIC is the gold standard for tariff data accessibility — English, publicly available, machine-readable, well-documented. It's the lowest-risk expansion and proves the architecture works across jurisdictions before tackling the harder cases (China, Brazil).

**Milestone test:** The same "wireless Bluetooth speaker from China" analyzed for US import AND EU import side-by-side. Both produce correct tariff stacks using their respective tax regime templates. The presenter can switch destination country and see how landed cost changes.

##### Phase 4B: China Knowledge Loading

**Goal:** The system computes landed costs for imports into China, covering the critical China↔US and China↔EU trade lanes.

**Deliverables:**
- **K1-CN: China Customs Tariff Schedule loaded.** 8-digit codes parsed from China Customs published schedule. Source: General Administration of Customs of China (中华人民共和国海关总署). Note: primary source is Chinese-language; WTO Tariff Download Facility provides English 6-digit rates, commercial providers offer 8-digit English translations.
- **K2-CN: China MFN duty rates loaded.** From customs tariff schedule.
- **K3-CN: China retaliatory tariff lists loaded.** China's tariff measures on US goods, EU goods. Source: MOFCOM announcements.
- **K6-CN: China AD/CVD orders loaded.** Active anti-dumping and countervailing duties from MOFCOM.
- **K7-CN: China Unreliable Entity List loaded.** Screening list for Engine 3B.
- **K13-CN: China VAT + consumption tax loaded.** VAT: standard 13% and reduced 9% rates by product category. Consumption tax: applicable to specific categories (cosmetics, jewelry, automobiles, alcohol, tobacco).
- **K10-CN: China CCC certification mapping loaded.** China Compulsory Certification requirements by product category, mapped to HS codes.
- **China tax regime template activated.** Pattern A (additive): Customs Duty → Consumption Tax (if applicable) → VAT on (CIF + duty + consumption tax). Template loaded into Engine 2.

**Why China second:** China is a critical trade lane for the target audience (China→US, China→EU, US→China). The data is public but requires more effort due to language barriers. The architecture proven with EU expansion reduces risk.

**Milestone test:** A product analyzed for import into China with correct duty, consumption tax (if applicable), and VAT computation. A US-origin product shows China's retaliatory tariff impact.

##### Phase 4C: Framework Jurisdictions (Brazil, India — if needed)

**Goal:** Demonstrate that the architecture handles even structurally different tax regimes.

**Deliverables (Brazil — Tier 3):**
- **K1-BR: Brazil NCM schedule loaded.** 8-digit NCM (Nomenclatura Comum do Mercosul) — essentially HS 6-digit + 2-digit Mercosur extension. Source: Receita Federal.
- **K2-BR: Brazil MFN (TEC) rates loaded.** Tarifa Externa Comum from Mercosur.
- **K3-BR: IPI rate table (TIPI) loaded.** IPI rate per NCM code.
- **K4-BR: ICMS state rates loaded.** Rate per state with product-specific exceptions.
- **K13-BR: PIS/COFINS + tax regime parameters loaded.** Standard import rates and grossup solver parameters.
- **Brazil tax regime template activated.** Pattern B (self-referencing/cascading): Import Duty → IPI on (CIF + duty) → ICMS on grossed-up base → PIS/COFINS on grossed-up base. The algebraic grossup formula must be implemented in Engine 2 for Pattern B.

**Deliverables (India — Tier 3):**
- **K1-IN: India Customs Tariff loaded.** 8-digit ITC-HS codes. Source: Indian Customs ICEGATE.
- **K2-IN: India BCD (Basic Customs Duty) rates loaded.**
- **K6-IN: India IGST rate loaded.** Standard 18% on (assessable value + BCD + SWS).
- **India tax regime template activated.** Pattern A (additive with social welfare surcharge): BCD → Social Welfare Surcharge (10% of BCD) → IGST on (CIF + BCD + SWS).

**Why last / conditional:** Brazil's Pattern B tax regime is the most architecturally challenging (algebraic grossup). India is straightforward Pattern A but adds another knowledge set. These are included only if the audience mix requires them — the US/EU/China core covers the primary trade lanes.

**Milestone test:** If Brazil is loaded, the ICMS grossup computation produces the correct effective tax rate (which is always higher than the nominal rate due to self-inclusion). If India is loaded, the cascading surcharge + IGST computation is correct.

#### Phase 5: Regulatory Change Intelligence — Engine 6

**Goal:** The system demonstrates awareness of regulatory change — both prospective (scenario modeling) and reactive (automated impact analysis).

**Deliverables:**
- **K17: Regulatory change signal database seeded.** 20-30 real signals across US, EU, and China covering: announced/proposed tariff changes, enacted changes with future effective dates, recent changes within last 90 days. Sourced from Federal Register, Official Journal of the EU, MOFCOM announcements.
- **Engine 6A: Signal Intelligence operational.** Displays active signals with fuzzy/confirmed status, source attribution, affected HS code ranges.
- **Engine 6B: Scenario Modeling operational.** User selects a signal (e.g., "proposed 100% tariff on EU steel") and the system recomputes landed cost for affected products in the portfolio, showing current vs. projected side-by-side.
- **Engine 6C: Automated Impact Analysis operational.** When a signal transitions from fuzzy to confirmed, the system automatically identifies affected SKUs/trade lanes in the portfolio and generates an impact assessment.
- **Regulatory Change Intelligence screen built.** Integrated into the experience layer with signal timeline, scenario comparison cards, and portfolio impact summary.

**Why here in the sequence:** Engine 6 requires (a) working tariff computation for at least US and EU (Phases 1 + 4A) to make scenario modeling meaningful, and (b) a portfolio concept (K15 dashboard data) to make impact analysis tangible. It does not require Engines 4 or 5. Building it after global expansion means the scenario modeling can show cross-jurisdiction impact — "if China retaliates on US goods, here's the impact on your China-bound shipments AND here's how the same goods entering the EU are unaffected."

**Milestone test:** The presenter shows a signal about a proposed tariff change, runs scenario modeling against the portfolio, sees the cost impact, then shows a confirmed change that triggered automatic impact analysis with affected products highlighted.

#### Phase 6: Experience — Three Application Surfaces and the Golden Path

**Goal:** The full presentation experience is built across all three application surfaces and polished for global scope. The surface-switching moments — the demo's thesis made visible — are designed and tested.

**Deliverables:**

**Platform Intelligence Surface (Command Center):**
- **All analytical screens built.** Product Analysis (multi-destination), Compliance Dashboard, Regulatory Change Intelligence, Trade Lane Comparison, Exception Resolution, Entity Screening, Burning Platform cards.
- **Progressive disclosure implemented.** Summary → reasoning → evidence layers for every analytical output. Smart surfacing: the dashboard triages entries by decision-relevance, not by entry number. Source annotations discoverable on hover, not front-and-center.
- **K15: Dashboard simulated data authored.** AutoParts Global shipment records across US, EU, and China trade lanes. Supplier data, SKU catalog with multi-destination cost profiles.
- **Dashboard engine integration.** FTA alerts, UFLPA alerts, regulatory change alerts, exception summaries generated from engines against K15 data.
- **Trade Lane Comparison screen.** Same product, multiple destinations — side-by-side landed cost from different tax regime templates.
- **Reasoning Chain Renderer polished.** Streaming, progressive, expandable, color-coded.
- **Tariff Stack Renderer polished.** Program-by-program with annotations. Adapts to destination country's tariff structure.
- **Regulatory Change alerts integrated into dashboard.** Signal count badge, drill-down to scenario modeling.

**Shipper Surface (Workflow-Embedded):**
- **Ship-Ready Panel built.** Takes engine output and renders: status (traffic-light), total landed cost (one number with optional breakdown), document checklist (plain-language with auto-generation markers), delivery estimate. No HTSUS codes, no tariff program names, no GRI citations, no confidence scores.
- **Plain-language translation layer implemented.** Engine output → shipper language. "Section 301 surcharge applicable" → "Additional import duty applies for products from China." "PGA jurisdiction: FDA — 21 CFR 177" → "Food contact declaration required." This translation layer is a distinct component, not ad hoc string replacement.
- **Notification rendering for regulatory changes.** Engine 6 signals → plain-language notifications. "Your shipping costs to the US will increase by $4.20 on March 1. No action needed — pricing updated automatically." Status levels (CONFIRMED/PROPOSED/DISCUSSED) are NOT exposed — the notification handles the nuance internally.
- **Document auto-generation stubs.** Commercial invoice and packing list marked as "auto-generated" where appropriate. In production, these would be populated from order data; for the demo, they demonstrate the concept.

**Buyer Surface (Checkout-Embedded):**
- **Checkout price component built.** Total landed price in destination currency with "duties included" badge. Optional expandable: product / shipping / duties-taxes — three lines.
- **Multi-currency rendering.** Price displays in the buyer's local currency (USD, EUR, CNY) based on destination.
- **Today vs. Tomorrow comparison.** Before/after split-screen showing surprise charges vs. transparent pricing. The "Tomorrow" side is live (Engine 2 output); the "Today" side is static.

**Cross-Surface:**
- **Surface switching navigation.** A clear, deliberate mechanism for the presenter to navigate between surfaces. Not a tab within a screen — a transition to a fundamentally different layout. The switch should feel like moving to a different application, not toggling a view.
- **Shared engine API layer verified.** Engines return full-depth results regardless of surface. Each surface's rendering layer extracts and reformats as appropriate. Verify that no analytical internals leak into Shipper or Buyer surfaces.
- **Animation and transitions.** Scene navigation, progressive rendering, enrichment animations. Surface switching should have a distinct transition that signals "different application" to the audience.
- **Resequenceability tested.** Every scene is independently accessible. Every surface is independently accessible from any scene. The presenter can navigate to any scene from any other scene, and to any surface from any scene, without setup dependencies.

**Why sixth:** The experience layer can be built concurrently with Phases 2-5 (against mock engine responses), but it reaches full fidelity only when the engines are working and global knowledge is loaded. This phase integrates everything into the coherent presentation experience. The three surfaces share engine interfaces but require independent design and testing — the Shipper surface in particular requires the plain-language translation layer, which is a meaningful component.

**Milestone test:** A full 60-minute rehearsal of the golden path with global scope and surface switching. The presenter runs through all scenes including: (a) at least two surface switches (Platform Intelligence → Shipper in Scene 1, Platform Intelligence → Buyer in Scene 3), (b) cross-jurisdiction comparisons, (c) regulatory change intelligence with Shipper notification reveal, (d) the triple surface reveal in Scene 6. One "bring your own product" test with a product the system hasn't seen, analyzed for at least two destination countries, shown across all three surfaces.

#### Phase 7: Depth — FTA Qualification and Exception Analysis

**Goal:** The remaining analytical engines are operational, completing the full suite.

**Deliverables:**
- **K11-US: USMCA rules of origin structured.** ~50 categories with executable qualification logic.
- **K11-EU: EU FTA rules of origin structured.** At minimum, EU-UK TCA or EU-Japan EPA to demonstrate non-US FTA capability.
- **Engine 4: FTA Qualification operational.** USMCA analysis with BOM input, tariff shift and RVC computation. EU FTA at framework level.
- **K12-US: CBP CROSS rulings indexed.** 1,000-2,000 rulings searchable by HS code and topic.
- **Engine 5: Exception Analysis operational.** Hold scenario → rulings search → analysis → draft response.
- **Dashboard FTA drill-down.** The USMCA savings alert works with Engine 4.
- **Exception Resolution scene.** Scene 5 runs on Engine 5 against authored hold scenarios.

**Why seventh:** These engines are important for the demo's depth but not gating for the core experience. A demo with Engines 1-3 + Engine 6 working, global scope live, and Engines 4-5 in realistic stub mode is already extremely powerful. Phase 7 adds analytical depth and makes Scenes 2 and 4 fully live rather than partially stubbed.

**Milestone test:** The presenter clicks into the USMCA savings alert and the system runs a live FTA qualification. The presenter enters a hold scenario and the system generates a response citing real rulings. An EU FTA scenario shows the framework is jurisdiction-aware.

#### Phase 8: Hardening and Rehearsal

**Goal:** The system is reliable, the data is current, the presenter is prepared across all jurisdictions.

**Deliverables:**
- **Rate data validation across all loaded jurisdictions.** Trade compliance SME validates US tariff rates against current HTSUS. EU rates validated against live TARIC. China rates validated against current customs schedule.
- **Classification calibration.** Run 100+ products through the classification engine, review results, tune prompts if needed. Include products from multiple chapters and test for US and EU sub-heading accuracy.
- **Regulatory change signals refreshed.** K17 updated to reflect most current signals as of demo date. All "as of" dates are current.
- **Exchange rates refreshed.** K18 updated to rates within 48 hours of demo.
- **Error handling tested.** Every failure mode in the orchestration section has been triggered and handled gracefully, including cross-jurisdiction fallback.
- **Performance profiled.** Classification < 8 seconds, tariff computation < 1 second, full pipeline < 15 seconds, trade lane comparison < 5 seconds.
- **Fallback mode tested.** If the LLM is unavailable, the system falls back to pre-computed results for the golden path products.
- **Surface integrity validated.** Verify that no analytical internals (HTSUS codes, tariff program names, engine annotations, confidence scores) leak into the Shipper or Buyer surfaces. Test every engine output path through all three rendering layers.
- **Plain-language translation layer audited.** Every Shipper-surface message reviewed for accuracy and clarity. "Additional import duty applies for products from China" must be traceable to the Section 301 engine output. Simplicity must not sacrifice accuracy.
- **Surface switching rehearsed.** Presenter practices the switching moments — Platform Intelligence → Shipper (Scene 1), Platform Intelligence → Buyer (Scene 3), Platform Intelligence → Shipper (Scene 4), and the triple reveal (Scene 6). The transitions should feel deliberate, not hurried. The pause-and-contrast rhythm is practiced.
- **Resequenceability rehearsed.** Presenter practices non-linear navigation, audience-driven scene selection, surface switching from any scene, and recovering from unexpected order changes.
- **3+ full rehearsals** with the presenter and a hostile test audience. At least one rehearsal focused on global trade lanes. At least one rehearsal where the audience drives the scene order. At least one rehearsal focused specifically on the surface switching moments — timing, pacing, the "all of that became this" contrast.

### 14.3 Critical Dependencies

```
US KNOWLEDGE DEPENDENCIES:

K1-US (HTSUS) ────────→ Engine 1 (Classification)
                      ├─→ Engine 2 (Tariff, US template)
                      └─→ Engine 3A (PGA)

K3-US/K4-US/K5-US ────→ Engine 2 (US surcharges)

K7-US/K8-US/K9-US ────→ Engine 3B (DPS)
                      └─→ Engine 3C (UFLPA)

K16-US (Fee Schedule) ─→ Engine 2 (MPF/HMF)

K11-US (USMCA Rules) ──→ Engine 4 (FTA)

K12-US (CROSS Rulings) → Engine 5 (Exception)

K14-US (Examples) ──────→ Engine 1 (improves quality)

K15 (Sim Data) ─────────→ Dashboard (Scene 2)


GLOBAL KNOWLEDGE DEPENDENCIES:

K1-EU (CN/TARIC) ───────→ Engine 2 (Tariff, EU template)
                        └─→ Engine 3A (EU compliance, CE/REACH)
K3-EU (EU trade remedies) → Engine 2 (EU surcharges)
K6-EU (EU AD/CVD) ─────→ Engine 2 (EU trade remedies)
K13-EU (VAT rates) ────→ Engine 2 (EU VAT computation)

K1-CN (China Tariff) ───→ Engine 2 (Tariff, China template)
K3-CN (Retaliatory) ────→ Engine 2 (China surcharges)
K6-CN (China AD/CVD) ───→ Engine 2 (China trade remedies)
K13-CN (VAT/Consumption) → Engine 2 (China tax computation)

K1-BR (NCM) ────────────→ Engine 2 (Tariff, Brazil template)
K3-BR/K4-BR/K13-BR ────→ Engine 2 (Brazil cascading tax — Pattern B)

K18 (Exchange Rates) ───→ Engine 2 (multi-currency display)


ENGINE 6 DEPENDENCIES:

K17 (Regulatory Signals) → Engine 6A (Signal Intelligence)
Engine 2 (any jurisdiction) → Engine 6B (Scenario Modeling — needs tariff recomputation)
K15 (Portfolio data) ────→ Engine 6C (Impact Analysis — needs SKU portfolio)


ORCHESTRATION DEPENDENCIES:

Engine 1 ────────────────→ Orchestration (gating step)
Engine 2 ────────────────→ Orchestration
Engine 3 ────────────────→ Orchestration
Engine 4 ────────────────→ Dashboard FTA alert
Engine 5 ────────────────→ Exception Resolution scene
Engine 6 ────────────────→ Regulatory Change Intelligence screen
```

### 14.4 What Can Be Parallelized

**Within US Foundation (Phases 1-3):**
- **K1-US parsing** and **K7-US/K8-US/K9-US loading** are independent. Start both simultaneously.
- **K3-US/K4-US/K5-US loading** depends on K1-US (you need the HTSUS structure to cross-reference program lists), but can start as soon as K1-US's code structure is available.
- **Engine 2** can be built as K1-US/K2-US are being loaded — build the tax regime template architecture and US stacking logic while the data is being prepared.
- **Engine 3B** is independent of all other engines and K7-US/K8-US/K9-US are easy to load. Can be completed very early.
- **Experience layer** UI components can be built against mock engine interfaces from the start.
- **K12-US (CROSS rulings)** and **K14-US (classification examples)** can be curated in parallel with engine development.
- **K15 (simulated company data)** can be authored anytime — it's creative work, not data engineering.

**Global Expansion (Phase 4):**
- **EU knowledge loading (Phase 4A)** can begin as soon as Phase 1 is complete (the tax regime template architecture exists). It does NOT depend on Phases 2 or 3.
- **China knowledge loading (Phase 4B)** can run in parallel with Phase 4A — the knowledge sets are independent. However, 4A is recommended first because TARIC is easier and proves the globalization pattern.
- **K18 (Exchange Rates)** is trivially loadable and can be done as soon as multi-currency display is needed.
- **All EU/CN knowledge loading** is independent of K11, K12, K14 (US depth collections). Global breadth and US depth can be parallelized.

**Regulatory Change Intelligence (Phase 5):**
- **K17 seeding** can begin as soon as there's a signal schema defined — it's research and data entry work, parallelizable with engine development.
- **Engine 6A (Signal Intelligence)** can be built independently once K17 schema exists.
- **Engine 6B (Scenario Modeling)** depends on Engine 2 being globalized (Phase 4A complete).
- **Engine 6C (Impact Analysis)** depends on K15 (portfolio data) existing.

**Key parallelization opportunities across phases:**
- Phases 4A, 4B, and 5 (K17 seeding) can all run in parallel once Phase 1 is complete.
- Experience layer work (Phase 6) can begin against mock interfaces at any time; it reaches full fidelity incrementally as each phase completes.
- Phase 7 (FTA/Exception depth) is independent of Phase 4 and Phase 5 and can be parallelized with either.

---

## Appendix: Open Questions for Stakeholder Discussion

These questions don't have answers in the design — they require stakeholder input.

1. **How current must tariff data be for the demo?** IEEPA rates can change with an executive order. EU trade remedies can shift on short notice. China retaliatory tariffs have been modified multiple times. Engine 6 (Regulatory Change Intelligence) shows the system's awareness of change, but the underlying rate data must be current as of demo day. What is the refresh cadence?

2. **Which HS chapters matter most for the target audience?** If we know the attendees' industries, we can prioritize classification quality and tariff data depth for the relevant chapters. This applies across all jurisdictions — knowing "auto parts and consumer electronics" lets us validate US, EU, AND China data for the same chapters.

3. **Which trade lanes are most critical for first demo audiences?** The system supports China→US, China→EU, US→EU, US→China at Tier 2 depth. Should any of these be elevated to Tier 1 depth? Are there other trade lanes (e.g., Vietnam→US, India→EU) that specific audiences will expect?

4. **Is USMCA the right FTA to demonstrate, or should it also include an EU FTA?** USMCA is the highest-impact for US imports from Mexico/Canada. But if stakeholders care about EU-UK TCA, EU-Japan EPA, or RCEP (for China/ASEAN), the FTA engine scope should be adjusted. Engine 4 can support multiple FTAs but each requires dedicated knowledge loading.

5. **Should the system accept image input in the demo?** Image-based classification (upload a product photo) is impressive but adds engineering complexity (multimodal LLM integration, image processing pipeline). Is it worth the investment for the demo, or is text-based classification sufficient to prove the concept?

6. **What is the target date for the first demo?** This determines: (a) whether Phases 4B/4C (China/Brazil knowledge) are included or deferred, (b) whether Engine 6 is live or in realistic stub mode, (c) whether Phase 7 (FTA/Exception depth) is included. The minimum viable demo is Phases 1-3 + 4A (US full depth + EU meaningful) + Phase 6 (experience layer).

7. **Which regulatory change signals should be seeded for K17?** Engine 6 is most compelling when the signals are real and current. Which pending trade actions are most relevant to the target audience? The signal database should be curated for maximum impact — 10 high-quality, real signals are better than 30 generic ones.

8. **Is Brazil a confirmed audience requirement?** Brazil's Pattern B (cascading) tax regime is the most architecturally distinct implementation. If Brazilian investors are a confirmed audience, Phase 4C should be prioritized. If they are speculative, the framework can be demonstrated narratively without full knowledge loading.

9. **What is the Shipper surface integration target for the narrative?** The demo shows the Shipper surface as a standalone panel and says "in production, this lives inside Shopify / your ERP / your shipping platform." Should we name a specific integration target (Shopify is the most recognizable for e-commerce audiences)? Or keep it generic? The answer may depend on the audience — a logistics audience may care about ERP/TMS integration; an e-commerce audience may care about Shopify/WooCommerce.

10. **How deeply should the Buyer surface simulate checkout?** The current design shows a mockup checkout with price, badge, and breakdown. Should it include a more realistic e-commerce page (product image, reviews, shipping options) to make the context more vivid? Or does the minimal mockup — three lines, one price — make the point more powerfully through its simplicity?

---

*This specification is part of the FedEx Global Clearance Knowledge Base and should be read alongside:*
- *[The Golden Path: Working System Design](golden_path_demo_design.md) — The experience this system enables*
- *[Conviction Strategy](conviction_strategy.md) — The broader experience portfolio*
- *[Authenticity Audit](authenticity_audit.md) — Skeptical review of all data points and terminology*
- *[Presenter Notes](presenter_notes.md) — Voiceover guidance for areas of inherent complexity*
