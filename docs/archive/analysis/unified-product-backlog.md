# Clearance Vibe — Unified Product Backlog

**Synthesized from**: `ux-analysis-backlog.md` (72 US-focused items, BL-001–BL-072) + `global-customs-backlog.md` (55 global items, GL-001–GL-055)
**Date**: 2026-02-07
**Unified backlog items**: 104 (UB-001 through UB-104)

---

## 1. Harmonization Summary

### Merge Statistics

| Metric | Count |
|---|---|
| **Input items** | 127 (72 BL + 55 GL) |
| **Direct duplicates merged** | 8 pairs → 8 items (16 inputs became 8) |
| **BL items generalized** to be jurisdiction-aware | 19 |
| **Conflicts found and resolved** | 2 |
| **GL items absorbed into generalized BL items** | 7 |
| **Net unified items** | 104 |

### Duplicates Found and Merged

| Unified ID | BL Source | GL Source | What Was Duplicated |
|---|---|---|---|
| UB-003 | BL-027 (Country code picker) | GL-001 (Jurisdiction detection) | Both require ISO country picker + jurisdiction resolver — country picker drives jurisdiction detection |
| UB-006 | BL-026 (IOR architecture) | GL-003/004/005 (EORI/IEC/CNPJ) | IOR is the generic concept; EORI, IEC, CNPJ are jurisdiction-specific implementations of "importer identity" |
| UB-009 | BL-009 (Doc Generate on Broker) | GL-007 (Jurisdiction-aware checklist) | Both address the broker checklist — merged into "jurisdiction-aware checklist with Generate/Upload/Request capabilities" |
| UB-011 | BL-054 (Entry number format) + BL-030 (CF-7501 fields) | GL-002 (Polymorphic declaration model) | Entry number format fix and CF-7501 field expansion belong inside the polymorphic declaration model work |
| UB-013 | BL-019 (CF-29 AI draft) | GL-009 (Generalized authority response) | CF-29 protest draft is a US instance of the generic "respond to authority inquiry" workflow |
| UB-015 | BL-010 (ISF workflow) | GL-026 (ICS2 ENS) | ISF and ICS2 are the US and EU versions of "pre-arrival security filing" — build one framework |
| UB-017 | BL-011 (ADD/CVD data) | GL-046 (India ADD table) | Anti-dumping/countervailing duty is a global concept — US and India both need it; build one framework |
| UB-037 | BL-067 (PGA per-agency) | GL-028/029/030/031 (Non-US regulatory triggers) | PGA expansion to per-agency for US is a subset of building jurisdiction-aware regulatory agency triggers |

### Conflicts Found and Resolved

| # | Conflict | Resolution |
|---|---|---|
| 1 | **Sprint timing**: BL backlog planned 5 sprints x 2 weeks; GL backlog planned 5 sprints x 3 weeks. Different cadences. | Unified to 6 sprints x 2-3 weeks. Foundation sprint is 3 weeks; action sprints are 2 weeks each; jurisdiction sprints are 3 weeks. |
| 2 | **BL-003/BL-012 (resolution steps → action buttons) vs GL-010 (jurisdiction state machines)**: BL items want to wire US-specific tool calls into resolution buttons; GL-010 requires resolution steps to be jurisdiction-dispatched. Building US-only resolution actions first means refactoring when jurisdictions arrive. | Resolution: Build resolution action framework with jurisdiction dispatch from the start (UB-019). Resolution steps call jurisdiction-aware tool endpoints. US is the first implementation but the wiring is generic. |

### Generalization Decisions

19 BL items were rewritten to be jurisdiction-aware rather than US-only. See Section 4 (Generalization Guide) for full details on each.

---

## 2. Unified Prioritized Backlog

### P0 — Foundation + Kill Placeholders (32 items)

These items either establish the jurisdiction framework that everything else depends on, or replace hardcoded/fake content that undermines credibility for ALL jurisdictions.

---

| ID | Original IDs | Title | What's Wrong / Missing | Why It Matters | Jurisdictions | Backend Available | Changes Needed | Effort | Priority |
|---|---|---|---|---|---|---|---|---|---|
| **UB-001** | GL-001, BL-027 | **Jurisdiction detection + ISO country picker** | No mechanism to determine customs jurisdiction from destination country. Country code field is free-text, accepts invalid codes ("CHN" instead of "CN"). | Without jurisdiction detection, the platform cannot adapt to non-US trade lanes. Invalid country codes silently break routing. This is the single atomic unit that unlocks everything. | ALL | `REGIME_MAP` dispatch exists in tariff engine; `EU_MEMBERS` frozenset exists; country lists in backend | Backend: jurisdiction resolver function mapping country→jurisdiction config; Frontend: searchable ISO-2 country picker with flags, `useJurisdiction()` hook, used globally across AddProduct, CreateOrder, ShipperProductDetail, buyer checkout | M | P0 |
| **UB-002** | GL-002, BL-054, BL-030 | **Polymorphic declaration model + entry number/field fixes** | `EntryFiling` hardcoded to US: entry_type "01", cbp_response JSONB, US port codes. Entry number format is random 3-7-1 digits (should be Filer Code + 7 digits + check digit). 12 of 25 CF-7501 fields missing or fabricated. | A broker cannot file an EU declaration, Indian Bill of Entry, or Brazilian DI. The US-specific model cannot represent MRN, BE number, or DI number. Fabricated entry numbers and missing fields signal "this is fake" to any domain expert. | ALL | `EntryFiling` model in `operational.py`; check digit algorithm is public; data model can accommodate via JSONB | Backend: add `jurisdiction`, `jurisdiction_config` JSONB, generalize `cbp_response` → `authority_response`; fix US entry number format (filer code + check digit); add missing CF-7501 fields (MID, entered value, SPI, export date, gross weight); migrate existing data as `jurisdiction="US"` | L | P0 |
| **UB-003** | GL-010 | **Jurisdiction-specific state machines** | `state_machine.py` has US-only transitions: `entry_filed → cf28_pending/cf29_pending/exam_scheduled`. EU, India, Brazil, China lifecycles cannot be represented. | EU declaration lifecycle, India RMS channels, Brazil parametrizacao channels are fundamentally different from US. One mega-machine cannot represent all. | ALL | State machine in `state_machine.py` | Backend: jurisdiction-specific state machine files or jurisdiction-context dispatch; US machine unchanged; framework for EU/IN/BR/CN machines | L | P0 |
| **UB-004** | GL-008 | **Frontend jurisdiction-aware rendering** | 111+ references to CBP-specific concepts across 7 broker surface files. "Submit to CBP", "CF-28", "CF-29" hardcoded everywhere. | The broker UI is unusable for non-US trade. An EU broker sees "Submit to CBP" for a German-bound shipment. | ALL | Frontend components exist but with US labels | Frontend: extract hardcoded labels into `JurisdictionConfig` registry; parameterize `StatusBadge`, `BrokerNav`, `EntryDetail`, `BrokerDashboard`; `useJurisdiction(destination)` hook drives all labels | L | P0 |
| **UB-005** | GL-009, BL-019 | **Generalized authority response screen + CF-29 protest draft** | `CBPResponses.tsx` is entirely US CBP-specific. CF-29 protest AI Draft button disabled (`!isCF29`). No EU customs query, India assessment query, or Brazil exigencia fiscal UI. | Brokers cannot respond to non-US customs authority inquiries. CF-29 protests require careful legal drafting but have no AI assistance. | ALL (US improved, EU/IN/BR/CN enabled) | CBP response handling in `cbp_authority.py`; `draft_cf28_response` tool can be extended for CF-29 | Frontend: abstract `CBPResponses.tsx` to `AuthorityInquiries` screen with jurisdiction-dispatched response modals; enable AI Draft for CF-29 with protest-specific template; Backend: generic inquiry/response endpoints, CF-29 draft capability | L | P0 |
| **UB-006** | GL-003, GL-004, GL-005, BL-026 | **Importer identity framework (IOR + EORI + IEC + CNPJ)** | No Importer of Record architecture. No customs bond, Power of Attorney, or CBP Form 5106 for US. No EORI (EU), IEC/GSTIN (India), or CNPJ/RADAR (Brazil) fields anywhere. | Without IOR, you cannot file a customs entry in ANY jurisdiction. EORI is mandatory for all EU trade. IEC is mandatory for all Indian trade. CNPJ/RADAR is mandatory for all Brazil trade. This is the universal "who is importing" identity. | ALL | Entity model needed; no jurisdiction-specific identifiers exist | Backend: `ImporterIdentity` model with jurisdiction-specific credential types: US (EIN, bond number, POA, Form 5106), EU (EORI with country prefix + 15 chars), India (IEC 10-digit, GSTIN 15-digit, AD Code), Brazil (CNPJ 14-digit with modulo-11, RADAR number + modality); validation per type; Frontend: IOR management UI, credential selector on orders/entries | XL | P0 |
| **UB-007** | BL-001 | **Buyer duty calculation — replace 5% hardcode with real tariff engine** | `ProductPage.tsx:51`: `const taxes = Math.round(product.declaredValue * 0.05)` — flat 5% regardless of product, HS code, origin, or tariff program. Real rates range 0-400%+. | Completely undermines the "duties included" value prop. EV batteries should show ~38.4%, not 5%. $4,175 understatement on the battery product alone. Uses jurisdiction-aware tariff engine so non-US buyers see correct duties too. | ALL (uses destination jurisdiction) | `/api/analyze/stream`, `e2_tariff.calculate_tariff()` with multi-jurisdiction regimes | Frontend: replace hardcoded calc with API call using buyer's destination country; add loading state; show itemized breakdown (base duty + jurisdiction-specific surcharges + fees) | M | P0 |
| **UB-008** | BL-002 | **Buyer product catalog from API, not static file** | `StoreFront.tsx:8` imports from `data/products.ts` — 5 static products with pre-computed prices. No API call. Cannot scale. `totalDuty` values in static data are frozen. | Buyer surface cannot scale past 5 demo products. Buyer surface disconnected from real engine. | ALL | Product catalog API exists; tariff engine exists | Backend: product listing endpoint for buyer with dynamic pricing; Frontend: API integration replacing static imports | L | P0 |
| **UB-009** | BL-009, GL-007 | **Jurisdiction-aware document checklist with Generate/Upload/Request** | `CHECKLIST_LABELS` and `_compute_checklist_state()` hardcoded to 8 US items. Broker surface has Upload-only (no Generate). `DocumentRequirements.tsx` on Shipper proves Generate + Upload works. | The broker checklist — the core workflow — shows US documents for EU/India/Brazil entries. EORI verification, BIS certificate, NF-e are invisible. Missing "Generate" capability means brokers manually create documents the system could auto-generate. | ALL | Checklist computation in `broker.py`; `DOCUMENT_FIELD_SCHEMAS` in `documents.py`; document generation pipeline works on Shipper | Backend: refactor `_compute_checklist_state()` to load items from jurisdiction config (US: bond/PGA/origin cert; EU: EORI/CE/REACH/EUR.1; India: IEC/BIS/FSSAI; Brazil: RADAR/NF-e/LI); Frontend: port `DocumentRequirements.tsx` pattern to Broker (Generate + Upload + Request from Shipper) | L | P0 |
| **UB-010** | BL-003, BL-018 | **Resolution steps become action buttons (jurisdiction-aware)** | `BrokerDashboard.tsx:474-482`, `EntryDetail.tsx:456-475` — Resolution steps render as `<li>` elements with no click handlers. "Verify entity identity against BIS/OFAC lists" is passive text. Same issue in risk flag cards. | Brokers cannot act on AI recommendations. Each step maps to an existing tool (`screen_entity`, `draft_communication`, `resolve_shipment_hold`). Resolution actions must be jurisdiction-dispatched (e.g., "Screen entity" checks OFAC for US, EU Consolidated List for EU). | ALL (US immediate, others via framework) | `screen_entity`, `draft_communication`, `resolve_shipment_hold`, `request_shipper_documents` — all in `tools.py` | Frontend: convert `<li>` to action buttons with tool invocation; action dispatch uses jurisdiction context; risk flag cards in EntryDetail get same treatment | M | P0 |
| **UB-011** | BL-004 | **`verify_classification` inline panel** | `EntryDetail.tsx:1016-1024` — "Improve classification confidence" opens chat panel instead of calling the tool directly. Tool returns current vs. suggested HS code, GRI analysis, and confidence. | Most critical broker tool hidden behind chat. This is the single most frequent broker action — verifying an HS code is correct before filing. Panel must understand jurisdiction-specific tariff code formats (HTS for US, CN/TARIC for EU, ITC-HS for India, NCM for Brazil). | ALL (tariff code format varies) | `verify_classification` tool in `tools.py:492-509` | Frontend: new inline panel component — current code vs. suggested, match indicator, confidence, GRI reasoning, "Reclassify" action if mismatch; display format adapts to jurisdiction (HTS-10 for US, CN-8+TARIC-2 for EU, ITC-HS-8 for India, NCM-8 for Brazil) | M | P0 |
| **UB-012** | BL-005, BL-034 | **`calculate_entry_fees` button + panel (jurisdiction-aware)** | `EntryDetail.tsx:1209-1239` — Falls back to `entry.declaredValue * 0.05`. The real tool calls `e2_tariff.calculate_tariff()` with Section 301, 232, AD/CVD for US. Sidebar insights use same crude fallback. | Inaccurate financial data for decision-making. Duty rate calculation is placeholder. Must show jurisdiction-appropriate breakdown: US (MFN + 301 + IEEPA + AD/CVD + MPF + HMF), EU (MFN + anti-dumping + VAT), India (BCD + SWS + IGST), Brazil (II + IPI + PIS + COFINS + ICMS). | ALL (breakdown varies) | `calculate_entry_fees` tool, `/entries/{id}/summary` endpoint, multi-jurisdiction tariff regimes | Frontend: "Calculate Actual Fees" button; show full jurisdiction-specific breakdown; also add "Generate Entry Summary" button; replace 5% fallback in sidebar; call on load | M | P0 |
| **UB-013** | BL-006 | **Fix `draft_communication` placeholder** | `broker.py:2543` — Fallback for unrecognized purposes: `"Dear {company},\n\n[Message body]\n\nBest regards,\n[Broker Name]"`. | Broken UX for catch-all communication purpose. The system has full shipment context (product, tracking number, missing docs) but outputs a template with a placeholder. | ALL | Shipment context available in endpoint | Backend: LLM-powered fallback generation using shipment context; ensure jurisdiction-appropriate terminology in generated text | S | P0 |
| **UB-014** | BL-007, BL-069 | **Regulatory intelligence from API + admin CRUD** | `RegulatoryIntel.tsx:16-89` — `const DEMO_SIGNALS` with 6 static entries. No API call. No CRUD. Dates from 2025. | #1 credibility problem for the entire platform. Static regulatory data doesn't track current policy changes. Must support signals for all jurisdictions (EU CBAM, India BIS updates, Brazil ICMS changes, China retaliatory tariffs). | ALL | `get_regulatory_signals` tool exists; needs persistence layer | Backend: signal CRUD API with jurisdiction field; Frontend: API integration + admin UI (create, edit, archive signals); signals filterable by jurisdiction | L | P0 |
| **UB-015** | BL-008 | **Compliance Dashboard — wire to real data** | `ComplianceDashboard.tsx:12-81` — `generateDemoEntries()` creates 47 random entries client-side. "Real-time compliance monitoring" label on fake data. | Most critical domain accuracy issue. Real simulation data exists in DB. | ALL | Dashboard aggregator API exists; simulation data in DB | Frontend: replace `generateDemoEntries()` with API call; add jurisdiction filter | M | P0 |
| **UB-016** | BL-055 | **Separate held vs. rejected status** | `broker.py:523` maps `held` to `status_counts.get("rejected", 0)`. These are fundamentally different states in every jurisdiction. | Held means customs is examining/detaining; Rejected means the entry was denied. Conflating them corrupts metrics and misleads brokers. | ALL | Status field in data model | Backend: separate held and rejected states; add specific status for examination vs. denial per jurisdiction | S | P0 |
| **UB-017** | BL-028 | **Global error handling + toast system** | `ShipperCatalog.tsx:21` `.catch(() => {})`, `AddProduct.tsx:88-90` catch does nothing, `CreateOrder.tsx:83-85` silent, `ShipperProductDetail.tsx:101-104` sets flag but no message. | User sees blank/frozen UI on failure with no recovery path. Single implementation fixes 5+ screens. | ALL | N/A | Frontend: global toast/notification system; network errors show retry; form errors show specific message | S | P0 |

### P0 Simulation Items (included because they affect data integrity across all jurisdictions)

| ID | Original IDs | Title | What's Wrong / Missing | Why It Matters | Jurisdictions | Backend Available | Changes Needed | Effort | Priority |
|---|---|---|---|---|---|---|---|---|---|
| **UB-018** | S-1 (BL backlog) | **Replace corridor flat-rate duties with HS-code-aware calculation** | `shipper.py:198`: All CN→US products get 38.4% regardless of HS code. | All financial aggregates across the entire platform become unrealistic. Must use jurisdiction-aware tariff engine. | ALL | Tariff engine with multi-jurisdiction regimes | Backend: chapter-level rate table at minimum; call tariff engine per jurisdiction | L | P0 |
| **UB-019** | S-2 (BL backlog) | **Make CustomsActor read the `misclassified` flag** | ShipperActor injects `misclassified` flag but CustomsActor ignores it. Same STP probability for clean and bad shipments. | Error injection becomes meaningless — currently defeated by random dice rolls. | US (extensible) | CustomsActor in simulation | Backend: reduce STP probability for misclassified shipments (30% instead of 87%) | S | P0 |
| **UB-020** | S-3 (BL backlog) | **Differentiate resolution timelines by hold type** | All holds resolve in 1-5 days at 75% success regardless of type. | Operational reality not reflected. UFLPA should be 30-180 days / 15% release. CF-28 should be 5-30 days / 80%. | US (extensible per jurisdiction) | ResolutionActor in simulation | Backend: per-hold-type resolution timelines; framework extensible for non-US hold types | M | P0 |

---

### P1 — Jurisdiction Parity + Unsurfaced Backend Capabilities (36 items)

These items either bring specific jurisdictions to functional parity or surface backend intelligence that exists but is hidden behind chat.

---

| ID | Original IDs | Title | What's Wrong / Missing | Why It Matters | Jurisdictions | Backend Available | Changes Needed | Effort | Priority |
|---|---|---|---|---|---|---|---|---|---|
| **UB-021** | GL-006 | **UK tariff engine separation from EU** | UK routes through `EUTaxRegime` or falls through as unsupported. UK tariff rates diverged from EU CET since Jan 2021. | UK duty calculations are wrong. Post-Brexit UK Global Tariff has different rates from EU CET. `GB` actually falls through as unsupported. | UK | `EUTaxRegime` exists; UK NOT in `EU_MEMBERS` frozenset | Backend: new `UKTaxRegime` in `regimes/uk.py`; dispatch `GB` to UK regime in `TariffEngine._get_regime()` | L | P1 |
| **UB-022** | GL-011 | **EU customs declaration workflow (H1-H7)** | No EU declaration types, no SAD/CDS forms, no MRN generation, no procedure codes. | EU import filing is completely non-functional. H1 (free circulation) is the EU equivalent of US CF-7501. | EU | `EUTaxRegime` computes tariff math | Backend: EU declaration model with declaration_type (H1-H7), procedure_code, MRN, customs_office_code; EU filing state machine | XL | P1 |
| **UB-023** | GL-012 | **UK CDS declaration workflow** | No UK Customs Declaration Service support. CDS uses Data Elements model, distinct from EU SAD. | UK import filing is non-functional. CDS replaced CHIEF entirely. | UK | None (UK falls through) | Backend: UK CDS declaration model, UK procedure codes, DUCR/MUCR, deferment account; depends on UB-021 | XL | P1 |
| **UB-024** | GL-013 | **India Bill of Entry / Shipping Bill workflow** | No ICEGATE filing model. No BE number, no BE types (home consumption/warehousing/ex-bond). | India import/export filing is non-functional. Bill of Entry is India's fundamental import declaration. | IN | `IndiaTaxRegime` computes BCD+SWS+IGST | Backend: Bill of Entry model (BE number, type, port_code, IEC, GSTIN, CHA license), Shipping Bill model; ICEGATE state machine | XL | P1 |
| **UB-025** | GL-014 | **Brazil DI/DUIMP filing workflow** | No Siscomex integration model. No DI number, no adicao management, no parametrizacao channels. | Brazil import filing is non-functional. DI registration in Siscomex is the Brazilian customs declaration. | BR | `BrazilTaxRegime` computes cascading taxes correctly | Backend: DI model (DI number, RADAR, CNPJ, NF-e access key, NCM additions), 4-channel parametrizacao state machine (green/yellow/red/grey) | XL | P1 |
| **UB-026** | GL-015 | **China GACC declaration workflow** | No Chinese customs declaration form. Only stub in `TERRITORY_FILINGS`. | China — the world's largest trading nation — has no filing capability beyond tariff calculation. | CN | `ChinaTaxRegime` computes MFN+retaliatory+consumption+VAT; territory stubs exist | Backend: GACC declaration model (50+ fields), trade mode codes, USCC identifier, Single Window response simulation | XL | P1 |
| **UB-027** | GL-016 | **EU FTA partners and preference rates** | No `FTA_PARTNERS_EU`. EU has the world's most extensive FTA network. | FTA utilization saves 3-15% duty. EU FTAs involve tariff quotas, cumulation rules, product-specific rules of origin. | EU, UK | `FTA_PARTNERS_US` exists as pattern | Backend: `FTA_PARTNERS_EU`, `FTA_PARTNERS_UK` dicts; preference rate lookup; origin proof type determination (EUR.1 vs REX vs invoice declaration) | L | P1 |
| **UB-028** | GL-017 | **India FTA partners and preference rates** | No `FTA_PARTNERS_IN`. India-ASEAN CECA, India-Japan CEPA, India-UAE CEPA (2022), India-Australia ECTA (2022) all missing. | FTA utilization is the #1 cost-saving opportunity for India importers. Preferential BCD can drop from 15-70% to 0-5%. | IN | `FTA_PARTNERS_US` exists as pattern | Backend: `FTA_PARTNERS_IN` dict; Certificate of Origin types per agreement (Form AI, Form AIJCEPA, etc.) | L | P1 |
| **UB-029** | GL-018 | **China RCEP and FTA preference rates** | No `FTA_PARTNERS_CN`. RCEP (world's largest FTA — 15 countries, 30% of global GDP) entirely missing. | RCEP is the single most important trade agreement for APAC. | CN, APAC | `FTA_PARTNERS_US` pattern | Backend: `FTA_PARTNERS_CN` with RCEP, ACFTA (Form E), China-Korea, ChAFTA rates; RCEP cumulation rules | L | P1 |
| **UB-030** | GL-019 | **India BCD database-backed lookup** | `IndiaTaxRegime.BCD_RATES` has 17 hardcoded rates that disagree with `seed_in_tariff.py` (coffee: 30% vs 100%; laptops: 0% vs 15%; toys: 20% vs 60%). | Incorrect duty calculations for India. Hardcoded rates and seed data conflict. | IN | `_lookup_mfn_db()` pattern exists in `USTaxRegime` | Backend: `_lookup_bcd_db()` method mirroring US pattern; reconcile BCD_RATES vs seed data | M | P1 |
| **UB-031** | GL-020 | **India IGST slab expansion** | Only 18% and 28% IGST rates. Missing 0%, 5%, and 12% slabs. | 5% IGST applies to basic food, fertilizers; 12% to processed food, apparel. Incorrect IGST = wrong landed cost. | IN | `IndiaTaxRegime` has 18%/28% only | Backend: per-HS-code IGST slab lookup (0/5/12/18/28%); add Compensation Cess | M | P1 |
| **UB-032** | GL-021 | **EU VAT mechanisms (PVA, Procedure 42)** | No Postponed VAT Accounting, no Procedure 42 VAT exemption, no reduced VAT rates. | PVA eliminates upfront VAT payment at border. Procedure 42 = zero import VAT for intra-EU dispatch. Missing these overstates cash flow requirements. | EU | `EUTaxRegime` has standard VAT rates for 27 MS | Backend: PVA flag; Procedure 42/63 VAT exemption logic; reduced/zero VAT rate lookup by HS code and member state | M | P1 |
| **UB-033** | GL-022 | **Brazil AFRMM and Taxa Siscomex** | AFRMM (8% of maritime freight for long-haul) and Taxa Siscomex (R$40 + R$10 per additional NCM) missing. | Mandatory fees on every Brazil import. AFRMM needs freight value field that doesn't exist. | BR | `BrazilTaxRegime` has correct tax cascade; no freight breakdown field | Backend: AFRMM calculation; Taxa Siscomex; requires CIF component breakdown (FOB + freight + insurance) | M | P1 |
| **UB-034** | GL-023 | **China provisional/interim tariff rates** | Only MFN rates. China frequently sets provisional rates lower than MFN for specific goods. | Provisional rates override MFN and are the actual applied rates for hundreds of tariff lines. | CN | `ChinaTaxRegime` has MFN lookup | Backend: provisional rate table; rate selection logic (provisional overrides MFN) | M | P1 |
| **UB-035** | GL-024 | **Japan tariff regime** | No `JapanTaxRegime`. No JP tariff computation at all. | Japan is the world's 4th largest economy. No tariff calculation for JP-destined goods. | JP | None | Backend: new `JapanTaxRegime` — MFN duty + consumption tax (10% standard, 8% reduced for food) | L | P1 |
| **UB-036** | GL-025 | **South Korea tariff regime** | No `KoreaTaxRegime`. No KR tariff computation. | Korea is a major US trade partner (KORUS FTA). | KR | None | Backend: new `KoreaTaxRegime` — MFN duty + VAT (10%); individual consumption tax for luxury goods | L | P1 |
| **UB-037** | BL-067, GL-028, GL-029, GL-030, GL-031 | **Jurisdiction-aware regulatory agency triggers** | US PGA shown as single "PGA Documentation" item, not per-agency. No non-US regulatory agency mapping at all. | Real PGA = per-agency (FDA, EPA, FCC, CPSC, APHIS, TTB). BIS certification is the biggest non-tariff barrier for India imports (~400+ mandatory products). CE marking is mandatory for EU products. Brazil requires pre-shipment licensing. CCC is mandatory for ~20 Chinese product categories. | ALL | `PGA_TRIGGERS` maps US agencies; `PGAActor` generates US PGA holds | Backend: `REGULATORY_TRIGGERS` registry per jurisdiction: US (FDA/EPA/CPSC/NHTSA/APHIS/TTB), EU (CE marking/REACH/UKCA), India (BIS/FSSAI/DGFT/WPC/CDSCO), Brazil (ANVISA/MAPA/INMETRO/ANATEL with pre-shipment LI), China (CCC/CIQ/NMPA/MIIT); Frontend: agency-specific checklist items per jurisdiction | L | P1 |
| **UB-038** | GL-027 | **EU/UK sanctions screening** | Entity screening only checks US lists (OFAC SDN, BIS Entity List, DPL). No EU Consolidated List, no UK OFSI. | Platform gives false compliance clearance for EU-destined goods. EU sanctions list differs from US. | EU, UK | `screen_entity` tool screens US lists | Backend: add EU Consolidated Financial Sanctions List, EU dual-use list (Annex I), UK OFSI Consolidated List | M | P1 |
| **UB-039** | GL-032 | **EU CBAM operational implementation** | Referenced in Engine 6 signal SIG-002 but no operational implementation. CBAM mandatory from 2026. | CBAM requires carbon certificates for iron/steel, aluminum, cement, fertilizers, electricity, hydrogen. Non-compliance blocks import. | EU | One CBAM signal in `e6_regulatory` | Backend: CBAM reporting module — HS code screening, quarterly report tracking, certificate requirement flagging | M | P1 |
| **UB-040** | GL-033 | **Import corridors to non-US destinations** | All corridors terminate at US. No EU-bound, India-bound, Brazil-bound, or China-bound corridors. | Simulation cannot generate non-US import scenarios. | ALL | Corridor definitions in `reference_data.py` | Backend: add corridors (CN→DE, CN→IN, CN→BR, US→CN, DE→IN, AR→BR, CN→JP, CN→KR, US→SG) | M | P1 |
| **UB-041** | GL-034 | **Non-US port reference data** | Only `US_PORT_CODES` dict (16 US ports). No EU customs offices, India customs stations, Brazil URF codes. | Port selection is US-only. | ALL | `US_PORT_CODES` in `reference_data.py` | Backend: `EU_CUSTOMS_OFFICES`, `IN_PORT_CODES`, `BR_URF_CODES`, `CN_CUSTOMS_DISTRICTS` | M | P1 |
| **UB-042** | GL-035 | **EU customs authority simulation actor** | No `EUCustomsAuthorityActor`. No AEO-differentiated clearance rates. | Cannot simulate EU customs clearance. AEO-F holders get ~95% auto-release vs 40% for new operators. | EU | `CBPAuthorityActor` exists as pattern | Backend: new `EUCustomsAuthorityActor` — risk profiling (AEO green lane), documentary checks, physical inspections, customs query, debt notification | L | P1 |
| **UB-043** | GL-036 | **India ICEGATE authority simulation actor** | No `ICEGATEAuthorityActor`. No RMS channel routing (green 70%/yellow 20%/red 10%). | Cannot simulate India customs clearance. | IN | `CBPAuthorityActor` as pattern | Backend: new `ICEGATEAuthorityActor` — RMS channels, First/Second Check, assessment query, speaking order, out-of-charge | L | P1 |
| **UB-044** | GL-037 | **Brazil Receita Federal simulation actor** | No `ReceitaFederalActor`. No 4-channel parametrizacao. | Cannot simulate Brazil customs clearance. Canal Cinza (grey channel) for fraud investigation is unique to Brazil. | BR | `CBPAuthorityActor` as pattern | Backend: new `ReceitaFederalActor` — 4-channel parametrizacao, exigencia fiscal, auto de infracao, 30-day response deadlines | L | P1 |
| **UB-045** | GL-038 | **Jurisdiction-aware agent tool descriptions** | All 12 agent tools have US-specific descriptions. `calculate_tariff` mentions "Section 301, 232, AD/CVD, MPF, HMF". | Agent gives US-specific advice for EU-destined shipments. Tells an India importer about "MPF" instead of "SWS". | ALL | `tools.py` has 12 tools with US descriptions | Backend: make tool descriptions dynamically context-aware based on shipment destination; update agent system prompts | M | P1 |
| **UB-046** | GL-039 | **New EU/APAC-specific agent tools** | No `check_eori_validity`, `lookup_preference_rate`, `check_tariff_quota`, `check_cbam_obligation`, `check_reach_status`, `file_ens`. | Agent cannot perform jurisdiction-specific operations. | EU, APAC | Tool framework in `tools.py` | Backend: 6-8 new tool implementations for EU/APAC-specific operations | L | P1 |
| **UB-047** | BL-012 | **Exception actions invoke APIs directly (jurisdiction-aware)** | `ExceptionActions.tsx:198-201` — Every action button calls `onOpenAssistant(action.prompt(...))`. All just open chat. | None invoke backend tools directly. Exception resolution screen cannot resolve exceptions. Actions must dispatch to jurisdiction-appropriate tools. | ALL | `screen_entity`, `draft_communication`, `resolve_shipment_hold` tools | Frontend: direct API calls instead of chat routing; jurisdiction-dispatched actions | M | P1 |
| **UB-048** | BL-013 | **Exception status transitions** | `ExceptionResolution.tsx:352-603` — No way to change exception status. Entirely read-only. | An exception resolution screen that cannot resolve. | ALL | Resolution endpoints exist | Frontend: status transition UI (Resolve, Escalate, Assign, Dismiss) with audit trail; Backend: status update API | M | P1 |
| **UB-049** | BL-014, BL-060 | **Broker queue search + sort** | `BrokerQueue.tsx:308-338` — Only status and priority dropdowns. No text search. No sort. | Critical when a broker gets a call about a specific entry. Brokers need to sort by days waiting, value, priority. | ALL | `search_by_reference` tool in `tools.py:651-668` | Frontend: search bar querying across entry number, company name, product, reference numbers; sortable column headers; default sort by urgency | S | P1 |
| **UB-050** | BL-015 | **Fix "Contact Shipper" button (no-op)** | `ExceptionQueue.tsx:379-384` — `onClick` only calls `e.stopPropagation()`. Button does literally nothing. | A dead button in the primary ops dashboard undermines trust. | ALL | `draft_communication` endpoint | Frontend: modal with draft communication flow using `draft_communication` backend | S | P1 |
| **UB-051** | BL-011, GL-046 | **Anti-dumping/countervailing duty framework** | No data field for AD/CVD case numbers or deposit rates anywhere. AD/CVD can add 100-400%+ to duty. India is one of the world's largest users of anti-dumping duties. | Wrong duties = post-liquidation demand letters. Framework serves both US and India (and others). | US, IN (extensible) | `CustomsActor` HAS AD/CVD case matching (US); India ADD cases specified in GL-046 | Backend: jurisdiction-aware ADD/CVD table (`add_cvd_cases` with jurisdiction field); case number, deposit rate, origin country; Frontend: flags + display on entries | L | P1 |
| **UB-052** | BL-010, GL-026 | **Pre-arrival security filing framework (ISF + ICS2)** | No ISF status indicator despite being legally required for US ocean shipments ($5K penalty). No EU ICS2 ENS support (mandatory for all EU imports, all modes). | ISF and ICS2 are parallel concepts — US ocean pre-arrival vs EU all-mode pre-arrival. Build one generic "pre-arrival filing" framework. | US, EU | ISFActor exists in simulation; ICS2 is analogous | Backend: generic pre-arrival filing model (US: ISF 10+2 ocean-only; EU: ICS2 ENS all-modes + PLACI + DNL); filing deadline tracking; Frontend: status badges, deadline countdowns, late filing warnings | L | P1 |
| **UB-053** | BL-016 | **Simulation control panel UI** | Backend has full simulation API (start/stop/pause/resume/reset, actor config, clock control). No frontend. | No UI to start/stop/configure simulations. Must support jurisdiction actor selection from day one. | ALL | Full simulation API at `/api/simulation/start|stop|pause|resume|reset` | Frontend: Simulation Control Panel — start/stop/pause/resume, scenario seed, actor parameter sliders (including jurisdiction actor selection), speed control, data reset, real-time event stream | L | P1 |
| **UB-054** | BL-017 | **Compose modal loses entry context** | `Communications.tsx:99-286` — When opened from "New Message", neither `defaultShipmentId` nor `defaultPurpose` is provided. AI Draft disabled. | AI Draft is always disabled when opening from Communications page directly. | ALL | Broker queue data available | Frontend: shipment selector dropdown; selecting a shipment enables AI Draft | S | P1 |
| **UB-055** | BL-025 | **Incoterms / valuation method on products and orders** | No field for terms of sale (FOB/CIF/EXW/DDP). Determines what is in declared value, who pays freight/insurance, risk transfer point. | Without Incoterms, duty calculation is unreliable. $10,000 FOB vs. $10,000 CIF produces different duty amounts. Critical for all jurisdictions (Brazil AFRMM needs CIF breakdown). | ALL | Tariff engine would use this for valuation | Backend: Incoterms field on data model; Frontend: dropdown + auto-adjust declared value calculation | M | P1 |
| **UB-056** | BL-024 | **Buyer shipping cost — replace $350 hardcode** | `ProductPage.tsx:49`: `const shippingEstimate = 350` for all products. 500kg tea and single sweater cost the same. | Pricing fiction extends to logistics. Missing MPF, HMF, broker fees, insurance, drayage. Must be jurisdiction-aware (different fee components per destination). | ALL | Weight data needed; routing engine available | Backend: shipping estimate endpoint using weight, origin, destination, mode; jurisdiction-specific fees; Frontend: dynamic display | M | P1 |

---

### P2 — Operational Depth + Passive to Actionable (25 items)

| ID | Original IDs | Title | What's Wrong / Missing | Why It Matters | Jurisdictions | Backend Available | Changes Needed | Effort | Priority |
|---|---|---|---|---|---|---|---|---|---|
| **UB-057** | BL-033 | **"Run Compliance Check" button** | `check_compliance` tool (PGA + DPS + UFLPA) not surfaced as button anywhere. Brokers access only via chat. | Compliance pre-check before submission catches issues early. Must invoke jurisdiction-appropriate compliance checks. | ALL | `check_compliance` tool | Frontend: compliance check panel + button; jurisdiction-dispatched checks | M | P2 |
| **UB-058** | BL-057 | **Entry readiness gate before submission** | Submit button appears based on status alone, not actual completeness. No `check_entry_readiness` pre-flight. | "Before showing 'Submit to [Authority]', run pre-flight checklist." Block submission if critical items incomplete. | ALL | `check_entry_readiness` tool | Frontend: pre-flight modal (doc completeness, classification confidence, fee calc status, compliance check); jurisdiction-aware gate | M | P2 |
| **UB-059** | BL-022 | **Cross-screen workflow linking** | Exception Resolution doesn't link to Entity Screening, Product Analysis, or Regulatory Intel. Each tool is an island. | Ops director's ideal flow: click alert → Exception Resolution → "Screen Entity" → Entity Screening pre-filled → result feeds back. | ALL | All screens exist; need URL parameter passing | Frontend: deep-link navigation with context passing | M | P2 |
| **UB-060** | BL-023 | **Orders and Entries pagination + filters** | `PlatformOrders.tsx:173-226` — All 1,958 orders rendered at once. No pagination, no filters, no search. | Performance problem. Orders stuck in Draft with no duty data. | ALL | Pagination patterns exist in BrokerQueue | Frontend: pagination + filters; Backend: paginated endpoint | M | P2 |
| **UB-061** | BL-029 | **CF-28/authority inquiry response — document attachment** | `CBPResponses.tsx:210-341` — Modal has text area but no file upload. Guidance suggests documents but cannot attach them. | Real authority inquiry responses almost always require documentary evidence. Backend schema has `attachments: list[str]` but UI doesn't expose it. Applies to all jurisdiction inquiry response flows. | ALL | Backend schema supports attachments | Frontend: file upload in response modal; pre-populate with suggested docs | S | P2 |
| **UB-062** | BL-036 | **Clickable KPI cards in Control Tower** | `PlatformPulse.tsx:130-163` — Numbers are display-only. "47.8K Entries" should link to entries list. | Dashboard should be a navigation hub, not just a display. | ALL | Views exist for all metrics | Frontend: click handlers + navigation | S | P2 |
| **UB-063** | BL-037 | **Computed KPI trend values** | `PlatformPulse.tsx:33,38,43,48,53` — "+8%", "+1.2%" are static strings. | Misleading in a live dashboard. | ALL | Historical data available in DB | Backend: trend computation endpoint; Frontend: dynamic display | M | P2 |
| **UB-064** | BL-038 | **Surface cargo exceptions in Exception Queue** | ExceptionsActor generates shortage/overage/damage/misdeclared weight — but Exception Resolution only shows compliance holds. | 7.5% of shipments generate cargo exceptions that are invisible. The #1 daily workflow for ops teams. | ALL | Events exist in DB from ExceptionsActor | Frontend: expand exception type filter; Backend: include cargo events in query | M | P2 |
| **UB-065** | BL-039 | **Active Shipments search/filter** | Only status and mode filters. No text search, no company filter, no date range. | Unmanageable for 100+ shipments. | ALL | Data available via API | Frontend: search + filter controls | S | P2 |
| **UB-066** | BL-040 | **Exception assignment model** | No concept of who is working on each exception. No "Assign to me." | Ops teams need ownership to prevent duplicate work. | ALL | Needs assignment field | Backend: owner field; Frontend: assignment UI | M | P2 |
| **UB-067** | BL-041 | **Entity screening history/audit log** | Every screening is ephemeral. No log of past screenings. | Compliance audits require evidence of screening. Must log which screening lists were checked (OFAC, EU Consolidated, etc.). | ALL | Needs persistence layer | Backend: screening history table (with jurisdiction/list info); Frontend: history view | M | P2 |
| **UB-068** | BL-042 | **Batch entity screening** | One entity at a time. Order with 5 parties requires 5 separate screens. | Multi-party screening is the norm. Must screen against jurisdiction-appropriate lists. | ALL | `screen_entity` handles one at a time | Backend: batch endpoint (screening against jurisdiction-relevant lists); Frontend: batch UI, CSV upload | M | P2 |
| **UB-069** | BL-043 | **"Req Docs" button — pass entry context** | `BrokerQueue.tsx:121-130` — Navigates to `/broker/messages` without context. | Broker loses entry context. | ALL | `draft_communication` endpoint accepts `shipment_id` and `purpose` | Frontend: modal with context passing | S | P2 |
| **UB-070** | BL-044 | **Deadline/urgency info on queue cards** | Only "days since arrival". No GO deadline, no storage costs, no cascade warnings. | Broker cannot prioritize from queue view. | ALL | Data available from backend | Frontend: deadline countdown badge, storage cost indicator, inquiry deadline | S | P2 |
| **UB-071** | BL-045 | **Work plan items — inline actions** | "Approve" navigates away. For simple actions, should act from dashboard. | Dashboard should support quick actions without navigation. | ALL | Approval endpoint exists | Frontend: inline action modals | S | P2 |
| **UB-072** | BL-046, BL-047 | **Buyer checkout — destination country + analysis before purchase** | Uses hardcoded US destination. Tariff analysis runs AFTER purchase, not before. | Duty rates are destination-specific. Analysis should inform pricing, not run post-commit. | ALL | Analysis endpoint accepts destination | Frontend: country selector; move analysis to product view/cart add time; recalculate on country change | M | P2 |
| **UB-073** | BL-048 | **ETA + filing deadlines on Shipper shipments** | No ETA, no ISF deadline, no entry number on shipment detail/list. | ETA is the #1 thing every importer checks daily. Filing deadlines vary by jurisdiction (ISF 24h for US, ICS2 ENS varies by mode for EU). | ALL | Routing engine has transit times; ISFActor tracks deadlines | Frontend: ETA display; Backend: ETA computation from route; jurisdiction-specific filing deadline display | M | P2 |
| **UB-074** | BL-049 | **Product Analysis — link to exceptions** | Re-classification results cannot be applied to classification dispute exceptions. | Tools are siloed. "Apply to exception" button needed. | ALL | Classification data available | Frontend: cross-linking + history | M | P2 |
| **UB-075** | BL-050 | **Shipper catalog — search, count, readiness** | No search, no filter, no compliance readiness per product card. | Catalog grows but discovery doesn't scale. | ALL | Last analysis data available per product | Frontend: readiness indicator (green/yellow/red) + search + count | S | P2 |
| **UB-076** | BL-051 | **Duty estimate on Create Order page** | Order summary shows product value only. No duty preview. | Shippers make decisions based on cost. Orders created without cost visibility. | ALL | Analysis endpoint exists | Frontend: inline estimate + API call; uses jurisdiction-aware tariff | M | P2 |
| **UB-077** | BL-052 | **CBP/Authority Responses — link to Entry Detail** | Entry number shown but not clickable. | Brokers need to review full entry before drafting response. | ALL | Navigation only | Frontend: clickable link to entry detail | S | P2 |
| **UB-078** | BL-053 | **Message threading in Communications** | Flat card list. 10 messages about one shipment mixed with others. | Broker handling 20 entries needs grouped communications. | ALL | Needs threading model | Backend: thread grouping; Frontend: threaded UI | M | P2 |
| **UB-079** | GL-048 | **Multi-currency tariff computation** | Engine computes in USD only. EU duties in EUR, India in INR, Brazil in BRL, China in RMB. | Duty amounts displayed in wrong currency. | ALL | All tariff engines output USD | Backend: exchange rate integration; per-jurisdiction currency; multi-currency display | M | P2 |
| **UB-080** | BL-032 | **Buyer price breakdown math consistency** | `buyerPrice` is a separate hardcoded number. Components don't sum to total. | Financial inconsistency destroys trust. | ALL | Tariff engine provides real numbers | Frontend: compute total from components; no separate hardcoded total | S (with UB-007) | P2 |
| **UB-081** | BL-059 | **Bulk actions across surfaces** | No bulk claim, approve, select, or export anywhere. | 5 separate API calls with 5 loading spinners for claiming 5 shipments. | ALL | Batch endpoints may need creation | Frontend: multi-select + batch toolbar; Backend: batch endpoints | M | P2 |

---

### P3 — Advanced Features + Proactive Intelligence (17 items)

| ID | Original IDs | Title | What's Wrong / Missing | Why It Matters | Jurisdictions | Backend Available | Changes Needed | Effort | Priority |
|---|---|---|---|---|---|---|---|---|---|
| **UB-082** | BL-062 | **Post-release / liquidation tracking** | Platform treats "Released" as terminal state. Real lifecycle: Released → Liquidated (~314 days for US) → Protest window. | Missed protest deadlines = lost refund opportunities. Every broker tracks post-release. Liquidation timelines vary by jurisdiction. | US (extensible) | Needs status expansion | Backend: status additions (Released → Liquidated → Protest Window); jurisdiction-specific timelines; Frontend: pipeline stage display | L | P3 |
| **UB-083** | BL-058 | **Time range selector on Control Tower** | Dashboard shows "month" data but no way to change. | Ops director needs different views for different decisions. | ALL | Historical data in DB | Frontend: selector (Today/Week/Month/Quarter/Custom); all metrics recalculate | M | P3 |
| **UB-084** | BL-065 | **Global floating assistant** | `ShipmentChat.tsx` only on ShipmentDetail. Not on OrderDetail, ProductDetail, or Resolution Center. | Useful tool locked to one screen. | ALL | Chat API works with any context | Frontend: lift to layout level; context-aware based on current screen and jurisdiction | M | P3 |
| **UB-085** | BL-066 | **Conversation persistence** | Messages stored in Zustand, not persisted across page refreshes. | Conversation lost on refresh or navigation. | ALL | Needs persistence endpoint or localStorage | Frontend: persistence layer | M | P3 |
| **UB-086** | BL-021 | **Held shipments — show hold context on cards** | Held shipments in unassigned list show red badge but only "Claim" action. No hold type, no cage status preview. | Broker needs to know WHY it is held before claiming. | ALL | `get_cage_status` tool, hold_type data in backend | Frontend: hold type badge, one-line cage status, hover preview | S | P3 |
| **UB-087** | BL-020 | **Missing docs sidebar — add actions** | `EntryDetail.tsx:1290-1303` — "Missing Documents" bullet points with no actions. | No action pathway from identified gap to resolution. | ALL | Document generation pipeline | Frontend: per-doc actions (click to scroll, Upload, Generate); "Request All" button | S | P3 |
| **UB-088** | BL-031 | **Improvement steps — make clickable** | `EntryDetail.tsx:881-890` — `improvementSteps` renders as bullet points. | Displayed information should connect to actions. | ALL | Document checklist exists in same screen | Frontend: click handlers + scroll-to + action triggers | S | P3 |
| **UB-089** | BL-035 | **Hold details banner — add action button** | Shows storage costs and GO deadline but no action. | Should have a "Begin Resolution" button. | ALL | `BrokerIntelligenceService.get_resolution_path()` | Frontend: resolution button + navigation | S | P3 |
| **UB-090** | BL-061 | **HTS/tariff code on Broker queue cards** | Cards show product, company, route, value — not the HS code. | The HS code is the single most important data point for a customs broker. Display must show jurisdiction-appropriate code format. | ALL | Data available from entry | Frontend: tariff code display (format varies by jurisdiction) + confidence indicator | S | P3 |
| **UB-091** | BL-063 | **Resolution steps fallback improvement** | When AI fails, shows only "Contact customs broker." | Zero actionable value. | ALL | Static content | Frontend: per-hold-type fallback steps; jurisdiction-specific guidance | S | P3 |
| **UB-092** | BL-064 | **Resolution Center — link to shipment** | Shows resolution steps but no "Go to Shipment." | Cannot take action on suggestions. | ALL | Navigation only | Frontend: "View Shipment" link | S | P3 |
| **UB-093** | BL-068 | **ShipmentDetail — layout restructure** | 12+ sections in one scroll. Information overload. | Hard to find specific data. | ALL | N/A | Frontend: tab layout or accordion (Overview / Analysis / Documents / Timeline / Financials) | M | P3 |
| **UB-094** | BL-056 | **Duplicate HS code display in ShipmentDetail** | Classification shown in "Analysis Results" AND again in separate "Classification" section. | Confusing redundancy. | ALL | N/A | Frontend: consolidate into single section | S | P3 |
| **UB-095** | GL-040 | **EU customs procedures (warehousing, IP, OP, transit)** | Only free circulation implied. No customs warehousing, inward processing, outward processing, or NCTS transit. | These procedures unlock significant business value. Warehousing defers duty; IP allows duty-free import for re-export manufacturing. | EU | No procedure code concept | Backend: procedure code field; IP/OP duty adjustments; transit declaration model; guarantee management | XL | P3 |
| **UB-096** | GL-041, GL-042 | **Export support — India (Shipping Bill/RoDTEP) + Brazil (DU-E/drawback)** | No export filing workflow for any jurisdiction. No duty drawback, no RoDTEP rate table, no DU-E. | India and Brazil export incentive schemes are essential for exporters. Platform is import-only. | IN, BR | No export workflow exists | Backend: Shipping Bill + DU-E models; drawback calculation; RoDTEP rate table; export NF-e linkage | L | P3 |
| **UB-097** | GL-043, GL-044 | **China special trade regimes (e-commerce + processing trade)** | No cross-border e-commerce tax regime (15%/25%/50% instead of duty+VAT for parcels under RMB 5,000). No processing trade duty exemption. | Cross-border e-commerce is fastest-growing China trade segment. Processing trade is ~15% of China trade volume. | CN | `ChinaTaxRegime` handles general trade only | Backend: e-commerce trade mode detection + tax calc; processing trade manual system; bond management | L | P3 |
| **UB-098** | GL-050 | **China GACC authority simulation actor** | No simulation of Chinese customs responses, CIQ inspection, or enterprise credit-based channel routing. | Cannot demonstrate China clearance workflow. | CN | `CBPAuthorityActor` as pattern | Backend: new `GACCAuthorityActor` — enterprise credit-based channels, CIQ inspection, price verification | L | P3 |

---

### P4 — Polish, Parity, Future (6 items)

| ID | Original IDs | Title | What's Wrong / Missing | Why It Matters | Jurisdictions | Backend Available | Changes Needed | Effort | Priority |
|---|---|---|---|---|---|---|---|---|---|
| **UB-099** | BL-071 | **Keyboard shortcuts in EntryDetail** | No shortcuts for Approve, Submit, Open Assistant. | Speed matters for high-volume brokers. | ALL | N/A | Frontend: keyboard handlers + shortcut hints | S | P4 |
| **UB-100** | BL-072 | **Broker license number format** | Shows "CHB CHB-4892" — CBP license numbers are 5 digits. | Minor domain credibility issue. License format should be jurisdiction-appropriate. | US | Static data fix | Frontend: format fix; generalize for non-US broker credential display | S | P4 |
| **UB-101** | BL-070 | **Quantity selector on buyer product page** | "Add to Cart" adds exactly 1 unit. | For B2B products like 500 aluminum brackets, this is wrong. | ALL | N/A | Frontend: quantity input with price recalculation | S | P4 |
| **UB-102** | GL-049 | **Windsor Framework (Northern Ireland)** | No GB→NI movement handling, "Not At Risk" determination, or green/red lane routing. | One of the most complex customs arrangements in the world. | UK | No NI-specific logic | Backend: Windsor Framework routing; "Not At Risk" determination; green/red lane | L | P4 |
| **UB-103** | GL-051, GL-052, GL-053 | **APAC expansion — Singapore, Vietnam, seasonal events** | No Singapore TradeNet, no Vietnam VNACCS/VCIS. Missing Chuseok, Tet, Songkran seasonal events. | APAC trade growth requires these. Singapore is a major transshipment hub; Vietnam is a top-5 US trade partner. | SG, VN, APAC | Singapore/Vietnam port data exists; seasonal events framework exists | Backend: TradeNet/VNACCS declaration stubs; VN tax regime; APAC seasonal events | L | P4 |
| **UB-104** | GL-054, GL-055, GL-045, GL-047 | **Deep jurisdiction features — EUDR, China UEL, EU excise, Brazil ICMS-ST** | EU Deforestation Regulation not tracked. China Unreliable Entity List not screened. EU excise duties missing. Brazil ICMS-ST (hundreds of product-state combinations) not computed. | Advanced compliance and tax completeness for domain experts. | EU, CN, BR | Minimal | Backend: EUDR commodity screening; China UEL in screening engine; EU excise by MS; Brazil ICMS-ST/DIFAL with MVA tables | XL | P4 |

---

## 3. Implementation Roadmap (6 Sprints)

### Sprint 1: "Jurisdiction Foundation + Kill Placeholders" (3 weeks)
**Theme**: Build the jurisdiction abstraction layer AND replace hardcoded content simultaneously. Every "placeholder fix" is built jurisdiction-aware from day one.

| ID | Item | Effort | Why This Sprint |
|---|---|---|---|
| UB-001 | Jurisdiction detection + ISO country picker | M | Unlocks everything — must come first |
| UB-002 | Polymorphic declaration model + entry number/field fixes | L | Foundation for all non-US declarations + fixes US credibility |
| UB-003 | Jurisdiction-specific state machines (framework) | L | Enables non-US lifecycles |
| UB-004 | Frontend jurisdiction-aware rendering (label extraction) | L | Parameterizes all hardcoded US labels |
| UB-007 | Buyer duty calculation — replace 5% hardcode | M | Highest-impact placeholder fix |
| UB-008 | Buyer product catalog from API | L | Second-highest placeholder fix |
| UB-013 | Fix draft_communication placeholder | S | Quick win — LLM fallback |
| UB-015 | Compliance Dashboard — wire to real data | M | 100% fake data → real data |
| UB-016 | Separate held vs. rejected status | S | Quick data model fix |
| UB-017 | Global error handling + toast system | S | Quick win — fixes 5+ screens |
| UB-018 | HS-code-aware simulation tariff | L | Simulation data integrity |
| UB-019 | CustomsActor reads misclassified flag | S | Quick simulation fix |

**Outcome**: Platform distinguishes jurisdictions. US behavior correct with real data. Labels parameterized. Framework ready. No more 5% duty, fake compliance dashboard, or placeholder communications.

---

### Sprint 2: "Information to Action" (2 weeks)
**Theme**: Convert passive text into clickable actions using jurisdiction-aware tool dispatch. Surface the 22 backend tools as buttons, not chat prompts.

| ID | Item | Effort | Why This Sprint |
|---|---|---|---|
| UB-005 | Generalized authority response screen + CF-29 draft | L | Abstracted for all jurisdictions while adding CF-29 |
| UB-009 | Jurisdiction-aware checklist with Generate/Upload/Request | L | Core broker workflow — jurisdiction-driven items |
| UB-010 | Resolution steps → action buttons | M | #1 broker workflow gap |
| UB-011 | verify_classification inline panel | M | Most-used broker tool |
| UB-012 | calculate_entry_fees button + panel | M | Financial accuracy |
| UB-014 | Regulatory intelligence from API + CRUD | L | Kill hardcoded signals |
| UB-020 | Hold-type-aware resolution timelines | M | Simulation realism |
| UB-047 | Exception actions invoke APIs directly | M | Stop routing everything to chat |
| UB-048 | Exception status transitions | M | Resolution screen that resolves |
| UB-049 | Broker queue search + sort | S | Essential operational tool |
| UB-050 | Fix Contact Shipper button | S | Dead button fix |

**Outcome**: Brokers and ops can act directly on every recommendation. 22 tools surfaced as buttons. Exception resolution works. Queue searchable. All built with jurisdiction dispatch.

---

### Sprint 3: "EU + UK First-Class" (3 weeks)
**Theme**: EU is the highest-value non-US jurisdiction. UK separation. EORI. Declaration workflow. FTA preferences. Sanctions screening.

| ID | Item | Effort | Why This Sprint |
|---|---|---|---|
| UB-006 | Importer identity framework (IOR + EORI + IEC + CNPJ) | XL | Foundation identity model — EU and UK need EORI immediately |
| UB-021 | UK tariff engine separation | L | UK duties currently wrong |
| UB-022 | EU customs declaration workflow (H1-H7) | XL | EU filing capability |
| UB-023 | UK CDS declaration workflow | XL | UK filing capability |
| UB-027 | EU FTA partners and preference rates | L | 3-15% duty savings |
| UB-032 | EU VAT mechanisms (PVA, Procedure 42) | M | Cash flow optimization |
| UB-037 | Jurisdiction-aware regulatory agency triggers (EU first) | L | CE/REACH/UKCA triggers |
| UB-038 | EU/UK sanctions screening | M | Compliance correctness |
| UB-039 | EU CBAM operational implementation | M | 2026 mandatory |
| UB-040 | Non-US corridors (EU portion) | S | EU simulation scenarios |
| UB-041 | Non-US port reference data (EU portion) | S | EU declaration needs offices |
| UB-042 | EU customs authority simulation actor | L | EU scenario simulation |
| UB-052 | Pre-arrival security filing (ISF + ICS2) | L | US ISF + EU ICS2 together |
| UB-055 | Incoterms / valuation method | M | Required for EU CIF valuation |

**Outcome**: EU and UK declarations can be filed and processed. EORI validated. EU-specific checklist. ICS2 ENS tracking. EU FTA preferences. CBAM flagging. UK tariff engine correct. Importer identity model serves all jurisdictions going forward.

---

### Sprint 4: "India + Brazil First-Class" (3 weeks)
**Theme**: India (fast-growing trade) and Brazil (complex tax, largest LatAm economy).

| ID | Item | Effort | Why This Sprint |
|---|---|---|---|
| UB-024 | India Bill of Entry workflow | XL | India filing capability |
| UB-025 | Brazil DI/DUIMP workflow | XL | Brazil filing capability |
| UB-028 | India FTA partners (ASEAN, UAE, Australia) | L | Preferential duty savings |
| UB-030 | India BCD database-backed lookup | M | Fix conflicting rates |
| UB-031 | India IGST slab expansion | M | Correct landed cost |
| UB-033 | Brazil AFRMM and Taxa Siscomex | M | Mandatory Brazil fees |
| UB-037 | Regulatory triggers (India + Brazil portion) | M | BIS/FSSAI + ANVISA/MAPA |
| UB-040 | Non-US corridors (India + Brazil portion) | S | Simulation scenarios |
| UB-041 | Non-US port reference data (India + Brazil portion) | S | Filing needs port codes |
| UB-043 | India ICEGATE simulation actor | L | India scenario simulation |
| UB-044 | Brazil Receita Federal simulation actor | L | Brazil scenario simulation |
| UB-051 | Anti-dumping/CVD framework (US + India) | L | Both jurisdictions need this |
| UB-053 | Simulation control panel (with jurisdiction actors) | L | Control all jurisdiction actors |

**Outcome**: India and Brazil declarations can be filed. Correct taxes with proper slabs. Regulatory agencies trigger appropriate holds. AD/CVD framework serves US and India. Simulation controllable with jurisdiction selection.

---

### Sprint 5: "China/APAC + Simulation Globalization" (3 weeks)
**Theme**: China first-class. Japan and Korea tariff regimes. Multi-jurisdiction simulation. Agent intelligence globalization.

| ID | Item | Effort | Why This Sprint |
|---|---|---|---|
| UB-026 | China GACC declaration workflow | XL | China filing capability |
| UB-029 | China RCEP and FTA preferences | L | World's largest FTA |
| UB-034 | China provisional/interim rates | M | Actual applied rates |
| UB-035 | Japan tariff regime | L | World's 4th economy |
| UB-036 | South Korea tariff regime | L | Major US trade partner |
| UB-037 | Regulatory triggers (China portion) | M | CCC/CIQ/NMPA |
| UB-040 | Non-US corridors (China + APAC portion) | S | Remaining corridors |
| UB-041 | Non-US port reference data (China + APAC) | S | Remaining ports |
| UB-045 | Jurisdiction-aware agent tool descriptions | M | Agent stops giving US advice for EU trade |
| UB-046 | New EU/APAC agent tools | L | EORI validity, preference lookup, CBAM check |
| UB-079 | Multi-currency tariff computation | M | Duties in correct currency |
| UB-098 | China GACC simulation actor | L | China scenario simulation |

**Outcome**: China declarations functional. Japan and Korea tariff calculations work. Simulation generates multi-jurisdiction scenarios. Agent adapts advice by jurisdiction. Multi-currency support.

---

### Sprint 6: "Advanced Features + Polish" (2 weeks)
**Theme**: Remaining operational depth, cross-cutting UX improvements, advanced jurisdiction features.

| ID | Item | Effort | Why This Sprint |
|---|---|---|---|
| UB-057 | Compliance check button | M | Pre-submission safety |
| UB-058 | Entry readiness gate | M | Pre-submission completeness |
| UB-059 | Cross-screen workflow linking | M | Tools connected |
| UB-060 | Orders/Entries pagination | M | Scalability |
| UB-064 | Cargo exceptions in queue | M | 7.5% invisible exceptions |
| UB-066 | Exception assignment model | M | Ownership tracking |
| UB-067 | Entity screening audit log | M | Compliance audit evidence |
| UB-072 | Buyer checkout — destination + pre-purchase analysis | M | Pricing accuracy |
| UB-078 | Message threading | M | Communication organization |
| UB-081 | Bulk actions | M | Operational efficiency |
| UB-082 | Post-release/liquidation tracking | L | Full lifecycle |
| UB-084 | Global floating assistant | M | Assistant everywhere |
| UB-085 | Conversation persistence | M | Context retention |
| UB-093 | ShipmentDetail layout restructure | M | Information architecture |

**Outcome**: Complete platform — every jurisdiction first-class, every tool surfaced, operational at scale, audit-ready.

### Git Branching Strategy

All work is done in feature branches off `main`. Each sprint gets its own long-lived branch, with per-feature sub-branches merged into it via PR:

| Branch | Sprint | Scope |
|--------|--------|-------|
| `sprint-1/jurisdiction-foundation` | Sprint 1 | Abstraction layer + placeholder kills. Merged to `main` before Sprint 2 starts — all subsequent work depends on this. |
| `sprint-2/information-to-action` | Sprint 2 | Tool surfacing, action buttons, jurisdiction-aware from day one. |
| `sprint-3/eu-uk-first-class` | Sprint 3 | EU/UK declaration workflows, EORI, ICS2, UK tariff engine, EU sanctions. |
| `sprint-4/india-brazil-first-class` | Sprint 4 | India (ICEGATE, BIS/FSSAI) + Brazil (Siscomex, DUIMP, parametrização). Can run parallel to Sprint 3 if team capacity allows. |
| `sprint-5/china-apac-simulation` | Sprint 5 | China/APAC clearance + simulation globalization. |
| `sprint-6/advanced-polish` | Sprint 6 | Advanced features, lifecycle, export support, polish. |

**Rules:**
- Feature branches: `sprint-N/feature-description` (e.g., `sprint-1/jurisdiction-config-registry`, `sprint-3/eori-field-support`)
- Sprint 1 MUST merge to `main` before other sprints branch — it is the foundation everything else builds on
- Sprints 3 and 4 may run in parallel branches off the Sprint 2 merge point if team size permits
- Each PR requires passing build + all tests before merge (per development standards)
- No long-lived branches beyond one sprint — merge to `main` at sprint completion

---

## 4. Generalization Guide

For each BL item that was rewritten to be jurisdiction-aware, here is exactly what changed.

| Unified ID | Original BL Item | Original US-Only Scope | New Jurisdiction-Aware Scope |
|---|---|---|---|
| **UB-001** | BL-027 | "Validate ISO-2 country codes, reject invalid" | Country picker drives `useJurisdiction()` hook. Selecting "DE" triggers EU config; "IN" triggers India config. Picker is the atomic unit for jurisdiction detection (merged with GL-001). |
| **UB-002** | BL-054 + BL-030 | "Fix US entry number format (filer code + check digit) + add missing CF-7501 fields" | Entry number, declaration type, and field schema all come from jurisdiction config. US: filer code + 7 + check digit. EU: MRN (18-char). India: BE number. Brazil: DI number. Missing field additions go into `jurisdiction_config` JSONB. |
| **UB-005** | BL-019 | "Add CF-29 protest AI draft capability" | CF-29 is the US instance of "authority notice response." Generalized to `AuthorityInquiries` screen dispatching to US (CF-28/CF-29), EU (customs query/audit), India (assessment query/speaking order), Brazil (exigencia fiscal/auto de infracao). AI Draft works for all. |
| **UB-006** | BL-026 | "Add IOR entity with EIN, bond, POA, Form 5106" | IOR becomes `ImporterIdentity` with jurisdiction-specific credential types. US: EIN + bond + POA. EU: EORI + AEO status. India: IEC + GSTIN + AD Code. Brazil: CNPJ + RADAR modality. One model, jurisdiction-dispatched validation. |
| **UB-007** | BL-001 | "Replace 5% with US tariff engine call" | Tariff call uses `destination` parameter. US buyers see MFN + 301 + MPF + HMF. EU buyers see MFN + anti-dumping + VAT. India buyers see BCD + SWS + IGST. Same API, jurisdiction-dispatched breakdown. |
| **UB-009** | BL-009 | "Port DocumentRequirements.tsx (Generate + Upload) to Broker surface" | Checklist items loaded from jurisdiction config. US: bond/PGA/origin cert. EU: EORI/CE/REACH/EUR.1. India: IEC/BIS/FSSAI. Brazil: RADAR/NF-e/LI. Generate/Upload/Request capabilities work for all document types. |
| **UB-010** | BL-003 + BL-018 | "Convert resolution `<li>` elements to buttons calling `screen_entity`, `draft_communication`, etc." | Action buttons dispatch to jurisdiction-appropriate tools. "Screen entity" checks OFAC for US entries, EU Consolidated List for EU entries, DGFT denied parties for India entries. Resolution path is jurisdiction-context-aware. |
| **UB-011** | BL-004 | "Add verify_classification inline panel showing HTS code, GRI analysis, confidence" | Panel displays tariff code in jurisdiction-appropriate format: HTS-10 (1234.56.7890) for US, CN-8 + TARIC-2 (12345678-90) for EU, ITC-HS-8 (12345678) for India, NCM-8 (1234.56.78) for Brazil. GRI analysis universal. |
| **UB-012** | BL-005 + BL-034 | "Show full US tariff breakdown: MFN + 301 + IEEPA + AD/CVD + MPF + HMF" | Breakdown is jurisdiction-specific. US: MFN + 301 + IEEPA + AD/CVD + MPF + HMF. EU: MFN + anti-dumping + VAT (+ CBAM if applicable). India: BCD + SWS + IGST (+ Compensation Cess). Brazil: II + IPI + PIS + COFINS + ICMS. |
| **UB-014** | BL-007 + BL-069 | "Fetch US regulatory signals from API; add CRUD for platform admin" | Signals have a `jurisdiction` field. Admin can create signals for any jurisdiction (EU CBAM enforcement, India BIS standard update, Brazil ICMS rate change, China retaliatory tariff). Filtered by relevant jurisdictions on display. |
| **UB-037** | BL-067 | "Expand PGA to per-agency: FDA, EPA, FCC, CPSC, APHIS, TTB" | PGA becomes `REGULATORY_TRIGGERS` — a jurisdiction-keyed registry. US: FDA/EPA/CPSC/NHTSA/APHIS/TTB. EU: CE directives/REACH/CBAM. India: BIS/FSSAI/DGFT/WPC/CDSCO. Brazil: ANVISA/MAPA/INMETRO/ANATEL. China: CCC/CIQ/NMPA/MIIT. |
| **UB-047** | BL-012 | "Exception action buttons call screen_entity, draft_communication directly instead of opening chat" | Tool invocation is jurisdiction-dispatched. "Screen Entity" screens against OFAC (US), EU Consolidated List (EU), DGFT (India). "Draft Communication" uses jurisdiction-appropriate authority references and terminology. |
| **UB-048** | BL-013 | "Add Resolve/Escalate/Assign/Dismiss status transitions for exceptions" | Status transitions are jurisdiction-aware. Resolution completion requirements may differ (US: file amended entry with CBP; EU: respond to customs query via CDS; India: present documents at customs station; Brazil: submit exigencia response via Siscomex). |
| **UB-051** | BL-011 | "Add AD/CVD case numbers and deposit rates for US entries" | AD/CVD becomes a jurisdiction-generic framework. US: USITC case numbers + Commerce Dept deposit rates. India: DECOM (Directorate of Trade Remedies) investigation numbers + provisional duty rates. Framework extensible to EU anti-dumping duties. |
| **UB-052** | BL-010 | "Add ISF status indicator, filing deadline countdown, late filing warnings for US ocean shipments" | ISF becomes "pre-arrival security filing" framework. US: ISF 10+2 (ocean only, 24h before lading). EU: ICS2 ENS (all modes, PLACI for air, DNL instructions). Generic model with jurisdiction-specific filing rules, deadlines, and penalty structures. |
| **UB-057** | BL-033 | "Run Compliance Pre-Check button calling check_compliance (PGA + DPS + UFLPA)" | Compliance check is jurisdiction-dispatched. US: PGA + DPS/SDN + UFLPA. EU: CE/REACH + EU Sanctions + CBAM. India: BIS/FSSAI + DGFT restricted list. Brazil: ANVISA/MAPA pre-shipment LI check. |
| **UB-058** | BL-057 | "Pre-flight gate before 'Submit to CBP' — doc completeness, classification confidence, fee calc, compliance" | Gate runs jurisdiction-specific checks. Label says "Submit to [Authority]" based on jurisdiction. Checklist items are jurisdiction-appropriate (US: bond posted? ISF filed? EU: EORI validated? ENS submitted? India: IEC active? BIS cert obtained?). |
| **UB-073** | BL-048 | "Show ETA and ISF deadline on Shipper shipments" | Filing deadlines are jurisdiction-specific. US: ISF deadline. EU: ICS2 ENS deadline. All: ETA computed from routing engine. Entry type and number displayed in jurisdiction-appropriate format. |

---

## 5. Dependency Graph

### Foundation Layer (Must Complete First)

```
UB-001 (Jurisdiction detection + country picker)
  ├── UB-002 (Polymorphic declaration model)
  │     ├── UB-022 (EU declaration workflow)
  │     ├── UB-023 (UK CDS workflow)
  │     ├── UB-024 (India Bill of Entry)
  │     ├── UB-025 (Brazil DI/DUIMP)
  │     └── UB-026 (China GACC declaration)
  ├── UB-003 (Jurisdiction state machines)
  │     ├── UB-022, UB-023, UB-024, UB-025, UB-026 (all declaration workflows)
  │     └── UB-082 (Post-release/liquidation tracking)
  ├── UB-004 (Frontend jurisdiction-aware rendering)
  │     ├── UB-005 (Authority response screen)
  │     ├── UB-009 (Jurisdiction-aware checklist)
  │     ├── UB-010 (Resolution action buttons)
  │     ├── UB-011 (verify_classification panel)
  │     ├── UB-012 (calculate_entry_fees panel)
  │     └── UB-058 (Entry readiness gate)
  └── UB-006 (Importer identity framework)
        ├── UB-022 (EU needs EORI)
        ├── UB-024 (India needs IEC/GSTIN)
        └── UB-025 (Brazil needs CNPJ/RADAR)
```

### Tariff Engine Dependencies

```
UB-021 (UK tariff separation)
  └── UB-023 (UK CDS workflow)

UB-027 (EU FTA partners)
  └── UB-032 (EU VAT mechanisms — Procedure 42 ties to FTA origin)

UB-030 (India BCD database lookup)
  └── UB-031 (India IGST slab expansion — IGST depends on correct BCD)

UB-055 (Incoterms)
  └── UB-033 (Brazil AFRMM — requires CIF breakdown which requires Incoterms)
```

### Simulation Dependencies

```
UB-018 (HS-code-aware simulation tariff)
  └── UB-019 (CustomsActor reads misclassified flag — more meaningful with real tariffs)

UB-040 (Non-US corridors)
  ├── UB-042 (EU simulation actor — needs EU corridors)
  ├── UB-043 (India simulation actor — needs India corridors)
  ├── UB-044 (Brazil simulation actor — needs Brazil corridors)
  └── UB-098 (China simulation actor — needs China corridors)

UB-041 (Non-US port data)
  ├── UB-022, UB-024, UB-025, UB-026 (all declaration workflows need port codes)
  └── UB-042, UB-043, UB-044, UB-098 (all simulation actors need ports)

UB-053 (Simulation control panel)
  └── Depends on UB-042/043/044/098 existing to select jurisdiction actors
```

### Cross-Cutting Dependencies

```
UB-014 (Regulatory Intel API)
  └── UB-039 (EU CBAM) — CBAM signals belong in regulatory intel

UB-037 (Regulatory agency triggers)
  ├── Depends on UB-001 (jurisdiction detection)
  ├── UB-038 (EU sanctions) — screening is a regulatory requirement
  └── UB-052 (Pre-arrival filing) — ICS2 ENS is a regulatory requirement

UB-007 (Buyer duty from tariff engine)
  ├── UB-072 (Buyer checkout destination) — destination drives tariff calc
  ├── UB-080 (Buyer price math) — real numbers must add up
  └── UB-056 (Buyer shipping cost) — completes cost picture

UB-045 (Agent tool descriptions)
  └── Depends on UB-001 (jurisdiction detection) to provide context

UB-046 (New EU/APAC agent tools)
  └── Depends on UB-006 (EORI), UB-027 (EU FTA), UB-039 (CBAM)
```

### Key Ordering Constraints

1. **UB-001 → UB-004 → ALL frontend jurisdiction work**: Cannot build jurisdiction-aware UI without detection and rendering framework.
2. **UB-002 → ALL declaration workflows**: Cannot file non-US declarations without polymorphic model.
3. **UB-006 → UB-022/024/025**: Cannot file EU/India/Brazil declarations without importer identity.
4. **UB-021 → UB-023**: UK CDS requires UK tariff engine to exist first.
5. **UB-040 + UB-041 → UB-042/043/044/098**: Simulation actors need corridors and ports.
6. **UB-055 (Incoterms) → UB-033 (Brazil AFRMM)**: AFRMM is calculated on freight value, which requires CIF breakdown from Incoterms.
7. **UB-018 (HS-aware tariff in sim) should precede UB-042-044/098**: New jurisdiction simulation actors should use real tariff data from day one.

### Anti-Pattern Warning

Building any of these US-only first would require refactoring:
- BL-003/012 (resolution buttons) without UB-004 (jurisdiction labels) = hardcoded "Submit to CBP" in resolution paths
- BL-009 (doc generate) without UB-009 (jurisdiction checklist) = hardcoded US document list
- BL-010 (ISF) without UB-052 (pre-arrival framework) = US-only ISF that needs rewrite for ICS2
- BL-011 (AD/CVD) without UB-051 (generic framework) = US-only table that needs rewrite for India ADD
- BL-026 (IOR) without UB-006 (multi-jurisdiction identity) = US-only EIN model that needs rewrite for EORI/IEC/CNPJ

The jurisdiction abstraction layer (Sprint 1) is a prerequisite for building these features correctly the first time.

---

## Appendix A: Priority Framework

| Priority | Definition | Item Count |
|---|---|---|
| **P0** | Foundation items that unblock everything + critical placeholders/hardcoded content that destroy credibility | 20 |
| **P1** | Jurisdiction parity (make each jurisdiction functional) + unsurfaced backend capabilities | 36 |
| **P2** | Operational depth (search/sort/filter/pagination) + passive information → actionable workflows | 25 |
| **P3** | Advanced features (post-release tracking, export, special trade regimes) + proactive intelligence | 17 |
| **P4** | Polish (keyboard shortcuts, format fixes, deep jurisdiction features) | 6 |

## Appendix B: Effort Legend

| Size | Definition | Approximate Scope |
|---|---|---|
| **S** | Single file, straightforward change | 1-2 days, <100 lines |
| **M** | 2-4 files, moderate complexity | 3-5 days, 100-500 lines |
| **L** | 5-10 files, significant new logic | 1-2 weeks, 500-1500 lines |
| **XL** | 10+ files, new subsystem | 2-3 weeks, 1500+ lines |

## Appendix C: Source Traceability

Every unified item traces back to its original source(s):

| Range | Source |
|---|---|
| UB-001 through UB-020 | P0 — mix of GL foundation + BL placeholder fixes |
| UB-021 through UB-056 | P1 — GL jurisdiction parity + BL unsurfaced capabilities |
| UB-057 through UB-081 | P2 — BL operational depth items (generalized) + GL multi-currency |
| UB-082 through UB-098 | P3 — BL advanced features + GL advanced jurisdictions |
| UB-099 through UB-104 | P4 — BL polish + GL deep features |

### Items NOT carried forward (absorbed into other items)

| Original ID | Absorbed Into | Reason |
|---|---|---|
| BL-034 (Sidebar crude estimates) | UB-012 | Same issue as BL-005 — 5% fallback in different location |
| BL-018 (Risk flag resolution in EntryDetail) | UB-010 | Same pattern as BL-003 in different screen location |
| BL-054 (Entry number format) | UB-002 | Entry number format is part of polymorphic declaration model |
| BL-030 (Missing CF-7501 fields) | UB-002 | CF-7501 field expansion is part of declaration model enrichment |
| BL-060 (Sort on Broker queue) | UB-049 | Search and sort are the same work item |
| BL-069 (Signal CRUD) | UB-014 | Admin CRUD is inseparable from API-backed signals |
| GL-028 through GL-031 (individual regulatory triggers) | UB-037 | Consolidated into one jurisdiction-aware regulatory trigger framework |
| S-4 (Wire ComplianceDashboard) | UB-015 | Identical to BL-008 |
| S-5 (Cargo exceptions) | UB-064 | Identical to BL-038 |
| S-6 (Enable run_real_engines) | UB-018 | Subsumed by HS-code-aware tariff work |
| S-7 (Two tariff engines inconsistent) | UB-018 | Resolved by unifying tariff calculation path |
