# Final Jurisdiction Validation Review — Clearance Platform Architecture (v5)

**Reviewer:** Licensed Customs Broker / International Trade Compliance Specialist (15 years, multi-jurisdiction: US, EU, CN, BR, IN, MX, CA)
**Date:** 2026-02-09
**Documents Reviewed:**
- `docs/architecture-clearance-platform.md` (v5, definitive reference)
- `docs/review-jurisdiction-deep-dive.md` (jurisdiction requirements analysis, 2026-02-08)
- `docs/review-broker-expert.md` (broker operations review, 2026-02-08)

**Purpose:** Validate that v5 architecture actually addresses the non-US customs requirements raised in prior reviews. This is a final "ready to build?" assessment.

---

## Executive Assessment

**The v5 architecture has made substantial progress toward multi-jurisdiction readiness.** The jurisdiction adapter pattern (Section 11), two-layer status model (Section 12.1), Party Management domain (Section 3.14), ENS filing entity, in-bond detail, and expanded entry types all directly address the P0 findings from the jurisdiction deep-dive and broker expert reviews. The foundation is strong.

**However, critical gaps remain in the jurisdiction-specific detail.** The adapter pattern is defined at the interface level but the per-jurisdiction implementations are only sketched. Several jurisdiction-specific requirements that would break filing operations are still missing from the domain models. The duty calculation model remains structurally US-centric despite acknowledging that formulas differ. Export declarations are not modeled as first-class entities.

**Verdict: The architecture is ready for US production and multi-jurisdiction development. It is NOT yet ready for non-US production deployment without the P0 items identified below.**

---

## 1. United States (Baseline Jurisdiction)

### 1.1 What v5 Gets Right

The US model is mature. Compared to v4, v5 adds:

- **Entry/release vs entry summary split** (lines 1045-1059): Correctly modeled with `release_number`, `release_status`, `summary_status`, `summary_due_date`. The two-step process (CBP 3461 for release, CBP 7501 for duty assessment) is now explicit. This was a P0 showstopper in the broker review -- resolved.

- **Expanded entry types** (lines 1030-1035): Now includes 01, 02, 03, 06, 11, 21, 23, 31, 46, 86. The suspended Section 321/Type 86 is noted with configurability for reinstatement. This covers the critical types the broker review flagged.

- **In-bond movement** (lines 697-710): `in_bond_detail` on HU with bond type (IT/IE/TE), origin/destination port, transit time limit, filing reference, extension tracking. The `in_bond` HU status is present. This was a P0 showstopper -- resolved.

- **ACAS on air consolidation** (line 852): Correctly modeled as a carrier-side obligation on the Consolidation entity, separate from ISF (broker/importer obligation for ocean). This was a medium-risk finding -- resolved.

- **HU status expansion** (lines 666-675): `exam_ordered`, `general_order`, `seized`, `refused`, `abandoned` -- all correctly added with status definitions matching real CBP operational states.

- **AD/CVD separate tracking** (line 1499-1501): Cash deposit, deposit rate, separate bond. The Reconciliation entity includes `cbp_reconciliation` type for entry type 09.

- **Liquidation lifecycle** (lines 1517-1525): Status enum includes `extended`, `suspended`, `deemed_liquidated`. Extensions tracked with count, reason, deadline. 4-year statutory deadline present.

- **PMS payment workflow** (lines 1512-1515): Enrollment flag, statement period, payment due date. The broker review's finding about PMS is addressed.

- **Bond sufficiency** (lines 1531-1533): $50K standard noted, 10% annual duties, separate AD/CVD bond at 100% of margin. Configurable thresholds.

- **Party Management with IOR** (lines 1903-1915): First-class ImporterOfRecord entity with `ior_type` (self, designated_broker, carrier_as_ior). POA model with continuous/per-entry types and revocability.

- **Carrier-integrated brokerage** (lines 1881-1882): `carrier_affiliation` and `is_integrated_carrier_broker` on Party entity. This addresses the express integrator model gap.

### 1.2 Remaining US Gaps

| Gap | Severity | Impact |
|-----|----------|--------|
| **Remote Location Filing (RLF)** not modeled | P2 | Broker can file at any port regardless of cargo location -- common in express ops. Affects broker assignment logic but not blocking. |
| **ABI transaction types** not enumerated | P2 | Specific ACE EDI messages (01/ID, 01/ES, 01/RL, cargo release notifications) are implementation detail, not architecture gap. Address during EDI integration. |
| **FTZ privileged vs non-privileged** distinction is described (line 759) but not a field on any entity | P3 | Operational detail for FTZ operations. Can be added to `jurisdiction_config` JSONB. |
| **Protest timeline tracking** (180-day window, 2-year reliquidation, CIT escalation) | P2 | The `ProtestFiled` event exists but deadline enforcement is not explicit. Add to financial lifecycle. |

**US assessment: Production-ready.** The v5 architecture covers the US customs workflow comprehensively. Remaining gaps are operational refinements, not structural blockers.

---

## 2. European Union

### 2.1 What v5 Addresses

- **ENS filing entity** (lines 1126-1142): First-class `ENSFiling` entity with consolidation FK, carrier ID, status enum (pending/filed/registered/do_not_load/report_required/no_action), MRN, filing deadline, HS codes, consignor/consignee. Correctly identifies ENS as a carrier obligation, not broker. **Verified against current requirements:** ICS2 Release 3 is fully operational as of February 3, 2026 with v3 messaging mandatory across all transport modes.

- **EORI in Party model** (line 1861): `jurisdiction_identifiers.EU.eori` field exists. EORI is mandatory for ALL EU customs transactions -- this gating requirement is now tracked.

- **Adapter pattern for EU** (lines 2642): Filing workflow (ENS -> import declaration H1-H7 -> supplementary), duty formula (customs duty + AD/CVD + member-state VAT), key party registration (EORI) are defined in the adapter specification.

- **Two-layer status mapping** (lines 2658-2665): EU statuses mapped: risk_analysis/document_check -> under_review, under_control -> held, customs_released -> cleared, customs_debt_confirmed -> released. This mapping is correct.

- **Jurisdiction-specific adjudication events** (lines 1306): `clearance.adjudication.eu.*` with declaration-accepted, under-control, customs-released, released-to-cleared.

### 2.2 Critical Gaps

**P0 -- EU member state sub-jurisdictions are NOT modeled.**

The architecture uses `jurisdiction: "EU"` but there is no single EU customs declaration system. Each member state has its own electronic system (ATLAS in Germany, DELTA in France, AGS/DMS in Netherlands, PLDA in Belgium, AIDA in Italy). The `jurisdiction_identifiers.EU.eori` field exists, but the filing itself must go to the national system. The adapter pattern (Section 11.2) says "ICS2 + national systems (ATLAS, DELTA, AGS)" but the `EntryFiling.jurisdiction` field has no mechanism for `"EU-DE"` vs `"EU-FR"` vs `"EU-NL"`.

**Fix required:** Add `sub_jurisdiction: string?` to EntryFiling (e.g., "DE", "FR", "NL") or change the jurisdiction scheme to support compound codes. The EUAdapter must be parameterized by member state to select the correct national declaration system and VAT rate. This is P0 because without it, the platform cannot file a single EU import declaration.

**P0 -- VAT rate per member state is NOT a field on any entity.**

The duty formula says "customs duty + AD/CVD + member-state VAT" but the `TariffCalculation` entity has no `vat_rate` or `vat_amount` field. It has `duty_rate`, `total_duty`, `effective_rate`, and `line_items`. VAT could go into `line_items`, but the architecture does not specify this. VAT varies from 17% (Luxembourg) to 27% (Hungary), with multiple reduced rates per state. Without explicit VAT modeling, the EU duty calculator cannot produce correct totals.

**Fix required:** Add VAT as a mandatory line item type in `TariffCalculation.line_items` for EU. The jurisdiction adapter's `duty_calculator` must accept sub-jurisdiction (member state) to look up the correct VAT rate. Alternatively, add `vat_rate: decimal?` and `vat_amount: decimal?` to TariffCalculation.

**P1 -- NCTS transit (T1/T2) not modeled as filing entity.**

The architecture notes EU T1/T2 as the equivalent of US in-bond (line 709) and the in-bond model on HU has a comment about EU equivalence, but there is no `T1T2Filing` entity parallel to `ISFFiling` or `ENSFiling`. NCTS Phase 5 is fully operational across 29 countries as of 2026. T1/T2 declarations have their own status machine (submitted -> MRN assigned -> goods released for transit -> arrived at destination -> transit ended), guarantee requirements (cash deposit or comprehensive guarantee), and multi-office coordination.

**Fix required:** Add an `NCTSTransitFiling` entity to Declaration Management, or generalize the in-bond model to support EU transit document types. The in_bond_detail JSONB on HU could be extended with `transit_type: "T1" | "T2" | "IT" | "IE" | "TE"` but the filing itself needs a separate entity because it has its own submission, MRN assignment, and status lifecycle separate from the import declaration.

**P1 -- Customs procedure codes not modeled.**

EU import declarations require a 4-digit Customs Procedure Code (CPC) specifying the customs regime: 4000 (free circulation), 5100 (inward processing), 5300 (temporary admission), 7100 (customs warehousing). The previous procedure code is also required when goods are transitioning between regimes. No field exists for this on EntryFiling.

**Fix required:** Add `customs_procedure_code: string?` and `previous_procedure_code: string?` to EntryFiling's `jurisdiction_config` or as explicit fields.

**P2 -- EU document types incomplete.**

The Document entity's `document_type` enum (line 1788) does not include: `ens_declaration`, `single_administrative_document`, `eur1_certificate`, `eur_med_certificate`, `t1_transit`, `t2_transit`, `binding_tariff_information`, `aeo_certificate`. While documents can be stored with any filename, the type enum drives requirements determination logic. Missing types mean the Document Management domain cannot automatically determine what documents are required for EU shipments.

**P2 -- AEO mutual recognition benefits not operationalized.**

The Party model has `trust_programs` (line 1872) which can hold AEO certification. However, the checklist generator and adjudication logic do not reference trust program status. An AEO-certified importer should get reduced examination rates, simplified procedures, and faster release. This is a business logic gap, not a structural one.

### 2.3 EU Assessment

**Not production-ready.** The ENS and EORI additions are significant progress, but the lack of member-state sub-jurisdictions, explicit VAT modeling, and NCTS transit filing are P0/P1 blockers. Filing an import declaration in the EU requires knowing WHICH member state's system to file with and WHAT VAT rate to apply. These are not optional details.

---

## 3. China (GACC)

### 3.1 What v5 Addresses

- **GACC registration in Party model** (line 1865): `jurisdiction_identifiers.CN.customs_reg` and `gacc_reg` fields. GACC Decree 248/249 registration for food/cosmetics manufacturers is tracked.

- **Adapter pattern for China** (lines 2643): Filing workflow (manifest -> declaration -> channel -> assessment -> release -> close), duty formula (import duty + VAT + consumption tax), party registration (customs registration, GACC for food/cosmetics).

- **Two-layer status mapping** (lines 2660-2664): Chinese statuses mapped correctly.

- **Jurisdiction-specific adjudication** (line 1309): `clearance.adjudication.cn.*` with `ciq-inspected`, `released`, `cleared`, `rejected`.

### 3.2 Critical Gaps

**P0 -- Channel assignment (green/yellow/red) not modeled on EntryFiling.**

China, like Brazil and India, uses automated channel assignment after declaration submission. The EntryFiling entity has no `channel_assignment` field. While this could go into `jurisdiction_config` JSONB, the absence means the platform cannot track which processing channel a Chinese declaration was assigned to, which determines whether document review or physical inspection is required.

**Fix required:** Add a generalized `risk_channel: string?` field to EntryFiling, or define it in the jurisdiction adapter's status machine. The `jurisdiction_status` field on EntryFiling could store the raw channel status (e.g., "green_channel", "yellow_channel"), but this needs to be explicitly defined in the CN adapter's status mapping.

**P0 -- Consumption tax not modeled in TariffCalculation.**

China's duty structure includes customs duty + VAT (13% or 9%) + consumption tax (for specific goods like alcohol, tobacco, cosmetics, luxury cars). The `TariffCalculation` entity has `duty_rate`, `total_duty`, and `line_items`. Consumption tax could go into `line_items`, but it is not specified. This is the same structural gap as EU VAT -- the architecture says "different formulas" but the entity model does not have explicit fields for non-US tax components.

**Fix required:** The TariffCalculation `line_items` array must be documented to support jurisdiction-specific tax types. Define a standard taxonomy: `{program: "CN_IMPORT_DUTY" | "CN_VAT" | "CN_CONSUMPTION_TAX", rate, amount, legal_citation}`. This is not a model change -- it is a documentation/contract requirement for the adapter implementations.

**P1 -- Trade mode not modeled.**

China distinguishes between general trade, processing trade, cross-border e-commerce (CBEC), and other trade modes. Trade mode affects duty treatment (CBEC has special consolidated tax at 70% of normal). No `trade_mode` field exists on EntryFiling or in jurisdiction_config specification.

**P1 -- Post-release supervision not modeled.**

In China, "released" (released from customs) is distinct from "customs closed" (all procedures complete). Goods can be released but still under post-clearance audit, price verification, or supervision for months. The platform_status mapping shows `cleared` = released and `released` = customs closed, but there is no mechanism to track the post-release supervision period or its end date.

**P2 -- CIQ inspection requirements not integrated.**

Since CIQ merged into GACC in 2018, inspection and quarantine requirements are part of customs processing. CIQ-regulated products (food, cosmetics, medical devices, chemicals) require additional inspection certificates. The Compliance & Screening domain's PGA model is US-centric (FDA, EPA, CPSC). A CN adapter needs equivalent agency mappings for CIQ categories.

**P2 -- 10-digit Chinese tariff code.**

The Product entity has `hs_codes` per jurisdiction, which could hold the 10-digit Chinese code. However, the HS code format varies: US uses 10-digit HTS, EU uses 10-digit TARIC, China uses 10-digit national extension, Brazil uses 8-digit NCM, India uses 8-digit ITC(HS). The classification engine must produce jurisdiction-specific code formats. This is acknowledged conceptually but not operationalized in the Classification entity.

### 3.3 China Assessment

**Not production-ready.** The Party model covers GACC registration. But the missing channel assignment field, consumption tax in duty calculations, and trade mode classification are P0/P1 blockers. A Chinese import declaration cannot be processed without knowing the trade mode and assessing consumption tax for applicable goods.

---

## 4. Brazil (Receita Federal)

### 4.1 What v5 Addresses

- **RADAR in Party model** (line 1862): `jurisdiction_identifiers.BR.cnpj`, `radar_type` (express/limited/unlimited), `radar_utilization`. RADAR is correctly identified as gating ALL Brazil imports.

- **Adapter pattern for Brazil** (lines 2644): Filing workflow (product catalog -> LPCO -> DUIMP -> parametrization -> NF-e), duty formula (cascading: II -> IPI -> PIS/COFINS -> ICMS -> AFRMM), party registration (RADAR + CNPJ).

- **Two-layer status mapping** (lines 2660-2664): Brazil statuses mapped: registrada -> pending, yellow/red/grey -> under_review, interrupted -> held, desembaracada -> cleared, entregue -> released.

- **Jurisdiction-specific adjudication** (line 1307): `clearance.adjudication.br.*` with `parameterized`, `cleared`, `rejected`.

### 4.2 Critical Gaps

**P0 -- Cascading duty calculation is NOT structurally supported.**

This is the single most important jurisdiction gap in the architecture. Brazil's tax calculation is cascading -- each tax is computed on a base that includes prior taxes:

```
II = CIF x rate
IPI = (CIF + II) x rate
PIS = circular formula (includes itself in base)
COFINS = circular formula (includes itself in base)
ICMS = (CIF + II + IPI + PIS + COFINS) / (1 - ICMS_rate) x ICMS_rate
```

The `TariffCalculation` entity stores `duty_rate`, `total_duty`, `line_items`. The `line_items` array can hold multiple tax lines, but the array structure does not enforce or represent the DEPENDENCY CHAIN between tax lines. The duty calculator adapter for Brazil must compute taxes in a specific order because each tax's base includes the previous taxes. The US formula is simple addition; Brazil's is iterative with a circular component (PIS/COFINS).

The architecture says "different FORMULA per jurisdiction" in the adapter interface, but the `DutyEngine` interface is not defined with enough specificity to ensure cascading calculation is supported. If `DutyEngine.calculate()` returns a flat list of `{program, rate, amount}` items, the cascading dependency is lost.

**Fix required:** The `DutyEngine` interface should specify that the output includes computation order and base amounts:
```
line_items: [{
  program: string,
  rate: decimal,
  base_amount: decimal,     # What this tax was calculated ON
  amount: decimal,
  order: int,               # Computation sequence (1=first, cascading)
  includes_in_base: [string] # Which prior programs are included in this base
}]
```

This is backward-compatible with the existing `line_items` structure but adds the fields needed for cascading taxes. Without it, the Brazil adapter cannot produce auditable duty calculations.

**P0 -- ICMS state-specific rates have no destination state field.**

ICMS rates vary by Brazilian state (4% interstate for imports, 7-18% by state, with 17-18% most common). The TariffCalculation entity has `destination_country` but no `destination_state` or `destination_region`. Without this, ICMS cannot be calculated correctly.

**Fix required:** Add `destination_subdivision: string?` to TariffCalculation (ISO 3166-2 code, e.g., "BR-SP" for Sao Paulo). This field is also useful for Canada (provincial HST/PST rates) and India (IGST vs SGST/CGST for intra-state).

**P0 -- DUIMP product catalog registration not modeled.**

As of 2026, DUIMP is the mandatory import declaration system in Brazil, replacing the legacy DI. DUIMP requires that EVERY product be pre-registered in the importer's Product Catalog (Catalogo de Produtos) with NCM code, attributes, and manufacturer details BEFORE a declaration can reference it. The architecture's Product Catalog domain aligns conceptually, but there is no Brazil-specific product registration workflow, no `catalogo_produto_id` field, and no mechanism to validate product catalog registration before filing.

**Fix required:** The BR adapter's `party_validator` (or a new `pre_filing_validator` interface method) must verify that products are registered in the Brazilian catalog before allowing DUIMP filing. Add `catalogo_produto_id: string?` to EntryFiling's jurisdiction_config for Brazil.

**P1 -- NF-e post-clearance requirement not modeled.**

Every movement of goods within Brazil requires a Nota Fiscal Eletronica (NF-e). After customs clearance, the importer must issue an import NF-e (CFOP 3101/3102) before goods can legally move domestically. This is a mandatory post-clearance step that does not exist in other jurisdictions. The architecture has no NF-e entity, no post-clearance document generation trigger, and no status tracking for NF-e issuance.

**Fix required:** Add a post-clearance step to the BR adapter's filing workflow. The `filing_workflow` state machine should include a post-clearance state (e.g., `cleared -> nfe_pending -> complete`). The Document Management domain should support NF-e as a document type with generation/validation capability.

**P1 -- Grey channel (customs valuation investigation) not distinguished.**

Brazil's unique grey channel triggers a full customs valuation investigation including price verification, transfer pricing analysis, and origin investigation. This can delay clearance by weeks or months. The two-layer status model maps grey to `under_review`, but grey is fundamentally different from yellow/red -- it involves a formal investigation process with different timelines and resolution paths. The Exception Management domain should create a specific exception type for grey channel assignments.

**P1 -- AFRMM (ocean freight surcharge) not modeled.**

Brazil charges AFRMM (8% of ocean freight) on all ocean imports. This is mode-specific (ocean only) similar to US HMF but at a much higher rate. No field or line item type exists for AFRMM.

**P2 -- NCM code format validation.**

Brazil uses the NCM (Nomenclatura Comum do Mercosul), which is an 8-digit code based on HS but with a national extension at the 8th digit. The Classification entity stores `hs_code` as a string, which can hold NCM codes. However, the classification engine needs to produce NCM-formatted codes for Brazil, not just HS-6 or HTS-10 codes.

### 4.3 Brazil Assessment

**Not production-ready.** The RADAR tracking in Party Management is good progress, but the cascading duty calculation gap is a fundamental blocker. Brazil has the most complex import tax structure of all target jurisdictions -- you cannot file a DUIMP without computing II, IPI, PIS/COFINS, and ICMS in the correct cascading sequence with state-specific ICMS rates. The product catalog registration prerequisite is also a P0 blocker.

---

## 5. India (CBIC / ICEGATE)

### 5.1 What v5 Addresses

- **IEC in Party model** (line 1864): `jurisdiction_identifiers.IN.iec` and `dsc_class`. IEC is mandatory for all Indian imports/exports.

- **Adapter pattern for India** (lines 2645): Filing workflow (IGM carrier -> BoE -> RMS -> assessment -> OOC), duty formula (sequential: BCD -> SWS -> IGST), party registration (IEC, DSC).

- **Two-layer status mapping** (lines 2660-2664): India statuses mapped: filed -> pending, under_assessment/query_raised -> under_review, on_hold -> held, out_of_charge -> cleared.

- **Jurisdiction-specific adjudication** (line 1308): `clearance.adjudication.in.*` with `duty-assessed`, `out-of-charge`, `cleared`, `rejected`.

### 5.2 Critical Gaps

**P0 -- Bill of Entry type not modeled.**

India has three BoE types with fundamentally different duty and processing implications:
- **Home Consumption (White BoE):** Direct clearance, duty paid upfront
- **Warehousing (Yellow BoE):** Into bonded warehouse, duty deferred
- **Ex-Bond BoE:** Withdrawal from warehouse, duty paid at removal

The EntryFiling entity has `entry_type` which is US-centric (01, 02, 03...). India's BoE types are structurally different from US entry types. There is no `boe_type` field on EntryFiling or in the IN adapter specification.

**Fix required:** The `entry_type` field should be generalized or the jurisdiction adapter should provide entry type validation per jurisdiction. Add `boe_type: "home_consumption" | "warehousing" | "ex_bond"` to EntryFiling's jurisdiction_config for India. The BR adapter needs similar treatment for DI vs DUIMP type, and MX for 76 pedimento clave codes.

**P0 -- IGST rate not modeled as explicit field.**

India's duty formula is BCD + SWS (10% of BCD) + IGST. The IGST rate changed with GST 2.0 (effective September 22, 2025) from 5 slabs to 3 primary rates: 5%, 18%, 40%. The TariffCalculation entity can hold IGST as a line item, but the IN adapter's duty calculator needs access to the current IGST rate schedule. The architecture does not specify where India-specific reference data (BCD schedule, IGST rates) is stored or how the IN duty calculator accesses it.

**Fix required:** The adapter pattern should include a `reference_data: ReferenceDataSource` interface (already specified in the deep-dive recommendation at line 1050). The IN adapter's reference data source must include: ITC(HS) 8-digit codes, BCD schedule, IGST rates (GST 2.0), SWS rate, anti-dumping duty orders, and CBE weekly exchange rates.

**P1 -- Out of Charge (OOC) timestamp not explicit.**

OOC is the single most important milestone in Indian customs clearance -- it is the moment goods are released. The `jurisdiction_status` field can store "out_of_charge" as a raw string, and the `platform_status` maps it to `cleared`. However, there is no explicit `ooc_timestamp` field for tracking when OOC was granted. For SLA monitoring and analytics, this timestamp needs to be captured.

**Fix required:** The `AdjudicationDecision.decided_at` timestamp effectively captures OOC timing when `decision_type = "cleared"` and `jurisdiction = "IN"`. This may be sufficient. But for the broker workbench, a jurisdiction-specific milestone timestamp (OOC for India, desembaraco for Brazil, customs_released for EU) should be surfaced.

**P1 -- e-Sanchit document upload not integrated.**

India's e-Sanchit system requires paperless document upload in PDF/A format with UDIN/DRN references. The Document Management domain stores documents in S3-compatible storage, which is good. But there is no mechanism for generating e-Sanchit-compatible document packages or tracking UDIN/DRN references per document.

**P1 -- SWIFT 2.0 PGA integration not modeled.**

India's SWIFT (Single Window Interface for Facilitating Trade) integrates PGA agencies similar to the US PGA model. NOC (No Objection Certificate) status from each participating government agency must be tracked. The Compliance & Screening domain's PGA model lists US agencies (FDA, EPA, CPSC). An IN adapter needs equivalent mappings for Indian PGAs (FSSAI for food, CDSCO for drugs, BIS for standards, PQ for plant quarantine, AQ for animal quarantine).

**P2 -- Shipping Bill (export) not modeled.**

India's export process uses Shipping Bills with four types (Free, Dutiable, Drawback, Ex-Bond) and requires a Let Export Order (LEO) as the final clearance milestone. No export filing entity exists.

**P2 -- Section 54 transshipment bond.**

India's domestic transit procedure (Section 54) requires a transshipment bond with bank guarantee. The in-bond model on HU notes India Section 54 as equivalent (line 710) but the bond guarantee requirement is not modeled.

### 5.3 India Assessment

**Not production-ready.** The IEC tracking is good. But the missing BoE type field, IGST rate reference data, and e-Sanchit integration are P0/P1 blockers. An Indian Bill of Entry cannot be filed without specifying the BoE type, and duty cannot be calculated without the current IGST rate schedule.

---

## 6. Mexico (SAT / Aduana)

### 6.1 What v5 Addresses

- **RFC and Padron in Party model** (line 1866): `jurisdiction_identifiers.MX.rfc`, `padron_type`, `immex_auth`. Padron de Importadores is correctly identified as mandatory.

- **Adapter pattern for Mexico** (lines 2651): Filing workflow (pre-validation -> pedimento -> payment -> modulacion), key difference (mandatory pre-validation, 76 pedimento clave codes).

- **Two-layer status mapping**: Mexico statuses can map through the generic adapter pattern, though specific Mexican statuses (pre_validated, modulacion_libre, reconocimiento_primero, reconocimiento_segundo) are not listed in the status mapping table.

### 6.2 Critical Gaps

**P0 -- Pedimento clave codes not modeled.**

Mexico has 76 pedimento clave codes in 176 scenarios. The most common are A1 (definitive import/export), IN (temporary import for IMMEX), RT (return of temporary), V1 (virtual transfer). The `entry_type` field on EntryFiling is US-centric. There is no `pedimento_clave` field. A Mexican customs declaration CANNOT be filed without specifying the pedimento clave.

**Fix required:** Add `pedimento_clave: string?` to EntryFiling's jurisdiction_config for Mexico. The MX adapter must validate the clave against the 76 valid codes and determine the applicable customs regime.

**P0 -- Mandatory pre-validation not modeled.**

Mexico uniquely requires that pedimentos pass through an authorized pre-validator (Prevalidadora) BEFORE submission to SAT/SAAI. No other target jurisdiction has this requirement. The filing workflow in the architecture lists pre-validation as a step, but there is no status or entity for tracking pre-validation results. The filing_status enum (draft -> pending_broker_approval -> submitted...) does not include a pre-validation state.

**Fix required:** The MX adapter's filing_workflow state machine must include `pre_validation_pending -> pre_validated` states before `submitted`. The `jurisdiction_status` field can track this, but it needs to be defined in the MX adapter's status mapping.

**P0 -- DTA (processing fee) not modeled.**

Mexico's DTA (Derecho de Tramite Aduanero) is a processing fee of 0.8% of customs value for formal entries, with fixed rates for certain regimes. This is similar to US MPF but at a higher rate and with regime-specific variations. No DTA field or line item type exists.

**P1 -- IVA (Mexican VAT) not modeled explicitly.**

Mexico charges 16% IVA on (CIF + IGI + DTA). Like EU VAT and China VAT, this is a consumption tax applied at the border. The TariffCalculation line_items can hold this, but it needs to be specified.

**P1 -- COVE (electronic value voucher) not modeled.**

Mexico's COVE system requires electronic value documentation linked to the pedimento. As of 2026, the Customs Value Declaration and annexes must be submitted electronically via VUCEM (effective April 1, 2026). No COVE reference field exists.

**P1 -- Modulacion (random selection) result not modeled.**

After duty payment, Mexican customs performs modulacion (random selection). Results are: desaduanamiento libre (free clearance), reconocimiento aduanero (first examination), or segundo reconocimiento (second examination). This is analogous to channel assignment in Brazil/India/China but happens AFTER duty payment, not before. The timing difference matters for the filing workflow.

**P2 -- IMMEX program not operationalized.**

IMMEX (maquiladora) is Mexico's temporary import program for manufacturers. Goods imported under IMMEX (pedimento clave IN) are duty-free but must be re-exported or transferred within a specified period. The Party model tracks `immex_auth` but there is no lifecycle tracking for temporary import obligations (re-export deadlines, transfer tracking).

### 6.3 Mexico Assessment

**Not production-ready.** The RFC/Padron tracking is good. But the missing pedimento clave, mandatory pre-validation step, and DTA fee are P0 blockers. A Mexican import cannot be filed without the pedimento clave code, and the filing cannot be submitted without passing through a pre-validator.

---

## 7. Canada (CBSA)

### 7.1 What v5 Addresses

- **BN in Party model** (line 1868): `jurisdiction_identifiers.CA.bn` and `rpp_status`. Business Number is mandatory for all Canadian imports. RPP (Release Prior to Payment) status determines whether the importer can get release before paying duties.

- **Adapter pattern for Canada** (lines 2652): Filing workflow (ACI carrier -> release -> CAD 5 days -> statement), key difference (two-step like US, monthly statement payment).

### 7.2 Critical Gaps

**P0 -- CAD type codes not modeled.**

Canada has multiple Commercial Accounting Declaration types: A/AB (standard), B (over-the-counter), C (cash entry), F (courier low value), V (voluntary), 10/13/20/21 (warehouse operations). The `entry_type` field on EntryFiling is US-centric. There is no `cad_type` field. A Canadian import cannot be accounted for without specifying the CAD type.

**Fix required:** Add `cad_type: string?` to EntryFiling's jurisdiction_config for Canada. The CA adapter must validate the type.

**P0 -- CARM portal integration not specified.**

As of January 1, 2026, all CARM transition measures have ended. CARM is now the mandatory system for Canadian customs. The architecture mentions CARM in the adapter pattern but does not specify the integration approach. CBSA has published CARM API v5.1 specifications (May 2025). The `electronic_system: SystemConnector` interface in the adapter pattern covers this conceptually, but the CA adapter needs CARM API integration as a production requirement. Without CARM integration, the platform cannot file entries in Canada.

**Fix required:** Specify CARM API integration in the CA adapter's electronic_system. Include support for the CARM Client Portal (CCP) for manual operations and CARM API for automated EDI integration.

**P1 -- Provincial tax variation not modeled.**

Canada has GST (5% federal) plus provincial taxes that vary: HST provinces (13-15% combined), PST provinces (separate provincial tax), and no-PST jurisdictions (Alberta, territories). Like Brazil's ICMS, the destination province determines the applicable rate. The `destination_subdivision` field proposed for Brazil would also serve Canada.

**P1 -- ACI (Advance Commercial Information) not modeled as filing entity.**

Canada's ACI is the equivalent of US ISF / EU ENS -- pre-arrival cargo data filed by the carrier. The architecture has ISFFiling (US ocean) and ENSFiling (EU). There is no `ACIFiling` entity for Canada. ACI is mandatory for all modes.

**P1 -- Monthly statement payment cycle.**

Canada uses a monthly Statement of Account for duty payment, similar to US PMS but standard (not opt-in). The FinancialRecord entity has `pms_enrollment` and `pms_statement_period` but these are US PMS-specific fields. Canada's statement cycle needs its own configuration.

**P1 -- Importer of Record liability changes.**

As of January 1, 2026, Section 17 amendments to Canada's Customs Act expand IOR liability. The entity identified as importer at accounting time is now jointly liable with the owner for duties, taxes, and post-accounting adjustments. The ImporterOfRecord entity should capture this joint liability relationship.

### 7.3 Canada Assessment

**Not production-ready.** The BN and RPP tracking are good. The two-step filing model (release -> accounting within 5 days) maps reasonably well to the US entry/release -> entry summary pattern. But the missing CAD type codes, CARM integration specification, and provincial tax modeling are P0/P1 blockers.

---

## 8. Cross-Cutting Validation

### 8.1 Jurisdiction Adapter Pattern

**Interface completeness: 85%.**

The adapter interface defines 7 concerns:
1. `filing_workflow: StateMachine` -- correctly identified, per-jurisdiction workflows specified
2. `duty_calculator: DutyEngine` -- correctly identified, but interface needs `base_amount` and computation order in output
3. `status_machine: StatusMachine` -- correctly implemented via two-layer model
4. `party_validator: PartyValidator` -- correctly identified, jurisdiction_identifiers on Party entity
5. `document_requirements: DocSpec` -- correctly identified, but document_type enum is US-centric
6. `electronic_system: SystemConnector` -- correctly identified, but integration specifications per jurisdiction are missing
7. `checklist_generator: ChecklistSpec` -- added in v5, correctly identified

**Missing interface methods:**

| Method | Purpose | Needed For |
|--------|---------|------------|
| `entry_type_validator` | Validate jurisdiction-specific entry/declaration type codes | MX pedimento clave, CA CAD type, IN BoE type |
| `pre_filing_validator` | Check prerequisites before filing (product catalog for BR, pre-validation for MX) | BR DUIMP, MX pedimento |
| `post_clearance_handler` | Handle post-clearance obligations (NF-e for BR, post-release supervision for CN) | BR, CN |
| `transit_procedure` | Define transit document types and workflows (T1/T2 for EU, DTA for BR, Section 54 for IN) | EU, BR, IN |
| `export_declaration` | Define export filing requirements (ECS for EU, DU-E for BR, Shipping Bill for IN) | ALL jurisdictions |

### 8.2 Two-Layer Status Model

**Assessment: Sound design, mostly complete mapping.**

The two-layer model (`jurisdiction_status: string` + `platform_status: enum`) is well-designed. The platform_status enum (`pending | under_review | held | cleared | released | rejected`) covers the fundamental states across all jurisdictions.

**Status mapping gaps:**

| Jurisdiction | Raw Status | Current Mapping | Issue |
|-------------|-----------|-----------------|-------|
| BR | grey_channel | under_review | Grey is a formal investigation, not just review. Should it map to `held`? |
| CN | customs_closed (already released) | released | Correct, but the gap between `cleared` (released) and `released` (customs closed) may cause confusion. |
| IN | query_raised | under_review | Correct, but query_raised has a response deadline that `under_review` does not convey. |
| MX | reconocimiento_segundo (second exam) | under_review | A second examination is escalation beyond first exam. The platform_status does not distinguish escalation levels. |
| EU | customs_debt_confirmed | released | This conflates "goods released" with "debt confirmed." What if goods are released but debt is disputed? |

**Recommendation:** The 6-value platform_status enum is sufficient for cross-jurisdiction normalization. The nuances above are handled by the `jurisdiction_status` raw string, which is the correct approach. No change needed to the enum. However, document the mapping rules explicitly in each adapter so that consumers know what `under_review` means in each jurisdiction's context.

### 8.3 Declaration Model (Entry/Release vs Entry Summary)

**Assessment: US model well-implemented. Generalization needs work.**

The v5 architecture explicitly models the US two-step process with `release_number`, `release_status`, `summary_status`, `summary_due_date`. Lines 1056-1059 note non-US equivalents:
- EU: ENS (carrier) -> import declaration -> customs released -> customs debt confirmed
- India: Prior BoE -> assessed BoE
- Brazil: DI/DUIMP registration -> parametrization

**Structural question:** Do these non-US workflows actually map to the entry/release vs entry summary split?

| Jurisdiction | Step 1 (pre-arrival / initial) | Step 2 (post-arrival / detailed) | Maps to release/summary? |
|-------------|-------------------------------|----------------------------------|--------------------------|
| US | Entry/release (CBP 3461) | Entry summary (CBP 7501) | Yes -- designed for this |
| EU | ENS (carrier pre-arrival) | Import declaration (H1-H7) | **No** -- ENS is carrier-filed, not broker. Import declaration is a single filing, not a two-step broker process. |
| EU (simplified) | Simplified declaration | Supplementary declaration (within 10 days) | **Partially** -- conceptually similar but different data elements and legal basis |
| India | Prior BoE (pre-arrival) | Assessed BoE (post-RMS) | **Partially** -- same document, but assessment changes it. Not a separate filing. |
| Brazil | DUIMP registration | Parametrization result | **No** -- parametrization is the authority's action, not a second broker filing |
| China | Pre-declaration | Formal declaration | **Partially** -- two phases but processed as one unit |
| Canada | Release request | CAD (within 5 days) | **Yes** -- very similar to US model |
| Mexico | Pre-validation | Pedimento submission | **No** -- pre-validation is by a third party, not a two-step broker process |

**Finding:** The entry/release vs entry summary split is specifically a US/Canada pattern. Other jurisdictions have their own multi-step processes, but they do not map cleanly to `release_status` / `summary_status`. The v5 architecture correctly models the US/CA pattern. For other jurisdictions, the filing workflow state machine in the adapter should define its own multi-step process without trying to reuse `release_status`/`summary_status`.

**Recommendation:** The `release_status` and `summary_status` fields should be documented as US/CA-specific. Non-US jurisdictions should use `jurisdiction_status` and the adapter's filing_workflow state machine to represent their multi-step processes. This is already the intended design, but the documentation should be explicit.

### 8.4 Party Model

**Assessment: Comprehensive. Minor additions needed.**

The Party entity with `jurisdiction_identifiers` JSONB covers:
- US: EIN, bond holder status
- EU: EORI
- BR: CNPJ, RADAR type/utilization
- IN: IEC, DSC class
- CN: customs registration, GACC registration
- MX: RFC, padron type, IMMEX authorization
- CA: BN, RPP status

**Gaps:**

| Jurisdiction | Missing Registration | Impact |
|-------------|---------------------|--------|
| BR | `inscricao_estadual` (state-level tax ID for ICMS credit) | P2 -- needed for ICMS credit claims |
| IN | AEO tier (T1/T2/T3/LO) | P2 -- `trust_programs` array can hold this, but not in `jurisdiction_identifiers` for fast lookup |
| MX | Padron Sectorial (sector-specific) | P2 -- `padron_type` is "general" or "sectorial" but which sector is not tracked |
| CA | CARM portal registration status | P1 -- as of 2026, CARM registration is mandatory; should be a gating check |
| EU | AEO type (AEOC/AEOS/combined) | P2 -- `trust_programs` holds this |

**POA model: Well-designed.** The PowerOfAttorney entity correctly models continuous vs per-entry, jurisdiction-specific, revocable relationships. This was a P1 gap in the broker review -- resolved.

**IOR model: Well-designed.** The ImporterOfRecord entity correctly models self, designated_broker, and carrier_as_ior types with bond linkage. This was a P1 gap in the broker review -- resolved.

### 8.5 Duty/Tax Formulas

**Assessment: Formulas are correctly specified in the adapter definitions but the TariffCalculation entity needs structural additions.**

| Jurisdiction | Formula | Correct? | Modeled? |
|-------------|---------|----------|----------|
| US | duty + MPF + HMF + Section 301/232/IEEPA + AD/CVD | Yes | Yes -- `duty_line_items`, `mpf`, `hmf`, `adcvd_cash_deposit` |
| EU | customs duty + AD/CVD + member-state VAT | Yes | **Partial** -- no explicit VAT fields, no member state lookup |
| BR | II + IPI + PIS/COFINS + ICMS + AFRMM (cascading) | Yes | **No** -- cascading computation not structurally supported |
| IN | BCD + SWS (10% of BCD) + IGST | Yes | **Partial** -- can use line_items but no explicit SWS/IGST fields |
| CN | customs duty + VAT + consumption tax | Yes | **Partial** -- can use line_items but no explicit consumption tax fields |
| MX | IGI + DTA + IVA + IEPS | Yes | **Partial** -- can use line_items but no explicit DTA/IVA fields |
| CA | customs duty + SIMA + excise + GST + provincial tax | Yes | **Partial** -- can use line_items but no provincial tax lookup |

**Key structural issue:** The `TariffCalculation.line_items` array is flexible enough to hold any jurisdiction's tax components. However:

1. The array does not enforce computation ORDER (critical for cascading taxes)
2. The array does not specify the BASE AMOUNT for each line item (critical for auditing)
3. The array does not indicate which prior line items are included in each base (critical for cascading)

**Recommendation:** Add `base_amount: decimal` and `computation_order: int` to each line item. Add `destination_subdivision: string?` to TariffCalculation for state/province-level tax lookup (BR ICMS, CA provincial, IN state-level).

### 8.6 Transit Procedures

**Assessment: US in-bond well-modeled. Non-US transit undermodeled.**

| Jurisdiction | Transit Procedure | Modeled? | Entity Location |
|-------------|-------------------|----------|----------------|
| US | IT/IE/T&E | Yes | HU.in_bond_detail with bond_type, ports, transit limits |
| EU | T1/T2 (NCTS) | **Noted but not modeled** | Comment on line 709 references EU T1/T2, no filing entity |
| BR | DTA | **Noted but not modeled** | Comment on line 710 references Brazil DTA |
| IN | Section 54 | **Noted but not modeled** | Comment on line 710 references India Section 54 |
| MX | Transito interno (H1), Transito internacional (T1) | **Not mentioned** | -- |
| CN | Transit between bonded zones | **Not mentioned** | -- |
| CA | In-transit (similar to US in-bond) | **Not mentioned** | -- |

**The `in_bond_detail` JSONB on HU is US-specific** (bond_type: IT | IE | TE). For EU T1/T2, the bond type concept maps but the guarantee mechanism is different (comprehensive guarantee vs US customs bond). For Brazil DTA, the transit document structure is different.

**Recommendation:** Generalize `in_bond_detail` to `transit_detail` with jurisdiction-aware transit type:
```
transit_detail: {
  jurisdiction: string,
  transit_type: string,          # US: IT/IE/TE, EU: T1/T2/TIR, BR: DTA, IN: Section_54
  origin_port: string,
  destination_port: string,
  filing_reference: string?,
  guarantee_type: string?,       # US: bond, EU: comprehensive_guarantee/cash_deposit/TIR_carnet
  guarantee_amount: decimal?,
  transit_time_limit_days: int,
  ...
}
```

Add a `TransitFiling` entity to Declaration Management for jurisdictions that require a separate transit declaration (EU NCTS, Brazil DTA).

### 8.7 Consolidation/HU Model

**Assessment: Universally sound.**

The HU -> Consolidation nesting hierarchy is transport-oriented, not jurisdiction-specific. It works across all jurisdictions:

- Air: HAWB (HU) -> MAWB (Consolidation) -- universal
- Ocean: HBL (HU) -> MBL (Consolidation) -> Container -> Vessel -- universal
- Ground: Pro# (HU) -> Manifest (Consolidation) -> Trailer -- universal

**Minor gaps:**
- `consolidation_type` enum should add `trailer` (Mexico cross-border trucking) and `rail_wagon` (EU/China rail freight) per the deep-dive review. These are P3 items.
- `container_type` should add reefer (20RF/40RF) and specialized containers (open top, flat rack, tank) per the broker review. P3.

### 8.8 Export Declarations

**Assessment: Major gap. Export is not modeled as a first-class operation.**

The architecture focuses almost entirely on import clearance. Export declarations are mandatory in ALL target jurisdictions:

| Jurisdiction | Export System | Export Document | Mandatory? |
|-------------|-------------|----------------|------------|
| US | AES (Automated Export System) | EEI (Electronic Export Information) | Yes, for goods >$2,500 or licensed |
| EU | ECS (Export Control System) | Export Declaration + MRN | Yes, for all exports |
| BR | Siscomex Exportacao / DU-E | DU-E | Yes, for all exports |
| IN | ICEGATE | Shipping Bill + LEO | Yes, for all exports |
| CN | Single Window | Export Customs Declaration | Yes, for all exports |
| MX | VUCEM | Pedimento (A1 export clave) | Yes, for all exports |
| CA | CBSA | CAED via CERS | Yes, for exports >$2,000 |

The HU entity has `export_clearance: not_required | pending | declared | cleared` which tracks export status. But there is no export declaration entity, no export filing workflow, and no export-specific checklist. For bidirectional trade lanes, the platform must handle BOTH the export declaration in the origin country AND the import declaration in the destination country.

**Recommendation:** Add an `ExportDeclaration` entity to Declaration Management with jurisdiction-specific filing workflows. The adapter pattern should include an `export_declaration` interface method. This is a P1 item for non-US jurisdictions (where export declarations are universally mandatory) and P2 for US (where EEI is only required above $2,500 or for licensed goods).

### 8.9 Bidirectional Trade Lanes

**Assessment: The architecture supports bidirectional trade conceptually but does not operationalize it.**

For a US->CN shipment: the platform needs to file a US export (EEI via AES) AND a CN import (customs declaration via Single Window). These are two separate filings with two separate authorities, governed by two different sets of regulations.

The EntryFiling entity is currently focused on imports. Adding an ExportDeclaration entity (per 8.8 above) would enable bidirectional support. The key requirement is that a single shipment/HU can have BOTH an export declaration (in the origin country) and an import declaration (in the destination country), each with its own jurisdiction adapter.

The HU entity's dual `export_clearance` and `import_clearance` fields already support this pattern at the status level. The missing piece is the filing entities.

---

## 9. Remaining Gaps (Prioritized)

### P0 -- Must fix before non-US deployment

| # | Gap | Jurisdiction(s) | Effort | Fix |
|---|-----|-----------------|--------|-----|
| 1 | EU member state sub-jurisdictions | EU | Medium | Add `sub_jurisdiction` to EntryFiling; parameterize EUAdapter by member state |
| 2 | VAT/consumption tax not explicit in TariffCalculation | EU, CN, MX | Low | Document line_items taxonomy; add `vat_rate`/`vat_amount` to TariffCalculation |
| 3 | Cascading duty calculation not structurally supported | BR | Medium | Add `base_amount`, `computation_order` to line_items; implement cascading engine |
| 4 | ICMS/provincial destination subdivision missing | BR, CA | Low | Add `destination_subdivision` to TariffCalculation |
| 5 | DUIMP product catalog prerequisite not modeled | BR | Medium | Add pre-filing validator to adapter interface; add catalogo_produto_id |
| 6 | BoE type / pedimento clave / CAD type not modeled | IN, MX, CA | Low | Add `declaration_subtype: string?` to EntryFiling or jurisdiction_config |
| 7 | Mexico mandatory pre-validation not in filing workflow | MX | Low | Add pre_validation state to MX adapter's filing_workflow |
| 8 | Channel assignment (CN/BR/IN) not a field | CN, BR, IN | Low | Use `jurisdiction_status` for channel; document in adapter mapping |

### P1 -- Required for production use

| # | Gap | Jurisdiction(s) | Effort | Fix |
|---|-----|-----------------|--------|-----|
| 9 | NCTS transit filing entity (T1/T2) | EU | Medium | Add TransitFiling entity or generalize in_bond_detail |
| 10 | EU customs procedure codes | EU | Low | Add to jurisdiction_config |
| 11 | NF-e post-clearance requirement | BR | Medium | Add post-clearance workflow step to BR adapter |
| 12 | Export declaration entity | ALL | High | Add ExportDeclaration to Declaration Management |
| 13 | CARM API integration specification | CA | Medium | Define CARM API v5.1 integration in CA adapter |
| 14 | ACI pre-arrival filing entity | CA | Low | Add ACIFiling or generalize ENSFiling for CA |
| 15 | Mexico DTA fee and IVA | MX | Low | Add to MX duty calculator line_items |
| 16 | India e-Sanchit integration | IN | Medium | Add document format requirements to IN adapter |
| 17 | China trade mode | CN | Low | Add to CN adapter jurisdiction_config |
| 18 | Document type enum expansion | ALL | Low | Add ENS, SAD, EUR.1, T1/T2, DUIMP, NF-e, BoE, Shipping Bill, pedimento |

### P2 -- Important for completeness

| # | Gap | Jurisdiction(s) | Effort | Fix |
|---|-----|-----------------|--------|-----|
| 19 | Grey channel special handling (BR) | BR | Low | Add grey_channel exception type |
| 20 | AFRMM ocean freight surcharge (BR) | BR | Low | Add to BR duty calculator |
| 21 | COVE electronic value voucher (MX) | MX | Low | Add reference field |
| 22 | Modulacion result tracking (MX) | MX | Low | Add to MX adapter status mapping |
| 23 | CIQ agency mappings (CN) | CN | Medium | Add CN PGA equivalents |
| 24 | India SWIFT 2.0 PGA integration | IN | Medium | Add IN PGA agency mappings |
| 25 | Jurisdiction-specific abandonment timelines | ALL | Low | Parameterize GO deadline per jurisdiction |
| 26 | IMMEX/MOOWR/inward processing lifecycle | MX, IN, EU | High | Add special regime tracking |
| 27 | AEO mutual recognition operationalization | ALL | Medium | Apply trust program benefits in adjudication logic |
| 28 | Reference data tables per jurisdiction | ALL | High | Add TARIC, NCM, ITC(HS), CN tariff schedules |
| 29 | Protest/appeal timelines per jurisdiction | ALL | Low | Parameterize per adapter |
| 30 | Consolidation type expansion (trailer, rail_wagon) | MX, EU, CN | Low | Add to enum |

---

## 10. Final Assessment

### What the Architecture Gets Right

1. **Domain model is jurisdiction-agnostic at the transport level.** The HU -> Consolidation nesting, movement cascade, border_crossing_mode, and document hierarchy (HAWB/MAWB) work identically across jurisdictions. This is the hard part, and it is done correctly.

2. **Adapter pattern is the right abstraction.** The interface definition (filing_workflow, duty_calculator, status_machine, party_validator, document_requirements, electronic_system, checklist_generator) covers the major axes of jurisdiction variation. This was the #1 recommendation from the deep-dive review, and it has been adopted.

3. **Two-layer status model is elegant and correct.** Storing the raw jurisdiction status alongside a normalized platform status allows each jurisdiction to have its own status vocabulary while giving the platform a uniform interface. No code changes to add a jurisdiction -- just configuration. This scales.

4. **Party Management is comprehensive.** The jurisdiction_identifiers JSONB, trust_programs, POA, and IOR models cover the party requirements for all 7 target jurisdictions. This was a P0 gap in both prior reviews -- resolved.

5. **US model is production-ready.** The entry/release split, expanded entry types, in-bond, liquidation lifecycle, PMS, AD/CVD tracking, and bond model are all operationally correct.

6. **Event choreography is jurisdiction-neutral.** The event-driven intelligence flow (ProductCreated -> ClassificationProduced -> TariffCalculated -> ComplianceScreened -> checklist populated) works identically regardless of jurisdiction. Adding a new jurisdiction does not require new event types.

### What Still Needs Work

1. **The adapter implementations are sketched, not specified.** The adapter interface is defined, but the per-jurisdiction implementations exist only as one-line summaries in a table. The actual state machine definitions, duty calculator formulas, document requirement lists, and electronic system specifications need to be fleshed out for each jurisdiction.

2. **The TariffCalculation entity is structurally US-centric.** Despite the adapter pattern acknowledging different formulas, the entity model has US-specific fields (`mpf`, `hmf`, `adcvd_cash_deposit`) as top-level fields while non-US tax components are expected to live in the generic `line_items` array without structural support for cascading, base amounts, or computation order.

3. **Export declarations are absent.** For a platform supporting bidirectional trade lanes, the complete absence of export filing entities is a significant gap. Every jurisdiction requires export declarations for goods leaving its territory.

4. **EU member state complexity is under-represented.** The architecture treats "EU" as a single jurisdiction, but in practice, filing requires engaging with national customs systems that have different interfaces, formats, and operational procedures.

5. **Transit procedures beyond US in-bond are not modeled.** EU NCTS (T1/T2), Brazil DTA, and India Section 54 are acknowledged in comments but have no filing entities or status machines.

### Readiness Verdict

| Jurisdiction | Production Ready? | Blocking Items |
|-------------|-------------------|----------------|
| **US** | **Yes** | None -- comprehensive model |
| **EU** | **No** | Member state sub-jurisdiction, VAT rates, NCTS transit |
| **CN** | **No** | Channel assignment, consumption tax, trade mode |
| **BR** | **No** | Cascading duty calculation, ICMS state rates, DUIMP product catalog, NF-e |
| **IN** | **No** | BoE type, IGST rates, e-Sanchit |
| **MX** | **No** | Pedimento clave, pre-validation, DTA |
| **CA** | **No** | CAD type, CARM integration, provincial tax |

**The architecture is ready for multi-jurisdiction DEVELOPMENT but not multi-jurisdiction DEPLOYMENT.** The adapter pattern provides the right framework. The domain models need the P0 additions identified above (8 items). These are structural additions to existing entities, not architectural redesigns. Estimated effort for all P0 items: 3-4 weeks of focused work.

**The recommended implementation order:**
1. Add `destination_subdivision`, `base_amount`/`computation_order` to TariffCalculation (enables BR, CA, all non-US duty calcs)
2. Add `declaration_subtype` to EntryFiling (enables IN BoE, MX pedimento, CA CAD types)
3. Add `sub_jurisdiction` to EntryFiling (enables EU member states)
4. Implement BR duty calculator with cascading engine (most complex duty formula)
5. Implement EU duty calculator with member-state VAT lookup
6. Add pre-filing validation hooks to adapter interface (enables BR product catalog, MX pre-validation)
7. Add ExportDeclaration entity (enables bidirectional trade lanes)
8. Add TransitFiling entity (enables EU NCTS, BR DTA)

After these additions, the architecture will be genuinely multi-jurisdiction-ready.

---

*Review completed 2026-02-09. All findings are grounded in current regulatory requirements as verified through official government sources. Web searches were conducted to confirm ICS2 Release 3 status, CARM implementation status, DUIMP mandate timeline, IEEPA tariff rates, and NCTS Phase 5 deployment status.*

*Regulatory sources consulted:*
- [EU ICS2 Import Control System 2](https://taxation-customs.ec.europa.eu/customs/customs-security/import-control-system-2_en) -- ICS2 v3 mandatory as of February 3, 2026
- [CBSA CARM](https://www.cbsa-asfc.gc.ca/services/carm-gcra/menu-eng.html) -- CARM transition measures ended December 31, 2025
- [India ICEGATE](https://www.icegate.gov.in/) -- Bill of Entry filing system with GST integration
- [Brazil DUIMP / Portal Unico](https://www.gov.br/siscomex/pt-br) -- DUIMP mandatory as of 2026
- [China GACC Registration](https://gacc.agency/) -- Decree 248/249 GACC registration for food manufacturers
- [Mexico VUCEM Customs Value Declaration](https://www.bakermckenzie.com/en/insight/publications/2026/01/mexico-customs-value-declaration) -- Electronic value declaration effective April 1, 2026
- [US Section 321 De Minimis Suspension](https://www.cbp.gov/sites/default/files/2025-08/factsheet_suspension_of_duty-free_de_minimis_treatment.pdf) -- Suspended for all countries since August 29, 2025
- [EU NCTS Phase 5](https://taxation-customs.ec.europa.eu/customs/union-customs-code/ucc-work-programme_en) -- Deployed across 29 countries
