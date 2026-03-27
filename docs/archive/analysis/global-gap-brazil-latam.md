# Brazil & Latin America Customs Operations — Gap Analysis

> **Prepared by:** Brazil/LatAm Customs Operations SME
> **Date:** 2026-02-07
> **Scope:** Comprehensive audit of Clearance Vibe codebase vs. Brazil import/export requirements, with secondary coverage of Mexico and Mercosur
> **Methodology:** File-by-file review of `br.py`, `engine.py`, `broker.py`, `state_machine.py`, `reference_data.py`, `operational.py`, and frontend broker surface

---

## Executive Summary

The existing Brazil tariff engine (`regimes/br.py`) is a **strong proof-of-concept** that correctly implements the most technically challenging aspect of Brazilian customs taxation — the ICMS algebraic grossup. However, reaching first-class parity with the current US CBP support requires work across **five major dimensions**:

1. **Tax regime completeness** — Missing AFRMM, Taxa Siscomex, ICMS-ST, DIFAL, and state incentive programs
2. **Filing system abstraction** — The entire broker workflow is hardcoded to US CBP (entry numbers, CF-28/CF-29, ACE). Brazil uses Siscomex/Portal Unico with DI/DUIMP and the Parametrização channel system
3. **Document & identifier framework** — No support for NF-e, CNPJ, CPF, RADAR, NCM 8-digit, or Brazilian-specific document types
4. **Regulatory agency coverage** — US has FDA/EPA/CPSC/NHTSA mapped; Brazil needs ANVISA, MAPA, INMETRO, IBAMA, Exercito
5. **Simulation & data model** — Corridors, products, carriers, and the state machine are US-centric; no Brazil-inbound corridors exist

**Priority recommendation:** Build the jurisdiction-abstraction layer first (Phase 0), then Brazil filing workflow (Phase 1), then tax completeness (Phase 2), then regulatory agencies (Phase 3), then simulation (Phase 4).

---

## 1. Tariff Engine: Tax Regime Completeness

### 1.1 What Exists (BR Engine)

**File:** `backend/app/engines/e2_tariff/regimes/br.py`

The `BrazilTaxRegime` class correctly implements:

| Tax | Implementation | Status |
|-----|---------------|--------|
| II (Imposto de Importacao) | NCM/HS-6 lookup with 17 product entries | Correct, needs expansion |
| IPI (Imposto sobre Produtos Industrializados) | HS heading lookup, base = CIF + II | Correct |
| PIS-Importacao | Fixed 2.1% on CIF | Correct (Law 10.865/2004) |
| COFINS-Importacao | Fixed 9.65% on CIF | Correct |
| ICMS (state VAT) | Algebraic grossup with 9 states + default | Correct — this is the hardest part |

**What's done well:** The grossup solver (`base_icms = pre_icms_total / (1 - icms_rate/100)`) is correct. The cascading order is correct. State-specific ICMS rates are right for the major importing states.

### 1.2 Missing Taxes & Fees

| Tax/Fee | US Benchmark | Brazil Equivalent | What's Needed | Priority |
|---------|-------------|-------------------|---------------|----------|
| MPF (Merchandise Processing Fee) | 0.3464% clamped $33.58-$651.50 | **Taxa Siscomex** — R$40.00 first addition + R$10.00 per HS code beyond first | Add as FEE line item. Amount = 40 + 10*(n_additions - 1) where n = distinct NCM codes in the DI. Simple formula. | P1 |
| HMF (Harbor Maintenance Fee) | 0.125% on ocean cargo | **AFRMM** (Adicional ao Frete para Renovacao da Marinha Mercante) — **8%** of maritime freight for long-haul, **25%** for cabotage, exempt for overland/air | Add transport-mode-aware fee. Must know ocean freight value (not just CIF). Requires freight breakdown field that doesn't exist in current `declared_value`. | P1 |
| None | N/A | **ICMS-ST** (Substituicao Tributaria) — Applies to specific product categories (electronics, beverages, cosmetics, auto parts). Calculated as `ICMS-ST = (base_ST * icms_rate) - ICMS_operacao_propria`. ST base often uses MVA (Margem de Valor Agregado) multiplier. | Requires per-NCM MVA lookup table, state-by-state ST applicability lists. Complex — hundreds of product-state combinations. | P2 |
| None | N/A | **ICMS-DIFAL** (Diferencial de Aliquota) — Interstate tax differential when goods move from importing state to destination state. Applies when ICMS rate differs between entry port state and final destination. | Need to capture both entry port state AND final destination state. Currently only one `state` param exists. | P2 |
| AD/CVD | Placeholder in US regime | **Direitos Antidumping/Compensatorios** — Brazil's DECOM (Departamento de Defesa Comercial) administers. Active cases on Chinese steel, BOPP film, rubber tires, glassware. | Add lookup table for active AD/CVD cases by NCM + origin. Similar structure to US `ADD_CVD_CASES`. | P2 |
| Section 301/232/IEEPA surcharges | Complex US-China tariff stack | **TEC (Tarifa Externa Comum)** variations — Mercosur common tariff reductions for intra-bloc trade (0% II for AR/PY/UY origin), Ex-tarifario reductions for capital goods/IT without domestic equivalent | Add origin-based tariff reduction logic: Mercosur partners → 0% II; Ex-tarifario approved NCMs → reduced II rate. | P1 |
| None | N/A | **IPI adjustment for Zona Franca de Manaus** — IPI exempt for goods destined to/from ZFM. ICMS also modified. Affects ~25% of electronics imports. | Add ZFM flag on destination; if ZFM, IPI=0, ICMS has special calculation. | P2 |

### 1.3 PIS/COFINS Nuances Not Captured

The engine uses fixed rates (2.1% PIS, 9.65% COFINS). In reality:

- **Monofasico (single-phase) products** — Cosmetics, pharma, auto parts, and beverages have *different* PIS/COFINS rates (often higher). E.g., cosmetics PIS = 2.2%, COFINS = 10.3%.
- **Aliquota zero** — Certain goods (basic food basket items, medical equipment per TIPI list) have PIS/COFINS at 0%.
- **Nota Tecnica adjustments** — Rates change with Receita Federal normative instructions (IN RFB), sometimes multiple times per year.

**Impact:** For the initial POC, fixed rates are acceptable for ~70% of import scenarios. For production accuracy, need a per-NCM PIS/COFINS rate table.

### 1.4 Engine Dispatch (`engine.py`)

The `TariffEngine.execute()` method dispatches to `BrazilTaxRegime` correctly via the `REGIME_MAP`. However:

- **`state` kwarg not forwarded** — `engine.py` line 152 calls `regime.compute()` without passing `**kwargs` through, so the `state` parameter from Brazil's `compute()` signature may not reach the engine. The current code does pass `**kwargs` at line 152 which would forward extra params, so this works — **confirmed OK**.
- **Currency handling** — Brazil transactions are in BRL, but the engine defaults to USD. Need BRL→USD conversion or native BRL support.
- **Freight value** — For AFRMM, the engine needs the freight component of CIF separately. Current `declared_value` is the total CIF. Need to split into FOB + Freight + Insurance.

---

## 2. Filing System Abstraction (Critical Gap)

### 2.1 US Benchmark: What Exists

The entire broker workflow is built around US CBP concepts:

| Component | US Implementation | File |
|-----------|-------------------|------|
| Entry number format | `{filer}-{sequence}-{check}` (US CBP format) | `broker.py:1103-1107` |
| Entry types | `01` (consumption), `11` (informal), `06` (FTZ) | `operational.py:574-576` |
| Filing statuses | draft → pending_broker_approval → submitted → accepted → cf28_pending → cf29_pending → exam_scheduled → released | `operational.py:578-580` |
| CBP response types | CF-28 (Request for Information), CF-29 (Notice of Action) | `cbp_authority.py` |
| Port codes | US port district codes (`2704` = LA/LB) | `reference_data.py` |
| State machine | `entry_filed → cleared/cf28_pending/cf29_pending/exam_scheduled/held` | `state_machine.py:38-56` |
| Dashboard stages | Pre-Filing → Pending Approval → Filed → CBP Response → Released | `broker.py:528-534` |
| Checklist items | 8 US-specific items (Commercial Invoice, Packing List, BL/AWB, HS Classification, Customs Valuation, Certificate of Origin, PGA Docs, Bond Verification) | `broker.py:96-105` |
| PGA agencies | FDA, EPA, CPSC, USDA, NHTSA | `reference_data.py:244-249` |
| Risk assessment | STP fast-lane, UFLPA screening, corridor scrutiny | `customs.py` |
| Bond types | single_transaction, continuous, no_bond | `broker.py:1364` |

### 2.2 Brazil Equivalent: What's Needed

| US Concept | Brazil Equivalent | Implementation Impact |
|-----------|-------------------|----------------------|
| Entry Number `123-4567890-1` | **DI number** (10-digit Siscomex sequence) or **DUIMP number** (Portal Unico format: `BRXXX-NNNN-YYYY/NNNNNN`) | New number format generator per jurisdiction |
| Entry Type 01/11/06 | **Modalidade de despacho**: Normal, Antecipado, Sobre Aguas, Entrega Fracionada, DUIMP (new). Plus export: DU-E. | Need jurisdiction-aware entry type enum |
| CF-28 (Request for Info) | **Exigencia fiscal** — Receita Federal can request additional docs/info via Siscomex. 30-day response window. | Map to equivalent "authority_request" abstraction |
| CF-29 (Notice of Action) | **Auto de Infração** — Tax assessment notice. Can be contested via "impugnacao" (administrative appeal). Very different timeline: 30 days for impugnacao, then CARF (administrative tribunal). | Different appeal workflow — not CF-29 equivalent; needs its own state machine branch |
| Physical Exam | **Canal Vermelho** (Red channel) — document + physical inspection. **Canal Cinza** (Grey channel) — special investigation (fraud suspicion). | Map to parametrização channels |
| STP Fast Lane | **Canal Verde** (Green channel) — auto-release. **Canal Amarelo** (Yellow channel) — document review only (no physical inspection). | Needs 4-channel model vs. US binary (STP vs manual) |
| Bond (continuous/single) | **No bond system** — Brazil uses RADAR license limits instead. RADAR modality determines import volume ceilings. | Replace bond verification with RADAR verification |
| Port of Entry codes | **Unidade da Receita Federal (URF)** codes — e.g., IRF Santos (0817800), IRF Guarulhos (0817600), IRF Paranagua (0917900) | New port code reference table |
| FTA partners | **Mercosur** (AR, PY, UY — 0% II), **ACE agreements** (Chile ACE-35, Colombia/Peru ACE-72, Mexico bilateral), **ALADI** framework | Much more complex origin certificate rules — need ACE number + tariff preference margin by NCM |

### 2.3 Required Filing Abstraction Layer

The current `EntryFiling` model (`operational.py:551-607`) is tightly coupled to US CBP:

```
Current:
  entry_type: "01" | "11" | "06"           ← US entry types
  filing_status: "cf28_pending" etc        ← US-specific statuses
  cbp_response: JSONB                      ← named "cbp_response"
  port_of_entry: US district code          ← US port codes
```

**Needed:** A jurisdiction-polymorphic filing model:

```
Proposed:
  jurisdiction: "US" | "BR" | "MX" etc
  filing_type: jurisdiction-specific enum
  filing_status: jurisdiction-specific enum  (or common + jurisdiction-specific extensions)
  authority_response: JSONB                  (generic name, not "cbp_response")
  filing_port: jurisdiction-specific port code
  filing_identifiers: JSONB {               (jurisdiction-specific IDs)
    # US: {entry_number, filer_code}
    # BR: {di_number, radar_number, cnpj, nf_e_chave, ncm_additions: [...]}
    # MX: {pedimento_number, rfc, clave_pedimento}
  }
```

### 2.4 State Machine Extension

Current `state_machine.py` transitions are US-centric:

```
at_customs → entry_filed → cf28_pending/cf29_pending/exam_scheduled/cleared/held
```

**Brazil needs:**

```
at_customs → registered (DI registered in Siscomex)
  registered → canal_verde (auto-release — GREEN)
  registered → canal_amarelo (document review — YELLOW)
  registered → canal_vermelho (doc + physical inspection — RED)
  registered → canal_cinza (special investigation — GREY)
  canal_verde → desembaracada (cleared)
  canal_amarelo → exigencia_fiscal (info request) | desembaracada
  canal_vermelho → exigencia_fiscal | vistoria (physical exam) | desembaracada
  canal_cinza → investigacao_especial → auto_infracao | desembaracada
  exigencia_fiscal → cumprida (responded) → desembaracada | auto_infracao
  auto_infracao → impugnacao (appeal) → carf (tribunal) → desembaracada | held
```

This requires either:
- **Option A:** Jurisdiction-specific state machine files (recommended)
- **Option B:** A mega state machine with jurisdiction prefixes (not recommended — combinatorial explosion)

---

## 3. Document & Identifier Framework

### 3.1 US Benchmark: Documents

| Checklist Item | US Implementation | Key |
|---------------|-------------------|-----|
| Commercial Invoice | Document upload + status check | `commercial_invoice` |
| Packing List | Document upload + status check | `packing_list` |
| Bill of Lading / AWB | Document upload + status check | `bill_of_lading` |
| HS Classification | Data check (codes JSONB) | `classification` |
| Customs Valuation | Data check (financials JSONB) | `valuation` |
| Certificate of Origin | FTA partner check + doc upload | `origin_cert` |
| PGA Documentation | PGA flags check + doc upload | `pga_docs` |
| Bond Verification | Bond type in financials | `bond_verification` |

### 3.2 Brazil Checklist Requirements

| Checklist Item | Brazil Requirement | Category | Notes |
|---------------|-------------------|----------|-------|
| Nota Fiscal Eletronica (NF-e) | **MANDATORY** — 44-digit access key, XML validated against SEFAZ | `document` | Required for ALL domestic movement of imported goods. Must be issued before cargo leaves customs zone. No US equivalent. |
| Fatura Comercial (Commercial Invoice) | Required, must contain NCM codes (not just HS-6) | `document` | Same as US but must include NCM (8-digit, not 6-digit HS) |
| Romaneio (Packing List) | Required | `document` | Same as US |
| BL/AWB/CRT | Required — Conhecimento de Transporte Internacional | `document` | Same as US but includes CRT (Conocimiento de Transporte por Carretera) for Mercosur ground |
| Classificacao NCM | **8-digit NCM** (not 6-digit HS) — Nomenclatura Comum do Mercosul | `data` | NCM adds 2 digits to HS-6 for Mercosur-specific subdivisions |
| Valoracao Aduaneira | CIF breakdown: FOB + Freight + Insurance. Requires Ficha de Informacao de Importacao when related-party | `data` | More detailed than US — need component breakdown |
| Certificado de Origem | Required for Mercosur, ACE, ALADI preferential rates. Form varies by agreement. | `document` | Different form per trade agreement (Mercosur Form A, ACE bilateral forms) |
| RADAR Verification | RADAR license active + modality (Limitada/Ilimitada/Expressa) + remaining capacity | `data` | **No US equivalent** — RADAR limits import volume by company |
| CNPJ do Importador | 14-digit company registration with Receita Federal | `data` | **No US equivalent** — must validate against CNPJ database |
| Despachante Aduaneiro | Licensed customs broker registration + procuracao (power of attorney) | `data` | Similar to US broker license but requires separate procuracao per importer |
| LI/LPCO | Licenca de Importacao or LPCO on Portal Unico — required for controlled goods before shipment | `document` | **No US equivalent at this level** — must check BEFORE goods ship, not at customs |
| Seguro Internacional | International cargo insurance certificate | `document` | Required element of CIF valuation |
| CNPJ Adquirente | If goods are for account of third party (encomenda/conta e ordem), CNPJ of the actual buyer | `data` | Complex — "importacao por conta e ordem" and "importacao por encomenda" are distinct legal regimes |

### 3.3 Required Identifiers

| Identifier | Format | Validation | Priority |
|-----------|--------|------------|----------|
| CNPJ | `XX.XXX.XXX/XXXX-XX` (14 digits + check) | Modulo-11 checksum algorithm | P1 |
| CPF | `XXX.XXX.XXX-XX` (11 digits + check) | Modulo-11 checksum algorithm | P1 |
| RADAR Number | Alphanumeric, tied to CNPJ | Validate against modality (Limitada ≤ USD 150k/semester, Ilimitada = unlimited, Expressa ≤ USD 50k/semester) | P1 |
| NCM Code | `XXXX.XX.XX` (8 digits) | First 6 = HS code, last 2 = Mercosur specific | P1 |
| DI Number | 10-digit Siscomex sequence (being replaced by DUIMP) | Sequential, assigned at registration | P1 |
| DUIMP Number | Portal Unico format | New system — being phased in | P2 |
| NF-e Access Key | 44 digits | Complex checksum including UF, CNPJ, modelo, serie, numero, tpEmis, cNF, cDV | P1 |
| URF Code | 7 digits (Receita Federal office) | Fixed reference table | P1 |
| Numero Adicao | Sequential per DI addition (1-999) | Each NCM is a separate "adicao" (addition) in the DI | P1 |

---

## 4. Regulatory Agency Coverage

### 4.1 US Benchmark: PGA Agencies

`reference_data.py` maps PGA agencies to product categories:

```python
PGA_TRIGGERS = {
    "FDA": ["pharma", "food"],
    "EPA": ["industrial"],
    "CPSC": ["textiles"],
    "USDA": ["food"],
    "NHTSA": ["automotive"],
}
```

### 4.2 Brazil Regulatory Agencies (Orgaos Anuentes)

| US Agency | Brazil Equivalent | Scope | LPCO Type | Priority |
|-----------|-------------------|-------|-----------|----------|
| FDA | **ANVISA** (Agencia Nacional de Vigilancia Sanitaria) | Pharma, medical devices, cosmetics, food additives, tobacco, sanitizers | LI obrigatoria (mandatory prior license) for pharma; AFE (Autorizacao de Funcionamento) required for importers | P1 |
| USDA/APHIS | **MAPA** (Ministerio da Agricultura, Pecuaria e Abastecimento) | Food, agricultural products, animal products, plant products, fertilizers, seeds | LPCO via SISCOMEX (automated for some products) | P1 |
| EPA | **IBAMA** (Instituto Brasileiro do Meio Ambiente) | Chemicals, pesticides, endangered species (CITES), forestry products, hazardous waste, ozone-depleting substances | CTF (Cadastro Tecnico Federal) registration + LI for controlled substances | P1 |
| CPSC | **INMETRO** (Instituto Nacional de Metrologia) | Toys, electronics, PPE, auto parts, building materials, gas appliances — mandatory certification (Compulsory Conformity Assessment) | Certificate of Conformity (model registration in Inmetro portal) required BEFORE import | P1 |
| NHTSA | **DENATRAN/CONTRAN** + **INMETRO** | Vehicles, auto parts, tires — homologation required for parts affecting safety | LCVM (Licenca para Uso da Configuracao de Veiculo ou Motor) for complete vehicles | P2 |
| ATF (Alcohol, Tobacco, Firearms) | **Exercito Brasileiro** (Brazilian Army, via DFPC) | Weapons, ammunition, explosives, chemical precursors, body armor, night vision | Autorizacao de importacao from Exercito — mandatory | P2 |
| DEA (Drug Enforcement) | **Policia Federal + ANVISA** | Controlled substances, precursor chemicals | Dual authorization required | P2 |
| FCC | **ANATEL** (Agencia Nacional de Telecomunicacoes) | Radio equipment, cellular phones, WiFi devices, Bluetooth — mandatory homologation | Certificate of Conformity from ANATEL — product model must be registered | P1 |
| None (no US equiv) | **ANP** (Agencia Nacional do Petroleo) | Petroleum products, biofuels, natural gas, lubricants | Import authorization from ANP required | P3 |
| None | **CNEN** (Comissao Nacional de Energia Nuclear) | Radioactive materials, nuclear equipment | Import authorization from CNEN | P3 |
| None | **DECEX/SECEX** (trade policy) | Trade policy controls: import quotas, non-automatic licensing, anti-dumping monitoring | Non-automatic LI for watched products | P2 |

### 4.3 Key Difference: Pre-Shipment Licensing

**US model:** PGA requirements are checked **at customs entry** — goods arrive, then PGA flags are raised.

**Brazil model:** Many products require **prior licensing (LI/LPCO) BEFORE shipment from origin**. If goods arrive without the required LI, they are subject to:
- Storage at customs (at importer's cost)
- Multa (fine) of 30% of CIF value for shipment without required LI
- In extreme cases, perdimento (seizure/forfeiture)

**Impact on workflow:** The system needs a "pre-shipment compliance check" that doesn't exist in the US flow:

```
BEFORE BOOKING:
  Check NCM against orgao anuente table
  If LI required:
    Verify LI/LPCO approved in Siscomex before allowing booking
    Store LI number in filing_identifiers
  If LI not required:
    Mark as "licenciamento automatico" (automatic license)
```

---

## 5. Trade Agreements & Preferential Tariffs

### 5.1 US Benchmark

The current system has FTA partner detection:
```python
FTA_PARTNERS_US = {...}  # Referenced in broker.py for origin cert requirements
```

### 5.2 Brazil Trade Agreements

| Agreement | Partners | II Preference | Certificate Required | Priority |
|-----------|----------|---------------|---------------------|----------|
| **Mercosur** (TEC) | Argentina, Paraguay, Uruguay | **0% II** (full preference) | Certificado de Origem Mercosul (Form A) | P1 |
| **ACE-35** | Chile | 0-100% reduction by NCM (phase-in schedule) | Certificado de Origem ACE-35 | P1 |
| **ACE-72** | Colombia, Peru (via CAN-Mercosul) | Partial preference by NCM | Certificado de Origem ACE-72 | P1 |
| **ACE-53** | Mexico (bilateral, limited scope) | Partial preference, specific NCMs only | Certificado de Origem ACE-53 | P2 |
| **ACE-59** | Ecuador, Colombia, Peru, Venezuela | Product-specific margins | ACE-59 form | P2 |
| **Mercosur-Israel** | Israel | Partial preference | Mercosur-Israel FTA certificate | P3 |
| **Mercosur-Egypt** | Egypt | Partial preference | Mercosur-Egypt FTA certificate | P3 |
| **Mercosur-SACU** | South Africa, Namibia, Botswana, Lesotho, Eswatini | Partial preference | PTA certificate | P3 |
| **GSP (SGP)** | Various developing countries exporting TO Brazil | N/A — Brazil is **beneficiary**, not grantor | N/A for imports | N/A |
| **Mercosur-EU** (pending) | EU-27 | Graduated preference schedule | Under ratification — timeline uncertain | P3 |
| **ALADI/LAIA** | All Latin American members | Framework for bilateral negotiations | Varies by bilateral ACE | P2 |

### 5.3 Implementation Requirements

The current origin cert check is binary (FTA partner → cert required, else not). Brazil needs:

1. **Per-NCM preference margins** — Same origin country can have different preference levels by product (unlike US FTAs which are typically all-or-nothing per partner)
2. **Margin-of-preference calculation** — Reduce II by the margin percentage, not necessarily to zero
3. **Rules of origin verification** — Mercosur requires minimum 60% regional content or change of tariff heading (CTH) rules
4. **Certificate form validation** — Different physical certificate forms per agreement

---

## 6. Customs Procedures & Special Regimes

### 6.1 Parametrizacao (Risk Channel System)

**US equivalent:** STP (Simplified Transaction Processing) fast-lane vs manual review

| US Channel | Brazil Equivalent | Description | Simulation Probability |
|-----------|-------------------|-------------|----------------------|
| STP auto-release | **Canal Verde** (Green) | Automatic release, no review | ~55% of DIs |
| Manual doc review | **Canal Amarelo** (Yellow) | Document-only review, no physical inspection | ~25% of DIs |
| Physical exam | **Canal Vermelho** (Red) | Document review + physical inspection | ~15% of DIs |
| N/A (no equivalent) | **Canal Cinza** (Grey) | Special investigation — fraud, transfer pricing, origin fraud. Most feared channel. | ~5% of DIs |

**What determines channel selection:**
- Importer's RADAR history and compliance record
- NCM risk profile
- Origin country risk (China, Paraguay high risk)
- Declared value vs. reference prices (preco de referencia)
- Random sampling
- ANVISA/MAPA products often forced to yellow/red

### 6.2 Special Customs Regimes (Regimes Aduaneiros Especiais)

| Regime | Description | US Equivalent | Priority |
|--------|-------------|---------------|----------|
| **Drawback** (Suspensao/Isencao/Restituicao) | Import duty suspension/exemption/refund for inputs used in export products | US Drawback (19 USC 1313) | P2 |
| **RECOF/RECOF-SPED** | Special customs warehouse regime for approved manufacturers (automotive, electronics, aeronautics) — duties suspended on inputs | US FTZ (Foreign Trade Zone) | P3 |
| **Entreposto Aduaneiro** | Bonded warehouse — duties deferred until goods enter domestic market | US Bonded Warehouse (Type 21) | P2 |
| **Admissao Temporaria** | Temporary admission for goods that will be re-exported (with or without economic utilization) | US TIB (Temporary Importation under Bond) | P2 |
| **Ex-tarifario** | Tariff reduction (typically to 0-2%) for capital goods and IT equipment without domestic equivalent | No direct equivalent | P2 |
| **Linha Azul** (AEO) | Authorized Economic Operator — expedited clearance for compliant importers | US C-TPAT / Trusted Trader | P3 |
| **REPETRO** | Oil & gas sector special regime — duty/tax exemption for offshore equipment | No equivalent | P3 |
| **PADIS** | Semiconductor/display industry incentive | No equivalent | P3 |
| **Simples Nacional** | Simplified tax regime for small companies (faturamento < R$4.8M/year) — unified tax rate | No equivalent | P2 |
| **Zona Franca de Manaus** | Special economic zone — IPI/ICMS/II exemptions for manufacturing in Manaus | US FTZ (but much broader) | P2 |

---

## 7. Export from Brazil

### 7.1 US Benchmark

Current system has no export workflow — it's import-only.

### 7.2 Brazil Export Requirements

| Component | Description | Priority |
|-----------|-------------|----------|
| **DU-E** (Declaracao Unica de Exportacao) | Replaced the old RE+DE with a single unified export declaration on Portal Unico | P2 |
| **NF-e de exportacao** | Special NF-e type (CFOP 7.xxx) with export-specific fields | P2 |
| **Drawback credits** | Track duty credits for inputs used in exported goods | P2 |
| **Reintegra** | Tax rebate program for exporters (1-3% of export FOB value) | P3 |
| **PROEX/BNDES-Exim** | Export financing programs | P3 |
| **Siscomex Exportacao** | Export module of Siscomex | P2 |

---

## 8. Simulation & Data Model Gaps

### 8.1 Corridors

Current corridors in `reference_data.py` are all `→ US` destinations. Need Brazil-inbound corridors:

| Origin | Destination | Export Port | Import Port | Transit Days | Mode | Priority |
|--------|-------------|-------------|-------------|-------------|------|----------|
| CN | BR | Shanghai | Santos | 30-40 | ocean | P1 |
| CN | BR | Shenzhen | Santos | 30-40 | ocean | P1 |
| US | BR | Houston | Santos | 18-25 | ocean | P1 |
| US | BR | Miami | Guarulhos | 1-3 | air | P1 |
| DE | BR | Hamburg | Santos | 20-28 | ocean | P1 |
| DE | BR | Frankfurt | Guarulhos | 1-3 | air | P1 |
| AR | BR | Buenos Aires | Santos | 3-5 | ocean | P1 |
| AR | BR | Buenos Aires | Foz do Iguacu | 2-4 | ground | P1 |
| KR | BR | Busan | Santos | 35-45 | ocean | P2 |
| JP | BR | Yokohama | Santos | 35-45 | ocean | P2 |
| PY | BR | Ciudad del Este | Foz do Iguacu | 1-2 | ground | P2 |

### 8.2 Products

Need Brazil-relevant products with NCM codes (8-digit):

| Product | NCM | Category | PGA-BR | Notes |
|---------|-----|----------|--------|-------|
| Auto Parts Kit (Suspensao) | 8708.80.00 | automotive | INMETRO | High volume CN→BR |
| Soja em Graos (Soybeans) | 1201.90.00 | food | MAPA | Major BR export (for export workflow) |
| Celular Smartphone | 8517.13.00 | electronics | ANATEL | ANATEL certification required |
| Medicamento Generico | 3004.90.99 | pharma | ANVISA | LI mandatory pre-shipment |
| Vinho Tinto | 2204.21.00 | beverages | MAPA | High IPI + ICMS-ST |
| Tecido Sintetico | 5407.61.00 | textiles | None | AD/CVD potential from CN |
| Peca Aeronautica | 8803.30.00 | aerospace | INMETRO+CTA | RECOF/REPETRO eligible |
| Brinquedo Plastico | 9503.00.99 | toys | INMETRO | INMETRO certification mandatory |
| Fertilizante NPK | 3105.20.00 | industrial | MAPA | MAPA registration required |

### 8.3 Simulation Actors

Need Brazil-equivalent actors:

| US Actor | Brazil Equivalent | What Changes |
|----------|-------------------|-------------|
| `cbp_authority.py` | `receita_federal.py` | Parametrizacao channel selection (4 channels vs binary), exigencia fiscal (vs CF-28), auto de infracao (vs CF-29), 30-day response deadlines |
| `customs.py` | `aduana_br.py` | UFLPA screening → No equivalent; STP → Canal Verde; corridor scrutiny → preco de referencia checks |
| `pga.py` | `orgaos_anuentes.py` | FDA/EPA → ANVISA/MAPA/IBAMA/INMETRO/ANATEL; pre-shipment LI checks |
| `broker_sim.py` | `despachante.py` | Different checklist, different response formats, NF-e generation |
| `financial.py` | `financeiro_br.py` | AFRMM calculation, Taxa Siscomex, state tax incentive application |
| `isf.py` | N/A | ISF (Importer Security Filing) is US-specific. No Brazil equivalent — can skip. |

### 8.4 Brazilian Ports Reference Data

| URF Code | Port Name | State | Primary Mode |
|----------|-----------|-------|-------------|
| 0817800 | Santos | SP | ocean |
| 0817600 | Guarulhos (Cumbica Airport) | SP | air |
| 0817100 | Campinas (Viracopos Airport) | SP | air |
| 0717600 | Rio de Janeiro (Galeao Airport) | RJ | air |
| 0710800 | Rio de Janeiro (Porto) | RJ | ocean |
| 0917900 | Paranagua | PR | ocean |
| 0517200 | Salvador | BA | ocean |
| 1017700 | Porto Alegre | RS | air/ocean |
| 0917800 | Curitiba | PR | air |
| 0417600 | Recife | PE | air/ocean |
| 1323300 | Manaus (ZFM) | AM | air/ocean |
| 0917100 | Foz do Iguacu | PR | ground |
| 1010200 | Uruguaiana | RS | ground |

---

## 9. Frontend Broker Surface

### 9.1 Current US-Centric Components

All frontend components in `frontend/src/surfaces/broker/` are US-specific:

| Component | US-Specific Elements |
|-----------|---------------------|
| `BrokerDashboard.tsx` | Status badges: "CF-28", "CF-29", "Exam". Pipeline stages: "Pre-Filing → Pending Approval → Filed → CBP Response → Released" |
| `EntryDetail.tsx` | ACRONYM_MAP has CBP, UFLPA, NII. Checklist keys are US-specific. CBP response modal. |
| `CBPResponses.tsx` | Entirely US CBP-specific: CF-28 questions, CF-29 protests, exam scheduling |
| `BrokerQueue.tsx` | Status filter options are US statuses |
| `Communications.tsx` | References CBP communication |
| `BrokerAssistant.tsx` | AI assistant likely trained on US customs context |

### 9.2 What Needs to Change

1. **Jurisdiction-aware component rendering** — Dashboard should show Brazil-appropriate stages ("Registro DI → Parametrização → Canal → Desembaraço") when viewing BR entries
2. **Checklist component** — Must render NF-e, RADAR, CNPJ fields instead of US checklist
3. **Authority response component** — Replace CBPResponses with jurisdiction-generic "AuthorityResponse" that renders Exigencia Fiscal for BR, CF-28 for US
4. **Status badges** — Need `canal_verde`, `canal_amarelo`, `canal_vermelho`, `canal_cinza`, `exigencia_fiscal`, `auto_infracao` in addition to US statuses
5. **Tax breakdown display** — Must show II → IPI → PIS → COFINS → ICMS cascade (exists in tariff engine but no frontend rendering for BR-specific tax stack)
6. **Port selector** — US port codes → URF codes for BR

---

## 10. Mexico & Mercosur Partners (Brief Coverage)

### 10.1 Mexico

| Area | Mexico Implementation | Impact |
|------|----------------------|--------|
| Filing system | **Pedimento** (customs declaration) via VUCEM (Ventanilla Unica) | New filing type + state machine |
| Tax authority | **SAT** (Servicio de Administracion Tributaria) | Authority actor |
| Identifiers | **RFC** (Registro Federal de Contribuyentes), **Patente de agente aduanal** (broker license) | Identifier framework |
| Tariffs | **LIGIE** (Ley del Impuesto General de Importacion y Exportacion) + **DTA** (Derecho de Tramite Aduanero) + **IVA** (16%) + **IEPS** (special excise) | New tax regime engine |
| IMMEX/Maquila | Temporary import for manufacturing/export — similar to US FTZ but much larger scale (~6,000 IMMEX companies) | Special regime handling |
| T-MEC/USMCA | Already partially covered in US FTA detection | Extend to MX→US origin cert |
| Customs channels | Semaforo fiscal (Green/Red) + second review | Binary channel system |

### 10.2 Argentina

| Area | Argentina Implementation | Impact |
|------|-------------------------|--------|
| Filing system | **SIMI/SIRA** (Sistema de Importaciones de la Republica Argentina) — mandatory pre-approval for ALL imports since 2022 | Blocking pre-import approval system |
| Tax authority | **AFIP** (Administracion Federal de Ingresos Publicos) / **DGA** (Direccion General de Aduanas) | Authority actor |
| Identifiers | **CUIT** (Clave Unica de Identificacion Tributaria — 11 digits), **Despachante de Aduana** registration | Identifier framework |
| FX controls | Central Bank approval for FX purchases to pay imports — **unique globally**. Payment delays of 30-180 days enforced. | Financial workflow addition |
| Tariffs | Same TEC as Brazil (Mercosur harmonized) but with Argentina-specific **Tasa Estadistica** (3% statistical fee, capped USD 150k) + **IVA** (21%) + **IVA Adicional** (20% or 10%) + **Ingresos Brutos** (provincial, 0-5%) + **Percepcion Ganancias** (6-11%) | New cascading tax engine (Pattern C — more additive than Brazil) |

### 10.3 Mercosur-Wide Considerations

| Area | Description | Priority |
|------|-------------|----------|
| **NCM harmonization** | All Mercosur countries use 8-digit NCM — same first 6 digits (HS), shared last 2. NCM tables synchronized via Mercosur resolutions. | P1 — Use NCM as base for all Mercosur jurisdictions |
| **Mercosur origin rules** | 60% regional content requirement, change of tariff heading (CTH), or qualifying process | P2 |
| **ACE preferences** | Partial-scope agreements with varying margins by NCM — need a preference margin table | P2 |
| **TEC exceptions** | Each Mercosur member maintains a list of TEC exceptions (LETEC) — products where they deviate from the common tariff | P2 |

---

## 11. Priority Implementation Roadmap

### Phase 0: Jurisdiction Abstraction Layer (Foundation) — P0

**Must do before any Brazil-specific work.**

1. Add `jurisdiction` field to `EntryFiling` model
2. Rename `cbp_response` → `authority_response` (with backward-compatible alias)
3. Create jurisdiction-specific state machine files
4. Add `filing_identifiers` JSONB field for jurisdiction-specific IDs
5. Make checklist items jurisdiction-configurable (not hardcoded 8 items)
6. Abstract port code reference into jurisdiction-keyed tables

### Phase 1: Brazil Filing Workflow — P1

1. **Siscomex filing state machine** — 4-channel parametrizacao flow
2. **DI/DUIMP registration** — Number generation, adicao management
3. **Document checklist** — NF-e, CNPJ, RADAR verification, LI/LPCO tracking
4. **Identifier validation** — CNPJ, CPF, RADAR, NCM validators
5. **Port reference data** — URF codes for major Brazilian ports

### Phase 2: Tax Engine Completion — P1-P2

1. **AFRMM** — Maritime freight surcharge (needs freight value field)
2. **Taxa Siscomex** — Per-addition fee
3. **Mercosur 0% II** — Origin-based tariff reduction for AR/PY/UY
4. **Ex-tarifario rates** — Reduced II for approved capital goods
5. **ICMS-ST** — Product + state MVA lookup (complex, large dataset)
6. **PIS/COFINS variations** — Per-NCM rate table for monofasico products

### Phase 3: Regulatory Agencies — P1-P2

1. **ANVISA** — Pharma/medical/cosmetics prior licensing
2. **MAPA** — Agriculture/food import controls
3. **INMETRO** — Certification requirement by NCM
4. **ANATEL** — Telecom homologation
5. **IBAMA** — Environmental controls
6. **Pre-shipment LI check** — New workflow step before booking

### Phase 4: Simulation — P2

1. **Brazil-inbound corridors** — CN→BR, US→BR, DE→BR, AR→BR
2. **Products with NCM** — Brazil-relevant product catalog
3. **Receita Federal actor** — 4-channel parametrizacao simulation
4. **Orgaos anuentes actor** — ANVISA/MAPA/INMETRO checks
5. **Despachante actor** — Brazilian customs broker behavior

### Phase 5: Frontend — P2

1. **Jurisdiction-aware dashboard** — Pipeline stages by jurisdiction
2. **BR checklist component** — NF-e, RADAR, CNPJ fields
3. **Authority response abstraction** — Exigencia Fiscal UI
4. **BR tax breakdown display** — Cascading tax visualization
5. **Parametrizacao channel display** — Green/Yellow/Red/Grey channel badges

---

## 12. Risk & Complexity Assessment

| Area | Complexity | Risk | Notes |
|------|-----------|------|-------|
| ICMS grossup (done) | HIGH | LOW | Already implemented correctly |
| ICMS-ST | VERY HIGH | HIGH | Hundreds of product-state combinations, frequent changes |
| NF-e integration | HIGH | MEDIUM | 44-digit key generation, SEFAZ validation (mock for simulation) |
| RADAR validation | MEDIUM | LOW | Simple modality + ceiling check |
| Parametrizacao channels | MEDIUM | LOW | 4-way probability split |
| LI/LPCO pre-shipment | HIGH | HIGH | Fundamentally different from US post-arrival model |
| Mercosur preferences | MEDIUM | LOW | Straightforward 0% II for partner origins |
| ACE partial preferences | HIGH | MEDIUM | Per-NCM margin tables needed |
| AFRMM | LOW | LOW | Simple percentage on freight value |
| State machine abstraction | HIGH | MEDIUM | Affects core architecture |
| Frontend jurisdiction abstraction | HIGH | MEDIUM | Pervasive US references in component tree |

---

## 13. Quick Wins (Can Do Now)

1. **AFRMM line item** in `br.py` — Simple: add freight_value param, compute 8% for ocean mode
2. **Taxa Siscomex line item** — Simple: add n_additions param, compute R$40 + R$10*(n-1)
3. **Mercosur 0% II** — Check origin_country in {AR, PY, UY}, set II to 0% with Mercosur TEC citation
4. **NCM validation** — Add 8-digit validator alongside existing HS-6 handling
5. **CNPJ/CPF validators** — Pure functions, no infrastructure needed
6. **URF port code reference** — Add to reference_data.py alongside US_PORT_CODES
7. **Missing ICMS states** — Add remaining 18 states to ICMS_RATES table (AM, GO, DF, ES, MS, MT, PA, MA, PI, AL, SE, RN, PB, TO, RO, AC, AP, RR)

---

*This analysis reflects the regulatory landscape as of February 2026. Brazilian customs regulations change frequently — the Receita Federal issues dozens of normative instructions (IN RFB) per year. The DUIMP transition from DI is ongoing and may accelerate. State-level ICMS changes happen via CONFAZ protocols and convenios, often with short notice.*
