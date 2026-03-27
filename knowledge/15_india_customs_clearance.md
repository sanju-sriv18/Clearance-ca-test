# India Customs Clearance: Import, Export, and Regulatory Framework

**Clearance Intelligence Engine -- Operational Knowledge Base**  
**Last Updated: February 2026**

---

# INDIA: Comprehensive Customs Clearance Intelligence

## 1. India Customs Authority -- Central Board of Indirect Taxes and Customs (CBIC)

### Organization Structure

The **Central Board of Indirect Taxes and Customs (CBIC)** operates under the **Ministry of Finance, Department of Revenue**, Government of India. It is the apex body responsible for the administration of customs, central excise, and GST laws. CBIC formulates policy concerning levy and collection of customs duties, prevention of smuggling, and enforcement of the Customs Act, 1962.

**Governing Legislation:** The **Customs Act, 1962** (Act No. 52 of 1962) is the principal legislation governing the import and export of goods into and from India. It provides the legal framework for:
- Levy and collection of customs duties
- Import/export prohibitions and restrictions
- Customs procedures for clearance of goods
- Powers of customs officers
- Offences and penalties
- Appeals and revisions
- Warehousing provisions (Sections 57-73)
- Post-clearance audit (Section 99A, introduced via Finance Act 2018)

### ICEGATE (Indian Customs Electronic Gateway)

**ICEGATE** is the national e-commerce portal of CBIC, serving as the interface between trade participants (importers, exporters, customs brokers, shipping lines) and the Indian Customs EDI System (ICES). Key capabilities:

- **Electronic filing** of Bills of Entry (import) and Shipping Bills (export)
- **E-payment** of customs duties (mandatory from January 1, 2025, for all duty types including pre-deposits, amendments, penalties, and interest)
- **Document upload** and tracking of clearance status
- **Integration** with regulatory agencies: DGFT, RBI, DGCI&S, Ministry of Steel, FSSAI, CDSCO, Wildlife Crime Control Bureau (WCCB)
- **Electronic Cash Ledger** for duty payments
- **ICETAB** (tablet-based examination) for digital customs inspections, expanded to export consignments from June 2025

ICEGATE is operational at **252 major customs locations**, handling approximately **98% of India's international trade** by value.

### Indian Customs Single Window -- SWIFT

The **Single Window Interface for Facilitating Trade (SWIFT)** initiative integrates clearance permissions from multiple regulatory agencies (called Participating Government Agencies or PGAs) into a single electronic platform. Under **SWIFT 2.0** (current version):

- **FSSAI** NOC for food imports: integrated across 84 points of entry
- **CDSCO** (Drug Controller) unified application: live
- **WCCB** (Wildlife/CITES) unified application: live from January 22, 2026
- **Plant Quarantine** import permits: issued through SWIFT
- Eliminates the need for separate paper-based NOC applications to each agency

### Customs Integrated System (CIS) -- Future Platform

The government issued an Expression of Interest (EoI) in December 2025 to build a **Customs Integrated System (CIS)** that will unify ICEGATE, RMS, and ICES into a single AI-enabled platform. Target: reduce cargo clearance time to **24 hours**. Assessment, refunds, and goods testing are planned to be made completely faceless, with fully digitized dispute resolution.

---

## 2. Import Process

### 2.1 IEC (Importer Exporter Code) -- Mandatory Registration

The **Importer Exporter Code (IEC)** is a **10-digit unique identification number** issued by the **Directorate General of Foreign Trade (DGFT)**. It is mandatory for all import and export activities in India, per Section 7 of the Foreign Trade (Development and Regulation) Act, 1992.

- **Requirement:** No export or import shall be made by any person without an IEC, unless specifically exempted
- **Issuing Authority:** DGFT (online via dgft.gov.in)
- **Format:** Same as PAN (Permanent Account Number) of the firm since GST implementation
- **Prerequisites:** PAN, bank account, valid Indian address
- **Application Fee:** INR 500
- **Processing:** Auto-generated upon submission of complete application with documents
- **Validity:** Lifetime, but mandatory annual profile update on DGFT portal (deactivated if not updated)
- **No return filing** requirement
- **Exemptions:**
  - Trade with Nepal/Myanmar (Indo-Myanmar border) / China (except Nathula) where single consignment CIF value does not exceed INR 25,000
  - Personal imports/exports not for resale
  - Government ministries and departments
  - Notified charitable organizations
  - Service exports (unless claiming FTP benefits)

Both **IEC and GST registration (GSTIN)** are mandatory to clear customs. Without both, goods will be held at port, incurring demurrage charges.

### 2.2 Bill of Entry (Pravesha Patra / Bill of Entry) -- Types

The Bill of Entry is the principal customs declaration document filed under **Section 46 of the Customs Act, 1962**. Four types exist:

#### (a) Bill of Entry for Home Consumption (White Form)
- Used when goods are intended for **direct use/consumption** within India
- All applicable duties (BCD, SWS, IGST, cess) must be paid **before clearance**
- Importer can claim **Input Tax Credit (ITC)** for IGST paid
- The standard and most common type

#### (b) Bill of Entry for Warehousing / "Into Bond" (Yellow/Buff Form)
- Filed under **Sections 46 and 60** of the Customs Act
- Used when the importer does not wish to pay duty immediately upon arrival
- Goods are stored in a **Customs Bonded Warehouse** (licensed under Sections 57/58/58A)
- Duty payment is deferred until goods are removed from the warehouse
- Benefit: **cash flow management** -- importer pays duty only when goods are actually needed
- Requires execution of a bond (and potentially a bank guarantee)

#### (c) Ex-Bond Bill of Entry (Green Form)
- Filed under **Section 68** of the Customs Act
- Used to clear goods from a bonded warehouse for home consumption
- Duty is calculated at the **rate in force on the date of actual removal** from the warehouse (not the date of original import)
- Filed as and when the importer requires goods from the warehouse

#### (d) Bill of Entry for Defence (Pink Form)
- Used exclusively for clearing imported goods for **defence establishments**

### 2.3 Prior Bill of Entry (Advance Filing)

Under **Section 46** of the Customs Act, an importer may file a Bill of Entry **prior to arrival** of goods in India:
- The vessel/aircraft must arrive within **30 days** from the date of presentation
- Five copies filed; the fifth copy is the "Advance Noting" copy
- Importer declares that the vessel is due within 30 days
- Must present the Bill of Entry for final noting once the **Import General Manifest (IGM)** is filed
- **Not available** for Into Bond Bill of Entry
- Significantly reduces clearance time for compliant importers

### 2.4 ICES (Indian Customs EDI System) Workflow

The end-to-end clearance workflow operates as follows:

1. **IGM Filing:** Shipping line/airline files the Import General Manifest electronically via ICEGATE before vessel arrival
2. **Bill of Entry Filing:** Importer/Customs Broker (formerly CHA) files BOE electronically on ICEGATE with GSTIN, IEC, and all shipment details
3. **RMS Processing:** The filed BOE is automatically processed by the Risk Management System
4. **Risk Assessment:** RMS assigns the consignment to a channel (Green or Red)
5. **Assessment:** For Red channel, customs officers verify classification, valuation, and applicable duty; for Green channel, this is bypassed or minimally checked
6. **Duty Payment:** Importer pays calculated duties via e-payment on ICEGATE
7. **Examination:** Physical examination (if required by RMS) at the Container Freight Station (CFS) or port
8. **Out of Charge (OOC) Order:** Once assessment, duty payment, and examination (if any) are complete, the customs officer grants the "Out of Charge" order electronically in ICES, releasing the goods

**Typical clearance time:** 1-3 working days for compliant shipments with correct documentation.

### 2.5 Risk Management System (RMS) -- Automated Targeting

The RMS is an IT-driven automated risk assessment system that:
- Uses **predefined algorithms and risk parameters** to profile every consignment
- Draws on historical compliance data, intelligence inputs, and trade patterns
- Assigns consignments to channels:

**Green Channel (Facilitated Release):**
- Low-risk consignment
- No physical examination required
- Only document verification (often automated)
- Goods proceed directly to Out of Charge after duty payment

**Red Channel (Assessment and Examination):**
- High-risk consignment flagged for scrutiny
- Requires **document verification by customs officer**
- Requires **physical examination** of goods
- May involve sampling, laboratory testing, or specialist review
- Assessment of classification, valuation, and applicable exemptions

**Green Channel Facility (for Major Importers):**
- Select importers with strong compliance history are granted permanent green channel status
- Clearance without routine examination
- Only marks and numbers are checked
- Exception: physical examination may still be ordered by senior officers or investigation wings (e.g., SIIB -- Special Intelligence and Investigation Branch) if specific doubts arise

### 2.6 Out of Charge Order (Mal Nikasi / OOC)

The **Out of Charge** order is the final customs release authorization:
- Granted electronically within ICES
- Signifies that all customs formalities are complete
- Triggers release of goods from the port/CFS/ICD for domestic movement
- After OOC, the goods are free for the importer to transport to their premises

### 2.7 Required Documents for Import Clearance

| Document | Purpose |
|----------|---------|
| **Bill of Entry** (filed on ICEGATE) | Customs declaration |
| **Commercial Invoice** | Value declaration for duty assessment |
| **Packing List** | Detailed contents, weights, dimensions |
| **Bill of Lading (B/L)** or **Airway Bill (AWB)** | Transport document / title to goods |
| **Certificate of Origin (CoO)** | For claiming preferential duty under FTAs/PTAs |
| **Import License** (where applicable) | DGFT authorization for restricted items |
| **Insurance Certificate** | For CIF value determination |
| **Letter of Credit / Bank Realization Certificate** | Payment proof |
| **Technical write-up / catalog** | For classification of specialized goods |
| **Test/Analysis reports** | Where required by PGAs (BIS, FSSAI, CDSCO) |
| **GSTIN** | For IGST and ITC purposes |
| **IEC** | Mandatory identification |

---

## 3. India's Tariff and Tax Structure

India operates a **multi-layered duty structure** on imports. The total landed duty is a cascading calculation.

### 3.1 Basic Customs Duty (BCD)

- **Legal basis:** First Schedule to the **Customs Tariff Act, 1975**
- **Nature:** Predominantly **ad valorem** (percentage of assessable value); some items have specific or compound rates
- **Assessable Value:** Generally based on **CIF value** (Cost, Insurance, and Freight) under the Customs Valuation (Determination of Value of Imported Goods) Rules, 2007
- **Rate range:** 0% to 150% (varies widely by product; most industrial goods 5%-20%)
- **Annual changes:** Rates are amended through the **Finance Act** (Union Budget) and periodic CBIC notifications
- **2025-26 Budget reform:** Customs tariff slabs reduced to only **8 rates** (including zero), down from 15+
- BCD is **not recoverable** as Input Tax Credit

### 3.2 Social Welfare Surcharge (SWS)

- Introduced in Budget 2018 (replacing Education Cess)
- **Rate: 10% of the BCD amount** (not of the assessable value)
- Funds government social welfare schemes
- **Calculation:** SWS = BCD Amount x 10%
- Not recoverable as ITC
- **2025-26 reform:** SWS exempted on **82 tariff lines** that are subject to another cess (one cess/surcharge per item rule)

### 3.3 Integrated Goods and Services Tax (IGST)

- Replaced the former **CVD** (Countervailing Duty / Additional Customs Duty) and **Special CVD (SAD)**
- Imports are treated as **inter-state supply** under GST law
- **Applicable rates:**
  - **0%:** Essential items (unprocessed food grains, life-saving drugs)
  - **5%:** Essential goods (basic food items, common medicines, economy transport)
  - **12%:** Processed food, business-class air tickets, some industrial inputs
  - **18%:** Standard rate (majority of goods and services)
  - **28%:** Luxury and demerit goods (automobiles, aerated beverages, tobacco)
- **Calculation base:** Assessable Value + BCD + SWS (i.e., IGST is levied on the cumulative value)
- **Key feature:** IGST paid at import is **recoverable** by registered businesses through **Input Tax Credit (ITC)** mechanism under GST, making it a pass-through cost for businesses (unlike BCD and SWS)
- IGST credits can be utilized against output CGST, SGST, or IGST liability

### 3.4 GST Compensation Cess

- Levied under **Section 8 of the GST (Compensation to States) Act, 2017**
- Applied on **luxury and demerit/sin goods**: tobacco, aerated beverages, certain motor vehicles, coal
- Originally intended to compensate states for revenue loss from GST transition (for 5 years from 2017)
- **GST 2.0 reform (September 2025):** CBIC Notification No. 02/2025 removed compensation cess across **19 major product categories**
- New structure under GST 2.0: 5% (merit), 18% (standard), and a new **40% special rate** for luxury/harmful products replaces the cess mechanism
- For software: check whether cess has been removed for specific HS codes or if transitional provisions apply

### 3.5 Agriculture Infrastructure and Development Cess (AIDC)

- Introduced via **Clause 115 of the Finance Bill, 2021**
- Imposed on **select imported products** specified in the First Schedule to the Customs Tariff Act
- Purpose: Fund agricultural infrastructure development across India
- **Products covered (29 items):** Gold (2.5%), silver (2.5%), apples (35%), crude palm oil (17.5%), various pulses, wines, alcohol (excluding beer), urea, petrol/diesel
- **Budget 2025-26 additions:** Marble/travertine/granite (20%), candles/PVC flex/solar cells (7.5%), and others
- **Revenue impact:** AIDC collections do not go into the divisible pool shared with states
- **Offsetting:** Where AIDC is imposed, BCD is typically reduced correspondingly so net consumer impact is minimized
- AIDC is calculated on the assessable value

### 3.6 Anti-Dumping Duty (ADD)

- Imposed under **Section 9A of the Customs Tariff Act, 1975**
- Recommended by **DGTR (Directorate General of Trade Remedies)**, enforced by CBIC
- Applied when goods are imported at prices **below fair market value**, causing injury to domestic industry
- **Duration:** Typically **5 years**, extendable after sunset review
- Country-specific and product-specific (e.g., Chinese steel, plastics machinery at 27-63%)
- India has been one of the most active users of anti-dumping measures globally, especially against Chinese imports

### 3.7 Safeguard Duty

- Imposed under **Section 8B of the Customs Tariff Act, 1975** (WTO Safeguards Agreement compliant)
- Triggered by **unexpected surge in import volumes** causing injury to domestic producers
- **Investigation:** Conducted by DGTR based on volume (not price)
- **Duration:** Up to **200 days** provisional; extendable to **4 years** (plus 1 sunset review)
- Recent example: 12% provisional safeguard duty on Hot Rolled Coil (HRC) and Cold Rolled Coil (CRC) from China, South Korea, Japan (February 2025)

### 3.8 Customs Handling Fee

- **1% of assessable value** assessed on all imports as customs handling charge

### 3.9 Duty Calculation Formula (Step-by-Step)

```
Step 1: Assessable Value (AV) = CIF Value (converted to INR at applicable exchange rate)
Step 2: BCD = AV x BCD Rate (%)
Step 3: SWS = BCD x 10%
Step 4: IGST Base = AV + BCD + SWS
Step 5: IGST = IGST Base x IGST Rate (%)
Step 6: Compensation Cess (if applicable) = (AV + BCD + SWS) x Cess Rate (%)
Step 7: AIDC (if applicable) = AV x AIDC Rate (%)
Step 8: ADD / Safeguard Duty (if applicable) = per specific notification
Step 9: Total Duty = BCD + SWS + IGST + Compensation Cess + AIDC + ADD/Safeguard

Note: Anti-dumping duty and safeguard duty are calculated per the specific notification
and are NOT included in the IGST base in most cases.
```

### 3.10 ITC (Input Tax Credit) Mechanism for IGST at Import

- IGST paid at the time of import is credited to the importer's **Electronic Credit Ledger** on the GST portal
- The importer reports the import in **GSTR-2B** (auto-populated from customs data)
- The credit appears in **GSTR-3B** and can be utilized against:
  - Output IGST liability (on inter-state sales)
  - Output CGST liability (on intra-state sales)
  - Output SGST/UTGST liability (on intra-state sales)
- **Matching:** The Bill of Entry number and GSTIN must match between customs (ICES) and GST portal (GSTN)
- This makes IGST effectively a **pass-through** for registered businesses, distinguishing it from the non-recoverable BCD and SWS

---

## 4. India's Tariff Classification

### 4.1 ITC-HS (Indian Trade Classification based on Harmonized System)

India follows the **ITC-HS** classification system, which is India's adaptation of the WCO Harmonized System:

**Structure:**
- **First 2 digits:** HS Chapter (e.g., 61 = Articles of apparel, knitted)
- **First 4 digits:** HS Heading (e.g., 6105 = Men's shirts, knitted)
- **First 6 digits:** HS Sub-heading (e.g., 610510 = Of cotton) -- internationally harmonized up to this level
- **7th-8th digits:** National Tariff Item (e.g., 61051010 = specific Indian statistical code)
- **9th-10th digits:** DGFT's ITC-HS extends to **10 digits** for import/export policy classification

**Key distinction:**
- **8-digit codes** are used for **customs duty** purposes under the Customs Tariff Act
- **10-digit codes** are used by **DGFT** for trade policy (import/export restrictions, licensing)

### 4.2 Customs Tariff Act, 1975

- **First Schedule:** Import duty rates against each 8-digit tariff item
- **Second Schedule:** Export duty rates (applied to select items only)
- **98 Chapters** organized into **21 Sections**
- Rates are amended annually via the **Finance Act/Union Budget** and through CBIC notifications throughout the year
- **General Rules for Interpretation (GRI):** Six rules for systematic classification
- **Section Notes, Chapter Notes, Sub-heading Notes, Supplementary Notes:** Legally binding classification guidance
- **WCO Explanatory Notes:** Not legally binding in India but treated as authoritative guidance

### 4.3 DGFT ITC-HS Classification

The **Directorate General of Foreign Trade (DGFT)** publishes the ITC-HS Classification of Import and Export Goods:
- **Schedule I:** Import policies (Free / Restricted / Prohibited)
- **Schedule II:** Export policies
- Determines whether an item requires an import license, is freely importable, or is prohibited
- Updated through DGFT notifications and public notices

### 4.4 Software Implications for Classification

- Duty rates are assigned at the **8-digit tariff item level**, not at the heading or sub-heading level
- The same HS heading may have vastly different duty rates for different tariff items
- Rates change frequently through notifications (not just annual budgets)
- A robust classification engine must maintain the full hierarchy: Section > Chapter > Heading > Sub-heading > Tariff Item, with all applicable notes

---

## 5. India's Trade Agreements

### 5.1 Active Agreements (Providing Preferential Tariff Rates)

| Agreement | Type | Signed/Effective | Key Details |
|-----------|------|-----------------|-------------|
| **India-ASEAN FTA (AIFTA / AITIGA)** | FTA (Goods) | 2009/2010 | Covers 90% of tariff lines; under active review (AITIGA Review) as of 2025 |
| **India-Japan CEPA** | CEPA | 2011 | Japan eliminated tariffs on >90% of Indian exports; covers goods, services, investment |
| **India-Korea CEPA** | CEPA | 2009/2010 | Covers goods, services, investment, IP, competition, customs cooperation |
| **India-Singapore CECA** | CECA | 2005 | Broad pact covering goods, services (banking, telecom), investment |
| **SAFTA** (South Asian Free Trade Area) | FTA | 2004/2006 | Members: Afghanistan, Bangladesh, Bhutan, India, Maldives, Nepal, Pakistan, Sri Lanka; India has 25-item sensitive list for LDCs, 695 for non-LDCs |
| **India-UAE CEPA** | CEPA | Feb 2022 | UAE eliminated duties on 97% of tariff lines (99% of Indian imports by value); benefits ~$26B of Indian products; negotiated in record 88 days |
| **India-Australia ECTA** | ECTA | Dec 2022 | High utilization: 79% export quota, 84% import quota; broader CECA negotiations ongoing |
| **India-EFTA TEPA** | TEPA | Mar 2024 | India's first FTA with Iceland, Liechtenstein, Norway, Switzerland; includes $100B investment commitment |
| **India-UK FTA** | FTA | Jul 2025 | Signed after 3+ years of negotiation; phased tariff reductions; projected to boost bilateral trade by $34B annually |
| **India-Oman CEPA** | CEPA | Dec 2025 | Signed December 18, 2025 |
| **India-New Zealand FTA** | FTA | Dec 2025 | Negotiations concluded December 2025; aims to double bilateral trade in 5 years |
| **India-Mauritius CECPA** | CECPA | 2021 | Limited scope; first trade agreement with an African country |
| **India-Thailand EHS** | Early Harvest | 2004 | Limited product coverage |
| **India-Chile PTA** | PTA | 2007 | Limited product coverage |
| **India-MERCOSUR PTA** | PTA | 2009 | Limited product coverage |
| **Asia-Pacific Trade Agreement (APTA / Bangkok Agreement)** | PTA | 1975/2006 | Members include China, South Korea, Bangladesh, Laos, Sri Lanka |

### 5.2 RCEP -- India is NOT a Member

**India formally withdrew from RCEP negotiations in November 2019** at the Bangkok ASEAN Summit, despite participating in 29 rounds of negotiations over 6+ years since 2013. RCEP was signed by the remaining 15 members on November 15, 2020, forming the world's largest FTA.

**Key reasons for withdrawal:**
- Fear of flooding by Chinese imports (India already had massive trade deficit with China without an FTA)
- Protection of domestic manufacturing (particularly dairy, textiles, electronics) aligned with Make in India / PLI schemes
- RCEP members accounted for ~70% of India's trade deficit
- Insufficient gains in services trade (India's strength area)
- Domestic political pressure from industry groups (notably dairy/Amul)
- Border tensions with China (Ladakh clashes in 2020)
- Disagreements on base year for tariff reduction (India wanted 2019, not 2014) and automatic safeguard mechanisms

**Status:** RCEP agreement leaves room for India to join at a later date, but no active steps toward accession have been taken.

### 5.3 Under Negotiation

| Agreement | Status |
|-----------|--------|
| **India-EU FTA** | 11th round concluded (May 2025); two-stage approach adopted; target to conclude by end of 2025 |
| **India-Australia CECA** | Advancing (broader than existing ECTA); covers digital trade, labor, gender, environment |
| **India-Canada CEPA/EPTA** | Negotiations paused/slow due to diplomatic tensions |
| **AITIGA Review (India-ASEAN)** | Active discussions to modernize the 2009 agreement |

### 5.4 Certificate of Origin (CoO) for Preferential Duty

- To claim preferential tariff rates under any FTA/CEPA, the importer must present a valid **Certificate of Origin** issued by the exporting country's designated authority
- India's DGFT issues CoOs for Indian exports
- **Utilization trend:** 132,116 CoOs issued in April-May 2025 (up from 120,598 in same period 2024), indicating growing FTA utilization
- Recent FTAs (UAE, Australia) show significantly higher utilization rates (79-84%) compared to older FTAs (ASEAN: 5-25%, Japan: 10-20%)

---

## 6. Foreign Trade Policy (Videsh Vyapar Niti)

### 6.1 DGFT -- Administering Authority

The **Directorate General of Foreign Trade (DGFT)**, under the **Ministry of Commerce and Industry**, administers India's foreign trade policy. Functions include:
- Issuing IEC codes
- Granting import/export licenses and authorizations
- Administering duty exemption and remission schemes
- Publishing ITC-HS classification schedules
- Interpreting and enforcing trade policy regulations

### 6.2 Foreign Trade Policy 2023 (FTP 2023)

The current **Foreign Trade Policy 2023**, effective from **April 1, 2023**, is notable for:
- **No expiry date** -- provides policy continuity (unlike previous 5-year FTPs)
- **Vision:** Increase India's exports to **USD 2 trillion by 2030**
- **Four pillars:**
  1. Legal certainty and institutional clarity
  2. Trade facilitation and ease of doing business (e-IEC, e-BRC, 24/7 helpdesk)
  3. Export promotion through targeted schemes
  4. Decentralization and inclusivity (Districts as Export Hubs, MSME support)

### 6.3 Advance Authorization (AA) Scheme

- Allows **duty-free import of inputs** physically incorporated in export products
- Exemption from **BCD, IGST, and Compensation Cess** on imported inputs
- Also covers fuel, oil, catalyst, and packaging materials consumed in production
- Available for both physical exports and deemed exports
- **Export Obligation (EO):** Must export finished products within prescribed period
- Includes **Advance Authorization for Annual Requirement** for regular exporters
- Processed electronically on the DGFT e-platform; applications processed in **1-2 days**

### 6.4 DFIA (Duty Free Import Authorization)

- Allows duty-free import of inputs and oil/catalyst consumed in export product manufacturing
- Exempts **only BCD** (unlike AA which also exempts IGST)
- Issued only for products with notified **Standard Input Output Norms (SION)**
- **Validity:** 12 months from date of issue
- **Transferable** after export obligation is fulfilled (unlike AA)
- Generally follows the same provisions as Advance Authorization

### 6.5 EPCG (Export Promotion Capital Goods) Scheme

- Allows import of **capital goods at zero customs duty** (including zero IGST and Compensation Cess) for manufacturing export goods
- **Export Obligation:** Typically a multiple of the duty saved, to be fulfilled within the prescribed timeframe
- Covers pre-production, production, and post-production capital goods
- **FTP 2023 changes:**
  - Post Export EPCG scheme removed
  - Dairy sector exempted from annual average maintenance requirement
  - Green technology products: export obligation reduced to 75%
  - Exports to SEZ units count toward EPCG export obligation
- Non-fulfillment attracts duty recovery with interest and penalties

### 6.6 SEZ (Special Economic Zones) Regime

Governed by the **SEZ Act, 2005** and SEZ Rules:
- **Duty-free imports** of goods for authorized operations
- Exemption from BCD, IGST (zero-rated under GST), and other levies
- Single-window clearance for central and state government matters
- Income tax benefits (for units set up before April 2020)
- Simplified customs procedures
- Deemed foreign territory for customs purposes
- SEZ units can sell in the Domestic Tariff Area (DTA) upon payment of applicable duties

### 6.7 Status Holder Benefits

- Exporters are recognized as **Status Holders** based on cumulative export performance
- Categories: One Star Export House, Two Star, Three Star, Four Star, Five Star
- **FTP 2023:** Lowered minimum export threshold, enabling smaller exporters to qualify
- Benefits include: self-certification of origin, faster customs processing, priority treatment

### 6.8 Other Key Schemes

- **RoDTEP (Remission of Duties and Taxes on Exported Products):** Refund of embedded duties/taxes not otherwise refunded
- **RoSCTL (Rebate of State and Central Taxes and Levies):** For textiles sector
- **Amnesty Scheme (FTP 2023):** One-time settlement for defaults in export obligation under AA/EPCG, allowing reduced duty + interest payment

---

## 7. Regulatory Requirements (Participating Government Agencies)

### 7.1 BIS (Bureau of Indian Standards)

- **Governing law:** Bureau of Indian Standards Act, 2016
- **Compulsory Registration Scheme (CRS):** Mandatory for electronics (mobile phones, laptops, adapters, LED lights, power banks, etc.), steel products, chemicals, cement, LPG cylinders, and other specified categories
- **ISI Marking:** Required for select products under mandatory certification
- **Process:** Foreign manufacturers must register with BIS, undergo factory inspection, and obtain BIS license before importing into India
- **Electronics:** Virtually all consumer electronics require CRS registration
- **Steel:** Quality Control Orders (QCOs) mandate BIS certification for most steel products
- **Chemicals:** Specified chemicals require BIS certification
- **Recent change:** FSSAI removed mandatory BIS/ISI certification for certain food items (April 2023), aligning with "One Nation, One Commodity, One Regulator" policy

### 7.2 FSSAI (Food Safety and Standards Authority of India)

- **Governing law:** Food Safety and Standards Act, 2006
- **Mandatory for:** All food product imports
- **Requirements:**
  - **FSSAI Import License** (separate from regular FSSAI license)
  - IEC mandatory to apply for FSSAI Import License
  - Compliance with FSSAI standards (Food Safety and Standards Regulations)
  - Labeling requirements (in English + Hindi for retail sale)
- **Clearance system:** Food Import Clearance System (FICS), integrated with ICEGATE via SWIFT
- **Risk-based sampling:** Selective sampling and testing based on FSSAI risk profiling at ICEGATE
- **Coverage:** Authorized Officers at 62 points of entry; Customs officials notified as Authorized Officers at additional 99 points
- **Products:** Processed foods, nutraceuticals, additives, fresh produce all require FSSAI compliance

### 7.3 DCGI / CDSCO (Drug Controller General of India / Central Drugs Standard Control Organization)

- **Governing law:** Drugs and Cosmetics Act, 1940 and Rules, 1945
- **Mandatory for:** Pharmaceuticals, bulk drugs (APIs), medical devices, cosmetics
- **Requirements:**
  - Registration of drugs and medical devices with CDSCO
  - Import license from DCGI
  - Each batch may require test/analysis certificate
  - Cosmetics require registration and label compliance
- **NOC:** Integrated with SWIFT 2.0 for electronic clearance

### 7.4 Plant Quarantine (Padap Sangarodh)

- **Governing law:** Destructive Insects & Pests Act, 1914; Plant Quarantine (Regulation of Import into India) Order, 2003
- **Administering body:** Directorate of Plant Protection, Quarantine & Storage (DPPQS) under Ministry of Agriculture
- **Requirements:**
  - **Import Permit** (applied through SWIFT) -- generally valid for 6 months
  - **Pest Risk Analysis (PRA):** Mandatory, submitted to the Plant Protection Adviser
  - **Phytosanitary Certificate** from the exporting country
  - Physical inspection at port of entry by plant quarantine officers
- **Applicable to:** All agricultural commodities, seeds, plants, plant products, soil, plant-based packaging material (ISPM-15 compliance for wood packaging)

### 7.5 Animal Quarantine

- **Governing law:** Livestock Importation Act, 1898; relevant DAHD (Department of Animal Husbandry and Dairying) notifications
- **Administering body:** Animal Quarantine and Certification Services (AQCS)
- **Requirements:**
  - Sanitary import permit
  - Health certificate from country of origin
  - Quarantine period upon arrival (varies by species)
- **Applicable to:** Live animals, animal products, meat, dairy inputs, animal genetic material

### 7.6 CITES / Wildlife Permits

- **International framework:** Convention on International Trade in Endangered Species (CITES)
- **Domestic law:** Wildlife (Protection) Act, 1972
- **Administering body:** Ministry of Environment, Forest and Climate Change (MoEF&CC); Wildlife Crime Control Bureau (WCCB)
- **Requirements:**
  - CITES permits for listed species (Appendices I, II, III)
  - Wildlife clearance certificate
  - NOC from WCCB (now integrated with SWIFT 2.0 from January 22, 2026)
- **Exceptionally restrictive:** India maintains strict controls on wildlife trade

---

## 8. India-Specific Challenges for Clearance Intelligence Software

### 8.1 Complex Duty Notification System

India's duty rates change with high frequency through multiple mechanisms:
- **Annual Union Budget** (Finance Act amendments to Customs Tariff Act schedules)
- **CBIC notifications** issued throughout the year (exemption, effective rate, anti-dumping, safeguard)
- **Major 2025 consolidation:** 31 separate customs notifications merged into single **Notification No. 45/2025-Customs** (effective November 1, 2025), but new notifications continue to be issued
- **Conditional exemptions:** Many duty exemptions have complex conditions (end-use certificates, quantity limits, time limits, specific ports)
- **Software requirement:** Must maintain a notification-aware duty engine that tracks not just tariff schedule rates but also the latest applicable notification rates and their conditions

### 8.2 Port-Specific Processing Differences

- Despite centralized ICES, different customs commissionerates (ports) may have:
  - Different interpretation of classification for borderline products
  - Different valuation practices
  - Different examination rates and intensity
  - Different PGA officer availability (especially smaller ports)
  - Different infrastructure (CFS quality, lab testing availability)
- Major ports: JNPT/Nhava Sheva (Mumbai), Chennai, Mundra, Kolkata/Haldia, Cochin, Visakhapatnam, Tughlakabad (Delhi ICD)

### 8.3 IGST Refund Mechanism

- For **exporters** who pay IGST at import on inputs used in export goods:
  - Can claim refund of accumulated ITC under **Rule 89** of CGST Rules (inverted duty structure or export without payment of tax)
  - Or can export with IGST payment and claim automatic refund under **Rule 96**
- **Automatic refund processing:** System matches Shipping Bill data (ICES) against GSTR-3B data (GSTN) at invoice level
- **Common issues:** Mismatches in invoice data between customs and GST returns cause refund blockages
- **Rule 96(10) complications:** Exporters who imported inputs under exemption notifications but then exported with IGST face refund challenges (retrospective amendments now regularize this)
- **Software requirement:** Track IGST credits separately from BCD/SWS; model refund eligibility based on export/domestic sale patterns

### 8.4 Bond and Bank Guarantee Requirements

- **Warehousing bonds:** Required for Into Bond Bill of Entry (Section 59 of Customs Act)
- **Export bonds / LUT (Letter of Undertaking):**
  - LUT preferred (no bank guarantee needed; valid for exporters not prosecuted for tax evasion > INR 2.5 crore)
  - Bond required for those ineligible for LUT, backed by bank guarantee (typically **15% of bond value**)
- **EPCG/AA bonds:** Exporters using duty exemption schemes must execute bonds for export obligation fulfillment
- **Provisional assessment bonds:** Where customs assessment is provisional pending finalization
- **Software requirement:** Model bond exposure and bank guarantee requirements based on import/export patterns and scheme utilization

### 8.5 Customs Audit (Post-Clearance Audit)

- **Legal basis:** Section 99A of the Customs Act, 1962 (introduced via Finance Act 2018)
- **Customs Audit Regulations, 2018** operationalize the framework
- **Two types:**
  - **Transaction Based Audit (TBA):** Review of specific Bills of Entry selected by RMS
  - **Premises Based Audit (PBA):** Comprehensive review of all import/export activity over a period, at the importer's premises
- **Auditee scope:** Importers, exporters, customs warehouse licensees, custodians (approved under Section 45), freight forwarders, and any person directly or indirectly involved in clearing/forwarding/stocking/selling imported/exported goods
- **Audit Commissionerates:** Dedicated units in Chennai, Delhi, Mumbai Zone-I, and Nhava Sheva with all-India jurisdiction
- **Powers:** Audit officers empowered under Section 17 (assessment) and Section 28 (demand and recovery) to issue show cause notices
- **Monitoring:** Chief Commissioners review 5% of audit reports quarterly
- **Software requirement:** Maintain audit trail of all classification, valuation, and duty calculation decisions for potential post-clearance scrutiny

### 8.6 Exchange Rate for Duty Calculation

- CBIC notifies **exchange rates** for conversion of foreign currency to INR for customs duty purposes
- Rates are updated periodically (typically fortnightly) via CBIC notifications
- The applicable rate is the rate in force on the **date of filing** the Bill of Entry (or date of entry inward of vessel, whichever is later for prior-filed BOEs)
- Software must track CBIC-notified exchange rates, not market rates

### 8.7 Valuation Disputes

- India follows **WTO Customs Valuation Agreement** (Transaction Value method as primary)
- Six sequential valuation methods under the Customs Valuation Rules, 2007
- Customs may reject declared transaction value and apply alternative methods
- **SVB (Special Valuation Branch):** Dedicated unit for investigating related-party transactions
- **NIDB (National Import Database):** Customs uses historical import data for value comparison
- Frequent disputes on: royalty/license fee inclusion, transfer pricing adjustments, buying commissions, and assists

---

## Terminology Reference (Hindi/Legal Terms)

| English Term | Hindi/Legal Term |
|-------------|-----------------|
| Customs | Sima Shulk (Customs Duty: Sima Shulk) |
| Bill of Entry | Pravesha Patra |
| Out of Charge | Mal Nikasi |
| Foreign Trade Policy | Videsh Vyapar Niti |
| Importer Exporter Code | Aayat Niryat Sankhya |
| Customs Tariff | Sima Shulk Darsuchak |
| Assessable Value | Nirdharan Yogya Mulya |
| Plant Quarantine | Padap Sangarodh |
| Bonded Warehouse | Bandhua Godam |
| Certificate of Origin | Utpatti Praman Patra |
| Show Cause Notice | Karan Batao Suchna |
| Duty Drawback | Shulk Vapasi |
| Anti-Dumping Duty | Dumping Virodhi Shulk |
| Safeguard Duty | Suraksha Shulk |

---

**Sources used in this research:**

- [CBIC Official Portal](https://www.cbic.gov.in/entities/customs)
- [ICEGATE Portal](https://www.icegate.gov.in)
- [DGFT Official Website](https://www.dgft.gov.in/)
- [India-Briefing: Customs Duty and Import-Export Taxes](https://www.india-briefing.com/doing-business-guide/india/taxation-and-accounting/customs-duty-and-import-export-taxes-in-india)
- [India-Briefing: India Revises Customs Tariff Under Union Budget 2025-26](https://www.india-briefing.com/news/india-revises-customs-tariff-under-union-budget-2025-26-36054.html/)
- [India-Briefing: India's Free Trade Agreements Updates 2025](https://www.india-briefing.com/news/indias-free-trade-agreements-updates-2025-36271.html/)
- [India-Briefing: India's International Free Trade Agreements](https://www.india-briefing.com/doing-business-guide/india/why-india/india-s-international-free-trade-and-tax-agreements)
- [PIB: Union Budget 2025-26 Customs Tariff Rates](https://www.pib.gov.in/PressReleasePage.aspx?PRID=2098364)
- [PIB: CBIC Trade Facilitative Measures](https://www.pib.gov.in/PressReleasePage.aspx?PRID=2124318)
- [PIB: Exports Powered by Trade Agreements](https://www.pib.gov.in/PressReleasePage.aspx?PRID=2206194)
- [Taxmann: Customs Tariff Act and HSN Classification](https://www.taxmann.com/post/blog/customs-tariff-act-hsn-classification)
- [India Code: Customs Tariff Act, 1975](https://www.indiacode.nic.in/handle/123456789/8774)
- [India Code: Customs Act, 1962](https://www.indiacode.nic.in/bitstream/123456789/15359/1/the_customs_act,_1962.pdf)
- [Vizag Customs: Import Clearance Procedure](https://www.vizagcustoms.gov.in/Home/ImportProcedure)
- [TaxGuru: Risk Management System in Imports](https://taxguru.in/custom-duty/introduction-risk-management-system-rms-imports.html)
- [TaxGuru: Customs Audit Under Customs Act 1962](https://taxguru.in/custom-duty/audit-under-customs-act-1962.html)
- [TaxScan: CBIC-FSSAI Digital Integration](https://www.taxscan.in/top-stories/cbic-and-fssai-integrate-digital-clearance-systems-via-swift-and-icegate-for-food-imports-at-84-entry-points-1427356)
- [TaxScan: AI-Enabled Customs Integrated System](https://www.taxscan.in/top-stories/budget-2026-govt-to-roll-out-ai-enabled-customs-integrated-system-in-2-years-read-finance-bill-2026-1442607)
- [Business Standard: Unified Customs Portal EoI](https://www.business-standard.com/india-news/govt-issues-eoi-unified-customs-portal-125122600826_1.html)
- [India-Briefing: India Scraps Compensation Cess Under GST 2.0](https://www.india-briefing.com/news/gst-cess-removal-india-2025-business-benefits-39879.html/)
- [FinTaxBlog: Consolidation of 31 Customs Duty Notifications](https://fintaxblog.com/consolidation-of-31-customs-duty-notifications-by-cbic-and-faqs/)
- [FSSAI Official Import Portal](https://www.fssai.gov.in/cms/imports.php)
- [PPQS: Plant Quarantine Import-Export Procedure](https://ppqs.gov.in/divisions/plant-quarantine/import-export-procedure)
- [ClearTax: IEC Import Export Code](https://cleartax.in/s/import-export-code)
- [ClearTax: GST Export Bond and LUT](https://cleartax.in/s/gst-export-bond-and-lut)
- [iKargos: Anatomy of Import Costs BCD IGST SWS](https://app.ikargos.com/blogs/the-anatomy-of-import-costs-how-duties-taxes-are-calculated-bcd-igst-sws-2319)
- [TarifDutyCalculator: India Import Duty Rates 2025](https://tariffdutycalculator.com/india-import-duty-rates/)
- [Wikipedia: SAFTA](https://en.wikipedia.org/wiki/South_Asian_Free_Trade_Area)
- [Wikipedia: Free Trade Agreements of India](https://en.wikipedia.org/wiki/Free_trade_agreements_of_India)
- [GlobalAsia: Why India Opted Out of RCEP](https://www.globalasia.org/v14no4/feature/a-step-too-far-why-india-opted-out-of-rcep_rajaram-panda)
- [NextIAS: Understanding India's Withdrawal from RCEP](https://www.nextias.com/ca/editorial-analysis/19-01-2024/understanding-indias-withdrawal-from-rcep)
- [Stratrich: Foreign Trade Policy Overview](https://stratrich.com/insights/an-overview-of-indias-foreign-trade-policy/)
- [DGFT FTP 2023 Chapter 4: Duty Exemption/Remission Schemes](https://content.dgft.gov.in/Website/dgftprod/a11b458e-0d61-460f-aef4-b201ed40a9aa/FTP2023_Chapter04.pdf)
- [DGFT FTP 2023 Chapter 5: EPCG Scheme](https://content.dgft.gov.in/Website/dgftprod/97e20e4c-eae3-4548-8ab5-bb805cd377a7/HBP2023_Chapter05.pdf)
- [Customs Audit Regulations 2018](https://upload.indiacode.nic.in/showfile?actid=AC_CEN_2_2_00042_196252_1534829466423&type=regulation&filename=45+of+2018+Customs_Audit_Regulations_2018.pdf)
- [ICMAI: Customs Audit and Ease of Doing Business](https://icmai.in/TaxationPortal/upload/IDT/Article_GST/238.pdf)