# Brazil Customs Clearance: Import, Export, and Regulatory Framework

**Clearance Intelligence Engine -- Operational Knowledge Base**  
**Last Updated: February 2026**

---

## 1. Brazil Customs (Receita Federal do Brasil -- RFB)

### 1.1 Organization and Role of RFB

The **Receita Federal do Brasil (RFB)** -- the Brazilian Federal Revenue Service -- is the primary authority responsible for customs administration in Brazil. It operates under the **Ministerio da Fazenda (Ministry of Finance)** and administers:

- Collection and oversight of all federal taxes on imports and exports
- Customs clearance (despacho aduaneiro) for import and export operations
- Management of special customs regimes (drawback, bonded warehouses, temporary admission, etc.)
- Border control and cargo inspection
- RADAR registration and enablement of foreign trade operators
- The OEA (Operador Economico Autorizado) certification program

Joint oversight of the foreign trade system is shared between the **Ministerio da Fazenda** and the **Ministerio do Desenvolvimento, Industria, Comercio e Servicos (MDIC)**.

### 1.2 Siscomex (Sistema Integrado de Comercio Exterior)

**Siscomex** was created in 1992 as Brazil's integrated electronic foreign trade system. It registers all import and export procedures and allows the government to have full supervision of international trading in Brazil.

- **Siscomex Importacao**: The import module where importers register Import Declarations (DI) and now the DUIMP. It handles import licensing (LI), customs clearance, and tax payments.
- **Siscomex Exportacao**: The export module for registering Export Declarations (DE/DU-E), export licenses, and related documentation.

To operate in Siscomex, every importer or exporter (whether individual or legal entity) must be enabled through a password obtained from the Receita Federal via the RADAR registration process.

### 1.3 Portal Unico de Comercio Exterior (Single Window for Foreign Trade)

The **Portal Unico Siscomex** is the modernization platform replacing the legacy Siscomex LI/DI system that had been in force since the 1990s. Key characteristics:

- Establishes a **single window** (guiche unico eletronico) to centralize interaction between government agencies and private operators
- Reformulates export and import processes for greater efficiency and harmonization
- Implements the **DUIMP** (Declaracao Unica de Importacao) as the unified import declaration
- Integrates the **LPCO** (Licencas, Permissoes, Certificados e Outros Documentos) module for administrative licensing
- Implements **Pagamento Centralizado de Comercio Exterior (PCCE)** -- centralized payment module
- Legal basis: Law 14.195/2021 and Coana Ordinance 165/2024

The extinction of the legacy Siscomex LI/DI system was targeted for completion by end of 2025. One of the determining factors was the beginning of Brazil's tax reform transition period in 2026.

---

## 2. Import Process

### 2.1 RADAR (Rastreamento da Atuacao dos Intervenientes Aduaneiros)

RADAR is the **mandatory registration** that grants companies access to Siscomex and the legal ability to import/export in Brazil. Without RADAR, a company cannot clear goods through Brazilian customs. The Receita Federal conducts a financial analysis of the company to determine the appropriate RADAR category.

**RADAR Categories:**

| Category | Import Limit (per semester) | Export Limit | Target |
|---|---|---|---|
| **Expresso (Express)** | USD 50,000 | Unlimited | Micro-entrepreneurs, small-volume importers |
| **Limitada (Limited)** | USD 150,000 | Unlimited | Beginner/mid-size import-export companies |
| **Ilimitada (Unlimited)** | No preset limit | Unlimited | Large, well-capitalized businesses |

- **Express**: Requires minimal documentation; simplified fiscal analysis
- **Limited**: Requires financial capacity of at least BRL 213,559 (but not exceeding BRL 640,679); stricter documentation
- **Unlimited**: Requires financial capacity exceeding BRL 640,679; extensive documentation including formation documents, tax filings, contracts for warehouse space

**Key details:**
- Maximum processing time: 10 business days
- Validity: 18 months for Limited and Unlimited (from grant date or last foreign trade operation)
- Upgrading from Limited to Unlimited requires documentation proving increased financial capacity

### 2.2 Import Licensing

Brazil employs two types of import licenses:

- **Automatic licenses (Licenca Automatica)**: Relatively simple, automatically approved upon registration. Applied to goods not subject to special controls.
- **Non-automatic licenses (Licenca Nao Automatica)**: Requires approval from the relevant Brazilian authority (DECEX, ANVISA, MAPA, IBAMA, INMETRO, the Army, etc.). Up to 16 different agencies may be involved depending on the product. Processing time: 15-60+ days. Generally must be requested before shipment.

Under the Portal Unico, licensing is integrated through the **LPCO module** (Licencas, Permissoes, Certificados e Outros Documentos), which consolidates the previously separate licensing workflows.

### 2.3 Import Declaration: DI to DUIMP Transition

- **DI (Declaracao de Importacao)**: The legacy import declaration registered in the Siscomex LI/DI system. Being phased out.
- **DUIMP (Declaracao Unica de Importacao)**: The new unified import declaration in the Portal Unico. Defined by Coana Ordinance No. 165/2024.

**DUIMP key features:**
- Can be registered **before cargo arrival**, initiating customs clearance earlier
- Centralizes all customs, administrative, commercial, financial, and tax data
- Integrates administrative agency approvals (ANVISA, MAPA, etc.)
- Mandatory since October 2024 for RECOF, REPETRO, and Temporary Admission regimes
- Payments processed through the Pagamento Centralizado (PCCE) module
- OEA-certified companies can defer duty payments up to 40 days after clearance

### 2.4 Parametrizacao (Channel System)

Upon registration of the DI or DUIMP, the Receita Federal's computerized systems perform an **automatic risk analysis** considering declared information, importer history, characteristics of goods, and other criteria. The declaration is classified into one of four channels:

| Channel | Name (Portuguese) | Procedure |
|---|---|---|
| **Green** | Canal Verde | Automatic release; no documentary or physical verification |
| **Yellow** | Canal Amarelo | Documentary analysis only; customs reviews commercial invoice, B/L, packing list, etc. |
| **Red** | Canal Vermelho | Documentary verification + physical inspection of goods (quantity, description, classification, physical characteristics) |
| **Grey** | Canal Cinza | Documentary + physical + special customs control procedure; triggered by suspected fraud (price manipulation, shell companies, document falsification, concealment of participants) |

**Consolidated Channel in DUIMP:** The new system presents both the Receita Federal's selection channel and the inspecting agency's channel (e.g., MAPA, ANVISA). The consolidated channel is always the more restrictive of the two. Example: RFB green + MAPA yellow = consolidated yellow.

### 2.5 Required Documents

| Document | Details |
|---|---|
| **Commercial Invoice (Fatura Comercial)** | Must include exporter/importer details, quantities, brands, net/gross weights, country of origin, payment conditions. Minimum 2-5 copies in English or Portuguese. Must bear importer's CNPJ. |
| **Packing List (Romaneio de Carga)** | Contents of each package, types/quantities of items, individual weights, package marks |
| **Bill of Lading (B/L) or Air Waybill (AWB)** | Must include NCM code (minimum first 4 digits), volume in cubic meters, freight in numbers and text, consignee address in Brazil |
| **Certificate of Origin (Certificado de Origem)** | Required for preferential tariff treatment under trade agreements (Mercosur, ALADI, ACE agreements). Not always mandatory otherwise. |
| **Import License (Licenca de Importacao / LPCO)** | Required for controlled products; obtained from relevant agency before or after shipment but before clearance |
| **Proof of Payment / Exchange Contract** | Evidence of financial settlement of the transaction |
| **Insurance Certificate** | When applicable to the Incoterms used |

---

## 3. Brazil's Cascading Tax Structure (Pattern B)

This is the defining characteristic of Brazil's import taxation and the reason software implementations require algebraic/gross-up solvers rather than simple sequential multiplication. The taxes cascade -- each builds on a base that includes previously calculated taxes, and ICMS includes itself in its own base.

### 3.1 Tax Calculation Order and Formulas

#### Step 1: Customs Value (Valor Aduaneiro)
Brazil uses **CIF valuation** (Cost, Insurance, and Freight) per the WTO Customs Valuation Agreement.

```
Customs_Value = FOB + International_Freight + Insurance
```

#### Step 2: Imposto de Importacao (II) -- Import Duty
- **Type**: Federal, ad valorem
- **Base**: Customs Value (CIF)
- **Rates**: Based on NCM/TEC code, typically 0-35% (most goods 10-20%)

```
II = Customs_Value * II_Rate
```

#### Step 3: IPI (Imposto sobre Produtos Industrializados) -- Federal Excise Tax
- **Type**: Federal excise tax on industrialized/manufactured products
- **Base**: Customs Value + II (cascading on import duty)
- **Rates**: Vary by NCM code, typically 0-30%+

```
IPI = (Customs_Value + II) * IPI_Rate
```

#### Step 4: PIS-Importacao and COFINS-Importacao -- Social Contributions
- **PIS-Importacao (Programa de Integracao Social)**: Social integration contribution
  - Standard rate: **2.1%** on goods imports
- **COFINS-Importacao (Contribuicao para Financiamento da Seguridade Social)**: Social security contribution
  - Standard rate: **9.65%** on goods imports
  - Additional 1% COFINS applies to certain listed products
- **Combined standard rate**: **11.75%**
- **Base**: Customs Value (CIF) -- per STF ruling, the base should not include ICMS or the contributions themselves
- Special increased rates apply to specific product categories (pharmaceuticals, cosmetics, machinery, vehicles -- up to 20% combined)

```
PIS = Customs_Value * PIS_Rate  (standard: 2.1%)
COFINS = Customs_Value * COFINS_Rate  (standard: 9.65%)
```

Note: For services imports, rates are 1.65% (PIS) and 7.6% (COFINS), combined 9.25%.

#### Step 5: ICMS (Imposto sobre Circulacao de Mercadorias e Servicos) -- State VAT

**THIS IS THE CRITICAL CALCULATION. ICMS is calculated on a tax-inclusive base -- the tax is part of its own calculation base.**

- **Type**: State-level value-added tax (imposto estadual)
- **Rates**: Vary by state, typically 17-20% (Sao Paulo: 18%, Rio de Janeiro: 20%, Minas Gerais: 18%)
- **Base**: CIF + II + IPI + PIS + COFINS + other charges + **ICMS itself**

**The gross-up (tax-inclusive) formula:**

```
Pre_ICMS_Base = Customs_Value + II + IPI + PIS + COFINS + Other_Charges
ICMS_Base = Pre_ICMS_Base / (1 - ICMS_Rate)
ICMS = ICMS_Base * ICMS_Rate
```

**Algebraic derivation:**
```
ICMS = ICMS_Rate * (Pre_ICMS_Base + ICMS)
ICMS = ICMS_Rate * Pre_ICMS_Base + ICMS_Rate * ICMS
ICMS - ICMS_Rate * ICMS = ICMS_Rate * Pre_ICMS_Base
ICMS * (1 - ICMS_Rate) = ICMS_Rate * Pre_ICMS_Base
ICMS = (ICMS_Rate * Pre_ICMS_Base) / (1 - ICMS_Rate)
```

**Effective rate impact**: Although the nominal ICMS rate in Sao Paulo is 18%, the gross-up means the effective burden is approximately **21.95%** of the pre-ICMS value (18% / (1 - 0.18) = 21.95%).

**ICMS rate examples (2025):**
| State | Internal Rate |
|---|---|
| Sao Paulo (SP) | 18% |
| Rio de Janeiro (RJ) | 20% |
| Minas Gerais (MG) | 18% |
| Rio Grande do Sul (RS) | 17% |
| Rio Grande do Norte (RN) | 20% (increased March 2025) |
| Parana (PR) | 19.5% |

Interstate differentials apply for goods moving between states.

#### Step 6: AFRMM (Adicional ao Frete para Renovacao da Marinha Mercante)
- **Type**: Federal contribution on ocean freight
- **Rate**: **8%** on international (long-course) freight value (reduced from 25% by Law 14.301/2022)
- **Base**: Freight value (including port handling charges) as registered on the CE-Mercante (Electronic Bill of Lading)
- Applies only to **sea freight** (not air or land transport)
- Administered by Receita Federal since Laws 12.599/12 and 12.788/13
- Triggering event: start of unloading at a Brazilian port

```
AFRMM = Ocean_Freight_Value * 0.08
```

**Exceptions:**
- Cargo from Mercosur member states (non-incidence)
- Cargo to/from ports in North and Northeast regions (cabotage exemption until January 7, 2027)
- River/lake navigation for liquid bulk in N/NE regions: 40%

**Critical note:** If AFRMM is not paid, the Mercante system blocks release in SISCARGA -- cargo will not be released from port.

#### Step 7: Siscomex Fee (Taxa de Utilizacao do Siscomex)
- **Base fee**: BRL 185.00 per import declaration
- **Additional**: BRL 29.50 per additional product (different NCM code) listed
- Fixed fee, not ad valorem

```
Siscomex_Fee = 185 + (29.50 * (number_of_NCM_items - 1))
```

### 3.2 Complete Worked Example

**Product**: Wristwatches  
**CIF Value**: BRL 100,000  
**II Rate**: 18% | **IPI Rate**: 10% | **ICMS Rate (SP)**: 18%  
**PIS Rate**: 2.1% | **COFINS Rate**: 9.65%  
**Ocean Freight**: BRL 10,000  

```
II     = 100,000 * 0.18                           = BRL  18,000
IPI    = (100,000 + 18,000) * 0.10                 = BRL  11,800
PIS    = 100,000 * 0.021                           = BRL   2,100
COFINS = 100,000 * 0.0965                          = BRL   9,650

Pre_ICMS_Base = 100,000 + 18,000 + 11,800 + 2,100 + 9,650 = BRL 141,550
ICMS_Base     = 141,550 / (1 - 0.18)              = BRL 172,621.95
ICMS          = 172,621.95 * 0.18                  = BRL  31,071.95

AFRMM  = 10,000 * 0.08                            = BRL     800
Siscomex Fee (1 NCM)                               = BRL     185

TOTAL TAXES = 18,000 + 11,800 + 2,100 + 9,650 + 31,071.95 + 800 + 185
            = BRL 73,606.95  (73.6% effective rate on CIF)
```

### 3.3 Why This Requires an Algebraic Solver

The ICMS gross-up means you cannot simply multiply rates sequentially. The equation `ICMS = Rate * (Base + ICMS)` is self-referential and requires solving algebraically. Additionally, the ordering of PIS/COFINS relative to ICMS creates interdependencies in some calculation methodologies. A software engine must implement either:

1. **Algebraic solver**: Directly solve the system of equations using the gross-up formula `ICMS = Rate * Base / (1 - Rate)`
2. **Iterative convergence**: Iterate until the ICMS value stabilizes (less efficient, not recommended)

---

## 4. Brazil's Tariff Classification

### 4.1 NCM (Nomenclatura Comum do Mercosul)

The **NCM** is the standardized 8-digit code system used by all Mercosur member states for classifying goods. Structure:

```
XXXX.XX.XX
|    |  |
|    |  +-- Mercosur subitem (digits 7-8, Mercosur-specific)
|    +----- HS subheading (digits 5-6)
+---------- HS heading and chapter (digits 1-4)
```

- Digits 1-6: Identical to the WCO Harmonized System (HS)
- Digits 7-8: Mercosur-specific additions agreed upon by member states
- Changes to NCM require submission to Technical Committee No. 1 (CT-1) of Mercosur, deliberation by GECEX (Executive Committee for Management of CAMEX), and then approval by CT-1 with all member states

### 4.2 TEC (Tarifa Externa Comum) -- Common External Tariff

The **TEC** is the common import tariff applied by all Mercosur members to goods from non-member countries, established January 1, 1995.

**TEC structure: 11 tariff levels from 0% to 20%**, increasing by 2 percentage points per level:

| Product Category | Typical TEC Range |
|---|---|
| Raw materials | 0-12% |
| Capital goods (Bens de Capital) | 12-16% |
| Consumer goods (final products) | 18-20% |

The TEC is composed of the NCM code plus the corresponding tariff rate at the 8-digit level.

### 4.3 Ex-Tarifario (Temporary Tariff Reductions)

The **Ex-Tarifario** regime provides temporary reductions of Import Duty (typically to **0%**) for:

- **BK (Bens de Capital)**: Capital goods without equivalent domestic production
- **BIT (Bens de Informatica e Telecomunicacoes)**: IT and telecommunications equipment without equivalent domestic production

Governed by Resolucao GECEX 512/2023 (as amended). Ex-tarifarios are identified by an "Ex" notation appended to the NCM code (e.g., "Ex 001" under a given NCM). They are regularly updated through GECEX resolutions -- multiple resolutions were issued in 2025 adding, amending, and revoking specific ex-tarifarios.

---

## 5. Brazil's Trade Agreements

### 5.1 Mercosur (Mercado Comun del Sur)

Full members: **Argentina, Brazil, Paraguay, Uruguay**. Venezuela's membership is suspended. Bolivia is in the process of accession.

Foundational instrument: **ACE-18** (Economic Complementation Agreement No. 18), signed under ALADI in 1991, implemented in Brazil by Decree 550/92. Intra-Mercosur trade generally benefits from 0% import duties on goods meeting rules of origin.

### 5.2 Mercosur External Agreements

| Agreement | Partner | Status | Key Details |
|---|---|---|---|
| Mercosur-EU | European Union | Concluded Dec 2024; pending ratification | Comprehensive partnership agreement covering trade, investment, procurement |
| Mercosur-Israel | Israel | In force | FTA covering ~8,000 tariff lines (Israel) and ~9,424 (Mercosur); 8-10 year elimination schedules. Brazil: Decree 7.159/2010 |
| Mercosur-Egypt | Egypt | In force | Preferential trade agreement |
| Mercosur-SACU | South Africa, Namibia, Botswana, Lesotho, Eswatini | In force | PTA: 1,026 tariff lines (SACU) and 1,076 (Mercosur) |
| Mercosur-India | India | In force | PTA: 450 lines (India) / 452 (Mercosur); preference margins of 10%, 20%, or 100%. Brazil: Decree 6.864/2009 |
| Mercosur-Colombia | Colombia | In force | ACE-72: Free trade zone. Brazil: Decree 9.230/2017 |
| Mercosur-Mexico (Automotive) | Mexico | In force | ACE-55: Bilateral automotive trade |

**Ongoing negotiations:** Mercosur-Canada (since 2018), Mercosur-South Korea, Mercosur-EFTA (since 2017), Mercosur-Indonesia (since 2021), Mercosur-UAE (launched July 2024).

### 5.3 ALADI (Associacao Latino-Americana de Integracao)

Created in 1980 by the Treaty of Montevideo. 13 member countries: Argentina, Bolivia, Brazil, Chile, Colombia, Cuba, Ecuador, Mexico, Paraguay, Panama, Peru, Uruguay, Venezuela.

**Key ALADI instruments for Brazil:**

| Instrument | Description |
|---|---|
| **RTPA-4** (Regional Tariff Preferences Agreement 4) | Tariff reductions based on development level of exporting country. Brazil: Decree 90.782/1984 |
| **RA-7** | Regional agreement on free circulation of cultural, educational, and scientific materials. Brazil: Decree 97.487/1989 |

### 5.4 Bilateral ACE Agreements (Acordos de Complementacao Economica)

| ACE | Partners | Scope |
|---|---|---|
| ACE-2 | Brazil-Uruguay | Automotive products. Decree 88.419/1983 |
| ACE-14 | Brazil-Argentina | Common market conditions; largely superseded by ACE-18 except automotive |
| ACE-35 | Mercosur-Chile | Free trade |
| ACE-53 | Brazil-Mexico | ~800 tariff codes with fixed preferences. Decree 4.383/2002 |
| ACE-55 | Mercosur-Mexico | Automotive sector |
| ACE-58 | Mercosur-Peru | Free trade |
| ACE-59 | Mercosur-Andean Community | Excluding Bolivia and Peru (covered separately) |
| ACE-69 | Brazil-Venezuela | Bilateral preferences |
| ACE-74 | Brazil-Paraguay | Free trade zone. Decree 10.448/2020 |

---

## 6. Special Customs Regimes (Regimes Aduaneiros Especiais)

Governed primarily by Decree-Law 37/1966 and Decree 6,759/2009 (Regulamento Aduaneiro).

### 6.1 Drawback

The drawback regime incentivizes exports by relieving taxes on inputs used in the production of exported goods. Managed by SECEX jointly with Receita Federal.

**Three modalities:**

| Modality | Mechanism | Taxes Covered | Status |
|---|---|---|---|
| **Suspensao (Suspension)** | Taxes suspended at import/domestic purchase; converted to exemption upon export completion | II, IPI, PIS-Import, COFINS-Import, AFRMM (imports); IPI, PIS, COFINS (domestic) | Most widely used |
| **Isencao (Exemption)** | Tax-free replenishment of stock equivalent to inputs previously used in exported products | II, IPI (reduced to zero), PIS, COFINS, PIS-Import, COFINS-Import | Actively used |
| **Restituicao (Restitution)** | Refund of taxes already paid on inputs used in exported goods | All import taxes paid | Virtually obsolete in practice |

**Drawback Integrado (Integrated)**: Extends suspension/exemption benefits to domestic purchases of inputs, not just imports.

**2025 developments:** Law Complementar 216/2025 expanded drawback suspensao to include **services** linked to exports (freight, insurance, customs clearance). Effective July 29, 2025, for concession acts from January 1, 2023. Taxes suspended on services: PIS/PASEP, COFINS, PIS/PASEP-Import, COFINS-Import.

**Tax reform impact:** Lei Complementar 214/2025 limits the new IBS/CBS taxes to the suspension modality only -- exemption and restitution modalities will not apply to the new taxes.

### 6.2 RECOF (Regime Aduaneiro Especial de Entreposto Industrial)

A special industrial warehousing regime allowing authorized companies to stock imported goods under **tax suspension** for use in manufacturing or re-export:

- Functions as a bonded manufacturing warehouse
- Taxes remain suspended while goods are in the regime
- If goods leave for domestic sale: taxes become due
- If goods leave for export: exempt
- Eligibility: high-volume importers/exporters with robust fiscal controls
- **RECOF-SPED**: Simplified version using digital bookkeeping (SPED)
- From October 2024, DUIMP is mandatory for RECOF operations
- RECOF regime expanded to include services effective January 1, 2026

### 6.3 Temporary Admission (Admissao Temporaria)

Allows temporary importation of goods with **suspension or reduction of taxes**, provided the item is re-exported within a set period. Use cases:

- Trade shows and exhibitions
- Equipment for repair
- Leased equipment
- Goods for testing or demonstration
- Cultural, scientific, or sporting events

Taxes are suspended for the duration of the temporary admission. Upon re-export, the suspension converts to exemption.

### 6.4 Bonded Warehouses (Entreposto Aduaneiro)

Allows storage of imported goods under customs control **without immediate tax collection**:

- **Taxes suspended**: II, IPI, PIS/COFINS-Import, and ICMS
- Goods can be sold to different Brazilian buyers while in the bonded warehouse
- Upon nationalization (clearance for domestic consumption): all taxes become due
- Upon re-export: suspension converts to exemption
- If goods are not cleared or re-exported within the concession period: taxes + fines + default interest apply
- Certified Bonded Warehouses (Entreposto Aduaneiro Certificado) offer additional services

### 6.5 Zona Franca de Manaus (ZFM -- Manaus Free Trade Zone)

A special economic zone in the Amazonas state offering significant tax incentives:

- **Import Duty (II)**: Exempt for approved industries
- **IPI**: Exempt for manufactured goods within the zone
- **ICMS**: Reduced rates
- Historically successful in attracting electronics, motorcycle, and automotive component manufacturers
- Goods moving out of ZFM to other Brazilian states are generally subject to ICMS
- Strategy: Import into ZFM, perform light processing/manufacturing, distribute within Brazil with tax advantages

### 6.6 REPETRO (Regime Aduaneiro Especial de Exportacao e de Importacao de Bens Destinados as Atividades de Pesquisa e de Lavra das Jazidas de Petroleo e de Gas Natural)

Tax incentive regime for the **offshore oil and gas** industry:

- Allows temporary importation of goods and equipment for exploration/production **without paying** II, IPI, PIS, COFINS
- Aims to promote investment in Brazil's offshore oil and gas sector
- Reduces costs and bureaucratic burdens for specialized equipment and materials
- From October 2024, DUIMP is mandatory for REPETRO operations
- Related regimes: RECOM and REPEX for specific sectoral needs

---

## 7. Compliance and Enforcement

### 7.1 Transfer Pricing Rules

Brazil's transfer pricing rules underwent a **fundamental transformation** effective January 1, 2024, moving from a unique fixed-margins system to **OECD arm's-length standard** alignment (Law 14,596/2023).

**Previous system (pre-2024):**
- Used predetermined fixed profit margins (unique to Brazil)
- Safe harbors allowed straightforward compliance
- Limited subjectivity in application

**New system (2024 onward):**
- Full adoption of OECD Transfer Pricing Guidelines 2022
- Arm's-length principle as the guiding standard
- No more safe harbors
- Required documentation: **Master File** (OECD-compliant), **Local File** (detailed intercompany transaction descriptions, method selection rationale), Transfer pricing forms in ECF (corporate income tax return, due June 30 annually)

**Interaction with customs valuation:**
- Law 14,596/2023 formally applies only to income taxes (IRPJ and CSLL)
- However, compensating adjustments that increase import costs may increase the base for II, ICMS, IPI, PIS/COFINS
- Adjustments decreasing import costs may provide grounds for reimbursement

### 7.2 Penalties for Incorrect Declarations

| Infraction | Penalty |
|---|---|
| **Incorrect NCM classification** | 1% of Customs Value (minimum BRL 500, maximum 10% of declaration value) |
| **Underpayment due to error** | 75% fine on tax difference |
| **Fraud / deliberate underdeclaration** | 150% fine on tax difference + potential seizure |
| **Under-declaration of value** | 150% fine + seizure |
| **Price misrepresentation** | 100% of the price difference (no minimum or maximum) |
| **Hiding/inaccurate information** | 1% of Customs Value (minimum BRL 500, maximum 10% of declaration) |
| **Missing import license** | 30% of Customs Value (minimum BRL 500, no maximum) |
| **Anti-dumping duty non-payment** | 75% of anti-dumping/compensatory duty amount |
| **Fraudulent interposition (using another company's name)** | 10% of operation value (minimum BRL 5,000) |

**Key legal framework:**
- Regulamento Aduaneiro -- Decree 6,759/2009
- Law 9,430/1996 (consultation procedures)
- IN RFB 2,057/2021 (formal NCM consultation procedures)

**Practical note:** Brazil's customs bureaucracy is highly formalistic. Documents must be meticulously prepared. Even minor errors in NCM classification can trigger fines regardless of whether any tax difference exists. Pre-import private tax rulings (consultas) are commonly used to prevent classification disputes.

### 7.3 Customs Litigation System

Tax penalties for transfer pricing non-compliance can reach up to **150% of taxes underreported**. The lack of a tax treaty between Brazil and the US (Brazil's largest trade partner among OECD countries) means disputes cannot be resolved through mutual agreement procedures, increasing litigation risk. Brazilian tax authorities (RFB) are historically strict in enforcement, and the transition to the subjective arm's-length standard is expected to increase disputes.

### 7.4 OEA (Operador Economico Autorizado) -- Authorized Economic Operator

Brazil's AEO equivalent, established in 2014 under the WCO SAFE Framework of Standards.

**OEA Modalities:**
- **OEA-Seguranca (Security)**: For exports; launched December 2014
- **OEA-Conformidade (Compliance)**: For imports
- **OEA-Integrado (Integrated)**: Combines both and integrates other public entities (ANVISA, VIGIAGRO)

**OEA Benefits:**
- Reduced percentage of cargo selected for inspection
- Priority processing of import/export clearance
- Green channel access for temporary admission
- Exemption from guarantees for customs transit
- Priority access to bonded warehouses
- Participation in Consultative Forum (can propose regulatory changes)
- Waiver of requirements already met in certification process
- Deferred payments (up to 40 days post-clearance for DUIMP)

**Mutual Recognition Agreements (MRAs):** Brazil's OEA is recognized by the US, Argentina, Uruguay, China, Mercosur, Bolivia, Peru, Mexico, and Colombia.

**Market significance:** As of mid-2021, OEA-certified operators represented approximately **26% of registered customs declarations** and **25% of FOB trade value**.

---

## 8. Upcoming Tax Reform (2026-2033 Transition)

Brazil is undergoing a historic tax reform that will fundamentally restructure the import tax landscape:

- **CBS (Contribuicao sobre Bens e Servicos)**: New federal tax replacing PIS and COFINS (beginning phase-in 2026-2027)
- **IBS (Imposto sobre Bens e Servicos)**: New state/municipal tax replacing ICMS and ISS
- **Imposto Seletivo (Selective Tax)**: New excise tax on specific products (replacing IPI for most goods)
- **Transition period**: 2026-2032, with old and new taxes coexisting
- **Full implementation**: By 2033, PIS, COFINS, IPI, ICMS, and ISS will be fully replaced by CBS, IBS, and Selective Tax

**Impact on import calculations:** During the transition period (2026-2032), both the legacy tax structure and the new dual VAT components will apply, significantly increasing calculation complexity. Software systems must handle both regimes simultaneously.

---

## Sources

- [Siscomex - Portal Gov.br](https://www.gov.br/siscomex/pt-br)
- [Portal Unico Siscomex](https://portalunico.siscomex.gov.br/portal/)
- [Customs Regulations for Brazil 2026 - Doerrenhaus](https://www.doerrenhaus.com/en/zollbestimmungen-brasilien/)
- [NOVATRADE - RADAR Habilitation](https://novatradebrasil.com/en/obtain-radar-habilitation-siscomex-brazil/)
- [The Brazil Business - Complete Guide to Brazilian Import](https://thebrazilbusiness.com/article/the-complete-guide-to-brazilian-import)
- [FGX - Brazil RADAR License Guide](https://fgx.com/blog/the-brazil-radar-license-guide-for-global-it-and-procurement-teams)
- [Traddal - Calculate Duties & Taxes on Imports to Brazil](https://traddal.com/resources/calculate-duties-taxes-imports-brazil)
- [The Brazil Business - How to Calculate Brazilian Import Duties and Taxes](https://thebrazilbusiness.com/article/how-to-calculate-brazilian-import-duties-and-taxes)
- [BPC Partners - Exports to Brazil / Importation Taxes](https://bpc-partners.com/brazilian-taxes-what-you-need-to-know/exports-to-brazil/)
- [NOVATRADE - Import Duties and Taxes in Brazil](https://novatradebrasil.com/en/import-duties-taxes-brazil/)
- [MyBusinessBrazil - Import Costs in Brazil 2026](https://mybusinessbrazil.com/import-costs-in-brazil-2026-what-foreign-companies-must-know/)
- [PWC - Brazil Corporate Other Taxes](https://taxsummaries.pwc.com/brazil/corporate/other-taxes)
- [Grant Thornton - Indirect Tax Brazil](https://www.grantthornton.global/en/insights/indirect-tax-guide/indirect-tax---Brazil/)
- [U.S. International Trade Administration - Brazil Import Tariffs](https://www.trade.gov/country-commercial-guides/brazil-import-tariffs)
- [Mercosur Portal - NCM](https://www.mercosur.int/pt-br/politica-comercial/ncm/)
- [NOVATRADE - NCM Code Classification Brazil](https://novatradebrasil.com/en/customs-code-classification-brazil-ncm/)
- [Chambers and Partners - International Trade 2025 Brazil](https://practiceguides.chambers.com/practice-guides/international-trade-2025/brazil)
- [NOVATRADE - International Agreements for Foreign Companies](https://novatradebrasil.com/en/international-agreements-for-foreign-companies-eyeing-expansion-into-brazil/)
- [OAS/SICE - Trade Agreements by Country: Brazil](http://www.sice.oas.org/ctyindex/brz/brzagreements_e.asp)
- [Chambers - Tax Reform and Special Customs Regimes](https://chambers.com/articles/tax-reform-and-special-customs-regimes)
- [Lexology - Snapshot: Customs Duties in Brazil](https://www.lexology.com/library/detail.aspx?g=f776ecd1-cf2b-4af7-bac5-d65e9ca5fc96)
- [VATupdate - Brazil Expands Drawback and RECOF to Include Services](https://www.vatupdate.com/2025/08/06/brazil-expands-drawback-and-recof-regimes-to-include-services-effective-2025-and-2026/)
- [International Tax Review - Brazilian Government Boosts Drawback Benefits](https://www.internationaltaxreview.com/article/2atv32mg832llrg3lju2o/sponsored/brazilian-government-boosts-the-benefits-applicable-under-the-drawback-regime)
- [The Brazil Business - Penalties for Errors When Importing to Brazil](https://thebrazilbusiness.com/article/penalties-for-errors-when-importing-to-brazil)
- [MyBusinessBrazil - Tax Risks in Importing](https://mybusinessbrazil.com/tax-risks-in-importing-common-classification-errors-and-how-to-avoid-them/)
- [EY - New Transfer Pricing Rules in Brazil](https://www.ey.com/en_ch/insights/tax/new-transfer-pricing-rules-in-brazil)
- [Chambers - Transfer Pricing 2025 Brazil](https://practiceguides.chambers.com/practice-guides/transfer-pricing-2025/brazil)
- [Gov.br - AFRMM](https://www.gov.br/portos-e-aeroportos/pt-br/assuntos/incentivos/fmm-fundo-da-marinha-mercante/adicional-ao-frete-para-a-renovacao-da-marinha-mercante-afrmm)
- [H-Arcana - ICMS 2025 for Foreign Investors](https://h-arcana.com/2025/07/18/icms-2025-for-foreign-investors-complete-guide-to-brazils-tax-system/)
- [Mondaq - Amounts in ICMS Calculation Base](https://www.mondaq.com/brazil/tax-authorities/453530/amounts-that-must-be-included-in-the-base-for-the-icms-calculation)
- [U.S. Trade.gov - Brazil Import Requirements and Documentation](https://www.trade.gov/country-commercial-guides/brazil-import-requirements-and-documentation)
- [DLA Piper - Global Expansion Guide Tax Brazil](https://intelligence.dlapiper.com/global-expansion-tax/?c=BR)
- [Gov.br - Receita Federal DUIMP](https://www.gov.br/receitafederal/pt-br/assuntos/aduana-e-comercio-exterior/manuais/despacho-de-importacao/sistemas/duimp)