# EU/UK Customs Operations Gap Analysis

> Making EU and UK import/export clearance a **first-class experience** equal to the current US CBP support.

**Author:** Customs Operations Audit Agent
**Date:** 2026-02-07
**Scope:** European Union (27 member states) + United Kingdom (post-Brexit)

---

## Executive Summary

The Clearance Vibe platform has **deep, production-quality US customs support**: a full CBP entry filing lifecycle (CF-7501 equivalent), CF-28/CF-29 response handling, ISF 10+2 tracking, PGA agency review, bond management, a risk-scored customs screening simulation, and a broker dashboard purpose-built around the US clearance pipeline. The EU/UK presence is limited to a **tariff calculation engine** (`EUTaxRegime` in `backend/app/engines/e2_tariff/regimes/eu.py`) that computes MFN duty + member-state VAT. No EU/UK customs declaration workflows, document types, regulatory identifiers, simulation actors, or frontend surfaces exist.

Reaching parity requires building the EU/UK equivalent of every layer the US system already has: declaration filing, customs authority response handling, regulatory identifiers, document types, trade agreement preference management, simulation actors, and agent tools.

### Gap Severity Summary

| Area | US Status | EU/UK Status | Gap Severity |
|------|-----------|-------------|--------------|
| Tariff calculation | Full (MFN + 301 + 232 + IEEPA + AD/CVD + MPF + HMF) | Basic (MFN + VAT only) | P1 |
| Customs declaration filing | Full CF-7501 lifecycle | None | **P0** |
| Customs authority responses | CF-28/CF-29/exam | None | **P0** |
| Required identifiers (EORI, AEO) | N/A (US uses EIN/SSN) | None | **P0** |
| Document types | US set (CI, PL, BoL, CoO, PGA) | None EU-specific | **P0** |
| Trade agreements / preferences | FTA_PARTNERS_US only | None | P1 |
| Regulatory compliance (ICS2, REACH, CBAM) | PGA/UFLPA/entity screening | None | P1 |
| Customs procedures (warehousing, IP, OP) | N/A | None | P1 |
| Simulation actors | Full US lifecycle | None | P1 |
| Agent tools | US-specific (CBP terminology) | None | P1 |
| Frontend broker surface | US-hardcoded | None | **P0** |
| UK post-Brexit specifics (CDS, Windsor) | N/A | None | P1 |

---

## 1. Customs Authority Workflows

### 1.1 What the US System Does (Benchmark)

The US platform implements a complete CBP entry filing lifecycle:

- **Entry creation**: `POST /api/broker/queue/claim` creates an `EntryFiling` with a generated entry number (filer-sequence-check format), entry type "01", default port "2704" (LA/Long Beach)
- **Pipeline stages**: Draft → Pending Approval → Submitted → CBP Response (CF-28/CF-29/Exam) → Released
- **State machine**: `backend/app/simulation/state_machine.py` defines transitions: `at_customs → entry_filed → cleared/cf28_pending/cf29_pending/exam_scheduled/held`
- **CBP Authority actor**: `backend/app/simulation/actors/cbp_authority.py` simulates CBP risk-based responses with CF-28 question templates (classification, valuation, origin, compliance) and CF-29 notices (rate advance, value adjustment, origin change)
- **Response deadlines**: Tracked with `deadline_days` and `protest_deadline_days` in the `cbp_response` JSONB
- **Exam types**: VACIS/NII, intensive, tailgate — differentiated by transport mode

### 1.2 What EU/UK Requires

#### EU Customs Declarations Under the Union Customs Code (UCC)

The EU does not use "CF-7501" or "entry filings." Instead, customs declarations are structured around **UCC declaration types**:

| Declaration Type | Purpose | US Equivalent |
|-----------------|---------|---------------|
| **H1** | Release for free circulation | CF-7501 (formal entry) |
| **H2** | Customs warehousing | No direct equivalent |
| **H3** | Temporary admission | ATA Carnet |
| **H4** | Inward processing | Drawback (loosely) |
| **H5** | Outward processing | 9802 provisions |
| **H6** | Simplified declaration for low-value consignments (<150 EUR) | Type 86/Section 321 |
| **H7** | Import One Stop Shop (IOSS) declarations | None |
| **I1** | Export declaration | AES/EEI |
| **C1** | Transit declaration (NCTS) | IT/ABI transit |

Each member state operates its own customs IT system:
- **Germany**: ATLAS (Automatisiertes Tarif- und Lokales Zollabwicklungssystem)
- **France**: DELTA G / DELTA X
- **Netherlands**: AGS (Aangiftesysteem)
- **Belgium**: PLDA (Paperless Douane en Accijnzen)
- **Italy**: AIDA

#### UK Customs Declaration Service (CDS)

Post-Brexit, the UK replaced CHIEF with the **Customs Declaration Service (CDS)**:
- Uses HMRC's Data Element (DE) model (not SAD boxes)
- Procedure codes: e.g., 40 00 000 (free circulation), 71 00 000 (customs warehousing)
- Declaration categories: H1-H5 (aligned with UCC but with UK-specific data elements)
- CDS also handles supplementary declarations for simplified procedures

#### EU/UK Customs Authority Response Equivalents

| US Concept | EU Equivalent | UK Equivalent |
|------------|--------------|---------------|
| CF-28 (Request for Information) | Pre-release query / customs query (varies by MS) | CDS query / C88 query |
| CF-29 (Notice of Action) | Customs debt notification / post-clearance adjustment | C18 (post-clearance demand) |
| Physical examination | Physical inspection / documentary check | Physical exam at BCP |
| Protest (19 USC 1514) | Appeal to national customs authority → Tribunal → CJEU | Appeal to HMRC → First-tier Tribunal |
| Liquidation | Customs debt calculation + notification | Duty calculation + C79 certificate |

### 1.3 What Already Exists in the Codebase

- The `EUTaxRegime` class computes MFN duty + VAT — **no declaration workflow exists**
- The `TariffEngine.execute()` dispatches to EU regime but only returns tariff results
- No EU-specific state machine transitions, no EU authority actor, no EU response types

### 1.4 What Needs to Be Built

#### Data Model (P0)
- `CustomsDeclaration` model (replaces/extends `EntryFiling` for EU/UK)
  - `declaration_type`: enum (H1-H7, I1, C1)
  - `procedure_code`: string (e.g., "40 00 000")
  - `declaration_system`: enum (ATLAS, DELTA, AGS, CDS, etc.)
  - `mrn` (Movement Reference Number): replaces US entry number
  - `customs_office_code`: replaces `port_of_entry`
  - `eori_declarant`: EORI of the declaring party
  - `eori_representative`: EORI of customs representative (if indirect)
  - `representation_type`: enum (direct, indirect)
  - `declaration_status`: enum matching EU lifecycle states
  - `customs_value_method`: enum (transaction value, deductive, computed, etc.)
  - `preference_code`: for FTA/GSP preference claims

#### Backend (P0)
- EU/UK customs authority response handler (equivalent to `cbp_authority.py`)
- EU declaration state machine (pre-lodgement → acceptance → risk analysis → control → release/query/examination)
- New API routes: `/api/broker/declarations` (EU/UK filing surface)
- MRN generation logic per member-state format

#### Frontend (P0)
- Declaration type selector (H1-H7) in entry creation flow
- EU-specific pipeline stages replacing the US CF-28/CF-29 stages
- Procedure code selector with UCC-compliant codes
- Customs office selector (EU customs office database ~3,000 entries)

#### Simulation (P1)
- `EUCustomsAuthorityActor`: risk profiling (AEO green lane vs standard), documentary checks, physical inspections
- `UKCDSAuthorityActor`: CDS-specific response simulation

**Priority: P0** — without declarations, the EU broker experience is non-functional.

---

## 2. Required Identifiers

### 2.1 What the US System Does

- Broker `license_number` and `license_district` stored on the `Broker` model
- Entry number format: `filer-sequence-check` (randomly generated in `claim_entry`)
- Importer identified by `company_name` (no explicit EIN/IOR number field)
- US port codes from `US_PORT_CODES` in `reference_data.py`

### 2.2 What EU/UK Requires

| Identifier | Purpose | Mandatory? | Format |
|-----------|---------|-----------|--------|
| **EORI** (Economic Operators Registration and Identification) | Identifies every economic operator in EU trade | Yes — ALL importers, exporters, customs reps | `{country_code}{up to 15 chars}` e.g., `DE123456789012345` |
| **AEO** (Authorised Economic Operator) | Trusted trader certification → simplified procedures, fewer inspections | No, but critical for fast clearance | AEO-C (customs), AEO-S (security), AEO-F (full) |
| **REX** (Registered Exporter) | Self-certification for GSP/EPA origin declarations | Yes for GSP exports >6,000 EUR | REX number |
| **Customs Representative Authorization** | Proof of authority to act on importer's behalf | Yes — always | Direct or indirect representation |
| **MRN** (Movement Reference Number) | Unique declaration reference | Yes — system-assigned | 18 chars: `{year}{country}{alphanum}` |
| **DUCR/MUCR** | Declaration Unique Consignment Reference / Master UCR | UK CDS only | Unique per consignment |
| **BTI** (Binding Tariff Information) | Pre-ruling on HS classification — legally binding | Optional but very useful | BTI reference number |
| **UK EORI** | Post-Brexit UK-specific EORI | Yes for UK trade | `GB{number}000` format |
| **VAT ID** | For postponed VAT accounting | Yes for Procedure 42/63 | Member-state VAT format |
| **Deferment Account Number (DAN)** | UK duty deferment | UK-specific | 7-digit number |

### 2.3 What Already Exists

- **Nothing EU/UK-specific.** The `Broker` model has `license_number` and `license_district` (US concepts). No EORI field, no AEO status, no customs representative authorization model.

### 2.4 What Needs to Be Built

#### Data Model (P0)
- Add `eori_number` field to `Broker` model (or create `JurisdictionCredential` model)
- Add `aeo_status` field (None/AEO-C/AEO-S/AEO-F) with expiry date
- Add `customs_representative_type` (direct/indirect)
- Add EORI validation (country prefix + length check)
- `ImporterProfile` model or extension with `eori`, `vat_id`, `deferment_account`
- `BindingTariffInformation` model for BTI reference tracking

#### Backend (P0)
- EORI validation endpoint (format check + optional EU EORI database lookup)
- AEO status lookup and green-lane routing logic
- MRN format generation per member state
- Customs representative authorization check in declaration flow

#### Frontend (P0)
- EORI input field with format validation in importer/declarant setup
- AEO status badge on broker/importer profile
- Customs representative type selector
- BTI reference field in classification workflow

**Priority: P0** — EORI is mandatory for ALL EU trade; without it, no declaration can be filed.

---

## 3. Document Types

### 3.1 What the US System Does

The US checklist (`CHECKLIST_LABELS` in `broker.py`) tracks 8 document/data items:
1. Commercial Invoice
2. Packing List
3. Bill of Lading / Airway Bill
4. HS Classification (data check)
5. Customs Valuation (data check)
6. Certificate of Origin
7. PGA Documentation
8. Bond Verification

Document field schemas are defined in `documents.py` (`DOCUMENT_FIELD_SCHEMAS`) for Commercial Invoice, Packing List, Bill of Lading, and Certificate of Origin. FTA partners are US-only (`FTA_PARTNERS_US`).

### 3.2 What EU/UK Requires

#### Core Declaration Documents

| Document | Purpose | US Equivalent | Priority |
|----------|---------|---------------|----------|
| **SAD** (Single Administrative Document) / CN23 | Customs declaration form (8 copies) | CF-7501 | P0 |
| **CDS Declaration** (UK) | UK customs declaration (Data Elements) | CF-7501 | P0 |
| **EUR.1** | Proof of EU preferential origin | Certificate of Origin (generic) | P0 |
| **EUR-MED** | Origin cert for Pan-Euro-Med cumulation | N/A | P1 |
| **ATR** (Movement Certificate) | Turkey customs union origin proof | N/A | P1 |
| **T1** (External Transit) | Goods not in EU free circulation | IT (in-bond) | P1 |
| **T2** (Internal Transit) | EU goods moving through non-EU territory | N/A | P1 |
| **DV1** (Customs Value Declaration) | Detailed valuation declaration (>20,000 EUR) | CF-7501 Box 34 | P0 |
| **INF** documents (INF1, INF3) | Inward/outward processing certificates | Drawback docs | P2 |
| **C79** (UK) | Import VAT certificate | None | P1 |
| **C88** (UK) | CDS entry summary declaration | CF-7501 | P0 |

#### Regulatory / Compliance Documents

| Document | Purpose | US Equivalent | Priority |
|----------|---------|---------------|----------|
| **REACH pre-registration** | Chemical substance registration proof | TSCA compliance | P1 |
| **CE Declaration of Conformity** | Product safety certification | CPSC certificate | P1 |
| **UKCA Declaration** | UK product safety (replacing CE) | CPSC certificate | P1 |
| **Phytosanitary Certificate** | Plant health for agri imports | APHIS certificate | P1 |
| **Health Certificate** | For food/animal products | FDA prior notice | P1 |
| **CBAM Report** | Carbon emissions declaration | None (new regime) | P1 |
| **Deforestation Due Diligence** | EU Deforestation Regulation compliance | None (new regime) | P2 |
| **Intrastat Declaration** | Intra-EU trade statistics | None (domestic) | P2 |
| **Binding Tariff Information (BTI)** | Pre-ruling on classification | CBP HQ Rulings | P1 |
| **ATA Carnet** | Temporary admission of goods | ATA Carnet (same) | P2 |

#### Safety and Security Documents (ICS2)

| Document | Purpose | US Equivalent | Priority |
|----------|---------|---------------|----------|
| **ENS** (Entry Summary Declaration) | Pre-arrival risk assessment (ICS2) | ISF 10+2 | P0 |
| **EXS** (Exit Summary Declaration) | Pre-departure security filing | AES/EEI | P1 |
| **ETSF authorization** | External Temporary Storage Facility | FTZ designation | P2 |

### 3.3 What Already Exists

- `DOCUMENT_FIELD_SCHEMAS` in `documents.py` has US document templates only
- `FTA_PARTNERS_US` is the only FTA lookup — no EU FTA partners defined
- `_DOC_TYPE_TO_CHECKLIST` mapping is US-specific
- PGA actor references US agencies only (FDA, EPA, CPSC, NHTSA)

### 3.4 What Needs to Be Built

#### Data Model (P0)
- Extend `DOCUMENT_FIELD_SCHEMAS` with EU document field templates (EUR.1 fields, DV1 fields, ENS data elements)
- Create `EU_DOCUMENT_CHECKLIST_LABELS` equivalent to `CHECKLIST_LABELS`
- Add jurisdiction-aware document requirements engine

#### Backend (P0)
- Jurisdiction-aware checklist computation (`_compute_checklist_state` must branch on destination country)
- EUR.1 / EUR-MED / ATR generation and validation logic
- ENS filing endpoint (ICS2 equivalent of ISF)
- EU FTA partners lookup (`FTA_PARTNERS_EU` dict)
- DV1 (customs value declaration) validation for shipments >20,000 EUR

#### Frontend (P0)
- EU-specific checklist items rendered in `EntryDetail.tsx`
- EUR.1/ATR document upload with field validation
- ENS filing status tracking in the declaration timeline

**Priority: P0** — the checklist is the core broker workflow; EU checklists are completely different from US ones.

---

## 4. Regulatory Frameworks

### 4.1 What the US System Does

- **Entity screening**: OFAC SDN, BIS Entity List, DPL (via `screen_entity` tool and compliance engine)
- **UFLPA**: Forced labor risk assessment (Uyghur Forced Labor Prevention Act) — integrated into classification and hold logic
- **PGA agency review**: FDA, EPA, CPSC, NHTSA simulated in `PGAActor`
- **Regulatory signals**: Engine 6 (`e6_regulatory`) tracks tariff changes, has one EU CBAM signal
- **ISF 10+2**: `ISFActor` simulates pre-arrival security filing for ocean cargo

### 4.2 What EU/UK Requires

#### Import Control System 2 (ICS2) — **P0**

ICS2 is the EU's equivalent of the US ISF 10+2 / AMS, but more comprehensive:
- **Release 3 (2024+)**: ALL transport modes require pre-arrival ENS filing
- **Carrier filing obligations**: Airlines must submit PLACI (Pre-Loading Advance Cargo Information) data
- **Risk assessment**: EU customs authorities assess ENS data before vessel/aircraft arrival
- **Do-Not-Load (DNL) instructions**: EU can stop shipments before they're loaded at origin
- **Multiple filing**: ENS may need to be filed in multiple member states for transshipment

**Gap**: No ICS2 ENS filing, no PLACI tracking, no multi-MS filing, no DNL event handling.

#### Export Control System (ECS) — **P1**

- Export declarations required for all goods leaving the EU customs territory
- MRN is issued at the office of export
- Exit confirmation at the office of exit
- Dual-use goods require export license (EU 2021/821)

#### REACH Regulation — **P1**

- All chemical substances imported into the EU in quantities >1 tonne/year must be registered
- REACH pre-registration / registration numbers required in customs documentation
- Restriction and authorization lists (SVHC — Substances of Very High Concern)
- Affects HS chapters 28-39 significantly

#### CE / UKCA Marking — **P1**

- Products sold in the EU/EEA must carry CE marking with Declaration of Conformity
- UK requires UKCA marking (transitional period extension ongoing)
- Relevant directives: Machinery Directive, Low Voltage, EMC, RoHS, Medical Devices, Toys
- Customs may request Declaration of Conformity at import

#### EU Sanctions (Consolidated List) — **P1**

| EU Screening Requirement | US Equivalent |
|-------------------------|---------------|
| EU Consolidated Financial Sanctions List | OFAC SDN List |
| EU Dual-Use List (Annex I, EU 2021/821) | BIS CCL/EAR |
| EU Military List (Common Military List) | ITAR/USML |
| EU Anti-Torture Regulation list | N/A |
| UK OFSI Consolidated List | OFAC SDN List |
| UK Export Control List | BIS CCL |

**Gap**: Entity screening only checks US lists. Need EU/UK screening integration.

#### CBAM (Carbon Border Adjustment Mechanism) — **P1**

- **Effective 2026**: Mandatory carbon certificates for imports of iron/steel, aluminum, cement, fertilizers, electricity, hydrogen
- Importers must declare embedded carbon emissions and purchase CBAM certificates
- Quarterly CBAM reports required
- Affects HS chapters: 72, 73, 76, 2523, 3105, 2716, 2804
- **Already referenced in Engine 6 signal SIG-002** but no operational implementation

#### EU Deforestation Regulation (EUDR) — **P2**

- Due diligence requirements for commodities linked to deforestation: soy, beef, palm oil, wood, cocoa, coffee, rubber
- Geolocation data required for production plots
- Effective date has been postponed but compliance preparation is essential
- Risk assessment and due diligence statements required at import

### 4.3 What Already Exists

- One CBAM regulatory signal in Engine 6 — no operational enforcement
- Entity screening is US-only (OFAC, BIS Entity List)
- PGA actor references US agencies only
- ISF actor is US-specific (ocean only, 10+2 format)

### 4.4 What Needs to Be Built

#### Backend (P0)
- ICS2 ENS filing data model and status tracking
- EU sanctions screening integration (EU Consolidated List + UK OFSI)
- Risk assessment engine that differentiates AEO vs standard operators

#### Backend (P1)
- CBAM reporting module (carbon emission declarations, certificate tracking)
- REACH compliance check (SVHC list matching against HS codes)
- CE/UKCA conformity document requirement logic in document engine
- ECS export declaration support
- EU dual-use export license check

#### Simulation (P1)
- EU regulatory compliance actor (REACH checks, CE marking verification, CBAM reporting)
- ICS2 risk assessment actor (ENS evaluation, DNL issuance)
- EU/UK sanctions screening actor

**Priority: P0 for ICS2 (mandatory for all EU imports), P1 for others.**

---

## 5. Trade Agreements & Preferences

### 5.1 What the US System Does

`FTA_PARTNERS_US` in `documents.py` defines US FTA partners:
```python
FTA_PARTNERS_US = {
    "CA": "USMCA", "MX": "USMCA", "AU": "AUSFTA", ...
}
```
Used in: checklist origin cert requirement, document requirements endpoint. Only checks if origin is FTA partner → requires Certificate of Origin.

### 5.2 What EU/UK Requires

The EU has the **world's most extensive network of trade agreements**:

#### EU Preferential Trade Agreements

| Agreement | Partners | Preference Mechanism | Origin Proof | Priority |
|-----------|----------|---------------------|-------------|----------|
| **EU-UK TCA** | UK | Zero-duty on qualifying goods | EUR.1 or origin self-certification | P0 |
| **EU-Japan EPA** | Japan | Progressive tariff elimination | Origin statement on invoice (REX) | P1 |
| **EU-Korea FTA** | South Korea | Zero duty on industrial goods | EUR.1 or invoice declaration | P1 |
| **EU-Canada CETA** | Canada | Tariff elimination | EUR.1 or origin statement | P1 |
| **EU-Singapore FTA** | Singapore | Tariff elimination | EUR.1 or invoice declaration | P1 |
| **EU-Vietnam FTA** | Vietnam | Progressive tariff reduction | EUR.1 or invoice declaration | P1 |
| **EU-Mercosur** | Brazil, Argentina, etc. | Pending ratification | TBD | P2 |
| **Pan-Euro-Mediterranean (PEM)** | 25+ countries | Cumulation of origin | EUR-MED certificate | P1 |
| **EU GSP** (Generalized System of Preferences) | ~70 developing countries | Reduced/zero duty | Form A or REX self-certification | P1 |
| **EBA** (Everything But Arms) | 46 LDCs | Duty-free, quota-free | Form A or REX | P1 |
| **EU-Turkey Customs Union** | Turkey | No customs duties on industrial goods | ATR movement certificate | P1 |
| **EEA Agreement** | Norway, Iceland, Liechtenstein | Free movement of goods | EUR.1 | P1 |
| **EU-Switzerland** | Switzerland | Bilateral agreements | EUR.1 | P1 |

#### UK Post-Brexit FTAs

| Agreement | Partners | Notes |
|-----------|----------|-------|
| **UK-EU TCA** | EU 27 | Rules of origin are complex |
| **UK-Japan CEPA** | Japan | Rollover from EU-Japan EPA |
| **UK-Australia FTA** | Australia | New post-Brexit deal |
| **UK-New Zealand FTA** | New Zealand | New post-Brexit deal |
| **UK-CPTPP** | 11 Asia-Pacific nations | UK acceded 2024 |
| **DCTS** (Developing Countries Trading Scheme) | ~65 countries | Replaces EU GSP |

#### Preference Calculation Impact

Unlike the US where FTA preference is binary (preferential rate vs MFN), EU preferences involve:
- **Tariff quotas** (TRQs): Preference only within quota volume
- **Tariff suspension**: Temporary duty reduction for specific HS codes
- **Cumulation rules**: Bilateral, diagonal, or full cumulation depending on agreement
- **Sufficient working or processing**: Rules of origin criteria (PSR — Product Specific Rules)
- **Tolerance rules**: De minimis non-originating content thresholds

### 5.3 What Already Exists

- `FTA_PARTNERS_US` only — no EU/UK FTA partner data
- No preference rate calculation in EU tariff engine (only MFN rates)
- No rules of origin validation
- No tariff quota tracking

### 5.4 What Needs to Be Built

#### Data Model (P1)
- `FTA_PARTNERS_EU` and `FTA_PARTNERS_UK` lookup dictionaries
- `PreferenceClaimm` model: FTA reference, origin proof type, cumulation type, preference rate
- `TariffQuota` model: quota volume, utilization tracking, in-quota vs out-of-quota rates
- Rules of origin criteria by agreement + HS code

#### Backend (P1)
- Preference rate lookup in EU tariff engine (preferential rate vs MFN rate selection)
- Tariff quota availability check endpoint
- Rules of origin validation logic
- Cumulation calculation for PEM zone
- Origin proof document type determination (EUR.1 vs REX vs invoice declaration)

#### Frontend (P1)
- FTA preference selector in tariff calculation UI
- Rules of origin compliance indicator
- Tariff quota utilization display
- Origin proof type guidance

**Priority: P1** — preferences significantly affect landed cost but clearance can technically proceed at MFN rates.

---

## 6. Tax Regime

### 6.1 What the US System Does

`USTaxRegime` computes: MFN duty + Section 301 + Section 232 + IEEPA + AD/CVD + MPF + HMF. No US sales tax is applied at import.

### 6.2 What EU/UK Requires

#### VAT on Import

The EU tariff engine already computes basic import VAT (`EUTaxRegime._get_vat_rate`). However, several critical VAT mechanisms are missing:

| VAT Mechanism | Description | Impact | Priority |
|--------------|-------------|--------|----------|
| **Postponed VAT Accounting (PVA)** | Defer import VAT to periodic VAT return (available in IE, NL, BE, UK, and more) | No upfront VAT payment at border | P1 |
| **Procedure 42** | Import with VAT exemption when goods immediately dispatched to another EU MS | Zero import VAT at first MS of entry | P1 |
| **Procedure 63** | Re-import after outward processing with VAT exemption + onward dispatch | Complex VAT calculation | P2 |
| **Reverse charge mechanism** | Buyer accounts for VAT instead of seller | Different VAT reporting | P1 |
| **Fiscal representation** | Required in some MS for non-established importers | Additional entity relationship | P1 |
| **Reduced VAT rates** | Certain goods qualify for reduced rates (food, medicines, books) | Different rate tiers | P1 |
| **IOSS** (Import One Stop Shop) | Simplified VAT for B2C imports <150 EUR | Different declaration pathway | P2 |

#### Excise Duties

| Product Category | Duty Type | Notes |
|-----------------|-----------|-------|
| Alcohol (HS 2203-2208) | Specific duty per hectolitre/degree | Varies dramatically by MS |
| Tobacco (HS 2402-2403) | Specific + ad valorem | Minimum excise duty rules |
| Energy products (HS 2701-2716) | Specific duty per litre/kg/GJ | EU Energy Tax Directive |

#### Customs Debt Management

- **Guarantee management**: Financial guarantee required for customs debt
- **Duty deferment**: Authorized traders can defer duty payment (monthly account)
- **Duty suspension**: Goods under customs procedures don't incur duty until release
- **Remission/repayment**: Process for claiming back overpaid duties

### 6.3 What Already Exists

- `EUTaxRegime` has: MFN duty lookup + standard VAT rates for 27 MS
- Missing: PVA, Procedure 42/63, excise duties, reduced VAT rates, guarantee management, fiscal representation

### 6.4 What Needs to Be Built

#### Backend (P1)
- PVA flag in tariff computation (VAT deferred vs paid at border)
- Procedure 42/63 VAT exemption logic
- Excise duty calculation engine for alcohol, tobacco, energy products
- Reduced/zero VAT rate lookup by HS code and member state
- Guarantee/deferment account management
- Duty suspension tracking for customs procedures

#### Frontend (P1)
- PVA indicator in tariff breakdown
- Excise duty line items in landed cost display
- Procedure selection affecting VAT computation

**Priority: P1** — VAT is computed but missing key mechanisms that affect cash flow significantly.

---

## 7. Customs Procedures

### 7.1 What the US System Does

The US system models one primary procedure: **import for consumption** (Entry Type 01). The `entry_type` field defaults to "01" in `claim_entry`. No other customs procedures are modeled.

### 7.2 What EU/UK Requires

The UCC defines **multiple customs procedures** that fundamentally change how goods are treated:

| Procedure | UCC Code | Description | Business Case | Priority |
|-----------|---------|-------------|--------------|----------|
| **Release for free circulation** | 40 | Standard import | Default import | P0 |
| **Customs warehousing** | 71 | Store goods without paying duty | Deferred duty, large distributors | P1 |
| **Inward processing** | 51 | Import materials duty-free for processing/re-export | Manufacturing | P1 |
| **Outward processing** | 21 | Export EU goods for processing, re-import at reduced duty | Repairs, finishing | P2 |
| **Temporary admission** | 53 | Import goods temporarily (exhibitions, tools) | Events, equipment | P2 |
| **End-use** | 44 | Reduced/zero duty conditional on specific use | Aviation, shipping parts | P2 |
| **Transit (NCTS)** | T1/T2 | Move goods through EU without duties | Transshipment | P1 |
| **Free zone** | — | Import into designated free zone | Logistics hubs | P2 |
| **Procedure 42** | 42 | Import + immediate intra-EU dispatch (VAT exempt) | B2B cross-border | P1 |

#### New Computerised Transit System (NCTS)

Transit declarations (T1/T2) are managed through NCTS Phase 5:
- Departure office → transit → destination office
- Guarantee management for transit
- Real-time tracking of transit movements
- EU-wide system covering all member states + EFTA + Turkey + Western Balkans

### 7.3 What Already Exists

- `entry_type` field on `EntryFiling` (hardcoded to "01")
- No procedure code concept, no customs warehousing, no IP/OP, no transit

### 7.4 What Needs to Be Built

#### Data Model (P1)
- `procedure_code` field on declaration model
- `CustomsWarehouse` model (authorization, warehouse ID, goods accounting)
- `InwardProcessingAuthorization` model
- `TransitDeclaration` model (departure/destination offices, guarantee, seals)

#### Backend (P1)
- Procedure code selection logic with eligibility checks
- IP/OP duty calculation adjustments in tariff engine
- Transit declaration filing and tracking
- Customs warehouse inventory tracking
- Guarantee management for procedures requiring financial security

#### Frontend (P1)
- Procedure selector in declaration workflow
- Transit tracking visualization
- Warehouse management dashboard

**Priority: P1** — free circulation (Procedure 40) is sufficient for MVP; other procedures unlock significant business value.

---

## 8. UK Post-Brexit Specifics

### 8.1 Current State

The UK is treated as a generic destination in the tariff engine — no UK-specific logic exists anywhere.

### 8.2 What UK Requires

#### Customs Declaration Service (CDS)

| Feature | Description | Priority |
|---------|-------------|----------|
| CDS declaration model | 100+ Data Elements (DEs), distinct from EU SAD | P0 |
| CHIEF/CDS migration | CHIEF is fully retired — CDS only | P0 |
| UK-specific procedure codes | e.g., 40 00 C10, 71 00 001 | P0 |
| Route Assessment (ALVS) | Automated risk channel assignment | P1 |
| Delayed declarations (Frontier Model) | Full or simplified frontier declaration | P1 |

#### UK Global Tariff

- UK has its own tariff schedule (diverged from EU CET since Jan 2021)
- Many goods have lower rates than the EU (e.g., UK simplified tariff on clothing)
- **Need separate `UKTaxRegime` class** — cannot reuse `EUTaxRegime`
- UK does not have Section 301/232 equivalents but has trade remedies (AD/CVD)

#### Windsor Framework (Northern Ireland)

- Northern Ireland follows EU customs rules for goods (EU Single Market for goods)
- GB → NI movements may require customs declarations
- "Not At Risk" determination: goods staying in NI → UK duty rates; goods potentially moving to EU → EU duty rates
- Green lane (internal UK trade) vs red lane (EU-bound goods)
- **This is one of the most complex customs arrangements in the world**

#### UK FTAs

- UK has rolled over most EU FTAs + negotiated new ones (Australia, NZ, CPTPP)
- Different rules of origin from EU FTAs (even where agreements exist with same partners)
- UK DCTS replaces EU GSP

### 8.3 What Needs to Be Built

#### Data Model (P0)
- `UKTaxRegime` class (separate from EU) with UK Global Tariff rates
- UK CDS data element model
- Northern Ireland / Windsor Framework flags on shipments

#### Backend (P0)
- UK tariff computation (separate from EU — different rates)
- CDS declaration type mapping
- Windsor Framework routing logic (NI → EU rules vs GB rules)
- UK FTA preference lookup

#### Frontend (P1)
- UK CDS-specific declaration screens
- Windsor Framework indicator for NI-bound goods
- UK-specific tariff display (differentiated from EU)

**Priority: P0 for tariff separation, P1 for CDS declaration details.**

---

## 9. Simulation Actor Needs

### 9.1 What the US System Does

The US simulation has a rich actor ecosystem:

| Actor | File | Purpose |
|-------|------|---------|
| `CustomsActor` | `customs.py` | STP vs manual screening, corridor scrutiny, holds |
| `CBPAuthorityActor` | `cbp_authority.py` | Entry response: accept/CF-28/CF-29/exam/reject |
| `PGAActor` | `pga.py` | FDA/EPA/CPSC/NHTSA agency reviews |
| `ISFActor` | `isf.py` | ISF 10+2 filing deadlines and penalties |
| `ComplianceEngineActor` | `compliance_engine.py` | Auto-populates analysis JSONB |
| `CageActor` | `cage.py` | Container/cage management at port |
| `BrokerSimActor` | `broker_sim.py` | Simulated broker actions |
| `ResolutionActor` | `resolution.py` | Hold resolution workflows |
| `ShipperActor` | `shipper.py` | Shipper booking generation |
| `CarrierActor` | `carrier.py` | Transit simulation |
| `DemurrageActor` | `demurrage.py` | Storage/demurrage charges |
| `TerminalActor` | `terminal.py` | Terminal processing |
| `DisruptionActor` | `disruption.py` | Trade disruption events |
| `FinancialActor` | `financial.py` | Financial calculations |
| `DocumentsActor` | `documents.py` | Document generation/tracking |
| `ShipperResponseActor` | `shipper_response.py` | Shipper response simulation |
| `ConsolidatorActor` | `consolidator.py` | Consolidation management |
| `PreclearanceActor` | `preclearance.py` | Pre-clearance screening |

### 9.2 What EU/UK Simulation Actors Would Look Like

#### EUCustomsAuthorityActor (P1)

Replaces `CBPAuthorityActor` for EU declarations:

| Feature | US (CBP) | EU Equivalent |
|---------|----------|---------------|
| Risk assessment | Random + corridor scrutiny | **AEO green lane** (AEO-F → ~95% auto-release) vs standard (stepped risk) |
| Documentary check | CF-28 | Pre-release query (varies by MS customs system) |
| Physical inspection | VACIS/NII, intensive, tailgate | Scanner, physical exam, sampling |
| Rate adjustment | CF-29 notice | Post-clearance demand / customs debt notification |
| Protest | Broker files protest | Administrative appeal → tribunal |
| Liquidation | Auto-liquidation at 314 days | Customs debt calculation at release |

**AEO Differentiation Logic:**
```
AEO-F holder → 95% auto-release, 4% documentary check, 1% physical
AEO-C holder → 85% auto-release, 10% documentary, 5% physical
Standard operator → 60% auto-release, 25% documentary, 15% physical
New operator → 40% auto-release, 30% documentary, 30% physical
```

#### UKCDSAuthorityActor (P1)

- CDS-specific response handling
- Route assessment (red/orange/green channel)
- Border Force inspection events
- HMRC post-clearance audit simulation

#### EUPGAEquivalentActor (P1)

EU equivalent agencies for product compliance:

| US Agency | EU Equivalent | Jurisdiction |
|-----------|--------------|-------------|
| FDA (food) | EFSA / DG SANTE / National food authorities | Food safety |
| FDA (drugs) | EMA / National medicines agencies | Pharmaceuticals |
| EPA | ECHA (REACH) / National environment agencies | Chemicals |
| CPSC | EU Product Safety authorities (CE marking) | Consumer safety |
| NHTSA | EU Type Approval authorities | Vehicle safety |
| USDA/APHIS | EFSA / National plant health authorities | SPS controls |
| TTB | National excise authorities | Alcohol/tobacco |
| FCC | CE marking (EMC Directive) | Radio/EMC |

#### ICS2ENSActor (P1)

EU equivalent of `ISFActor`:
- ENS filing deadline tracking (before vessel/aircraft loading)
- PLACI screening results
- Do-Not-Load instruction simulation
- Multi-MS ENS filing for transshipment cargo
- ENS amendment handling

#### EUSanctionsActor (P1)

- EU Consolidated Financial Sanctions List screening
- EU dual-use export license verification
- UK OFSI screening
- Screening at declaration acceptance (not just at customs)

#### PostClearanceAuditActor (P2)

EU customs authorities perform significant post-clearance auditing:
- PCA targeting based on import patterns
- AEO monitoring and re-assessment
- Retrospective duty demands (up to 3 years)
- Customs debt recovery procedures

### 9.3 What Needs to Be Built

#### Config (P1)
- `EUCustomsConfig` with AEO-differentiated clearance rates
- `UKCDSConfig` with UK-specific parameters
- `ICS2Config` with ENS filing parameters
- EU corridors in `reference_data.py` (currently all corridors terminate at US)

#### Actors (P1)
- `EUCustomsAuthorityActor` — replaces CBP authority for EU-destined shipments
- `UKCDSAuthorityActor` — UK-specific customs handling
- `ICS2ENSActor` — pre-arrival security filing
- `EURegulatoryActor` — REACH, CE/UKCA, CBAM checks
- `EUSanctionsActor` — EU/UK sanctions screening

#### State Machine (P1)
- EU declaration lifecycle transitions (separate from US states, or unified with jurisdiction context)
- New states: `ens_filed`, `declaration_accepted`, `eu_query_pending`, `pca_selected`

**Priority: P1** — simulation is not required for the declaration filing MVP but essential for realistic broker training.

---

## 10. Agent Tools

### 10.1 What the US System Does

`tools.py` defines 12 agent tools, most with US-specific descriptions and behavior:

| Tool | US-Specific Elements |
|------|---------------------|
| `calculate_tariff` | Description mentions "Section 301, Section 232, AD/CVD, MPF, HMF" |
| `check_compliance` | References "UFLPA forced labor risk assessment" |
| `screen_entity` | Screens against "OFAC SDN, BIS Entity List, DPL" |
| `identify_documents` | Returns US document requirements |
| `resolve_shipment_hold` | US hold concepts |
| `classify_product` | References "HTS" (US nomenclature); HS is international |

### 10.2 What EU/UK Requires

#### Modified Tool Descriptions

All tool descriptions need jurisdiction-awareness:

| Tool | EU/UK Adaptation |
|------|-----------------|
| `calculate_tariff` | Should mention "Common External Tariff, VAT, excise duties, CBAM" for EU destinations |
| `check_compliance` | Should check REACH, CE/UKCA, ICS2, EU sanctions list, CBAM requirements |
| `screen_entity` | Should screen against EU Consolidated List, UK OFSI, EU dual-use list |
| `identify_documents` | Should return EUR.1, SAD, DV1, ENS, REACH docs for EU destinations |
| `classify_product` | Should reference TARIC (EU) / UK Trade Tariff for EU/UK destinations |
| `get_regulatory_signals` | Should include EU regulatory feeds (Official Journal, DG Trade) |

#### New EU/UK-Specific Tools

| Tool | Purpose | Priority |
|------|---------|----------|
| `check_eori_validity` | Validate EORI number against EU database | P0 |
| `lookup_preference_rate` | Find FTA preferential rate for EU/UK FTAs | P1 |
| `check_tariff_quota` | Check availability of tariff quota | P1 |
| `validate_origin_rules` | Check if goods qualify for preferential origin | P1 |
| `check_cbam_obligation` | Determine CBAM reporting requirements | P1 |
| `check_reach_status` | Verify REACH pre-registration/registration | P1 |
| `file_ens` | Submit Entry Summary Declaration (ICS2) | P1 |
| `lookup_customs_procedure` | Determine eligible customs procedures | P1 |

### 10.3 What Needs to Be Built

#### Backend (P1)
- Make tool descriptions dynamically context-aware based on shipment destination
- Add EU/UK screening list references to `screen_entity`
- Create new EU-specific tool implementations
- Update agent system prompts to include EU/UK regulatory knowledge

**Priority: P1** — the agent works but gives US-specific advice for EU-destined shipments.

---

## 11. Frontend Broker Surface

### 11.1 What the US System Has

The entire broker surface is US-hardcoded:

| Component | US-Specific Elements |
|-----------|---------------------|
| `BrokerNav.tsx` | "CBP Responses" link, CBP response badge count |
| `BrokerDashboard.tsx` | Pipeline stages: "Filed" → "CBP Response" → "Released"; StatusBadge maps: `cf28_pending`, `cf29_pending` |
| `CBPResponses.tsx` | Entire screen is CBP-specific: "CF-28 Request for Information", "CF-29 Notice of Action" |
| `EntryDetail.tsx` | US checklist (8 items), US acronym map (CBP, UFLPA, PGA), transport mode icons |
| `BrokerQueue.tsx` | `cbp_response` JSONB rendering |
| `Communications.tsx` | References to CBP |
| `BrokerAssistant.tsx` | Agent prompt context is US-focused |
| `brokerStore.ts` | All types reference CBP concepts |

### 11.2 What EU/UK Requires

#### Navigation
- Replace "CBP Responses" with jurisdiction-aware "Customs Authority Responses" (or tabbed: US/EU/UK)
- Add "Declarations" tab for EU/UK declarations
- Add "ENS Filings" tab for ICS2

#### Dashboard
- Pipeline stages must be jurisdiction-aware:
  - EU: Pre-Declaration → Declaration Accepted → Risk Assessment → Release/Query/Examination
  - UK: Pre-Filing → Submitted to CDS → Route Assessment → Release/Query
- Status badges for EU/UK states (not CF-28/CF-29)

#### Entry/Declaration Detail
- Dynamic checklist based on destination jurisdiction
- Procedure code display for EU declarations
- MRN instead of US entry number
- EORI display for declarant and representative
- AEO status badge
- EU FTA preference claim indicator

#### Response Handling
- EU customs query response (replaces CF-28 modal)
- EU customs debt notification (replaces CF-29 modal)
- UK CDS query response

### 11.3 What Needs to Be Built

#### Frontend (P0)
- Jurisdiction detection from shipment destination → dynamic UI rendering
- Abstract `CustomsAuthorityResponses` component (CF-28/CF-29 become one of several jurisdiction-specific variants)
- EU checklist component
- UK CDS checklist component
- Shared `DeclarationDetail` component that branches by jurisdiction
- MRN/entry number abstraction layer

#### Types (P0)
- Extend TypeScript types for EU/UK declaration models
- Jurisdiction-aware alert types
- EU-specific filing status enum

**Priority: P0** — the frontend is the primary user-facing gap. Currently 111 references to CBP-specific concepts across 7 broker surface files.

---

## 12. Implementation Roadmap

### Phase 0: Foundation (P0) — Unified Jurisdiction Framework

Build the abstraction layer that makes the entire platform jurisdiction-aware:

1. **Jurisdiction detection** — determine EU/UK/US from shipment destination
2. **Declaration model abstraction** — `EntryFiling` → generalized `CustomsDeclaration` supporting US entries, EU declarations, UK CDS declarations
3. **EORI validation** — mandatory identifier for EU/UK trade
4. **MRN generation** — for EU/UK declarations
5. **EU/UK tariff engine separation** — `UKTaxRegime` as separate from `EUTaxRegime`
6. **Jurisdiction-aware checklist** — `_compute_checklist_state` branches by destination
7. **Frontend jurisdiction routing** — detect destination and render appropriate UI

### Phase 1: EU Declaration Lifecycle

1. EU declaration state machine (parallels US entry filing states)
2. EU customs authority response handling
3. ENS (ICS2) filing model
4. EU document types and field schemas
5. EU FTA partner lookup and preference rates
6. PVA / Procedure 42 in tariff computation

### Phase 2: UK CDS Support

1. UK tariff engine with UK Global Tariff rates
2. CDS declaration model
3. Windsor Framework routing logic
4. UK FTA preferences (separate from EU)

### Phase 3: Simulation & Intelligence

1. EU/UK customs authority simulation actors
2. ICS2 ENS actor
3. EU sanctions screening
4. CBAM reporting module
5. Agent tool descriptions updated for multi-jurisdiction
6. EU/UK corridors in simulation reference data

### Phase 4: Advanced Procedures & Compliance

1. Customs warehousing workflow
2. Inward/outward processing
3. Transit (NCTS) declarations
4. REACH compliance checking
5. EU Deforestation Regulation
6. Post-clearance audit simulation

---

## Appendix A: Codebase Files Requiring Modification

| File | Change Type | Priority |
|------|------------|----------|
| `backend/app/knowledge/models/operational.py` | Add EORI, MRN, procedure_code fields; extend EntryFiling or create CustomsDeclaration | P0 |
| `backend/app/api/routes/broker.py` | Jurisdiction-aware checklist, declaration routes, FTA lookup | P0 |
| `backend/app/api/schemas/broker.py` | EU/UK declaration types, response types | P0 |
| `backend/app/engines/e2_tariff/engine.py` | Add UK regime dispatch; UK is not EU post-Brexit | P0 |
| `backend/app/engines/e2_tariff/regimes/eu.py` | Add preference rates, excise, PVA, Procedure 42 logic | P1 |
| `backend/app/engines/e2_tariff/regimes/uk.py` | **NEW** — UK Global Tariff regime | P0 |
| `backend/app/api/routes/documents.py` | `FTA_PARTNERS_EU`, `FTA_PARTNERS_UK`, EU doc field schemas | P1 |
| `backend/app/api/agent/tools.py` | Jurisdiction-aware tool descriptions, new EU tools | P1 |
| `backend/app/api/agent/prompts.py` | EU/UK regulatory knowledge in agent system prompt | P1 |
| `backend/app/simulation/state_machine.py` | EU declaration state transitions | P1 |
| `backend/app/simulation/reference_data.py` | EU corridors, EU products, EU port codes, EU customs offices | P1 |
| `backend/app/simulation/actors/cbp_authority.py` | Abstract to `CustomsAuthorityActor` base class | P1 |
| `backend/app/simulation/actors/eu_customs.py` | **NEW** — EU customs authority actor | P1 |
| `backend/app/simulation/actors/uk_cds.py` | **NEW** — UK CDS authority actor | P1 |
| `backend/app/simulation/actors/ics2.py` | **NEW** — ICS2 ENS actor | P1 |
| `backend/app/simulation/actors/pga.py` | Add EU equivalent agencies (EFSA, ECHA, etc.) | P1 |
| `backend/app/simulation/actors/isf.py` | Abstract ISF into pre-arrival security filing concept | P1 |
| `backend/app/simulation/config.py` | EU/UK customs configs | P1 |
| `backend/app/engines/e6_regulatory/engine.py` | EU regulatory signals (CBAM, EUDR, REACH) | P1 |
| `frontend/src/surfaces/broker/BrokerNav.tsx` | "Customs Authority Responses" (jurisdiction-aware) | P0 |
| `frontend/src/surfaces/broker/screens/BrokerDashboard.tsx` | Jurisdiction-aware pipeline stages, status badges | P0 |
| `frontend/src/surfaces/broker/screens/CBPResponses.tsx` | Abstract to `CustomsResponses.tsx` | P0 |
| `frontend/src/surfaces/broker/screens/EntryDetail.tsx` | Jurisdiction-aware checklist, MRN display, EORI | P0 |
| `frontend/src/surfaces/broker/screens/BrokerQueue.tsx` | EU/UK response rendering | P0 |
| `frontend/src/types.ts` or `frontend/src/types/` | EU/UK declaration types | P0 |
| `frontend/src/api/client.ts` | EU/UK declaration API calls | P0 |
| `frontend/src/store/brokerStore.ts` | Jurisdiction-aware state | P0 |

## Appendix B: Critical Technical Decisions

### 1. Unified vs Separate Declaration Models

**Option A: Extend `EntryFiling`** — Add optional EU/UK fields, use `declaration_type` discriminator
- Pro: Less migration, shared pipeline logic
- Con: Increasingly wide model, null-heavy for each jurisdiction

**Option B: Polymorphic `CustomsDeclaration` base** — Separate `USEntry`, `EUDeclaration`, `UKDeclaration`
- Pro: Clean separation, jurisdiction-specific validation
- Con: More complex queries, API surface duplication

**Recommendation: Option B** — the declaration workflows are fundamentally different (CF-28 vs EU query, MRN vs entry number, EORI vs EIN). A shared base class with jurisdiction-specific subtypes provides cleaner separation.

### 2. State Machine Strategy

**Option A: Single state machine with jurisdiction context** — Same states, different actors
**Option B: Separate state machines per jurisdiction** — Independent lifecycle definitions

**Recommendation: Option B** — the EU lifecycle (`declaration_accepted → risk_analysis → release/query/examination`) doesn't map 1:1 to the US lifecycle (`entry_filed → cf28_pending/cf29_pending/exam_scheduled`). Forcing them into one machine creates awkward mappings.

### 3. UK Tariff Engine Separation

The UK **must** have its own tax regime class. Post-Brexit, UK tariff rates have diverged from the EU CET. Using `EUTaxRegime` for UK destinations would produce incorrect duty calculations. The `TariffEngine._get_regime()` currently routes `GB` to `EU` via `EU_MEMBERS` (UK is NOT in `EU_MEMBERS`, so `GB` actually falls through to "unsupported destination"). This must be fixed immediately.

### 4. Frontend Architecture

**Recommendation: Jurisdiction adapter pattern** — A `useJurisdiction(destination)` hook returns the correct component variants, labels, checklist structure, and API endpoints. Each jurisdiction implements the same interface but with jurisdiction-specific rendering.

---

## Appendix C: EU Customs Declaration Lifecycle (for State Machine Design)

```
┌──────────────┐
│   Pre-filed   │  (Declaration lodged before goods arrive)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Accepted    │  (MRN assigned, formal acceptance)
└──────┬───────┘
       │
       ▼
┌──────────────────┐
│  Risk Assessment  │  (Automated + manual risk profiling)
└──────┬───────────┘
       │
       ├────► Green lane → RELEASE (auto-clearance)
       │
       ├────► Orange lane → DOCUMENTARY CHECK
       │                        │
       │                        ├──► Pass → RELEASE
       │                        └──► Query → CUSTOMS QUERY PENDING
       │                                       │
       │                                       ├──► Response received → re-assess
       │                                       └──► No response → CUSTOMS DEBT
       │
       └────► Red lane → PHYSICAL EXAMINATION
                            │
                            ├──► Pass → RELEASE
                            ├──► Issues found → CUSTOMS QUERY / SEIZURE
                            └──► Prohibited goods → DETENTION / DESTRUCTION
```

---

*This analysis covers the EU/UK gap comprehensively. The companion analyses for India, Brazil/LatAm, and China/APAC are produced separately by peer agents.*
