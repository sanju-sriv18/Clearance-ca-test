# India Customs Operations — Gap Analysis for First-Class Support

> **Prepared for**: Clearance Vibe Platform — Global Expansion
> **Benchmark**: US CBP support (current production)
> **Scope**: Import & Export clearance operations under Indian customs (CBIC/ICEGATE)
> **Date**: 2026-02-07

---

## Executive Summary

The Clearance Vibe platform currently provides **deep, first-class support for US CBP operations**: CF-28/CF-29 handling, Section 301/232/IEEPA surcharges, ISF filing, ACE entry submission, PGA routing, bond management, and a full broker simulation. India support is limited to a **basic tariff engine** (BCD + SWS + IGST) with ~55 seeded ITC-HS codes. To bring India to parity, the platform needs: an ICEGATE filing model, Bill of Entry / Shipping Bill workflows, India-specific regulatory bodies (BIS, FSSAI, CDSCO, WPC, DGFT), trade agreement engines (CEPA/CECA/ECTA), export incentive schemes (RoDTEP, MEIS, drawback), and a customs authority simulation actor replacing the US-specific CBPAuthorityActor.

---

## 1. ICEGATE & Filing System

### US Benchmark
- **ACE (Automated Commercial Environment)**: Full entry filing via `EntryFiling` model with `entry_number`, `filing_status`, `cbp_response` JSONB
- **Filing states**: `draft → pending_broker_approval → submitted → accepted/cf28_pending/cf29_pending/exam_scheduled → released/rejected`
- **CBPAuthorityActor** simulates realistic responses with risk scoring
- **ISF 10+2** (Importer Security Filing) via `ISFActor` for ocean shipments
- **ACAS** (Air Cargo Advance Screening) for air shipments
- **Entry Summary 7501** referenced in document requirements

### India Equivalent (What's Needed)
| Filing Type | India Term | Current State | Priority |
|---|---|---|---|
| Import entry | **Bill of Entry (BE)** | Not modeled | **P0** |
| Export entry | **Shipping Bill (SB)** | Not modeled | **P0** |
| Advance filing | **Prior Bill of Entry** | Not modeled | **P1** |
| Document upload | **eSanchit** (paperless) | Not modeled | **P1** |
| EDI filing | **ICES (Indian Customs EDI System)** | Not modeled | **P1** |
| Assessment | **Faceless Assessment (Turant Customs)** | Not modeled | **P2** |
| Cargo clearance target | **SWIFT (Single Window Interface for Facilitating Trade)** | Not modeled | **P2** |

### What Exists Today
- `IndiaTaxRegime` in `regimes/india.py` computes BCD + SWS + IGST — **tariff math only**
- `seed_in_tariff.py` seeds ~55 ITC-HS codes into `htsus_headings` with `jurisdiction = 'IN'`
- `TariffEngine` dispatches to India when `destination_country = "IN"`
- One corridor: `IN → US` (Nhava Sheva → Newark) in simulation reference data
- No `IN ← *` import corridors exist (India as destination is untested in simulation)

### What's Needed
1. **Bill of Entry model** — extends `EntryFiling` with India-specific fields:
   - `be_number` (auto-generated 7-digit), `be_type` (home consumption / warehousing / ex-bond)
   - `port_code` (Indian customs station code, not US CBP port code)
   - `iec_number`, `ad_code`, `gstin` of importer
   - `cha_license_number` (Customs House Agent — India's equivalent of US licensed broker)
   - `igcr_declaration` (IGCR = Import of Goods at Concessional Rate)
   - `esanchit_irn` (eSanchit Image Reference Number for uploaded documents)
2. **Shipping Bill model** — for export filings:
   - `sb_number`, `sb_type` (dutiable / free / drawback / EPCG / advance authorization)
   - `scroll_number`, `let_export_order_date`
   - `fob_value`, `freight`, `insurance`
   - `reward_scheme` (RoDTEP / drawback / MEIS)
3. **ICEGATE filing states** (replaces ACE states):
   - `draft → submitted_to_ices → first_check → second_check → assessed → out_of_charge → delivered`
   - `query_raised` (≈ CF-28), `speaking_order` (≈ CF-29)
   - `examination_ordered` (first check / second check)
4. **ICEGATEAuthorityActor** — simulation actor replacing `CBPAuthorityActor`:
   - Risk Management System (RMS) green/red/yellow routing
   - Faceless assessment (remote officer assignment)
   - Query vs. speaking order workflow
   - First Check (physical exam before assessment) vs. Second Check (exam after assessment)
5. **Import corridors**: Add `* → IN` corridors to `reference_data.py`:
   - `CN → IN` (Shanghai → Nhava Sheva/Mundra), `DE → IN` (Hamburg → Nhava Sheva)
   - `KR → IN` (Busan → Chennai), `JP → IN` (Yokohama → Nhava Sheva)
   - `US → IN` (Los Angeles → Nhava Sheva)

**Priority**: **P0** — Without filing models, India cannot function beyond tariff calculation.

---

## 2. Required Identifiers

### US Benchmark
- **EIN/Importer Number**: Implied via company data, not explicitly modeled
- **CBP Broker License**: `Broker.license_number` + `license_district` in operational model
- **Bond**: `bond_type` in `financials` JSONB (continuous / single_entry), with `BOND_TYPES` reference data
- **Port codes**: `US_PORT_CODES` dict (16 ports), used in `EntryFiling.port_of_entry`

### India Equivalent
| US Identifier | India Equivalent | Current State | Priority |
|---|---|---|---|
| EIN | **IEC (Import Export Code)** — 10-digit, mandatory for all import/export | Not modeled | **P0** |
| — | **AD Code (Authorized Dealer)** — bank code for forex transactions | Not modeled | **P0** |
| — | **GSTIN** — 15-digit GST registration | Not modeled | **P0** |
| — | **PAN** — linked to IEC | Not modeled | **P1** |
| Broker license | **CHA License** (Customs House Agent, now called Customs Broker under CBLR 2018) | Not modeled — `Broker.license_number` is US-specific district format | **P0** |
| — | **BIN (Business Identification Number)** — for SEZ/EOU units | Not modeled | **P2** |
| Port codes | **Indian Customs Station Codes** (INNSA1 = Nhava Sheva, INMAA1 = Chennai, etc.) | Not modeled — only US port codes exist | **P0** |

### What's Needed
1. Add `iec_number`, `ad_code`, `gstin`, `pan` to importer/company models
2. Add `IN_PORT_CODES` reference data:
   - `INNSA1` = Nhava Sheva (JNPT), `INBOM4` = Mumbai Air
   - `INMUN1` = Mundra, `INMAA1` = Chennai
   - `INDEL4` = Delhi Air (IGI), `INCCU1` = Kolkata
   - `INBLR4` = Bangalore Air, `INHYD4` = Hyderabad Air
   - `INTUT1` = Tuticorin, `INKOK1` = Kochi
3. Extend `Broker` model with `cblr_number` (Customs Broker License Regulations number) and `customs_station` assignment
4. India bond equivalent: **Bank Guarantee (BG)** / **surety bond** — different from US continuous/single-entry model

**Priority**: **P0** — IEC and GSTIN are mandatory for every Indian import/export transaction.

---

## 3. Document Types

### US Benchmark
The broker checklist (`_compute_checklist_state`) validates 8 document categories:
1. Commercial Invoice
2. Packing List
3. Bill of Lading / Airway Bill
4. HS Classification (data check)
5. Customs Valuation (data check)
6. Certificate of Origin (FTA-conditional)
7. PGA Documentation (conditional on PGA flags)
8. Bond Verification

`MODE_DOCUMENTS` in reference data defines mode-specific requirements (HAWB, MBL, ISF 10+2, PAPS, Entry Summary 7501).

### India Equivalent
| US Document | India Equivalent | Exists? | Priority |
|---|---|---|---|
| Commercial Invoice | **Commercial Invoice** (same) | Yes (shared) | — |
| Packing List | **Packing List** (same) | Yes (shared) | — |
| Bill of Lading | **Bill of Lading / AWB** (same) | Yes (shared) | — |
| Certificate of Origin | **Certificate of Origin** (for FTA: CEPA/CECA/ECTA certificates) | Partial — `FTA_PARTNERS_US` exists but no India FTA map | **P0** |
| Entry Summary (7501) | **Bill of Entry** (replaces 7501) | Not modeled | **P0** |
| ISF 10+2 | No India equivalent (India uses AMS/ACS for advance cargo info) | N/A | — |
| PGA docs | **BIS certificate**, **FSSAI license**, **Drug license**, **Phytosanitary cert**, **Fumigation cert**, **WPC ETA**, **Legal Metrology declaration** | Not modeled | **P1** |
| — | **Test reports** (for BIS-mandatory products: electronics, toys, helmets) | Not modeled | **P1** |
| — | **SCOMET end-user certificate** (strategic goods) | Not modeled | **P2** |
| — | **DGFT license** (restricted items) | Not modeled | **P1** |
| — | **Insurance certificate** (mandatory for CIF valuation) | Not modeled | **P1** |

### What's Needed
1. **India-specific checklist template** — replace US 8-item checklist:
   - Commercial Invoice, Packing List, BL/AWB (shared)
   - **Bill of Entry** (filing data check)
   - **IEC verification** (data check)
   - **IGST payment challan** or ITC claim reference
   - **BIS/FSSAI/WPC/CDSCO clearance** (conditional per HS code)
   - **Certificate of Origin** (conditional per FTA)
   - **DGFT license** (conditional per import policy)
   - **Test report** (conditional per BIS mandatory list)
2. **eSanchit document upload mapping** — each document type maps to an eSanchit document code
3. **`FTA_PARTNERS_IN`** reference map for India trade agreements (see Section 6)

**Priority**: **P0-P1** — Without an India checklist, the broker surface cannot validate India filings.

---

## 4. Regulatory Frameworks (PGA Equivalents)

### US Benchmark
- `PGA_TRIGGERS` maps FDA, EPA, CPSC, USDA, NHTSA → product categories
- `PGAActor` generates PGA flags during simulation
- PGA documentation appears in broker checklist
- FDA product codes with prior notice hours, inspection rates
- APHIS requirements (phytosanitary, wood packaging ISPM-15)
- TTB for alcohol/tobacco

### India Equivalent
| US Agency | India Equivalent | Scope | Current State | Priority |
|---|---|---|---|---|
| FDA | **FSSAI** (Food Safety and Standards Authority of India) | Food products, dietary supplements | Not modeled | **P0** |
| FDA (drugs) | **CDSCO** (Central Drugs Standard Control Organisation) | Pharmaceuticals, medical devices | Not modeled | **P1** |
| CPSC | **BIS** (Bureau of Indian Standards) | Electronics, toys, helmets, appliances — mandatory certification (CRS scheme) | Not modeled | **P0** |
| EPA | **MOEFCC** (Ministry of Environment) + **CPCB** (Central Pollution Control Board) | Hazardous waste, e-waste, chemicals | Not modeled | **P2** |
| NHTSA | **CMVR** (Central Motor Vehicle Rules) / **ARAI** | Vehicles and components | Not modeled | **P2** |
| FCC | **WPC** (Wireless Planning and Coordination Wing) | Wireless/radio equipment — ETA (Equipment Type Approval) | Not modeled | **P1** |
| — | **Legal Metrology** | Pre-packaged goods labeling requirements | Not modeled | **P2** |
| USDA/APHIS | **Plant Quarantine** (DPPQS) | Plants, seeds, phytosanitary requirements | Not modeled | **P2** |
| — | **DGFT** (Directorate General of Foreign Trade) | Import/export licensing, restricted items, SCOMET | Not modeled | **P1** |
| TTB | **State Excise** | Alcohol (state-level regulation in India) | Not modeled | **P2** |
| — | **AERB** (Atomic Energy Regulatory Board) | Radioactive materials | Not modeled | **P3** |
| — | **Textile Committee** | Mandatory marking for textile imports | Not modeled | **P2** |

### What's Needed
1. **`PGA_TRIGGERS_IN`** mapping India regulatory bodies to HS chapters:
   ```python
   PGA_TRIGGERS_IN = {
       "BIS": ["84", "85", "95", "87"],  # Electronics, toys, auto
       "FSSAI": ["01"-"24"],  # Food chapters
       "CDSCO": ["30", "90"],  # Pharma, medical devices
       "WPC": ["8517", "8525", "8527"],  # Wireless equipment
       "DGFT": ["*restricted*"],  # Per DGFT ITC-HS policy
       "Plant Quarantine": ["06", "07", "08", "12"],  # Plants, seeds
   }
   ```
2. **BIS CRS (Compulsory Registration Scheme)** data: ~400+ products require mandatory BIS certification before import. This is the single biggest non-tariff barrier for India imports.
3. **FSSAI license** requirements: Product approval, lab testing certificates, shelf life declarations
4. **DGFT import policy codes**: Free / Restricted / Prohibited per ITC-HS classification — this replaces the US concept of PGA flags with a more comprehensive licensing model
5. **IndiaRegulatoryActor** — simulation actor to apply BIS/FSSAI/DGFT holds

**Priority**: **P0** — BIS and FSSAI are the most common reasons for India import delays; without them the platform gives a false picture.

---

## 5. Tax Regime

### US Benchmark (USTaxRegime — 7 line items)
1. MFN Duty (HTSUS Column 1 General)
2. Section 301 (CN origin, USTR lists)
3. Section 232 (steel/aluminum)
4. IEEPA (country-level emergency tariffs)
5. AD/CVD (placeholder)
6. MPF (Merchandise Processing Fee)
7. HMF (Harbor Maintenance Fee)

Full DB-backed lookups: `_lookup_mfn_db`, `_check_301_db`, `_check_232_db`, `_lookup_ieepa_db`. Hardcoded fallbacks for ~20 golden-path HS codes.

### India Current State (IndiaTaxRegime — 3 line items)
1. BCD (Basic Customs Duty) — 17 hardcoded rates
2. SWS (Social Welfare Surcharge) — 10% of BCD
3. IGST — 18% standard / 28% luxury (2 chapters + 1 heading)

**No DB-backed lookups.** No origin-specific surcharges. No FTA preferential rates.

### India Full Tax Stack (What's Needed)
| Duty/Tax | Base | Current State | Priority |
|---|---|---|---|
| **BCD** | CIF value | Exists (17 codes hardcoded) | **P0** — expand to DB lookup |
| **SWS** | 10% of BCD | Exists | Done |
| **IGST** | CIF + BCD + SWS | Exists (18%/28%) | **P1** — add 5%, 12% slabs |
| **Anti-Dumping Duty (ADD)** | Per-unit or ad valorem | Not modeled | **P1** |
| **Safeguard Duty** | Ad valorem or specific | Not modeled | **P1** |
| **Countervailing Duty (CVD)** | To offset subsidies | Not modeled | **P1** |
| **AIDC (Agriculture Infrastructure & Development Cess)** | On specific items (gold, apple, alcohol, etc.) | Not modeled | **P1** |
| **Education Cess** | Largely subsumed by SWS, but some notifications still apply | Not modeled | **P2** |
| **Compensation Cess** | On luxury/demerit goods under GST (additional to IGST) | Not modeled | **P1** |
| **FTA Preferential BCD** | Reduced BCD per CEPA/CECA/ECTA rate schedule | Not modeled | **P0** |
| **Advance Ruling** | Binding classification/valuation ruling | Not modeled | **P3** |

### BCD Rate Discrepancy Warning
The hardcoded rates in `IndiaTaxRegime.BCD_RATES` and `seed_in_tariff.py` **disagree** in several places:
- Coffee `0901.21`: `BCD_RATES` says 30%, seed says 100%
- Laptops `8471.30`: `BCD_RATES` says 0% (ITA), seed says 15%
- Beauty `3304.99`: `BCD_RATES` says 20%, seed says 10%
- Toys `9503.00`: `BCD_RATES` says 20%, seed says 60%
- Bolts `7318.15`: `BCD_RATES` says 15%, seed says 10%
- Motor vehicles `8703.23`: `BCD_RATES` says 70%, seed says 100%

**This must be reconciled** — the seed file should be the source of truth, and `BCD_RATES` should fall back to DB (like `USTaxRegime._lookup_mfn_db`).

### What's Needed
1. **DB-backed BCD lookup** — add `_lookup_bcd_db()` method mirroring US `_lookup_mfn_db()`
2. **IGST slab expansion**: Add 0%, 5%, 12% slabs (not just 18%/28%). Map via HS code lookup.
3. **Compensation Cess**: Applies to motor vehicles (1%-22%), tobacco (cess per 1000 sticks), aerated beverages (12%), coal (₹400/tonne)
4. **AIDC**: Gold 2.5%, apple 35%, crude palm oil 7.5%, alcohol 100%, etc.
5. **Anti-Dumping duties**: India is one of the world's largest users of ADD — need `add_orders_in` table
6. **FTA preferential rates**: Require `origin_country` + `fta_certificate_type` to determine preferential BCD
7. **Reconcile BCD_RATES vs. seed data** — single source of truth

**Priority**: **P0** for DB lookup and rate reconciliation; **P1** for ADD/AIDC/compensation cess.

---

## 6. Trade Agreements

### US Benchmark
`FTA_PARTNERS_US` dict in `documents.py` maps country codes to agreement names. Used in:
- Broker checklist: Certificate of Origin required if FTA partner
- Compliance flags: FTA opportunity identification

### India Trade Agreements (Not Modeled)
| Agreement | Partners | Type | Tariff Impact | Priority |
|---|---|---|---|---|
| **India-ASEAN CECA** | BN, KH, ID, LA, MY, MM, PH, SG, TH, VN | Comprehensive Economic Cooperation | Reduced BCD for ~80% of tariff lines | **P0** |
| **India-Japan CEPA** | JP | Comprehensive Economic Partnership | Reduced/zero BCD on many industrial goods | **P0** |
| **India-Korea CEPA** | KR | Comprehensive Economic Partnership | Reduced BCD, special safeguards | **P0** |
| **India-UAE CEPA (2022)** | AE | Comprehensive Economic Partnership | Zero/reduced BCD for ~90% of lines | **P0** |
| **India-Australia ECTA (2022)** | AU | Economic Cooperation and Trade Agreement | Zero BCD for ~85% of tariff lines | **P0** |
| **India-EFTA TEPA (2024)** | CH, IS, LI, NO | Trade and Economic Partnership | Reduced BCD, investment provisions | **P1** |
| **SAFTA** | AF, BD, BT, LK, MV, NP, PK | South Asian Free Trade Area | Preferential rates, sensitive lists | **P1** |
| **India-Mercosur PTA** | AR, BR, PY, UY | Preferential Trade Agreement | Margin of preference on select lines | **P2** |
| **India-Chile PTA** | CL | Preferential Trade Agreement | Narrow product coverage | **P2** |
| **India-Singapore CECA** | SG | Comprehensive Economic Cooperation (bilateral, overlaps ASEAN) | Zero BCD on most lines | **P1** |
| **APTA (Asia-Pacific Trade Agreement)** | BD, CN, IN, KR, LK, MN | Formerly Bangkok Agreement | Margin of preference | **P2** |

### What's Needed
1. **`FTA_PARTNERS_IN`** dict mirroring `FTA_PARTNERS_US`:
   ```python
   FTA_PARTNERS_IN = {
       "JP": "India-Japan CEPA",
       "KR": "India-Korea CEPA",
       "AE": "India-UAE CEPA",
       "AU": "India-Australia ECTA",
       "SG": "India-Singapore CECA / ASEAN",
       "TH": "India-ASEAN CECA",
       "MY": "India-ASEAN CECA",
       "ID": "India-ASEAN CECA",
       "VN": "India-ASEAN CECA",
       "PH": "India-ASEAN CECA",
       "CH": "India-EFTA TEPA",
       "NO": "India-EFTA TEPA",
       # SAFTA members...
   }
   ```
2. **Preferential rate schedule tables** — each FTA has its own tariff concession schedule (often thousands of lines). Need `fta_preferential_rates` DB table.
3. **Rules of Origin (RoO)** — each FTA has different RoO criteria (value addition %, change of tariff heading, product-specific rules). This determines if preferential rate applies.
4. **Certificate of Origin types**: Form AI (ASEAN), Form AIJCEPA (Japan), Form AIKFTA (Korea), Form D (SAFTA), specific forms for UAE CEPA and Australia ECTA.

**Priority**: **P0** — FTA utilization is the #1 cost-saving opportunity for India importers.

---

## 7. Customs Procedures

### US Benchmark
- **Entry types**: Formal / Informal (modeled via `entry_type` parameter)
- **Bond types**: Continuous / Single Entry (modeled in `BOND_TYPES` reference data)
- **Filing workflow**: Fully modeled in `state_machine.py` and `CBPAuthorityActor`
- **Protest mechanism**: `protest_filed` state with CBP adjudication

### India Customs Procedures (Not Modeled)
| Procedure | Description | US Equivalent | Priority |
|---|---|---|---|
| **Home Consumption (BE Type)** | Standard import clearance for domestic use | Formal Entry (consumption) | **P0** |
| **Warehousing (Section 49/59)** | Import into bonded warehouse without duty payment | Warehouse Entry | **P1** |
| **Ex-Bond Clearance** | Clear goods from bonded warehouse (pay duty at removal) | Withdrawal from Warehouse | **P1** |
| **Re-import** | Duty exemption on re-imported Indian goods (Section 20) | TIB / ATA Carnet return | **P2** |
| **Re-export** | Return imported goods without consumption | Re-export entry | **P2** |
| **MOOWR** | Manufacturing and Other Operations in Warehouse (2019 regs) | FTZ manufacturing | **P2** |
| **SEZ Operations** | Special Economic Zone imports (duty-free with conditions) | FTZ entry | **P2** |
| **EPCG Scheme** | Import capital goods at zero/reduced duty against export obligation | — (no US equivalent) | **P1** |
| **Advance Authorization** | Duty-free import of inputs for export production | — | **P1** |
| **DFIA** | Duty Free Import Authorization (transferable) | — | **P2** |

### What's Needed
1. **Bill of Entry types enum**: `home_consumption`, `warehousing`, `ex_bond`, `reimport`
2. **Shipping Bill types enum**: `dutiable`, `free_shipping_bill`, `drawback`, `epcg`, `advance_auth`, `dfia`
3. **Bonded warehouse model**: Track warehouse licenses, bond amounts, duty-deferred inventory
4. **EPCG/Advance Auth tracking**: Export obligation monitoring (value + quantity + time period)

**Priority**: **P0** for home consumption; **P1** for warehousing + EPCG/Advance Auth.

---

## 8. Export-Specific Features

### US Benchmark
- **AES (Automated Export System)** referenced in `TERRITORY_FILINGS`
- No deep export filing workflow — platform is import-focused

### India Export Features (Not Modeled)
| Feature | Description | Priority |
|---|---|---|
| **Shipping Bill filing** | Export declaration via ICES | **P0** |
| **Let Export Order (LEO)** | Customs permission to export | **P0** |
| **IGST refund on exports** | Refund of IGST paid on exported goods | **P1** |
| **Duty Drawback** | Refund of customs duties on imported inputs used in exports (All Industry Rates + Brand Rates) | **P1** |
| **RoDTEP** | Remission of Duties and Taxes on Exported Products — replaces MEIS | **P1** |
| **LUT (Letter of Undertaking)** | Export without IGST payment (bond-free since 2017) — most exporters use this | **P1** |
| **EGM (Export General Manifest)** | Carrier files after vessel departure | **P2** |
| **Scroll** | Duty drawback disbursement tracking | **P2** |
| **GST refund for zero-rated exports** | Accumulated ITC refund for zero-rated supplies | **P2** |

### What's Needed
1. **Export filing workflow**: `draft → submitted → leo_granted → shipped → egm_filed`
2. **Drawback calculation engine**: All Industry Rates per tariff item, brand rate applications
3. **RoDTEP rate table**: Per-product remission rates (published by DGFT)
4. **LUT management**: Track LUT validity (annual), bond amount if LUT not available
5. **IGST refund tracking**: Filed vs. disbursed, deficiency memos

**Priority**: **P0** for basic Shipping Bill; **P1** for drawback/RoDTEP.

---

## 9. Risk Management System (RMS)

### US Benchmark
- `CBPAuthorityActor._calculate_risk_score()` — 5 risk factors (origin, value, HS chapter, AD/CVD, first-time importer)
- Risk score modifies accept/CF28/CF29/exam/reject probabilities
- Hold codes: `CBP_HOLD_CODES` (1H, 7H, CET, ADD, CVD, MEX, AGR, FDP, EPA, TSC, NHT)

### India RMS Equivalent
| US Concept | India Equivalent | Description | Priority |
|---|---|---|---|
| Risk scoring | **RMS (Risk Management System)** | CBIC's automated risk assessment; assigns green/red/yellow channel | **P0** |
| Green channel | **Green channel / facilitated** | Out-of-charge without assessment — ~70% of BEs | **P0** |
| Yellow channel | **First Check Assessment** | Document examination by assessing officer (may be remote via Turant Customs) | **P0** |
| Red channel | **Second Check / Physical Examination** | Physical inspection + assessment | **P0** |
| CF-28 | **Query raised by assessing officer** | Request for information/documents | **P0** |
| CF-29 | **Speaking Order** | Reasoned order on classification/valuation dispute | **P1** |
| Protest (19.1519) | **Appeal to Commissioner (Appeals)** → CESTAT → High Court | Multi-tier appeal mechanism | **P2** |
| Hold codes | **No standardized hold codes** — holds are officer-specific notes | — | **P2** |
| Faceless assessment | **Turant Customs** | Assessment done by officers not at port of import (reduces corruption) | **P2** |

### What's Needed
1. **`ICEGATEAuthorityActor`** replacing `CBPAuthorityActor`:
   - RMS channel assignment: green (70%), yellow (20%), red (10%) — percentages vary by importer history
   - First Check vs. Second Check branching
   - Query workflow (30-day response window, similar to CF-28)
   - Out-of-charge order (India's "release")
   - Speaking order workflow (India's CF-29 equivalent)
2. **Risk factors specific to India**:
   - Import history of IEC holder (new IEC = higher risk)
   - HS chapters with high evasion risk (textiles, electronics, gold)
   - Country of origin (CN goods face more scrutiny)
   - Value vs. contemporaneous import data (under-invoicing detection)
   - BIS/FSSAI compliance status
3. **Assessment officer model**: Faceless assessment means the assessing officer may be at any customs station, not at the port of import

**Priority**: **P0** — This is the heart of India customs clearance.

---

## 10. Unique India Considerations

### Simulation & Platform Gaps

| Feature | Description | Exists? | Priority |
|---|---|---|---|
| **Indian customs ports** | Nhava Sheva, Mundra, Chennai, Delhi Air, etc. in reference data | Only Nhava Sheva (as export port) | **P0** |
| **Indian carriers** | Shipping Corporation of India, Air India Cargo, local freight forwarders | Not modeled | **P2** |
| **INR currency** | India tariff computations in INR, not USD | Not modeled — engine uses USD only | **P1** |
| **Customs duty payment** | E-payment via ICEGATE (NEFT/RTGS/debit) with TR-6 challan | Not modeled | **P1** |
| **IGST Input Tax Credit** | Importer claims IGST paid at customs as ITC on GST returns | Not modeled | **P1** |
| **Seasonal events** | Diwali (exists in reference data), but also: Eid, monsoon port disruptions, fiscal year-end rush (March) | Partial — Diwali exists | **P1** |
| **DPD (Direct Port Delivery)** | Select importers clear goods directly at port without CFS | Not modeled | **P2** |
| **SWIFT targets** | Cargo release time targets: 24 hrs (sea), 12 hrs (air) for green channel | Not modeled | **P2** |
| **Customs audit (post-clearance)** | CBIC can audit imports within 2 years of clearance | Not modeled | **P3** |
| **E-Sanchit mandatory** | All supporting documents must be uploaded digitally | Not modeled | **P1** |

### Frontend Broker Surface Gaps

The current broker screens are **100% US-CBP-centric**:
- `CBPResponses.tsx` — references CF-28/CF-29 exclusively
- `BrokerNav.tsx` — "CBP Responses" nav label
- `EntryDetail.tsx` — "Submit to CBP" button, bond verification with US bond types
- `BrokerQueue.tsx` — `cf28_pending`/`cf29_pending` status badges
- `BrokerAssistant.tsx` — "Draft CF-28 response" prompt

**All of these need jurisdiction-conditional rendering** or India-specific equivalents:
- "CBP Responses" → "Customs Authority Responses" (generic) or "ICEGATE Responses" (India)
- "Submit to CBP" → "Submit to ICEGATE" or "File Bill of Entry"
- CF-28 → "Assessment Query"
- CF-29 → "Speaking Order"
- Bond verification → Bank Guarantee verification
- Port selector → Indian customs station selector

---

## Priority Summary

### P0 — Must Have for MVP India Support
1. Bill of Entry / Shipping Bill data models
2. ICEGATE filing state machine (replaces ACE states)
3. ICEGATEAuthorityActor (replaces CBPAuthorityActor)
4. IEC / GSTIN / AD Code identifier fields
5. India port codes reference data
6. BCD DB-backed lookup + rate reconciliation
7. `FTA_PARTNERS_IN` + Certificate of Origin for India FTAs
8. BIS / FSSAI regulatory body triggers
9. RMS channel routing (green/yellow/red)
10. Import corridors (→ India) in simulation reference data

### P1 — Required for Production-Quality Support
1. IGST slab expansion (0/5/12/18/28%)
2. Compensation Cess, AIDC
3. Anti-Dumping Duty table (India is a heavy ADD user)
4. EPCG / Advance Authorization scheme tracking
5. Duty drawback + RoDTEP calculation
6. INR currency support in tariff engine
7. DGFT import policy (Free/Restricted/Prohibited)
8. WPC/CDSCO regulatory actors
9. eSanchit document upload workflow
10. Prior Bill of Entry (advance filing)
11. LUT / IGST refund tracking for exports
12. Jurisdiction-conditional frontend labels (CBP → ICEGATE)

### P2 — Nice to Have for Comprehensive Support
1. Warehousing / ex-bond clearance workflows
2. MOOWR / SEZ operations
3. Faceless assessment (Turant Customs) simulation
4. SWIFT cargo clearance time tracking
5. Direct Port Delivery (DPD) support
6. Legal Metrology compliance
7. Indian carrier reference data
8. Post-clearance customs audit modeling
9. SAFTA / Mercosur / Chile PTA preferential rates
10. BIN for SEZ/EOU units

### P3 — Future Enhancement
1. Advance ruling management
2. AERB (radioactive materials)
3. Multi-tier appeal mechanism (Commissioner → CESTAT → HC)
4. Education cess legacy notifications
5. Customs audit (2-year post-clearance window)

---

## Estimated Effort

| Phase | Scope | Estimated Complexity |
|---|---|---|
| **Phase 1**: Data models + tariff | Bill of Entry model, BCD DB lookup, FTA map, rate reconciliation | Medium — ~2-3 weeks |
| **Phase 2**: Filing state machine + actor | ICEGATE states, ICEGATEAuthorityActor, RMS routing | Large — ~3-4 weeks |
| **Phase 3**: Regulatory bodies | BIS/FSSAI/DGFT triggers, PGA_TRIGGERS_IN, import policy engine | Medium — ~2-3 weeks |
| **Phase 4**: Export support | Shipping Bill, drawback, RoDTEP, LUT | Medium — ~2-3 weeks |
| **Phase 5**: Frontend adaptation | Jurisdiction-conditional broker screens, India checklist, ICEGATE UI | Large — ~3-4 weeks |
| **Phase 6**: Simulation + corridors | Import corridors, Indian ports, seasonal events, disruptions | Small — ~1-2 weeks |

**Total estimated**: ~13-19 weeks for full P0+P1 coverage, bringing India to parity with US CBP support.
