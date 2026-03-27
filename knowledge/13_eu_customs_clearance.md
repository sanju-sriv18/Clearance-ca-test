# EU Customs Clearance: Import, Export, and Regulatory Framework

**Clearance Intelligence Engine -- Operational Knowledge Base**  
**Last Updated: February 2026**

---


## 1. EU CUSTOMS FRAMEWORK

### 1.1 Union Customs Code (UCC) -- Regulation (EU) No 952/2013

The **Union Customs Code (UCC)**, established by **Regulation (EU) No 952/2013** of the European Parliament and of the Council of 9 October 2013, is the foundational legislation governing all customs operations in the European Union. It sets out the general rules and procedures applicable to goods brought into or taken out of the customs territory of the EU.

**Key legal instruments forming the UCC framework:**

| Instrument | Reference | Purpose |
|---|---|---|
| **Union Customs Code** | Regulation (EU) No 952/2013 | Main customs law -- general rules and procedures |
| **Delegated Act (DA)** | Commission Delegated Regulation (EU) 2015/2446 | Supplements UCC with detailed rules on customs aspects |
| **Implementing Act (IA)** | Commission Implementing Regulation (EU) 2015/2447 | Ensures uniform conditions for UCC implementation across all Member States |
| **Transitional Delegated Act (TDA)** | Commission Delegated Regulation (EU) 2016/341 | Transitional rules until IT systems are fully operational |
| **Electronic Systems** | Commission Implementing Regulation (EU) 2023/1070 | Technical arrangements for electronic customs systems |

**Entry into force and application:**
- Entered into force: 30 October 2013
- Substantive provisions applicable from: 1 May 2016
- Transitional period for IT systems: through 31 December 2025
- Replaced: Regulation (EC) No 450/2008 (Modernised Customs Code)

**Core objectives (Article 3 UCC):**
Customs authorities are primarily responsible for supervising the Union's international trade, contributing to fair and open trade, implementing the external aspects of the internal market, the common trade policy, and other common Union policies having a bearing on trade, as well as ensuring overall supply chain security.

**IT systems status:** Of the 17 IT systems mandated by the Code, eight were deployed by 2020, four more in 2021, and the remaining five are being deployed through end of 2025.

### 1.2 Role of National Customs Administrations vs. EU-Level Coordination

The EU customs system operates on a dual-level structure:

**EU-Level (European Commission, DG TAXUD):**
- Sets uniform customs legislation (UCC, Delegated Act, Implementing Act)
- Manages TARIC database and the Common Customs Tariff
- Coordinates IT system development and deployment
- Manages trade defence investigations (anti-dumping, countervailing duties)
- Oversees AEO mutual recognition agreements
- Coordinates risk management through the Common Risk Management Framework

**National Customs Administrations (27 Member States):**
- Implement and enforce EU customs legislation at the national level
- Operate national customs clearance IT systems (e.g., ATLAS in Germany, DELTA in France, AGS/DMS in Netherlands)
- Process customs declarations and collect customs duties, import VAT, and excise duties
- Conduct physical inspections and risk-based controls
- Grant authorizations (AEO, simplified procedures, special procedures)
- Apply national VAT rates to imports
- Manage national aspects of excise duty collection

### 1.3 TARIC (Integrated Tariff of the European Communities)

**Legal basis:** Council Regulation (EEC) No 2658/87 of 23 July 1987

TARIC is the EU's integrated tariff database, consolidating all commercial, agricultural, and customs legislation affecting tariff measures. It ensures uniform application across all 27 Member States.

**TARIC 10-digit code structure:**

| Digits | Level | Description |
|---|---|---|
| 1--6 | HS code | International Harmonized System (WCO) |
| 7--8 | CN subheadings | EU Combined Nomenclature -- product differentiation |
| 9--10 | TARIC subheadings | EU-specific measures (anti-dumping, suspensions, quotas) |
| 11 | National | National use only (e.g., VAT rates, national restrictions) |

TARIC is published and managed by the European Commission (DG TAXUD). Daily electronic transmissions update national systems to ensure uniform, correct tariff application across the EU. The database integrates customs duty rates, trade preferences, anti-dumping duties, tariff quotas, suspensions, and all regulatory measures.

### 1.4 Combined Nomenclature (CN)

**Legal basis:** Council Regulation (EEC) No 2658/87, Annex I

The Combined Nomenclature is the EU's 8-digit tariff and statistical nomenclature. It serves dual purposes: (1) the Common Customs Tariff duty rates and (2) EU external trade statistics.

**Structure:**
- Digits 1--6: Harmonized System (HS) headings and subheadings (international, WCO-maintained)
- Digits 7--8: CN subheadings (EU-specific subdivisions)

**Annual updates:** The Commission adopts an Implementing Regulation each year reissuing the complete CN with autonomous and conventional duty rates. Published no later than 31 October in the Official Journal, effective 1 January of the following year.

### 1.5 EORI (Economic Operators Registration and Identification)

**Legal basis:** Articles 9 and 22 of the UCC; Articles 6--8 of the Implementing Act

The EORI number is a unique identifier assigned by the customs authority of an EU Member State to economic operators and other persons involved in customs activities.

**Key characteristics:**
- **Mandatory** for all customs operations (import, export, transit) in the EU customs territory
- **One EORI per entity** -- a person can hold only one valid EORI number at any time
- **EU-wide validity** -- valid across all 27 Member States once registered in any one Member State
- **No expiration** -- remains valid indefinitely
- **Free of charge** -- registration is done online at no cost
- **Format:** Country code (2 letters) + national identifier (up to 15 characters), e.g., DE1234567890123

**Who needs one:**
- Any economic operator established in the EU customs territory
- Non-EU operators carrying out customs activities in the EU (registered in the first EU Member State where they conduct a customs operation)
- Customs agents and brokers

**Validation:** The European Commission provides an EORI validation service to verify whether a number is active and properly registered.

---

## 2. IMPORT DECLARATION PROCESS

### 2.1 Standard Customs Declaration -- H1 (Release for Free Circulation)

The EU Customs Data Model (EUCDM) defines several declaration types using the H-series:

| Code | Purpose |
|---|---|
| **H1** | Release for free circulation and end-use |
| **H2** | Customs warehousing |
| **H3** | Temporary admission |
| **H4** | Inward processing |
| **H5** | Introduction of goods in trade with special fiscal territories |
| **H6** | Postal traffic -- release for free circulation |
| **H7** | Low-value consignments -- release for free circulation (below EUR 150) |

The H1 declaration is the standard full-dataset import declaration for placing goods into free circulation within the EU. It contains all data elements required for customs clearance, fiscal assessment, trade statistics, and regulatory compliance.

### 2.2 Simplified Declaration Procedures (Article 166 UCC)

**Authorization requirement:** Granted to AEO-C holders or operators meeting equivalent AEO criteria.

Under a Simplified Declaration (SD), the economic operator may lodge a customs declaration with a reduced data set, omitting certain details or documents at the time of declaration. A **supplementary declaration** must follow within a stipulated period. The relevant EUCDM column is 7a.

If a simplified declaration is used only on an occasional (non-regular) basis, no authorization is required.

### 2.3 Entry in the Declarant's Records (EIDR) -- Article 182 UCC

EIDR is the most advanced form of customs simplification. An authorized economic operator releases goods to a customs procedure by making an entry in their own electronic commercial records, without submitting a standard customs declaration at the time of release.

**Key features:**
- Goods are released via an entry in the declarant's electronic records
- Customs authorities must have concurrent access to the relevant particulars
- A supplementary declaration containing full fiscal and statistical data is lodged subsequently
- EIDR can be used for: export, re-export, temporary admission, inward processing, outward processing, customs warehousing, and end-use
- An authorization from customs is required, with a control plan specifying frequency of checks
- EUCDM column: 7c

### 2.4 Centralised Clearance (CC) -- Article 179 UCC

**Implementing provisions:** Article 149 DA, Articles 229--232 IA

Centralised Clearance for Import (CCI) allows an economic operator to lodge customs declarations at a single customs office in their Member State of establishment (Supervising Customs Office, SCI), even when goods are physically presented at a customs office in another Member State (Presentation Customs Office, PCI).

**Requirements:**
- Must hold valid Authorised Economic Operator for Customs Simplifications (AEOC) status
- Application submitted at least 120 days before intended use via the Customs Decision System (CDS)
- EUCDM column: 7b

**Benefits:** Centralizes accounting, logistics, and distribution functions; reduces interaction with multiple customs offices; eliminates need for multiple brokers across Member States.

### 2.5 Single Authorisation for Simplified Procedures (SASP)

SASP enables the use of EIDR or Simplified Declaration to perform customs formalities in the Member State where the operator is established, regardless of where goods are physically located at import or export. The Supervising Member State is where the operator is established and where all records and accounts are accessible. A periodic supplementary declaration is lodged.

### 2.6 National Import Systems

While the UCC mandates interoperable electronic systems, each Member State operates its own national customs clearance platform:

| Country | System | Full Name |
|---|---|---|
| **Germany** | ATLAS | Automatisiertes Tarif- und Lokales Zollabwicklungssystem (Automated Tariff and Local Customs Clearance System) |
| **France** | DELTA IE | Dédouanement En Ligne par Traitement Automatisé (Import/Export) |
| **Netherlands** | AGS / DMS | Automated Goods System / Declaration Management System |
| **Belgium** | PLDA | Paperless Douane en Accijnzen |
| **Italy** | AIDA | Automazione Integrata Dogane e Accise |
| **Spain** | AEAT | Agencia Estatal de Administracion Tributaria |
| **Sweden** | TDS/TESS | Tullverkets Deklarationssystem |
| **UK (post-Brexit)** | CDS | Customs Declaration Service |

Germany's ATLAS Import handles release for free circulation under normal or simplified procedures, customs warehousing, and inward processing. Electronic export declarations have been mandatory via ATLAS since 1 July 2009. ATLAS IMPOST (since 15 January 2022) handles low-value consignments up to EUR 150.

France's DELTA IE is a complete overhaul of the previous DELTA-G system, fully digitizing customs clearance, ending paper-based declarations, and enabling advance declarations. It is being rolled out progressively.

---

## 3. EU TARIFF STRUCTURE

### 3.1 Common Customs Tariff (CCT)

The CCT is the uniform external tariff applied to all goods imported into the EU from third countries, regardless of which Member State they enter. Every product is classified in the tariff nomenclature using 10-digit TARIC codes, which determine applicable duty rates and trade measures.

**Legal basis:** Article 28 TFEU (Treaty on the Functioning of the European Union); Council Regulation (EEC) No 2658/87

**Key principle:** Duties are calculated on the **CIF value** (Cost + Insurance + Freight to the EU border), as declared in the customs value.

### 3.2 MFN Rates (Most-Favoured-Nation / WTO Bound Rates)

MFN tariffs apply to imports from third countries with which the EU does not have a preferential trading arrangement. These rates are negotiated and bound in the WTO framework.

**Key statistics:**
- EU simple average applied MFN tariff: approximately **5.6%**
- Nearly half of all imports enter at MFN zero tariffs
- The EU may apply tariffs lower than its bound rates (applied tariffs)
- MFN rates cannot be raised unilaterally except under WTO-authorized safeguard measures

### 3.3 Preferential Rates under FTAs and GSP

The EU offers lower-than-MFN tariffs for approximately **59.8%** of all HS06 products from its preferential trade partners.

**Types of preferential arrangements:**
- **Free Trade Agreements (FTAs):** Reciprocal zero or reduced tariffs (e.g., EU-Canada CETA, EU-Japan EPA)
- **Customs Unions:** Zero tariffs on essentially all products (e.g., EU-Turkey Customs Union for industrial goods)
- **GSP:** Unilateral preferences for developing countries (see Section 4)
- **Association Agreements:** Broader political agreements with trade components

### 3.4 Autonomous Tariff Suspensions

**Legal basis:** Article 31 TFEU; Council Regulation (EU) No 2021/2278

Autonomous tariff suspensions provide total or partial waiver of normal customs duties on imported raw materials, semi-finished goods, or components not produced or insufficiently produced in the EU.

**Key rules:**
- Only raw materials, semi-finished goods, or components qualify -- no finished products
- Minimum annual duty savings threshold: EUR 20,000 (SMEs may group together)
- Applications submitted through national authorities to the European Commission
- Reviewed by the Economic Tariff Questions Group (ETQG)
- Updated semi-annually by Council Regulation
- Objections may be filed by EU producers of equivalent goods
- Unlimited quantity = suspension; limited quantity = autonomous tariff quota

### 3.5 Tariff Rate Quotas (TRQs)

TRQs permit a predetermined quantity of imports at a lower in-quota duty rate. Once exhausted, the normal MFN rate applies.

**Two types:**
1. **Preferential TRQs:** Under FTAs and preferential arrangements -- reduced/zero duties for goods originating in a specified country, up to the quota volume
2. **Autonomous TRQs:** Erga omnes (open to all countries) -- typically for raw materials, semi-finished goods, or components insufficiently available in the EU

**Management:** Most TRQs are managed on a **first-come, first-served** basis by DG TAXUD. Certain agricultural TRQs are managed by DG AGRI.

### 3.6 Anti-Dumping Duties and Countervailing Duties

**Legal basis:** Regulation (EU) 2016/1036 (Anti-Dumping Basic Regulation); Regulation (EU) 2016/1037 (Anti-Subsidy Basic Regulation)

**Anti-dumping duties** counter non-EU manufacturers selling goods in the EU below normal value on their domestic market, where this causes material injury to EU producers. The duty rate covers the margin between the fair value and the export price.

**Countervailing duties (CVDs)** offset the effects of unfair subsidies from a trading partner's government. CVDs can take the form of:
- Additional ad valorem duty
- Specific duty (per unit)
- Minimum import price
- Price undertaking (exporter agrees to sell above a minimum price)

Both measures are authorized under WTO rules (GATT Article VI, Anti-Dumping Agreement, SCM Agreement). Investigations and measures are administered by the European Commission (DG TRADE).

### 3.7 Import VAT

Import VAT is charged on almost all goods imported into the EU. The rate depends on the **Member State of importation**. VAT is calculated on: goods value + shipping + insurance + import duty.

**EU Standard VAT Rates by Member State (as of 2025):**

| Member State | Standard Rate | Member State | Standard Rate |
|---|---|---|---|
| Austria | 20% | Latvia | 21% |
| Belgium | 21% | Lithuania | 21% |
| Bulgaria | 20% | Luxembourg | 17% (lowest) |
| Croatia | 25% | Malta | 18% |
| Cyprus | 19% | Netherlands | 21% |
| Czech Republic | 21% | Poland | 23% |
| Denmark | 25% | Portugal | 23% |
| Estonia | 24% | Romania | 21% |
| Finland | 25.5% | Slovakia | 23% |
| France | 20% | Slovenia | 22% |
| Germany | 19% | Spain | 21% |
| Greece | 24% | Sweden | 25% |
| Hungary | 27% (highest) | | |
| Ireland | 23% | | |
| Italy | 22% | | |

**EU average:** 21.9%. **EU minimum requirement:** 15%.

**Note:** Some Member States have regional regimes (e.g., Canary Islands in Spain, Azores/Madeira in Portugal). Denmark applies only a single standard rate with no general reduced bands.

### 3.8 Excise Duties

**Legal basis:** Council Directive 2020/262 (Excise Directive -- general arrangements)

EU-harmonized excise duties apply to three product categories: alcohol and alcoholic beverages, tobacco products, and energy products (including electricity).

**Alcohol -- Minimum rates:**
| Product | EU Minimum Rate |
|---|---|
| Beer | EUR 0.748/degree Plato/hl or EUR 1.87/degree/hl |
| Wine (still/sparkling) | EUR 0/hl |
| Intermediate products (port, sherry) | EUR 45/hl |
| Spirits | EUR 550/hl of pure alcohol |

Member States set rates above these minimums, resulting in significant national variation.

**Tobacco:**
- Cigarettes: A combined excise consisting of a specific component (EUR per 1,000 cigarettes) and an ad valorem component (% of retail price). Member States applying EUR 115+ per 1,000 cigarettes are exempt from the 60% minimum tax burden criterion.
- Combined taxes average over 80% of retail price across the EU (2024).
- Excise burden ranges from EUR 2.02/pack (Bulgaria) to EUR 10.71/pack (Ireland).

**Energy products:** Covered by the Energy Taxation Directive (Council Directive 2003/96/EC) with minimum rates for motor fuels, heating fuels, and electricity.

**Cross-border principle:** Excise duties are owed in the Member State of consumption (destination principle).

---

## 4. EU FTAs AND PREFERENTIAL TRADE

### 4.1 EU-UK Trade and Cooperation Agreement (TCA)

- **Signed:** 24 December 2020; **In force:** 1 May 2021
- **Tariff provisions:** Zero tariffs, zero quotas on all goods meeting rules of origin
- **Rules of origin:** Product-specific rules; origin declarations on commercial documents by exporters
- **Scope:** Beyond trade in goods -- covers services, digital trade, energy, fisheries, law enforcement cooperation, and more

### 4.2 EU-Japan Economic Partnership Agreement (EPA)

- **In force:** 1 February 2019
- **Scope:** One of the largest bilateral free trade zones, covering roughly 635 million people and about one-third of global GDP
- **Tariff elimination:** The EU eliminates customs duties on approximately 97% of tariff lines; Japan eliminates approximately 99%
- **Key sectors:** Automotive (phased elimination over 7 years), agricultural products (cheese, wine, pork), industrial goods

### 4.3 EU-Canada CETA (Comprehensive Economic and Trade Agreement)

- **Signed:** 30 October 2016; **Provisionally applied:** 21 September 2017
- **Status:** Trade-related provisions in force; investment chapter awaiting ratification by all Member State parliaments
- **Tariff elimination:** 98% of tariff lines liberalized
- **Key features:** First "new generation" EU trade agreement; covers investment, government procurement, intellectual property, regulatory cooperation

### 4.4 EU-South Korea FTA

- **In force:** 1 July 2011 (provisionally); fully in force 13 December 2015
- **Tariff provisions:** Virtually all customs duties eliminated
- **Notable feature:** Includes duty drawback provisions -- refund on duties previously paid on non-originating materials used in products exported under preferential tariff
- **Recent addition:** EU-South Korea Digital Trade Agreement (DTA) concluded March 2025

### 4.5 EU-Vietnam FTA (EVFTA)

- **In force:** 1 August 2020
- **Tariff elimination:** Nearly 99% of customs duties eliminated between the parties
- **Timeline:** Vietnam eliminates 48.5% of tariff lines immediately; 91.8% after 7 years; 98.3% after 10 years. EU eliminates 71% immediately; remaining phased out over 7 years.
- **Key sectors:** Electronics, food products, pharmaceuticals, textiles, footwear
- **Investment protection:** Separate EU-Vietnam Investment Protection Agreement (EVIPA) still awaiting ratification by all 27 Member State parliaments

### 4.6 EU-Singapore FTA (EUSFTA)

- **In force:** 21 November 2019
- **Tariff elimination:** 84% of Singapore products enter EU tariff-free from day one; remaining 16% eliminated over 3--5 years. Singapore binds tariffs at zero for all EU goods.
- **ASEAN cumulation:** Allows Singapore manufacturers to include components sourced from other ASEAN Member States as originating content
- **Digital trade:** EU-Singapore Digital Trade Agreement (EUSDTA) signed 7 May 2025

### 4.7 EU-Mercosur Partnership Agreement

- **Political agreement reached:** 6 December 2024
- **Council authorization:** 9 January 2026 (qualified majority: 21 in favor, 5 against, 1 abstention)
- **Signed:** 17 January 2026 in Paraguay
- **Status:** European Parliament referred the agreement to the European Court of Justice in January 2026 to rule on whether provisional application is permissible -- this may delay implementation by up to two years
- **Scope:** Covers approximately 780 million people; EU and Mercosur will gradually reduce duties on 91--92% of exports over 15 years
- **Key provisions:** Eliminates high Mercosur tariffs on EU agri-food (wine 35%, chocolate 20%, olive oil 10%); binding commitments on Paris Agreement compliance and deforestation

### 4.8 EU GSP, GSP+, and EBA

**Legal basis:** Regulation (EU) No 978/2012 (extended through 2027 by amendment); new revised regulation to apply from 1 January 2027

**Three tiers:**

| Tier | Description | Tariff Benefit |
|---|---|---|
| **Standard GSP** | Lower/lower-middle income countries without other preferential access | Duty reductions on 66% of tariff lines |
| **GSP+** | Vulnerable developing countries implementing 27 international conventions on human rights, labor, governance, environment | Zero duties on 66% of tariff lines |
| **EBA (Everything But Arms)** | Least Developed Countries (LDCs) as classified by the UN | Duty-free, quota-free access for all products except arms and ammunition |

**Current GSP+ beneficiaries (8 countries):** Bolivia, Cape Verde, Kyrgyzstan, Mongolia, Pakistan, Philippines, Sri Lanka, Uzbekistan

**EBA graduation:** Several countries are graduating from LDC status (Bhutan, Angola, Solomon Islands, Sao Tome and Principe already graduated; Laos, Nepal, Bangladesh expected to graduate by 2026 with transition periods).

**Conditionality:** All GSP beneficiaries must respect 15 core human rights and labor conventions. Preferences may be temporarily withdrawn for serious and systematic violations.

### 4.9 Pan-Euro-Mediterranean Convention (PEM Convention)

**Signed on behalf of EU:** April 2011

The PEM Convention harmonizes rules of origin across approximately 60 bilateral FTAs between countries in the pan-Euro-Mediterranean zone (EU, EFTA states, Turkey, North Africa, Middle East, Western Balkans).

**Contracting parties include:** EU, EFTA states (Iceland, Liechtenstein, Norway, Switzerland), Barcelona Declaration signatories (Algeria, Egypt, Israel, Jordan, Lebanon, Morocco, Palestinian Authority, Syria, Tunisia, Turkey), SAP countries (Western Balkans), and others (Georgia, Moldova, Montenegro, UK, Ukraine).

**Revised PEM Convention (effective 1 January 2025 in parallel; sole applicable rules from 1 January 2026):**
- Higher tolerance threshold for non-originating materials: raised from 10% to 15%
- Simplified product-specific rules
- Full origin cumulation for most products (except textiles)
- Possibility of duty drawback
- Removal of EUR-MED certificate; introduction of electronic certification
- Extended proof of origin validity from 4 to 10 months

### 4.10 Rules of Origin: Proofs of Origin

| Proof Type | Description | Threshold |
|---|---|---|
| **EUR.1 Movement Certificate** | Issued by customs authorities of the exporting country | No value limit |
| **Invoice/Origin Declaration** | Made by an approved exporter on commercial documents | By approved exporter: no limit; by any exporter: up to EUR 6,000 |
| **EUR-MED Certificate** | Used under original PEM Convention for diagonal cumulation (being phased out under revised PEM) | No value limit |
| **REX System** | Registered Exporter system -- electronic self-certification for GSP, certain FTAs | Registered exporters only |

---

## 5. CUSTOMS VALUATION IN THE EU

### 5.1 Transaction Value (Primary Method) -- Article 70 UCC

The primary basis for customs value is the **transaction value** -- the price actually paid or payable for the goods when sold for export to the EU customs territory. This follows the WTO Customs Valuation Agreement (Agreement on Implementation of Article VII of GATT 1994).

The customs authority hierarchy of valuation methods (Articles 70--74 UCC):
1. **Transaction value** (Article 70) -- primary and mandatory method
2. **Transaction value of identical goods** (Article 74(2)(a))
3. **Transaction value of similar goods** (Article 74(2)(b))
4. **Deductive method** (Article 74(2)(c))
5. **Computed method** (Article 74(2)(d))
6. **Fall-back method** (Article 74(3))

The importer does not have freedom to choose -- the methods must be applied in sequential order.

### 5.2 CIF Basis

EU customs value is determined on a **CIF (Cost, Insurance, Freight)** basis -- i.e., the value at the point of entry into the EU customs territory. This contrasts with the US approach which uses FOB (Free on Board).

### 5.3 Additions and Deductions -- UCC Articles 71 and 72

**Article 71 -- Additions to the price actually paid or payable:**

| Addition | Reference |
|---|---|
| Commissions and brokerage (except buying commissions) | Art. 71(1)(a) |
| Cost of containers and packing | Art. 71(1)(a) |
| **Assists:** Value of materials, tools, dies, engineering, artwork, etc. supplied by the buyer free of charge or at reduced cost | Art. 71(1)(b) |
| **Royalties and licence fees** related to the goods, paid as a condition of sale | Art. 71(1)(c) |
| Proceeds of subsequent resale accruing to the seller | Art. 71(1)(d) |
| **Transport and insurance costs** to the point of entry into the EU customs territory | Art. 71(1)(e) |
| Loading and handling charges at point of entry | Art. 71(1)(e) |

**Article 72 -- Deductions (elements NOT included in customs value):**

| Deduction | Reference |
|---|---|
| Transport costs after arrival at the place of introduction into the EU | Art. 72(a) |
| Charges for construction, erection, assembly, maintenance after importation | Art. 72(b) |
| Interest charges under a financing arrangement | Art. 72(c) |
| Charges for reproduction rights | Art. 72(d) |
| Buying commissions | Art. 72(e) |
| Import duties and taxes payable in the EU | Art. 72(f) |

### 5.4 Valuation Simplification -- Article 73 UCC

Where certain elements (assists, royalties, transport costs) cannot be quantified at the time of import, the customs value may be determined using simplified means -- e.g., using provisional values adjusted later. This is only available for the transaction value method.

### 5.5 Binding Valuation Information (BVI)

Per UCC Recital (22), all customs decisions including binding information are covered by the same rules framework. Companies may request binding valuation rulings from customs authorities to gain legal certainty on how specific elements affect their customs value. The ruling is binding on the customs authorities of the issuing Member State for a defined period.

---

## 6. SPECIAL PROCEDURES

**Legal basis:** Title VII of the UCC (Articles 210--262), supplemented by the DA (2015/2446) and IA (2015/2447)

### 6.1 Transit

#### Union Transit (T1/T2)

| Type | Description |
|---|---|
| **T1 (External Transit)** | Non-Union goods moved through the EU customs territory with suspension of duties and taxes until customs clearance at destination |
| **T2 (Internal Transit)** | Union goods moved temporarily outside the EU customs territory (e.g., through Switzerland) and back in without losing their Union status |

#### Common Transit Convention (CTC)
- **Legal basis:** Convention of 20 May 1987
- **Parties:** EU Member States, EFTA countries (Iceland, Norway, Liechtenstein, Switzerland), Turkey, North Macedonia, Serbia, UK (since 1 January 2021), Ukraine (since 1 October 2022), Georgia, Moldova, Montenegro
- **IT system:** New Computerised Transit System (NCTS) -- electronic declaration and processing; currently deployed as Release 5, with Release 6 in implementation

#### TIR (Transports Internationaux Routiers)
- Uses TIR carnet as customs declaration and internationally valid guarantee
- Primarily used for movements to/through non-EU countries (Turkey, Ukraine, Middle East, North Africa)
- TIR holders must also lodge data in the NCTS system

### 6.2 Customs Warehousing

Non-Union goods may be stored in the EU indefinitely with suspension of import duties, taxes, and commercial policy measures. Working/processing on warehoused goods is limited to preservation. Goods may subsequently be released for free circulation (with duty payment), placed under another special procedure, or re-exported.

### 6.3 Inward Processing (IP)

Non-Union goods are imported for processing operations (manufacturing, repair, transformation) within the EU customs territory with suspension of customs duties and import VAT during processing. If processed goods are re-exported, no duties are collected. If released for free circulation within the EU, duties become payable.

**Authorization required** from customs authorities, specifying the discharge period.

### 6.4 Outward Processing (OP)

The reverse of inward processing: Union goods are temporarily exported from the EU for processing (manufacturing, assembly, repair) in a non-EU country. Processed products may be re-imported with total or partial relief from import duties. Equivalent goods provisions may apply.

### 6.5 Temporary Admission

Non-Union goods may be used within the EU for a limited period with **full or partial relief** from import duties and VAT, provided:
- No alteration of goods beyond normal depreciation
- Goods are clearly identifiable
- Holder is generally established outside the EU
- Goods are re-exported at the end of the period

### 6.6 End-Use

Goods may be released for free circulation at a **reduced or zero rate of duty** if used for a specific purpose listed in the EU rules. Customs supervision continues for up to two years after first use for goods suitable for repeated use. End-use eligible goods are identified in TARIC.

### 6.7 Free Zones

**Legal basis:** Articles 243--249 UCC

Parts of the EU customs territory designated by Member States where non-Union goods may be introduced free of import duty, taxes, and commercial policy measures. Goods may subsequently be: released for free circulation (with duty payment); placed under another special procedure; or re-exported. Union goods may also enter free zones.

**Key difference from other special procedures:** No authorization is required for use of free zones; no financial guarantee required.

---

## 7. ICS2 AND SAFETY/SECURITY

### 7.1 Import Control System 2 (ICS2)

**Legal basis:** UCC Articles 127--129; Commission Implementing Regulation (EU) 2015/2447 (as amended)

ICS2 is the EU's advance cargo information and security system, requiring all economic operators bringing goods into or transiting through the EU to submit safety and security data before arrival.

### 7.2 Entry Summary Declaration (ENS)

All goods transported to or through the EU require a complete ENS filed in the ICS2 system prior to arrival. The ENS must contain all required data elements specific to the mode of transport and business model, including:
- 6-digit HS code for each commodity line
- Detailed goods descriptions
- Shipper and consignee information
- Container and transport details

Based on ENS data, customs authorities perform **pre-loading or pre-arrival risk analysis**, which may result in:
- Requests for additional information
- High-risk cargo/mail screening orders
- Do-not-load instructions

### 7.3 ICS2 Release Schedule (Phased Rollout)

| Release | Date | Scope |
|---|---|---|
| **Release 1** | 15 March 2021 | Air cargo pre-loading data (postal/express) |
| **Release 2** | 1 March 2023 | All air cargo -- house-level ENS data |
| **Release 3 (maritime)** | 4 December 2024 | Maritime carrier ENS filing begins |
| **Release 3 (maritime house-level)** | 1 April 2025 | House-level filing mandatory for maritime |
| **Release 3 (road/rail)** | 1 September 2025 | Road and rail carriers must comply |

### 7.4 ICS2 Release 3 -- Key Requirements (2025)

- **Multiple filing:** Different parties in the supply chain can submit partial ENS data; carriers and house filers must coordinate for complete and compliant filings
- **House-level data:** Forwarders must either provide full house-level data to carriers or submit separate filings
- **Message version transition:** ICS2 message version 3 (v3) becomes mandatory 3 February 2026; v2 decommissioned
- **Temporary derogations:** Several EU Member States and Northern Ireland received temporary extensions to the 1 September 2025 implementation deadline

### 7.5 Non-Compliance Consequences

Missing, incomplete, or poor-quality ENS data may result in:
- Shipment delays
- Penalties
- Refusal of entry into the EU
- Cargo stopped at EU borders

---

## 8. EU REGULATORY REQUIREMENTS (EQUIVALENT TO US PGAs)

### 8.1 REACH (Registration, Evaluation, Authorisation and Restriction of Chemicals)

**Legal basis:** Regulation (EC) No 1907/2006

REACH is the main EU regulation protecting human health and the environment from chemical risks.

**Key obligations:**
- **Registration:** Compulsory for any company manufacturing or importing 1 tonne or more per year of a substance. Non-registered substances must not be marketed or used.
- **Evaluation:** ECHA evaluates registration dossiers and testing proposals
- **Authorisation:** Substances of Very High Concern (SVHCs) on the Candidate List (250 entries as of June 2025) require authorization for continued use
- **Restriction:** Annex XVII lists restricted substances and conditions of use

**Requirements for non-EU manufacturers:**
- Cannot register directly with ECHA
- Must use one of three pathways: EU branch office, EU-based importer, or Only Representative (OR)

**Recent developments:** Commission Regulation (EU) 2025/1090 (3 June 2025) added two new restricted substances to Annex XVII. The comprehensive REACH revision, anticipated since 2022, is covered by the EU Chemicals Industry Action Plan (CIAP, July 2025).

### 8.2 CE Marking Requirements

CE marking (Conformite Europeenne) indicates conformity with EU health, safety, and environmental protection standards. Required for products covered by specific EU directives/regulations including:

- Machinery Directive (2006/42/EC) / new Machinery Regulation (EU) 2023/1230
- Low Voltage Directive (2014/35/EU)
- Electromagnetic Compatibility Directive (2014/30/EU)
- Radio Equipment Directive (2014/53/EU)
- Medical Devices Regulation (EU) 2017/745
- RoHS Directive (2011/65/EU) -- Restriction of Hazardous Substances in electrical/electronic equipment
- Personal Protective Equipment Regulation (EU) 2016/425
- Construction Products Regulation (EU) No 305/2011

Manufacturers, importers, and distributors must ensure products bear the CE mark before placing them on the EU market. The manufacturer (or authorized representative) issues a Declaration of Conformity.

### 8.3 Product Safety -- General Product Safety Regulation (GPSR)

**Legal basis:** Regulation (EU) 2023/988 (GPSR), directly applicable from 13 December 2024

The GPSR replaced the previous General Product Safety Directive (2001/95/EC). It applies to all consumer products placed on the EU/EEA market and establishes the basic legal minimum safety requirements. Economic operators (manufacturers, importers, distributors, online marketplaces) must ensure products are safe before marketing.

### 8.4 Food Safety -- RASFF System and Health Certificates

**RASFF (Rapid Alert System for Food and Feed):**
- Established to ensure rapid exchange of information between EU countries when risks to public health are detected in the food chain
- Linked to TRACES (see below) for comprehensive import control

**Health Certificates:**
- **Mandatory** for all EU imports of animal-origin products
- **Phytosanitary certificates** required for all plant products that could introduce pests
- The EU only accepts veterinary certificates from competent authorities in supplying countries after EU recognition of the country's animal health status and food safety guarantees

### 8.5 Phytosanitary and Veterinary Controls -- TRACES System

**TRACES (Trade Control and Expert System):**
- The European Commission's online platform for sanitary and phytosanitary (SPS) certification
- Used by approximately 90 countries, with 113,000+ users worldwide
- Mandatory for imports of animals, animal products, food/feed of non-animal origin, and plants into the EU

**Key TRACES modules:**

| Module | Function |
|---|---|
| **CHED-PP** | Phytosanitary: consignments of plants/plant products requiring phytosanitary certificate |
| **CHED-D** | Food/feed of non-animal origin subject to increased temporary controls |
| **CHED-A** | Animals and germinal products |
| **CHED-P** | Products of animal origin, composite products |
| **PHYTO** | Issuance of phytosanitary certificates by non-EU countries (connected to IPPC ePhyto Hub since May 2020) |
| **IMPORT** | Issuance of official certificates by non-EU countries |

**Customs integration:** TRACES is connected to **EU CSW-CERTEX** (EU Single Window Environment for Customs, Regulation (EU) 2022/2399), enabling real-time automated exchange of information between customs and SPS authorities. All EU Member States must be connected to EU CSW-CERTEX by **3 March 2025** for CHED documents and COI certificates.

### 8.6 Veterinary Controls

**Legal basis:** Regulation (EU) 2017/625 (Official Controls Regulation)

Veterinary Border Control Posts (BCPs) perform documentary, identity, and physical checks on:
- Live animals
- Products of animal origin
- Germinal products (semen, embryos)
- Animal by-products

All consignments must be pre-notified in TRACES with the relevant CHED document before arrival at the BCP.

### 8.7 Dual-Use Goods -- EU Dual-Use Regulation

**Legal basis:** Regulation (EU) 2021/821 (replaced Regulation (EC) No 428/2009; in force since 9 September 2021)

This regulation establishes an EU regime for the control of exports, brokering, technical assistance, transit, and transfer of dual-use items -- goods and technologies with both civilian and military/WMD applications.

**Annex I** lists controlled items, aligned with multilateral regimes:
- Wassenaar Arrangement
- Missile Technology Control Regime (MTCR)
- Nuclear Suppliers Group (NSG)
- Australia Group

**Authorization types:**
| Type | Description |
|---|---|
| **EU General Export Authorisations (UGEAs)** | Pre-authorized exports to specified low-risk destinations (AU, CA, IS, JP, LI, NZ, NO, CH, UK, US) |
| **EU007** | Intra-group export of software and technology |
| **EU008** | Encryption items |
| **National general export authorisations** | Issued by Member States, consistent with UGEAs |
| **Individual/global authorisations** | Issued by national authorities for specific exporters/items, valid up to 2 years |
| **Large project authorisation** | New under 2021/821, for major projects |

**Customs interaction:** Proof of export authorization must be furnished to the customs office handling the export declaration. Member States may restrict dual-use customs formalities to specifically empowered customs offices.

**Cyber-surveillance catch-all:** New provision for export control of unlisted digital surveillance and interception technologies that may be used for internal repression or human rights violations.

**Annual updates:** The Annex I list is updated annually via Delegated Regulation to align with multilateral export control regime decisions (most recent update: September 2024).

---

## KEY EU CUSTOMS REFERENCE LINKS

- EUR-Lex UCC: https://eur-lex.europa.eu/eli/reg/2013/952/oj/eng
- DG TAXUD UCC Legislation: https://taxation-customs.ec.europa.eu/customs/union-customs-code/ucc-legislation_en
- TARIC Consultation: https://taxation-customs.ec.europa.eu/customs/calculation-customs-duties/customs-tariff/eu-customs-tariff-taric_en
- Access2Markets Portal: https://trade.ec.europa.eu/access-to-markets/en/content/tariffs-2
- ICS2: https://taxation-customs.ec.europa.eu/customs/customs-security/import-control-system-2_en
- TRACES: https://food.ec.europa.eu/horizontal-topics/traces_en
- EORI Validation: https://taxation-customs.ec.europa.eu/customs/customs-procedures-import-and-export/customs-operations/economic-operators-registration-and-identification-number-eori_en
- EU GSP Hub: https://gsphub.eu/about-gsp
- PEM Convention: https://taxation-customs.ec.europa.eu/customs/international-affairs/pan-euro-mediterranean-cumulation-and-pem-convention_en
- EU Customs Valuation Compendium (2025): https://taxation-customs.ec.europa.eu/document/download/9a13b89e-9e5e-482e-be0b-f593d96bc815_en

---

## KEY REGULATION NUMBERS SUMMARY

| Regulation | Number | Subject |
|---|---|---|
| Union Customs Code | (EU) No 952/2013 | Main customs law |
| UCC Delegated Act | (EU) 2015/2446 | Supplementary detailed rules |
| UCC Implementing Act | (EU) 2015/2447 | Uniform implementation conditions |
| UCC Transitional Delegated Act | (EU) 2016/341 | Transitional IT rules |
| UCC Electronic Systems | (EU) 2023/1070 | Electronic customs systems |
| Combined Nomenclature / TARIC | (EEC) No 2658/87 | Tariff nomenclature |
| Anti-Dumping Basic Regulation | (EU) 2016/1036 | Anti-dumping investigations |
| Anti-Subsidy Basic Regulation | (EU) 2016/1037 | Countervailing duty investigations |
| GSP Regulation | (EU) No 978/2012 | Generalized Scheme of Preferences |
| Autonomous Tariff Suspensions | (EU) No 2021/2278 | Tariff suspensions |
| REACH | (EC) No 1907/2006 | Chemicals registration/restriction |
| General Product Safety Regulation | (EU) 2023/988 | Consumer product safety |
| Dual-Use Regulation | (EU) 2021/821 | Export controls on dual-use items |
| Official Controls Regulation | (EU) 2017/625 | Food/feed/animal/plant controls |
| Single Window for Customs | (EU) 2022/2399 | EU CSW-CERTEX |
| Excise Directive | Council Directive 2020/262 | General excise arrangements |
| Energy Taxation Directive | Council Directive 2003/96/EC | Energy product excise |
| RoHS Directive | 2011/65/EU | Hazardous substances in electronics |
| Machinery Regulation | (EU) 2023/1230 | Machinery safety (replaces 2006/42/EC) |
| Medical Devices Regulation | (EU) 2017/745 | Medical device safety |

---

**Sources consulted for this research:**

- [EUR-Lex -- Regulation (EU) No 952/2013](https://eur-lex.europa.eu/eli/reg/2013/952/oj/eng)
- [DG TAXUD -- UCC Legislation](https://taxation-customs.ec.europa.eu/customs/union-customs-code/ucc-legislation_en)
- [EUR-Lex -- Union Customs Code summary](https://eur-lex.europa.eu/EN/legal-content/summary/union-customs-code.html)
- [EUR-Lex -- TARIC online database summary](https://eur-lex.europa.eu/EN/legal-content/summary/the-online-integrated-customs-tariff-database-taric.html)
- [DG TAXUD -- EU Customs Tariff (TARIC)](https://taxation-customs.ec.europa.eu/customs/calculation-customs-duties/customs-tariff/eu-customs-tariff-taric_en)
- [DG TAXUD -- Common Customs Tariff](https://taxation-customs.ec.europa.eu/customs/calculation-customs-duties/customs-tariff_en)
- [DG TAXUD -- Customs Declaration](https://taxation-customs.ec.europa.eu/customs-4/customs-procedures-import-and-export-0/customs-procedures/customs-declaration_en)
- [DG TAXUD -- Centralised Clearance](https://taxation-customs.ec.europa.eu/customs/customs-procedures-import-and-export/customs-operations/centralised-clearance_en)
- [DG TAXUD -- EORI](https://taxation-customs.ec.europa.eu/customs/customs-procedures-import-and-export/customs-operations/economic-operators-registration-and-identification-number-eori_en)
- [DG TAXUD -- Customs Valuation](https://taxation-customs.ec.europa.eu/customs/calculation-customs-duties/customs-valuation_en)
- [DG TAXUD -- ICS2](https://taxation-customs.ec.europa.eu/customs/customs-security/import-control-system-2_en)
- [DG TAXUD -- Free Zones](https://taxation-customs.ec.europa.eu/customs/free-zones_en)
- [DG TAXUD -- Specific Use](https://taxation-customs.ec.europa.eu/customs/customs-procedures-import-and-export/what-importation/specific-use_en)
- [DG TAXUD -- Autonomous Tariff Suspensions](https://taxation-customs.ec.europa.eu/customs/calculation-customs-duties/customs-tariff/suspensions-autonomous-tariff-suspensions_en)
- [DG TAXUD -- Quota (TRQs)](https://taxation-customs.ec.europa.eu/customs/calculation-customs-duties/customs-tariff/quota-tariff-quotas-and-ceilings_en)
- [DG TAXUD -- NCTS](https://taxation-customs.ec.europa.eu/online-services/online-services-and-databases-customs/new-computerised-transit-system-ncts_en)
- [DG TAXUD -- PEM Convention](https://taxation-customs.ec.europa.eu/customs/international-affairs/pan-euro-mediterranean-cumulation-and-pem-convention_en)
- [DG TAXUD -- Excise Duties](https://taxation-customs.ec.europa.eu/taxation/excise-duties_en)
- [DG TAXUD -- VAT Rates](https://taxation-customs.ec.europa.eu/taxation/vat/vat-directive/vat-rates_en)
- [DG TRADE -- EU Trade Agreements](https://policy.trade.ec.europa.eu/eu-trade-relationships-country-and-region/negotiations-and-agreements_en)
- [DG TRADE -- GSP](https://policy.trade.ec.europa.eu/development-and-sustainability/generalised-scheme-preferences_en)
- [DG TRADE -- Anti-Dumping Measures](https://policy.trade.ec.europa.eu/enforcement-and-protection/trade-defence/anti-dumping-measures_en)
- [Access2Markets -- FTAs](https://trade.ec.europa.eu/access-to-markets/en/content/free-trade-agreements)
- [Access2Markets -- PEM Convention Rules of Origin](https://trade.ec.europa.eu/access-to-markets/en/content/rules-origin-pan-euro-mediterranean-convention)
- [Access2Markets -- EU-South Korea FTA](https://trade.ec.europa.eu/access-to-markets/en/content/eu-south-korea-free-trade-agreement)
- [FIATA -- ICS2 Release 3 Alert](https://fiata.org/n/eu-ics2-alert-ics2-release-3-goes-live-on-1-september-2025/)
- [European Commission -- TRACES](https://food.ec.europa.eu/horizontal-topics/traces_en)
- [European Commission -- REACH Regulation](https://environment.ec.europa.eu/topics/chemicals/reach-regulation_en)
- [EUR-Lex -- Dual-Use Regulation 2021/821](https://eur-lex.europa.eu/eli/reg/2021/821/oj/eng)
- [EUR-Lex -- GSP summary](https://eur-lex.europa.eu/EN/legal-content/glossary/generalised-system-of-preferences-gsp.html)
- [Tax Foundation -- 2026 EU VAT Rates](https://taxfoundation.org/data/all/eu/value-added-tax-vat-rates-europe/)
- [Your Europe -- VAT Rules and Rates](https://europa.eu/youreurope/business/taxation/vat/vat-rules-rates/index_en.htm)
- [EU Consilium -- EU-Mercosur](https://www.consilium.europa.eu/en/press/press-releases/2026/01/09/eu-mercosur-council-greenlights-signature-of-the-comprehensive-partnership-and-trade-agreement/)
- [EU Customs Reform 2025](https://taxation-customs.ec.europa.eu/news/milestone-eu-customs-reform-member-states-adopt-common-position-new-union-customs-code-ucc-2025-06-27_en)