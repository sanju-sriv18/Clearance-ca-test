# China & APAC Customs Operations Gap Analysis

> **Goal**: Make China and APAC import/export clearance a first-class experience in Clearance Vibe, equal to the current US CBP support.
>
> **Methodology**: Codebase audit of existing engines, broker surface, simulation actors, and reference data compared against real-world regulatory requirements for CN, JP, KR, and ASEAN markets.

---

## Executive Summary

The platform has strong US CBP clearance as its anchor — full lifecycle from ACAS/ISF filing through CF-28/CF-29 response workflows, examination scheduling, broker approval queues, and a comprehensive tariff engine. China has the most existing engine support (MFN + retaliatory + consumption tax + VAT in `cn.py`), but the **entire broker experience, document workflow, regulatory filing, and simulation infrastructure is US-only**. No APAC customs authority filing, document template, or regulatory response workflow exists in the frontend or broker API.

### What Exists Today (China/APAC)

| Layer | What's There | What's Missing |
|-------|-------------|----------------|
| **Tariff Engine** | CN regime: MFN duty, 125% retaliatory, consumption tax grossup, 13% VAT | Cross-border e-commerce tax (行邮税), RCEP/FTA preferential rates, provisional/interim rates, processing trade exemptions, bonded zone policies |
| **Trade Corridors** | CN→US (ocean, 25% weight) | CN as destination, intra-APAC (CN↔JP, CN↔KR, CN↔VN, CN↔SG), ASEAN↔US, JP→US exists |
| **Simulation** | CN as origin country for US-bound shipments; CNY seasonal events; typhoon risk for Shanghai/Shenzhen | No CN-import simulation actors, no GACC filing simulation, no CIQ inspection events |
| **Port Data** | Shanghai CN, Shenzhen CN, Hong Kong HK, Busan KR, Yokohama JP, Singapore SG, Ho Chi Minh VN, Laem Chabang TH, Kaohsiung TW — climate + congestion data | Port codes for APAC customs (no CN customs district codes), no NACCS/UNI-PASS/TradeNet port mappings |
| **Territory Filings** | `TERRITORY_FILINGS["CN"]` has `customs_declaration` + `CIQ_inspection` (export), `customs_declaration` + `CIQ_inspection` (import) | Stub-level only — no actual GACC Single Window integration logic, no pre-arrival manifest filing |
| **Territory Risks** | `TERRITORY_RISKS["CN"]` = `export_control`; `SG` = `strategic_goods_control`; `KR` = `sanctions_screening` | No GACC enterprise classification risk system, no CN negative list screening, no dual-use controls |
| **Broker Surface** | 100% US-oriented: CBP Responses, CF-28/CF-29 workflows, ACE entry filing, US port codes, MPF/HMF fees | Zero APAC customs authority UI — no GACC response handling, no CIQ inspection workflow, no NACCS/UNI-PASS integration |
| **Reference Data** | AD/CVD cases reference CN origin; flagged shippers include CN entities (Xinjiang Zhongtai, Changji Esquel, Skyrizon) | No CN-side restricted entity lists (unreliable entity list), no APAC sanctions lists, no CN export control item catalog |

---

## CHINA (Primary — World's Largest Trading Nation by Volume)

### 1. GACC & Single Window (国际贸易单一窗口)

**US Benchmark**: ACE (Automated Commercial Environment) — single portal for all US import/export filings. Platform files entries via ACE, simulates ACE response codes (`ACE_ENTRY_RESPONSES`), and handles holds (`CBP_HOLD_CODES`).

**China Equivalent**: China International Trade Single Window (国际贸易单一窗口, `singlewindow.cn`) — unified platform for customs declaration (报关单), inspection/quarantine, tax, forex, and 30+ agencies. Operated by GACC (General Administration of Customs of China, 海关总署).

**What Exists**: `TERRITORY_FILINGS["CN"]["import"]` lists `customs_declaration` and `CIQ_inspection` — stub-level only.

**What's Needed**:

| Component | Priority | Description |
|-----------|----------|-------------|
| **GACC Declaration Model** | P0 | Data model for Chinese customs declaration form (报关单) — 50+ fields including: declaration number (报关单号), customs district code (关区代码), trade mode (贸易方式代码), supervision mode (监管方式), transport mode, transaction terms (成交方式 — FOB/CIF/C&F), and complete goods description in Chinese |
| **Pre-Arrival Manifest Filing** | P0 | China requires manifest data 24h before vessel arrival / before loading for air. Analogous to US ISF/ACAS but different format and timing |
| **Single Window Integration Simulation** | P1 | Simulate GACC response codes (similar to `ACE_ENTRY_RESPONSES`): declaration accepted, sent back for correction (退回), inspection ordered (查验), release (放行), tax payment notice (税款缴纳通知) |
| **CIQ Integration** | P1 | China Inspection & Quarantine (now merged into GACC) — inspection/quarantine declaration for food, animal/plant, chemicals, cosmetics. Analogous to US PGA agencies but different workflow |
| **Electronic Payment (电子支付)** | P2 | Duty/tax payment via bank guarantee or e-payment through Single Window. Different from US periodic monthly statement |
| **Declaration Correction (报关单修改/撤销)** | P2 | Post-filing amendment process — more bureaucratic than US post-summary correction |

### 2. Required Identifiers

**US Benchmark**: EIN/IOR number, customs broker license, power of attorney, CBP bond. All modeled in `Broker` model (license_number, license_district).

**China Equivalent**:

| Identifier | Chinese Name | US Equivalent | Status |
|-----------|-------------|---------------|--------|
| **USCC** (Unified Social Credit Code) | 统一社会信用代码 | EIN | Not modeled |
| **Customs Registration Certificate** | 海关注册登记证 | Customs importer number | Not modeled |
| **CIQ Organization Code** | 检验检疫组织机构代码 | FDA establishment registration | Not modeled |
| **China AEO Classification** | AEO认证 (AA/A/B/C/D) | C-TPAT | Not modeled |
| **Customs Broker License (报关员证)** | 报关企业注册登记证 | CHB license | Not modeled |
| **Foreign Trade Operator Registration** | 对外贸易经营者备案 | No direct equivalent | Not modeled |

**What's Needed**:
- **P0**: Add jurisdiction-aware identifier schema to the `Broker` and company models — each jurisdiction has different required IDs
- **P1**: AEO classification field (AA/A/B/C/D) — affects channel assignment (green/yellow/red), exam rates, and clearance speed
- **P2**: Validation rules per jurisdiction (USCC format = 18 chars, alphanumeric; CHB license format varies by country)

### 3. Document Types

**US Benchmark**: Comprehensive document management — Commercial Invoice, Packing List, Bill of Lading, Certificate of Origin, Entry Summary (7501). Full checklist computation in `_compute_checklist_state()` with upload, validation, and override workflows.

**China Required Documents**:

| Document | Chinese Name | US Equivalent | Exists? |
|----------|-------------|---------------|---------|
| Customs Declaration Form | 报关单 | Entry Summary 7501 | No |
| Inspection/Quarantine Application | 报检单 | PGA filing | No |
| CCC Certificate | 中国强制性产品认证 | UL listing / FCC | No |
| CFDA/NMPA Registration | 药品/医疗器械注册证 | FDA 510(k) | No |
| Phytosanitary Certificate | 植物检疫证书 | APHIS phyto cert | No (only US APHIS modeled) |
| Dangerous Goods Declaration | 危险品申报 | DOT hazmat | No |
| Import License (进口许可证) | Various by category | Import permit | No |
| Fumigation/Heat Treatment Certificate | 熏蒸/热处理证书 | ISPM-15 | Partial (APHIS requirements) |
| Pre-shipment Inspection Certificate | 装船前检验证书 | Pre-shipment inspection | No |

**What's Needed**:
- **P0**: Jurisdiction-aware document checklist — the `_compute_checklist_state()` function needs to branch based on destination country to show CN-specific documents
- **P0**: Document type registry per jurisdiction (replacing hard-coded US-only `CHECKLIST_LABELS`)
- **P1**: CCC/CFDA/NMPA certificate validation workflows — these are product-type gated (electronics, medical, food)
- **P2**: Chinese-language document support (OCR, validation rules for Chinese commercial invoices)

### 4. Regulatory Frameworks

**US Benchmark**: PGA agencies modeled in `PGA_TRIGGERS` (FDA, EPA, CPSC, USDA, NHTSA). Simulation generates PGA holds. Broker UI shows PGA documentation status.

**China Regulatory Agencies**:

| Agency | Chinese Name | Jurisdiction | US Equivalent | Exists? |
|--------|-------------|-------------|---------------|---------|
| **GACC** | 海关总署 | All imports/exports | CBP | Stub only |
| **SAMR** (CCC) | 国家市场监管总局 | Electronics, auto, toys, medical | FCC + CPSC + NHTSA | No |
| **NMPA** | 国家药品监督管理局 | Pharma, medical devices, cosmetics | FDA | No |
| **MARA** | 农业农村部 | Agriculture, seeds, fertilizer | USDA/APHIS | No |
| **MIIT** | 工业和信息化部 | Telecom equipment, radio | FCC | No |
| **MEE** | 生态环境部 | Chemicals, waste, ODS | EPA | No |
| **MOFCOM** | 商务部 | Trade licenses, quotas, anti-dumping | ITC + Commerce Dept | No |
| **SFA** (Forestry) | 国家林草局 | CITES, timber, wildlife | USFWS | No |

**What's Needed**:
- **P0**: China PGA equivalent mapping — `CN_REGULATORY_TRIGGERS` keyed by product category → agency → required documents/certifications
- **P0**: Negative list screening — China maintains import/export negative lists (禁止/限制类目录) that gate whether goods can enter at all
- **P1**: Unreliable Entity List (不可靠实体清单) — China's equivalent of US Entity List, needs screening in E3 compliance engine
- **P1**: Export Control Law items catalog (出口管制法 — effective 2020) — dual-use, military, nuclear, missile tech
- **P2**: CITES/wildlife permit requirements for CN

### 5. Tax Regime

**US Benchmark**: Full tariff computation — MFN duty, Section 301, Section 232, IEEPA reciprocal, AD/CVD, MPF, HMF. All computed in `us.py`.

**China Tax Regime — What Exists** (`cn.py`):
- MFN Duty (golden-path rates at 6-digit HS): 17 products covered
- Retaliatory Tariff: 125% on US-origin goods
- Consumption Tax: grossup formula for 4 HS chapters (auto, cosmetics, wine, tobacco)
- VAT: 13% standard rate

**China Tax Regime — What's Missing**:

| Tax Component | Description | Priority |
|--------------|-------------|----------|
| **Reduced VAT rates** | 9% for agriculture/transport/utilities, 6% for services/financial — current engine only has 13% | P1 |
| **Cross-border e-commerce tax (行邮税)** | Completely different rate structure for 9610/1210/9710/9810 trade modes — personal postal articles tax rates (15%/25%/50%) replacing duty+VAT for parcels under RMB 5,000 | P1 |
| **Provisional/Interim tariff rates (暂定税率)** | China frequently sets provisional rates lower than MFN for specific goods — these override MFN and change semi-annually | P1 |
| **RCEP preferential rates** | Phased tariff reduction schedules for 15-country RCEP — year-specific rates based on product schedule | P1 |
| **FTA preferential rates** | ACFTA (ASEAN), China-Korea, China-Australia, China-NZ, China-Switzerland, China-Pakistan, China-Peru rates | P1 |
| **Processing trade duty exemption** | 来料加工 (supplied-material processing) = full duty exemption; 进料加工 (purchased-material processing) = duty deferral with bond — massive volume | P2 |
| **Bonded zone preferential policies** | Goods entering Shanghai FTZ, Hainan FTP, comprehensive bonded zones — different duty treatment | P2 |
| **Consumption tax expanded categories** | Only 4 chapters covered; should include: jewelry (7113/7114), watches (9101), yachts (8903), batteries/coatings, fireworks, golf equipment | P2 |
| **Anti-dumping/countervailing duties** | China imposes its own AD/CVD measures on imports (e.g., Australian wine, US chicken, Japanese chemicals) | P2 |
| **Retaliatory tariff expansion** | Only US origin covered; CN also has retaliatory measures against other countries | P3 |

### 6. Trade Agreements

**US Benchmark**: `FTA_PARTNERS_US` dict in `documents.py` covers USMCA, KORUS, US-Australia, CAFTA-DR, etc. Used in checklist to determine Certificate of Origin requirement.

**China FTA Network** (none modeled):

| Agreement | Partners | Coverage | Priority |
|-----------|----------|----------|----------|
| **RCEP** | 15 countries (ASEAN-10 + CN, JP, KR, AU, NZ) | Largest FTA in history — 30% of global GDP | P0 |
| **ACFTA** (China-ASEAN FTA) | 10 ASEAN members | Form E certificate of origin | P1 |
| **China-Korea FTA** | South Korea | Extensive tariff elimination schedule | P1 |
| **China-Australia FTA** (ChAFTA) | Australia | Wine, dairy, beef, resources | P1 |
| **China-New Zealand FTA** | New Zealand | Dairy, meat, wood — fully in effect | P2 |
| **China-Switzerland FTA** | Switzerland | Watches, pharma, machinery | P2 |
| **China-Pakistan FTA** | Pakistan | Phase II expanded coverage | P2 |
| **China-Peru FTA** | Peru | Mining, agriculture | P3 |
| **ECFA** (cross-strait) | Taiwan | Controversial — political sensitivity | P3 |
| **CPTPP** (applied to join) | 11 countries | China applied 2021 — accession ongoing | P3 |

**What's Needed**:
- **P0**: `FTA_PARTNERS_CN` registry with rules of origin per agreement (RCEP regional value content ≥40%, ACFTA change in tariff classification)
- **P0**: Certificate of Origin types — different form per FTA (RCEP Form RCEP, ACFTA Form E, China-Korea Form FTA, etc.)
- **P1**: RCEP cumulation rules — allows content from all 15 member countries to count toward origin
- **P1**: FTA tariff reduction schedules (year-specific rates — RCEP has 20-year phase-in periods)

### 7. Customs Procedures (Trade Modes)

**US Benchmark**: Entry types 01 (consumption), 02 (temporary import), etc. Modeled as `entry_type` field. Simulation generates formal entries.

**China Trade Modes** (监管方式) — vastly more complex than US:

| Code | Trade Mode | Chinese Name | Volume | Priority |
|------|-----------|-------------|--------|----------|
| 0110 | General Trade | 一般贸易 | ~50% | P0 |
| 0214 | Supplied-material Processing | 来料加工 | ~15% combined | P1 |
| 0615 | Purchased-material Processing | 进料加工 | ~15% combined | P1 |
| 1210 | Bonded E-Commerce (B2B2C) | 保税备货 | Growing rapidly | P1 |
| 9610 | Cross-border E-Commerce (B2C direct) | 跨境电商零售 | Growing rapidly | P1 |
| 9710 | Cross-border E-Commerce B2B export | B2B海外仓 | New model | P2 |
| 9810 | Cross-border E-Commerce B2B export via overseas warehouse | 跨境电商出口海外仓 | New model | P2 |
| 1300 | Bonded Warehouse | 保税仓库 | ~5% | P2 |
| 2025 | Bonded Logistics | 保税物流 | ~3% | P2 |
| 0500 | Temporary Import | 暂时进出口 | ~2% | P2 |
| 2700 | Free Trade Zone | 自贸区 | Growing | P3 |

**What's Needed**:
- **P0**: Trade mode field on entry/shipment model (currently entry_type is US-specific "01"/"02")
- **P0**: Trade mode determines: duty treatment, document requirements, bond/guarantee requirements, supervision period
- **P1**: Processing trade manual system (加工贸易手册) — tracks duty-free raw materials imported → finished goods exported
- **P1**: Cross-border e-commerce specific workflows (personal ID verification, RMB 5,000 per transaction / RMB 26,000 annual limits)

### 8. Risk Classification (Enterprise Credit Management)

**US Benchmark**: C-TPAT (Customs-Trade Partnership Against Terrorism) — mentioned in knowledge base. Affects exam rates.

**China Enterprise Credit System** (企业信用管理):

| Rating | Channel | Exam Rate | Benefits | Priority |
|--------|---------|-----------|----------|----------|
| **AA (Advanced AEO)** | Green | <1% | MRA with 46 countries, priority clearance, lower bond, assigned customs officer | P1 |
| **A (General AEO)** | Green | ~5% | Priority processing, reduced documentation, easier amendment | P1 |
| **B (General)** | Yellow | ~10% | Standard processing | P0 |
| **C (Watchlist)** | Red | ~30% | Enhanced scrutiny, higher bond, more frequent audits | P1 |
| **D (Discredited)** | Red | ~50% | Severe restrictions, may be denied clearance | P1 |

**What's Needed**:
- **P1**: Enterprise credit rating field on company model — affects clearance simulation speed, exam probability, and channel assignment
- **P1**: AEO Mutual Recognition — China has MRA agreements with EU, Korea, Singapore, Japan, US (since 2024), and others. AA-rated companies get green-channel treatment in partner countries
- **P2**: Credit scoring simulation — periodic reviews can upgrade/downgrade companies

### 9. Export Controls

**US Benchmark**: E3 Compliance engine screens against US restricted party lists (SDN, Entity List, UFLPA). `FLAGGED_SHIPPER_NAMES` includes Xinjiang entities. `TERRITORY_RISKS["US"]` includes `sanctions_screening`, `ad_cvd_review`.

**China Export Control Regime**:

| Control | Description | US Equivalent | Priority |
|---------|-------------|---------------|----------|
| **Export Control Law (2020)** | 出口管制法 — comprehensive framework for controlled items | EAR/ITAR | P1 |
| **Dual-Use Items** | 两用物项 — technology, equipment, materials | Commerce Control List (CCL) | P1 |
| **Rare Earth Restrictions** | Strategic mineral export quotas and licensing | No direct equivalent | P1 |
| **Technology Export Restrictions** | 技术出口限制目录 — AI, biotech, crypto | Deemed export rules | P2 |
| **Unreliable Entity List** | 不可靠实体清单 — China's retaliatory list | Entity List | P1 |
| **Anti-Foreign Sanctions Law** | 反外国制裁法 — blocking statute | No direct equivalent | P2 |
| **Data Export Restrictions** | 数据出境安全评估 — cross-border data transfer | No direct equivalent | P3 |

**What's Needed**:
- **P1**: Chinese restricted party list screening in E3 engine — unreliable entity list, export control item catalog
- **P1**: Dual-use item classification — check HS codes against China's controlled items list
- **P2**: Rare earth / critical mineral export license requirements
- **P2**: Technology export restriction screening (affects software, AI models, semiconductor equipment)

### 10. China-Specific Operational Challenges

| Challenge | Description | Platform Impact | Priority |
|-----------|-------------|----------------|----------|
| **Frequent regulatory changes** | GACC issues circulars (海关公告) continuously — rates, procedures, prohibited goods change without long lead times | Need regulatory update pipeline | P1 |
| **Dual invoicing** | Customs may suspect under-invoicing; separate domestic/export pricing is scrutinized | Valuation validation rules | P2 |
| **Transfer pricing scrutiny** | Related-party transactions face heightened review under GACC TP rules | Risk flag for related-party entries | P2 |
| **RMB settlement requirements** | Some transactions must settle in RMB; forex conversion regulations | Multi-currency support beyond USD | P1 |
| **Golden Shield integration** | China's internet restrictions may affect cloud-based filing tools | On-premise / CN-hosted deployment option | P3 |
| **Hainan Free Trade Port** | Special regime (海南自由贸易港) with unique duty-free policies effective 2025+ | Separate trade mode handling | P3 |
| **Processing trade complexity** | Manual system (手册) tracking raw material imports against finished good exports — bank guarantees, reconciliation | Complex lifecycle management | P2 |

---

## JAPAN

### 1. NACCS (Nippon Automated Cargo and Port Consolidated System)

**US Benchmark**: ACE for electronic filing. Full simulation of ACE entry responses and holds.

**Japan Equivalent**: NACCS — Japan's 24/7 electronic customs clearance system covering all import/export declarations, permits, payments, and port/airport operations. Operated by NACCS Center (independent administrative agency).

**What Exists**: JP corridors (JP→US via Yokohama), JP seasonal events (Golden Week), JP port climate data. No NACCS filing simulation.

**What's Needed**:

| Component | Priority | Description |
|-----------|----------|-------------|
| **NACCS Declaration Model** | P1 | Import declaration (輸入申告書) data model — different field structure than US 7501 or CN 報関単. Includes: import declaration number, customs office code, declarant code, importer code, tax classification |
| **NACCS Response Simulation** | P1 | Declaration acceptance, examination notice (検査通知), release notification (許可通知), duty payment notice |
| **Pre-arrival Filing** | P2 | Preliminary declaration (予備審査) — can be filed before vessel arrival for faster clearance |
| **AEO Benefits Simulation** | P2 | Japan AEO importers can self-assess, reduced inspection, simplified declaration |

### 2. Identifiers

| Identifier | Japanese Name | US Equivalent | Priority |
|-----------|-------------|---------------|----------|
| **Customs Broker License** | 通関業者許可 | CHB license | P1 |
| **Japan AEO Certification** | AEO認定 (importer/exporter/broker/warehouse/carrier) | C-TPAT | P1 |
| **Japan Customs Registration** | 輸出入者コード | Importer number | P1 |
| **NACCS User ID** | NACCS利用者コード | ACE portal user | P2 |

### 3. Tax Regime

**What Exists**: No JP tariff regime in E2 engine.

**What's Needed**:

| Tax Component | Rate | Priority |
|--------------|------|----------|
| **Customs Duty** | Varies by HS code — Japan's tariff schedule | P1 |
| **Consumption Tax (消費税)** | 10% standard, 8% reduced rate (food/beverages excl. alcohol) | P1 |
| **Anti-Dumping Duties** | Japan has AD measures on select products | P2 |
| **Countervailing Duties** | Rare but exists | P3 |

**What's Needed**:
- **P1**: `jp.py` tariff regime with MFN rates + consumption tax
- **P1**: Reduced consumption tax rate (8%) for food category — similar to EU reduced VAT
- **P2**: EPA/FTA preferential rates (Japan-EU EPA, CPTPP, RCEP)

### 4. Regulatory Agencies

| Agency | Japanese Name | Jurisdiction | US Equivalent | Priority |
|--------|-------------|-------------|---------------|----------|
| **MAFF** | 農林水産省 | Agriculture, food import inspection, plant/animal quarantine | USDA + APHIS | P1 |
| **MHLW** | 厚生労働省 | Pharmaceuticals, medical devices, food safety | FDA | P1 |
| **METI** | 経済産業省 | Trade control, dual-use (via CISTEC), strategic goods | Commerce/BIS | P1 |
| **MoE** | 環境省 | Chemicals (Chemical Substances Control Law), waste | EPA | P2 |
| **PSE/PSC** | 電気用品安全法/消費生活用品安全法 | Electrical/consumer product safety marks | UL/FCC/CPSC | P2 |

**What's Needed**:
- **P1**: `JP_REGULATORY_TRIGGERS` mapping product categories to regulatory agencies
- **P1**: Food sanitation law (食品衛生法) screening — Japan has strict residue and additive standards
- **P2**: CISTEC (Center for Information on Security Trade Control) strategic goods screening

### 5. Trade Agreements

| Agreement | Key Features | Priority |
|-----------|-------------|----------|
| **RCEP** | 15-country mega-regional — shared with China | P1 |
| **Japan-EU EPA** | Zero tariff on most industrial goods; wine, cheese, auto parts | P1 |
| **CPTPP** | 11-country comprehensive agreement (originally TPP) | P1 |
| **Japan-US (Phase 1)** | Limited — digital trade + some agriculture | P2 |
| **Japan-UK CEPA** | Post-Brexit replacement for Japan-EU EPA | P2 |
| **Japan-ASEAN** | AJCEP — comprehensive economic partnership | P2 |

---

## SOUTH KOREA

### 1. UNI-PASS (Korea Customs Electronic Clearance System)

**US Benchmark**: ACE electronic filing.

**Korea Equivalent**: UNI-PASS — Korea Customs Service (KCS) electronic clearance system. One of the most advanced customs IT systems globally — exported to 20+ countries. Average clearance time: 1.5 hours for AEO importers.

**What Exists**: KR corridors (KR→US via Busan), Busan port data, AD/CVD case for KR steel pipe. No KR-as-destination support.

**What's Needed**:

| Component | Priority | Description |
|-----------|----------|-------------|
| **UNI-PASS Declaration Model** | P1 | Import declaration data model — different from US/CN/JP formats. Includes: declaration number, customs office, HS classification, C&F/CIF selection, origin country verification |
| **UNI-PASS Response Simulation** | P2 | Declaration acceptance (수리), examination notice, release, duty notice |
| **AEO-MRA Integration** | P2 | Korea has mutual recognition with US, EU, JP, CN, SG, and others — AEO status affects clearance speed |
| **Pre-Import Declaration** | P2 | Can be filed up to 5 days before vessel arrival |

### 2. Tax Regime

**What's Needed**:

| Tax Component | Rate | Priority |
|--------------|------|----------|
| **Customs Duty** | Varies by HS code | P1 |
| **VAT** | 10% flat rate | P1 |
| **Individual Consumption Tax** | Luxury goods: 5-20% (jewelry, furs, automobiles) | P2 |
| **Education Tax** | 30% of individual consumption tax | P2 |
| **Special Excise** | Petroleum, tobacco, alcohol | P3 |

**What's Needed**:
- **P1**: `kr.py` tariff regime with MFN + VAT
- **P2**: KORUS FTA preferential rates, RCEP rates
- **P2**: Individual consumption tax for luxury categories

### 3. Regulatory Agencies

| Agency | Korean Name | Jurisdiction | Priority |
|--------|-----------|-------------|----------|
| **MFDS** | 식품의약품안전처 | Food, drugs, medical devices, cosmetics | P1 |
| **KCC** | 방송통신위원회 | Radio/telecom equipment certification (KC mark) | P2 |
| **KATS** | 국가기술표준원 | Standards, product safety (KS mark) | P2 |
| **NIFS** | 국립수산물품질관리원 | Fishery products inspection | P3 |

### 4. Trade Agreements

| Agreement | Priority |
|-----------|----------|
| **RCEP** | P1 |
| **Korea-EU FTA** | P1 |
| **KORUS FTA** (Korea-US) | Already partially modeled as FTA partner; needs KR-as-destination rates | P1 |
| **CPTPP** (acceding) | P2 |
| **Korea-UK FTA** | P2 |
| **Korea-ASEAN** | P2 |

---

## ASEAN (Association of Southeast Asian Nations)

### 1. ASEAN Single Window (ASW)

**US Benchmark**: ACE.

**ASEAN Equivalent**: ASW is an interoperability initiative connecting National Single Windows of all 10 ASEAN members. Implementation maturity varies dramatically:

| Country | System | Maturity | Avg Clearance | Priority |
|---------|--------|----------|---------------|----------|
| **Singapore** | TradeNet | World-class (4 min avg) | 4 minutes | P1 |
| **Vietnam** | VNACCS/VCIS | Good (Japanese-built) | 1-3 hours | P1 |
| **Thailand** | e-Customs | Good | 2-4 hours | P2 |
| **Indonesia** | INSW | Moderate (complex licensing) | 4-48 hours | P2 |
| **Malaysia** | uCustoms | Good | 2-6 hours | P2 |
| **Philippines** | e2m | Developing | 3-7 days | P3 |
| **Myanmar** | MACCS | Early | Variable | P4 |
| **Cambodia** | ASYCUDA World | Developing | Variable | P4 |
| **Laos** | ASYCUDA World | Early | Variable | P4 |
| **Brunei** | BDNSW | Limited | Variable | P4 |

### 2. Key Markets — Detailed

#### Singapore (SG)
**What Exists**: Singapore port data (1.0 base dwell days, 0.8 congestion factor — world's best), `TERRITORY_FILINGS["SG"]` with `TradeNet_declaration`, `TERRITORY_RISKS["SG"]` with `strategic_goods_control`.

**What's Needed**:

| Component | Priority | Description |
|-----------|----------|-------------|
| **TradeNet Declaration Model** | P1 | Singapore's electronic trade declaration — permit types: IN (import), OUT (export), TF (transhipment) |
| **GST Computation** | P1 | 9% GST (increased from 8% in 2024) on import value + duty |
| **Strategic Goods Control** | P2 | Singapore Strategic Goods (Control) Act — dual-use items require SG permits |
| **SG FTZ Processing** | P2 | Goods can be stored in FTZ (Jurong, Changi) without duty payment |

#### Vietnam (VN)
**What Exists**: VN corridor (VN→US, 8% weight), Ho Chi Minh port data.

**What's Needed**:

| Component | Priority | Description |
|-----------|----------|-------------|
| **VNACCS/VCIS Model** | P2 | Vietnam's customs system (built by JICA/Japan). Green/Yellow/Red channel classification |
| **VN Tax Regime** | P2 | Import duty + Special consumption tax (luxury) + VAT (10% standard, 5% reduced) + Environmental tax |
| **VN Regulatory** | P2 | MARD (agriculture), MOH (health), MOST (standards) — SPS measures |
| **FTA Benefits** | P2 | CPTPP (massive benefit for VN), EVFTA (EU-Vietnam), RCEP, UKVFTA |

#### Thailand (TH)
**What Exists**: TH corridor (TH→US, 3% weight), Laem Chabang port data.

**What's Needed**:

| Component | Priority | Description |
|-----------|----------|-------------|
| **e-Customs Model** | P2 | Thailand Customs Department e-filing |
| **TH Tax Regime** | P2 | Import duty + VAT (7%) + excise tax |
| **BOI Privileges** | P3 | Board of Investment incentives — duty exemptions for promoted industries |
| **EEC Incentives** | P3 | Eastern Economic Corridor special investment zone |

#### Indonesia (ID)
**What Exists**: No ID-specific data in corridors or reference data.

**What's Needed**:

| Component | Priority | Description |
|-----------|----------|-------------|
| **INSW Model** | P2 | Indonesia National Single Window — most complex ASEAN customs due to licensing requirements |
| **Import Licensing** | P2 | API-U (general importer) vs API-P (producer importer) — different permitted goods lists |
| **ID Tax Regime** | P2 | Import duty + VAT (11%) + Income Tax Article 22 (advance payment on import) + luxury goods sales tax (PPnBM) |
| **Pre-shipment Inspection** | P2 | Surveyor verification for certain goods — Sucofindo, SGS |

#### Malaysia (MY)
**What Exists**: No MY-specific data.

**What's Needed**:

| Component | Priority | Description |
|-----------|----------|-------------|
| **uCustoms Model** | P3 | Malaysia Royal Customs electronic system |
| **MY Tax Regime** | P3 | Import duty + Sales tax (5/10%) + Service tax (8%) — Malaysia repealed GST in 2018 |
| **MITI Licensing** | P3 | Ministry of International Trade and Industry import/export permits |

### 3. ATIGA (ASEAN Trade in Goods Agreement)

**What's Needed**:
- **P1**: Form D Certificate of Origin — required for ASEAN preferential rates
- **P1**: ATIGA rules of origin (40% Regional Value Content or Change in Tariff Classification)
- **P1**: RCEP cumulation across ASEAN+5 — allows content from all 15 RCEP countries

### 4. RCEP Impact (Cross-cutting)

RCEP is the single most important trade agreement for APAC operations:

| Feature | Description | Priority |
|---------|-------------|----------|
| **Cumulation** | Content from any RCEP country counts toward origin — enables multi-country supply chains | P0 |
| **Simplified Rules of Origin** | Product-specific rules harmonized across 15 countries | P1 |
| **Single Certificate of Origin** | RCEP CoO format replaces bilateral FTA forms for intra-RCEP trade | P1 |
| **Tariff Schedules** | Country-specific reduction schedules (immediate/5yr/10yr/15yr/20yr) | P1 |
| **Self-Certification** | Approved exporters can self-certify origin — no need for chamber of commerce | P2 |

---

## Cross-Cutting Platform Gaps (APAC-Wide)

### Backend Architecture

| Gap | Description | Priority |
|-----|-------------|----------|
| **Jurisdiction-aware broker model** | `Broker` model assumes US CHB license structure. Need jurisdiction-specific license fields, regulatory registrations, and operating authorities | P0 |
| **Multi-jurisdiction entry filing** | `EntryFiling` model is US-centric (entry_type "01", CBP response fields). Need generic filing model that dispatches to jurisdiction-specific schemas | P0 |
| **Customs authority response framework** | `cbp_response` JSONB field on EntryFiling is CBP-specific. Need generic customs authority response model supporting GACC circulars, NACCS notices, UNI-PASS decisions, etc. | P0 |
| **Multi-currency tariff computation** | Current engine computes in USD only. CN duties are in RMB, JP in JPY, KR in KRW. Need exchange rate integration and multi-currency landed cost | P1 |
| **Tariff regime expansion** | No JP or KR regimes in E2 engine. CN regime needs FTA rates, provisional rates, cross-border e-commerce rates | P1 |
| **APAC corridor expansion** | All corridors are X→US. Need US→CN, US→JP, US→KR, intra-APAC corridors | P1 |
| **APAC regulatory triggers** | `PGA_TRIGGERS` is US-only. Need equivalent for each APAC jurisdiction | P1 |
| **Lifecycle events for APAC** | `AIR_LIFECYCLE_EVENTS`, `OCEAN_LIFECYCLE_EVENTS`, `GROUND_LIFECYCLE_EVENTS` reference ACE/CBP filing. Need jurisdiction-specific lifecycle events (e.g., GACC declaration filed, NACCS permit granted) | P1 |
| **Time zone-aware processing** | US-only = Eastern time focus. APAC operations span UTC+5:30 to UTC+12, affecting business hours, filing deadlines, and payment windows | P2 |

### Frontend Architecture

| Gap | Description | Priority |
|-----|-------------|----------|
| **US-hardcoded navigation** | `BrokerNav.tsx` has "CBP Responses" as a nav item — needs to be "Customs Authority Responses" with jurisdiction-specific sub-views | P0 |
| **US-hardcoded entry detail** | `EntryDetail.tsx` says "Submit to CBP", "Awaiting CBP response", "CF-28 Request", "CF-29 Notice" — all US-specific terminology | P0 |
| **US-hardcoded dashboard** | `BrokerDashboard.tsx` has "CBP Response" stat tile — needs jurisdiction-neutral or multi-jurisdiction display | P0 |
| **Checklist US-centricity** | Checklist labels (`CHECKLIST_LABELS`) and computation are US-specific (FTA partners check uses `FTA_PARTNERS_US`) | P0 |
| **Port code display** | `US_PORT_CODES` mapping used in broker UI. Need equivalent for CN customs districts, JP customs offices, KR customs offices | P1 |
| **Multi-language support** | Chinese customs documents, Japanese NACCS forms, Korean UNI-PASS declarations need CJK character support in document viewer | P2 |
| **Currency display** | All values shown in USD. Need multi-currency display with conversion | P1 |

### Simulation Infrastructure

| Gap | Description | Priority |
|-----|-------------|----------|
| **APAC-destination corridors** | Need X→CN, X→JP, X→KR, X→SG, X→VN corridors for import simulation | P1 |
| **APAC customs actors** | Current `CustomsActor` simulates CBP behavior. Need jurisdiction-specific customs actors (GACC actor, NACCS actor) with different hold rates, exam types, and response workflows | P1 |
| **APAC seasonal events** | Have CNY and Golden Week. Need: Chuseok (KR), Tet (VN), Songkran (TH), Ramadan (ID/MY), GACC system maintenance windows, end-of-fiscal-year (JP March, CN December) | P2 |
| **APAC carrier services** | Carriers like SF Express, YTO, ZTO (CN domestic/regional) not represented. Also missing: Nippon Express, Sagawa, Yamato (JP), CJ Logistics, Hanjin (KR) | P2 |
| **APAC-specific disruption events** | Need: Golden Shield/Great Firewall outages affecting cloud filing, GACC system maintenance (happens frequently), CN customs strike equivalent (none — but local slowdowns), COVID-like border closures | P3 |

---

## Implementation Roadmap

### Phase 1: Foundation (P0 — Architectural Changes)
1. **Jurisdiction-aware data models** — Generalize `EntryFiling`, `Broker`, `ChecklistItem` to support multi-jurisdiction
2. **Generic customs authority response framework** — Replace CBP-specific `cbp_response` with jurisdiction-dispatched response handling
3. **Jurisdiction-aware broker UI** — Parameterize all US-specific terminology ("CBP" → "Customs Authority", "CF-28" → jurisdiction-specific form codes)
4. **Jurisdiction-aware checklist** — Branch `_compute_checklist_state()` by destination country with jurisdiction-specific document requirements and FTA partner sets

### Phase 2: China First-Class (P1)
1. **CN tariff regime expansion** — Add VAT tiers (9%/6%), provisional rates, RCEP preferential rates
2. **GACC declaration model + simulation** — CN customs declaration form, GACC response codes, CIQ inspection workflow
3. **CN document types** — CCC, NMPA, phytosanitary, import license requirements
4. **CN regulatory triggers** — SAMR, NMPA, MARA, MIIT, MEE agency mapping
5. **RCEP origin rules engine** — Cumulation, self-certification, Form RCEP
6. **CN export controls** — Unreliable entity list, dual-use items catalog, rare earth restrictions
7. **CN trade mode support** — General trade (0110) + processing trade (0214/0615) + bonded e-commerce (1210/9610)

### Phase 3: Japan & South Korea (P1-P2)
1. **JP tariff regime** (`jp.py`) — MFN rates + consumption tax (10%/8%)
2. **KR tariff regime** (`kr.py`) — MFN rates + VAT (10%) + individual consumption tax
3. **NACCS/UNI-PASS declaration models**
4. **JP/KR regulatory triggers** — MAFF, MHLW, METI, MFDS, KCC
5. **JP/KR FTA rate integration** — Japan-EU EPA, CPTPP, KORUS expansion

### Phase 4: ASEAN Core (P2)
1. **Singapore TradeNet model** — permit types, GST
2. **Vietnam VNACCS model** — channel classification, tax regime
3. **Thailand e-Customs model** — duty + VAT
4. **Indonesia INSW model** — API licensing, complex tax stack
5. **ATIGA Form D origin certification**

### Phase 5: Deep Integration (P3)
1. **Cross-border e-commerce (行邮税) engine** — separate tax computation for 9610/1210 modes
2. **Processing trade manual system** — lifecycle tracking for 来料加工/进料加工
3. **Bonded zone preferential policies** — Shanghai FTZ, Hainan FTP
4. **APAC carrier expansion** — regional carriers for CN, JP, KR, ASEAN
5. **Multi-language document support** — CJK OCR, bilingual document templates
6. **Time zone-aware operations** — APAC business hours, filing deadlines, payment windows

---

## Effort Estimates

| Phase | Scope | Backend | Frontend | Tests | Total |
|-------|-------|---------|----------|-------|-------|
| Phase 1 | Foundation | 3-4 weeks | 2-3 weeks | 1-2 weeks | 6-9 weeks |
| Phase 2 | China First-Class | 4-6 weeks | 3-4 weeks | 2-3 weeks | 9-13 weeks |
| Phase 3 | Japan & Korea | 3-4 weeks | 2-3 weeks | 1-2 weeks | 6-9 weeks |
| Phase 4 | ASEAN Core | 4-5 weeks | 3-4 weeks | 2-3 weeks | 9-12 weeks |
| Phase 5 | Deep Integration | 3-4 weeks | 2-3 weeks | 1-2 weeks | 6-9 weeks |
| **Total** | | | | | **36-52 weeks** |

---

## Key Risks

1. **Regulatory data sourcing** — China's tariff rates, provisional rates, and regulatory requirements change frequently. Need a reliable data pipeline (consider GACC API, WCO databases, or commercial providers like Descartes/Amber Road)
2. **CJK character handling** — Chinese/Japanese/Korean characters in product descriptions, company names, and documents require proper encoding, search, and display
3. **Time zone complexity** — Filing deadlines, business hours, and payment windows span UTC+5:30 to UTC+12
4. **Data sovereignty** — China's data localization requirements (PIPL, Data Security Law) may require CN-hosted infrastructure for GACC-facing features
5. **Political sensitivity** — Taiwan trade mode (ECFA), Hong Kong special status, sanctions regimes require careful handling
6. **Processing trade complexity** — China's processing trade manual system is genuinely complex (bond management, reconciliation, duty exemption tracking) and represents ~15% of trade volume
