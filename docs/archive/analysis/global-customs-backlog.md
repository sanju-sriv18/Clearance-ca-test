# Global Customs Operations — Supplemental Backlog

**Synthesized from**: 5 global gap analyses (EU/UK, India, Brazil/LatAm, China/APAC, Unified UX Architecture)
**Existing backlog**: 72 US-focused items in `ux-analysis-backlog.md`
**This supplement**: 55 items (GL-001 through GL-055) for multi-jurisdiction parity
**Date**: 2026-02-07

---

## 1. Executive Summary

The Clearance Vibe platform has **world-class tariff engines for 5 jurisdictions** (US, EU, CN, BR, IN) and a powerful multi-modal routing engine — but the entire operational workflow is US-only. Every broker screen, every filing state, every document checklist, every simulation actor, and every agent tool assumes US CBP. The tariff engines prove the architecture can handle global tax complexity; the challenge is extending that same quality to declarations, documents, regulatory agencies, simulation, and the broker UX.

**What exists globally today**:
- **US**: Full lifecycle — CF-7501, CF-28/CF-29, ISF, PGA, bond management, 19 simulation actors, broker dashboard
- **EU**: MFN duty + VAT calculation only (`EUTaxRegime`)
- **India**: BCD + SWS + IGST calculation only (`IndiaTaxRegime`, ~55 seeded ITC-HS codes)
- **Brazil**: II + IPI + PIS + COFINS + ICMS with correct algebraic grossup (`BrazilTaxRegime`)
- **China**: MFN + 125% retaliatory + consumption tax + VAT (`ChinaTaxRegime`)

**What's missing everywhere except US**: Customs declaration filing, authority response handling, required identifiers (EORI/IEC/CNPJ), document types, trade agreement preferences, regulatory agency coverage, simulation actors, agent tools, and frontend surfaces.

This supplement ensures **every jurisdiction is first-class** — not an afterthought bolted onto a US-centric platform.

---

## 2. Jurisdiction Parity Matrix

| Capability | US | EU | UK | India | Brazil | China | Japan | Korea |
|---|---|---|---|---|---|---|---|---|
| **Tariff engine** | Full (MFN+301+232+IEEPA+AD/CVD+MPF+HMF) | Basic (MFN+VAT) | None (falls through) | Basic (BCD+SWS+IGST) | Good (II+IPI+PIS+COFINS+ICMS) | Good (MFN+retaliatory+consumption+VAT) | None | None |
| **Declaration filing** | Full (CF-7501 lifecycle) | None | None | None | None | None | None | None |
| **Authority responses** | Full (CF-28/CF-29/exam) | None | None | None | None | None | None | None |
| **Required identifiers** | EIN/broker license | None (needs EORI/AEO) | None (needs UK EORI) | None (needs IEC/GSTIN) | None (needs CNPJ/RADAR) | None (needs USCC) | None | None |
| **Document types** | 8-item checklist | None EU-specific | None UK-specific | None India-specific | None Brazil-specific | None CN-specific | None | None |
| **FTA/preferences** | FTA_PARTNERS_US only | None | None | None | None | None | None | None |
| **Regulatory bodies** | FDA/EPA/CPSC/NHTSA/APHIS | None | None | None | None | None | None | None |
| **Simulation actors** | 19 actors | None | None | None | None | CN as origin only | None | None |
| **Agent tools** | US-specific | None | None | None | None | None | None | None |
| **Frontend broker UI** | Full (100% US-hardcoded) | None | None | None | None | None | None | None |
| **Import corridors** | All terminate at US | None (no EU-bound) | None | 1 export only (IN→US) | None (no BR-bound) | CN as origin only | JP as origin only | KR as origin only |

**Legend**: Full = production-ready | Good = tariff math works | Basic = minimal tariff only | None = not implemented

---

## 3. Global Architecture Requirements

### The Jurisdiction Abstraction Layer

The unified UX analysis identified the core architectural pattern: **jurisdiction is a property of each declaration, not a mode of the application**. One adaptive interface, not separate US/EU/India screens.

#### Key Abstractions Required

1. **`JurisdictionConfig` registry** — TypeScript interface mapping jurisdiction code to: customs authority name, declaration form, importer ID label, tax component labels, checklist items, status lifecycle, regulatory agencies, guarantee/bond config, and filing system name

2. **`CustomsDeclaration` polymorphic model** — Replace US-specific `EntryFiling` with a base model supporting US entries (`CF-7501`), EU declarations (`SAD/CDS`), India Bills of Entry, Brazil DI/DUIMP, and China customs declarations. Core fields universal; jurisdiction-specific fields in JSONB extension

3. **Jurisdiction-specific state machines** — EU lifecycle (`pre-filed → accepted → risk_assessment → release/query/examination`) does not map 1:1 to US (`entry_filed → cf28_pending/cf29_pending/exam_scheduled`). Separate state machines per jurisdiction, not one mega-machine

4. **Generalized authority response framework** — Rename `cbp_response` to `authority_response`. Abstract CF-28/CF-29 into inquiry/notice pattern that maps to EU customs queries, India assessment queries, Brazil exigencia fiscal, and China price verification

5. **Jurisdiction-aware checklist** — Universal items (commercial invoice, packing list, transport document, classification, valuation) plus jurisdiction-specific additions (US: bond/PGA/origin cert; EU: EORI/CE/REACH/EUR.1; India: IEC/BIS/FSSAI; Brazil: RADAR/NF-e/LI)

6. **Adaptive terminology** — `useJurisdiction(destination)` hook drives labels: "Submit to CBP" vs "Submit to Customs" vs "File via ICEGATE" vs "Register via SISCOMEX"

---

## 4. Prioritized Backlog (GL-001 through GL-055)

### P0 — Jurisdiction Framework Foundation (Must-Have for Any Non-US Support)

| ID | Title | What's Missing | Why It Matters for Global Credibility | Jurisdictions | Backend Foundation | Changes Needed | Effort | Priority |
|---|---|---|---|---|---|---|---|---|
| GL-001 | **Jurisdiction detection from shipment destination** | No mechanism to determine customs jurisdiction from destination country | Without this, the entire platform cannot adapt to non-US trade lanes. EU members, UK, India, Brazil, China all need detection | ALL | `REGIME_MAP` dispatch exists in tariff engine; `EU_MEMBERS` frozenset exists | Backend: jurisdiction resolver function; Frontend: `useJurisdiction()` hook consuming destination country | S | P0 |
| GL-002 | **Polymorphic declaration model (EntryFiling → CustomsDeclaration)** | `EntryFiling` hardcoded to US: entry_type "01", cbp_response JSONB, US port codes | A broker cannot file an EU declaration, Indian Bill of Entry, or Brazilian DI using the current model. No non-US declaration can be represented | ALL | `EntryFiling` model in `operational.py` | Backend: add `jurisdiction`, `jurisdiction_config` JSONB, generalize `cbp_response` → `authority_response`; migrate existing data as `jurisdiction="US"` | L | P0 |
| GL-003 | **EORI identifier support for EU/UK trade** | No EORI field anywhere. EORI is mandatory for ALL EU import/export | Without EORI, no EU customs declaration can be filed. Every economic operator in EU trade must have one | EU, UK | `Broker` model has `license_number` (US-only) | Backend: `JurisdictionCredential` model or EORI field on Broker/Importer; EORI format validation (country prefix + 15 chars) | M | P0 |
| GL-004 | **IEC/GSTIN/AD Code for India trade** | No India-specific identifier fields. IEC is mandatory for every Indian import/export | Without IEC and GSTIN, no Bill of Entry can be filed. These are legally required for ALL Indian trade | IN | No India identifiers exist | Backend: IEC (10-digit), GSTIN (15-digit), AD Code fields on importer/company models; validation logic | M | P0 |
| GL-005 | **CNPJ/RADAR for Brazil trade** | No Brazil-specific identifiers. CNPJ is mandatory company registration; RADAR controls import volume limits | Without CNPJ and RADAR, no DI can be registered in Siscomex. RADAR modality (Limitada/Ilimitada/Expressa) determines import ceiling | BR | No Brazil identifiers exist | Backend: CNPJ (14-digit with modulo-11 check), RADAR number and modality fields; validation logic | M | P0 |
| GL-006 | **UK separated from EU tariff engine** | UK routes through `EUTaxRegime` or falls through as unsupported. UK tariff rates diverged from EU CET since Jan 2021 | UK duty calculations are wrong. Post-Brexit UK Global Tariff has different rates from EU CET. Using EU rates for UK = incorrect duties | UK | `EUTaxRegime` exists; UK is NOT in `EU_MEMBERS` frozenset, so `GB` actually falls through | Backend: new `UKTaxRegime` class in `regimes/uk.py`; dispatch `GB` to UK regime in `TariffEngine._get_regime()` | L | P0 |
| GL-007 | **Jurisdiction-aware document checklist** | `CHECKLIST_LABELS` and `_compute_checklist_state()` are hardcoded to 8 US items | The broker checklist — the core workflow — shows US documents for EU/India/Brazil entries. EORI verification, BIS certificate, NF-e are invisible | ALL | Checklist computation in `broker.py`; `DOCUMENT_FIELD_SCHEMAS` in `documents.py` | Backend: refactor `_compute_checklist_state()` to load items from jurisdiction config; Frontend: render dynamic checklist | L | P0 |
| GL-008 | **Frontend jurisdiction-aware rendering** | 111+ references to CBP-specific concepts across 7 broker surface files. "Submit to CBP", "CF-28", "CF-29" hardcoded | The broker UI is unusable for non-US trade. An EU broker sees "Submit to CBP" for a German-bound shipment | ALL | Frontend components exist but with US labels | Frontend: replace hardcoded labels with `JurisdictionConfig` lookups; parameterize `StatusBadge`, `BrokerNav`, `EntryDetail`, `BrokerDashboard` | L | P0 |
| GL-009 | **Generalized customs authority response screen** | `CBPResponses.tsx` is entirely US CBP-specific. No EU customs query, India assessment query, or Brazil exigencia fiscal UI | Brokers cannot respond to non-US customs authority inquiries. The response workflow is the second-most-used broker screen | ALL | CBP response handling in `cbp_authority.py`; response modal in `CBPResponses.tsx` | Frontend: abstract to `RegulatoryInquiries` screen with jurisdiction-dispatched response modals; Backend: generic inquiry/response endpoints | L | P0 |
| GL-010 | **Jurisdiction-specific state machines** | `state_machine.py` has US-only transitions: `entry_filed → cf28_pending/cf29_pending/exam_scheduled` | EU declaration lifecycle, India RMS channels, Brazil parametrizacao channels cannot be represented | ALL | State machine in `state_machine.py` | Backend: jurisdiction-specific state machine files or jurisdiction-context dispatch; new states per jurisdiction | L | P0 |

### P1 — Declaration Filing & Tax Completeness (Required for Each Jurisdiction to Function)

| ID | Title | What's Missing | Why It Matters for Global Credibility | Jurisdictions | Backend Foundation | Changes Needed | Effort | Priority |
|---|---|---|---|---|---|---|---|---|
| GL-011 | **EU customs declaration workflow (H1-H7)** | No EU declaration types, no SAD/CDS forms, no MRN generation, no procedure codes | EU import filing is completely non-functional. H1 (free circulation) is the EU equivalent of US CF-7501 | EU | `EUTaxRegime` computes tariff math | Backend: EU declaration model with declaration_type (H1-H7), procedure_code, MRN, customs_office_code; EU filing state machine | XL | P1 |
| GL-012 | **UK CDS declaration workflow** | No UK Customs Declaration Service support. CDS uses Data Elements model, distinct from EU SAD | UK import filing is non-functional. CDS replaced CHIEF entirely | UK | None (UK falls through) | Backend: UK CDS declaration model, UK procedure codes, DUCR/MUCR, deferment account; depends on GL-006 | XL | P1 |
| GL-013 | **India Bill of Entry / Shipping Bill workflow** | No ICEGATE filing model. No BE number, no BE types (home consumption/warehousing/ex-bond) | India import/export filing is non-functional. Bill of Entry is India's fundamental import declaration | IN | `IndiaTaxRegime` computes BCD+SWS+IGST | Backend: Bill of Entry model (BE number, type, port_code, IEC, GSTIN, CHA license), Shipping Bill model; ICEGATE state machine | XL | P1 |
| GL-014 | **Brazil DI/DUIMP filing workflow** | No Siscomex integration model. No DI number, no adicao management, no parametrizacao channels | Brazil import filing is non-functional. DI registration in Siscomex is the Brazilian customs declaration | BR | `BrazilTaxRegime` computes cascading taxes correctly | Backend: DI model (DI number, RADAR, CNPJ, NF-e access key, NCM additions), 4-channel parametrizacao state machine (green/yellow/red/grey) | XL | P1 |
| GL-015 | **China GACC declaration workflow** | No Chinese customs declaration form (报关单). Only stub in `TERRITORY_FILINGS` | China — the world's largest trading nation — has no filing capability beyond tariff calculation | CN | `ChinaTaxRegime` computes MFN+retaliatory+consumption+VAT; territory stubs exist | Backend: GACC declaration model (50+ fields), trade mode codes (0110 general, 0214/0615 processing), USCC identifier, Single Window response simulation | XL | P1 |
| GL-016 | **EU FTA partners and preference rates** | No `FTA_PARTNERS_EU`. EU has the world's most extensive FTA network (EU-UK TCA, EU-Japan EPA, EU-Korea, CETA, PEM zone, GSP for ~70 countries) | FTA utilization saves 3-15% duty. EU FTAs are far more complex than US FTAs — involve tariff quotas, cumulation rules, product-specific rules of origin | EU, UK | `FTA_PARTNERS_US` exists as pattern; `EUTaxRegime` computes MFN only | Backend: `FTA_PARTNERS_EU`, `FTA_PARTNERS_UK` dicts; preference rate lookup in tariff engine; origin proof type determination (EUR.1 vs REX vs invoice declaration) | L | P1 |
| GL-017 | **India FTA partners and preference rates** | No `FTA_PARTNERS_IN`. India-ASEAN CECA, India-Japan CEPA, India-Korea CEPA, India-UAE CEPA (2022), India-Australia ECTA (2022) all missing | FTA utilization is the #1 cost-saving opportunity for India importers. Preferential BCD can drop from 15-70% to 0-5% | IN | `FTA_PARTNERS_US` exists as pattern; `IndiaTaxRegime` has no preference logic | Backend: `FTA_PARTNERS_IN` dict; Certificate of Origin types per agreement (Form AI, Form AIJCEPA, etc.); preferential BCD rate schedule tables | L | P1 |
| GL-018 | **China RCEP and FTA preference rates** | No `FTA_PARTNERS_CN`. RCEP (world's largest FTA — 15 countries, 30% of global GDP) entirely missing | RCEP is the single most important trade agreement for APAC. Covers cumulation across 15 countries | CN, APAC | `FTA_PARTNERS_US` pattern; `ChinaTaxRegime` has no preference logic | Backend: `FTA_PARTNERS_CN` with RCEP, ACFTA (Form E), China-Korea, ChAFTA, China-NZ, China-Switzerland rates; RCEP cumulation rules | L | P1 |
| GL-019 | **India BCD database-backed lookup** | `IndiaTaxRegime.BCD_RATES` has 17 hardcoded rates that **disagree** with `seed_in_tariff.py` (coffee: 30% vs 100%; laptops: 0% vs 15%; toys: 20% vs 60%) | Incorrect duty calculations for India. Hardcoded rates and seed data conflict. Need single source of truth | IN | `_lookup_mfn_db()` pattern exists in `USTaxRegime` | Backend: `_lookup_bcd_db()` method mirroring US pattern; reconcile BCD_RATES vs seed data; expand to DB-backed lookup | M | P1 |
| GL-020 | **India IGST slab expansion** | Only 18% and 28% IGST rates. Missing 0%, 5%, and 12% slabs | 5% IGST applies to basic food, fertilizers; 12% to processed food, apparel. Incorrect IGST = wrong landed cost | IN | `IndiaTaxRegime` has 18%/28% only | Backend: per-HS-code IGST slab lookup (0/5/12/18/28%); add Compensation Cess for motor vehicles, tobacco, coal | M | P1 |
| GL-021 | **EU VAT mechanisms (PVA, Procedure 42)** | No Postponed VAT Accounting, no Procedure 42 VAT exemption, no reduced VAT rates | PVA eliminates upfront VAT payment at border (available in IE, NL, BE, UK). Procedure 42 = zero import VAT for intra-EU dispatch. Missing these overstates cash flow requirements | EU | `EUTaxRegime` has standard VAT rates for 27 MS | Backend: PVA flag in tariff computation; Procedure 42/63 VAT exemption logic; reduced/zero VAT rate lookup by HS code and member state | M | P1 |
| GL-022 | **Brazil AFRMM and Taxa Siscomex** | AFRMM (8% of maritime freight for long-haul) and Taxa Siscomex (R$40 + R$10 per additional NCM) missing | These are mandatory fees on every Brazil import. AFRMM needs freight value field that doesn't exist in current `declared_value` | BR | `BrazilTaxRegime` has correct tax cascade; no freight breakdown field | Backend: AFRMM calculation (8% ocean freight / 25% cabotage / 0% air); Taxa Siscomex (R$40 + R$10*(n_additions-1)); requires CIF component breakdown (FOB + freight + insurance) | M | P1 |
| GL-023 | **China provisional/interim tariff rates** | Only MFN rates. China frequently sets provisional rates lower than MFN for specific goods — change semi-annually | Provisional rates override MFN and are the actual applied rates for hundreds of tariff lines. Missing them = overstated duties | CN | `ChinaTaxRegime` has MFN lookup | Backend: provisional rate table; rate selection logic (provisional overrides MFN when available); semi-annual update mechanism | M | P1 |
| GL-024 | **Japan tariff regime** | No `JapanTaxRegime`. No JP tariff computation at all | Japan is the world's 4th largest economy. No tariff calculation for JP-destined goods | JP | None — no JP regime exists | Backend: new `JapanTaxRegime` in `regimes/jp.py` — MFN duty + consumption tax (10% standard, 8% reduced for food); dispatch in `TariffEngine` | L | P1 |
| GL-025 | **South Korea tariff regime** | No `KoreaTaxRegime`. No KR tariff computation | Korea is a major US trade partner (KORUS FTA). No tariff calculation for KR-destined goods | KR | None — no KR regime exists | Backend: new `KoreaTaxRegime` in `regimes/kr.py` — MFN duty + VAT (10%); individual consumption tax for luxury goods | L | P1 |
| GL-026 | **ICS2 ENS pre-arrival filing (EU equivalent of ISF)** | No Entry Summary Declaration support. ICS2 Release 3 requires ALL transport modes to file pre-arrival ENS | ICS2 is mandatory for all EU imports. Airlines must submit PLACI data. EU can issue Do-Not-Load instructions before shipment | EU | `ISFActor` exists for US ocean ISF; ICS2 is analogous but broader | Backend: ENS filing model, PLACI tracking, multi-member-state filing for transshipment, DNL event handling | L | P1 |
| GL-027 | **EU sanctions screening integration** | Entity screening only checks US lists (OFAC SDN, BIS Entity List, DPL). No EU Consolidated List, no UK OFSI | Platform gives false compliance clearance for EU-destined goods. EU sanctions list differs from US — entities sanctioned by EU may not be on OFAC SDN | EU, UK | `screen_entity` tool screens US lists | Backend: add EU Consolidated Financial Sanctions List, EU dual-use list (Annex I), UK OFSI Consolidated List to screening engine | M | P1 |

### P1 — Regulatory Agency Coverage

| ID | Title | What's Missing | Why It Matters for Global Credibility | Jurisdictions | Backend Foundation | Changes Needed | Effort | Priority |
|---|---|---|---|---|---|---|---|---|
| GL-028 | **India BIS/FSSAI/DGFT regulatory triggers** | No India PGA equivalents. BIS (electronics, toys, helmets — mandatory CRS certification), FSSAI (food safety), DGFT (import licensing) not modeled | BIS certification is the single biggest non-tariff barrier for India imports (~400+ mandatory products). FSSAI is mandatory for all food imports | IN | `PGA_TRIGGERS` maps US agencies; `PGAActor` generates US PGA holds | Backend: `PGA_TRIGGERS_IN` mapping BIS→Ch.84/85/95, FSSAI→Ch.01-24, CDSCO→Ch.30/90, WPC→8517/8525/8527, DGFT→restricted list | M | P1 |
| GL-029 | **EU CE/UKCA marking and REACH compliance** | No CE marking requirement tracking. No REACH substance pre-registration/registration checks | CE marking is mandatory for products sold in EU (machinery, electronics, medical devices, toys). REACH affects HS Ch.28-39 for chemicals | EU, UK | No EU regulatory agency mapping exists | Backend: CE marking requirement by HS code + applicable directive; REACH SVHC list matching; UKCA marking for UK | M | P1 |
| GL-030 | **Brazil ANVISA/MAPA/INMETRO/ANATEL triggers** | No Brazil regulatory agency coverage. ANVISA (pharma — mandatory LI pre-shipment), MAPA (agriculture), INMETRO (certification), ANATEL (telecom) | Brazil requires **pre-shipment licensing** for many products — fundamentally different from US post-arrival PGA model. Shipping without required LI = 30% CIF fine | BR | `PGA_TRIGGERS` for US; no Brazil equivalent | Backend: `PGA_TRIGGERS_BR` mapping agencies to NCM codes; pre-shipment LI check workflow (new concept — check BEFORE booking, not at customs) | L | P1 |
| GL-031 | **China CCC/CIQ/NMPA regulatory triggers** | No China regulatory agency mapping. CCC (compulsory certification for electronics, auto, toys), CIQ (inspection/quarantine), NMPA (pharma, medical devices, cosmetics) | CCC is mandatory for ~20 product categories. CIQ inspection is required for food, chemicals, cosmetics. NMPA registration mandatory for pharma | CN | `TERRITORY_FILINGS["CN"]` has CIQ stub | Backend: `PGA_TRIGGERS_CN` mapping SAMR/CCC→electronics/auto/toys, NMPA→pharma/cosmetics, MARA→agriculture, MIIT→telecom, MEE→chemicals | M | P1 |
| GL-032 | **EU CBAM (Carbon Border Adjustment Mechanism)** | Already referenced in Engine 6 signal SIG-002 but no operational implementation. CBAM mandatory from 2026 | CBAM requires carbon certificates for iron/steel (Ch.72-73), aluminum (Ch.76), cement, fertilizers, electricity, hydrogen. Non-compliance blocks import | EU | One CBAM signal in `e6_regulatory`; no operational enforcement | Backend: CBAM reporting module — HS code screening against CBAM-covered products, quarterly report tracking, certificate requirement flagging | M | P1 |

### P1 — Simulation & Corridor Expansion

| ID | Title | What's Missing | Why It Matters for Global Credibility | Jurisdictions | Backend Foundation | Changes Needed | Effort | Priority |
|---|---|---|---|---|---|---|---|---|
| GL-033 | **Import corridors to non-US destinations** | All corridors terminate at US. No EU-bound, India-bound, Brazil-bound, or China-bound corridors | Simulation cannot generate non-US import scenarios. Cannot demonstrate EU/India/Brazil clearance in action | ALL | Corridor definitions in `reference_data.py` | Backend: add corridors — CN→DE (Shanghai→Hamburg, 25-30d ocean), CN→IN (Shanghai→Nhava Sheva, 14-18d ocean), CN→BR (Shanghai→Santos, 30-40d ocean), US→CN, DE→IN, AR→BR (ground), CN→JP, CN→KR, US→SG | M | P1 |
| GL-034 | **Non-US port reference data** | Only `US_PORT_CODES` dict exists (16 US ports). No EU customs offices, India customs stations, Brazil URF codes, China customs districts | Port selection is US-only. An EU declaration needs customs office codes; India needs INNSA1/INMAA1; Brazil needs URF codes | ALL | `US_PORT_CODES` in `reference_data.py` | Backend: `EU_CUSTOMS_OFFICES`, `IN_PORT_CODES` (INNSA1, INMAA1, INDEL4, etc.), `BR_URF_CODES` (0817800 Santos, 0817600 Guarulhos, etc.), `CN_CUSTOMS_DISTRICTS` | M | P1 |
| GL-035 | **EU customs authority simulation actor** | No `EUCustomsAuthorityActor`. No AEO-differentiated clearance rates | Cannot simulate EU customs clearance. AEO-F holders get ~95% auto-release vs 40% for new operators | EU | `CBPAuthorityActor` exists as pattern; AEO differentiation logic specified in gap analysis | Backend: new `EUCustomsAuthorityActor` — risk profiling (AEO green lane), documentary checks, physical inspections, EU customs query, customs debt notification | L | P1 |
| GL-036 | **India ICEGATE authority simulation actor** | No `ICEGATEAuthorityActor`. No RMS channel routing (green 70%/yellow 20%/red 10%) | Cannot simulate India customs clearance. India's RMS channel system is fundamental to import processing | IN | `CBPAuthorityActor` as pattern | Backend: new `ICEGATEAuthorityActor` — RMS green/yellow/red channel assignment, First Check vs Second Check, assessment query (≈CF-28), speaking order (≈CF-29), out-of-charge order | L | P1 |
| GL-037 | **Brazil Receita Federal simulation actor** | No `ReceitaFederalActor`. No 4-channel parametrizacao (green/yellow/red/grey) | Cannot simulate Brazil customs clearance. Canal Cinza (grey channel) for fraud investigation is unique to Brazil | BR | `CBPAuthorityActor` as pattern | Backend: new `ReceitaFederalActor` — 4-channel parametrizacao, exigencia fiscal (≈CF-28), auto de infracao (≈CF-29), 30-day response deadlines | L | P1 |

### P1 — Agent Intelligence

| ID | Title | What's Missing | Why It Matters for Global Credibility | Jurisdictions | Backend Foundation | Changes Needed | Effort | Priority |
|---|---|---|---|---|---|---|---|---|
| GL-038 | **Jurisdiction-aware agent tool descriptions** | All 12 agent tools have US-specific descriptions. `calculate_tariff` mentions "Section 301, 232, AD/CVD, MPF, HMF". `check_compliance` references "UFLPA". `screen_entity` screens US lists only | Agent gives US-specific advice for EU-destined shipments. Tells an India importer about "MPF" instead of "SWS" | ALL | `tools.py` has 12 tools with US descriptions | Backend: make tool descriptions dynamically context-aware based on shipment destination; update agent system prompts to include multi-jurisdiction regulatory knowledge | M | P1 |
| GL-039 | **New EU/APAC-specific agent tools** | No `check_eori_validity`, `lookup_preference_rate`, `check_tariff_quota`, `check_cbam_obligation`, `check_reach_status`, `file_ens` | Agent cannot perform jurisdiction-specific operations. Can't validate EORI, check FTA preference availability, or verify CBAM obligations | EU, APAC | Tool framework in `tools.py` | Backend: 6-8 new tool implementations for EU/APAC-specific operations | L | P1 |

### P2 — Advanced Procedures, Export, and Tax Completeness

| ID | Title | What's Missing | Why It Matters for Global Credibility | Jurisdictions | Backend Foundation | Changes Needed | Effort | Priority |
|---|---|---|---|---|---|---|---|---|
| GL-040 | **EU customs procedures (warehousing, IP, OP, transit)** | Only free circulation (Procedure 40) implied. No customs warehousing (71), inward processing (51), outward processing (21), or NCTS transit | These procedures unlock significant business value. Warehousing defers duty; IP allows duty-free import for re-export manufacturing | EU | No procedure code concept exists | Backend: procedure code field on declaration; IP/OP duty calculation adjustments; transit declaration model; guarantee management | XL | P2 |
| GL-041 | **India export support (Shipping Bill, drawback, RoDTEP)** | No export filing workflow. No duty drawback calculation, no RoDTEP rate table, no LUT management | India's export incentive schemes (RoDTEP replacing MEIS, drawback) are essential for exporters. Platform is import-only | IN | No export workflow exists | Backend: Shipping Bill filing model and state machine; drawback calculation engine (All Industry Rates); RoDTEP rate table; LUT validity tracking | L | P2 |
| GL-042 | **Brazil export support (DU-E, drawback)** | No Brazil export declaration. No DU-E (Declaracao Unica de Exportacao), no export NF-e (CFOP 7.xxx) | Brazil is a major exporter (soybeans, iron ore, aircraft). Platform cannot handle Brazil export operations | BR | No export workflow exists | Backend: DU-E model; export NF-e linkage; drawback credits tracking (suspensao/isencao/restituicao) | L | P2 |
| GL-043 | **China cross-border e-commerce tax regime** | No 行邮税 (personal postal articles tax) for trade modes 9610/1210. Completely different rate structure (15%/25%/50%) replacing duty+VAT for parcels under RMB 5,000 | Cross-border e-commerce is one of China's fastest-growing trade segments. Different tax treatment from general trade | CN | `ChinaTaxRegime` handles general trade only | Backend: e-commerce trade mode detection; personal postal articles tax calculation; per-transaction (RMB 5,000) and annual (RMB 26,000) limits | M | P2 |
| GL-044 | **China processing trade duty exemption** | No support for 来料加工 (supplied-material processing — full duty exemption) or 进料加工 (purchased-material processing — duty deferral with bond). ~15% of China trade volume | Processing trade is a massive segment. Manufacturers importing raw materials for re-export get full duty exemption | CN | `ChinaTaxRegime` has no processing trade logic | Backend: processing trade manual system (手册) — track duty-free raw materials imported → finished goods exported; bond management | L | P2 |
| GL-045 | **EU excise duties** | No excise duty calculation for alcohol (HS 2203-2208), tobacco (HS 2402-2403), or energy products (HS 2701-2716) | Excise duties vary dramatically by member state and can be significant. Missing them = incorrect landed cost | EU | `EUTaxRegime` has MFN + VAT only | Backend: excise duty calculation engine for alcohol/tobacco/energy; per-member-state rates | M | P2 |
| GL-046 | **India Anti-Dumping Duty table** | India is one of the world's largest users of anti-dumping duties. No ADD table for India | Active India ADD cases on Chinese steel, telecom equipment, chemicals, textiles. Missing ADD can understate duties by 20-100%+ | IN | US `ADD_CVD_CASES` reference exists in simulation | Backend: `add_orders_in` table; India ADD/CVD lookup by ITC-HS code + origin country; DECOM (Directorate of Trade Remedies) data | M | P2 |
| GL-047 | **Brazil ICMS-ST and DIFAL** | No ICMS-ST (Substituicao Tributaria) — applies to specific product-state combinations. No ICMS-DIFAL (interstate differential) | ICMS-ST affects electronics, beverages, cosmetics, auto parts. Hundreds of product-state combinations with MVA multipliers. Very complex but very common | BR | `BrazilTaxRegime` has standard ICMS grossup | Backend: per-NCM MVA (Margem de Valor Agregado) lookup table; state-by-state ST applicability; DIFAL for interstate movement | XL | P2 |
| GL-048 | **Multi-currency tariff computation** | Engine computes in USD only. EU duties in EUR, India in INR, Brazil in BRL, China in RMB, Japan in JPY, Korea in KRW | Duty amounts displayed in wrong currency. Exchange rate fluctuations affect real landed cost | ALL | All tariff engines output USD | Backend: exchange rate integration; per-jurisdiction currency designation; multi-currency landed cost display | M | P2 |
| GL-049 | **Windsor Framework (Northern Ireland)** | No support for GB→NI movements requiring customs declarations, "Not At Risk" determination, or green/red lane routing | One of the most complex customs arrangements in the world. NI follows EU customs rules for goods while being in UK customs territory | UK | No NI-specific logic | Backend: Windsor Framework routing logic; "Not At Risk" determination; green lane (UK duty) vs red lane (EU duty) selection | L | P2 |

### P2 — Simulation Actors & Seasonal Events

| ID | Title | What's Missing | Why It Matters for Global Credibility | Jurisdictions | Backend Foundation | Changes Needed | Effort | Priority |
|---|---|---|---|---|---|---|---|---|
| GL-050 | **China GACC authority simulation actor** | No simulation of Chinese customs responses, CIQ inspection, or enterprise credit-based channel routing | Cannot demonstrate China clearance workflow. AA-rated AEO companies get <1% exam rate vs 50% for D-rated entities | CN | `CBPAuthorityActor` as pattern; enterprise credit system specified | Backend: new `GACCAuthorityActor` — enterprise credit-based channel routing (AA/A/B/C/D), CIQ inspection simulation, price verification, declaration correction workflow | L | P2 |
| GL-051 | **APAC seasonal events and disruption modeling** | Have CNY and Golden Week. Missing Chuseok (KR), Tet (VN), Songkran (TH), Ramadan (ID/MY), Japan fiscal year-end (March), GACC system maintenance | APAC customs operations affected by regional holidays. Clearance times spike during CNY (already modeled) but also during Tet, Chuseok, etc. | APAC | Seasonal events framework exists in `reference_data.py` | Backend: add APAC seasonal events; GACC system maintenance windows; monsoon disruptions for Indian ports (June-September) | S | P2 |

### P3 — Deep Jurisdiction Features & Future

| ID | Title | What's Missing | Why It Matters for Global Credibility | Jurisdictions | Backend Foundation | Changes Needed | Effort | Priority |
|---|---|---|---|---|---|---|---|---|
| GL-052 | **Singapore TradeNet integration model** | No Singapore-specific customs support despite world-class port (4-minute average clearance). Only port climate/congestion data | Singapore is a major transshipment hub AND destination. TradeNet is one of the world's most efficient customs systems | SG | Singapore port data exists; `TERRITORY_FILINGS["SG"]` has TradeNet stub | Backend: TradeNet declaration model (IN/OUT/TF permit types); 9% GST computation; Strategic Goods Control Act screening | L | P3 |
| GL-053 | **Vietnam VNACCS/VCIS customs model** | No Vietnam customs support despite being a top-5 US trade partner and CPTPP/EVFTA beneficiary | Vietnam is one of the fastest-growing trade partners. CPTPP and EVFTA create massive FTA opportunities | VN | VN corridor exists (VN→US); Ho Chi Minh port data exists | Backend: VNACCS declaration model; VN tax regime (duty + special consumption + 10% VAT); FTA benefit calculation (CPTPP, EVFTA, RCEP) | L | P3 |
| GL-054 | **EU Deforestation Regulation (EUDR)** | No due diligence requirements for commodities linked to deforestation (soy, beef, palm oil, wood, cocoa, coffee, rubber) | EUDR requires geolocation data for production plots. Effective date postponed but compliance preparation essential for importers of covered commodities | EU | No EUDR logic exists | Backend: EUDR commodity screening by HS code; due diligence statement tracking; geolocation data requirement flagging | M | P3 |
| GL-055 | **China Unreliable Entity List screening** | Entity screening only checks US restricted party lists. China's Unreliable Entity List (不可靠实体清单) not screened | Platform gives incomplete compliance picture for China-facing trade. Companies on China's UEL face trade restrictions | CN | `screen_entity` screens US lists | Backend: add China Unreliable Entity List, Export Control Law items catalog to screening engine | M | P3 |

---

## 5. Trade Agreement Coverage Plan

### Beyond USMCA: Global FTA Network

The platform currently supports only `FTA_PARTNERS_US`. The global expansion requires FTA coverage across 5 jurisdiction blocs:

| Agreement | Type | Partners | Platform Priority | Tariff Impact | Origin Proof |
|---|---|---|---|---|---|
| **RCEP** | Mega-regional | 15 countries (ASEAN-10 + CN, JP, KR, AU, NZ) | P0 | 30% of global GDP; phased reductions over 20 years | Form RCEP |
| **CPTPP** | Mega-regional | 11 countries (incl. UK since 2024) | P1 | Comprehensive tariff elimination; high-standard rules of origin | CPTPP certificate or self-certification |
| **EU-UK TCA** | Bilateral | EU-27 ↔ UK | P0 | Zero-duty on qualifying goods (complex RoO) | EUR.1 or self-certification |
| **EU-Japan EPA** | Bilateral | EU ↔ Japan | P1 | Zero tariff on most industrial goods | Origin statement on invoice (REX) |
| **EU-Korea FTA** | Bilateral | EU ↔ South Korea | P1 | Zero duty on industrial goods | EUR.1 or invoice declaration |
| **EU-Canada CETA** | Bilateral | EU ↔ Canada | P1 | Tariff elimination | EUR.1 or origin statement |
| **India-UAE CEPA** | Bilateral | India ↔ UAE | P0 | Zero/reduced BCD for ~90% of tariff lines | Specific CEPA form |
| **India-Australia ECTA** | Bilateral | India ↔ Australia | P0 | Zero BCD for ~85% of tariff lines | Specific ECTA form |
| **India-ASEAN CECA** | Regional | India + 10 ASEAN | P0 | Reduced BCD for ~80% of tariff lines | Form AI |
| **India-Japan CEPA** | Bilateral | India ↔ Japan | P1 | Reduced/zero BCD on industrial goods | Form AIJCEPA |
| **India-Korea CEPA** | Bilateral | India ↔ South Korea | P1 | Reduced BCD with special safeguards | Form AIKFTA |
| **India-EFTA TEPA** | Regional | India + CH, IS, LI, NO | P1 | Reduced BCD + investment provisions | TEPA certificate |
| **ACFTA** | Regional | China + 10 ASEAN | P1 | Extensive tariff elimination | Form E |
| **China-Korea FTA** | Bilateral | China ↔ South Korea | P1 | Extensive tariff elimination schedule | Form FTA |
| **ChAFTA** | Bilateral | China ↔ Australia | P1 | Wine, dairy, beef, resources | ChAFTA certificate |
| **Mercosur TEC** | Regional | Brazil, Argentina, Paraguay, Uruguay | P1 | **0% II** within bloc | Certificado de Origem Mercosul |
| **ACE-35** | Bilateral | Brazil ↔ Chile | P2 | 0-100% reduction by NCM | ACE-35 certificate |
| **ACE-72** | Regional | Mercosur ↔ Colombia, Peru | P2 | Partial preference by NCM | ACE-72 certificate |
| **SAFTA** | Regional | 8 South Asian countries | P2 | Preferential rates with sensitive lists | SAFTA certificate |
| **Mercosur-EU** | Inter-regional | Mercosur ↔ EU | P3 | Pending ratification — timeline uncertain | TBD |
| **KORUS FTA** | Bilateral | Korea ↔ US | Already partial | Needs KR-as-destination rates | Already in FTA_PARTNERS_US |
| **ECFA** (cross-strait) | Bilateral | China ↔ Taiwan | P3 | Political sensitivity | ECFA certificate |

### Implementation Approach

1. **Phase 1**: `FTA_PARTNERS_EU`, `FTA_PARTNERS_UK`, `FTA_PARTNERS_IN`, `FTA_PARTNERS_CN`, `FTA_PARTNERS_BR` — simple country-to-agreement mapping dicts (like existing `FTA_PARTNERS_US`)
2. **Phase 2**: Preferential rate schedule tables — per-FTA, per-HS-code reduced duty rates with phase-in years
3. **Phase 3**: Rules of origin engine — value content rules, change of tariff classification, product-specific rules per agreement
4. **Phase 4**: Cumulation support — RCEP allows content from all 15 countries; PEM zone allows diagonal cumulation across 25+ countries

---

## 6. Simulation Globalization Plan

### Extending 19 Actors for Multi-Jurisdiction

The simulation currently generates US-only scenarios. Globalization requires:

#### Actor Adaptation Matrix

| Current US Actor | EU Adaptation | India Adaptation | Brazil Adaptation | China Adaptation |
|---|---|---|---|---|
| `CBPAuthorityActor` | New `EUCustomsAuthorityActor` (AEO-differentiated, risk lanes) | New `ICEGATEAuthorityActor` (RMS green/yellow/red) | New `ReceitaFederalActor` (4-channel parametrizacao) | New `GACCAuthorityActor` (enterprise credit A-D) |
| `PGAActor` (FDA/EPA/CPSC/NHTSA) | New `EURegulatoryActor` (CE/REACH/CBAM/ICS2) | New `IndiaRegulatoryActor` (BIS/FSSAI/DGFT/WPC) | New `OrgaosAnuentesActor` (ANVISA/MAPA/INMETRO/ANATEL) | New `ChinaRegulatoryActor` (CCC/CIQ/NMPA) |
| `ISFActor` (ocean ISF 10+2) | New `ICS2ENSActor` (all modes, PLACI, DNL) | N/A (India uses AMS/ACS) | N/A (no Brazil equivalent) | Pre-arrival manifest actor |
| `CustomsActor` (STP vs manual) | EU risk assessment (AEO green lane) | India RMS (70% green / 20% yellow / 10% red) | Brazil parametrizacao (55% green / 25% yellow / 15% red / 5% grey) | China enterprise credit channel |
| `BrokerSimActor` | EU customs representative (direct/indirect representation) | India CHA (Customs House Agent, now Customs Broker under CBLR 2018) | Brazil despachante aduaneiro (requires procuracao per importer) | China customs broker (报关企业) |
| `FinancialActor` (MPF/HMF/bond) | EU guarantee management, duty deferment | India bank guarantee, IGST ITC claim | Brazil AFRMM, Taxa Siscomex, RADAR ceiling | China duty payment via Single Window |
| `DocumentsActor` (US doc types) | EU doc types (EUR.1, DV1, ENS) | India doc types (Bill of Entry, BIS cert, eSanchit) | Brazil doc types (NF-e, LI/LPCO, DI) | China doc types (报关单, CCC, CIQ application) |

#### Multi-Jurisdiction Scenario Generator

```
Default scenario distribution:
  US-bound:     40% (current — reduces from 100%)
  EU-bound:     25% (DE, NL, FR, IT primary)
  India-bound:  10% (MUM, MAA, DEL primary)
  Brazil-bound: 10% (Santos, Guarulhos primary)
  China-bound:  10% (Shanghai, Shenzhen primary)
  Other APAC:    5% (JP, KR, SG)

Per-jurisdiction clearance time benchmarks:
  US:     ~2.5 days average
  EU:     ~2.0 days average (AEO significantly faster)
  India:  ~6.0 days average (green channel: 1 day; red channel: 12 days)
  Brazil: ~8.0 days average (green channel: instant; grey channel: 30+ days)
  China:  ~3.0 days average (AA enterprise: <1 day)
  Japan:  ~1.5 days average (world-class efficiency)
  Korea:  ~1.5 hours average (UNI-PASS, AEO importers)
```

---

## 7. Implementation Roadmap

### Sprint 1: "Jurisdiction Foundation" (3 weeks)

**Theme**: Build the abstraction layer that makes the entire platform jurisdiction-aware. Zero jurisdiction-specific features — just the framework.

| ID | Item | Effort |
|---|---|---|
| GL-001 | Jurisdiction detection from destination | S |
| GL-002 | Polymorphic declaration model | L |
| GL-007 | Jurisdiction-aware checklist framework | L |
| GL-008 | Frontend jurisdiction-aware rendering (label extraction, config registry) | L |
| GL-009 | Generalized authority response screen | L |
| GL-010 | Jurisdiction-specific state machine framework | L |

**Outcome**: Platform can distinguish jurisdictions. US behavior unchanged. Labels parameterized. Framework ready for jurisdiction-specific implementations.

### Sprint 2: "EU First-Class" (3 weeks)

**Theme**: EU is the highest-value non-US jurisdiction. UK separation, EORI, declaration workflow, ICS2.

| ID | Item | Effort |
|---|---|---|
| GL-003 | EORI identifier support | M |
| GL-006 | UK tariff engine separation | L |
| GL-011 | EU customs declaration workflow (H1-H7) | XL |
| GL-016 | EU FTA partners and preference rates | L |
| GL-021 | EU VAT mechanisms (PVA, Procedure 42) | M |
| GL-026 | ICS2 ENS pre-arrival filing | L |
| GL-027 | EU sanctions screening integration | M |
| GL-029 | CE/UKCA and REACH compliance triggers | M |
| GL-033 | EU-bound corridors (partial) | S |
| GL-034 | EU customs office codes (partial) | S |

**Outcome**: EU declarations can be filed and processed. EORI validated. EU-specific checklist. ICS2 ENS tracking. EU FTA preferences available.

### Sprint 3: "India & Brazil First-Class" (3 weeks)

**Theme**: India (2nd largest population, fast-growing trade) and Brazil (largest Latin American economy, complex tax system).

| ID | Item | Effort |
|---|---|---|
| GL-004 | IEC/GSTIN/AD Code for India | M |
| GL-005 | CNPJ/RADAR for Brazil | M |
| GL-013 | India Bill of Entry workflow | XL |
| GL-014 | Brazil DI/DUIMP workflow | XL |
| GL-017 | India FTA partners | L |
| GL-019 | India BCD database lookup + reconciliation | M |
| GL-020 | India IGST slab expansion | M |
| GL-022 | Brazil AFRMM and Taxa Siscomex | M |
| GL-028 | India BIS/FSSAI/DGFT triggers | M |
| GL-030 | Brazil ANVISA/MAPA/INMETRO triggers | L |
| GL-033 | India + Brazil corridors (partial) | S |
| GL-034 | India + Brazil port codes (partial) | S |

**Outcome**: India and Brazil declarations can be filed. Correct tax calculations with proper slabs. Regulatory agencies trigger appropriate holds.

### Sprint 4: "China & APAC + Simulation" (3 weeks)

**Theme**: China first-class support. Japan and Korea tariff regimes. Multi-jurisdiction simulation.

| ID | Item | Effort |
|---|---|---|
| GL-015 | China GACC declaration workflow | XL |
| GL-018 | China RCEP + FTA preferences | L |
| GL-023 | China provisional/interim rates | M |
| GL-024 | Japan tariff regime | L |
| GL-025 | South Korea tariff regime | L |
| GL-031 | China CCC/CIQ/NMPA triggers | M |
| GL-035 | EU customs authority simulation actor | L |
| GL-036 | India ICEGATE simulation actor | L |
| GL-037 | Brazil Receita Federal simulation actor | L |
| GL-038 | Jurisdiction-aware agent tools | M |
| GL-039 | New EU/APAC agent tools | L |
| GL-050 | China GACC simulation actor | L |

**Outcome**: China declarations functional. Japan and Korea tariff calculations work. Simulation generates multi-jurisdiction scenarios. Agent adapts advice by jurisdiction.

### Sprint 5: "Advanced Features & Parity" (2 weeks)

**Theme**: Fill remaining gaps, polish, ensure no jurisdiction is treated as afterthought.

| ID | Item | Effort |
|---|---|---|
| GL-012 | UK CDS declaration workflow | XL |
| GL-032 | EU CBAM reporting | M |
| GL-040 | EU customs procedures (warehousing, IP, transit) | XL |
| GL-041 | India export (Shipping Bill, drawback, RoDTEP) | L |
| GL-042 | Brazil export (DU-E, drawback) | L |
| GL-043 | China cross-border e-commerce tax | M |
| GL-045 | EU excise duties | M |
| GL-046 | India Anti-Dumping Duty table | M |
| GL-048 | Multi-currency tariff computation | M |
| GL-049 | Windsor Framework (NI) | L |
| GL-051 | APAC seasonal events | S |

**Outcome**: All jurisdictions at production quality. UK fully separated from EU. Advanced customs procedures available. Multi-currency support. Export workflows for India and Brazil.

---

## 8. Integration with Existing Backlog

### How These 55 Items Interleave with the 72 US-Focused Items

The existing `ux-analysis-backlog.md` contains 72 items (BL-001 through BL-072) focused on US operational quality — fixing placeholders, surfacing backend intelligence, adding search/sort/filter, and simulation realism. The global backlog (GL-001 through GL-055) is **complementary, not competing**.

#### Dependency Map

| Global Item | Depends On (Existing Backlog) | Why |
|---|---|---|
| GL-008 (Frontend jurisdiction rendering) | BL-008 (Compliance Dashboard real data) | Both require moving away from hardcoded US content |
| GL-009 (Generalized authority response) | BL-019 (CF-29 AI draft) | Generalize the response screen while adding CF-29 support |
| GL-007 (Jurisdiction-aware checklist) | BL-009 (Document Generate on Broker) | Generalize checklist while adding Generate capability |
| GL-038 (Jurisdiction-aware agent tools) | BL-004 (verify_classification button), BL-005 (calculate_entry_fees button) | Surface tools directly first, then make them jurisdiction-aware |
| GL-033 (Non-US corridors) | BL-007 (Regulatory Intel from API) | Corridors need regulatory signals for non-US destinations |

#### Recommended Interleaving

```
Month 1:  Existing BL Sprint 1 ("Kill the Placeholders") + GL Sprint 1 ("Jurisdiction Foundation")
          → Fix US hardcoded values WHILE building jurisdiction abstraction layer
          → Result: US works correctly + framework ready for global

Month 2:  Existing BL Sprint 2 ("Information → Action") + GL Sprint 2 ("EU First-Class")
          → Surface US tools as buttons WHILE building EU declaration workflow
          → Result: US tools accessible + EU operational

Month 3:  Existing BL Sprint 3 ("Operational Depth") + GL Sprint 3 ("India & Brazil")
          → Add search/filter/sort WHILE building India + Brazil filing
          → Result: Platform scalable for 100+ entries across 4 jurisdictions

Month 4:  Existing BL Sprint 4 ("Domain Credibility") + GL Sprint 4 ("China & APAC + Simulation")
          → Add ISF/ADD/CVD/liquidation tracking WHILE building China/APAC + multi-jurisdiction simulation
          → Result: Domain expert-credible across all jurisdictions

Month 5:  Existing BL Sprint 5 ("Intelligence Layer") + GL Sprint 5 ("Advanced Features & Parity")
          → Simulation realism + IOR architecture WHILE filling jurisdiction gaps
          → Result: Complete platform — every jurisdiction first-class, every tool surfaced
```

#### Items That Become Easier When Done Together

| Existing BL Item | Related GL Item | Synergy |
|---|---|---|
| BL-003 (Resolution steps → actions) | GL-010 (Jurisdiction state machines) | Resolution steps are jurisdiction-specific — build both together |
| BL-007 (Regulatory Intel from API) | GL-032 (EU CBAM) | CBAM is a regulatory signal — surface alongside the API CRUD |
| BL-010 (ISF workflow) | GL-026 (ICS2 ENS) | ISF and ICS2 are parallel concepts — build generic pre-arrival filing framework |
| BL-011 (ADD/CVD data) | GL-046 (India ADD table) | ADD/CVD framework serves both US and India — build once |
| BL-016 (Simulation control panel) | GL-035/36/37/50 (Jurisdiction actors) | Control panel should support jurisdiction actor selection from day one |
| BL-026 (IOR architecture) | GL-003/04/05 (EORI/IEC/CNPJ) | IOR is the jurisdiction-generic concept; EORI/IEC/CNPJ are jurisdiction-specific implementations |
| BL-027 (Country code picker) | GL-001 (Jurisdiction detection) | Country picker drives jurisdiction detection — build together |
| BL-030 (CF-7501 fields) | GL-002 (Polymorphic declaration model) | Improving US declaration fields while making the model polymorphic |
| BL-054 (Entry number format) | GL-002 (Polymorphic declaration model) | MRN, BE number, DI number alongside fixed US entry number format |
| BL-067 (PGA per-agency) | GL-028/29/30/31 (Non-US regulatory triggers) | Expand PGA to per-agency for US while adding non-US agencies |

---

## Appendix A: Effort Legend

| Size | Definition | Approximate Scope |
|---|---|---|
| **S** (Small) | Single file, straightforward change | 1-2 days, <100 lines changed |
| **M** (Medium) | 2-4 files, moderate complexity | 3-5 days, 100-500 lines changed |
| **L** (Large) | 5-10 files, significant new logic | 1-2 weeks, 500-1500 lines changed |
| **XL** (Extra Large) | 10+ files, new subsystem | 2-3 weeks, 1500+ lines changed |

## Appendix B: Source Analyst Documents

| Document | Scope | Key Contributions to This Backlog |
|---|---|---|
| `global-gap-eu-uk.md` | EU-27 + UK post-Brexit | GL-003, GL-006, GL-011, GL-012, GL-016, GL-021, GL-026, GL-027, GL-029, GL-032, GL-035, GL-040, GL-045, GL-049, GL-054 |
| `global-gap-india.md` | India (CBIC/ICEGATE) | GL-004, GL-013, GL-017, GL-019, GL-020, GL-028, GL-036, GL-041, GL-046 |
| `global-gap-brazil-latam.md` | Brazil + Mercosur + Mexico/Argentina coverage | GL-005, GL-014, GL-022, GL-030, GL-037, GL-042, GL-047 |
| `global-gap-china-apac.md` | China + Japan + Korea + ASEAN | GL-015, GL-018, GL-023, GL-024, GL-025, GL-031, GL-039, GL-043, GL-044, GL-050, GL-051, GL-052, GL-053, GL-055 |
| `global-gap-unified-ux.md` | Cross-cutting UX architecture | GL-001, GL-002, GL-007, GL-008, GL-009, GL-010, GL-034, GL-038, GL-048 |
| `ux-analysis-backlog.md` | Existing 72-item US-focused backlog | Integration map in Section 8 |
