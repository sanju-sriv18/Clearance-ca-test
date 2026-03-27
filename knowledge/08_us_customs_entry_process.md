# US Customs: Entry Types, Process, Bonds, Brokers, and Post-Entry

> Clearance Intelligence Engine -- Operational Knowledge Base

> Last Updated: February 2026

---

## 1. CBP ENTRY TYPES

Every import into the United States must be classified under a specific entry type code. The first digit identifies the general category; the second digit defines the specific processing type. Below is the complete catalogue.

### Consumption Entries (0x)

| Code | Name | Description & Trigger |
|------|------|----------------------|
| **01** | Consumption (Free & Dutiable) | The most common entry type. Used for all commercial shipments valued over $2,500 destined for direct consumption in the US. Covers both duty-free and dutiable goods. Requires a customs bond. Filed through ACE/ABI. |
| **02** | Consumption -- Quota/Visa | Used when imported merchandise is subject to quantitative quota restrictions or textile visa requirements. Triggers when the HTS classification falls under a quota or visa category. |
| **03** | Consumption -- AD/CVD | Used when goods are subject to antidumping duties (AD) or countervailing duties (CVD) orders. Triggered when the merchandise falls under an active AD/CVD case. Requires a case number and may require cash deposits. |
| **04** | Appraisement | Used when the value of goods cannot be determined at time of entry and requires CBP appraisement. **Not supported in ACE; must be filed on paper at the local port.** |
| **05** | Vessel Repair | Covers repairs, parts, materials, and equipment purchased for vessels in foreign ports. Subject to a 50% ad valorem duty under 19 USC 1466. **Not supported in ACE; paper filing required.** |
| **06** | Consumption -- Foreign Trade Zone (FTZ) | Used for merchandise withdrawn from a Foreign Trade Zone for consumption. Under 19 CFR 146.63(c)(1), qualified FTZ operators may file "estimated weekly entries" covering all removals during a 7-day period. Only quantities are estimated per HTS line item. Supplemental entries are required if quantities are exceeded. |
| **07** | Consumption -- AD/CVD + Quota/Visa | Combination entry used when goods are simultaneously subject to both AD/CVD orders and quota/visa requirements. |
| **08** | Duty Deferral | Used for NAFTA/USMCA duty deferral claims. The importer defers duty payment on goods that will be re-exported to another USMCA country. Filed on CBP Form 7501. |
| **09** | Reconciliation Summary | Used to reconcile previously flagged entry summaries when outstanding information (value, classification, or USMCA eligibility) becomes available. Part of CBP's Reconciliation program, which allows importers to flag entries at the time of filing and later file a single reconciliation entry to adjust multiple entries at once. |

### Informal Entries (1x)

| Code | Name | Description & Trigger |
|------|------|----------------------|
| **11** | Informal -- Dutiable | Used for commercial shipments valued at $2,500 or less, or personal shipments. Simplified documentation. No customs bond required for most shipments. Liquidates at time of entry. |
| **12** | Informal -- Quota/Visa | Informal entry for low-value goods that are also subject to quota or visa requirements. |

**Note on Type 86 (Section 321):** Entry Type 86 was a pilot program for de minimis shipments (under $800) allowing electronic filing through ACE with PGA data. As of Executive Order dated July 30, 2025, de minimis duty-free treatment under 19 USC 1321(a)(2)(C) was globally suspended effective August 29, 2025. ACE now rejects all Type 86 cargo release transactions and Section 321 manifest filings. All shipments, even those valued under $800, must now use formal (Type 01) or informal (Type 11) entries.

### Warehouse Entries (2x)

| Code | Name | Description & Trigger |
|------|------|----------------------|
| **21** | Warehouse | Used to enter goods into a CBP-bonded warehouse (19 CFR Part 144) without payment of duties. Goods may remain in a bonded warehouse up to 5 years from date of importation (19 CFR 144.6). Duty is deferred until withdrawal. Used by importers who want to store goods duty-free pending sale or re-export. |
| **22** | Re-Warehouse | Used to transfer goods from one bonded warehouse to another. The 5-year storage period runs from the original date of importation, not the re-warehouse date. |
| **23** | Temporary Importation Under Bond (TIB) | Used for goods imported temporarily for specific purposes (exhibitions, testing, repair, professional equipment, etc.) under 19 CFR 10.31-10.40 and HTSUS Chapter 98, Subchapter XIII. Requires a bond equal to double the estimated duties. Goods must be exported or destroyed within 1 year (extendable up to 3 years total). No liquidation occurs because no duties are assessed if goods are timely exported. Failure to export triggers bond forfeiture. |
| **24** | Trade Fair | For goods entered under the Trade Fair Act for display at qualified trade fairs. **Not supported in ACE.** |
| **25** | Permanent Exhibition | For goods admitted for permanent exhibition at approved venues. **Not supported in ACE.** |
| **26** | Warehouse -- FTZ Admission | Used for merchandise admitted into a Foreign Trade Zone. **Not supported in ACE; paper filing at local port required.** |

### Warehouse Withdrawal Entries (3x)

| Code | Name | Description & Trigger |
|------|------|----------------------|
| **31** | Warehouse Withdrawal -- Consumption | Used when goods are withdrawn from a bonded warehouse for entry into US commerce. Duties, taxes, and fees are assessed and paid at this point. This is the most common warehouse withdrawal type. |
| **32** | Warehouse Withdrawal -- Quota/Visa | Withdrawal for consumption of goods subject to quota or visa restrictions. |
| **33** | Aircraft & Vessel Supply | Withdrawal for supplies and equipment for aircraft and vessels. |
| **34** | Warehouse Withdrawal -- AD/CVD | Withdrawal of goods subject to antidumping or countervailing duty orders. |
| **35** | Warehouse Withdrawal -- Transportation | Withdrawal for in-bond transportation to another port. |
| **36** | Warehouse Withdrawal -- Immediate Exportation | Withdrawal for direct export from the warehouse without entering US commerce. |
| **37** | Warehouse Withdrawal -- Transportation & Exportation | Withdrawal for transportation to a port of exportation and then export. |
| **38** | Warehouse Withdrawal -- AD/CVD + Quota/Visa | Combination withdrawal for goods subject to both AD/CVD and quota/visa. |

### Drawback Entries (4x)

| Code | Name | Description & Trigger |
|------|------|----------------------|
| **41** | Manufacturing Drawback | Refund claim for duties on imported materials used in manufacturing of exported goods (19 USC 1313(a)/(b)). |
| **42** | Same Condition Drawback | Refund for goods exported in the same condition as imported (now covered under unused merchandise drawback). |
| **43** | Rejected Importation Drawback | Refund for imported goods that are rejected and returned or destroyed. |
| **47** | Drawback (General) | General drawback claim filed electronically through ACE. The primary electronic drawback entry type. |

### Government/Military Entries (5x)

| Code | Name | Description & Trigger |
|------|------|----------------------|
| **51** | DCASR/DCMAO | Defense Contract Administration Service Region entries. Military imports filed by the P99 filer code. |
| **52** | Government -- Dutiable | Government imports of dutiable merchandise. |
| **53** | Government -- Free Entry | Government imports of duty-free merchandise. |

### Transportation Entries (6x)

| Code | Name | Description & Trigger |
|------|------|----------------------|
| **61** | Immediate Transportation (IT) | In-bond movement from port of arrival to an interior port for entry. |
| **62** | Transportation & Exportation (T&E) | In-bond movement of goods through the US for export to another country. |

### ACE Support Status

Entry types **supported in ACE**: 01, 02, 03, 06, 07, 08, 09, 11, 12, 21, 22, 23, 31, 32, 34, 38, 47, 51, 52.

Entry types **NOT supported in ACE** (paper filing at local port required): 04, 05, 24, 25, 26.

---

## 2. THE CBP ENTRY PROCESS -- STEP BY STEP

The US customs entry process is governed primarily by 19 USC 1484 (Entry of Merchandise), 19 CFR Part 141 (Entry of Merchandise), and 19 CFR Part 142 (Entry Process).

### Step 1: Pre-Arrival -- Importer Security Filing (ISF / 10+2)

**Legal basis:** Safe Port Act of 2006; Trade Act of 2002; 19 CFR Part 149.

For **ocean cargo only**, the ISF Importer (or their agent, typically a customs broker) must electronically submit 10 data elements to CBP **at least 24 hours before the cargo is loaded** onto the vessel at the foreign port of origin:

1. Importer of Record number (or FTZ applicant ID)
2. Consignee number
3. Seller (owner) name and address
4. Buyer (owner) name and address
5. Manufacturer (supplier) name and address
6. Ship-to party
7. Country of origin
8. Commodity HTS number (minimum 6 digits)
9. Container stuffing location
10. Consolidator (stuffer) name and address

The ocean carrier provides the "+2" elements: vessel stow plan and container status messages.

**Penalties:** $5,000 per violation for late, inaccurate, or incomplete filing. Repeat violations: $10,000. C-TPAT Tier 2/3 members may receive up to 50% mitigation. First violations may be mitigated to $1,000-$2,000 depending on circumstances.

**Exemptions:** Bulk cargo (grain, coal, oil) is exempt. Air freight has separate advance security filing requirements (not called ISF).

### Step 2: Entry Filing -- CBP Form 3461 (Entry/Immediate Delivery)

**Legal basis:** 19 USC 1484; 19 USC 1448(b); 19 CFR 142.3, 142.16.

CBP Form 3461 is the "cargo release" document. It initiates the release of goods from CBP custody. There are two release procedures:

**Standard Entry (19 USC 1484):** The entry documents are filed, and CBP processes them to determine if the goods can be released.

**Immediate Delivery (19 USC 1448(b)):** A special permit for immediate delivery is applied for on CBP Form 3461 **prior to the arrival** of the merchandise. If approved, the shipment is released expeditiously upon arrival. Carriers in the Automated Manifest System can receive conditional release authorizations after leaving the foreign country and up to 5 days before landing in the US.

**ACE Cargo Release:** The electronic equivalent of Form 3461, requiring 12 data elements: importer of record, buyer name/address, buyer EIN (consignee number), seller name/address, manufacturer/supplier name/address, HTS 10-digit number, country of origin, bill of lading, house air waybill number, BOL issuer code, entry number, entry type, and estimated shipment value.

**Required accompanying documents:**
- Commercial invoice
- Packing list
- Bill of lading or air waybill
- Evidence of a posted customs bond
- Any required PGA documentation (permits, licenses, certificates)

**What happens after filing:** CBP evaluates the entry data, checks for holds or exam flags, and either releases the cargo or places it on hold for further examination.

### Step 3: Bond Requirements

**Legal basis:** 19 CFR Part 113; CBP Form 301.

A customs bond is a legal contract between the importer (principal), a surety company, and CBP, ensuring all duties, taxes, and fees will be paid.

**When required:** All commercial imports valued over $2,500, all ISF filings, and any shipments subject to PGA oversight regardless of value.

**Single Entry Bond (SEB):**
- Covers one specific shipment through one designated port
- Bond amount = total entered value + all duties, taxes, and fees
- For PGA-regulated or quota/visa goods: bond amount = 3x total entered value
- Minimum cost: $100 per CBP regulations
- Cannot change port after bond is issued
- Best for infrequent importers

**Continuous Bond (CB):**
- Covers all shipments at all US ports for one calendar year
- Minimum bond amount: $50,000
- Formula: nearest $10,000 to 10% of duties/taxes/fees paid in prior calendar year
- Bonds increase by $10K increments until $100K, then by $100K increments
- Also covers ISF bond requirements for ocean shipments
- Valid until terminated by principal or surety
- Only one continuous bond may be on file at a time
- CBP processes in 1-2 weeks; issues unique bond number tied to importer number
- Filed electronically with Revenue Division in Indianapolis, IN

### Step 4: Entry Summary -- CBP Form 7501

**Legal basis:** 19 USC 1484; 19 CFR Part 141; 19 CFR Part 142.

The entry summary is the detailed, comprehensive record of the import transaction. It must be filed within **10 working days** of the release of merchandise (19 CFR 142.12).

**Key data fields include:**
- Block 1: 11-digit entry number (filer code + entry number + check digit)
- Block 2: Two-digit entry type code
- Block 3: Summary date (MM/DD/YYYY)
- Block 4: Surety number from customs bond
- Block 5: Bond type
- Port code, importing carrier, country of origin
- HTS classification (10-digit)
- Entered value, duty rate, calculated duties
- Merchandise Processing Fee (MPF), Harbor Maintenance Tax (HMT)

**Filing method:** Almost exclusively electronic through ACE via ABI (Automated Broker Interface). The ACE Secure Data Portal does NOT have entry summary filing capabilities.

**Duty payment:** Estimated duties, taxes, and fees must be deposited with CBP at the time the entry summary is filed or within the 10-working-day window.

**Entry summary at time of entry:** In some cases, both steps can be combined -- CBP Form 7501 serves as both entry and entry summary, and Form 3461 is not required. This applies to certain warehouse entries (Type 21/22).

**Penalties for late filing:** CBP may impose fines and demand liquidated damages against the importer's bond.

### Step 5: CBP Review and Liquidation

**Legal basis:** 19 USC 1500 (Appraisement, Classification, Liquidation); 19 USC 1504 (Limitation on Liquidation).

**Liquidation** is CBP's final determination of the correct rate and amount of duty owed on an entry. It is the point at which CBP's assessment becomes final for most purposes.

**Timeline:**
- **~314 days after entry:** CBP's operational target for liquidating most entries. This is NOT a statutory deadline but rather the typical processing timeframe.
- **1 year from date of entry:** Statutory deadline under 19 USC 1504(a). If CBP fails to liquidate within 1 year, the entry is "deemed liquidated" at the rate and value asserted by the importer.
- **Extensions:** CBP may extend liquidation for additional 1-year periods, not to exceed **4 years total** from the date of entry (19 USC 1504(b)), if: (a) information needed for proper appraisement/classification is unavailable, or (b) the importer requests extension with good cause.
- **Suspended entries (e.g., AD/CVD):** Liquidation may be suspended by statute or court order. Once suspension is lifted, CBP has **6 months** to liquidate (19 USC 1504(d)).

**Pre-liquidation actions available to importers:**
- Post-Summary Corrections (PSC): May be filed within 300 days from entry or 15 days before scheduled liquidation (whichever is earlier). Electronic via ACE.
- Request liquidation extension under 19 USC 1504(b).
- File reconciliation entries (Type 09) for flagged entries.

**During review, CBP may issue:**
- **CBP Form 28** (Request for Information): Asks the importer for additional documentation.
- **CBP Form 29** (Notice of Action): Notifies the importer of a proposed change to classification, valuation, or duty rate.

### Step 6: Protest Rights

**Legal basis:** 19 USC 1514 (Protest Against Decisions of Customs Service); 19 CFR Part 174.

After liquidation, the importer's **only** administrative remedy is to file a protest.

**Who may protest:** The importer of record, consignee, surety, or any person paying charges or exacting a refund (19 USC 1514(c)(2)).

**Deadline:** **180 days** from the date of liquidation for entries made on or after December 18, 2004 (90 days for entries before that date). If no protest is filed within 180 days, liquidation becomes final and cannot be challenged administratively.

**Protestable decisions include:**
- Appraised value of merchandise
- Classification and rate of duty
- Amount of duties chargeable
- All charges within DHS jurisdiction
- Exclusion from entry/delivery or demands for redelivery
- Liquidation or reliquidation

**Filing method:** CBP Form 19, filed either through the ACE Protest Module or on paper. Any written document that can be construed as contesting a CBP decision and is signed by a party of interest may be treated as a protest.

**CBP review:** Generally within 2 years. Importers may request "accelerated disposition" under 19 USC 1515 -- if CBP does not decide within 30 days of the request, the protest is deemed denied.

**If denied:** The protesting party may file a civil action in the **US Court of International Trade** within **180 days** after mailing of the denial notice.

**Voluntary reliquidation:** CBP has discretionary authority to reliquidate within the first 90 days after liquidation under 19 CFR 173.4, but this is a narrow, unreliable avenue.

---

## 3. CUSTOMS BROKER ROLE

**Legal basis:** 19 USC 1641; 19 CFR Part 111 (Customs Brokers); Modernization Final Rule (87 FR 63267, October 18, 2022).

### Licensed Customs Brokers

A customs broker is a private individual or business entity licensed and regulated by CBP to assist importers and exporters in conducting customs business. The regulations are set forth in 19 CFR Part 111, issued under the authority of 19 USC 1641.

**License requirements:**
- Must pass the Customs Broker License Examination (CBLE), administered twice per year
- Must possess adequate character and reputation
- Must be a US citizen (for individual licenses) or organized under US law (for entity licenses)
- Must hold a national permit (19 CFR 111.19), which constitutes sufficient authority to conduct customs business anywhere in the US customs territory

**Broker obligations (post-2022 Modernization):**
- **Responsible supervision and control:** Every individual broker (sole proprietor), licensed partner, or licensed officer must exercise responsible supervision and control over the firm's customs business transactions
- **Due diligence (19 CFR 111.39):** Must exercise due diligence to ascertain correctness of information provided to clients, including advice on duty payment
- **Duty to advise:** Must advise clients on proper corrective actions for noncompliance, errors, or omissions, and retain records of such communications
- **Recordkeeping:** All records must be maintained within US customs territory for at least 5 years after date of entry. Powers of attorney must be retained until revoked, then 5 years after revocation or 5 years after client ceases to be "active," whichever is later

### Power of Attorney (POA)

**Legal basis:** 19 CFR Part 141, Subpart C; 19 CFR 111.36 (as modernized).

A POA is the written appointment of a broker as the true and lawful agent of the principal (importer) to transact customs business on their behalf.

**Types:**
- **Limited POA:** Covers a specified part of the principal's customs business
- **General POA:** Covers all of the principal's customs business
- **CBP Form 5291** may be used to execute a POA; if not used, a limited POA must be equally explicit

**2022 Modernization Requirements:**
- Brokers must execute a POA **directly with the importer of record** or drawback claimant -- not through a freight forwarder or other unlicensed third party
- "Execute" means having a POA directly negotiated with and signed by the client through direct communication
- A freight forwarder may NOT assign a POA to a broker on behalf of a client
- Existing POAs signed by freight forwarders must be replaced with new, directly executed POAs
- Brokers must notify clients on or attached to the POA that: "Payment to the broker will not relieve you of liability for customs charges in the event the charges are not paid by the broker"

**Penalties for broker violations:** Up to $10,000 per client for failure to collect, verify, secure, retain, or make available required information. CBP may also revoke or suspend the broker's license or permit.

### Importer of Record (IOR) Responsibilities

The IOR is the entity legally responsible for ensuring that imported goods comply with all US laws and regulations. Key responsibilities include:

- **Reasonable care standard (19 USC 1484):** The IOR must use reasonable care in classifying goods, determining value, providing accurate country of origin, and meeting all other entry requirements
- **Ultimate financial liability:** The IOR remains liable for all duties, taxes, and fees even if a broker is used and even if the broker fails to remit payment
- **Bond obligation:** The IOR is responsible for obtaining and maintaining the required customs bond
- **Record retention:** Must maintain all entry-related records for 5 years (19 CFR 163.4)
- **PGA compliance:** Must ensure goods comply with all Partner Government Agency requirements

### Self-Filing vs. Broker-Filed Entries

**Self-filing:**
- An importer transacting customs business solely on its own account is not required to be licensed (19 CFR 111.2)
- Must use ABI-certified software (ACE entry summaries can only be filed via EDI/ABI, not through the ACE Portal)
- Must complete EDI testing with CBP
- Must have a CBP-assigned Client Representative
- Foreign importers must maintain a US office with on-site personnel
- ISF portal self-filing is limited to no more than 12 filings per year

**Broker-filed:**
- The vast majority of US imports (estimated 95%+) are filed by licensed customs brokers
- Brokers have specialized expertise, dedicated ABI connectivity, backup personnel, and handle exam coordination
- Broker filing fees are a cost of doing business but reduce risk of errors, penalties, and delays

---

## 4. RELEASE TYPES AND HOLDS

### Release Types

**Standard Release:** CBP processes the entry documentation (Form 3461 or ACE Cargo Release) and, if no issues are found, authorizes release of the cargo. The carrier receives the delivery authorization.

**Immediate Delivery:** An expedited release process. The importer files Form 3461 before arrival requesting a special permit. If approved, goods are released immediately upon arrival, with the entry summary (Form 7501) to follow within 10 working days.

**Conditional Release:** Goods are released conditionally while PGA review continues. Some agencies (e.g., CPSC) allow goods to proceed to destination under conditional release pending further review.

**1USG Release:** The final "all clear" signal in ACE, meaning all agencies with authority over the shipment -- both CBP and all relevant PGAs -- have released the goods.

### CBP Examination Types

CBP has the legal right to examine any imported shipment (19 USC 1467). The importer bears all costs of examination.

**1. VACIS / NII (Non-Intrusive Inspection) Exam -- "7H Hold"**
- The container is driven through a Vehicle and Cargo Inspection System (VACIS) x-ray/gamma-ray scanner
- CBP reviews the images for anomalies
- If clear, cargo is released; if suspicious, escalated to further examination
- Cost: $150-$350 per container (varies by port and container size)
- Timeline: 2-3 days for ocean freight

**2. Tailgate Exam**
- Physical inspection at the pier/terminal
- CBP breaks the container seal and visually inspects cargo
- If compliant, released; if concerning, escalated to intensive exam
- Cost: $150-$350 per container
- Timeline: 2-3 days for ocean; hours for air freight

**3. Intensive Exam**
- Container is transported to a Customs Examination Site (CES)
- Cargo is fully unloaded, separated, parcels opened, and thoroughly inspected
- The most invasive and costly examination type
- Cost: $1,000-$2,500+ depending on labor, container size, and port
- Timeline: 5-7 days

**Targeting:** CBP uses risk-based targeting systems that score each shipment based on factors including shipper, importer, tariff number, country of origin/export, anomalous patterns, and intelligence. Scores above a threshold trigger further review or examination.

### Hold Types

**Manifest Hold:** Based on missing or incorrect data on the carrier's manifest or ISF. Resolved by providing correct documentation.

**Commercial Enforcement Hold:** Placed when CBP identifies a potential regulatory issue -- classification disputes, valuation concerns, marking violations, or intellectual property (IPR) issues.

**2H Hold:** A general HOLD status placed by CBP or another regulating authority. Cargo cannot be released until the hold is removed.

**7H Hold (NII/X-Ray Hold):** Indicates a Non-Intrusive Inspection has been ordered. If the scan reveals something suspicious, the hold may be escalated.

**PGA Hold:** Placed by a Partner Government Agency (not CBP itself). The specific agency must review and clear the shipment.

### Partner Government Agency (PGA) Holds

Nearly 50 agencies may regulate imports. The most commonly encountered:

**FDA (Food and Drug Administration):**
- Regulates food for humans and animals, pharmaceuticals, medical devices, cosmetics, tobacco, and radiation-emitting products
- Requires Prior Notice for food imports (filed through ACE)
- Hold types: FDA HOLD (documentation review), FDA EXAM (physical inspection), FDA SAMPLE (product testing)
- Risk-based assessment: importers should source from FDA-registered facilities

**USDA/APHIS (Department of Agriculture / Animal and Plant Health Inspection Service):**
- Regulates plants, animals, seeds, and wood/paper packing materials
- Enforces ISPM-15 (wood packaging material treatment standards)
- May hold shipments for fumigation if packing materials contain pests

**EPA (Environmental Protection Agency):**
- Regulates chemicals (Toxic Substances Control Act / TSCA certification required)
- Regulates vehicles, engines, and equipment with emissions standards
- Requires importers to certify compliance with TSCA or declare exemption

**CPSC (Consumer Product Safety Commission):**
- Regulates consumer products, especially children's products
- Enforces lead content limits, small parts regulations, flammability standards
- Office of Import Surveillance (EXIS) works with CBP on examination
- May allow conditional release to destination while review continues
- No permits or licenses required if goods comply with safety standards

**PGA Filing in ACE:** The PGA Message Set is filed through ACE and includes standard header data, commodity/manufacturer information, and regulatory data (permit numbers, certificates, test results). PGA "flags" -- three-character alphanumeric codes tied to HTS numbers -- indicate which agencies have jurisdiction over specific products.

### ISF (Importer Security Filing / 10+2) -- Detailed Requirements

**Applicability:** All cargo arriving by vessel to the United States. Air cargo has separate requirements (not ISF).

**Filing deadline:** At least 24 hours before cargo is loaded onto the vessel at the foreign port. CBP uses the first bill of lading file date as a proxy indicator of timeliness.

**Updates and amendments:** Certain data elements (e.g., HTS, ship-to party) may be updated after initial filing but before arrival. Changes after arrival require justification.

**Bond requirement:** An ISF bond is required. A continuous import bond covers ISF requirements. A standalone ISF bond costs approximately $50-70.

**Enforcement timeline:** Enforcement began May 2015. As of current practice, CBP ports issue liquidated damages directly, without the prior "three strikes" warning system. Liquidated damage claims must be initiated within 90 days of learning of the violation.

---

## 5. POST-ENTRY PROCESSES

### Liquidation

**Legal basis:** 19 USC 1500, 1504, 1505.

Liquidation is CBP's final computation of duties owed. It is the definitive conclusion of the entry process.

**How it works:**
1. CBP reviews the entry summary (Form 7501) data
2. CBP may accept the entry as filed, or may issue CF-28 (Request for Information) or CF-29 (Notice of Action)
3. CBP makes final determination of classification, valuation, and duty rate
4. Liquidation is posted as a bulletin notice at the port of entry and in ACE
5. Duties are either found to be correctly paid, or additional duties are assessed / refund is issued

**Deemed liquidation (19 USC 1504(a)):** If CBP fails to liquidate within 1 year (or the extended period), the entry is deemed liquidated at the importer's asserted rate -- a powerful protection for importers.

**Informal entries (Type 11):** Generally liquidate at the time of entry itself, meaning the entry is considered final almost immediately.

### Post-Summary Corrections (PSC)

**Legal basis:** 19 CFR 101.9(b) (ACE prototype); 19 USC 1484, 1485.

**What:** Electronic corrections to entry summary data filed through ACE, before liquidation.

**When:** Within 300 days from date of entry OR 15 days before scheduled liquidation date (whichever is earlier). Extensions available if liquidation extension has been granted or for suspended AD/CVD entries.

**Requirements:** Entry must be in "accepted" status in ACE, under CBP control, fully paid, not flagged for team review, and not liquidated.

**Important:** PSCs are NOT the same as prior disclosures. CBP explicitly states that PSCs should not be used as a means of submitting prior disclosures.

### Drawback

**Legal basis:** 19 USC 1313; 19 CFR Part 190 (Modernized Drawback); TFTEA (Trade Facilitation and Trade Enforcement Act of 2015).

Drawback is a refund of up to **99%** of duties, taxes, and fees paid on imported merchandise that is subsequently exported or destroyed.

**Types of drawback:**

1. **Manufacturing Drawback -- Direct Identification (19 USC 1313(a)):** Refund when specific imported materials are used in manufacturing articles that are then exported. The actual imported goods must be traced to the exported product.

2. **Manufacturing Substitution Drawback (19 USC 1313(b)):** Refund when goods of the same 8-digit HTS classification as the imported goods are used in manufacturing exported articles. The actual imported goods need not be used -- substitution is permitted. Must occur within 5 years of importation.

3. **Unused Merchandise Drawback -- Direct Identification (19 USC 1313(j)(1)):** Refund when imported goods are exported or destroyed without being used in the United States. The specific imported goods must be identified.

4. **Unused Merchandise Substitution Drawback (19 USC 1313(j)(2)):** Refund when goods of the same 8-digit HTS classification are exported in place of the imported goods. Substituted goods must be of the same 8-digit HTS (with additional 10-digit requirements if the 8-digit is described as "other"). Refund limited to 99% of the lesser of: duties paid on import or duties that would apply to the exported article if it were imported.

**TFTEA Modernization (effective December 18, 2018):**
- Replaced "commercially interchangeable" standard with same 8-digit HTS classification
- Extended substitution period from 3 years to 5 years
- Required all drawback claims to be filed electronically through ACE
- Codified in 19 CFR Part 190

**Procedural requirements:**
- CBP Form 7553 (Notice of Intent to Export, Destroy, or Return) must be filed at least 5 working days before intended exportation
- Claimant must have possession or operational control of the merchandise
- All claims filed electronically as Entry Type 47

### Prior Disclosure

**Legal basis:** 19 USC 1592(c)(4); 19 CFR 162.74.

A prior disclosure is a voluntary notification to CBP of circumstances of a customs violation before CBP discovers it and notifies the party. It is a powerful penalty mitigation tool.

**Who may file:** Importers, customs brokers, exporters, shippers, foreign suppliers, and manufacturers -- any party involved in the importation.

**Requirements for a valid disclosure (19 CFR 162.74):**
1. Identify the class or kind of merchandise involved
2. Identify the importation by entry number, drawback claim number, or port and approximate dates
3. Specify the material false statements, omissions, or acts -- including how and when they occurred
4. Set forth the true and accurate information, with a commitment to supply unknown data within 30 days

**Timing:** Must be filed BEFORE CBP or ICE/HSI discovers the violation and notifies the party. No fixed deadline, but the statute of limitations for non-fraudulent violations is 5 years.

**Penalty reduction:**
- For negligent violations: penalty is reduced to **interest on the duty underpayment** only (rather than the standard penalty amounts)
- The disclosing party must tender all lost duties, taxes, and fees either at the time of disclosure or within 30 days of CBP's calculation
- Failure to tender results in denial of the prior disclosure

**Process:**
- Often submitted in two parts: an initial "blanket" submission and a "perfected" disclosure
- Per CBP Directive 5350-020A (November 17, 2021): initial extension is 30 days, with one additional 60-day extension -- significantly shorter than the previous 180-day practice
- Oral disclosures must be followed by written submission within 10 days (19 CFR 162.74)

**Criminal implications:** While rare, prior disclosures of fraudulent violations could theoretically trigger criminal referral. Consultation with customs/criminal counsel is strongly recommended for potentially fraudulent violations.

### Penalties and Mitigation

**Legal basis:** 19 USC 1592 (Fraud, Gross Negligence, Negligence); 19 CFR Part 171 (Fines, Penalties, Forfeitures).

**19 USC 1592** penalizes any person who enters or attempts to enter merchandise by means of a material false statement, omission, or act that could affect classification, appraisement, or admissibility -- even if no revenue is actually lost.

**Penalty schedule (WITH revenue loss):**

| Culpability | Standard | Maximum Penalty |
|-------------|----------|----------------|
| **Fraud** | Voluntary and intentional | Domestic value of the merchandise |
| **Gross Negligence** | Actual knowledge or wanton disregard | Lesser of domestic value or 4x the loss of duties |
| **Negligence** | Failure to exercise reasonable care | Lesser of domestic value or 2x the loss of duties |

**Penalty schedule (WITHOUT revenue loss):**

| Culpability | Maximum Penalty |
|-------------|----------------|
| **Fraud** | Domestic value |
| **Gross Negligence** | 40% of dutiable value |
| **Negligence** | 20% of dutiable value |

**Burden of proof:**
- Fraud: Government must prove by clear and convincing evidence
- Gross Negligence: Government must prove all elements
- Negligence: Burden shifts to the importer to demonstrate reasonable care

**Exception:** Clerical errors or mistakes of fact are excepted from penalties unless part of a pattern of negligent conduct.

**Administrative process:**
1. CBP issues a **Pre-Penalty Notice** -- importer has 30 days to respond
2. If CBP proceeds, it issues a **Penalty Notice** (19 USC 1592 penalty)
3. Importer may file a **Petition for Mitigation** under 19 CFR Part 171
4. If denied, importer may file a **Supplemental Petition** under 19 CFR 171.61
5. Supplemental petition reviewed by same office, then escalated to Regulations & Rulings (HQ) for penalties over $10,000
6. After supplemental petition decision, administrative remedies are exhausted

**Mitigating factors:**
- Contributing CBP error
- Cooperation with investigation
- Immediate remedial action
- Inexperience in importing
- Prior good record
- Inability to pay
- CBP knowledge of the violation without informing the violator

**Prior disclosure mitigation:** A valid prior disclosure under 19 USC 1592(c)(4) limits penalty exposure to interest on the duty underpayment for negligent violations, and prevents seizure of merchandise.

**Duty restoration:** Regardless of whether a monetary penalty is assessed, CBP **must** require restoration of all lost duties, taxes, and fees (19 USC 1592(d)).

### Summary of Post-Entry Correction Mechanisms

| Mechanism | Timing | Legal Basis | Purpose |
|-----------|--------|-------------|---------|
| **Post-Summary Correction (PSC)** | Before liquidation (within 300 days of entry or 15 days before liquidation) | 19 CFR 101.9(b); 19 USC 1484/1485 | Correct data errors on entry summary |
| **Protest** | After liquidation (within 180 days) | 19 USC 1514; 19 CFR Part 174 | Contest CBP liquidation decisions |
| **Prior Disclosure** | Any time before CBP investigation commences | 19 USC 1592(c)(4); 19 CFR 162.74 | Disclose violations to reduce penalties |
| **Reconciliation (Type 09)** | After flagged entries, when data available | 19 CFR 181/191 | Adjust multiple flagged entries at once |
| **Last Resort -- 19 CFR 159.12** | If PSC and protest windows have closed | 19 CFR 159.12 | Request CBP extend liquidation period |

---

## KEY STATUTORY AND REGULATORY CITATIONS

| Citation | Subject |
|----------|---------|
| 19 USC 1484 | Entry of Merchandise |
| 19 USC 1448(b) | Immediate Delivery |
| 19 USC 1500 | Appraisement, Classification, and Liquidation |
| 19 USC 1504 | Limitation on Liquidation |
| 19 USC 1514 | Protest Against Decisions of Customs Service |
| 19 USC 1515 | Review of Protests |
| 19 USC 1592 | Penalties for Fraud, Gross Negligence, Negligence |
| 19 USC 1313 | Drawback and Refunds |
| 19 USC 1321 | Administrative Exemptions (De Minimis) |
| 19 USC 1466 | Equipment and Repairs of Vessels |
| 19 USC 1641 | Customs Brokers |
| 19 CFR Part 10 | Articles Conditionally Free (TIB) |
| 19 CFR Part 101 | General Provisions (PSC via ACE) |
| 19 CFR Part 111 | Customs Brokers |
| 19 CFR Part 113 | CBP Bonds |
| 19 CFR Part 141 | Entry of Merchandise |
| 19 CFR Part 142 | Entry Process |
| 19 CFR Part 144 | Warehouse and Rewarehouse Entries |
| 19 CFR Part 146 | Foreign Trade Zones |
| 19 CFR Part 149 | Importer Security Filing |
| 19 CFR Part 159 | Liquidation of Duties |
| 19 CFR Part 162 | Inspection, Search, and Seizure (Prior Disclosure) |
| 19 CFR Part 171 | Fines, Penalties, and Forfeitures |
| 19 CFR Part 173 | Administrative Review (Reliquidation) |
| 19 CFR Part 174 | Protests |
| 19 CFR Part 190 | Modernized Drawback |

---

## Sources

- [CBP: ACE Transaction Details](https://www.cbp.gov/trade/automated/ace-transaction-details)
- [eCFR: 19 CFR Part 142 -- Entry Process](https://www.ecfr.gov/current/title-19/chapter-I/part-142)
- [CBP Form 7501: Entry Summary](https://www.cbp.gov/trade/programs-administration/entry-summary/cbp-form-7501)
- [CBP Form 3461: Entry/Immediate Delivery](https://www.cbp.gov/document/forms/form-3461-entryimmediate-delivery-ace)
- [CBP: Importer Security Filing 10+2](https://www.cbp.gov/border-security/ports-entry/cargo-security/importer-security-filing-102)
- [CBP: Protests](https://www.cbp.gov/trade/programs-administration/entry-summary/protests)
- [CBP: Post Summary Corrections](https://www.cbp.gov/trade/programs-administration/entry-summary/post-summary-correction)
- [CBP: Drawback Overview](https://www.cbp.gov/trade/programs-administration/entry-summary/drawback-overview)
- [CBP: Temporary Importation Under Bond](https://www.cbp.gov/trade/programs-administration/entry-summary-and-post-release-processes/temporary-importation-under-bond)
- [CBP: Section 321 Programs](https://www.cbp.gov/trade/trade-enforcement/tftea/section-321-programs)
- [CBP: About Foreign-Trade Zones](https://www.cbp.gov/border-security/ports-entry/cargo-security/cargo-control/foreign-trade-zones/about)
- [CBP: Reconciliation](https://www.cbp.gov/trade/programs-administration/entry-summary/reconciliation)
- [CBP: Customs Broker Modernization](https://www.cbp.gov/trade/programs-administration/customs-brokers/modernization)
- [CBP: How to Use ACE](https://www.cbp.gov/trade/automated/how-to-use-ace)
- [eCFR: 19 CFR Part 111 -- Customs Brokers](https://www.ecfr.gov/current/title-19/chapter-I/part-111)
- [eCFR: 19 CFR Part 113 -- CBP Bonds](https://www.ecfr.gov/current/title-19/chapter-I/part-113)
- [eCFR: 19 CFR Part 141 Subpart C -- Powers of Attorney](https://www.ecfr.gov/current/title-19/chapter-I/part-141/subpart-C)
- [eCFR: 19 CFR Part 144 -- Warehouse and Rewarehouse](https://www.ecfr.gov/current/title-19/chapter-I/part-144)
- [eCFR: 19 CFR Part 146 -- Foreign Trade Zones](https://www.ecfr.gov/current/title-19/chapter-I/part-146)
- [eCFR: 19 CFR Part 174 -- Protests](https://www.ecfr.gov/current/title-19/chapter-I/part-174)
- [eCFR: 19 CFR Part 190 -- Modernized Drawback](https://www.ecfr.gov/current/title-19/chapter-I/part-190)
- [eCFR: 19 CFR 162.74 -- Prior Disclosure](https://www.ecfr.gov/current/title-19/chapter-I/part-162/subpart-G/section-162.74)
- [19 USC 1504 -- Limitation on Liquidation](https://www.law.cornell.edu/uscode/text/19/1504)
- [19 USC 1514 -- Protest Against Decisions](https://www.law.cornell.edu/uscode/text/19/1514)
- [19 USC 1592 -- Penalties for Fraud, Gross Negligence, Negligence](https://www.law.cornell.edu/uscode/text/19/1592)
- [CBP: Prior Disclosure Informed Compliance Publication](https://www.cbp.gov/sites/default/files/assets/documents/2020-Feb/Prior%20Disclosure%20FINAL.pdf)
- [CBP: Mitigation Guidelines for 19 USC 1592](https://www.cbp.gov/document/publications/mitigation-guidelines-icp-fraud-gross-negligence-negligence-1592)
- [Federal Register: Modernization of the Customs Broker Regulations (87 FR 63267)](https://www.federalregister.gov/documents/2022/10/18/2022-22445/modernization-of-the-customs-broker-regulations)
- [Federal Register: Modernized Drawback Final Rule](https://www.federalregister.gov/documents/2018/12/18/2018-26793/modernized-drawback)
- [CBP: ISF Enforcement and Penalties](https://www.shapiro.com/isf-enforcement-penalties-filing-errors-and-mitigation/)
- [CBP: Suspension of Duty-Free De Minimis Treatment](https://www.cbp.gov/sites/default/files/2025-08/factsheet_suspension_of_duty-free_de_minimis_treatment.pdf)
- [Customs Bonds FAQ -- IIA](https://www.iiacorp.org/cbp-bond-faqs)
- [help.CBP.gov: Bond Types](https://www.help.cbp.gov/s/article/Article1074?language=en_US)
- [PSC vs Protest vs Prior Disclosure -- Barakat + Bossa](https://b2b.legal/post-summary-corrections-protests-prior-disclosures-oh-my/)
- [Great Lakes Customs Law: Prior Disclosure](https://greatlakescustomslaw.com/prior-disclosure-to-cbp-of-19-usc-1592-violations/)
- [Great Lakes Customs Law: 19 USC 1592 Penalties](https://greatlakescustomslaw.com/customs-violations/fraud-negligence-penalties-19-usc-1592/)