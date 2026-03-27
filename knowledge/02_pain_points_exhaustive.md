# Global Customs Clearance: Exhaustive Pain Points

> **FedEx Global Clearance Knowledge Base**
> Last Updated: January 2026

---

## Executive Summary

Customs clearance pain points are not hypothetical — they are experienced daily by every participant in the cross-border trade ecosystem. This document catalogs the operational, financial, and strategic pain points across six key stakeholder groups. Each pain point is documented with its root cause, scale, and downstream consequences. These documented pain points form the foundation for our clearance transformation vision — every element of the future state must map back to solving one or more of these specific, real-world problems.

---

## 1. Express Carriers (FedEx, UPS, DHL)

### 1.1 Volume Surge from De Minimis Elimination

**The Problem:** The suspension of US Section 321 de minimis effective August 29, 2025, converted approximately 4 million daily parcels from minimal-data informal entries to full formal entries requiring 10-digit HTSUS classification, country of origin, valuation, and PGA data. Similar changes are rolling out in the EU (July 2026), UK (by 2029), and Thailand (2024).

**Root Cause:** Governments can no longer afford the revenue loss and security gap created by millions of unexamined, untaxed parcels. The de minimis exemption was designed for an era of occasional personal imports, not industrial-scale cross-border e-commerce.

**Scale:** FedEx processes millions of international shipments daily. The formal entry conversion increases clearance workload per shipment by an estimated 5-10x for what were previously de minimis parcels.

**Consequences:**
- Clearance processing times lengthen across the board, not just for formerly de minimis shipments — the increased volume creates queue effects that delay all entries.
- Technology systems (ACE filing, broker workstations, shipment management) face throughput demands they were never designed for.
- Staffing models built around historical formal entry volumes are inadequate.
- Transit time commitments (next-day, 2-day international) become difficult to guarantee when clearance becomes the bottleneck.

### 1.2 Caging Overflow and Facility Constraints

**The Problem:** Shipments placed on hold by CBP or PGAs for examination, document review, or UFLPA detention must be physically stored in bonded facilities ("cages"). The surge in formal entries — combined with increased enforcement on UFLPA, fentanyl, counterfeits, and unsafe products — has overwhelmed caging capacity.

**Root Cause:** Cage infrastructure was sized for historical hold rates applied to historical formal entry volumes. Both variables have increased dramatically.

**Scale:**
- Cage dwell times that historically averaged 1-3 days for routine holds now frequently extend to 5-10 days, and UFLPA detentions can last weeks to months.
- Physical cage space at major gateway facilities (Memphis, Indianapolis, Newark, Los Angeles, Miami) is at or exceeding capacity.
- Overflow requires use of off-site bonded warehouses, adding transport costs and complicating the chain of custody.

**Consequences:**
- Increased facility costs (lease, staffing, security for expanded cage operations).
- Deterioration of perishable goods held in examination.
- Customer dissatisfaction — shipments "stuck in customs" with no clear resolution timeline.
- Carrier liability for goods that deteriorate, are abandoned, or require disposal.

### 1.3 Carrier Disposal Liability

**The Problem:** When importers abandon shipments after customs holds, exam failures, or duty assessments, the carrier is left holding the physical goods and the legal and financial liability for their disposition.

**Root Cause:** The importer of record is legally responsible for goods presented to customs, but when they refuse to pay duties or provide required documentation, the goods remain in the carrier's physical custody. CBP's general order process is slow, and carriers bear storage costs.

**Consequences:**
- Costs of storage, destruction, and re-export of abandoned goods.
- Environmental liability for hazardous goods disposal.
- Diversion of operational resources to manage abandoned inventory.
- No revenue against these costs — it is pure loss.

### 1.4 Shipper Data Quality

**The Problem:** The quality of trade data provided by shippers — particularly e-commerce sellers and small businesses — is chronically inadequate for customs compliance. Descriptions are vague ("gift," "household goods," "sample"), values are understated, and HTSUS codes are absent or incorrect.

**Root Cause:** Many shippers have never been required to provide customs data (because de minimis exempted them). They lack knowledge of customs requirements, have no systems for trade data management, and may not understand the language of the destination country's customs forms.

**Scale:** Industry estimates suggest that **30-50% of e-commerce shipment data** requires manual intervention, correction, or enhancement before it can be filed with customs.

**Consequences:**
- Manual data correction by carrier clearance teams adds cost and time to every affected shipment.
- Incorrect data leads to misclassification, which leads to incorrect duty assessment, which leads to importer disputes and potential penalties.
- Poor data triggers algorithmic risk flags, increasing hold and exam rates for the carrier's traffic.
- Chronic data quality issues erode carrier credibility with customs authorities.

### 1.5 System Capacity and Technology Limitations

**The Problem:** Core clearance technology systems — including internal broker workstations, ACE filing interfaces, and shipment management platforms — were architected for historical volumes and entry types. They are straining under the current load.

**Root Cause:** Legacy system architectures with batch processing, limited API throughput, and manual exception handling were not designed for the throughput, speed, and data complexity now required.

**Consequences:**
- System slowdowns and outages during peak processing windows.
- Inability to scale linearly with volume growth.
- Manual workarounds required for edge cases that systems cannot handle.
- Integration challenges with new regulatory data requirements (UFLPA evidence, CBAM data, EUDR certifications).

### 1.6 24/7 Staffing for Global Clearance

**The Problem:** Customs clearance operates on customs authority business hours, which vary by country. But express carrier operations run 24/7, and customer expectations are for continuous progress on their shipments.

**Root Cause:** Customs authorities in most countries operate on government business hours. While some have extended hours for express operations, true 24/7 customs processing is rare. Carrier clearance teams must staff across time zones to file and manage entries when each country's customs authority is active.

**Consequences:**
- High labor cost for round-the-clock, multilingual clearance operations.
- Difficulty recruiting and retaining talent for overnight and weekend shifts.
- Shipments arriving outside customs hours face unavoidable delays until the next processing window.

### 1.7 Exam and Hold Management Complexity

**The Problem:** When CBP or PGAs issue a hold or exam order, the carrier must coordinate physical handling of the shipment (routing to examination facility, presenting for exam, managing exam outcomes), documentation (responding to CBP requests, gathering information from shippers/importers), and communication (notifying customers, managing expectations).

**Root Cause:** The exam process is inherently multi-party (carrier, customs, importer, broker, PGA) and largely manual. There is no single system that tracks exam status end-to-end. Communication between parties often happens via email, fax, or phone.

**Consequences:**
- Each exam requires significant operational coordination and labor.
- Lack of real-time exam status visibility means carriers cannot accurately inform customers when their shipment will be released.
- Exam outcomes (release, further hold, seizure) are unpredictable, making downstream logistics planning difficult.

---

## 2. Customs Brokers

### 2.1 Labor Shortage and Workforce Demographics

**The Problem:** The licensed customs broker workforce is aging and not being replenished at a rate sufficient to meet growing demand. The CBP Customs Broker License Examination has a pass rate of only **5-15%** in recent years, and fewer candidates are sitting for the exam.

**Root Cause:** The profession is not well-known outside the trade community, compensation has historically lagged other finance and compliance roles, and the work is perceived as highly technical and unglamorous. The exam is extremely difficult, covering tariff classification, customs law, valuation, trade agreements, and penalty provisions.

**Scale:**
- The average age of licensed customs brokers is increasing, with a significant cohort approaching retirement.
- Demand for broker services is surging due to de minimis elimination, tariff complexity, and regulatory expansion.
- The gap between broker supply and demand is widening.

**Consequences:**
- Rising labor costs as experienced brokers command premium compensation.
- Increased error rates as less experienced staff handle complex entries.
- Longer processing times as broker capacity is stretched.
- Concentration risk — key-person dependencies on a small number of experienced brokers.

### 2.2 Legacy Technology and Manual Processes

**The Problem:** Many customs brokerage operations still rely on legacy systems, manual data entry, and paper-based workflows. Even where technology exists, it often consists of disconnected systems that require duplicate data entry and manual exception management.

**Root Cause:** The brokerage industry has historically underinvested in technology. Many brokerages are small businesses (fewer than 50 employees) that lack the capital or technical expertise for modernization. Even large brokerages have accumulated technical debt from decades of incremental system additions.

**Consequences:**
- High per-entry processing cost driven by manual labor.
- Error rates from manual data entry (estimated 2-5% of entries contain errors requiring amendment).
- Inability to scale operations without proportional headcount increases.
- Slow adaptation to new regulatory requirements that demand system changes.

### 2.3 Margin Compression

**The Problem:** Per-entry brokerage fees have been declining for years as shippers treat customs brokerage as a commoditized service and negotiate aggressively on price. Simultaneously, the cost of providing brokerage services is increasing due to regulatory complexity, technology requirements, and labor costs.

**Root Cause:** The market perceives customs brokerage as a transactional commodity — file the entry, pay the duty, move on. In a commoditized market, price competition drives margins down. The value of compliance expertise, risk management, and trade optimization is not adequately captured in per-entry pricing.

**Consequences:**
- Brokerages cut corners to maintain margins — reducing quality assurance, training, and technology investment.
- Industry consolidation as small brokerages cannot sustain operations.
- Talent flight to adjacent, better-compensated roles in compliance, consulting, and technology.
- Underinvestment in the capabilities (AI, automation, advisory services) needed for the future.

### 2.4 Regulatory Change Velocity

**The Problem:** The pace of regulatory change has accelerated dramatically. New tariff programs, trade agreements, sanctions, compliance mandates, and filing requirements are introduced on timelines that exceed broker capacity to absorb and implement.

**Root Cause:** Geopolitical volatility, domestic political priorities, and the regulatory response to e-commerce and supply chain security are generating an unprecedented volume of regulatory change. See Regulatory Landscape document for the full catalog.

**Scale:** In the 2023-2026 period alone, brokers have had to implement changes for: IEEPA reciprocal tariffs (multiple rounds), Section 301 increases, de minimis elimination, UFLPA expansion, EU ICS2, CBAM, EUDR, UK BTOM phases, and dozens of smaller changes.

**Consequences:**
- Compliance risk from incorrect or delayed implementation of new requirements.
- Training burden — every change requires updating procedures, systems, and staff knowledge.
- Client communication burden — importers look to their broker to explain what changed and what it means.
- Change fatigue — staff morale and retention suffer from constant disruption.

### 2.5 Data Quality from Shippers

**The Problem:** Brokers are downstream of shipper data quality. When shippers provide incomplete, inaccurate, or ambiguous product descriptions, values, and origin information, the broker must invest time correcting, clarifying, and validating data before filing.

**Root Cause:** Same as carrier data quality issues (Section 1.4). Compounded by the fact that brokers typically receive data after goods are already in transit or at the border — leaving little time for correction.

**Consequences:**
- Manual data enhancement adds cost that brokers struggle to pass through to clients.
- "Garbage in, garbage out" — entries filed with poor data are at higher risk for holds, exams, and penalties.
- Broker liability risk — the broker's license is on the line for entries filed with incorrect information, even when the source data was provided by the importer.

---

## 3. Importers and Exporters

### 3.1 Classification Complexity

**The Problem:** Determining the correct HTSUS code for a product is the foundational act of customs compliance, yet it remains one of the most complex, subjective, and error-prone steps in the process.

**Root Cause:** The Harmonized System (HS) was designed in the 1980s for a world of simpler products. Modern products — particularly technology, multi-material goods, and novel compositions — do not fit neatly into the HS structure. The US adds further specificity with its own 8- and 10-digit subheadings, creating **over 17,000 unique tariff lines**.

**Scale:**
- CBP estimates that **up to 20% of entries** contain classification errors.
- With tariff programs stacking, a misclassification that shifts a product from one HS heading to another can change the duty rate by 50 percentage points or more.
- Binding rulings from CBP (via HQ or NY rulings) take months and sometimes produce results that conflict with industry practice.

**Consequences:**
- Financial exposure from underpayment (penalties up to 4x the lost revenue) or overpayment (lost margin from duties that should not have been paid).
- Operational delays when customs questions a classification and places the shipment on hold.
- Legal risk from intentional misclassification allegations (fraud penalties, criminal liability).
- Competitive disadvantage — companies with better classification practices pay less duty, legally.

### 3.2 Unexpected Duty Bills and Tariff Volatility

**The Problem:** Importers frequently discover the true cost of importing only after goods have shipped, cleared customs, and duty bills arrive — sometimes weeks or months after the transaction. With stacking tariff programs and frequent rate changes, predicting the duty cost of a shipment has become extremely difficult.

**Root Cause:** Tariff rates are determined by a combination of product classification, country of origin, applicable tariff programs (Section 301, 232, IEEPA, AD/CVD, FTA preferences), and valuation methodology. Any of these variables can change with little notice. The interaction between programs is complex and not intuitive.

**Scale:**
- IEEPA reciprocal tariffs were announced and modified multiple times in 2025, sometimes with days or weeks of notice.
- For China-origin goods, effective duty rates went from historically 0-10% (pre-2018) to 60-145%+ in some categories — a change that fundamentally alters product economics.
- Companies report customs duty spend increasing 200-400% in recent years without any change in their product mix or sourcing.

**Consequences:**
- Profit margin erosion or elimination on affected product categories.
- Inability to accurately price products for customers (particularly DDP shipments).
- Cash flow impact — duty payments are required before goods are released, creating working capital strain.
- Disputes between importers, brokers, and carriers over duty amounts.

### 3.3 Free Trade Agreement (FTA) Underutilization

**The Problem:** Significant percentages of eligible trade do not claim available FTA preferences (USMCA, KORUS, US-Australia, etc.), resulting in importers paying duties they are legally entitled to avoid.

**Root Cause:** Claiming FTA preferences requires:
- Determining that the product qualifies under the FTA's rules of origin (which can be complex product-specific rules requiring detailed BOM and sourcing data).
- Obtaining and maintaining valid certificates of origin from suppliers.
- Documenting the origin determination and maintaining records for audit.

Many importers — particularly SMEs — lack the expertise, systems, and supplier relationships to perform these steps.

**Scale:**
- Industry estimates suggest **15-30% of trade eligible for FTA preferences** does not claim them.
- For USMCA alone, this represents billions of dollars in unclaimed duty savings annually.

**Consequences:**
- Direct financial loss — duties paid that could have been legally avoided.
- Competitive disadvantage versus competitors who do claim preferences.
- Reluctance to invest in FTA utilization because the compliance cost seems high relative to the per-shipment benefit (not recognizing the cumulative value).

### 3.4 Documentation Burden

**The Problem:** International trade requires extensive documentation — commercial invoices, packing lists, certificates of origin, phytosanitary certificates, conformity certificates, safety data sheets, and more. Requirements vary by product, country, and regulatory regime.

**Root Cause:** Each country's customs authority, PGAs, and trade partners have their own documentation requirements. There is no universal standard for trade documents, and requirements change frequently. Many processes still require paper documents or specific electronic formats.

**Consequences:**
- Administrative overhead — dedicated staff or departments manage trade documentation.
- Delays when documents are missing, incorrect, or in the wrong format.
- Risk of goods being held at the border for documentary deficiencies.
- Duplication of effort — the same information is entered into multiple documents and systems.

### 3.5 Valuation Disputes

**The Problem:** Customs valuation — determining the price on which duties are assessed — is a frequent source of disputes between importers and customs authorities. Transfer pricing between related parties, assists, royalties, buying commissions, and first sale valuation create complexity.

**Root Cause:** WTO Valuation Agreement provides a hierarchy of valuation methods, but application to specific transactions is often subjective. Related-party transactions (which account for approximately 40% of global trade) are particularly scrutinized.

**Consequences:**
- Duty reassessments and penalties when customs authorities disagree with declared value.
- Cash deposits and bonds tied up during valuation disputes.
- Audit exposure — valuation is one of the most common areas of post-entry audit focus.

### 3.6 UFLPA Supply Chain Mapping

**The Problem:** Compliance with UFLPA requires importers to trace their supply chain from raw materials through finished product to demonstrate goods are not produced with forced labor in Xinjiang. Most importers do not have this level of supply chain visibility.

**Root Cause:** Global supply chains are long, complex, and opaque. A finished product may contain dozens of components from multiple tiers of suppliers. Raw materials (cotton, polysilicon, minerals) are fungible and often mixed in processing. Few companies have systems to trace inputs beyond their direct (tier-1) suppliers.

**Consequences:**
- Inability to respond to UFLPA detentions with the required "clear and convincing" evidence.
- Goods detained for weeks or months, with potential seizure and forfeiture.
- Need to invest in supply chain mapping technology and supplier audit programs.
- Potential loss of suppliers who cannot or will not provide the required traceability data.

---

## 4. E-Commerce Sellers and Platforms

### 4.1 De Minimis Elimination: Business Model Disruption

**The Problem:** Many cross-border e-commerce business models were built on de minimis — the assumption that low-value goods would cross borders duty-free. The elimination of de minimis in the US (and planned elimination in the EU and UK) fundamentally undermines these models.

**Root Cause:** De minimis was a regulatory arbitrage that e-commerce sellers and platforms exploited, intentionally or not. Goods were priced and marketed without accounting for duties because no duties were owed. Now they are.

**Scale:**
- Platforms like Shein, Temu, AliExpress, and TikTok Shop built businesses shipping millions of low-value parcels daily under de minimis.
- For some product categories, duties (including IEEPA tariffs) can exceed **60-100% of the product's value** — more than doubling the cost to the consumer.
- The $800 de minimis threshold was used for over 1 billion shipments annually entering the US.

**Consequences:**
- Dramatic price increases for consumers on formerly duty-free goods.
- Restructuring of e-commerce logistics (shift to domestic warehousing, bonded warehouses, FTZ processing).
- Platform sellers exiting markets where duty burden makes products uncompetitive.
- Increased demand for DDP (Delivered Duty Paid) services, which shift the duty burden and compliance responsibility to the seller or platform.

### 4.2 DDP Cost Unpredictability

**The Problem:** Sellers offering Delivered Duty Paid (DDP) terms commit to absorbing all customs costs (duties, taxes, fees). But accurately predicting these costs at the point of sale is extremely difficult given tariff volatility and classification complexity.

**Root Cause:** DDP pricing requires knowing the exact duty rate before the sale, but duty rates depend on classification (which may be disputed), origin (which may be questioned), valuation (which includes freight costs not yet finalized), and tariff programs (which may change between pricing and clearance).

**Consequences:**
- Sellers underestimate duties and absorb losses on DDP shipments.
- Sellers overestimate duties and become price-uncompetitive.
- Frequent repricing disrupts customer expectations and marketplace positioning.
- Some sellers abandon DDP altogether, shifting the burden (and the surprise) to consumers.

### 4.3 Returns Clearance Friction

**The Problem:** International returns require reverse customs clearance — re-exporting goods from the destination country and potentially re-importing them into the origin country. This process is poorly supported by current systems and creates significant friction.

**Root Cause:** Customs systems and processes are designed for imports, not returns. Duty drawback (recovering duties paid on returned goods) is complex and rarely worth pursuing for low-value items. Many countries do not have streamlined return-import processes.

**Consequences:**
- Returns rates for international e-commerce are lower than domestic, partly because the process is so difficult.
- Consumer frustration — returning an international purchase is significantly harder than a domestic one.
- Inventory trapped in destination countries that cannot be cost-effectively repatriated.
- Environmental waste — products destroyed in-country rather than returned.

### 4.4 Marketplace Seller Compliance

**The Problem:** E-commerce platforms host millions of third-party sellers, many of whom are small businesses or individual entrepreneurs with no customs compliance knowledge. The platform bears increasing regulatory responsibility for these sellers' compliance.

**Root Cause:** Regulatory frameworks (EU customs reform, INFORM Act, US de minimis rules) are shifting compliance responsibility to platforms. But platforms have limited visibility into their sellers' products, supply chains, and trade practices.

**Consequences:**
- Platforms must invest in compliance infrastructure (classification tools, restricted party screening, origin verification) for millions of product listings.
- Seller onboarding becomes more complex and expensive.
- Non-compliant sellers create regulatory risk for the platform.
- Tension between platform growth (add more sellers, more products) and compliance (vet every seller, classify every product).

### 4.5 Importer of Record Ambiguity

**The Problem:** In cross-border e-commerce, the question of "who is the importer of record?" is often unclear. Is it the seller, the platform, the consumer, or the logistics provider? Different parties may have different understandings, and the answer has significant legal and financial implications.

**Root Cause:** Traditional customs law was designed for B2B trade where the importer of record is clear (the buyer). In B2C e-commerce, the "buyer" is a consumer who has no customs knowledge, and the "seller" may be a foreign entity with no presence in the destination country. The EU is addressing this by making platforms the deemed importer, but this approach is not universal.

**Consequences:**
- Legal liability uncertainty — who is responsible for duty payment, compliance violations, and penalties?
- Operational confusion — who should the broker contact when information is needed?
- Consumer surprise — when the consumer is the importer of record, they receive unexpected duty bills.
- Potential for regulatory penalties when no party takes clear responsibility.

---

## 5. Consumers

### 5.1 Unexpected Charges

**The Problem:** Consumers purchasing goods from international sellers frequently receive unexpected demands for customs duties, taxes, and processing fees upon delivery.

**Root Cause:** When shipments are sent DDU (Delivered Duty Unpaid), the consumer is the importer of record and is responsible for all customs charges. Most consumers do not understand this when they complete an online purchase. Sellers may not disclose the potential for additional charges, or may advertise "free shipping" that does not include customs costs.

**Scale:** Consumer complaints about unexpected customs charges are among the most common international e-commerce complaints. In some markets, refusal-to-pay rates for COD duty collection exceed **10-15%** of deliveries.

**Consequences:**
- Refused deliveries — consumers decline packages when presented with unexpected charges.
- Negative seller reviews and brand damage.
- Return-to-origin costs for carriers.
- Consumer reluctance to purchase internationally in the future.

### 5.2 "Black Box" Customs Tracking

**The Problem:** When a shipment enters customs processing, consumers typically see a tracking status of "Shipment in customs" or "Clearance in progress" with no further detail or estimated timeline.

**Root Cause:** Customs clearance involves multiple parties (carrier, broker, customs authority, PGAs) and the specific reason for a delay may be complex (missing data, classification question, PGA review, exam, UFLPA detention). Carriers and customs authorities do not share detailed clearance status with consumers, and the information is often not available in real time.

**Consequences:**
- Consumer anxiety and frustration — "where is my package and why is it stuck?"
- Repeated calls to customer service seeking updates, driving up support costs.
- Erosion of trust in international e-commerce.
- Perception that customs is an opaque, arbitrary process.

### 5.3 Delivery Delays

**The Problem:** Customs clearance is the primary source of unpredictable delay in international deliveries. While domestic shipping has become fast and reliable (same-day, next-day, 2-day), international shipping remains subject to multi-day customs delays that are unpredictable at the time of purchase.

**Root Cause:** Clearance time is a function of data quality, regulatory screening, exam rates, PGA coordination, and customs authority processing speed — none of which are fully within the carrier's control.

**Consequences:**
- Consumer disappointment when delivery takes significantly longer than expected.
- Competitive disadvantage for international e-commerce versus domestic alternatives.
- Carriers unable to provide accurate delivery date estimates for international shipments.

### 5.4 Package Disposal

**The Problem:** Shipments that cannot clear customs — due to prohibited contents, failure to pay duties, abandoned by importer, or compliance violations — may ultimately be destroyed or disposed of. Consumers may never receive goods they paid for.

**Root Cause:** When goods fail to clear customs and no party takes action to resolve the issue within statutory timeframes, the goods are subject to general order, seizure, and eventual disposal. Consumers may not understand their obligation to respond or may not know how.

**Consequences:**
- Financial loss for consumers who paid for goods they never received.
- Chargebacks and disputes with sellers and payment processors.
- Environmental waste from destruction of goods.
- Loss of consumer confidence in cross-border commerce.

---

## 6. Government Agencies

### 6.1 Parcel Volume Overwhelming Screening Capacity

**The Problem:** Customs and border agencies worldwide are overwhelmed by the volume of small parcels entering their jurisdictions. CBP processes over **4 million parcels daily** — the vast majority of which, until recently, entered with minimal data under de minimis.

**Root Cause:** The explosion of cross-border e-commerce has created volumes that dwarf the capacity of traditional customs processing infrastructure. Systems, staffing, and procedures were designed for hundreds of thousands of commercial entries per day, not millions of individual parcels.

**Consequences:**
- Low inspection rates — CBP physically examines less than 1% of incoming parcels.
- Risk-based targeting algorithms are only as good as the data available, and de minimis parcels historically provided minimal data.
- Known contraband (fentanyl, counterfeits) flows through the gaps in screening capacity.

### 6.2 Fentanyl and Illicit Goods Screening

**The Problem:** The international parcel stream has been exploited for trafficking fentanyl, fentanyl precursors, and other illicit substances. Government agencies face the challenge of finding dangerous goods in a massive volume of legitimate commerce.

**Root Cause:** The combination of high parcel volumes, limited data requirements (under de minimis), and the physical impossibility of inspecting every package created ideal conditions for illicit trafficking. Even with de minimis elimination improving data availability, the physical screening challenge remains.

**Scale:**
- Fentanyl is the leading cause of death for Americans aged 18-45.
- CBP and postal inspectors intercept thousands of fentanyl shipments annually, but acknowledge that the vast majority of illicit imports go undetected.
- Precursor chemicals (often shipped as misidentified industrial chemicals) are also smuggled through the parcel stream.

**Consequences:**
- Public health crisis with hundreds of thousands of deaths.
- Political pressure on CBP and postal agencies to increase screening — which conflicts with trade facilitation objectives.
- Investment in detection technology (X-ray, spectroscopy, canine units) that is expensive and still insufficient for the volume.

### 6.3 Counterfeit and Unsafe Goods

**The Problem:** Cross-border e-commerce parcels frequently contain counterfeit goods (trademark-infringing products), pirated content (copyright-infringing goods), and products that do not meet domestic safety standards (unregulated electronics, cosmetics with banned ingredients, children's products with lead).

**Root Cause:** Low-value shipments have historically received minimal scrutiny. Foreign sellers face little accountability for selling counterfeit or unsafe goods. Consumers cannot easily distinguish genuine from counterfeit products when purchasing online.

**Consequences:**
- Consumer safety risk from products that have not been tested or certified.
- Intellectual property rights holders lose revenue and brand equity.
- Legitimate domestic manufacturers face unfair competition from counterfeit imports.
- Government agencies must balance enforcement against trade facilitation.

### 6.4 Interagency Coordination

**The Problem:** In the US, 49 federal agencies have authority over goods at the border. Coordinating requirements, data systems, and processing timelines across these agencies is a persistent challenge.

**Root Cause:** Each PGA has its own statutory mandate, data requirements, IT systems, and organizational priorities. While ACE serves as a partial single window, full interagency integration has not been achieved. PGAs often have independent processing timelines that are not synchronized with customs release.

**Consequences:**
- Shipments cleared by CBP may still be held for PGA review (FDA, USDA, CPSC, EPA).
- Duplicate data submission — traders may need to file information with both CBP and individual PGAs.
- Inconsistent processing times — FDA review may take days or weeks even for routine products.
- Policy conflicts — facilitation goals (speed goods through) can conflict with safety goals (hold goods for testing).

### 6.5 Technology Modernization Funding

**The Problem:** Government customs technology systems require significant investment to modernize, but funding is constrained by budget limitations, competing priorities, and the lengthy government procurement process.

**Root Cause:** Government IT modernization is notoriously difficult — complex procurement rules, legacy system dependencies, multi-year implementation timelines, and insufficient technical talent in government service. CBP's ACE system, while a major advancement, is already approaching end-of-life architecture.

**Scale:**
- ACE 2.0 (next-generation trade processing system) requires congressional authorization and multi-billion-dollar appropriation.
- The EU Customs Data Hub has a projected budget of EUR 2+ billion with a decade-long implementation timeline.
- Many developing countries' customs systems are even further behind, relying on 1990s-era technology or paper processes.

**Consequences:**
- Customs authorities cannot process the volume or complexity of modern trade with current systems.
- Innovation (AI risk assessment, real-time data analytics, automated classification) is constrained by legacy infrastructure.
- Trade facilitation improvements are delayed by years or decades.
- The gap between what technology can do and what customs systems actually do continues to widen.

---

## Cross-Cutting Pain Points

### Data Quality: The Universal Root Cause

Across nearly every stakeholder and pain point, **data quality** emerges as the single most pervasive root cause. Poor product descriptions, incorrect classifications, missing origin information, understated values, and absent PGA data create cascading problems:
- Carriers cannot file accurate entries.
- Brokers spend excessive time on data correction.
- Importers face unexpected duties and penalties.
- Customs authorities cannot effectively assess risk.
- Consumers experience delays and unexpected charges.

The solution to the data quality problem is not just technology — it requires capturing accurate, structured trade data **at the point of origin** (the seller, the purchase order, the product catalog) rather than trying to fix it downstream.

### Regulatory Complexity: The Compounding Challenge

Every new regulation, tariff program, and compliance mandate adds another layer of complexity to an already overburdened system. The challenge is not any single regulation — each one may be individually justified and manageable. The challenge is the **cumulative, compounding effect** of dozens of overlapping requirements, each with its own data demands, timelines, and consequences for non-compliance.

### The Volume-Complexity-Speed Trilemma

The customs clearance ecosystem faces a fundamental trilemma:
- **Volume** is surging (de minimis elimination, e-commerce growth).
- **Complexity** is increasing (stacking tariffs, new compliance mandates).
- **Speed** expectations are rising (same-day/next-day delivery norms).

Current systems and processes cannot simultaneously handle more volume, more complexity, and faster throughput. Something has to give — unless the fundamental approach to clearance is transformed. This trilemma is the ultimate forcing function for the clearance of the future.

---

*This document is part of the FedEx Global Clearance Knowledge Base. For related context, see:*
- *[01: Regulatory Landscape](01_regulatory_landscape.md)*
- *[03: Competitive Landscape](03_competitive_landscape.md)*
- *[04: Clearance Process Today](04_clearance_process_today.md)*
