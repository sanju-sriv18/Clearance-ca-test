# CBP Integration & Trusted Trade Programs — Technical Requirements

**Date:** 2026-03-06
**Branch:** `production-hardening`
**Status:** Research & Planning

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current State Assessment](#2-current-state-assessment)
3. [ACE/ABI Transmission](#3-aceabi-transmission)
4. [CATAIR Data Element Compliance](#4-catair-data-element-compliance)
5. [Global Business Identifier (GBI) Program](#5-global-business-identifier-gbi-program)
6. [Trusted Trader Programs](#6-trusted-trader-programs)
7. [ISF (Importer Security Filing)](#7-isf-importer-security-filing)
8. [Reference Data Feeds](#8-reference-data-feeds)
9. [Bond Sufficiency & Financial Controls](#9-bond-sufficiency--financial-controls)
10. [Party Validation](#10-party-validation)
11. [Document Image System (DIS)](#11-document-image-system-dis)
12. [AD/CVD Enforcement](#12-adcvd-enforcement)
13. [Implementation Priority](#13-implementation-priority)

---

## 1. Executive Summary

The Clearance Platform has **72% of CBP rules implemented** (94 of 131) with full entry filing models, duty calculations, and compliance screening in place. However, all CBP interaction currently runs through **simulation only** — no live connection to CBP systems exists.

This document catalogs the technical requirements for production CBP integration, the GBI program, and trusted trader programs. The platform's architecture (jurisdiction adapters, two-layer status model, event-driven intelligence, CQRS) is well-positioned — these are **additive features, not architectural rewrites**.

---

## 2. Current State Assessment

### What We Have

| Capability | Implementation | Location |
|-----------|---------------|----------|
| Entry Filing Model | Full ORM with CBP fields (entry_number, entry_type, port_of_entry, release_number, release/summary status) | `declaration_management/models.py` |
| Two-Step Filing (3461 + 7501) | Modeled — release_status and summary_status tracked independently | `declaration_management/models.py` |
| Entry Type Determination | Automated (formal/informal/de minimis/section 321/type 86/FTZ/quota/TIB/drawback) | `declaration_management/entry_type.py` |
| Customs Valuation | Related party detection, assists, royalties, proceeds, alternative methods | `declaration_management/valuation.py` |
| CBP Fee Constants (FY2026) | MPF 0.3464% ($31.67–$614.35), HMF 0.125%, bond minimums | `contracts/constants/cbp_fees.py` |
| Compliance Checklist | 6 readiness gates (classification, tariff, compliance, documents, bond, ISF) | `declaration_management/routes.py` |
| CF-28/CF-29 Handling | Template categories, deadline tracking, LLM-assisted drafting | `declaration_management/routes.py` |
| CBP Authority Simulation | Risk-based acceptance/rejection, CF-28/CF-29 generation, exam scheduling | `app/simulation/actors/cbp_authority.py` |
| Broker Queue | 47 endpoints, entry CRUD, checklist, approval workflows, workload tracking | `declaration_management/routes.py` |
| Trade Remedies | Section 301, 232, 201 TRQ, AD/CVD data model | E2 tariff engine, `trade_intelligence` |
| PGA Requirements | 13/15 rules, agency-to-commodity mapping | E3 compliance engine |
| Trust Program Storage | JSONB field on Party model | `party_management/models.py` |
| ISF Model | Filing entity with 10+2 fields, deadline tracking | `declaration_management/models.py` |

### What We Don't Have

- No ABI network connectivity (transmission is simulated)
- No CATAIR field-level validation (6 proxy gates only)
- No GBI identifier support
- No live reference data feeds (OFAC, AD/CVD, HTSUS, exchange rates all static)
- No DIS (Document Image System) integration
- Bond sufficiency endpoint is a stub
- AD/CVD lookup exists in DB but is never queried by the tariff engine
- Trust programs stored but not integrated into clearance logic
- No EDI message formatting (X12, EDIFACT)

---

## 3. ACE/ABI Transmission

### Overview

The Automated Broker Interface (ABI) is the electronic filing channel to CBP's Automated Commercial Environment (ACE). All entry filings, status updates, and CBP responses flow through ABI.

### Technical Requirements

| Requirement | Description |
|------------|-------------|
| **ABI Client Registration** | Register as an ABI software vendor with CBP; obtain client representative and test credentials |
| **ABI Message Formatting** | Construct entry data in ABI record format per CATAIR specifications |
| **Transmission Protocol** | Connect to ABI network for message send/receive |
| **Entry Transmission** | File CBP Form 3461 (entry/immediate delivery) and Form 7501 (entry summary) |
| **Response Parsing** | Parse ABI response messages: acceptance, rejection, request for information, liquidation |
| **Error Handling** | Handle ABI error codes, validation failures, and resubmission logic |
| **Status Polling** | Poll ACE for entry status updates (release, hold, exam, liquidation) |
| **Entry Number Assignment** | Generate entry numbers: broker filer code prefix + 7-digit sequence + check digit |

### What Exists Today

- `EntryFiling` model has all required fields for a 3461/7501
- `filing_status` tracks the full lifecycle (draft → submitted → accepted → released)
- Two-layer status model (`jurisdiction_status` + `platform_status`) designed for authority responses
- CBP Authority simulation actor models realistic response patterns

### Gap

No actual ABI connectivity. Transmission is simulated through `cbp_authority.py`. This is the **single largest integration gap**.

---

## 4. CATAIR Data Element Compliance

### Overview

CATAIR (Customs and Trade Automated Interface Requirements) defines the ~50+ data elements required for ACE entry filings. Our 6-gate readiness checklist is a useful proxy but does not constitute field-level CATAIR compliance.

### Key Data Elements Requiring Validation

| Data Element | Current Status | Required Validation |
|-------------|---------------|-------------------|
| Entry Number | Free-text string | Broker filer code (3 chars) + entry number (7 digits) + check digit (1 digit) |
| Port of Entry | Field exists | 4-digit CBP port code from Schedule D |
| Entry Type | Enum exists | Validate against CBP entry type codes (01, 02, 06, 09, 11, 21, 23, 31, 46, 86) |
| IOR Number | Field exists | EIN format (XX-XXXXXXX) or SSN validation |
| MID | Not computed | Country code (2) + manufacturer name chars + city chars per CBP algorithm |
| HS/HTSUS Code | Field exists | 10-digit HTSUS with statistical suffix |
| Country of Origin | Field exists | ISO 3166 → CBP Schedule C country code mapping |
| Unit of Measure | Field exists | CBP-specific UOM codes per HTSUS heading |
| Declared Value | Field exists | Currency conversion to USD at CBP exchange rate |
| Surety Code | Not implemented | 3-digit surety company code for bond |
| Bond Type | Field exists | Single entry vs. continuous, with proper coding |
| Carrier Code | Field exists | SCAC (Standard Carrier Alpha Code) or IATA code |
| Consignee Number | Field exists | Validate format per party type |

### Gap

No field-by-field CATAIR validation exists. Need a validation layer that checks every ACE-required data element before allowing submission.

---

## 5. Global Business Identifier (GBI) Program

### Overview

GBI is a CBP pilot program (active through **February 23, 2027**) that aims to **replace the Manufacturer/Shipper ID (MID)** with standardized global business identifiers. It provides deeper supply chain visibility and signals low-risk, highly-compliant trade.

Since ACE began accepting GBI data in **December 2022**, the program has undergone multiple enhancements. CBP is testing whether to **mandate GBI** as a MID alternative.

### Accepted Identifiers

Four Identity Management Companies (IMCs) issue accepted identifiers:

| Identifier | Issuer | Format | Notes |
|-----------|--------|--------|-------|
| **DUNS** | Dun & Bradstreet | 9-digit number | Most widely held by US businesses |
| **GLN** | GS1 | 13-digit (with check digit) | Global standard for location/party identification |
| **LEI** | GLEIF | 20-character alphanumeric (ISO 17442) | Legal entity identification, common in financial sector |
| **Altana ID** | Altana Technologies USG Inc. | Proprietary | Newest IMC; also provides AI-powered Product Passports for supply chain traceability |

### Applicable Supply Chain Parties

GBI identifiers may be submitted for:

- **Manufacturer** — entity that produced the goods
- **Shipper** — entity that ships the goods
- **Seller** — entity that sold the goods
- **Exporter** — entity that exports from origin country
- **Distributor** — intermediary in the supply chain
- **Packager** — entity that packaged the goods

### Filing Process

1. Importers/brokers identify supply chain parties that need GBI identifiers
2. Parties obtain identifiers from one or more IMCs (DUNS, GLN, LEI, or Altana ID)
3. Importer/broker enrolls GBI identifiers with CBP (email GBI@cbp.dhs.gov)
4. Software provider configures ABI transmission to include GBI data elements
5. GBI identifiers are submitted via ACE as part of **Cargo Release** filings
6. Identifiers are submitted **prior to goods being used for Cargo Release**

### Altana Product Passports

CBP selected Altana's AI-powered Product Passports for next-generation supply chain traceability. Product Passports:

- Track products from raw materials to final product
- Enable importers to share supply chain provenance with CBP before manufacturing or arrival
- Allow CBP to review and approve passports, resolving issues before formal customs entry
- Directly relevant to **UFLPA enforcement** (forced labor supply chain visibility)

### Relationship to MID

GBI is designed as a **MID replacement**. The current MID (Manufacturer/Shipper Identification Code) is a CBP-constructed code using country + manufacturer name + city. GBI provides:

- Verifiable, globally standardized identifiers (vs. CBP's proprietary MID format)
- Richer entity data from IMC databases
- Supply chain mapping beyond just manufacturer/shipper
- Better support for forced labor and supply chain compliance investigations

### Technical Implementation Requirements

#### Data Model Changes

**Party Management** (`party_management/models.py`):
```
Party:
  gbi_identifiers: JSONB
    # Schema:
    # {
    #   "duns": "123456789",
    #   "gln": "1234567890123",
    #   "lei": "5493001KJTIIGC8Y1R12",
    #   "altana_id": "...",
    #   "enrolled_with_cbp": true,
    #   "enrollment_date": "2026-01-15",
    #   "verified": true
    # }
  gbi_roles: JSONB
    # Which GBI roles this party fills:
    # ["manufacturer", "shipper", "seller", "exporter", "distributor", "packager"]
```

**Entry Filing Association** (`declaration_management/models.py`):
```
EntryFiling:
  gbi_data: JSONB
    # GBI identifiers submitted with this entry, keyed by role:
    # {
    #   "manufacturer": {"party_id": "...", "duns": "123456789", "gln": "..."},
    #   "shipper": {"party_id": "...", "lei": "5493001KJTIIGC8Y1R12"},
    #   "seller": {"party_id": "...", "duns": "987654321"}
    # }
```

#### Validation Rules

| Identifier | Validation |
|-----------|-----------|
| DUNS | Exactly 9 digits |
| GLN | Exactly 13 digits, valid GS1 check digit (mod-10) |
| LEI | Exactly 20 characters, alphanumeric, ISO 17442 format (chars 1-4: LOU prefix, 5-18: entity code, 19-20: check digits) |
| Altana ID | Format TBD — validate against Altana API |

#### ABI Integration

- Include GBI data elements in ABI entry transmission alongside traditional MID
- Map GBI identifiers to the appropriate CATAIR record positions
- Support dual submission (MID + GBI) during the transition period

#### Enrollment Workflow

- API endpoint to register party GBI identifiers
- Track enrollment status with CBP per party
- Flag entries where GBI-enrolled parties reduce risk profile
- Dashboard visibility for broker to see GBI coverage across supply chain

---

## 6. Trusted Trader Programs

### Overview

Trusted trader programs provide trade facilitation benefits (faster release, fewer exams, lower risk scores) to participants who demonstrate supply chain security and compliance.

### Programs by Jurisdiction

| Program | Jurisdiction | Tiers | Benefits |
|---------|-------------|-------|----------|
| **C-TPAT** | US (CBP) | Tier 1, 2, 3 | Reduced exams, priority processing, front-of-line cargo release |
| **AEO** | EU | AEOC (customs), AEOS (security), AEOF (full) | ~95% auto-release vs 40% for new operators |
| **PIP** | Canada (CBSA) | — | Reduced exam rates, expedited release |
| **Linha Azul** | Brazil | — | Favorable parametrizacao (clearance channel) |
| **AEO (India)** | India (ICEGATE) | T1, T2, T3 | Tiered benefits based on compliance history |
| **AEO (Mexico)** | Mexico (SAT) | — | Expedited processing |
| **Mutual Recognition** | Cross-border | — | C-TPAT ↔ AEO ↔ PIP benefits recognized across borders |

### Current State

The `Party` model has a `trust_programs` JSONB field that stores enrollment data, but this data is **never consumed** by any clearance logic.

### Technical Implementation Requirements

#### Risk Score Integration

- Feed trust program status into the E0 pre-clearance risk scoring
- C-TPAT Tier 2/3 → lower risk score, reduced exam probability
- AEO-F holders → auto-release eligibility
- Mutual recognition: C-TPAT member importing via EU should benefit from AEO recognition

#### Clearance Logic Changes

| Integration Point | Change |
|-------------------|--------|
| Pre-qualification (E0) | Check C-TPAT/AEO membership; reduce risk flags |
| Exam scheduling | Lower exam rate for trusted traders (CBP assigns ~50% fewer exams to C-TPAT Tier 3) |
| Release priority | Front-of-line processing for C-TPAT members |
| Compliance screening (E3) | Adjust thresholds for trusted parties |
| Jurisdiction adapters | Each adapter applies program-specific benefits per jurisdiction |

#### Program Management

- Track certification status, tier, expiration, audit history
- Alert on upcoming certification renewals
- Track compliance incidents that could affect program standing
- Dashboard view: trust program coverage across importer portfolio

---

## 7. ISF (Importer Security Filing)

### Overview

ISF ("10+2") requires importers to submit advance cargo information for **ocean shipments** at least 24 hours before vessel loading.

### Current State

`ISFFiling` model exists with fields for all 10 importer elements (shipper, consignee, seller, buyer, ship_to, consolidator, container_stuffing_location, hs_codes, country_of_origin, manufacturer) plus deadline tracking.

### Gap

- No automated deadline enforcement (filing not blocked if ISF is overdue)
- No ISF submission to ACE via ABI
- No ACAS (Air Cargo Advance Screening) filing for air shipments
- ISF validation issues tracked but not linked to entry release holds

---

## 8. Reference Data Feeds

### Overview

CBP compliance requires up-to-date reference data from multiple government sources. Currently all reference data is static/hardcoded.

### Required Live Feeds

| Feed | Source | Frequency | Current State | Impact |
|------|--------|-----------|--------------|--------|
| **OFAC SDN List** | Treasury OFAC | Daily XML | 50+ hardcoded entities | Blocked party screening incomplete |
| **HTSUS Schedule** | USITC | Quarterly | ~20 hardcoded MFN rates | Missing ~17,980 tariff lines |
| **AD/CVD Orders** | DOC/ITA | As published | DB table exists, never queried | Trade remedy duties not enforced |
| **Section 301 Lists** | USTR | As modified | Static list | Tariff rate changes not propagated |
| **CBP Exchange Rates** | Treasury/CBP | Quarterly | Assumes USD only | Currency conversion required for valuation |
| **UFLPA Entity List** | DHS FLETF | As published | 14 hardcoded entities | Forced labor screening incomplete |
| **CBP CSMS** | CBP | Real-time RSS | Monitored (regulatory monitor) | System changes and cargo alerts |
| **Federal Register** | GPO | Daily | Monitored (regulatory monitor) | Rule changes and notices |

### Existing Infrastructure

The regulatory monitor (`REGULATORY_MONITOR_ENABLED`) already polls Federal Register, CBP CSMS, USTR, and Google News on a configurable interval. Ingestion pipeline scaffolding exists in `shared/ingestion/` for 7 feed types. The gap is in **applying** the ingested data to the tariff and compliance engines.

---

## 9. Bond Sufficiency & Financial Controls

### Current State

- Bond type and amount fields exist on `EntryFiling`
- Constants defined: $50,000 continuous bond minimum, single entry bond $100 minimum
- Bond sufficiency endpoint exists but **returns stub** `{"sufficient": true}`

### Required Implementation

| Requirement | Rule |
|------------|------|
| Continuous bond calculation | Max of $50,000 or 10% of annual duties/taxes/fees |
| Single entry bond | Equal to total duties + taxes + fees for the entry |
| AD/CVD bond | 100% of margin deposit required (separate from regular bond) |
| Bond type selection | Continuous vs. single entry based on filing volume |
| Surety validation | Verify surety company code (3-digit) against CBP-approved list |
| Broker bond vs. importer bond | Distinguish which bond covers the entry |

---

## 10. Party Validation

### Current State

2 of 8 CBP party validation rules implemented. IOR exists with jurisdiction identifiers; POA lifecycle is complete.

### Required Implementation

| Requirement | Status |
|------------|--------|
| IOR EIN format validation (XX-XXXXXXX) | Not implemented |
| CBP Form 5106 registration check | Not implemented |
| Manufacturer ID (MID) computation (country + name + city) | Not implemented |
| Broker license verification against CBP database | String field only, no verification |
| Related party disclosure mechanism | Not implemented |
| Consignee number validation | Not implemented |
| Foreign exporter identification | Partial (party model exists) |
| Ultimate consignee tracking | Not implemented |

---

## 11. Document Image System (DIS)

### Overview

CBP's Document Image System requires electronic upload of supporting documents for certain entry types and commodities.

### Required Implementation

- Identify which documents require DIS upload per entry type and commodity
- Track upload status per document per entry
- Integrate with CBP DIS upload endpoint
- Link DIS confirmation to entry filing readiness checklist
- Support document types: commercial invoice, packing list, bill of lading, certificates of origin, PGA-specific documents

### Current State

Document management domain exists with full document storage and metadata. Gap is the DIS-specific upload workflow and CBP confirmation tracking.

---

## 12. AD/CVD Enforcement

### Current State

- `trade_intelligence.adcvd_orders` table exists with columns: hs_code, country_code, case_number, ad_rate_pct, cvd_rate_pct, status
- E2 tariff engine **explicitly skips** AD/CVD: "AD/CVD omitted for POC golden path"

### Required Implementation

| Requirement | Description |
|------------|-------------|
| Runtime AD/CVD lookup | Query `adcvd_orders` in E2 tariff calculation |
| Manufacturer/exporter-specific rates | Look up rates by specific manufacturer, not just country+HS |
| "All others" rate | Apply country-wide rate when specific manufacturer rate unavailable |
| PRC-wide rates | Special handling for China-wide AD/CVD rates |
| Cash deposit requirements | Calculate and enforce AD/CVD cash deposits |
| Case-specific instructions | Track DOC instructions per AD/CVD case number |
| AD/CVD bond requirements | 100% margin deposit bond separate from regular entry bond |
| Liquidation tracking | AD/CVD entries have special liquidation timelines |

---

## 13. Implementation Priority

### Tier 1 — Required for Live CBP Filing

| # | Capability | Effort | Dependencies |
|---|-----------|--------|-------------|
| 1 | **ABI Connectivity** | Large | CBP vendor registration, test environment access |
| 2 | **CATAIR Field Validation** | Medium | CATAIR specification document |
| 3 | **Entry Number Generation** | Small | Broker filer code assignment |
| 4 | **AD/CVD Enforcement** | Medium | Live AD/CVD data feed |
| 5 | **Bond Sufficiency** | Small | None (constants exist) |

### Tier 2 — Required for Compliance

| # | Capability | Effort | Dependencies |
|---|-----------|--------|-------------|
| 6 | **Live Reference Data Feeds** | Large | Feed source access, ingestion pipeline wiring |
| 7 | **Party Validation (MID, EIN, 5106)** | Medium | None |
| 8 | **ISF Filing Enforcement** | Small | ABI connectivity |
| 9 | **DIS Integration** | Medium | CBP DIS endpoint access |

### Tier 3 — Competitive Advantage

| # | Capability | Effort | Dependencies |
|---|-----------|--------|-------------|
| 10 | **GBI Program Support** | Medium | Party management model extension, ABI connectivity |
| 11 | **Trusted Trader Integration (C-TPAT/AEO)** | Medium | Risk engine changes |
| 12 | **Altana Product Passports** | Medium | Altana API access |

### Tier 4 — Multi-Jurisdiction EDI

| # | Capability | Effort | Dependencies |
|---|-----------|--------|-------------|
| 13 | **EDI Message Formats (X12, EDIFACT)** | Large | Per-jurisdiction specifications |
| 14 | **AES/EEI Export Filing** | Medium | AES Direct access |
| 15 | **Non-US Jurisdiction Connectivity** | Very Large | Per-country customs authority registration |

---

## References

- [CBP GBI Program](https://www.cbp.gov/trade/programs-administration/gbi)
- [GBI FAQ](https://www.cbp.gov/trade/programs-administration/gbi/global-business-identifier-test-frequently-asked-questions)
- [GBI Fact Sheet](https://www.cbp.gov/document/fact-sheets/global-business-identifier-gbi)
- [GBI Identifier Factsheets](https://www.cbp.gov/document/fact-sheets/global-business-identifier-how-obtain-identifier-factsheets)
- [Federal Register — GBI Modification (Aug 2025)](https://www.federalregister.gov/documents/2025/08/08/2025-15060/modification-of-the-national-customs-automation-program-test-concerning-the-submission-of-global)
- [CBP Launches GBI Pilot](https://www.cbp.gov/newsroom/national-media-release/cbp-launches-global-business-identifier-pilot-increase-supply-chain)
- [CBP Rules Engine Gap Analysis](../clearance-engine/docs/cbp-rules-engine-gap-analysis.md) (internal)
- [Platform Architecture](./architecture-clearance-platform.md) (internal)
