# Can Customs Clearance Data Serve as Supply Chain Traceability Infrastructure?

> **FedEx Global Clearance Knowledge Base**
> Research Assessment: January 2026

---

## Executive Summary: The Honest Answer

**No, customs clearance data alone cannot serve as supply chain traceability infrastructure. But it is a surprisingly powerful starting point when combined with other data sources -- and several companies are proving this right now.**

Customs clearance events capture a rich snapshot of each border-crossing moment: what crossed, where it came from, who shipped it, who received it, what it's worth, and what it's made of. The problem is that these snapshots are **isolated by design**. Customs systems were built to assess duties and enforce trade law at a single point in time, not to connect the journey of materials through multi-tier manufacturing transformations across countries. Linking cotton (HS 5201) imported into Vietnam to fabric (HS 5208) exported from Vietnam to a garment (HS 6204) imported into the US requires inferring relationships that no single customs declaration captures.

That said, the companies that are doing this -- Altana AI most prominently -- have demonstrated that trade data, when combined with entity resolution, corporate registry data, and machine learning, can produce supply chain graphs that are directionally useful. They are not perfect. They are not proof-grade for UFLPA detention responses. But they are the best available approximation of multi-tier supply chain structure at global scale, and they are getting better.

This document assesses each question honestly: what works, what doesn't, and what FedEx should understand before betting on this approach.

---

## 1. What Data Is Captured at Each Customs Clearance Event?

A customs clearance event -- specifically, a formal import entry in the US (Entry Type 01 filed in ACE) -- captures the following data elements. This is the richest dataset available, and it's important to understand both its depth and its limits.

### Data Elements Captured

**Product Identification:**
- **HTSUS code** (10-digit in the US; 6-digit HS code harmonized globally under WCO). This identifies WHAT the product is at a fairly granular level. HS 5201.00 is "cotton, not carded or combed." HS 5208.11 is "woven fabrics of cotton, containing 85% or more by weight of cotton, weighing not more than 200 g/m2, unbleached, plain weave." HS 6204.62 is "women's trousers, breeches, etc., of cotton."
- **Product description** (free-text field, quality varies enormously -- from precise "100% cotton twill trousers, women's, style #4452" to useless "clothing").
- **Quantity and unit of measure** (e.g., 5,000 kg, 10,000 pieces, 200 dozen).

**Origin and Routing:**
- **Country of origin** (where the goods were manufactured or underwent "substantial transformation"). This is NOT necessarily where they were shipped from.
- **Country of export** (where the shipment physically departed).
- **Manufacturer ID** (MID -- a coded identifier for the manufacturer, required on US entries since 2009, format includes country code + manufacturer name + city).
- **Port of entry** (the US port/gateway where goods arrive).

**Parties:**
- **Importer of record** (IOR -- the US entity legally responsible for the import, identified by IRS number or CBP-assigned number).
- **Consignee** (the entity receiving the goods, often the same as the IOR).
- **Exporter/shipper** (the foreign entity that shipped the goods).
- **Manufacturer** (may differ from the exporter -- a trading company may export goods made by someone else).
- **Customs broker** (the licensed intermediary filing the entry).

**Financial:**
- **Transaction value** (declared customs value of the goods, in the transaction currency, converted to USD).
- **Applicable duties, taxes, and fees** (MFN duty, Section 301/232/IEEPA tariffs, AD/CVD, MPF, HMF).
- **Incoterms** (FOB, CIF, DDP, etc. -- determines which costs are included in the declared value).
- **Currency of transaction.**

**Compliance and Regulatory:**
- **FTA preference claims** (whether USMCA, KORUS, etc. was claimed, and the basis).
- **PGA flags** (FDA product codes, EPA TSCA certifications, CPSC data).
- **AD/CVD case numbers** (if applicable).
- **Section 301/232/IEEPA indicators.**
- **Bond information** (surety, bond amount).

**Logistics:**
- **Transport mode** (air, ocean, truck, rail).
- **Carrier SCAC code.**
- **Bill of lading / air waybill number.**
- **Container numbers (ocean) or tracking numbers (express).**
- **Estimated and actual arrival dates.**

### What's Rich About This Data

This is a LOT of data. For every single border crossing, you have: the product at HS-code granularity, the origin country, the specific manufacturer (MID), the importer, the value, and the quantity. Multiply this across millions of entries per day in the US alone, and across all major trading nations, and you have an extraordinary dataset describing global trade flows.

### What's Missing -- The Critical Gaps

**1. No upstream linkage.** The US entry for a cotton garment (HS 6204) from Vietnam says "manufactured in Vietnam by [MID]." It does NOT say: "This garment was made from fabric imported into Vietnam from China, which was woven from yarn spun in India, from cotton grown in Uzbekistan." The entry captures ONE hop in the supply chain -- the border crossing into the US. It says nothing about what happened before.

**2. No bill of materials.** The entry says "cotton trousers." It does NOT say: "made from 1.2 meters of cotton twill (HS 5208), 0.3 meters of polyester lining (HS 5407), one metal zipper (HS 9607), five plastic buttons (HS 9606), and thread (HS 5204)." The input materials are invisible.

**3. No transformation record.** The entry does not describe what manufacturing process occurred. It does not say "cut and sewn in Factory X in Ho Chi Minh City." The MID field gives you the manufacturer name and city, but nothing about what they did or what inputs they used.

**4. Manufacturer ID is imprecise.** The MID format (country code + first 3 characters of manufacturer name + first 3 characters of city + first 4 characters of address) creates collisions. Multiple factories in the same city with similar names can share a MID. And trading companies frequently appear as the manufacturer when they are actually intermediaries.

**5. Free-text descriptions are unreliable.** Product descriptions range from excellent to useless. "Cotton twill trousers, women's, 97% cotton 3% spandex, style 4452, color navy" is actionable. "Garments" is not. You cannot reliably infer material composition or product details from free-text descriptions across millions of entries.

**6. No inter-country linkage.** The US import entry exists in ACE. The Vietnamese export record exists in Vietnam Customs' system. The Chinese export of fabric to Vietnam exists in China's GACC system. These systems do not talk to each other. There is no global identifier that links the Chinese fabric export to the Vietnamese garment export to the US garment import.

---

## 2. Can Clearance Events Be Linked Across Tiers? The Cotton-to-Garment Example

### The Theoretical Chain

Let's trace the example: cotton raw material to finished garment.

| Tier | HS Code | Event | Origin | Destination | Parties |
|------|---------|-------|--------|-------------|---------|
| 4 (Raw Material) | 5201.00 | Cotton exported from Uzbekistan | UZ | CN | Uzbek cotton trader -> Chinese yarn spinner |
| 3 (Intermediate) | 5205.11 | Cotton yarn exported from China | CN | VN | Chinese yarn mill -> Vietnamese fabric weaver |
| 2 (Component) | 5208.12 | Cotton fabric exported from China/imported into Vietnam | CN | VN | Chinese/Vietnamese textile company |
| 1 (Finished Good) | 6204.62 | Cotton trousers exported from Vietnam | VN | US | Vietnamese garment factory -> US apparel importer |

### What You Can See from Clearance Data Alone

If you had access to the import/export records of ALL countries involved (a big if -- see Section 3), you could observe:

1. **US import record:** A US apparel company imported cotton trousers (HS 6204.62) from Vietnam, from manufacturer [Vietnamese Garment Factory MID], valued at $X.
2. **Vietnamese import record:** Vietnamese Garment Factory imported cotton fabric (HS 5208.12) from China, from [Chinese Textile Company], valued at $Y.
3. **Chinese import record:** Chinese Textile Company imported raw cotton (HS 5201) from Uzbekistan, from [Uzbek Cotton Trader], valued at $Z.

### What You Can Infer (Probabilistically, Not Definitively)

With the above records, you can build a **probabilistic** supply chain graph:

- US Apparel Importer <-- Vietnamese Garment Factory <-- Chinese Textile Company <-- Uzbek Cotton Trader
- The HS code progression (5201 -> 5208 -> 6204) is consistent with a cotton-to-fabric-to-garment transformation.
- The timing makes sense (cotton imported to China months before fabric exported; fabric imported to Vietnam weeks before garments exported).
- The quantities are plausible (X kg of cotton could produce Y meters of fabric could produce Z garments).

### What You Cannot Prove

**You cannot prove that THIS cotton became THAT fabric became THOSE trousers.**

- The Chinese textile company may import cotton from 15 different suppliers across 5 countries. The cotton is fungible -- bales are blended, spun together, and there is no physical traceability from a specific Uzbek cotton bale to a specific bolt of fabric.
- The Vietnamese garment factory may import fabric from 8 different mills. Which fabric went into which garment order is an internal manufacturing record, not a customs record.
- The time lag between imports and exports at each tier makes it impossible to link specific shipments without internal production records.

### The Fungibility Problem

This is the fundamental barrier. Raw materials and intermediate goods are **fungible** -- they are mixed, blended, and processed in ways that destroy the identity of the original input. Cotton from Xinjiang and cotton from Texas become indistinguishable once they are blended and spun into yarn. Polysilicon from different sources is mixed in the same crucible. Cobalt from different mines is refined together. Customs data can tell you what a factory imports and what it exports, but it cannot tell you which specific imports became which specific exports.

This is why UFLPA requires "clear and convincing evidence" and why that evidence must come from **the supply chain participants themselves** (bills of materials, lot tracking, manufacturing records, third-party audits) -- not from customs data.

### What Clearance Data CAN Do: Risk Assessment vs. Proof

Clearance data is excellent for **risk assessment** and poor for **proof**.

- **Risk assessment:** "This Vietnamese garment factory imports cotton fabric from a Chinese textile company that imports raw cotton from Uzbekistan. Uzbekistan has documented forced labor in cotton harvesting. This supply chain has UFLPA risk." This is valuable intelligence that can trigger further investigation.
- **Proof:** "This specific pair of cotton trousers contains NO cotton from Xinjiang." Customs data cannot establish this. Only internal supply chain records, batch-level traceability, and third-party audits can.

---

## 3. Technical Challenges of Linking Clearance Events Across Tiers

### Challenge 1: Different Importers/Exporters at Each Tier

At each tier of the supply chain, different companies are the importer and exporter. The US garment importer has no direct trade relationship with the Chinese cotton mill three tiers back. Connecting them requires:

- **Entity resolution:** Matching the "exporter" field on one country's records with the "importer" field on another country's records. This sounds simple but is not. Company names are rendered differently in different alphabets, transliterations, and legal naming conventions. "Dongguan Yuxing Textile Co., Ltd." on a Chinese export record might appear as "Yuxing Textile" or "DG Yuxing" or "Yu Xing Textiles" on a Vietnamese import record. Corporate subsidiaries, trading names, and holding company structures add further complexity.
- **Corporate ownership mapping:** A factory operating under one legal name may be owned by a parent company that operates dozens of factories under different names. Connecting the corporate family requires corporate registry data, beneficial ownership databases, and entity relationship mapping -- none of which comes from customs data.
- **Trading company obfuscation:** In many trade lanes (especially East Asian), trading companies (sogoshosha in Japan, similar intermediaries elsewhere) sit between the manufacturer and the importer. The customs record shows the trading company as the exporter, not the actual factory. This deliberately breaks the chain of visibility.

### Challenge 2: Different Countries and Data Access

Each country's customs data exists in that country's sovereign systems. Accessing it requires either:

- **Government-to-government data sharing agreements:** These exist (e.g., WCO Mutual Administrative Assistance agreements, bilateral customs cooperation treaties) but are primarily used for enforcement investigations, not bulk data access.
- **Commercial trade data providers:** Companies like ImportGenius, Panjiva (now part of S&P Global), Datamyne, Descartes, and others collect trade data from countries that make it publicly available. **Critically, not all countries make trade data available.** The US publishes import data (via the AMS/Census Bureau); many countries publish varying degrees of import/export data. But China, India, and several other major trading nations restrict access to their trade data or publish only aggregate statistics.
- **Bill of lading data:** For ocean shipments, bills of lading are filed with customs and some data providers collect them. This gives shipper-consignee pairs and goods descriptions for ocean freight. But express/air shipments have different documentation, and B/L data is less structured than customs entry data.

**The data access problem is the single biggest practical barrier.** If China does not make its export records available (and it does not, at the shipment level), you cannot directly observe what a Chinese textile company exports to Vietnam. You can observe what Vietnam imports from China (if Vietnam publishes import data -- it does to a limited extent), but the picture is always incomplete.

### Challenge 3: Manufacturing Transformation (Inputs Become Outputs with Different HS Codes)

This is the most intellectually challenging problem. When raw cotton (HS 5201) enters a Chinese yarn mill, the output is cotton yarn (HS 5205). The HS code changes. The product description changes. The physical form changes. The value changes (value is added through manufacturing).

Linking across transformations requires:

- **HS code adjacency knowledge:** Understanding which HS codes are inputs to which other HS codes. Cotton (52xx) is an input to cotton textiles (52xx-53xx), which are inputs to garments (61xx-62xx). This knowledge can be encoded (and companies like Altana have built "HS transformation models"), but it is probabilistic. A factory that imports HS 5201 cotton and HS 3920 plastic film could be making cotton bags (HS 4202) or automotive interiors (HS 9401) or dozens of other things.
- **Quantity/value reasonableness:** Does the volume of cotton imported by a factory roughly correspond to the volume of yarn it exports, given reasonable conversion ratios? This is a useful sanity check but not conclusive.
- **Temporal alignment:** Imports of raw materials should precede exports of finished goods by a plausible manufacturing lead time. Cotton imported in January, fabric exported in March, garments exported in June -- plausible. Cotton imported in January, garments exported the same week -- implausible.

### Challenge 4: Time Delays Between Tiers

Global supply chains operate on long timelines. Cotton harvested in September may be imported to a spinner in November, spun into yarn in December-January, exported to a weaver in February, woven into fabric in March-April, exported to a garment factory in May, cut and sewn in June-July, and exported to the US in August. The total cycle from raw material to finished good can be 6-12 months or longer.

This creates two problems:
1. **Inventory pooling:** Factories maintain inventories of inputs. A batch of fabric imported in February may be used in production that runs from March through August, mixed with fabric imported in January, March, and April. You cannot determine which specific fabric import corresponds to which specific garment export.
2. **Correlation degradation:** The longer the time gap between tiers, the weaker the statistical correlation between specific import and export events. With a 6-month lag and multiple input sources, the probabilistic linkage becomes very noisy.

---

## 4. Who Has Attempted to Build Supply Chain Graphs from Trade/Customs Data?

### Commercial Efforts

#### Altana AI -- The Most Ambitious Attempt

**What they are building:** Altana has built what they call the "Altana Atlas" -- a knowledge graph of the global supply chain, mapping relationships between companies, factories, products, and trade flows. As of 2025-2026, they claim to have mapped over 300 million entities and billions of relationships.

**Data sources:** Altana ingests:
- US and international customs/trade data (import and export records from countries that publish them)
- Corporate registry data (company formations, officers, ownership structures) from 200+ jurisdictions
- Shipping and logistics data (bills of lading, AIS vessel tracking)
- Sanctions and watchlists
- Open source intelligence (news, regulatory filings, social media)
- Satellite imagery (for factory and facility identification)
- Client-contributed data (companies that use Altana share their own supply chain data, which enriches the graph)

**Methodology:**
1. **Entity resolution at scale:** Altana's core technical achievement is resolving entity identities across data sources. Matching "Dongguan Yuxing Textile Co., Ltd." in a Chinese corporate registry to "YUXING TEXTILE" on a US import record to "Yu Xing Textiles Dongguan" on a Vietnamese bill of lading. This uses NLP, fuzzy matching, corporate ownership graphs, and machine learning.
2. **Trade flow analysis:** Once entities are resolved, Altana maps who-ships-to-whom at scale. If Company A in China ships goods (per B/L and customs data) to Company B in Vietnam, and Company B in Vietnam ships goods to Company C in the US, Altana infers a supply chain: A -> B -> C.
3. **HS code transformation modeling:** Altana builds models of which HS codes are inputs to which outputs. If Company A exports cotton yarn (5205) to Company B, and Company B exports cotton fabric (5208), Altana infers that A is a supplier of input materials to B.
4. **Risk scoring:** Altana layers compliance risk on top of the graph -- flagging entities in Xinjiang, entities on UFLPA/sanctions lists, entities with corporate ownership links to sanctioned parties, etc.

**Customers:** Altana's customers include US government agencies (CBP, Commerce Department), large importers, and financial institutions. CBP has publicly acknowledged using Altana-type supply chain intelligence for UFLPA enforcement targeting.

**Funding and validation:** Altana raised over $200M (Series B in 2023) at a $1B+ valuation. Investors included venture firms and strategic partners. The US government has awarded contracts for supply chain intelligence.

**Honest assessment of Altana's capabilities:**
- *Strengths:* Altana has the most comprehensive commercial attempt at mapping global supply chains from trade data. Their entity resolution technology appears genuinely advanced. The combination of trade data + corporate registries + shipping data produces supply chain graphs that are directionally correct for many relationships.
- *Limitations:*
  - **Tier depth is limited.** Altana can reliably map Tier 1 (direct suppliers to US importers) and Tier 2 (suppliers to Tier 1) with reasonable confidence. Beyond Tier 2, confidence drops rapidly because data becomes sparse (many countries don't publish granular trade data) and entity resolution becomes noisier.
  - **Fungibility is unsolved.** Altana can tell you that a Vietnamese garment factory sources fabric from Chinese mills that source cotton from Uzbekistan. Altana cannot tell you that a *specific* garment contains cotton from a *specific* Uzbek farm. The fungibility problem is inherent to the methodology.
  - **False positive rate is non-trivial.** Entity resolution at scale produces false matches. A company incorrectly linked to a Xinjiang entity could face unjustified UFLPA scrutiny. Altana acknowledges this and provides confidence scores, but users must understand the uncertainty.
  - **Data gaps in key countries.** China's trade data is not publicly available at the shipment level. Altana uses B/L data, satellite imagery, and corporate registries to compensate, but the picture of Chinese manufacturing is less complete than the picture of US importing.
  - **Not proof-grade for UFLPA.** Altana's intelligence is useful for risk assessment and targeting, but it is not sufficient as "clear and convincing evidence" for a UFLPA detention response. Importers still need their own supply chain documentation.

#### Panjiva / S&P Global Market Intelligence

**What they built:** Panjiva (acquired by S&P Global in 2018 for ~$160M) is a trade data analytics platform that aggregates customs and shipping data from 170+ countries.

**Methodology:** Primarily a data aggregation and search platform. Panjiva collects import/export records and bills of lading and makes them searchable by company, product, HS code, country, etc. Users can look up "who does Company X import from?" and "who does Company Y export to?" and manually build supply chain maps.

**Limitations:** Panjiva is a data access tool, not a supply chain graph. It does not perform automated entity resolution, HS code transformation modeling, or multi-tier supply chain inference. Users must do the analytical work themselves. It is a powerful research tool for trade analysts, not an automated supply chain mapping platform.

#### Everstream Analytics

**What they do:** Everstream is a supply chain risk management platform. Their focus is on identifying disruption risks (natural disasters, geopolitical events, supplier financial distress, compliance risks) for specific supply chains.

**Methodology:** Everstream combines:
- Client-provided supply chain data (supplier lists, tier-1 and tier-2 maps that the client already knows)
- External data sources (news, weather, financial data, sanctions lists)
- Some trade data (for enrichment and validation)

**Honest assessment:** Everstream does NOT primarily use trade data to discover unknown supply chain relationships. Their model is: the client tells them their supply chain, and Everstream monitors risks against it. This is valuable, but it is a fundamentally different approach from Altana's "discover the supply chain from trade data" model. Everstream's supply chain graphs are only as complete as what clients provide.

#### Resilinc

**What they do:** Resilinc is a supply chain risk management and mapping platform, primarily serving manufacturing and automotive industries.

**Methodology:** Resilinc's approach is **survey-based supply chain mapping:**
1. Resilinc sends structured surveys to Tier 1 suppliers asking: "Who are YOUR suppliers? Where are YOUR factories? What materials do you source and from where?"
2. Resilinc then sends surveys to Tier 2 suppliers (the suppliers' suppliers), and so on.
3. This data is assembled into a multi-tier supply chain map.
4. Risk monitoring (disruption events, compliance risks) is layered on top.

**Honest assessment:** Resilinc's approach produces the most accurate multi-tier supply chain maps available -- because the data comes directly from the supply chain participants. The limitation is that it depends entirely on supplier cooperation (which can be incomplete or inaccurate), it is slow (surveying through tiers takes months), and it does not scale to map all global supply chains. Resilinc covers specific client supply chains, not the entire world. Trade data plays a minor role -- primarily for validation.

#### Other Notable Efforts

- **Sourcemap:** A supply chain transparency platform focused on sustainability compliance. Uses a combination of client-provided data, third-party certifications, and some trade data to map supply chains. Strong in agricultural commodities (cocoa, coffee, cotton) where traceability standards exist.
- **Sayari:** A corporate intelligence and supply chain mapping platform. Uses corporate registry data, beneficial ownership data, and trade data to map entity relationships and identify hidden ownership/control structures. Particularly focused on sanctions evasion and illicit finance. Acquired Kharon (trade compliance data provider) to strengthen trade data capabilities.
- **Project44 / FourKites:** Primarily real-time visibility platforms for logistics, not supply chain mapping. They track where shipments are in transit, not who the upstream suppliers are.

### Academic Research

Academic literature on using trade data for supply chain mapping is limited but growing:

- **Network analysis of bilateral trade flows** (using UN Comtrade data) has been a productive area of economics research for decades. Researchers have mapped global trade networks at the country-product level, identifying which countries are central to which product supply chains. This work (by economists like Daron Acemoglu, Vasco Carvalho, and others) demonstrates that trade data can reveal supply chain structure at the aggregate level.
- **Firm-level trade network research** is newer and more relevant. Studies using firm-level customs data (available from some countries' customs authorities for research purposes) have mapped buyer-seller networks within individual countries. Research by Jonathan Eaton, Samuel Kortum, and others has used Colombian, French, and Chilean customs data to study firm-level trade networks.
- **The OECD Trade in Value Added (TiVA) project** uses input-output tables and trade data to estimate how much value each country adds at each stage of production. This is aggregate-level (country x sector), not firm-level, but it quantifies the multi-tier transformation chain at the macro level.
- **Limitations of academic work:** Most academic research uses aggregate or country-level trade data, not firm-level customs records. The firm-level work that exists is constrained to individual countries (where the researchers obtained customs data access) and does not attempt cross-country, multi-tier mapping. The computational and data access challenges are recognized as major barriers.

---

## 5. What Additional Data Beyond Clearance Is Needed?

### Must-Have Data Sources for Multi-Tier Traceability

| Data Source | What It Provides | Who Has It | Accessibility |
|---|---|---|---|
| **Bills of materials (BOMs)** | What inputs go into each product, in what quantities | Manufacturers | Private -- only available if the manufacturer shares it |
| **Manufacturing/production records** | Which input lots were used in which production runs | Manufacturers | Private -- rarely shared outside the company |
| **Supplier declarations** | Statements from suppliers about their own upstream sources | Tier 1+ suppliers | Available if requested; reliability varies |
| **Third-party audit reports** | Independent verification of factory conditions, material sources | Audit firms (SGS, Bureau Veritas, Intertek) | Available to the client who commissioned the audit |
| **Corporate ownership/registry data** | Who owns which companies, corporate structures, beneficial owners | Government registries, commercial databases (Orbis, Dun & Bradstreet) | Semi-public; aggregators charge for access |
| **Shipping/logistics data** | Container tracking, bills of lading, routing | Carriers, freight forwarders, port authorities | Partially public (B/L data); partially private |
| **Satellite imagery** | Physical existence and activity of factories, farms, mines | Commercial satellite providers (Planet, Maxar) | Commercial; available for purchase |
| **Product certifications** | GOTS (organic cotton), Fairtrade, FSC (wood), RSPO (palm oil) | Certification bodies | Semi-public; varies by certification |
| **Blockchain/DLT traceability** | Immutable record of chain-of-custody transfers | Traceability platforms (TextileGenesis, Provenance) | Only for participants who opted in |
| **Financial transaction data** | Payment flows between buyer and supplier at each tier | Banks, payment processors | Highly restricted (banking secrecy, privacy law) |

### The Hierarchy of Evidence for UFLPA Compliance

This is illustrative of what "traceability" actually requires in practice:

**Tier 1 (Sufficient for UFLPA "clear and convincing evidence"):**
- Complete supply chain map from raw material to finished good
- Bills of material for each tier showing inputs and outputs
- Production records showing which inputs were used in which production runs
- Third-party audits of working conditions at each tier
- Shipping records linking specific shipments between tiers
- Certificates of origin and composition from independent testing labs

**Tier 2 (Useful for risk assessment, not sufficient for UFLPA proof):**
- Trade data showing buyer-seller relationships at each tier
- Corporate ownership maps showing which entities are related
- HS code transformation analysis showing plausible input-output chains
- Quantity and timing analysis showing plausible material flows
- Satellite imagery confirming factory existence and activity

**Tier 3 (Useful for targeting/flagging, not sufficient for anything else):**
- Country-level trade flow statistics
- Industry-level input-output analysis
- News and open-source intelligence about specific companies or regions

Customs clearance data, even at its best, provides **Tier 2** evidence. Getting to **Tier 1** requires data from the supply chain participants themselves.

---

## 6. How Do the Supply Chain Mapping Companies Actually Work?

### Altana AI: Deep Dive

**Architecture:** Altana builds a knowledge graph where nodes are entities (companies, factories, government agencies, individuals) and edges are relationships (ships-to, owns, located-at, manufactures). The graph is constructed from multiple data sources and continuously updated.

**Entity resolution pipeline:**
1. Ingest raw records (customs entries, B/Ls, corporate filings, etc.)
2. Extract entity mentions (company names, addresses, identifiers)
3. Normalize entity mentions (transliterate, standardize naming conventions)
4. Cluster entity mentions that likely refer to the same real-world entity
5. Merge clusters into canonical entity records
6. Link entities to known databases (sanctions lists, UFLPA Entity List, corporate registries)

**Supply chain inference:**
1. For each entity, aggregate all observed trade relationships (who they ship to, who they receive from)
2. For each trade relationship, classify the type (raw material supply, component supply, finished good trade, etc.) based on HS codes
3. Build multi-tier chains by connecting entity A's exports to entity B's imports to entity C's exports
4. Score confidence based on: data quality, number of confirming records, temporal consistency, quantity plausibility

**How successful is it?** Altana is genuinely useful for:
- Identifying that a US importer's Tier 1 supplier sources from entities with Xinjiang exposure (UFLPA screening)
- Mapping the general structure of a supply chain (who are the likely Tier 2 and Tier 3 suppliers)
- Identifying hidden corporate relationships (a sanctioned entity's subsidiary operating under a different name)
- Providing starting points for deeper investigation

Altana is NOT sufficient for:
- Proving the specific provenance of a specific product (the fungibility problem)
- Providing legally defensible evidence for UFLPA detention responses
- Identifying all suppliers with certainty (false negatives are inevitable when data is incomplete)

### Everstream and Resilinc: The Client-Data Model

These companies take a fundamentally different approach. Instead of inferring supply chains from external data, they ask clients and their suppliers to provide the data directly.

**Advantages of this approach:**
- Higher accuracy (data comes from the source)
- Can capture details that external data cannot (BOMs, production processes, sub-tier relationships)
- Produces maps that can be used as evidence (not just risk indicators)

**Disadvantages:**
- Slow (surveying takes months; supplier cooperation is inconsistent)
- Incomplete (suppliers may not disclose all their sources; sub-tier visibility drops off rapidly)
- Not scalable to the entire global economy (you can only map supply chains where you have client relationships)
- Subject to gaming (suppliers may provide inaccurate data to hide risks)

### The Hybrid Approach (What's Emerging)

The most sophisticated practitioners are combining both approaches:
1. Use trade data and external intelligence to build a **baseline supply chain graph** (Altana-style)
2. Use supplier surveys and client-provided data to **validate and enrich** the graph (Resilinc-style)
3. Use ongoing trade data monitoring to **detect changes** (new suppliers appearing, existing suppliers changing sourcing patterns)
4. Use certification and audit data to **verify compliance** at specific nodes in the graph

This hybrid approach is where the industry is heading. No single data source is sufficient; the power is in the combination.

---

## 7. What Role Does FedEx (or Any Logistics Provider) Play?

### What FedEx Sees

FedEx, as an express carrier and customs broker, has visibility into specific segments of the supply chain:

**What FedEx has:**
- US import entry data for every shipment FedEx clears (product, HS code, value, origin, parties)
- Advance shipping data from shippers (product descriptions, values, destinations)
- Global shipment tracking data (who shipped what to whom, when, from where, to where)
- Historical patterns (which importers buy from which exporters, how frequently, what products, what values)
- Shipper and consignee identity data (name, address, contact information)
- The customs declaration data for every country where FedEx clears goods

**What FedEx does NOT have:**
- The upstream supply chain behind each shipment (who made the components, where the raw materials came from)
- Bills of materials or manufacturing records
- Supplier-supplier relationships (unless both tiers ship via FedEx, which is unlikely for all tiers)
- Visibility into ocean freight (most raw material and intermediate goods move by ocean, not express)
- Visibility into domestic logistics within manufacturing countries (factory-to-factory movements don't cross borders)

### Can FedEx Connect the Dots?

**Partially, but with significant limitations.**

FedEx sees the **last mile of the supply chain** -- the border-crossing event where goods enter the destination country. For express shipments, FedEx has rich data about that event. But FedEx does NOT see:
- How the product was made
- What it was made from
- Where the inputs came from before they arrived at the factory
- The multi-tier supply chain upstream of the shipper

FedEx's data is, at best, the **Tier 1 view** -- the direct importer-exporter relationship. To get Tier 2 and beyond, FedEx would need to combine its data with other sources (which is exactly what Altana does, using trade data from multiple carriers and data providers).

**There is one scenario where FedEx has unusually deep visibility:** When FedEx handles shipments at MULTIPLE tiers of the same supply chain. For example, if a Chinese fabric mill ships fabric to Vietnam via FedEx, and then the Vietnamese garment factory ships garments to the US via FedEx, FedEx has data on both border crossings. In theory, FedEx could link these. In practice:
- This is rare. Different tiers of a supply chain typically use different logistics providers (ocean for bulk raw materials, express for finished goods).
- Even when both tiers use FedEx, the shipments are filed by different parties with different accounts -- there is no automated linkage between "fabric shipped from China" and "garments shipped from Vietnam."
- FedEx does not systematically analyze its own data across tiers in this way today.

### FedEx's Realistic Role in Supply Chain Traceability

Based on this assessment, FedEx's role in supply chain traceability should be:

1. **Rich Tier 1 data provider:** FedEx has excellent data about the direct importer-exporter relationship for every shipment it handles. This is valuable as an input to supply chain mapping (by Altana or similar platforms) and for FedEx's own risk assessment.

2. **Risk assessment platform (not proof platform):** FedEx can score supply chain risk based on its clearance data (country of origin, manufacturer ID, product category, shipper history) and third-party supply chain intelligence (Altana, Sourcemap, etc.). This is the approach the vision document already describes (Outcome 3, continuous risk scoring).

3. **Evidence facilitation (not evidence generation):** FedEx can provide a platform for importers to store, manage, and deploy their own supply chain evidence (audit reports, BOMs, certificates of origin) -- making it faster to respond to UFLPA detentions or other compliance inquiries. FedEx facilitates; the importer provides the substance.

4. **Data enrichment partner:** FedEx could contribute its clearance data (in anonymized/aggregated form) to third-party supply chain mapping platforms (like Altana) in exchange for enriched supply chain intelligence. This is a data partnership, not an in-house capability build.

---

## 8. Realistic Assessment: What Works and What Doesn't

### What Works

| Capability | Feasibility | Notes |
|---|---|---|
| Mapping Tier 1 supplier relationships from clearance data | **High** | This works reliably. Customs data directly shows who ships to whom. |
| Inferring Tier 2 relationships from combined multi-country trade data | **Medium** | Works when data is available for both countries. Entity resolution is the main challenge. |
| Inferring Tier 3+ relationships | **Low-Medium** | Data becomes sparse, entity resolution noisier, and confidence drops significantly. |
| Building HS code transformation chains (cotton -> fabric -> garment) | **Medium** | Conceptually sound; practically limited by fungibility and data gaps. |
| Risk screening (flagging supply chains with exposure to high-risk regions/entities) | **High** | This is the primary proven use case. Altana, Sayari, and others do this effectively. |
| Providing proof-grade traceability for UFLPA compliance | **Low** | Not achievable from trade data alone. Requires supply chain participant data. |
| Real-time supply chain monitoring (detecting supplier changes, new sourcing patterns) | **Medium-High** | Trade data is published with delay (weeks to months), so "real-time" is approximate. But trend detection works. |
| Linking specific products to specific raw material sources | **Very Low** | The fungibility problem makes this essentially impossible from external data. |

### What Doesn't Work

1. **Trade data as a substitute for supply chain due diligence.** It cannot be. Trade data provides intelligence and risk indicators, not evidence and proof. Any system that claims to provide "full supply chain traceability" from trade data alone is overselling.

2. **Automated multi-tier mapping without human validation.** Automated supply chain graphs from trade data contain significant noise (false relationships, missing relationships, misidentified entities). Human analysts must validate key relationships. The automation speeds up the work; it does not eliminate it.

3. **Cross-country linkage without data access agreements.** If you cannot access China's customs data at the shipment level, you cannot observe what Chinese companies export. You can infer from what other countries import from China, but you have a blind spot inside China's borders.

4. **Real-time traceability.** Customs data is published with significant lag (US data: weeks; many countries: months or never). "Real-time" supply chain monitoring from trade data is actually "recent past" monitoring.

5. **Traceability for commodities.** For fungible commodities (cotton, minerals, chemicals, bulk agricultural products), lot-level traceability from trade data is essentially impossible. Physical tracing (isotope analysis, DNA marking, blockchain-based chain of custody) is required.

---

## 9. Implications for FedEx's Clearance Vision

### What This Means for the Vision Document

The vision document (Outcome 3, Compliance as a Competitive Advantage) already takes a largely correct position. It states:

> "FedEx has shipment data (what shipped, from where, when). We do not have supply chain data (which factory, which raw material source, which tier-3 supplier)."

This is accurate. The vision then correctly describes an "ecosystemic" approach:
- Importer-managed supply chain profiles
- Third-party supply chain intelligence (Altana, Sourcemap)
- Continuous risk scoring

This is the right framework. The research in this document supports and extends it.

### Specific Recommendations

1. **Partner with Altana (or similar) rather than building supply chain mapping in-house.** Supply chain graph construction from trade data is Altana's core competency, built over years with $200M+ in investment. FedEx should consume this intelligence, not try to replicate it. FedEx's value-add is in the last-mile clearance data and the importer relationship, not in entity resolution and graph construction.

2. **Position FedEx as the "evidence locker," not the "evidence factory."** Importers need a platform to store, organize, and rapidly deploy their supply chain documentation when a UFLPA detention hits. FedEx, as the party most directly affected by detentions (goods sitting in FedEx's cage), has strong incentive to facilitate this. Build the platform for importers to maintain their compliance evidence, linked to their clearance data.

3. **Use clearance data for risk scoring, not traceability claims.** FedEx can legitimately say: "Our system scores every shipment for UFLPA risk based on country of origin, manufacturer, product category, and third-party supply chain intelligence." FedEx should NOT say: "Our system traces your supply chain from raw material to finished good." The first claim is supportable; the second is not, from clearance data.

4. **Contribute data to the ecosystem.** FedEx's clearance data -- in aggregated, anonymized form -- is valuable to supply chain mapping platforms. Consider data-sharing partnerships where FedEx provides trade flow data and receives enriched supply chain intelligence in return.

5. **Watch the regulatory evolution.** The EU Customs Data Hub (2028-2032), 21CCF in the US, and various mutual recognition agreements may eventually create the cross-country data linkages that are currently missing. If customs authorities share data with each other (and with trusted private parties), the feasibility of multi-tier traceability from trade data improves dramatically. FedEx should be at the table as these systems are designed.

6. **Be honest with customers.** When selling compliance-as-a-service, be transparent about what trade data can and cannot do. Risk assessment: yes. Full traceability: no (unless the importer provides their own supply chain data). Overselling creates legal liability and reputational risk when a UFLPA detention cannot be resolved because the "traceability" was actually just statistical inference.

---

## 10. Summary: The Bottom Line

**Can customs clearance data serve as supply chain traceability infrastructure?**

- **As a foundation for risk assessment and Tier 1 mapping: Yes.** This works today. Companies are doing it. FedEx should leverage it.
- **As a foundation for multi-tier supply chain graphs: Partially.** It works for Tiers 1-2 with reasonable confidence when data is available. Beyond Tier 2, it becomes speculative. Companies like Altana have made impressive progress, but the results are probabilistic, not deterministic.
- **As a substitute for actual supply chain traceability (BOMs, production records, audits): No.** Not today, not in the foreseeable future. The fungibility problem, the data access problem, and the transformation problem are structural barriers that better algorithms alone cannot overcome.

The right strategy for FedEx is to use clearance data for what it's good at (risk scoring, Tier 1 visibility, pattern detection), partner with specialists for deeper supply chain intelligence (Altana, Sourcemap), build the platform for importers to manage their own evidence, and be honest about the boundaries.

Supply chain traceability is not a single-source problem. It's an ecosystem problem. FedEx has a valuable piece of the ecosystem. The mistake would be believing it's the whole picture.

---

*This document is part of the FedEx Global Clearance Knowledge Base. For related context, see:*
- *[01: Regulatory Landscape](01_regulatory_landscape.md) -- UFLPA, CBAM, EUDR requirements driving traceability demand*
- *[02: Pain Points Exhaustive](02_pain_points_exhaustive.md) -- Section 3.6: UFLPA Supply Chain Mapping*
- *[03: Competitive Landscape](03_competitive_landscape.md) -- Altana AI valuation and competitive positioning*
- *[04: Clearance Process Today](04_clearance_process_today.md) -- Data elements captured in the clearance process*
- *[Vision: Clearance of the Future](../vision/clearance_of_the_future_vision.md) -- Outcome 3: Compliance as a Competitive Advantage*
