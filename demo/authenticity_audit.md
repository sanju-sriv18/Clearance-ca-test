# Authenticity Audit: Golden Path Demo & Build Specification

## Skeptical Review of Every Term, Data Point, Field Label, and Interaction

---

> **Purpose:** To identify everything in our design documents that a trade professional, customs broker, CBP officer, compliance manager, or logistics executive would flag as inauthentic, incorrect, or amateur. The test: does any element create a distraction by inviting a "that's not how it works" comment from someone in the audience?

> **Methodology:** Every HS code, duty rate, fee calculation, field label, regulatory citation, and interaction in the Golden Path Design and Build Specification was cross-referenced against: the current HTSUS (via USITC), CBP fee schedules, regulatory text (CFR/CPG), and standard trade industry terminology.

---

## Severity Levels

| Level | Definition |
|---|---|
| **WRONG** | Factually incorrect. A trade professional will notice immediately. Must be fixed before any demo. |
| **MISLEADING** | Technically defensible but will cause confusion or invite a correction from a knowledgeable audience member. Should be fixed. |
| **IMPRECISE** | Uses informal or slightly off terminology that a purist would flag. Fix if easy; acknowledge if not. |
| **FINE** | Reviewed and confirmed authentic. No action needed. |

---

## CRITICAL FINDINGS (Must Fix)

### 1. WRONG — MFN Duty Rate for Ceramic Mug (HS 6912.00.48)

**What we say:** "MFN Duty: 8.0% ad valorem"

**What it actually is:** The General (MFN) rate for HS 6912.00.48 is **9.8% ad valorem**, not 8.0%. Furthermore, a rate adjustment was noted for 2026.

**Where it appears:** Golden Path Design, Scene 1 reasoning chain; Scene 3 consumer checkout; the original Data Bible

**Risk:** A trade professional or anyone who looks up the rate will see 9.8%, not 8%. This instantly undermines the "every number is verifiable" promise. Since this is the FIRST product the system analyzes in the demo, getting it wrong poisons everything that follows.

**Fix:** Change all references to 9.8%. Recalculate the landed cost:
- Duty: $38.00 × 9.8% = $3.72 (was $3.04)
- This cascades through the total landed cost and the consumer checkout scene

**Source:** [HTSUS 6912.00.48 — UNIS](https://www.unisco.com/hts/69120048), [USITC HTS](https://hts.usitc.gov/)

---

### 2. WRONG — MPF Calculation for a $38 Parcel

**What we say:** "MPF: $0.35 (minimum applies)"

**What's actually happening:** The $0.35 figure is neither the percentage (0.3464% of $38 = $0.13) nor any current minimum.

The real question is: **what entry type is a $38 parcel?**

A shipment valued at $38 is well below the $2,500 threshold for formal entry. With de minimis eliminated, this parcel requires entry — but it would almost certainly use **informal entry** (Entry Type 11), not formal entry.

| Entry Type | MPF | Source |
|---|---|---|
| Formal entry (value > $2,500) | 0.3464%, min $33.58, max $651.50 (FY2026) | [CBP MPF Table](https://www.cbp.gov/document/guidance/merchandise-processing-fee-mpf-table) |
| Informal entry, automated | **$2.69** (FY2026) | [CBP FY2026 Fee Adjustment](https://www.federalregister.gov/documents/2025/07/23/2025-13869/customs-user-fees-to-be-adjusted-for-inflation-in-fiscal-year-2026-cbp-dec-25-10) |
| Informal entry, CBP-prepared | $12.09 (FY2026) | Same source |

**Where it appears:** Golden Path Design (Scene 1 landed cost, Scene 3 consumer checkout); Build Specification (Engine 2 outputs table, which cites the now-outdated FY2025 minimums of $31.67/$614.35)

**Risk:** A customs broker will immediately ask: "Why is this a formal entry? It's a $38 parcel. The MPF should be about $2.69 for an informal entry." This is a fundamental misunderstanding of entry types that signals the team doesn't understand how low-value shipments are processed.

**Fix:**
- For the Maria scenario ($38 mug): use informal entry MPF of $2.69
- Recalculate landed cost accordingly
- For the David scenario (higher-value commercial entries): formal entry MPF is correct
- The Build Specification must address the formal/informal entry distinction — the system should determine entry type based on value and show the appropriate MPF
- Update all FY2026 fee amounts: formal minimum $33.58, maximum $651.50

**Deeper question this raises:** With de minimis eliminated, what entry type DO low-value express parcels use? CBP has been evolving guidance on this. The system should know about:
- Informal entry (Type 11) for goods ≤$2,500
- Formal entry (Type 01) for goods >$2,500
- The now-deprecated Type 86 (Section 321 + advance data)
- Whether FedEx or the consumer is the importer of record

The presenter must be prepared for the question "who is the IOR on a $38 parcel from Portugal?" and have a clear answer.

**Source:** [CBP MPF Table](https://www.cbp.gov/document/guidance/merchandise-processing-fee-mpf-table), [Federal Register FY2026 Fee Adjustment](https://www.federalregister.gov/documents/2025/07/23/2025-13869/customs-user-fees-to-be-adjusted-for-inflation-in-fiscal-year-2026-cbp-dec-25-10)

---

### 3. WRONG — Bluetooth Speaker Classification

**What we say:** The system would classify to "8518.22 (loudspeakers in enclosures) or 8527.92 (radio reception apparatus)"

**What it actually is:** A portable Bluetooth speaker with a single driver is classified as **8518.21.0000** ("Single loudspeakers, mounted in their enclosures"), NOT 8518.22.

- 8518.21 = Single loudspeaker in enclosure
- 8518.22 = Multiple loudspeakers in the same enclosure (like a sound bar)

CBP has issued rulings specifically classifying Bluetooth speakers under 8518.21.0000. The MFN rate is **Free** (0%).

**Where it appears:** Golden Path Design, Scene 1 second product analysis

**Risk:** We use this example to show "classification ambiguity" between 8518 and 8527. The ambiguity concept is valid (is it a speaker or a radio?), but we cite the wrong 8518 subheading. A trade professional who has classified speakers before will notice .21 vs .22 immediately. More importantly, since 8518.21 has a 0% MFN rate (not a duty-bearing rate), the tariff drama changes — the drama comes entirely from 301 + IEEPA, not from MFN + 301 + IEEPA.

**Fix:**
- Change 8518.22 to 8518.21 everywhere
- Update the tariff stack: MFN 0% + Section 301 7.5% (List 4a, not List 1 25%) + IEEPA 60% = 67.5% total
- Note: Section 301 List/rate depends on exactly which 301 action covers 8518.21 — this MUST be verified against the current 301 product lists. The original design assumed List 1 at 25%, but 8518.21 may be on a different list.
- The classification ambiguity between 8518 (loudspeaker) and 8527 (radio) is still valid — keep that, but use the correct subheading

**Source:** [CBP CROSS Rulings for 8518.21](https://rulings.cbp.gov/search?term=8518.21.0000), [CMTradelaw Bluetooth Speaker Ruling](https://www.cmtradelaw.com/2020/08/customs-ruling-of-the-week-classification-of-drips-bluetooth-speaker/), [Flexport HS 851821](https://www.flexport.com/data/hs-code/851821-single-loudspeakers-mounted-in-their-enclosures/)

---

### 4. WRONG — CPTPP Reference for US-Canada Trade

**What we say (Scene 3):** "What about Canada under CPTPP?"

**Reality:** The United States is **not a member of CPTPP**. The US withdrew from the TPP in January 2017 and has not rejoined under any subsequent administration. US-Canada trade is governed by USMCA, not CPTPP.

**Where it appears:** Golden Path Design, Scene 3 consumer landed cost

**Risk:** This is an embarrassing factual error. Any trade policy professional in the room will flag this immediately. It signals the team doesn't know basic trade agreement geography.

**Fix:** Replace the CPTPP reference. Options:
- "What if Maria ships to Canada? Under USMCA, the duty could be zero."
- "What about shipping to the UK?" (no FTA with the US — demonstrates a different scenario)
- "What about Japan under USJTA?" (the US-Japan Trade Agreement provides tariff reductions on some goods)

**Source:** [CPTPP Overview — Congress.gov](https://www.congress.gov/crs-product/IF12078)

---

## SIGNIFICANT FINDINGS (Should Fix)

### 5. MISLEADING — "Product Value" as a Label

**What we say:** "Product Value: $38.00" in the landed cost breakdown

**Correct terminology:** In customs, the basis for duty is "Customs Value" or "Entered Value" (defined as transaction value under 19 USC 1401a). "Product Value" is not a recognized customs term.

**Risk:** A broker might wonder: "Do they mean transaction value? FOB value? Retail price? These are different things in customs."

**Fix:** Use "Customs Value" or "Declared Value" in the tariff and landed cost outputs. "Product Value" is acceptable in consumer-facing contexts (Scene 3 checkout), but not in the analytical engine output.

---

### 6. MISLEADING — State Tax Calculation Base

**What we say:** Illinois state tax 7.56% applied to what appears to be (value + shipping + duty) = $53.44, producing $4.04

**Issue:** In most US states, import duty is NOT included in the taxable base for state sales/use tax. The tax is typically on (value + shipping) = $50.40 for a taxable amount. At 7.56%, that's $3.81, not $4.04.

The taxable base varies by state. Illinois specifically: sales tax applies to the selling price, which may or may not include shipping depending on whether shipping is separately stated.

**Risk:** A tax professional or e-commerce operator would question the calculation base.

**Fix:** Research Illinois's specific rules for import sales/use tax calculation. For the demo, use (declared value) as the taxable base unless shipping is shown to be included. Show the calculation explicitly: "State Tax (7.56% of $38.00): $2.87" or whatever the correct base and rate produce.

---

### 7. MISLEADING — "DPS lists" Terminology

**What we say:** "Restricted Party Check: No matches on DPS lists"

**Issue:** "DPS" (Denied Party Screening) is the process, not the lists. The lists have specific names: OFAC SDN List, BIS Entity List, UFLPA Entity List, etc. Saying "DPS lists" sounds like someone who knows the acronym but not what's behind it.

**Fix:** Either say "Restricted Party Screening: No matches found on OFAC, BIS, or UFLPA lists" (specific and credible) or "Denied Party Screening: Clear" (process-oriented). Don't say "DPS lists."

---

### 8. MISLEADING — "CLEAR FOR SHIPMENT" Status

**What we say:** "Status: CLEAR FOR SHIPMENT" at the bottom of the compliance panel

**Issue:** "Clear for shipment" sounds like an export authorization. For imports, the relevant status would be:
- "NO COMPLIANCE HOLDS IDENTIFIED"
- "ADMISSIBLE" (CBP's term when goods are released)
- "CLEAR" (informal but directionally right)
- "ELIGIBLE FOR ENTRY" (more precise)

No real customs system displays "CLEAR FOR SHIPMENT."

**Fix:** Use "NO COMPLIANCE HOLDS IDENTIFIED" or "ENTRY ELIGIBLE — No PGA holds, no restricted party matches." This sounds like what an actual compliance system would display.

---

### 9. MISLEADING — Build Specification MPF Numbers Are FY2025, Not FY2026

**What the build spec says:** "MPF minimum $31.67, maximum $614.35 for formal entries"

**Current (FY2026, effective Oct 1, 2025):** Minimum $33.58, maximum $651.50

**Where it appears:** Build Specification, Engine 2 outputs table

**Fix:** Update to FY2026 amounts. Add a note that these are adjusted annually for inflation and must be verified before demo.

---

### 10. IMPRECISE — 21 CFR 109.16 Citation for Lead in Ceramics

**What we say:** "If glaze contains lead above FDA threshold, FDA may assert jurisdiction under 21 CFR 109.16"

**What 21 CFR 109.16 actually covers:** Ornamental and decorative ceramicware labeling requirements — specifically, it says that ceramic items that LOOK food-safe but contain lead must be labeled as "Not for Food Use." It is about labeling obligations for decorative ceramicware, not about the lead testing threshold itself.

**The correct references are:**
- **CPG Sec. 545.450** — FDA's Compliance Policy Guide for lead contamination in pottery/ceramics, including action levels for leachable lead
- **21 CFR 109.16** — labeling requirements for ornamental ceramicware (narrower scope)
- **FDA Guidance on Imported Traditional Pottery** — covers "Lead Free" claims

**Risk:** A regulatory affairs professional or FDA-experienced compliance officer would note that 109.16 is about labeling decorative ceramics, not about asserting jurisdiction over lead-containing food-safe ceramics. The system's note would be directionally right but technically imprecise.

**Fix:** Change the citation to: "FDA may test imported ceramic foodware for leachable lead under CPG Sec. 545.450. Description states 'food-safe' — assumed below FDA action levels for leachable lead." This is more precise and cites the right authority.

**Source:** [CPG Sec. 545.450 — FDA](https://www.fda.gov/regulatory-information/search-fda-guidance-documents/cpg-sec-545450-pottery-ceramics-import-and-domestic-lead-contamination), [21 CFR 109.16 — eCFR](https://www.ecfr.gov/current/title-21/chapter-I/subchapter-B/part-109/subpart-A/section-109.16)

---

### 11. IMPRECISE — "Certificate of Origin" vs. "Certification of Origin" (USMCA)

**What we say:** "Certification of Origin elements (as specified in USMCA Chapter 5, Article 5.2)" and "Certificate of Origin — auto-generated, ready for review"

**Issue:** USMCA deliberately moved away from a formal "Certificate of Origin" (NAFTA had a specific form — CBP 434). Under USMCA, the requirement is a **"certification of origin"** — a set of minimum data elements (9 specified in Article 5.2) that can appear on any document (invoice, commercial document, standalone form). There is no required format.

The difference matters: saying "Certificate of Origin" implies a formal document with a specific format. Saying "certification of origin" implies the data elements required by the treaty. Trade professionals who work with USMCA daily will notice this distinction.

**Fix:** Use "USMCA certification of origin" (lowercase, referring to the data requirements). Show the 9 required data elements per Article 5.2 rather than a certificate format.

---

### 12. IMPRECISE — "Origin" as a Form Field Label

**What we say:** The input form has a field labeled "Origin: [Portugal]"

**Issue:** "Origin" is ambiguous in trade. It could mean:
- Country of origin (where the goods were manufactured — relevant for duty)
- Country of export (where the goods shipped from — can be different)
- Country of shipment origin (logistics term)

In customs, these are distinct. A product manufactured in China but shipped from a Hong Kong warehouse has country of origin = China and country of export = Hong Kong. Tariffs apply based on country of origin.

**Fix:** Label the field "Country of Origin" (full term, no ambiguity). The dropdown should show full country names: "Portugal (PT)" rather than just "Portugal."

---

### 13. IMPRECISE — "Altana AI" Named as Data Source

**What we say:** "Source: Altana AI supply chain intelligence" on the UFLPA risk alert

**Issue:** Naming a specific vendor implies a partnership or data relationship that may not exist. If FedEx doesn't have an agreement with Altana, this creates a false impression. If they do, naming them in a pre-funding demo may require permission.

**Fix:** Use "Supply chain intelligence provider" or "Third-party supply chain mapping" without naming a specific vendor. If the presenter is asked who, they can mention Altana as one example of such a provider.

---

## ITEMS CONFIRMED AUTHENTIC

### 14. FINE — HS 6912.00.48 as a Classification Code

The code exists in the current HTSUS. It falls under "Ceramic tableware, kitchenware, other household articles and toilet articles, other than of porcelain or china" with a catch-all subheading of .48 for "Other: Other: Other."

The classification reasoning (Section XIII → Chapter 69 → Heading 6912 → not porcelain, therefore not 6911 → subheading 6912.00.48) is correct.

---

### 15. FINE — Section 301 Only Applies to China-Origin Goods

The system says "Section 301: N/A — Portugal is not China." Section 301 tariffs currently apply to goods that are products of China. This is correct. The phrasing is informal but accurate.

---

### 16. FINE — Section 232 Applies to Steel and Aluminum

The system says "Section 232: N/A — ceramic, not steel/aluminum." Section 232 tariffs apply to steel (primarily Chapters 72-73) and aluminum (Chapter 76) products. Ceramics are not covered. Correct.

---

### 17. FINE — HMF Not Applicable for Air Shipments

"HMF: N/A (air shipment)" — Harbor Maintenance Fee applies only to cargo loaded or unloaded at US ports (ocean/waterborne commerce). Air cargo is exempt. The rate (0.125% of value) is also correct.

---

### 18. FINE — General Rules of Interpretation Numbering

GRI 1 through 6 are referenced correctly in the build specification. The descriptions of GRI 1 (terms of headings + notes), GRI 2(a) (incomplete articles), GRI 2(b) (mixtures), GRI 3 (multiple headings), GRI 4 (most akin), GRI 5 (containers), GRI 6 (sub-classification) are accurate.

---

### 19. FINE — OFAC SDN List and BIS Entity List as Screening Sources

These are the standard restricted party lists. The list names, maintaining agencies, and general update frequencies are correct.

---

### 20. FINE — UFLPA Enforcement Priority Sectors

Cotton, tomatoes, polysilicon, and PVC are correctly identified as DHS enforcement priority sectors. The HS code ranges mapped to these sectors are reasonable (Chapters 52/61-62 for cotton, 0702/2002 for tomatoes, 2804/8541 for polysilicon, 3904/3917/3918 for PVC).

---

### 21. FINE — 4 Million Parcels Per Day De Minimis Volume

This figure has been cited by CBP and industry sources. It is defensible.

---

### 22. FINE — CBP CROSS Database Reference

"rulings.cbp.gov" is the correct URL for the Customs Rulings Online Search System. It is publicly accessible and searchable.

---

### 23. FINE — Cable Assembly Classification Ambiguity (8544.42 vs. 8544.49)

The distinction between "fitted with connectors" (8544.42) and "other" (8544.49) is a real and common classification question. This is a realistic exception scenario.

---

## ADDITIONAL CONCERNS THAT DON'T APPEAR IN THE DOCUMENTS BUT SHOULD

### 24. Entry Type / Importer of Record — Not Addressed Anywhere

The system analyzes products and computes landed cost but never addresses:
- **Who is the importer of record?** For Maria's Shopify mug, is it Maria (the seller), the consumer (the buyer), or FedEx (as customs broker acting on behalf of one of them)?
- **What entry type is used?** Informal (Type 11) for ≤$2,500? A modified formal entry?
- **Who pays the duty?** Under DDP (Delivered Duty Paid), the seller pays. Under DDU (Delivered Duty Unpaid), the buyer pays.

These questions are fundamental to how clearance actually works. A trade professional WILL ask "who's the IOR?" and the system should have a position, even if it's configurable.

**Recommendation:** Add a field or assumption for Incoterms (DDP/DDU/FOB/CIF). For the Maria scenario, assume DDP (Maria's Shopify listing includes all costs). For the consumer checkout scene, show DDP vs. DDU as the key difference.

---

### 25. "Declared Value" Needs Clarification on What's Included

The system takes "Declared Value: $38.00" — but in customs, the declared value (transaction value) may need to include or exclude various elements:
- Assists (tools/dies provided by the buyer)
- Royalties/license fees
- Proceeds of subsequent resale accruing to the seller
- Packing costs
- Selling commissions

For a simple Shopify sale ($38 retail price, direct to consumer), the transaction value IS the retail price. But for the David scenario (commercial importer with more complex transactions), the system should at least acknowledge that customs valuation can be more complex than "what's on the invoice."

**Recommendation:** For the demo, using retail/invoice price as declared value is sufficient. But the presenter should be prepared for the question "does this handle assists and royalties?" and answer: "In production, yes — the system would capture valuation adjustments. For this demo, we're using transaction value."

---

### 26. The Dashboard Volume (4,847 Entries Overnight) Is Implausible for the Company Profile

AutoParts Global is described as having "100 suppliers and 1,000 SKUs" with imports from Guadalajara 3x/week. This profile suggests maybe 20-50 entries per week, not 4,847 per night.

4,847 entries per day is more appropriate for a very large broker, a major retailer, or FedEx's own brokerage operation — not a mid-sized auto parts importer.

**Fix:** Either:
- Reduce the volume to something plausible for a mid-sized importer (e.g., "47 entries filed overnight, 45 cleared, 2 need attention")
- Change AutoParts Global's profile to a much larger company (Fortune 500 auto manufacturer)
- Frame the dashboard as FedEx's operational view across multiple customers (which would make 4,847 realistic)

---

### 27. HS Code Format Inconsistency

The documents alternate between formats:
- "6912.00.48" (8-digit with dots)
- "6912.00.4800" (10-digit with dots)
- "HS 6912.00.48" (with "HS" prefix)
- "HTSUS 6912.00.4800" (with "HTSUS" prefix)

In practice:
- The HS system (international, WCO) goes to 6 digits: 6912.00
- The HTSUS (US-specific) goes to 10 digits: 6912.00.4800
- ACE filings use 10 digits without dots: 6912004800
- In written trade communication, 8-digit is most common with dots: 6912.00.48

**Fix:** Standardize on one format. Recommendation: use 10-digit HTSUS format with dots (6912.00.48.00) in engine outputs, and HS 6-digit (6912.00) when discussing international classification.

---

## SUMMARY: PRIORITY FIXES BEFORE ANY DEMO

| # | Issue | Severity | Effort |
|---|---|---|---|
| 1 | MFN rate for 6912.00.48 is 9.8%, not 8.0% | WRONG | Low — change number, recalculate landed cost |
| 2 | MPF for $38 parcel should use informal entry rate ($2.69), not $0.35 or formal entry minimum | WRONG | Medium — requires entry type logic in the system |
| 3 | Bluetooth speaker is 8518.21 (single), not 8518.22 (multiple) | WRONG | Low — change number |
| 4 | US is not in CPTPP — remove reference | WRONG | Low — replace example |
| 5 | "Product Value" → "Customs Value" or "Declared Value" | MISLEADING | Low |
| 6 | State tax calculation base — research correct base, recalculate | MISLEADING | Medium |
| 7 | "DPS lists" → name the actual lists | MISLEADING | Low |
| 8 | "CLEAR FOR SHIPMENT" → customs-appropriate status language | MISLEADING | Low |
| 9 | Update MPF minimums/maximums to FY2026 amounts | MISLEADING | Low |
| 10 | 21 CFR 109.16 → CPG Sec. 545.450 for lead testing | IMPRECISE | Low |
| 11 | "Certificate" → "certification" of origin for USMCA | IMPRECISE | Low |
| 12 | "Origin" → "Country of Origin" as field label | IMPRECISE | Low |
| 13 | Remove specific vendor name (Altana AI) | IMPRECISE | Low |
| 24 | Add entry type / IOR consideration | MISSING | Medium |
| 25 | Clarify customs valuation assumptions | MISSING | Low |
| 26 | Fix dashboard entry volume for company profile | MISLEADING | Low |
| 27 | Standardize HS code format | IMPRECISE | Low |

---

*This audit should be updated with verified data each time the demo date approaches. Tariff rates, MPF amounts, and 301/IEEPA program details change. Every number must be re-verified within one week of any demo.*
