# Jurisdiction Deep-Dive Review — Non-US Customs Requirements

**Reviewer:** Licensed Customs Broker / International Trade Compliance Specialist (15 years, multi-jurisdiction)
**Date:** 2026-02-08
**Document Reviewed:** `docs/architecture-clearance-platform.md` (v4)
**Prior Review Referenced:** `docs/review-broker-expert.md` (Section 5.2)

---

## Executive Summary

The architecture document is **deeply US-centric**. The 13-domain model, event choreography, and physical entity hierarchy are well-designed foundations — but the entity models, status machines, filing workflows, duty structures, and document requirements are modeled almost exclusively around US CBP processes. Non-US jurisdictions are treated as configuration variants via `jurisdiction_config: {}?` JSONB blobs, when in reality they require **fundamentally different entity structures, status lifecycles, filing workflows, and party models**.

This is not a matter of extending the US model to other countries. Each jurisdiction has:
- A **different electronic system** with different message formats and integration protocols
- A **different filing workflow** (some two-step, some single-step, some pre-arrival mandatory)
- A **different status machine** (Brazil has 4 channels, India has RMS facilitation, EU has NCTS transit)
- A **different duty/tax structure** (cascading taxes in Brazil, GST in India, VAT+customs duty in EU)
- **Different party registration requirements** (EORI, RADAR, IEC, RFC — none optional)
- **Different document hierarchies** (NF-e in Brazil, ENS in EU, Bill of Entry in India)

**The architecture's string-based `jurisdiction` field + JSONB `jurisdiction_config` approach is NOT scalable.** It will become an unmanageable bag of exceptions. The platform needs a **jurisdiction abstraction layer** — a pluggable adapter pattern where each jurisdiction defines its own filing workflow, status machine, document requirements, duty calculation formula, and party registration rules.

Below I analyze each jurisdiction in detail, then provide architectural recommendations.

---

## 1. European Union (EU)

### 1.1 Customs Authority System

**Primary system:** ICS2 (Import Control System 2) for pre-arrival security data, plus **national declaration systems** per member state.

| Member State | Declaration System | Notes |
|-------------|-------------------|-------|
| Germany | ATLAS (Automatisiertes Tarif- und Lokales Zollabwicklungssystem) | XML-based, strict formatting |
| France | DELTA-G / DELTA-C / DELTA-T | DELTA-G for goods, DELTA-C for community transit, DELTA-T for transit |
| Netherlands | AGS (Automated System for Customs Declaration) / DMS (Declaration Management System) | DMS replacing AGS |
| Belgium | PLDA (Paperless Douane en Accijnzen) | Integrated with port of Antwerp systems |
| Italy | AIDA (Automazione Integrata Dogane e Accise) | |
| Spain | AAEE (Administración de Aduanas e Impuestos Especiales) | |

**Critical architecture impact:** There is **no single EU declaration system**. A platform operating across the EU must integrate with EACH member state's national system. The architecture's `jurisdiction: "EU"` is an oversimplification — it should be `jurisdiction: "EU-DE"`, `"EU-FR"`, `"EU-NL"`, etc. The legal framework (UCC) is common, but the electronic systems are national.

### 1.2 Filing Requirements

**ICS2 (mandatory since Release 3, fully operational 2024-2025 for all modes):**

| Requirement | Details |
|-------------|---------|
| **Filing type** | ENS (Entry Summary Declaration) — pre-arrival security data |
| **Who files** | Carrier (air/ocean), postal operator, express carrier, or their agents |
| **NOT the broker** | ENS is a CARRIER obligation, not a customs broker obligation |
| **Deadline (air)** | Before loading at origin airport (PLACI — Pre-Loading Advance Cargo Information) |
| **Deadline (ocean)** | 24 hours before loading at port of departure (deep sea), 4 hours for short sea |
| **Deadline (rail)** | 2 hours before arrival at EU entry point |
| **Deadline (road)** | 1 hour before arrival at EU entry point |
| **Data elements** | 6-digit HS code, goods description, consignor, consignee, weight, package count, transport document number |
| **Response** | Risk assessment result: "Do not load" / "Report required" / "No action" |

**Import declaration (post-arrival):**

| Dataset | When Used | Data Elements |
|---------|-----------|---------------|
| **H1** | Standard import declaration (free circulation) | ~54 data elements |
| **H2** | Free circulation with reduced dataset (goods < €22 - threshold abolished 2021) | Reduced set |
| **H3** | Temporary storage declaration | Limited set |
| **H4** | Customs warehousing | Extended set |
| **H5** | Temporary admission | Extended set |
| **H6** | Low-value consignment (IOSS < €150) | Simplified set |
| **H7** | Super-reduced dataset for <€150 postal/express | Minimal set |

**Two-step process (YES, but different from US):**
- EU allows **simplified declarations** (incomplete declaration filed first, supplementary declaration within 10 days)
- **Entry in Declarant's Records (EIDR):** Goods released based on a record entry; formal declaration filed within specified period
- This is structurally different from US entry/release (3461) + entry summary (7501)

### 1.3 Document Hierarchy

| Document | EU Name | US Equivalent | Notes |
|----------|---------|---------------|-------|
| Pre-arrival security filing | ENS (Entry Summary Declaration) | ISF 10+2 / ACAS | Filed by carrier, NOT broker |
| Import declaration | SAD (Single Administrative Document) / electronic equivalent | Entry Summary (CBP 7501) | Filed via national system |
| Transit document | T1 (external) / T2 (internal) | In-bond (CBP 7512) | Via NCTS system |
| Export declaration | ECS (Export Control System) | AES/SED | |
| AEO authorization | AEO-C / AEO-S / AEO-F | C-TPAT | Mutual recognition with C-TPAT |
| Origin certificate | EUR.1, EUR-MED, Form A | USMCA Certificate | For preferential tariff rates |

**Architecture gap:** The Document Management domain's `document_type` enum does not include ENS, SAD, T1/T2, NCTS declaration, or EUR.1 certificates.

### 1.4 Status Lifecycle

**EU import declaration processing statuses:**

```
Declaration Submitted
    |
    v
Declaration Registered (assigned MRN - Master Reference Number)
    |
    v
Risk Analysis (automated, similar to US STP)
    |
    +---> Released (no intervention) ≈ 70-80% of declarations
    |
    +---> Document Check (officer reviews documents)
    |         |
    |         +---> Released
    |         +---> Physical Examination ordered
    |
    +---> Physical Examination
    |         |
    |         +---> Released
    |         +---> Goods Seized / Detained
    |
    +---> Under Control (customs supervision pending)
    |
    v
Customs Released (goods enter free circulation)
    |
    v
Customs Debt Confirmed (duty/VAT amount finalized)
```

**Key difference from US:** EU does not have a separate "liquidation" period. Duty is assessed and confirmed at the time of release (or shortly after for simplified procedures). There is no 314-day liquidation timeline equivalent.

**NCTS Transit statuses (T1/T2):**

```
Transit Declaration Submitted
    |
    v
MRN Assigned
    |
    v
Goods Released for Transit
    |
    v
In Transit (between offices)
    |
    v
Arrived at Destination Office
    |
    v
Control Results Received
    |
    v
Transit Procedure Ended / Write-Off
```

### 1.5 Duty/Tax Structure

**EU duty calculation is fundamentally different from US:**

```
Customs Value (Transaction Value, typically CIF)
    x Customs Duty Rate (from Common Customs Tariff)
    = Customs Duty
    + Anti-dumping duty (if applicable)
    + Countervailing duty (if applicable)
    = Total Customs Charges

(Customs Value + Total Customs Charges)
    x VAT Rate (member state specific)
    = Import VAT

Total = Customs Duty + AD/CVD + Import VAT
```

**VAT rates vary by member state:**

| Country | Standard VAT | Reduced VAT |
|---------|-------------|-------------|
| Germany | 19% | 7% |
| France | 20% | 5.5%, 10% |
| Netherlands | 21% | 9% |
| Italy | 22% | 4%, 5%, 10% |
| Spain | 21% | 4%, 10% |
| Ireland | 23% | 9%, 13.5% |
| Luxembourg | 17% | 8% |

**Architecture impact:** The TariffCalculation model needs member-state-specific VAT rates. `jurisdiction: "EU"` is insufficient — must be `"EU-DE"`, `"EU-FR"`, etc. The `duty_line_items` array must support VAT as a separate line item (not part of customs duty).

**No MPF/HMF equivalent.** EU does not charge Merchandise Processing Fee or Harbor Maintenance Fee. These are US-specific. The architecture's `mpf` and `hmf` fields on FinancialRecord are US-only.

### 1.6 Party Requirements

| Registration | Who Needs It | Format | Notes |
|-------------|-------------|--------|-------|
| **EORI** (Economic Operators Registration and Identification) | ALL parties in any EU customs transaction | Country code + up to 15 alphanumeric (e.g., `DE123456789012345`) | Mandatory for import/export declarations, ENS, transit |
| **AEO** (Authorized Economic Operator) | Voluntary — trusted traders | EORI-based | AEOC (customs simplifications), AEOS (security), combined |

**The Product model, Shipment model, and Broker model all need EORI fields.** Without EORI, no declaration can be filed in the EU. The architecture has no EORI field anywhere.

### 1.7 Special Programs

| Program | EU Name | US Equivalent | Notes |
|---------|---------|---------------|-------|
| Trusted trader | AEO (Authorized Economic Operator) | C-TPAT | Mutual recognition agreement |
| FTZ | Free Zone | Foreign-Trade Zone | Similar concept, different legal framework |
| Inward processing | Inward Processing Authorization | Manufacturing drawback | Import duty-free for re-export after processing |
| Outward processing | Outward Processing Authorization | No direct equivalent | Export for processing, re-import at reduced duty |
| Customs warehousing | Customs Warehousing Authorization | Bonded Warehouse (Entry Type 21) | Up to unlimited storage time |
| Temporary admission | Temporary Admission Authorization | TIB (Entry Type 23) | ATA Carnet or customs authorization |
| End-use | End-Use Authorization | No direct equivalent | Reduced/zero duty for specific uses |

### 1.8 Export Requirements

| Requirement | Details |
|-------------|---------|
| **ECS** (Export Control System) | Electronic export declaration mandatory for ALL exports from EU |
| **Filing** | Via national systems (ATLAS-Ausfuhr in DE, DELTA in FR) |
| **Deadline** | Before goods leave the EU — pre-departure notification |
| **Dual-use controls** | EU Dual-Use Regulation (2021/821) — export license required for controlled items |
| **MRN** | Each export declaration gets a Master Reference Number for tracking |

### 1.9 Transit Procedures

**NCTS (New Computerised Transit System):**

| Transit Type | Use Case | Guarantee Required |
|-------------|----------|-------------------|
| **T1** (External transit) | Non-EU goods moving through EU customs territory | Yes — cash deposit or comprehensive guarantee |
| **T2** (Internal transit) | EU goods moving through non-EU territory (e.g., CH, NO) | Yes |
| **TIR** (International road transit) | Under TIR Convention, carnet-based | TIR carnet |

**The architecture's in-bond model (currently absent) must support T1/T2 for EU, not just IT/IE/T&E for US.**

### 1.10 Architecture Impact

**The EntryFiling model must support:**
- `filing_type`: ENS, H1-H7 datasets, simplified declaration, EIDR
- `mrn`: Master Reference Number (EU's unique filing identifier, different from US entry number)
- `national_system`: ATLAS, DELTA, AGS, etc.
- `eori_declarant`, `eori_importer`, `eori_representative`: EORI numbers for all parties
- `vat_rate`, `vat_amount`: Separate from customs duty
- `customs_procedure_code`: 4-digit EU CPC (e.g., 4000 = free circulation, 5100 = inward processing)
- `previous_procedure_code`: Required when goods are under a previous customs procedure

**The Adjudication status machine must include:**
- `registered` (MRN assigned)
- `risk_analysis` (automated assessment)
- `document_check` (officer review)
- `under_control` (customs supervision)
- `customs_released` (goods released to free circulation)
- `customs_debt_confirmed` (duty amount finalized)

**New domain or sub-domain needed:** NCTS Transit Management — T1/T2 declarations with their own status machine, guarantee tracking, and multi-office coordination.

---

## 2. China (GACC — General Administration of Customs)

### 2.1 Customs Authority System

**Primary system:** China Single Window (国际贸易单一窗口) — integrates customs, CIQ (inspection and quarantine), tax, foreign exchange, and port authority systems.

| System Component | Purpose |
|-----------------|---------|
| **China Single Window** | Unified electronic filing portal |
| **H2010/H2018** | Customs declaration processing systems |
| **CIQ** (merged into GACC in 2018) | Inspection and quarantine (now part of customs) |
| **Golden Gate** | Legacy system being phased out |
| **CIFER** | Registration system for overseas food manufacturers |

### 2.2 Filing Requirements

**Import declaration process:**

| Step | Details |
|------|---------|
| **1. Pre-arrival manifest** | Carrier files manifest via Single Window before vessel/aircraft arrival |
| **2. Customs declaration** | Filed by licensed customs broker (报关行) or self-filing enterprise |
| **3. Channel assignment** | Automated risk-based assignment to green/yellow/red channel |
| **4. Assessment** | Duty calculated, taxes assessed |
| **5. Payment** | Duties/taxes paid via bank or electronic payment |
| **6. Release** | Goods released from customs supervision |

**Filing deadline:** Within 14 days of vessel arrival (ocean), within 14 days of aircraft arrival (air). Late filing incurs penalties (0.05% of duty per day).

**Who can file:**
- Licensed customs brokers (报关行) — must have GACC license
- Self-filing enterprises — must have customs registration certificate
- Authorized Economic Operators (AEO) — expedited processing

### 2.3 Document Hierarchy

| Document | Chinese Name | Required | Notes |
|----------|-------------|----------|-------|
| **Customs Declaration Form** | 报关单 | Mandatory | Core filing document |
| **Commercial Invoice** | 商业发票 | Mandatory | Must match declaration values |
| **Packing List** | 装箱单 | Mandatory | Weight, quantity, packaging |
| **Bill of Lading / Airway Bill** | 提单/空运单 | Mandatory | Transport document |
| **Certificate of Origin** | 原产地证明 | Conditional | For preferential tariff rates |
| **Inspection Certificate** | 检验证书 | Conditional | For CIQ-regulated products |
| **Import License** | 进口许可证 | Conditional | For restricted goods |
| **Automatic Import License** | 自动进口许可证 | Conditional | For monitored goods |
| **GACC Registration** | 海关总署注册 | Conditional | Mandatory for overseas food/cosmetics manufacturers |

### 2.4 Status Lifecycle

**China customs declaration processing statuses:**

```
已申报 (Declared/Submitted)
    |
    v
Channel Assignment (Green/Yellow/Red)
    |
    +---> Green Channel → 已审结 (Review Completed) → 已放行 (Released) → 已结关 (Customs Closed)
    |
    +---> Yellow Channel → Document Review → 已审结 → 已放行 → 已结关
    |
    +---> Red Channel → Document Review + Physical Inspection → 已审结 → 已放行 → 已结关
    |
    v
已通关 (Customs Clearance Complete — all procedures finished)
```

**Key statuses:**
| Status (Chinese) | Status (English) | Meaning |
|------------------|-----------------|---------|
| 已申报 | Declared | Electronic data received by customs system |
| 已审结 | Review Completed | Risk assessment and/or officer review done |
| 已放行 | Released | Goods may be picked up / moved from customs area |
| 已结关 | Customs Closed | All customs procedures completed for this declaration |
| 已通关 | Clearance Complete | Full clearance (including post-release supervision if applicable) |

**Critical difference from US:** "Released" (已放行) ≠ "Cleared" (已结关). Goods can be released but still under customs supervision (e.g., post-clearance audit, price verification). The platform must model this distinction.

### 2.5 Duty/Tax Structure

**China's cascading tax structure:**

```
CIF Value (customs value)
    x Import Duty Rate (MFN rate or preferential rate)
    = Import Duty

(CIF Value + Import Duty)
    x VAT Rate (13% standard, 9% for certain goods)
    = Import VAT

(CIF Value + Import Duty)
    x Consumption Tax Rate (for luxury/demerit goods)
    = Consumption Tax

Total = Import Duty + VAT + Consumption Tax
```

**MFN rates vs. General rates:**
- MFN (Most Favored Nation): Applied to WTO member countries — most common
- General rates: Applied to countries without MFN status — significantly higher (often 2-3x)
- Preferential rates: Under FTAs (RCEP, China-ASEAN, China-Korea, etc.)
- Retaliatory tariffs: Additional tariffs on specific goods from specific countries (e.g., US goods)

**Cross-border e-commerce (CBEC):**
- Special consolidated tax: Duty rate 0% for goods in "positive list," combined tax = (VAT + Consumption Tax) × 70%
- Per-transaction limit: RMB 5,000
- Annual per-person limit: RMB 26,000
- Must be through registered CBEC platform

### 2.6 Party Requirements

| Registration | Who Needs It | Format | Notes |
|-------------|-------------|--------|-------|
| **Customs Registration Certificate** | All importers/exporters | 10-digit registration number | Unified Social Credit Code used since 2019 |
| **GACC Registration** (Decree 248/249) | Overseas food/cosmetics manufacturers | 18-digit format: C-[Country Code]-[Numbers] | Mandatory since January 2022 |
| **AEO Certification** | Voluntary — trusted enterprises | Classification: General Certified, Advanced Certified | Mutual recognition with US C-TPAT, EU AEO, etc. |

**GACC Decree 248/249 is critical:** As of January 2022, ALL overseas food manufacturers exporting to China must be registered with GACC. For 18 product categories (meat, dairy, seafood, etc.), registration must be recommended by the competent authority of the exporting country. For other food categories, the manufacturer registers directly through the CIFER online system. The 18-digit registration number follows: `C-[2-letter country code]-[unique digits]`.

### 2.7 Special Zones

| Zone Type | Purpose | Duty Treatment |
|-----------|---------|---------------|
| **Comprehensive Bonded Zone** (综合保税区) | Processing, logistics, R&D | Full duty/tax deferral until goods enter domestic market |
| **Free Trade Zone** (自由贸易区) | Pilot reform areas (Shanghai, Hainan, etc.) | Simplified customs procedures, some duty exemptions |
| **Bonded Logistics Park** (保税物流园区) | Bonded warehousing and distribution | Duty deferred |
| **Export Processing Zone** (出口加工区) | Manufacturing for export | Duty-free import of materials for re-export |

**"First line liberalization, second line control" principle:** Goods entering bonded zones from abroad ("first line") face minimal customs formalities. Goods moving from bonded zones to domestic market ("second line") require full customs declaration and duty payment.

### 2.8 Export Requirements

| Requirement | Details |
|-------------|---------|
| **Export declaration** | Mandatory for all exports via Single Window |
| **Export license** | Required for restricted goods (dual-use, strategic materials, cultural relics) |
| **CIQ inspection** | Required for regulated products |
| **Export tax** | Applied to certain raw materials (rare earths, some metals) |
| **Export controls** | China Export Control Law (2020) — dual-use items, military items, nuclear materials |

### 2.9 Architecture Impact

**The EntryFiling model must support:**
- `declaration_number`: China's 18-digit customs declaration number (different format from US entry number)
- `channel_assignment`: green | yellow | red (mandatory after submission)
- `gacc_registration`: Overseas manufacturer registration number
- `hs_code_cn`: 10-digit Chinese tariff code (HS 8-digit + 2-digit national extension)
- `consumption_tax_rate`, `consumption_tax_amount`: China-specific tax component
- `vat_rate`, `vat_amount`: 13% or 9%
- `trade_mode`: General trade (一般贸易), processing trade (加工贸易), CBEC, etc.
- `customs_supervision_end_date`: Date when post-release supervision ends

**The Adjudication status machine must include:**
- `declared` (已申报)
- `channel_assigned` (green/yellow/red)
- `under_document_review` (yellow/red channel)
- `under_physical_inspection` (red channel)
- `review_completed` (已审结)
- `released` (已放行)
- `customs_closed` (已结关)
- `clearance_complete` (已通关)

---

## 3. Brazil (Receita Federal)

### 3.1 Customs Authority System

**Primary system:** Siscomex (Sistema Integrado de Comércio Exterior) — transitioning to Portal Único de Comércio Exterior (Single Window).

| System | Purpose | Status |
|--------|---------|--------|
| **Siscomex Importação** | Import declarations (DI) | Being replaced by DUIMP |
| **Siscomex Exportação** | Export declarations (RE) | Being replaced by DU-E |
| **DUIMP** (Declaração Única de Importação) | Unified import declaration | Rolling out 2024-2025, mandatory by end of 2025 |
| **DU-E** (Declaração Única de Exportação) | Unified export declaration | Active |
| **Portal Único** | Single window for all trade operations | Active, expanding |
| **LPCO** | Licenças, Permissões, Certificados e Outros Documentos | New module replacing import licenses |
| **Catálogo de Produtos** | Product catalog linked to DUIMP | Required for DUIMP filings |

### 3.2 Filing Requirements

**Import declaration process (DUIMP — current standard):**

| Step | Details |
|------|---------|
| **1. Product catalog registration** | Products must be registered in Catálogo de Produtos with NCM code, description, attributes |
| **2. LPCO** | Import license obtained (if product requires administrative control from anuentes) |
| **3. DUIMP filed** | Electronic declaration via Portal Único |
| **4. Parametrization** | Automated channel assignment (green/yellow/red/grey) |
| **5. Customs clearance** | Processing per assigned channel |
| **6. Release / Desembaraço** | Goods released from customs |

**Key difference: product catalog is a PREREQUISITE.** Brazil requires that products be registered in a centralized catalog BEFORE a declaration can be filed. This catalog includes NCM code, product description, manufacturer, and attributes. The architecture's Product Catalog domain aligns well with this concept, but it needs a Brazil-specific product registration workflow.

**NF-e (Nota Fiscal Eletrônica) requirement:**
- **Every movement of goods within Brazil requires an NF-e.** This is not optional.
- Import NF-e (CFOP 3101/3102): Issued after customs clearance for bringing imported goods into domestic commerce
- The NF-e is a fiscal document, not a customs document — but it's REQUIRED for goods to legally move after customs clearance
- The platform must generate or validate NF-e references for Brazil-bound goods

### 3.3 Parametrization Channels

| Channel | Color | Action | Typical % |
|---------|-------|--------|-----------|
| **Green** (Verde) | 🟢 | Automatic release — no document check, no physical exam | ~60-70% |
| **Yellow** (Amarelo) | 🟡 | Document examination only — officer reviews declaration and documents | ~15-20% |
| **Red** (Vermelho) | 🔴 | Document examination + physical inspection of goods | ~10-15% |
| **Grey** (Cinza) | ⬜ | Full investigation — document exam, physical exam, AND customs valuation analysis + origin investigation | <5% |

**Channel assignment factors:**
- Importer's compliance history
- Product risk classification
- Country of origin
- Declared value vs. historical reference prices
- RADAR classification (limited vs. unlimited)
- AEO (Linha Azul) certification — heavily favors green channel
- Random selection component

**Critical difference from US:** The grey channel is unique to Brazil — it triggers a full customs valuation investigation including price verification, transfer pricing analysis, and origin investigation. This can delay clearance by weeks or months. No US equivalent exists.

### 3.4 Document Hierarchy

| Document | Portuguese Name | Required | Notes |
|----------|----------------|----------|-------|
| **DUIMP** | Declaração Única de Importação | Mandatory | Core import declaration |
| **Commercial Invoice** | Fatura Comercial | Mandatory | Must be in Portuguese or with certified translation |
| **Packing List** | Romaneio | Mandatory | Detailed packaging description |
| **Bill of Lading / AWB** | Conhecimento de Embarque | Mandatory | Transport document |
| **Certificate of Origin** | Certificado de Origem | Conditional | For Mercosul, ALADI, other preferential agreements |
| **Import License (LPCO)** | Licença de Importação | Conditional | For products under administrative control |
| **NF-e** | Nota Fiscal Eletrônica | Post-clearance mandatory | For domestic goods movement |
| **RADAR registration** | Habilitação RADAR | Mandatory | Importer must be RADAR-registered |

### 3.5 Status Lifecycle

**Brazil import declaration (DUIMP) statuses:**

```
DUIMP Registered (Registrada)
    |
    v
Parametrization (channel assigned)
    |
    +---> Green → Desembaraçada (Cleared) → Entregue (Delivered to importer)
    |
    +---> Yellow → Document Examination → Desembaraçada → Entregue
    |
    +---> Red → Document + Physical Exam → Desembaraçada → Entregue
    |
    +---> Grey → Full Investigation → Desembaraçada → Entregue
    |
    +---> Interrupted (Interrompida) → requires additional documents/information
    |
    v
Exigência (Requirement) — customs demands additional information/documentation
    |
    v
Cumprida (Requirement fulfilled) → returns to processing
```

### 3.6 Duty/Tax Structure

**Brazil has the most complex import tax structure of all target jurisdictions — taxes cascade:**

```
CIF Value (converted to BRL at PTAX rate)

1. II (Imposto de Importação) = CIF × II Rate (0-35%, varies by NCM)
2. IPI (Imposto sobre Produtos Industrializados) = (CIF + II) × IPI Rate (0-330%)
3. PIS (Contribuição para o PIS) = (CIF + II + IPI + ICMS + PIS + COFINS) × 2.1% *
4. COFINS (Contribuição para o COFINS) = (CIF + II + IPI + ICMS + PIS + COFINS) × 9.65% *
5. ICMS (state-level VAT) = (CIF + II + IPI + PIS + COFINS) / (1 - ICMS rate) × ICMS rate
   ICMS rates: 4% (interstate for imports), 7-18% (varies by state, 17-18% most common)
6. AFRMM (Adicional ao Frete para Renovação da Marinha Mercante) = 8% of ocean freight (ocean only)

* PIS/COFINS calculation is circular (they include themselves in the base) — resolved by formula
```

**Total effective import tax rate in Brazil often reaches 60-100%+ of CIF value.** This is NOT a bug — it's how Brazil's tax system works. The cascading nature means each tax is calculated on a base that includes previous taxes.

**Architecture impact:** The TariffCalculation model needs:
- `ii_rate`, `ii_amount` (federal import duty)
- `ipi_rate`, `ipi_amount` (excise tax)
- `pis_rate`, `pis_amount` (social contribution)
- `cofins_rate`, `cofins_amount` (social contribution)
- `icms_rate`, `icms_amount` (state VAT — varies by STATE, not just country)
- `afrmm_amount` (ocean freight surcharge)
- `state_of_destination`: Determines ICMS rate

**The current `duty_total` + `duty_line_items` structure is insufficient for Brazil's cascading taxes.** Each tax depends on the previous ones — they must be calculated in sequence, not independently.

### 3.7 Party Requirements

| Registration | Who Needs It | Types | Notes |
|-------------|-------------|-------|-------|
| **RADAR** (Registro e Rastreamento da Atuação dos Intervenientes Aduaneiros) | ALL importers | Express (up to $50K/semester), Limited (up to $150K/semester), Unlimited (no limit) | RADAR type limits import VALUE, not volume |
| **CNPJ** | All Brazilian legal entities | 14-digit number | Federal taxpayer ID |
| **Inscrição Estadual** | All entities subject to ICMS | State-specific format | Required for ICMS credit |

**RADAR is a GATING requirement.** Without RADAR registration, a company CANNOT import into Brazil — period. The type of RADAR (Express/Limited/Unlimited) determines the maximum value of imports allowed per semester. This is a party-level attribute that the platform must track.

### 3.8 Special Regimes

| Regime | Purpose | Duty Treatment |
|--------|---------|---------------|
| **Drawback** | Import inputs for export products | Suspension, exemption, or refund of import taxes |
| **RECOF** | Industrial bonded warehouse under computerized control | Full duty/tax deferral |
| **Entreposto Aduaneiro** | Customs warehouse | Duty deferred until withdrawal |
| **Admissão Temporária** | Temporary import | Proportional tax payment or full suspension |
| **Linha Azul** | AEO equivalent | Green channel preference, reduced exams |
| **OEA** (Operador Econômico Autorizado) | Full AEO program | Maximum facilitation |

### 3.9 Transit Procedures

| Procedure | Document | Use Case |
|-----------|----------|----------|
| **DTA** (Declaração de Trânsito Aduaneiro) | Transit declaration | Goods moving between customs zones within Brazil under customs bond |
| **MIC/DTA** | Mercosul transit document | Goods in transit through Mercosul countries |

### 3.10 Architecture Impact

**The EntryFiling model must support:**
- `ncm_code`: 8-digit Nomenclatura Comum do Mercosul (NOT HS code — format differs at 8th digit)
- `radar_type`: express | limited | unlimited (gating check)
- `channel_assignment`: green | yellow | red | grey
- `catálogo_produto_id`: Reference to product catalog registration
- `lpco_number`: Import license number if required
- `nfe_reference`: NF-e reference number for post-clearance goods movement
- `state_destination`: Brazilian state (determines ICMS rate)
- `ptax_exchange_rate`: Exchange rate used for BRL conversion (specific PTAX rate)

**New entities needed:**
- `ProductCatalogRegistration` (Brazil-specific — mapping products to NCM codes with attributes)
- `NFeReference` (tracking NF-e issuance for imported goods)
- `RADARRegistration` (party-level — tracks RADAR type and utilization against limits)

---

## 4. India (CBIC — Central Board of Indirect Taxes and Customs)

### 4.1 Customs Authority System

**Primary system:** ICEGATE (Indian Customs Electronic Gateway) — the electronic portal for all customs trade transactions.

| System Component | Purpose |
|-----------------|---------|
| **ICEGATE** | Electronic filing portal |
| **ICES** (Indian Customs EDI System) | Backend processing engine |
| **RMS** (Risk Management System) | Automated risk profiling and channel assignment |
| **SWIFT 2.0** | Single Window Interface for Facilitating Trade — PGA integration |
| **e-Sanchit** | Paperless document upload (PDF/A format) |
| **DGFT Portal** | IEC validation, Advance Authorization, DFIA license verification |
| **e-Payment Gateway** | Duty payment via authorized banks |

**2025-2026 development:** India has issued an Expression of Interest (EoI) to build a unified national customs platform integrating ICEGATE, RMS, and ICES into a single system (target: 24-hour cargo clearance, bids due January 2026).

### 4.2 Filing Requirements

**Bill of Entry (BoE) types:**

| Type | Section | Purpose | Duty Payment |
|------|---------|---------|-------------|
| **Home Consumption** (White BoE) | Section 46 | Direct clearance into domestic market | Upfront at filing |
| **Warehousing** (Yellow/Into-Bond BoE) | Section 46 + 59 | Storage in bonded warehouse | Deferred until ex-bond |
| **Ex-Bond BoE** | Section 68 | Clearance from bonded warehouse to domestic market | At time of removal |

**Import General Manifest (IGM):**
- Filed by master/agent of vessel or airline
- Vessel: Within 24 hours of arrival
- Aircraft: Within 12 hours of arrival
- Advance filing: Up to 14 days before expected arrival

**Prior Bill of Entry:** India allows filing BoE BEFORE goods arrive — vessel/aircraft must arrive within 30 days of filing. This is India's version of pre-clearance.

### 4.3 Export Process

**Shipping Bill types:**

| Type | Color Code | Purpose |
|------|-----------|---------|
| Free/Duty-Free | White | No export duty, no incentive claims |
| Dutiable | Yellow | Goods subject to export duty |
| Drawback | Green | Claiming refund of duties on imported inputs |
| Ex-Bond | — | Re-export from bonded warehouse |

**Let Export Order (LEO):** Final customs clearance authorizing goods to be loaded on vessel/aircraft. Critical milestone in India's export lifecycle.

### 4.4 Status Lifecycle

**Bill of Entry (Import) lifecycle:**

```
IGM Filed (by carrier)
    |
    v
BoE Filed/Submitted (via ICEGATE)
    |
    v
RMS Processing (automated risk assessment)
    |
    +---> Green (Facilitated) → Duty Payable → Duty Paid → Out of Charge (OOC) → Goods Released
    |     (~80%+ of shipments)
    |
    +---> Yellow (Document Check) → Under Assessment → Assessed → Duty Paid → OOC → Released
    |
    +---> Red (Assessment + Exam) → Assessment → Examination Ordered → Examined → OOC → Released
    |
    v
Query Raised (officer questions — importer must respond)
    |
    v
On Hold (additional scrutiny pending)
```

**Key ICEGATE status codes:**

| Status | Meaning |
|--------|---------|
| Filed/Submitted | BoE successfully lodged in ICES |
| RMS Processed | Risk assessment complete, channel assigned |
| Under Assessment | Officer reviewing valuation/classification |
| Query Raised | Officer has raised a query; importer must respond |
| Assessed | Duty amount finalized |
| Duty Paid | Payment confirmed via e-Payment |
| Examination Ordered | Physical inspection required |
| Examined | Examination complete |
| **Out of Charge (OOC)** | Customs clearance granted; goods may be released |
| Pending | Awaiting documentation or action |
| On Hold | Held for additional scrutiny |

**"Out of Charge" (OOC)** is the key India-specific status — it's the moment when customs releases goods. There is no US equivalent status name. The architecture must model this.

**Faceless Assessment (Turant Customs):** India now allows remote assessment — officers in different jurisdictions can assess declarations. Faceless Assessment Groups (FAGs) virtually connect officers across customs stations.

### 4.5 Duty/Tax Structure

**India's duty calculation:**

```
Assessable Value (AV) = CIF Value in INR
    (FOB + Freight + Insurance, converted at CBE exchange rate)

BCD = AV × BCD Rate% (0% to 100%+, varies by ITC-HS code)
SWS = BCD × 10% (Social Welfare Surcharge)
IGST = (AV + BCD + SWS) × IGST Rate% (5%, 18%, or 40% post-GST 2.0)
Total Duty = BCD + SWS + IGST + [Anti-dumping duty if applicable]
```

**GST 2.0 reform (effective September 22, 2025):** Simplified from 5 slabs (0%, 5%, 12%, 18%, 28%) to 3 primary rates: **5%**, **18%**, **40%**.

**Architecture impact:** The TariffCalculation model needs:
- `bcd_rate`, `bcd_amount` (Basic Customs Duty)
- `sws_amount` (Social Welfare Surcharge — 10% of BCD)
- `igst_rate`, `igst_amount` (Integrated GST)
- `anti_dumping_duty` (case-specific)
- `compensation_cess` (for luxury/demerit goods)
- `exchange_rate_source`: CBE (Central Board of Excise) weekly exchange rate, not market rate

### 4.6 Party Requirements

| Registration | Who Needs It | Format | Notes |
|-------------|-------------|--------|-------|
| **IEC** (Import Export Code) | ALL importers/exporters | 10-digit (= PAN number since GST rollout) | Lifetime validity, annual update required |
| **AEO** | Voluntary | Tiers: T1, T2, T3, LO (Logistics) | T2+ get deferred duty payment |
| **DSC** (Digital Signature Certificate) | All ICEGATE filers | Class 3 | Required for electronic filing |

### 4.7 Special Programs

| Program | Purpose | Benefit |
|---------|---------|---------|
| **AEO-T1** | Basic trusted trader | Direct Port Delivery, reduced examination |
| **AEO-T2** | Enhanced trusted trader | All T1 + Deferred Duty Payment ("Clear First, Pay Later") |
| **AEO-T3** | Highest trust | All T2 + near-zero bank guarantee, self-sealing |
| **SEZ** | Special Economic Zones | Duty-free import for authorized operations |
| **Advance Authorization** | Duty-free import of inputs for export | No duty on imported inputs if exported |
| **DFIA** | Duty Free Import Authorisation | Transferable duty exemption (post-export) |
| **MOOWR** (Section 65) | Manufacture in bonded warehouse | Full duty deferment on inputs AND capital goods |

### 4.8 Transit/Bonded Warehousing

**Customs bonded warehousing (Sections 57-68):**
- Section 49: Temporary storage pending clearance
- Section 58: Private bonded warehouse license
- Section 59: Warehousing bond — **3x the assessed duty amount**
- Section 61: Storage period — 1 year (general), 5 years (capital goods), unlimited (MOOWR)
- Section 65: **MOOWR** — Manufacturing and Other Operations in Warehouse (permanent license, no export obligation)

**Transshipment:**
- Section 54 of Customs Act
- Transshipment bond with bank guarantee required
- Goods sealed by customs for movement between ports/ICDs
- All-India National Transshipment Bond facility (since 2022) — single bond covers all stations

### 4.9 Architecture Impact

**The EntryFiling model must support:**
- `boe_type`: home_consumption | warehousing | ex_bond
- `iec_number`: 10-digit Import Export Code
- `igm_reference`: Import General Manifest line number
- `rms_channel`: green | yellow | red
- `ooc_timestamp`: Out of Charge timestamp (the key clearance moment)
- `esanchit_documents`: Array of UDIN/DRN references for uploaded documents
- `swift_noc_status`: NOC statuses from each PGA agency
- `assessment_type`: facilitated | first_check | second_check | group_assessed
- `shipping_bill_type` (export): free | dutiable | drawback | ex_bond
- `leo_timestamp` (export): Let Export Order timestamp

**The Adjudication status machine must include:**
- `filed` / `submitted`
- `rms_processed` (channel assigned)
- `under_assessment`
- `query_raised` (with response deadline)
- `assessed` (duty determined)
- `duty_paid`
- `examination_ordered`
- `examined`
- `out_of_charge` (OOC — the key release status)
- `on_hold`

---

## 5. Mexico (SAT / Aduana)

### 5.1 Customs Authority System

**Primary system:** VUCEM (Ventanilla Única de Comercio Exterior Mexicano) — Mexico's single window for foreign trade.

| System Component | Purpose |
|-----------------|---------|
| **VUCEM** | Electronic single window for all trade operations |
| **SAAI** (Sistema Automatizado Aduanero Integral) | Automated customs processing system |
| **Pre-validators** (Prevalidadoras) | Authorized companies that validate pedimentos before submission |
| **COVE** (Comprobante de Valor Electrónico) | Electronic value voucher system |

### 5.2 Filing Requirements

**The Pedimento — Mexico's core customs document:**

| Aspect | Details |
|--------|---------|
| **What is it** | The pedimento is Mexico's equivalent of the US entry summary — the fundamental customs declaration |
| **Who files** | Licensed Mexican customs broker (Agente Aduanal) ONLY — no self-filing |
| **Pre-validation** | Mandatory — pedimento must pass through an authorized pre-validator before submission to SAT |
| **Filing** | Electronic via VUCEM after pre-validation |

**Pedimento clave (code) types — 76 codes in 176 scenarios:**

| Code | Name | Purpose |
|------|------|---------|
| **A1** | Importación/Exportación Definitiva | Standard definitive import or export |
| **A3** | Importación Definitiva Virtual | Virtual definitive import (maquila transfers) |
| **AF** | Retorno de Temporales | Return of temporary imports |
| **G1** | Exportación Temporal | Temporary export |
| **H1** | Tránsito Interno | Internal customs transit |
| **IN** | Importación Temporal | Temporary import (IMMEX) |
| **K1** | Retorno de Exportación Definitiva | Return of definitive export |
| **RT** | Retorno de Temporal | Return of temporary (IMMEX) |
| **T1** | Tránsito Internacional | International transit |
| **V1** | Transferencia Virtual | Virtual transfer between IMMEX companies |

### 5.3 Status Lifecycle

**Mexican customs clearance process:**

```
Pedimento Pre-validated (by authorized pre-validator)
    |
    v
Pedimento Submitted (via VUCEM to SAAI)
    |
    v
Pago de Contribuciones (Duties/taxes paid at authorized bank)
    |
    v
Modulación (Random selection for inspection)
    |
    +---> Desaduanamiento Libre (Free clearance — GREEN) → Released
    |     (No irregularities found)
    |
    +---> Reconocimiento Aduanero (First examination — RED)
    |         |
    |         +---> Released (no issues)
    |         |
    |         +---> Second Examination (Segundo Reconocimiento)
    |                   |
    |                   +---> Released
    |                   +---> Detained / PAMA proceedings
    |
    v
Desaduanamiento (Customs clearance complete)
```

**PAMA (Procedimiento Administrativo en Materia Aduanera):** Mexico's administrative customs proceedings — equivalent to US seizure/forfeiture proceedings. Triggered when irregularities are found during inspection.

### 5.4 Duty/Tax Structure

```
Customs Value (Transaction Value, typically CIF)

1. IGI (Impuesto General de Importación) = CIF × Tariff Rate (per tariff schedule)
2. DTA (Derecho de Trámite Aduanero) = 0.8% of customs value (formal entries), fixed rate for some regimes
3. IVA (Impuesto al Valor Agregado) = (CIF + IGI + DTA) × 16%
4. IEPS (Impuesto Especial sobre Producción y Servicios) = applies to specific goods (alcohol, tobacco, fuel)
5. Cuotas Compensatorias (anti-dumping/countervailing duties) — case-specific

Total = IGI + DTA + IVA + IEPS (if applicable) + Cuotas Compensatorias (if applicable)
```

### 5.5 Party Requirements

| Registration | Who Needs It | Format | Notes |
|-------------|-------------|--------|-------|
| **RFC** (Registro Federal de Contribuyentes) | ALL importers/exporters | 12 characters (individuals) or 13 characters (legal entities) | Mexican tax ID |
| **Padrón de Importadores** | ALL importers | Enrollment via SAT | Must be registered to import |
| **Padrón Sectorial** | Importers of specific sectors | Sector-specific enrollment | Steel, textiles, chemicals, etc. |
| **IMMEX** | Maquiladora operations | IMMEX authorization | Temporary import of materials for export processing |
| **OEA** (Operador Económico Autorizado) | Voluntary — trusted traders | Certification by SAT | Mexico's AEO program |

**Padrón de Importadores is MANDATORY.** Without enrollment, a company CANNOT import into Mexico — even with a valid RFC. Sector-specific padrones add another layer for regulated goods.

### 5.6 Special Programs

| Program | Purpose | Benefit |
|---------|---------|---------|
| **IMMEX** | Maquiladora (manufacturing for export) | Temporary import duty-free for materials incorporated into export products |
| **PROSEC** (Programas de Promoción Sectorial) | Sector promotion | Reduced import tariffs for specific sectors |
| **OEA** | Authorized Economic Operator | Expedited clearance, fewer inspections |
| **Certified Company** (Empresa Certificada) | VAA, Socio Comercial variants | Benefits include express lanes, fewer exams |
| **NEEC** (Nuevo Esquema de Empresas Certificadas) | Enhanced certification | Maximum facilitation |

### 5.7 Architecture Impact

**The EntryFiling model must support:**
- `pedimento_number`: Mexico's pedimento number format
- `pedimento_clave`: A1, IN, RT, V1, etc. (76 possible codes)
- `customs_regime`: Definitive, temporary, transit, bonded warehouse, etc.
- `pre_validation_result`: Status from pre-validator
- `rfc_importador`: Importer's RFC
- `padron_type`: general | sectorial (and which sector)
- `immex_authorization`: If applicable
- `cove_reference`: Electronic value voucher reference
- `modulacion_result`: libre | reconocimiento_primero | reconocimiento_segundo

---

## 6. Canada (CBSA)

### 6.1 Customs Authority System

**Primary system:** CARM (CBSA Assessment and Revenue Management) — Canada's modernized customs system.

| System Component | Purpose | Status |
|-----------------|---------|--------|
| **CARM Client Portal (CCP)** | Web-based portal for importers, brokers, consultants | Fully operational (Release 2 since October 2024) |
| **CARM API** | Application Programming Interface for EDI integration | Published API inventory, Version 5.1 specs (May 2025) |
| **ICES** (legacy) | Integrated Customs Enforcement System | Being replaced by CARM |
| **ACI** (Advance Commercial Information) | Pre-arrival cargo data | Mandatory for all modes |

### 6.2 Filing Requirements

**Two-step process (similar to US but with different terminology):**

| Step | Document | Deadline |
|------|----------|----------|
| **Release** | Release package (minimum documentation) | At or before arrival |
| **Accounting** | Commercial Accounting Declaration (CAD) | Within 5 business days of release |

**CAD declaration types:**

| Type | When Used |
|------|-----------|
| **Type A / AB** | Standard post-release accounting (most common) |
| **Type B** | Over-the-counter (importers at border) |
| **Type C** | Cash entry (no RPP financial security on file) |
| **Type F** | CLVS (Courier Low Value Shipment) |
| **Type V** | Voluntary accounting (goods entered without official release) |
| **Type 10** | Goods entering customs bonded warehouse |
| **Type 13** | Transfer between bonded warehouses |
| **Type 20** | Removal from bonded warehouse for consumption |
| **Type 21** | Removal from bonded warehouse for export |

**ACI (Advance Commercial Information) — mandatory pre-arrival data:**

| Mode | Deadline | Filed By |
|------|----------|----------|
| Marine | 24 hours before loading (non-US/MX origin) | Carrier |
| Air | As early as practicable, no later than wheels-up | Carrier |
| Highway | 1 hour before arrival at border | Carrier |
| Rail | 2 hours before arrival at border | Carrier |

### 6.3 Status Lifecycle

```
ACI Data Filed (by carrier, pre-arrival)
    |
    v
Release Requested (minimum documentation at arrival)
    |
    v
CBSA Risk Assessment (automated targeting)
    |
    +---> Released (no examination) → Accounting (CAD within 5 days)
    |
    +---> Referred for Examination
    |         |
    |         +---> Released after exam → Accounting
    |         +---> Detained
    |
    v
CAD Submitted (within 5 business days)
    |
    v
CBSA Assessment (duties/taxes calculated)
    |
    v
Statement of Account Generated (monthly)
    |
    v
Payment Due (by due date on statement)
```

### 6.4 Duty/Tax Structure

```
Customs Value (Transaction Value)

1. Customs Duty = Value × Tariff Rate (per Canadian Customs Tariff)
2. SIMA Duty (Special Import Measures Act) = Anti-dumping/countervailing duties
3. Excise Duty = On specific goods (alcohol, tobacco, cannabis)
4. GST (Goods and Services Tax) = (Value + Duty + SIMA + Excise) × 5%
5. Provincial taxes vary:
   - HST provinces: Combined rate (13-15%)
   - PST provinces: Separate provincial sales tax
   - No PST: Alberta, Northwest Territories, Nunavut, Yukon
```

### 6.5 Party Requirements

| Registration | Who Needs It | Format | Notes |
|-------------|-------------|--------|-------|
| **BN** (Business Number) | ALL importers | 9-digit + RM0001 (import-export account) | Register with CRA |
| **CARM portal registration** | All trade chain partners | GCKey or Sign-in Partner | Required for CAD filing |
| **RPP** (Release Prior to Payment) | Importers wanting release before paying | Financial security on file | Without RPP, must pay cash at time of release (Type C) |
| **PIP** (Partners in Protection) | Voluntary — trusted traders | CBSA certification | Canada's C-TPAT equivalent |
| **CSA** (Customs Self-Assessment) | High-volume importers | CBSA authorization | Streamlined release + accounting |
| **FAST** | Trucking carriers | Joint US-CA program | Pre-approved drivers, expedited border crossing |

### 6.6 Architecture Impact

**The EntryFiling model must support:**
- `cad_type`: A | AB | B | C | F | V | 10 | 13 | 20 | 21
- `bn_number`: Business Number + RM account
- `release_number`: CBSA release reference
- `accounting_deadline`: 5 business days from release
- `rpp_status`: Whether importer has Release Prior to Payment
- `aci_reference`: Advance Commercial Information reference
- `sima_duty`: Anti-dumping/countervailing duty (Canada-specific name)
- `gst_rate`, `gst_amount`: 5% GST (federal)
- `provincial_tax_rate`, `provincial_tax_amount`: HST or PST (varies by province)
- `statement_account_period`: Monthly statement cycle

---

## 7. Cross-Jurisdiction Architectural Assessment

### 7.1 The Current Approach is NOT Scalable

The architecture's current approach is:
1. `jurisdiction: string` field on EntryFiling
2. `jurisdiction_config: {}?` JSONB blob for jurisdiction-specific configuration
3. Jurisdiction-specific NATS subject patterns (`clearance.adjudication.us.*`, `clearance.adjudication.eu.*`)
4. Jurisdiction-specific adjudication decision types (hard-coded in the architecture doc)

**Why this fails:**

| Problem | Details |
|---------|---------|
| **JSONB becomes a bag of untyped exceptions** | Every jurisdiction has different required fields. Without schema enforcement, the JSONB blob becomes an undocumented mess. Brazil needs `ncm_code`, `radar_type`, `channel`, `state_destination`, `ptax_rate`. India needs `boe_type`, `iec_number`, `igm_reference`, `rms_channel`. Stuffing these into a flat JSONB is asking for data quality problems. |
| **Status machines are fundamentally different** | The `filing_status` enum is US-centric: `draft | pending_broker_approval | submitted | accepted | cf28_pending | cf29_pending | exam_scheduled | rejected | released`. None of these match Brazil's parametrization channels, India's RMS/OOC flow, or EU's MRN/customs-released lifecycle. |
| **Duty calculations cannot be parameterized** | US duties are additive (duty + MPF + HMF). Brazil's are cascading (each tax includes prior taxes in its base). India has a specific sequence (BCD → SWS → IGST). You can't use a single formula with different rates — the FORMULA ITSELF is different. |
| **Document requirements are structurally different** | The Document Management domain lists US-centric document types. EU needs ENS, SAD, EUR.1, T1/T2. Brazil needs DUIMP, LPCO, NF-e. India needs BoE, Shipping Bill, e-Sanchit references. These aren't just different names — they're different entities with different lifecycles. |
| **Party registrations are mandatory per jurisdiction** | EORI (EU), RADAR (BR), IEC (IN), RFC+Padrón (MX), BN (CA). These aren't optional — they GATE filing. The platform must validate party registrations before allowing declaration filing. |

### 7.2 Recommended Architecture: Jurisdiction Adapter Pattern

**Replace string-based jurisdiction + JSONB with a pluggable adapter pattern:**

```
JurisdictionAdapter (interface)
├── filing_workflow: StateMachine         # Different per jurisdiction
├── duty_calculator: DutyEngine           # Different formula per jurisdiction
├── document_requirements: DocSpec        # Different requirements per jurisdiction
├── party_validator: PartyValidator       # Different registrations per jurisdiction
├── status_machine: StatusMachine         # Different statuses per jurisdiction
├── electronic_system: SystemConnector    # Different integration per jurisdiction
└── reference_data: ReferenceDataSource   # Different tariff schedules per jurisdiction

USAdapter implements JurisdictionAdapter
├── filing_workflow: US two-step (entry/release → entry summary)
├── duty_calculator: Additive (duty + MPF + HMF + Section 301 + IEEPA + ADCVD)
├── status_machine: draft → submitted → accepted → cf28_pending → released → liquidated
├── party_validator: Validates importer has bond, POA, ABI access
├── electronic_system: CBP ACE integration
└── reference_data: HTSUS, Section 301/232/IEEPA lists, ADCVD orders

EUAdapter implements JurisdictionAdapter (parameterized by member state)
├── filing_workflow: ENS (carrier) → Import declaration (H1-H7) → Supplementary (if simplified)
├── duty_calculator: Customs duty + AD/CVD + member-state VAT
├── status_machine: registered → risk_analysis → document_check → customs_released → debt_confirmed
├── party_validator: Validates EORI for all parties, AEO status
├── electronic_system: ICS2 + national system (ATLAS, DELTA, AGS, etc.)
└── reference_data: EU TARIC, CN (Combined Nomenclature), member-state VAT rates

BRAdapter implements JurisdictionAdapter
├── filing_workflow: Product catalog → LPCO (if needed) → DUIMP → Parametrization → NF-e
├── duty_calculator: Cascading (II → IPI → PIS/COFINS → ICMS → AFRMM)
├── status_machine: registered → parametrized (green/yellow/red/grey) → desembaraçada → entregue
├── party_validator: Validates RADAR (type + utilization), CNPJ, Inscrição Estadual
├── electronic_system: Portal Único / Siscomex
└── reference_data: NCM (Nomenclatura Comum do Mercosul), TEC, TIPI, state ICMS tables

INAdapter implements JurisdictionAdapter
├── filing_workflow: IGM (carrier) → BoE (importer/CHA) → RMS → Assessment → OOC
├── duty_calculator: Sequential (BCD → SWS → IGST + anti-dumping)
├── status_machine: filed → rms_processed → under_assessment → assessed → duty_paid → ooc
├── party_validator: Validates IEC, DSC (Digital Signature Certificate), AEO tier
├── electronic_system: ICEGATE + SWIFT 2.0 + e-Sanchit
└── reference_data: ITC(HS) 8-digit codes, BCD schedule, IGST rates (GST 2.0)

CNAdapter implements JurisdictionAdapter
├── filing_workflow: Manifest (carrier) → Declaration → Channel → Assessment → Release → Close
├── duty_calculator: Sequential (Import Duty → VAT → Consumption Tax)
├── status_machine: 已申报 → channel_assigned → 已审结 → 已放行 → 已结关 → 已通关
├── party_validator: Validates customs registration, GACC registration (food/cosmetics)
├── electronic_system: China Single Window
└── reference_data: Chinese tariff schedule (10-digit), MFN/General/Preferential rates

MXAdapter implements JurisdictionAdapter
├── filing_workflow: Pre-validation → Pedimento → Payment → Modulación → Desaduanamiento
├── duty_calculator: IGI + DTA + IVA + IEPS + Cuotas Compensatorias
├── status_machine: pre_validated → submitted → paid → modulacion → released | examination
├── party_validator: Validates RFC, Padrón de Importadores, Padrón Sectorial, IMMEX
├── electronic_system: VUCEM / SAAI
└── reference_data: Mexican tariff schedule, pedimento clave codes

CAAdapter implements JurisdictionAdapter
├── filing_workflow: ACI (carrier) → Release → CAD (within 5 days) → Statement
├── duty_calculator: Customs duty + SIMA + Excise + GST + Provincial tax
├── status_machine: aci_filed → released → cad_submitted → assessed → statement_generated
├── party_validator: Validates BN, RPP status, PIP/CSA enrollment
├── electronic_system: CARM Client Portal / CARM API
└── reference_data: Canadian Customs Tariff, SIMA orders, provincial tax rates
```

### 7.3 Does the Consolidation Model Work for All Jurisdictions?

**Yes, with minor extensions.** The HU→Consolidation nesting model is transport-oriented, not jurisdiction-specific. A MAWB from Shanghai to Frankfurt works the same physically whether it clears EU or US customs. The `border_crossing_mode` concept is universally applicable.

**Needed extensions:**
- `consolidation_type` should add: `trailer` (Mexico cross-border trucking), `rail_wagon` (EU/China rail)
- Transit documents (T1/T2 for EU, DTA for Brazil, Section 54 for India) should attach at consolidation level
- Multi-jurisdiction transit (e.g., goods transiting EU under T1 to reach UK) needs the consolidation to carry multiple jurisdiction touchpoints

### 7.4 Does the HU Model Work for All Jurisdictions?

**Yes, the HU model is sound.** Physical cargo units are universal. The key extension needed:

- `export_clearance` and `import_clearance` enums need jurisdiction-specific values OR they should reference the jurisdiction adapter's status machine
- `cage_status` is US-centric (GO deadline = 15 days). Other jurisdictions have different detention rules:
  - EU: Goods not claimed within specific period become subject to disposal
  - Brazil: Goods not cleared within 90 days may be declared abandoned
  - India: Section 48 — goods not cleared within 30 days may be sold by customs

### 7.5 Does the Declaration Checklist Approach Scale?

**The checklist concept is excellent; the implementation needs to be adapter-driven.**

The dynamic checklist model (Section 3.8, lines 951-962) is the right idea — but the checklist items must be generated by the jurisdiction adapter, not hard-coded. Each jurisdiction has fundamentally different readiness criteria:

| Jurisdiction | Checklist Items |
|-------------|----------------|
| **US** | Classification, Tariff, Compliance, Bond, Documents, ISF (ocean), ACAS (air) |
| **EU** | Classification, EORI validation, ENS status (carrier), Documents, AEO status, Customs procedure code |
| **Brazil** | Product catalog registration, RADAR validation, NCM classification, LPCO (if needed), Documents, State destination (ICMS) |
| **India** | IEC validation, BoE type, Classification (ITC-HS), Documents via e-Sanchit, PGA NOC status (SWIFT), Prior notice (if food) |
| **China** | Customs registration, Classification (10-digit), GACC registration (if food/cosmetics), Documents, Trade mode |
| **Mexico** | RFC validation, Padrón registration, Pre-validation, Classification, IMMEX authorization (if applicable), COVE |
| **Canada** | BN validation, RPP status, ACI status (carrier), Classification, Documents, Provincial tax destination |

---

## 8. Bidirectional Trade Lane Requirements

The platform must support ALL combinations of the primary and secondary jurisdictions. Key considerations for each direction:

### 8.1 Export Requirements by Origin

**Every export requires its own declaration.** The platform's current focus on IMPORT clearance means export declarations are undermodeled. For bidirectional trade lanes:

| Origin | Export System | Export Document | Key Requirements |
|--------|-------------|----------------|-----------------|
| US | AES (Automated Export System) | EEI (Electronic Export Information) | License for controlled items, EAR/ITAR compliance |
| EU | ECS (Export Control System) | Export Declaration + MRN | National system per member state, dual-use license |
| China | Single Window | Export Customs Declaration | Export license for controlled items, CIQ inspection |
| Brazil | Siscomex Exportação / DU-E | DU-E (Declaração Única de Exportação) | RE → DU-E → NF-e for domestic movement to port |
| India | ICEGATE | Shipping Bill → LEO | Shipping Bill type determines incentive eligibility |
| Mexico | VUCEM | Pedimento (A1 export) | Pre-validation mandatory, Carta Porte for domestic movement |
| Canada | CBSA | Canadian Export Declaration (CAED) | Via CERS (Canadian Export Reporting System) |

### 8.2 Corridor-Specific Complexities

| Corridor | Special Considerations |
|----------|----------------------|
| **US↔EU** | IEEPA reciprocal tariffs + EU retaliatory tariffs; AEO/C-TPAT mutual recognition |
| **US↔CN** | Section 301 tariffs (US→CN), retaliatory tariffs (CN→US), GACC registration for food |
| **EU↔CN** | EU anti-dumping duties on CN steel/solar/EV; GACC registration; ICS2 ENS for carrier |
| **US↔BR** | RADAR utilization limits; NF-e mandatory post-clearance; cascading tax calculation |
| **US↔IN** | BCD rates highly variable (0-100%+); IGST input tax credit considerations; e-Sanchit |
| **US↔MX** | USMCA Certificate of Origin; IMMEX/maquiladora; high truck volume at border |
| **US↔CA** | CUSMA/USMCA; CARM transition; FAST program for trucking; ACI mandatory |
| **BR↔EU** | Mercosur-EU FTA (pending ratification); NF-e + SAD requirements |
| **IN↔EU** | India-EU FTA negotiations; EORI + IEC requirements on both sides |
| **CN↔BR** | Growing trade corridor; GACC + RADAR; complex tax cascading on both sides |

---

## 9. Prioritized Recommendations

### P0 — Showstoppers (blocks non-US operations entirely)

| # | Gap | Recommendation | Effort |
|---|-----|---------------|--------|
| 1 | No jurisdiction adapter pattern | Implement pluggable jurisdiction adapters for filing workflow, duty calculation, status machine, party validation, document requirements | High |
| 2 | No EORI support | Add EORI fields to party model; validate EORI before EU filing | Low |
| 3 | No ENS/ICS2 model | Add ENS filing as a carrier-side pre-arrival declaration (separate from broker ISF) | Medium |
| 4 | No RADAR support (Brazil) | Add RADAR registration type + utilization tracking to party model | Low |
| 5 | No IEC support (India) | Add IEC validation to party model | Low |
| 6 | No NF-e integration (Brazil) | Add NF-e reference tracking for post-clearance goods movement | Medium |
| 7 | Duty calculator is US-only | Implement jurisdiction-specific duty engines (cascading for BR, sequential for IN, VAT-based for EU) | High |
| 8 | Status machines are US-centric | Implement per-jurisdiction status machines via adapter pattern | High |

### P1 — Required for production use

| # | Gap | Recommendation | Effort |
|---|-----|---------------|--------|
| 9 | No T1/T2 transit (EU) | Extend in-bond model to support EU NCTS transit | Medium |
| 10 | No BoE types (India) | Add Bill of Entry type to filing model | Low |
| 11 | No pedimento codes (Mexico) | Add 76 pedimento clave codes and customs regime types | Medium |
| 12 | No CARM integration path (Canada) | Define CARM API/EDI integration architecture | Medium |
| 13 | No VAT/member-state-specific rates (EU) | Add VAT calculation with per-member-state rates | Medium |
| 14 | No ICMS state-specific rates (Brazil) | Add state-level tax calculation for Brazil | Medium |
| 15 | No channel assignment model | Add parametrization (BR), RMS (IN), channel (CN) status model | Medium |
| 16 | No product catalog registration (Brazil) | Add Brazil-specific product catalog workflow | Medium |
| 17 | No GACC registration tracking (China) | Add overseas manufacturer registration validation | Low |

### P2 — Important for completeness

| # | Gap | Recommendation | Effort |
|---|-----|---------------|--------|
| 18 | No export declaration model | Add export declaration domain with per-jurisdiction workflows | High |
| 19 | No Padrón tracking (Mexico) | Add importers registry validation | Low |
| 20 | No AEO/trusted trader per-jurisdiction benefits | Model per-jurisdiction AEO benefits (deferred duty for IN-T2, green channel for BR Linha Azul, etc.) | Medium |
| 21 | No multi-jurisdiction transit | Support goods transiting through multiple jurisdictions (EU T1 → UK, US in-bond → Mexico, etc.) | High |
| 22 | No abandonment/disposal rules per jurisdiction | Different timeframes: US 15-day GO, BR 90-day, IN 30-day | Low |
| 23 | Document types incomplete | Add ENS, SAD, T1/T2, DUIMP, LPCO, NF-e, BoE, Shipping Bill, pedimento to Document domain | Medium |
| 24 | Reference data tables US-only | Add EU TARIC, BR NCM/TEC, IN ITC(HS), CN 10-digit tariff, MX tariff, CA tariff schedules | High |
| 25 | No special regime support per jurisdiction | Model IMMEX (MX), MOOWR (IN), RECOF (BR), Inward Processing (EU), SEZ (IN), Comprehensive Bonded Zone (CN) | High |

---

## 10. Summary Assessment

**The architecture's foundation is excellent.** The 13-domain model, event choreography, HU→Consolidation physical hierarchy, and CQRS pattern are all sound and jurisdiction-agnostic. The hard work of getting the physical/commercial/regulatory separation right has been done.

**The jurisdiction-specific layer needs a fundamentally different approach.** The current `jurisdiction: string` + `jurisdiction_config: {}?` pattern treats non-US customs as a parameterization of the US model. It's not. Each jurisdiction is a **separate implementation** of a **common interface**. The platform needs:

1. A **Jurisdiction Adapter interface** that each country implements
2. **Per-jurisdiction status machines** (not a single enum with all possible values)
3. **Per-jurisdiction duty calculators** (with different formulas, not just different rates)
4. **Per-jurisdiction party validators** (mandatory registrations vary)
5. **Per-jurisdiction document requirements** (different document types and lifecycles)
6. **Per-jurisdiction filing workflows** (different sequences, different participants, different deadlines)

The good news: this can be implemented as an evolution of the current architecture, not a rewrite. The domain boundaries are already right. The adapter pattern plugs into the existing Declaration Management, Financial Settlement, and Customs Adjudication domains without restructuring them.

---

*Review completed 2026-02-08. Every finding is grounded in real regulatory requirements as of 2025-2026. Happy to discuss any jurisdiction in deeper detail.*
