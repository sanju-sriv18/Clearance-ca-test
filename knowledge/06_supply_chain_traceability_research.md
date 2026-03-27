# Multi-Tier Supply Chain Visibility and Traceability: Research & Analysis

> **FedEx Global Clearance Knowledge Base**
> Research Compiled: January 2026

---

## Executive Summary

This document examines the current state of multi-tier supply chain visibility and traceability, the platforms and approaches that have been attempted, the specific problem of transformation tracking (how raw materials become finished goods), and the realistic potential for a customs/clearance platform to contribute to supply chain traceability. The analysis is intended to be critical and balanced -- identifying genuine opportunities alongside real obstacles.

**The core finding:** Multi-tier supply chain visibility remains an unsolved problem at scale. Most companies can see their tier-1 suppliers but have limited or no visibility beyond that. No single platform has achieved comprehensive, real-time, multi-tier traceability across industries. The platforms that have succeeded are narrowly scoped (single commodity, single industry, strong regulatory mandate). A customs/clearance platform has a genuine but limited role to play -- it captures border-crossing events with structured data, but it cannot observe intra-country manufacturing, material transformations, or the supplier relationships that define a supply chain. Bridging this gap requires partnerships, not platform-building alone.

---

## 1. The Current State of Multi-Tier Supply Chain Visibility

### How Far Can Companies Actually See?

The data on supply chain visibility is remarkably consistent across research sources, and it paints a sobering picture:

**Tier-1 visibility (direct suppliers):**
- Most large enterprises have reasonable visibility into their tier-1 suppliers -- the companies they contract with directly. Surveys from McKinsey, Deloitte, and Gartner consistently find that **65-80% of large companies** have "good" or "comprehensive" visibility into their tier-1 suppliers.
- Even at tier 1, "visibility" often means knowing the supplier's name, location, and general capabilities -- not having real-time data on their production processes, sub-suppliers, or labor practices.

**Tier-2 visibility (suppliers' suppliers):**
- Visibility drops dramatically at tier 2. Research from MIT Center for Transportation & Logistics and multiple industry studies finds that only **20-35% of companies** have meaningful visibility into their tier-2 suppliers.
- For most companies, tier-2 visibility is partial: they may know some tier-2 suppliers for their highest-risk products but have no systematic mapping.

**Tier-3 and beyond (raw material sources):**
- Visibility into tier 3 and deeper is rare. Only **5-10% of companies** claim meaningful visibility beyond tier 2, and even those claims are typically limited to specific product lines under regulatory scrutiny.
- For most supply chains, the raw material origin is effectively unknown. A brand may know the factory that assembled their product (tier 1) and possibly the fabric mill that wove the fabric (tier 2), but not the cotton farm (tier 3) or the gin that processed the raw cotton (tier 2.5).

**The McKinsey finding that recurs:** McKinsey's 2022 supply chain survey found that only **2% of surveyed companies** had visibility beyond tier 2 across their full supply chain. This statistic has been widely cited and has held up in subsequent research. The number may have improved slightly to 3-5% by 2025 as UFLPA and EUDR forced investment in supply chain mapping for specific product categories, but the structural problem remains.

**The COVID-era reality check:** The pandemic exposed how little companies knew about their own supply chains. Companies discovered they were dependent on single-source suppliers they did not know existed. The Texas winter storm of 2021 and the Suez Canal blockage revealed hidden concentration risks that would have been visible with tier-2+ mapping. Post-COVID, investment in supply chain mapping surged -- but most of that investment has been directed at risk assessment (identifying which suppliers are in which geographies) rather than true traceability (tracking the physical flow of materials through transformations).

### Why Is Multi-Tier Visibility So Hard?

The difficulty is not primarily technological. It is structural:

**1. Supplier reluctance to share data.**
Tier-2 and tier-3 suppliers have no direct commercial relationship with the brand or end buyer. They sell to their customer (the tier-1 supplier), and their customer relationships, pricing, and capacity are trade secrets. Asking a tier-3 cotton farmer to share production data with a brand three tiers away is asking them to expose their business to a party they have no relationship with and no incentive to help.

**2. Competitive dynamics at every tier.**
A tier-1 factory often sources from multiple tier-2 mills and considers its sourcing strategy a competitive advantage. Revealing its supplier network to the brand (its customer) creates the risk that the brand will go direct, cutting out the tier-1. This incentive to conceal, not reveal, supply chain relationships is present at every tier.

**3. The data doesn't exist in structured form.**
At tiers 2-4, suppliers in developing economies may keep records in paper ledgers, WhatsApp messages, or not at all. The digital infrastructure for structured data capture is absent. Even where it exists, data formats are incompatible across systems, languages, and jurisdictions.

**4. Supply chains are dynamic, not static.**
A tier-1 supplier may switch between three tier-2 fabric mills based on price, capacity, and delivery time -- sometimes order by order. A "supply chain map" captured in January may be obsolete by March. Traceability requires not a point-in-time snapshot but a continuous, transaction-level data flow.

**5. The transformation problem.**
Raw materials change form as they move through the supply chain. Cotton becomes yarn becomes fabric becomes a shirt. Ore becomes steel becomes an auto part. At each transformation step, traceability requires linking the output to the input -- which requires the manufacturer at each stage to maintain and share production records. This is the hardest problem in traceability and is addressed in detail in Section 3.

---

## 2. Platforms and Approaches for Multi-Tier Traceability

### 2.1 Blockchain-Based Approaches

Blockchain was the dominant traceability narrative from approximately 2017-2022. The premise: a shared, immutable ledger that all supply chain participants write to, creating a trusted record of provenance that cannot be tampered with. The reality has been more nuanced.

#### TradeLens (Maersk + IBM) -- Failed

- **Launched:** 2018. **Shut down:** November 2022.
- **Scope:** Blockchain-based supply chain platform for ocean shipping. Not specifically a traceability platform, but aimed to create a shared record of trade events (booking, shipping, port handling, customs) across the supply chain.
- **Peak adoption:** 150+ organizations, 5 of the top 6 ocean carriers, hundreds of ports.
- **Why it failed (detailed analysis):**
  1. **Governance problem.** TradeLens was perceived as Maersk's platform, despite efforts to create independent governance. Competitors were uncomfortable sharing operational data on a platform controlled by their largest rival. The joint venture with IBM did not alleviate this concern -- IBM was the technology partner, but Maersk was the domain partner, and Maersk's strategic interests were inseparable from the platform.
  2. **Insufficient value per participant.** TradeLens provided visibility benefits, but these were incremental over existing EDI and API solutions. Customs authorities already had their own systems. Small forwarders could not justify the integration cost. Banks were interested in the trade finance use case, but it was never fully developed.
  3. **Technology overhead without commensurate benefit.** Blockchain added latency, complexity, and governance overhead compared to centralized cloud platforms. The "trustless" property of blockchain (you do not need to trust any single party) was unnecessary in a context where participants already had commercial relationships and contracts. A well-designed API platform with access controls would have delivered 90% of the value at 10% of the complexity.
  4. **The network effect trap.** TradeLens needed everyone to join to be valuable to anyone. But no one wanted to join until it was already valuable. This is the classic two-sided marketplace problem, and it was exacerbated by the governance concern.
  5. **Revenue model never materialized.** TradeLens never established a sustainable revenue model. Transaction fees were not firmly established, and the platform burned cash while seeking adoption.

**The lesson:** Blockchain is not a traceability solution. It is a data architecture choice. The traceability problem is fundamentally about data collection (getting participants to contribute data) and data linkage (connecting records across tiers), not about data immutability. TradeLens solved the wrong problem.

#### IBM Food Trust -- Survived (Narrowly)

- **Launched:** 2018, built on Hyperledger Fabric.
- **Scope:** Food traceability -- tracking food products from farm through processor, distributor, and retailer to consumer.
- **Anchor customer:** Walmart. In 2018, Walmart mandated that all leafy green suppliers use IBM Food Trust to provide end-to-end traceability. This was a critical adoption lever -- Walmart's buying power forced participation.
- **Why it survived where TradeLens failed:**
  1. **Single dominant buyer as adoption driver.** Walmart's mandate solved the network effect problem. Suppliers did not need to see the value proposition themselves -- Walmart told them to participate or lose their largest customer.
  2. **Clear, regulatory-aligned use case.** The 2018 romaine lettuce E. coli outbreak created a public health crisis that demanded faster traceability. FDA's FSMA Section 204 subsequently mandated traceability for high-risk foods. The regulatory tailwind was real and urgent.
  3. **Narrow scope.** IBM Food Trust focused on one industry (food) and initially on one product category (leafy greens). Narrowness enabled focus and iterability.
  4. **Genuine problem solved.** Before Food Trust, tracing a contaminated lettuce batch from store shelf back to the specific farm took **7 days**. With Food Trust, it took **2.2 seconds**. This was a dramatic, measurable improvement with clear public health value.
- **Caveats on "survival":**
  - Adoption outside Walmart's supply chain has been limited. Other retailers have not adopted at the same scale.
  - IBM has been restructuring its blockchain business. IBM Food Trust has been repositioned under IBM Consulting's supply chain practice rather than as a standalone blockchain product.
  - The platform increasingly emphasizes its data management and traceability capabilities rather than the blockchain substrate -- a tacit acknowledgment that the value is in the data, not the chain.
  - Competitors (FoodLogiQ/Controlant, Ripe.io, sTraceability) have gained traction with cloud-native approaches that do not use blockchain.

#### De Beers Tracr -- Narrow Success in a Narrow Domain

- **Launched:** 2018 (pilot); 2022 (production scale).
- **Scope:** Diamond traceability from mine to retail.
- **How it works:** Each rough diamond is registered on Tracr at the point of extraction, with a unique digital ID. As the diamond moves through cutting, polishing, grading, and sale, each handler records the transformation on Tracr. The final retail diamond has a provable chain of custody back to the specific mine.
- **Why it works:**
  1. **High-value, individually identifiable items.** Each diamond is unique and can be physically identified (via laser inscription and grading characteristics). The unit economics of per-item traceability work for diamonds. They do not work for commodity goods like cotton or palm oil.
  2. **Concentrated industry.** De Beers controls roughly 30% of global rough diamond production. A single company's mandate can move the industry.
  3. **Clear consumer demand.** Post-Blood Diamond (film) and Kimberley Process, consumers want provenance proof. Retailers want to differentiate on ethical sourcing.
  4. **Simple supply chain.** A diamond's supply chain is relatively linear: mine -> sorter -> cutter/polisher -> grader -> retailer. Compared to a t-shirt (cotton farm -> gin -> spinner -> weaver -> dyer -> cutter -> sewer -> finisher -> brand -> retailer), the diamond supply chain has fewer tiers and fewer transformation steps.

#### Key Blockchain Traceability Insight

The pattern across blockchain traceability projects is clear: **they succeed when a single powerful actor mandates adoption and the item being tracked is individually identifiable and high-value.** They fail when adoption is voluntary, the item is a fungible commodity, or the supply chain is long and complex.

This has direct implications for the clearance/traceability question: customs clearance data is generated at border crossings, which are mandatory (like Walmart's mandate), but the goods being tracked are often fungible commodities (like cotton), and the supply chain between border crossings is invisible to the clearance platform.

### 2.2 Network-Based Approaches

A newer generation of platforms has taken a fundamentally different approach: instead of asking supply chain participants to contribute data to a shared ledger, they infer supply chain relationships from public and semi-public data sources.

#### Altana AI

- **Founded:** 2020. **Funding:** $200M+ raised (Series B, 2023).
- **Approach:** Altana builds a "global supply chain graph" by ingesting and cross-referencing billions of records from:
  - **Bills of lading** (shipping records that show who shipped what to whom). The US requires public disclosure of import bills of lading, which Altana ingests at scale.
  - **Customs records** (where available, including US import data from CBP via FOIA and commercial data aggregators).
  - **Corporate registration databases** (ownership, officers, addresses).
  - **Sanctions and enforcement lists** (UFLPA Entity List, SDN list, etc.).
  - **Satellite imagery** (identifying factory locations, monitoring production activity).
  - **News and public reporting** (media coverage of forced labor, environmental violations, etc.).
- **What it produces:** An entity-resolution platform that can answer questions like: "What factories supply this brand?" "Is this factory connected to entities on the UFLPA Entity List?" "What is the ownership structure of this supplier?"
- **Strengths:**
  - Does not require supplier cooperation. Data is inferred from public records, not volunteered.
  - Scales across industries without industry-specific integration.
  - Useful for risk assessment and compliance screening (UFLPA, sanctions) -- exactly the use case UFLPA requires.
  - Used by US government agencies (CBP, Commerce Department) for enforcement.
- **Limitations:**
  - **Inference, not observation.** Altana infers supply chain relationships from trade records. It does not observe physical production. A bill of lading shows that Company A shipped fabric to Company B, but it does not show that the fabric was made from cotton from Region X. The transformation chain is invisible.
  - **US-centric data.** Bills of lading are publicly available for US imports. For other countries, equivalent data is less available or not public.
  - **Entity resolution is imperfect.** Matching company names across languages, jurisdictions, and transliterations produces false positives and false negatives.
  - **Point-in-time, not continuous.** Bill of lading data reflects what was shipped, not what is being produced now. It is backward-looking.
  - **Cannot prove a negative.** Altana can flag that a supply chain has connections to Xinjiang-linked entities, but it cannot prove that a specific shipment was NOT produced with forced labor. UFLPA requires "clear and convincing evidence" of the negative -- Altana provides risk assessment, not proof.

#### Sourcemap

- **Founded:** 2011. **Focus:** Supply chain mapping and due diligence.
- **Approach:** Combines bottom-up supplier surveys (companies invite their suppliers to map their own suppliers) with top-down data analytics (trade data, facility databases, risk indicators).
- **Strengths:** Good at structured supply chain mapping for companies that invest in the process. Used by major brands for UFLPA, EUDR, and conflict minerals compliance.
- **Limitations:** Depends on supplier willingness to participate. Coverage is only as good as the survey response rate. Most useful for deep-dive mapping of specific product lines, not comprehensive supply chain visibility.

#### Transparency-One (now part of LSEG)

- **Founded:** 2012. Acquired by London Stock Exchange Group (LSEG) in 2019.
- **Approach:** Multi-tier supply chain mapping through supplier onboarding and data collection. Focuses on compliance use cases (modern slavery, environmental due diligence).
- **Strengths:** Integration with LSEG's data assets (corporate registries, financial data). Strong in EU regulatory compliance context.
- **Limitations:** Same adoption dependency as Sourcemap. The brand must actively drive its suppliers to participate, and suppliers must drive their suppliers, and so on down the chain. Participation decay at each tier is severe -- if 70% of tier-1 suppliers participate, and 50% of their tier-2 suppliers participate, you have visibility into only 35% of tier-2.

### 2.3 Industry-Specific Approaches

The most successful traceability systems are not general-purpose platforms. They are industry-specific systems built for specific commodities under specific regulatory mandates.

#### Responsible Minerals Initiative (RMI) -- Conflict Minerals

- **Context:** Section 1502 of the Dodd-Frank Act (2010) required US-listed companies to determine whether their products contain "conflict minerals" (tin, tantalum, tungsten, gold -- the "3TG") originating from the Democratic Republic of Congo or adjoining countries, and if so, to conduct due diligence.
- **Mechanism:** RMI developed the **Conflict Minerals Reporting Template (CMRT)** -- a standardized Excel-based survey that companies send down their supply chains to trace mineral origins. RMI also operates the **Responsible Minerals Assurance Process (RMAP)**, which conducts independent third-party audits of smelters and refiners (the chokepoint in the mineral supply chain).
- **Why it works:**
  1. **Regulatory mandate** (Dodd-Frank 1502) compelled large companies to act.
  2. **Smelters/refiners are a natural chokepoint.** There are only ~350 tin, tantalum, tungsten, and gold smelters/refiners globally. Auditing this chokepoint is feasible. Tracing the mineral before the smelter (artisanal mining, trading, aggregation) is much harder and is handled through chain-of-custody programs at the mine and trading level with varying effectiveness.
  3. **Industry collaboration.** RMI has 400+ member companies that collectively demand RMAP compliance from smelters. No single company could compel a smelter to submit to audit. The coalition's collective buying power does.
- **What it actually traces:** RMI traces minerals to the smelter/refiner level. It can say "this smelter is RMAP-compliant" or "this smelter is not." It cannot, in most cases, trace the mineral back to the specific mine. This is "smelter-level assurance," not mine-to-product traceability.
- **Relevance to customs/clearance:** The conflict minerals model shows that chokepoint-based assurance (audit the critical transformation node) is more feasible than end-to-end traceability. Customs entry points are another form of chokepoint -- but they are at the country border, not at the transformation stage.

#### Better Cotton Initiative (BCI)

- **Context:** BCI (now "Better Cotton") is the world's largest cotton sustainability program, accounting for ~25% of global cotton production.
- **Traceability model:** **Mass balance** (see Section 3 for full explanation). Better Cotton does not guarantee that the physical cotton in a specific garment was grown under BCI standards. Instead, it tracks volume: for every unit of Better Cotton a brand claims, an equivalent volume of Better Cotton was produced somewhere in the supply chain. The physical cotton may or may not be the same cotton.
- **Why mass balance, not identity preservation:** Cotton is a fungible commodity that is ginned, baled, traded, blended, spun, and woven through a complex, multi-tier supply chain. Maintaining physical segregation of "certified" cotton from "conventional" cotton through every step is prohibitively expensive and operationally impractical. Mass balance allows the economic incentive (premium for certified cotton) to flow through the supply chain without requiring physical segregation.
- **Limitations:** Mass balance does not tell you whether the cotton in your shirt is actually "better cotton." Critics argue this is greenwashing. Supporters argue it is pragmatic -- it creates economic incentives for sustainable production even if physical traceability is infeasible.
- **Relevance:** BCI illustrates the fundamental tension between traceability purity and operational feasibility. Any customs-based traceability system for commodity goods will face the same tension.

#### Roundtable on Sustainable Palm Oil (RSPO)

- **Context:** The RSPO certifies sustainable palm oil production. Palm oil is in ~50% of consumer packaged goods and is a major driver of tropical deforestation.
- **Traceability models:** RSPO uses a tiered system:
  1. **Identity Preserved (IP):** Physical segregation from a single certified mill. The oil in the product can be traced to a specific mill and its supply base. Most expensive and operationally demanding.
  2. **Segregated (SG):** Physical segregation of certified palm oil, but from multiple certified mills. Cannot trace to a specific mill, but guarantees all oil in the product is certified.
  3. **Mass Balance (MB):** Certified and non-certified palm oil may be mixed, but the volume of certified oil purchased by the buyer matches the volume of certified oil produced in the supply chain.
  4. **Book and Claim (Credits):** The buyer purchases certificates representing sustainable palm oil production without any physical link. Purely financial mechanism.
- **Adoption:** Mass balance and book-and-claim dominate because identity-preserved and segregated supply chains are expensive. Only ~20% of RSPO-certified palm oil trades through IP or SG chains.
- **EU Deforestation Regulation impact:** The EUDR requires geolocation data for the production plots of specific commodities (including palm oil). This is stricter than any RSPO chain of custody model -- it requires actual traceability to the plot, not just certification. EUDR is forcing a move from mass balance toward identity preservation for EU-bound supply chains, and the industry is struggling with the cost and complexity.

#### DSCSA (Drug Supply Chain Security Act) -- Pharmaceuticals

- **Context:** US law requiring end-to-end, interoperable, electronic traceability for pharmaceutical products by November 2024 (extended to November 2025 for full enforcement).
- **How it works:** Every prescription drug package is serialized with a unique identifier (National Drug Code + serial number + lot number + expiration date). At every change of custody -- manufacturer to wholesaler to dispenser (pharmacy) -- the transaction data (product identifier, transaction information, transaction history, transaction statement) must be exchanged electronically. The system enables verification of individual packages and tracing from the patient back to the manufacturer.
- **Why it works:**
  1. **Federal law with enforcement teeth.**
  2. **Individually serialized items.** Each drug package has a unique identifier (analogous to diamonds).
  3. **Concentrated distribution channel.** The pharma supply chain has relatively few wholesalers (McKesson, AmerisourceBergen, Cardinal Health control ~90% of US distribution).
  4. **Patient safety motivation.** Counterfeit drugs kill people. The incentive for traceability is existential, not just commercial.
- **Relevance:** DSCSA demonstrates that end-to-end traceability is achievable for serialized, high-value products moving through a concentrated, regulated supply chain. It is NOT a model for commodity goods (cotton, palm oil, minerals) where items are fungible and supply chains are diffuse.

### 2.4 Summary: What Works and What Doesn't

| Factor | Enables Traceability | Inhibits Traceability |
|---|---|---|
| **Regulatory mandate** | Strong (Dodd-Frank, FSMA, DSCSA, UFLPA, EUDR) | Voluntary programs struggle with adoption |
| **Product type** | Individually identifiable, serialized, high-value | Fungible commodities (cotton, palm oil, minerals) |
| **Supply chain structure** | Short, linear, concentrated chokepoints | Long, branching, many small actors |
| **Dominant buyer** | Single buyer can mandate participation (Walmart, De Beers) | Fragmented buyer base cannot compel participation |
| **Technology** | Necessary but not sufficient | Blockchain adds complexity without solving adoption |
| **Transformation steps** | Few and simple (diamond: cut/polish) | Many and complex (cotton: gin/spin/weave/dye/sew) |

---

## 3. The Transformation Tracking Problem

### The Core Challenge

The hardest problem in supply chain traceability is tracking transformations -- the process by which raw materials become components become finished products. This is fundamentally different from tracking the movement of a finished good from warehouse to warehouse (which logistics companies do well).

**Example: Cotton to T-Shirt**

```
Tier 5: Cotton farm (Uzbekistan, India, US, Brazil, etc.)
  |  Cotton is picked and baled
Tier 4: Cotton gin
  |  Cotton is cleaned, seeds removed, fiber compressed into bales
  |  -- Often, cotton from multiple farms is blended at this stage --
Tier 3: Spinning mill
  |  Cotton fiber is spun into yarn
  |  -- Yarn from multiple gins may be blended --
Tier 2: Weaving/knitting mill
  |  Yarn is woven or knitted into fabric
  |  -- Multiple yarn lots may be combined --
Tier 2: Dyeing/finishing facility
  |  Fabric is dyed, printed, treated with chemicals
Tier 1: Cut-and-sew factory (garment manufacturer)
  |  Fabric is cut and sewn into a t-shirt
  |  Trim (thread, labels, buttons) added from other supply chains
Tier 0: Brand/retailer
  |  T-shirt is labeled, packaged, shipped to market
```

At each transformation step:
1. The physical identity of the input material changes (cotton becomes yarn; yarn becomes fabric).
2. Materials from multiple sources may be blended (cotton from 5 farms blended into one yarn lot).
3. Waste and scrap is generated (fabric cutting waste, dyeing wash water).
4. The entity performing the transformation changes (different company, different country, different data system).

**To trace the t-shirt back to the cotton farm, you would need:**
- The garment factory to record which fabric lots were used for each t-shirt order.
- The dyeing facility to record which greige fabric lots were dyed in each batch.
- The weaving mill to record which yarn lots were used for each fabric lot.
- The spinning mill to record which cotton bales were used for each yarn lot.
- The gin to record which farm cotton was received from.
- Each of these records to be linked in a digital system that spans 5-6 entities across 3-4 countries.

This is what "transformation tracking" means. And it is why most traceability systems compromise on one of three approaches:

### Three Approaches to Transformation Traceability

#### Identity Preserved (IP)

**Definition:** The physical material is kept completely separate from all other material throughout the supply chain. The specific batch of raw material can be traced through every transformation step to the specific finished product.

**How it works:** Cotton from Farm A is ginned separately, spun separately, woven separately, dyed separately, and sewn into garments that are labeled "made from Farm A cotton." At no point is it mixed with cotton from any other source.

**Where it is used:**
- Organic food (USDA Organic requires identity preservation for organic certification).
- Specialty coffee (single-origin beans).
- RSPO Identity Preserved palm oil (highest tier).
- De Beers Tracr (diamonds are inherently identity-preserved because each is unique).

**Strengths:**
- Highest credibility. You can make definitive claims: "This product contains material from Source X."
- Required by some regulations and certifications.

**Weaknesses:**
- Extremely expensive. Every handling and transformation step must maintain physical segregation. Dedicated processing lines, separate storage, separate transport. Costs are typically 20-50% higher than unsegregated processing.
- Logistically complex. Requires coordination across the entire supply chain. A single mixing event destroys the identity preservation.
- Only feasible for premium products where the consumer or regulation pays the premium.
- Does not scale to commodity volumes.

**Relevance to customs:** Customs data can support identity preservation by documenting that a specific shipment crossed a specific border at a specific time. But customs cannot verify that the physical segregation was maintained during manufacturing in the origin country.

#### Segregation

**Definition:** Certified/traceable material is kept physically separate from non-certified material, but material from multiple certified sources may be mixed together. You know the product contains only certified material, but you cannot trace it to a specific source.

**How it works:** Cotton from any BCI-certified farm can be mixed at the gin, mixed at the spinning mill, etc., as long as it is never mixed with non-certified cotton. The finished garment can be labeled "contains 100% certified cotton" but cannot be labeled "contains cotton from Farm A."

**Where it is used:**
- RSPO Segregated palm oil.
- Some organic certification schemes.
- Fairtrade cocoa (physical segregation of Fairtrade cocoa from conventional).
- Responsible wool standard.

**Strengths:**
- Moderately expensive -- cheaper than identity preserved because you do not need single-source processing lines.
- Claims are credible: the product genuinely contains only certified material.
- Feasible for medium-volume supply chains with cooperative participants.

**Weaknesses:**
- Still requires physical segregation infrastructure at every stage.
- Breaks down when supply chains are long, complex, or involve many small actors.
- Does not satisfy regulations that require source-level traceability (EUDR requires plot-level geolocation, not just "certified").

**Relevance to customs:** Customs can document that a segregated shipment crossed the border with documentation claiming certification. But customs cannot verify the segregation claim -- that requires audits at the manufacturing/processing level.

#### Mass Balance

**Definition:** Certified and non-certified material may be physically mixed at any point in the supply chain, but the total volume of certified material purchased equals the total volume of certified claims made. It is an accounting system, not a physical traceability system.

**How it works:** A spinning mill buys 100 tons of BCI-certified cotton and 200 tons of conventional cotton. All 300 tons are processed through the same equipment, producing 300 tons of yarn. The mill can sell 100 tons of yarn as "BCI mass-balance" -- meaning the equivalent quantity of BCI cotton entered the system, even though the physical molecules in any given yarn lot may be mixed. The buyer of the "BCI" yarn may receive yarn that is physically 0%, 50%, or 100% BCI cotton -- the label refers to the accounting, not the physics.

**Where it is used:**
- BCI/Better Cotton (the dominant cotton sustainability program).
- RSPO Mass Balance palm oil.
- UTZ/Rainforest Alliance (cocoa, coffee, tea) -- transitioning toward mass balance.
- Many forestry certifications (FSC Mix).
- EU Renewable Energy Directive (mass balance for biofuels).

**Strengths:**
- Lowest cost. No physical segregation required. Existing processing infrastructure can be used without modification.
- Highest scalability. Can be applied to any commodity volume.
- Creates economic incentive for sustainable production without requiring physical segregation.
- Pragmatic: it acknowledges that physical traceability of fungible commodities through complex supply chains is often infeasible.

**Weaknesses:**
- Lowest credibility. The product may contain none of the certified material. Critics call this "greenwashing."
- Does NOT satisfy UFLPA. UFLPA requires proof that the specific goods entering the US were not produced with forced labor. A mass-balance certificate that says "the equivalent volume of non-forced-labor cotton was purchased somewhere" does not meet the "clear and convincing evidence" standard.
- Does NOT satisfy EUDR. EUDR requires geolocation of the specific production plot. Mass balance does not provide this.
- Increasingly under regulatory scrutiny as governments demand more rigorous traceability.

**Relevance to customs:** Mass balance is the model most compatible with customs data, because customs data is inherently about quantities and values, not physical molecule-tracking. But it is also the model least compatible with the regulatory direction of UFLPA and EUDR.

### Which Industries Have "Solved" Transformation Tracking?

Honest assessment: **no industry has fully solved transformation tracking for fungible commodities at scale.** What exists is a spectrum:

| Industry | Level of Traceability | How Achieved | Limitations |
|---|---|---|---|
| **Diamonds** | High (identity preserved) | Unique items, concentrated industry, physical identification | Only works because items are individually unique |
| **Pharmaceuticals** | High (serialized) | Federal law (DSCSA), serialized packages, concentrated distribution | Only works for high-value serialized items |
| **Conflict minerals** | Moderate (chokepoint) | Smelter-level auditing (350 smelters), Dodd-Frank mandate | Cannot trace from smelter back to mine in most cases |
| **Organic food** | Moderate (identity preserved for premium) | USDA regulation, audit-based certification, IP supply chains | Expensive, fraud is a known issue |
| **Sustainable palm oil** | Low-Moderate (mostly mass balance) | RSPO certification tiers, EUDR forcing improvement | IP/segregated = expensive; mass balance = weak claims |
| **Cotton** | Low (mass balance) | BCI mass balance is dominant model | Cannot trace cotton to farm for most supply chains |
| **Apparel** | Very Low | Minimal industry-wide systems despite pressure from UFLPA | Long, complex supply chains; many small actors; limited digital infrastructure |

The industries that come closest to "solving" it share common features: high-value products (diamonds, pharma), individually identifiable items (serialized drugs, unique diamonds), and concentrated chokepoints (smelters, wholesalers) where auditing is feasible.

For the commodities most relevant to UFLPA enforcement -- cotton, polysilicon, tomatoes -- transformation tracking remains a deep challenge.

---

## 4. Could a Customs/Clearance Platform Build Transformation Chains?

This is the critical question for the FedEx clearance vision. The analysis must be honest about both the opportunity and the limitations.

### The Argument FOR: Clearance as a Traceability Backbone

**1. Clearance is mandatory and captures structured data.**
Every international shipment must clear customs. Unlike voluntary supply chain mapping platforms (Sourcemap, Transparency-One), customs clearance captures data from 100% of international trade -- not just the companies that opt in. The data is structured (HS codes, values, origins, shipper/consignee names, quantities) and machine-readable. This is a universal dataset that no voluntary platform can match.

**2. Clearance happens at every border crossing.**
A supply chain that spans 4 countries creates 3+ border-crossing events (more with re-exports and transshipments). At each border crossing, customs captures who shipped it, what it is, where it is going, what it is worth, and what it is made of (via product descriptions and HS codes). Linking these border-crossing events creates a multi-country shipment trail.

**3. Bills of lading and customs data can be cross-referenced.**
Altana AI has already demonstrated that bills of lading (public) and customs records can be combined to infer supply chain relationships. A clearance platform with access to its own filing data (not just public records) has an even richer dataset for entity resolution and relationship inference.

**4. Regulatory mandate creates data collection leverage.**
As UFLPA, EUDR, and CBAM require increasingly detailed supply chain data at the border, the customs entry becomes the forcing function for traceability data collection. Importers must provide supply chain documentation to clear their goods. The clearance platform becomes the natural repository for this documentation.

**5. Clearance platforms already capture the most important transformation indicator: HS code changes.**
When cotton (HS 5201) enters a country and cotton fabric (HS 5208) is exported from that country, the HS code change indicates that a transformation occurred. When fabric (HS 5208) enters another country and garments (HS 6109) are exported, another transformation is indicated. HS code changes at borders, combined with entity resolution, can outline the transformation chain at a coarse level.

### The Argument AGAINST: The Gaps Are Structural

**1. Clearance does not capture intra-country manufacturing.**
The transformation from cotton to yarn to fabric to garment typically happens within a single country (e.g., China, Vietnam, Bangladesh). Customs only sees the goods when they cross a border. If cotton enters Vietnam as a bale and leaves Vietnam as a finished garment, customs sees the input and output but nothing in between. The 3-4 transformation steps that occurred within Vietnam are invisible.

**2. Different entities at each tier.**
The company that imports cotton bales (a trading company) is typically not the company that exports finished garments (a manufacturing/export company). There is no automatic link between the import entry for cotton and the export entry for garments. The customs system treats them as unrelated transactions by unrelated parties. Building the link requires inferring the connection -- which requires either the importer to voluntarily provide the linkage data or an AI to infer it from patterns.

**3. Fungible goods cannot be tracked through mixing.**
When a spinning mill receives cotton from 10 different sources and blends them into yarn, there is no way to determine which cotton molecules ended up in which yarn lot. Customs data cannot solve a problem that exists at the physical processing level.

**4. Data linkage across jurisdictions is technically and legally difficult.**
Linking a US import entry for garments to a Vietnam export entry for garments to a Vietnam import entry for cotton to an India export entry for cotton requires access to customs data from 4 countries, entity resolution across different naming conventions and languages, and legal authorization to cross-reference data across jurisdictions. Most countries treat customs data as sovereign and do not share it freely.

**5. Customs descriptions are often insufficient for transformation tracking.**
A customs entry for "cotton fabric" does not specify what cotton was used. A customs entry for "men's knit t-shirt" does not specify what fabric was used. The product descriptions in customs entries are designed for tariff classification, not supply chain traceability.

**6. The time dimension.**
A supply chain may have 3-6 months between raw material sourcing and finished good export. Cotton purchased in March may become garments exported in August. Matching the import entry for cotton (March) to the export entry for garments (August) requires tracking inventory and production over months -- data that customs does not capture.

### What Would Be Needed to Bridge the Gaps?

If a customs/clearance platform wanted to credibly contribute to transformation tracking, it would need to supplement its border-crossing data with:

**1. Importer-declared supply chain data.**
The most direct approach: require importers to declare, as part of their customs entry, who their suppliers are, what inputs were used, and where transformation occurred. This is essentially what UFLPA already requires (supply chain traceability documentation). The clearance platform would collect, validate, and store this data alongside the entry. The platform does not generate the traceability data -- the supply chain participants do. The platform makes it easier to submit, manage, and retrieve.

**2. Third-party supply chain intelligence integration.**
Partners like Altana AI, Sourcemap, and industry-specific programs (RMI, BCI, RSPO) have supply chain mapping data that the clearance platform does not. Integrating these data sources into the clearance workflow enables risk assessment and compliance screening without the clearance platform building supply chain visibility from scratch. This is the approach the FedEx vision document already describes.

**3. Cross-border data-sharing agreements.**
Mutual recognition agreements for customs data between jurisdictions would enable linking import and export entries across borders. The WCO has promoted this concept, and some bilateral arrangements exist (US-EU Customs Cooperation Agreement, APEC single window initiatives). But comprehensive, real-time, transaction-level data sharing between customs authorities remains aspirational.

**4. HS code transformation inference.**
AI models can be trained to identify plausible transformation chains from HS code changes, entity relationships, and trade flow patterns. For example: if Entity A in Vietnam imports HS 5201 (cotton) from India in March, and Entity B in Vietnam (same corporate group as Entity A) exports HS 6109 (t-shirts) to the US in August, an inference engine can hypothesize a transformation chain. This is probabilistic, not definitive -- but it is useful for risk scoring and enforcement targeting.

**5. Industry-specific traceability data standards.**
For specific commodities under regulatory mandates, the clearance platform could adopt industry-specific traceability data standards:
- **Cotton:** US Cotton Trust Protocol, BCI chain of custody documentation.
- **Minerals:** RMI CMRT (Conflict Minerals Reporting Template), EMRT (Extended Minerals Reporting Template).
- **Palm oil:** RSPO chain of custody certification.
- **Wood/Paper:** FSC or PEFC chain of custody.
- **Pharmaceuticals:** DSCSA serialized product data.

The clearance platform would not replace these systems but would integrate their outputs into the customs compliance workflow.

### Honest Assessment: What a Clearance Platform Can and Cannot Do

| Capability | Clearance Platform | What's Needed Beyond Clearance |
|---|---|---|
| Track goods crossing borders | Yes -- core function | N/A |
| Identify who shipped to whom | Yes -- shipper/consignee data | Entity resolution for corporate groups |
| Detect transformation (HS code changes) | Yes -- coarse level | Intra-country production records |
| Prove specific material origin | No -- not enough data | Supply chain documentation from participants |
| Track material through mixing/blending | No -- physically impossible from border data | Identity preservation or mass balance accounting at the processing level |
| Risk-score supply chains for UFLPA | Partially -- can flag suspicious patterns | Third-party intelligence (Altana, Sourcemap) |
| Store and retrieve compliance evidence | Yes -- document management is feasible | Importers must generate the evidence |
| Enable EUDR compliance | Partially -- can capture geolocation at entry | Geolocation must be generated at the farm level |

**The realistic role of a customs/clearance platform in traceability is as an integrator and enforcement point, not as the source of traceability data.** The platform sits at the chokepoint (the border) where it can require, collect, validate, and store traceability documentation that supply chain participants generate. It can integrate third-party intelligence to enrich risk assessment. It can use its own filing data to identify suspicious patterns and enforce compliance. But it cannot observe or verify what happens inside factories, farms, and processing facilities between border crossings.

This is not a negligible role. The border is the single most powerful enforcement point in any supply chain. No goods enter the market without clearing customs. A clearance platform that effectively uses this chokepoint for traceability enforcement -- collecting the right data, scoring risk accurately, flagging suspicious patterns, and enabling fast compliance responses -- adds enormous value to the traceability ecosystem even if it cannot trace a cotton boll from Xinjiang to a t-shirt on a shelf.

---

## 5. Precedents for Logistics + Consulting Firm Traceability Infrastructure

### Direct Precedents

**Maersk + IBM (TradeLens):** The most direct precedent -- a carrier (Maersk) partnering with a technology firm (IBM) to build supply chain infrastructure. As documented extensively elsewhere in this knowledge base and in Section 2.1 above, it failed. But the failure was due to governance and adoption issues, not the carrier+tech-firm model itself. TradeLens was trying to build a multi-party platform. A clearance platform that serves the carrier's own operations first (and expands outward) avoids TradeLens's core failure mode.

**Walmart + IBM (IBM Food Trust):** Not a logistics company, but a dominant supply chain participant (Walmart) partnering with a technology firm (IBM) to build traceability infrastructure. The critical lesson: Walmart's buying power mandated adoption. A clearance platform has a similar lever -- if you want your goods cleared, you provide the data.

**DHL + multiple partners (TAS platform):** DHL built its Trade Automation Services platform with consulting and technology partners. TAS includes supply chain compliance capabilities (restricted party screening, origin management, FTA qualification). DHL has not positioned TAS as a traceability platform per se, but it collects and manages supply chain compliance data as part of its clearance workflow. This is the closest analog to what the FedEx vision describes.

**CMA CGM + Google Cloud:** A carrier (CMA CGM) partnering with a technology firm (Google) for AI-powered logistics and supply chain optimization, including customs documentation. Not specifically a traceability play, but demonstrates the carrier+tech model for supply chain intelligence.

### Analogous Partnerships in Adjacent Domains

**Financial services: SWIFT + consulting firms for compliance infrastructure.**
SWIFT (Society for Worldwide Interbank Financial Telecommunication) operates the messaging infrastructure that enables international financial transactions. SWIFT has partnered with consulting firms (Accenture, Deloitte) to build compliance capabilities -- KYC (Know Your Customer) utilities, sanctions screening, transaction monitoring -- layered on top of the existing financial messaging infrastructure. The analogy is direct: SWIFT sits at the financial chokepoint the way customs sits at the trade chokepoint. SWIFT does not know why money is being sent or what it is buying, but it captures structured data about who is sending money to whom, in what amount, and from where. Compliance infrastructure is layered on top.

**Pharmaceutical: GS1 + technology partners for DSCSA traceability.**
GS1 (the standards organization that manages barcodes and data standards) partnered with technology providers to build the data standards and infrastructure for pharmaceutical traceability under DSCSA. GS1 provides the standard (serialization format, data exchange protocols). Technology providers (SAP, TraceLink, Antares Vision) build the platforms that implement the standard. Pharma companies implement the platforms. This is a multi-party, standards-based traceability infrastructure built through partnership -- not by a single company.

**Food safety: FDA + industry + technology (FSMA 204).**
FDA's FSMA Section 204 traceability rule requires food companies to maintain records that enable rapid tracing of certain high-risk foods. The implementation is being enabled by technology partnerships: GS1 standards, cloud platforms from companies like FoodLogiQ (now part of Controlant) and ripe.io, and industry bodies like the Produce Traceability Initiative. No single company built the traceability infrastructure. It emerged from regulatory mandate + industry standards + technology implementation.

### Key Insight for FedEx-Accenture

The successful pattern is not "logistics company builds traceability platform." It is:
1. **Regulatory mandate creates the demand** (UFLPA, EUDR, DSCSA, FSMA 204).
2. **Industry standards define the data model** (GS1, RMI CMRT, RSPO chain of custody).
3. **Technology partners build the platform** (consulting firms, SaaS providers).
4. **Chokepoint actors enforce compliance** (Walmart mandating suppliers, customs requiring documentation at the border).

A FedEx-Accenture clearance platform plays role #4 -- the chokepoint enforcer. It is most effective when combined with #1 (regulatory mandates are already in place), #2 (adopting existing industry data standards rather than inventing new ones), and #3 (Accenture's platform engineering capability).

The platform should not try to be the single source of traceability truth for global supply chains. It should be the gateway through which traceability data flows at the most critical moment -- when goods enter a regulated market.

---

## 6. Market Size and Growth Projections

### Supply Chain Traceability Market

**Total market (2024):** $15-20 billion globally, encompassing traceability software, consulting, auditing, certification, and data services.

**Growth projections:**
- **CAGR 2024-2030:** 12-18%, depending on the source and scope definition.
- **Projected market (2030):** $35-55 billion.
- **Primary growth drivers:**
  1. UFLPA enforcement (US) -- forcing investment in supply chain mapping for goods with China exposure.
  2. EU Deforestation Regulation (EUDR) -- requiring plot-level geolocation for 7 commodity groups.
  3. EU CBAM -- requiring verified carbon emissions data for carbon-intensive imports.
  4. EU CSDDD (Corporate Sustainability Due Diligence Directive) -- requiring human rights and environmental due diligence across value chains.
  5. Consumer demand for transparency (fashion, food, electronics).
  6. ESG/sustainability reporting requirements (CSRD in the EU, SEC climate disclosure rules).

**Segmentation:**
| Segment | Size (2024) | Growth Rate | Key Drivers |
|---|---|---|---|
| Supply chain mapping/visibility software | $3-5B | 15-20% CAGR | UFLPA, EUDR, ESG mandates |
| Third-party auditing and certification | $5-7B | 8-12% CAGR | RSPO, FSC, RMI, organic, Fairtrade |
| Trade compliance software (customs-adjacent) | $1.5-2.5B | 10-15% CAGR | Tariff complexity, de minimis elimination |
| Supply chain data/intelligence (Altana-type) | $1-2B | 25-35% CAGR | UFLPA enforcement, sanctions, risk management |
| Blockchain/DLT-based traceability | $0.5-1B | Declining from peak hype; stabilizing | Niche use cases (diamonds, pharma, luxury goods) |

### Overlap with Customs Technology Market

The supply chain traceability market overlaps significantly with the customs/trade technology market documented in the platform partnership research (document 05). The intersection -- traceability capabilities embedded in customs/clearance platforms -- is the specific opportunity space for the FedEx-Accenture vision.

**Customs-adjacent traceability opportunity:** $2-4 billion annually by 2028, growing at 15-20% CAGR. This includes:
- Supply chain compliance data management for UFLPA (collecting and storing importer-provided traceability documentation).
- EUDR geolocation data management (capturing and transmitting plot-level data with customs entries).
- CBAM emissions data management (linking verified carbon data to customs entries).
- Conflict minerals due diligence integration (CMRT data linked to customs filings).
- Origin certification and verification (FTA compliance, rules of origin documentation).

### Investment Activity

| Company/Event | Date | Amount/Valuation | Relevance |
|---|---|---|---|
| Altana AI Series B | 2023 | $200M+ raised | Supply chain intelligence for compliance |
| Sourcemap (private) | Ongoing | Multiple rounds | Supply chain mapping for due diligence |
| Everstream Analytics | 2022 | $24M Series B | Supply chain risk analytics |
| Inspectorio | 2022 | $46M Series C | Supply chain quality and compliance |
| TrusTrace | 2023 | $14M Series A | Fashion supply chain traceability |
| o9 Solutions | 2022 | $3.7B valuation | Supply chain planning with visibility |
| Project44 | 2022 | $2.4B valuation | Real-time supply chain visibility |
| FourKites | 2021 | $1.8B valuation | Supply chain visibility |

**Note on valuations:** Many of these valuations were set during the 2021-2022 funding peak. Post-2022 corrections have tempered some valuations, but the underlying markets continue to grow, driven by regulatory mandates rather than discretionary corporate spending.

---

## 7. Critical Assessment: The Genuine Opportunity and Real Obstacles

### The Genuine Opportunity

1. **Regulatory mandates are creating non-discretionary demand.** UFLPA, EUDR, CBAM, and CSDDD are not optional. Companies must invest in traceability or face import bans, fines, and reputational damage. This is not a "nice to have" market -- it is a compliance mandate market with regulatory teeth.

2. **The border is the ultimate enforcement chokepoint.** No goods enter a regulated market without clearing customs. A clearance platform that effectively uses this chokepoint -- collecting traceability data, screening against risk databases, flagging suspicious supply chains -- is positioned at the most powerful point in the compliance enforcement chain.

3. **The integration role is genuinely valuable.** No one system will provide end-to-end traceability. The value is in integration: connecting Altana's supply chain intelligence with RMI's smelter audits with the importer's supplier surveys with the customs entry data. A clearance platform that serves as the integration point -- where all this data converges at the moment of import -- creates significant value even without generating any of the data itself.

4. **The data asset compounds over time.** Every customs entry that includes supply chain documentation creates a record that future entries can reference. Over years, a clearance platform accumulates a supply chain knowledge base that becomes increasingly valuable for risk scoring, pattern detection, and compliance automation.

5. **First-mover advantage in a market transitioning from voluntary to mandatory.** The shift from voluntary CSR-driven traceability to mandatory regulatory traceability is still in its early stages. The company that builds the clearance-traceability integration first will set the standard.

### The Real Obstacles

1. **The clearance platform cannot generate the traceability data it needs.** The most important traceability data -- factory-level production records, material sourcing decisions, processing batch records -- is generated inside factories in developing countries. A clearance platform sitting at the border of developed countries has no mechanism to create or verify this data. It is dependent on supply chain participants to provide it, and on third parties (auditors, intelligence platforms) to validate it.

2. **"Garbage in, garbage out" is a real risk.** If importers provide fraudulent supply chain documentation to clear customs (and they do -- UFLPA enforcement has revealed fabricated supply chain maps), the clearance platform becomes a filing cabinet for false evidence, not a traceability system. Combating this requires cross-referencing importer-provided data against independent intelligence (Altana, satellite imagery, audit records) -- which adds cost and complexity.

3. **The transformation problem has no technology solution for fungible commodities.** No amount of customs data, blockchain, or AI can trace which cotton molecules ended up in which t-shirt after blending at a spinning mill. For commodities like cotton, palm oil, and polysilicon, the industry will continue to rely on mass balance (an accounting system) or chokepoint auditing (checking the processor, not the material). The clearance platform must accept this reality and design for it.

4. **Data sovereignty creates real architectural constraints.** Supply chain traceability data spans jurisdictions by definition. But data sovereignty laws (GDPR, China's PIPL, India's DPDP) restrict cross-border data flows. Building a federated traceability data architecture that complies with data sovereignty requirements across 50+ jurisdictions is a non-trivial engineering challenge -- as the FedEx vision document already acknowledges.

5. **The market is fragmented and no standard has won.** There is no single traceability data standard that spans industries. RMI has its CMRT, RSPO has its chain-of-custody standard, DSCSA has its serialization standard, EUDR will have its own data format. A clearance platform must either support all of these or bet on convergence that hasn't happened yet.

6. **The competitive landscape includes well-funded pure-play competitors.** Altana AI ($200M+ raised), Sourcemap, and others are building supply chain traceability platforms as their core business. They are focused, well-funded, and have head starts. A clearance platform entering this space as an adjacent offering competes against companies for whom traceability is existential.

7. **Adoption at the supplier level remains the fundamental bottleneck.** Every traceability system depends on data from the supply chain's lowest tiers -- the farms, mines, and small factories in developing countries. These actors have the least digital infrastructure, the least incentive, and the least capacity to provide data. This bottleneck is not solved by building a better platform at the border. It requires investment at the source -- in digital infrastructure, farmer training, and economic incentives for participation.

### Strategic Recommendation

The FedEx-Accenture clearance platform should position itself as **the integration and enforcement layer for supply chain traceability, not the traceability system itself**. Specifically:

1. **Be the best data collector at the border.** Design the customs entry workflow to collect traceability data (supply chain documentation, certification numbers, geolocation data, emissions declarations) alongside traditional customs data. Make it easy for importers to provide this data and painful not to.

2. **Integrate, don't build.** Partner with Altana AI (supply chain intelligence), Sourcemap (supply chain mapping), industry certification bodies (RMI, RSPO, BCI), and audit firms (rather than building competing capabilities). The platform's value is in integration -- connecting these data sources at the point of customs entry.

3. **Use the chokepoint wisely.** The border crossing is the most powerful enforcement point in the supply chain. Use it to require data, score risk, and enforce compliance. Importers who provide rich traceability data get faster clearance (the incentive architecture already in the vision). Importers who do not get scrutiny, delays, and potential detention.

4. **Build the data asset over time.** Every customs entry with traceability data creates a record that enriches future risk scoring. Over years, the platform accumulates a supply chain knowledge base that is genuinely unique -- because no one else has customs entry data combined with supply chain documentation at this scale.

5. **Be honest about what the platform cannot do.** It cannot trace cotton to a specific farm (unless the importer provides that data and it is independently verified). It cannot prove goods were not made with forced labor (it can only facilitate the evidence assembly). It cannot observe what happens inside factories. Overselling the platform's traceability capabilities would be both dishonest and commercially risky.

---

*This document is part of the FedEx Global Clearance Knowledge Base. For related context, see:*
- *[01: Regulatory Landscape](01_regulatory_landscape.md) -- UFLPA, EUDR, CBAM requirements driving traceability demand*
- *[02: Pain Points Exhaustive](02_pain_points_exhaustive.md) -- Supply chain mapping burden on importers*
- *[03: Competitive Landscape](03_competitive_landscape.md) -- Altana AI, Sourcemap, and other traceability competitors*
- *[04: Clearance Process Today](04_clearance_process_today.md) -- How customs data is captured and processed*
- *[05: Platform Partnership Research](05_platform_partnership_research.md) -- TradeLens failure analysis, partnership models*
- *[Vision: Clearance of the Future](../vision/clearance_of_the_future_vision.md)*
