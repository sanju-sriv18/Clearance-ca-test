# Strategic Analysis: FDX Identifier as a CBP Global Business Identifier (GBI)

**Date:** 2026-03-06
**Status:** Strategic Exploration
**Classification:** Internal — Not for External Distribution

---

## Table of Contents

1. [Thesis](#1-thesis)
2. [The GBI Landscape Today](#2-the-gbi-landscape-today)
3. [What It Takes to Be an IMC](#3-what-it-takes-to-be-an-imc)
4. [FedEx Dataworks' Unique Data Position](#4-fedex-dataworks-unique-data-position)
5. [The FDX Identifier Concept](#5-the-fdx-identifier-concept)
6. [Why CBP Would Want This](#6-why-cbp-would-want-this)
7. [The Altana Precedent](#7-the-altana-precedent)
8. [Challenges and Risks](#8-challenges-and-risks)
9. [Alternative Path: D&B Partnership Leverage](#9-alternative-path-db-partnership-leverage)
10. [Competitive Positioning](#10-competitive-positioning)
11. [Recommended Next Steps](#11-recommended-next-steps)

---

## 1. Thesis

FedEx Dataworks (distinct from FedEx the carrier) could create a proprietary **FDX Identifier** and apply to CBP as the 5th Identity Management Company (IMC) under the Global Business Identifier (GBI) program. FedEx Dataworks possesses a data asset that no existing IMC has: **proof of real commercial movement** across 220+ countries and ~15 million packages per day. This positions an FDX Identifier not as another registry-based ID, but as a **movement-verified business identity** — a fundamentally different and more valuable signal for customs enforcement.

---

## 2. The GBI Landscape Today

### Program Status

- **Active** through February 23, 2027 (subject to extension)
- ACE has accepted GBI data since December 2022
- CBP is evaluating whether to **mandate GBI as a MID replacement**
- Multiple Federal Register Notices have expanded the program (latest: August 2025)

### Current Identity Management Companies (IMCs)

| IMC | Identifier | Core Asset | Established |
|-----|-----------|-----------|------------|
| **Dun & Bradstreet** | DUNS (9-digit) | Business credit/registration database, 500M+ entities worldwide | Original IMC (2022) |
| **GS1** | GLN (13-digit) | Global product/location standards, barcode infrastructure | Original IMC (2022) |
| **GLEIF** | LEI (20-char, ISO 17442) | Legal entity registry, originated from financial regulation (post-2008 crisis) | Original IMC (2022) |
| **Altana Technologies** | Altana ID (proprietary) | AI-powered supply chain knowledge graph, Product Passports | Added August 2025 |

### What CBP Gets From IMCs

Per the August 2025 Federal Register Notice, CBP accesses the underlying entity data associated with each identifier. Data elements include:

- Entity identifier numbers
- Official business titles and names
- Addresses
- Financial data
- Trade names
- Payment history
- Economic status
- Executive names

### Applicable Supply Chain Parties

GBI identifiers are submitted for: **manufacturer, shipper, seller, exporter, distributor, and packager** — the full cast of actors behind imported goods.

---

## 3. What It Takes to Be an IMC

Based on the existing IMC agreements and Federal Register notices, an IMC must:

| Requirement | Description |
|------------|-------------|
| **Unique identifier issuance** | Issue a globally unique, verifiable identifier per business entity |
| **Entity registry** | Maintain an authoritative database of entity records behind each identifier |
| **Data sharing agreement** | Enter into a formal agreement with CBP granting access to underlying entity data |
| **API/data access** | Provide CBP with a mechanism to resolve identifiers to entity records |
| **Supply chain party coverage** | Cover the party types CBP cares about (manufacturers, shippers, sellers, exporters, distributors, packagers) |
| **Data quality and maintenance** | Keep entity records current; handle entity changes (mergers, closures, name changes) |
| **Verifiability** | Identifiers must be traceable to a real, verifiable business entity |

The process to become an IMC is initiated through CBP's GBI inbox (GBI@cbp.dhs.gov) and involves negotiation of a data-sharing agreement. CBP has stated it "establishes a process for other IMCs to support CBP in the test."

---

## 4. FedEx Dataworks' Unique Data Position

### The Data Nobody Else Has

FedEx Dataworks sits on a data asset that is fundamentally different from every existing IMC:

| Data Dimension | FedEx Dataworks | D&B | GS1 | GLEIF | Altana |
|---------------|----------------|-----|-----|-------|--------|
| **Entity registration** | Implicit (shipper/consignee records) | Primary asset | Primary asset | Primary asset | AI-inferred |
| **Physical movement proof** | ~15M packages/day, 220+ countries | None | None | None | None |
| **Commercial relationship mapping** | Who ships to whom, how often, what volumes | Credit relationships | Product/location links | Legal ownership chains | AI-modeled from public data |
| **Real-time activity** | Live network telemetry | Quarterly updates | Static registry | Annual renewal | Periodic model refresh |
| **Geographic granularity** | Origin/destination addresses per shipment | Business address | Location codes | Registered address | Varies |
| **Product-level data** | Declared contents, weight, dimensions per package | None | Product codes (GTIN) | None | AI-inferred |
| **Express/e-commerce** | Dominant position | Weak coverage | Weak coverage | None | Growing |

### Key Insight

D&B, GS1, and GLEIF know entities exist **on paper**. Altana infers supply chain relationships from **public data + AI**. FedEx Dataworks knows entities are **actually moving goods** — and can prove it with shipment records.

### Existing Partnerships That Amplify This

**D&B + FedEx Dataworks (February 2026):**
- Joint "Retail Momentum Index" combining shipping intelligence with business data
- Already linking FedEx movement data to D&B entity records
- Creates a natural bridge: D&B entity identity + FedEx movement verification

**ServiceNow + FedEx Dataworks (October 2025):**
- Supply chain workflow automation with agentic AI
- Real-time intelligence into supply chain performance
- Source-to-pay operations integration (Q1 2026)

**fdx Commerce Platform:**
- End-to-end commerce data network
- Connects demand, fulfillment, logistics, and returns
- Entity relationships implicit in every transaction

---

## 5. The FDX Identifier Concept

### Definition

An **FDX Identifier** would be a unique, persistent identifier assigned to a business entity resolved from FedEx Dataworks' shipping and commerce data. Unlike registry-based identifiers (DUNS, GLN, LEI), it would be **movement-verified** — backed by actual commercial activity observed across the FedEx network.

### Proposed Characteristics

| Attribute | Specification |
|-----------|--------------|
| **Format** | Alphanumeric, globally unique (e.g., `FDX-` prefix + entity hash) |
| **Issuance basis** | Entity resolution from shipper/consignee records across FedEx network |
| **Verification** | Cross-referenced against shipment history — entities must show real commercial activity |
| **Enrichment** | Volume patterns, trade lanes, commodity types, relationship graph |
| **Persistence** | Stable identifier that survives address changes (entity resolution, not address-based) |
| **Coverage** | Global — any entity that ships or receives through FedEx network |

### Entity Resolution Approach

FedEx Dataworks would need to build (or may already have) an entity resolution system that:

1. Clusters shipper/consignee records across millions of shipments into distinct entities
2. Handles name variations, address changes, transliterations (Chinese/Arabic business names)
3. Resolves subsidiaries, DBAs, and trade names to parent entities
4. Assigns a persistent FDX Identifier to each resolved entity
5. Maintains a confidence score for each resolution

### Data Behind the Identifier

For each FDX Identifier, the entity record could include:

- Official business name(s) and trade names
- Address(es) — registered and observed shipping locations
- Activity profile — shipment volume, frequency, trade lanes, commodity categories
- Relationship graph — who this entity ships to/from and how often
- First seen / last seen dates on the FedEx network
- Risk signals — sudden volume changes, new trade lanes, pattern anomalies

---

## 6. Why CBP Would Want This

### Alignment with CBP Priorities

| CBP Priority | FDX Identifier Value |
|-------------|---------------------|
| **Section 321 / de minimis enforcement** | FedEx handles massive express/e-commerce volume — exactly where de minimis abuse concentrates. FDX Identifiers on express shipments would give CBP entity-level visibility into the highest-risk segment |
| **UFLPA / forced labor** | Movement-verified supply chain mapping: "this manufacturer in Xinjiang ships to this distributor in Vietnam who ships to this importer in LA" — derived from actual package flow, not AI inference |
| **Fentanyl interdiction** | Pattern detection on small parcel shipments from high-risk origins — entity identity tied to real shipping behavior |
| **MID replacement** | MID is a constructed code (country + name + city) that's easy to manipulate. FDX Identifier is resolved from actual shipping records — harder to spoof |
| **Trusted trade facilitation** | Entities with years of clean FedEx shipping history can be fast-tracked. Movement data is a natural trust signal |

### The Express Gap

Current IMCs (D&B, GS1, GLEIF) have **weak coverage in express/e-commerce** — the exact segment CBP is most concerned about. Small Chinese sellers shipping via FedEx Express often lack DUNS numbers, GLN codes, or LEI registrations. But they have FedEx shipping records. An FDX Identifier fills a coverage gap that matters to CBP enforcement.

---

## 7. The Altana Precedent

Altana's acceptance as the 4th IMC in August 2025 is the critical precedent. It establishes that CBP is open to:

- **Non-traditional identifiers** — Altana ID is proprietary, not an industry standard like DUNS/GLN/LEI
- **AI-powered entity resolution** — Altana builds its knowledge graph from public data + AI, not from entity self-registration
- **Supply chain graph products** — Altana Product Passports (tracking raw materials to final product) were specifically selected by CBP for next-generation traceability
- **Technology companies as IMCs** — not just standards bodies or data bureaus

### How FDX Identifier Would Differ From Altana

| Dimension | Altana ID | FDX Identifier |
|-----------|----------|---------------|
| **Data source** | Public records, corporate filings, trade data, news, AI inference | Actual shipping records (15M packages/day) |
| **Entity verification** | AI confidence scoring | Movement-verified (entity demonstrably ships goods) |
| **Supply chain mapping** | Inferred from public data | Observed from actual shipment origin/destination pairs |
| **Product traceability** | Product Passports (AI-modeled provenance) | Package-level tracking with declared contents |
| **Real-time capability** | Periodic model refresh | Live network telemetry |
| **Coverage** | Broader (not limited to one carrier's network) | Deeper within FedEx network; weaker outside it |

The key differentiator: Altana **infers** supply chain relationships. FedEx Dataworks **observes** them.

---

## 8. Challenges and Risks

### Carrier Neutrality

**Risk:** An FDX Identifier only covers entities in the FedEx network. Entities that exclusively use UPS, DHL, or ocean freight would be invisible.

**Mitigations:**
- D&B partnership extends coverage beyond FedEx-only entities
- Express/e-commerce is the highest-priority segment for CBP anyway
- GBI program already accepts multiple identifiers per entity — FDX Identifier would complement, not replace, DUNS/GLN/LEI
- FedEx Trade Networks handles ocean and ground freight too, expanding beyond express

### Conflict of Interest

**Risk:** FedEx is simultaneously a carrier, a customs broker (FedEx Trade Networks), and a proposed IMC. CBP might question the independence of an identifier issued by a commercially interested party.

**Mitigations:**
- FedEx Dataworks is a separate business unit from FedEx carrier operations and FedEx Trade Networks
- Altana also has commercial interests (they sell to importers who want GBI compliance) — precedent exists
- D&B sells commercial products to the same companies it identifies — same dynamic
- Structural separation: FDX Identifier issuance could be governed by an independent data governance board

### Customer Privacy and Competition

**Risk:** Sharing FedEx shipping data with CBP could raise concerns from FedEx customers about competitive intelligence leakage. Importers might worry that their shipping patterns (suppliers, volumes, trade lanes) become visible to competitors through CBP data.

**Mitigations:**
- IMC agreements with CBP are structured to protect commercial confidentiality
- Entity-level data (name, address, identifier) is already public or semi-public
- Aggregated movement patterns shared with CBP need not expose individual shipment details
- Opt-in model: importers choose to submit FDX Identifiers (GBI is voluntary during the test phase)

### Data Quality and Entity Resolution

**Risk:** Shipper/consignee records on packages are notoriously messy — misspellings, abbreviations, inconsistent formatting across countries. Entity resolution at FedEx's scale is a hard problem.

**Mitigations:**
- FedEx Dataworks already processes this data for their commerce platform and D&B partnership
- Entity resolution is a solved problem at this scale (D&B, Altana, and others do it commercially)
- FedEx has the advantage of seeing the **same entity repeatedly** across thousands of shipments — pattern-based resolution is more reliable than single-record matching

### Regulatory and Legal

**Risk:** Sharing logistics data with a government agency has regulatory implications in multiple jurisdictions (GDPR, data sovereignty, cross-border data transfer).

**Mitigations:**
- GBI data sharing is limited to entities involved in US import transactions
- Other IMCs (D&B, Altana) already share cross-border entity data with CBP
- FedEx already complies with CBP's Advance Electronic Information requirements (manifest data)

---

## 9. Alternative Path: D&B Partnership Leverage

Rather than becoming a standalone IMC, FedEx Dataworks could pursue a **faster path** through the existing D&B partnership:

### The Joint Identifier Concept

- D&B is already a GBI IMC (DUNS number)
- FedEx Dataworks + D&B announced a strategic collaboration in February 2026
- They could create an **enhanced DUNS** — a DUNS number enriched with FedEx movement verification

### How It Would Work

1. Entity starts with a standard DUNS number (D&B registry)
2. FedEx Dataworks matches the DUNS entity to shipping records in the FedEx network
3. A **movement verification score** is appended to the DUNS record
4. CBP receives the enhanced data: "DUNS 123456789 — verified active shipper, 2,400 shipments in past 12 months, primary trade lanes: Shenzhen → Los Angeles"

### Advantages of This Path

- No need to become a separate IMC (D&B is already approved)
- Faster time to market — leverage existing D&B infrastructure and CBP relationship
- Broader coverage — D&B's 500M+ entity database with FedEx movement enrichment
- Lower regulatory risk — D&B handles the CBP data-sharing agreement

### Disadvantage

- FedEx Dataworks doesn't own the identifier — D&B does
- Less strategic control and brand differentiation
- Revenue sharing with D&B rather than capturing the full IMC opportunity

---

## 10. Competitive Positioning

### If FedEx Dataworks Becomes an IMC

| Positioning | Impact |
|------------|--------|
| **Regulatory technology player** | Transcends carrier identity — becomes a customs technology infrastructure provider |
| **Section 321 enforcement partner** | Only IMC with deep express/e-commerce entity data where de minimis abuse concentrates |
| **Movement-verified trust** | "Proven shipper" signal that no other IMC can generate |
| **fdx platform extension** | GBI compliance as a value-added service on the fdx commerce platform — importers get FDX Identifiers as part of their fdx subscription |
| **Government partnership** | Deeper relationship with CBP beyond carrier obligations — strategic partnership on trade facilitation |

### Competitive Threat If They Don't

- Altana's Product Passports are expanding into supply chain traceability — directly competing with FedEx Dataworks' value proposition
- If CBP mandates GBI (which they're evaluating), importers will need identifiers from *some* IMC — FedEx Dataworks would be sending its customers to D&B, GS1, GLEIF, or Altana instead of offering the service itself
- UPS and DHL could pursue the same opportunity

---

## 11. Recommended Next Steps

### Phase 1: Internal Assessment (1-2 months)

1. Evaluate FedEx Dataworks' current entity resolution capabilities — can they resolve shipper/consignee records into distinct, persistent entities today?
2. Assess the D&B partnership data — how much entity matching is already done?
3. Quantify express/e-commerce entity coverage vs. the other IMCs' coverage of the same segment
4. Legal review: data sharing implications, conflict of interest analysis, multi-jurisdiction compliance

### Phase 2: CBP Engagement (2-3 months)

1. Informal outreach to CBP GBI program office (GBI@cbp.dhs.gov) to gauge interest
2. Prepare a concept paper: "Movement-Verified Business Identity" — what FedEx Dataworks can offer that no other IMC provides
3. Identify a pilot scope: e.g., Section 321 express shipments from high-risk origins

### Phase 3: Pilot (3-6 months)

1. Build FDX Identifier issuance system (entity resolution + persistent identifier assignment)
2. Create a data access API for CBP (entity lookup, movement verification, relationship graph)
3. Negotiate IMC agreement with CBP
4. Pilot with a subset of FedEx Trade Networks' brokerage clients

### Alternative: D&B Fast Track (1-3 months)

1. Extend the existing D&B partnership to include movement verification enrichment of DUNS records
2. Propose to CBP as an enhanced data element under D&B's existing IMC agreement
3. No new IMC registration required — faster to market

---

## Appendix: Key References

- [CBP GBI Program Overview](https://www.cbp.gov/trade/programs-administration/gbi)
- [CBP GBI FAQ](https://www.cbp.gov/trade/programs-administration/gbi/global-business-identifier-test-frequently-asked-questions)
- [Federal Register — GBI Test Modification (Aug 2025)](https://www.federalregister.gov/documents/2025/08/08/2025-15060/modification-of-the-national-customs-automation-program-test-concerning-the-submission-of-global)
- [Federal Register — GBI Extension (Feb 2024)](https://www.federalregister.gov/documents/2024/02/12/2024-02788/extension-and-modification-of-the-national-customs-automation-program-test-concerning-the-submission)
- [Altana Product Passports + CBP (Oct 2025)](https://www.businesswire.com/news/home/20251001718752/en/U.S.-Customs-and-Border-Protection-Selects-Altanas-AI-Powered-Product-Passports-to-Drive-Next-Generation-Supply-Chain-Traceability-Trusted-Trade)
- [FedEx Dataworks + D&B Partnership (Feb 2026)](https://newsroom.fedex.com/newsroom/global-english/dun-bradstreet-and-fedex-dataworks-to-launch-predictive-insights-tracking-u-s-retail-supply-and-demand)
- [FedEx Dataworks + ServiceNow (Oct 2025)](https://newsroom.fedex.com/newsroom/global-english/fedex-dataworks-and-servicenow-unite-ai-data-and-workflows-to-power-supply-chains-of-the-future)
- [FedEx Dataworks Platform](https://www.fedex.com/en-us/dataworks.html)
- [fdx Commerce Platform](https://www.fdx.com/content/fdx-com/sites/us/en_us/fdx/about.html)
