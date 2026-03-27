# CPSC eFiling Mandate — Regulatory Analysis & Platform Integration

## 1. Regulatory Background

The U.S. Consumer Product Safety Commission (CPSC) approved a final rule on December 18, 2024 (published in the Federal Register on January 8, 2025) amending **16 CFR Part 1110**. The rule mandates electronic filing (eFiling) of certificate of compliance data for imported consumer products through CBP's Automated Commercial Environment (ACE).

### Legal Basis

- **Section 14 of the CPSA**: Requires certificates of compliance for regulated consumer products
- **Section 17(a) of the CPSA**: Authorizes refusal of admission for products not accompanied by required certificates
- **CPSIA of 2008**: Consumer Product Safety Improvement Act — established third-party testing and Children's Product Certificate requirements

### Mandatory Dates

| Date | Milestone |
|------|-----------|
| Now – July 2026 | Voluntary eFiling stage (no penalties, mistakes don't affect risk scores) |
| **July 8, 2026** | **Mandatory eFiling** for all imported CPSC-regulated consumer products |
| January 8, 2027 | Mandatory eFiling for goods entering through Foreign Trade Zones (FTZs) |

### Scope

- Applies to **all imported finished products** subject to a mandatory CPSC safety standard
- Includes **de minimis shipments** (under $800)
- CPSC PGA agency code in ACE: **`CPS`**

---

## 2. What Must Be Filed

### Three PGA Message Set Types

#### 2a. Full PGA Message Set (7 data elements per entry line)

| # | Data Element | Format / Notes |
|---|-------------|----------------|
| 1 | **Product identification** | One of 7 types: GTIN, SKU, UPC, Model#, Serial#, Registered#, Alternate ID |
| 2 | **Citation codes** | Specific CPSC mandatory standards (16 CFR parts, ASTM standards) |
| 3 | **Manufacture date** | Month and year |
| 4 | **Manufacture place** | Factory name, address, country |
| 5 | **Most recent test date** | Date of last compliance testing |
| 6 | **Testing lab** | Name and contact info (must be CPSC-accepted for children's products) |
| 7 | **Certificate attestation** | Checkbox confirming a valid Section 14 certificate exists |
| 8* | **Testing exclusions** | *(new in final rule)* Any testing exclusions relied upon |

#### 2b. Reference PGA Message Set (3 identifiers per entry line)

| Field | Purpose |
|-------|---------|
| Certifier ID | Identifies the importer/certifier in CPSC's Product Registry |
| Product ID (Primary) | Identifies the product in the registry |
| Version ID | Identifies which version of the certificate data to use |

Pre-requires loading full certificate data into the **CPSC Product Registry** (a standalone system, not connected to ACE).

#### 2c. Disclaim PGA Message Set

Filed for shipments that do NOT contain CPSC-regulated products. Optional but improves the importer's CPSC risk score.

### Certificate Types

- **Children's Product Certificate (CPC)**: Required for products designed for children 12 and under. Must be based on third-party testing by a CPSC-accepted laboratory. Covers: lead content, phthalates, small parts, flammability, ASTM F963, etc.
- **General Certificate of Conformity (GCC)**: Required for regulated general-use consumer products. First-party testing acceptable.

---

## 3. Data Transmission Architecture

### Flow

```
Importer (prepares certificate data)
    │
    ▼
Customs Broker (compiles entry in broker software)
    │
    ▼
CBP ACE System (PGA Message Set attached to entry filing)
    │
    ▼
CPSC (receives data, feeds into Risk Assessment Methodology)
```

Data is **not filed directly to CPSC**. It rides alongside the existing CBP entry filing (3461/7501) as a PGA Message Set segment within ACE.

### Actor Responsibilities

| Actor | Responsibility |
|-------|---------------|
| **Importer/Shipper** | Obtains/maintains certificates (CPC or GCC), provides certificate data or Registry IDs to broker. Optionally pre-loads data into CPSC Product Registry. |
| **Customs Broker** | Incorporates PGA Message Set data into ACE entry filing. Transmits via existing ACE EDI/ABI connection. |
| **CBP ACE** | Receives entry + PGA data, routes CPSC segments to CPSC. Initially issues warnings (not rejections) for missing data. |
| **CPSC** | Receives data, feeds into RAM (Risk Assessment Methodology) which scores shipments for targeting/examination. |

### CPSC Product Registry (for Reference method)

Three upload methods:
- **Web UI** — manual entry through CPSC portal
- **CSV batch upload** — bulk load certificate data
- **API** — programmatic upload (specs in CPSC eFiling Document Library)

Self-registration closes at 2,000 participants.

---

## 4. Products Regulated by CPSC

### HS Code Ranges (from PGA_MAPPINGS in pga.py)

| HS Range | Product Category | Requirements | Citation |
|----------|-----------------|--------------|----------|
| 9503–9505 | Toys and children's products | CPSIA Certificate, Lead/Phthalate Testing, Third-Party Lab Certificate | CPSIA 2008 |
| 9401–9404 | Furniture (flammability) | Flammability Certificate, General Certificate of Conformity | 16 CFR 1633; CPSA §14(a) |
| 6111–6112 | Children's apparel | CPSIA Certificate, Lead Testing | CPSIA §101; 16 CFR 1501 |

### Chapter-Level Flagging (from USAdapter.register_product in us.py)

Broader 2-digit chapter check for consumer products:
- Chapters 39, 40 (plastics, rubber)
- Chapter 42 (leather goods)
- Chapter 44 (wood articles)
- Chapters 61–65 (apparel, textiles, headgear, footwear)
- Chapters 94–96 (furniture, toys, miscellaneous manufactured)

### Keyword Triggers (dual-trigger screening)

Keywords in product descriptions: `"children"`, `"toy"`, `"infant"`, `"child safety"`

---

## 5. Common CPSC Citation Codes

These are the mandatory standards most commonly applicable to imported consumer products:

| Citation | Standard | Product Type |
|----------|----------|-------------|
| ASTM F963 | Standard Consumer Safety Specification for Toy Safety | Toys |
| 16 CFR 1501 | Small Parts for Toys and Children's Articles | Children's products with small parts |
| 16 CFR 1303 | Ban of Lead-Containing Paint | Products with painted surfaces |
| CPSIA §101 | Lead Content Limits (100 ppm) | Children's products |
| CPSIA §108 | Phthalate Content Limits | Children's toys and child care articles |
| 16 CFR 1610 | Standard for Flammability of Clothing Textiles | Apparel |
| 16 CFR 1615/1616 | Standard for Flammability of Children's Sleepwear | Children's sleepwear |
| 16 CFR 1633 | Standard for Flammability of Mattress Sets | Mattresses and mattress sets |
| 16 CFR 1219/1220 | Safety Standard for Full/Non-Full Size Baby Cribs | Cribs |
| ASTM F2057 | Standard Safety Specification for Clothing Storage Units | Dressers, clothing storage |

---

## 6. Current Platform State

### What Exists

| Component | Location | What It Does |
|-----------|----------|-------------|
| PGA Screener | `app/engines/e3_compliance/pga.py` | Flags CPSC for HS ranges 9503-9505, 9401-9404, 6111-6112 + keyword triggers |
| USAdapter | `shared/jurisdiction/us.py` | Chapter-level CPSC flagging (chapters 39-96) during product registration |
| PGAFlag schema | `shared/schemas/analysis.py` | Pydantic model with `agency`, `requirement`, `form_number`, `mandatory` |
| Product model | `domains/product_catalog/models.py` | Has `catalog_registrations` JSONB column (currently unstructured) |
| SourceLocation | `shared/schemas/products.py` | Has `country_code`, `facility_name`, `facility_type` (no address) |
| Document Requirements UI | `shipper/components/DocumentRequirements.tsx` | Per-document status tracking, generation, upload, and completion |
| Product Detail UI | `shipper/screens/ShipperProductDetail.tsx` | Route analysis with PGA flag surfacing |

### What's Missing

| Gap | Detail |
|-----|--------|
| PGA Message Set data model | No schema for the 7 required certificate data elements |
| ACE PGA integration | Platform flags CPSC but doesn't structure data for ACE electronic filing |
| CPSC Product Registry | No support for Reference PGA Message Set flow |
| Disclaim flow | No ability to file a "disclaim" for non-CPSC-regulated shipments |
| Citation code catalog | Requirements are free-text ("CPSIA Certificate") not mapped to actual 16 CFR codes |
| Testing lab validation | No model for CPSC-accepted third-party labs |
| De minimis coverage | Unclear if platform handles de minimis entries for PGA screening |
| Testing exclusions | No data field for "testing exclusions relied upon" |
| Facility address | `SourceLocation` missing `address` field needed for manufacture place |
| Certificate data capture UI | No form for shippers to enter the 7 data elements |

---

## 7. Integration Design

### Data Model: `catalog_registrations.cpsc`

The existing `catalog_registrations` JSONB column on the Product model is the natural home for CPSC certificate data:

```json
{
  "cpsc": {
    "certifier_id": "CERT-001234",
    "product_id_primary": "PROD-567890",
    "version_id": "V3",

    "product_identifier": {
      "type": "GTIN",
      "value": "00012345678905"
    },
    "citation_codes": [
      "16 CFR 1501",
      "ASTM F963",
      "16 CFR 1303"
    ],
    "manufacture_date": "2026-01",
    "manufacture_place": {
      "facility_name": "Shenzhen Toy Factory",
      "address": "123 Industrial Rd, Shenzhen",
      "country_code": "CN"
    },
    "last_test_date": "2025-11-15",
    "testing_lab": {
      "name": "Intertek Testing Services",
      "contact": "labs@intertek.com",
      "cpsc_accepted": true
    },
    "certificate_exists": true,
    "testing_exclusions": [],
    "certificate_type": "CPC"
  }
}
```

### System Flow

```
Product Catalog                    Declaration Management
┌──────────────────────┐          ┌──────────────────────────┐
│ Product              │          │ Entry Filing              │
│  ├─ hs_codes         │──PGA──→ │  ├─ line items            │
│  ├─ source_locations │ screen  │  │   ├─ hs_code           │
│  ├─ manufacturer_id  │         │  │   ├─ product_id (FK)   │
│  └─ catalog_         │         │  │   └─ pga_messages: [   │
│     registrations    │─────────│──│       { agency: "CPS",  │
│       └─ cpsc: {     │  pull   │  │         method: "REF",  │
│           certifier_id,│ cert  │  │         certifier_id,   │
│           product_id,  │ data  │  │         product_id,     │
│           version_id,  │       │  │         version_id }    │
│           citations,   │       │  │     ]                   │
│           ...        │         │  └─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │
│       }              │         │                            │
└──────────────────────┘         └──────────┬─────────────────┘
                                            │ ACE transmission
                                            ▼
                                 ┌──────────────────────────┐
                                 │ CBP ACE (PGA Message Set) │
                                 │  → CPSC receives data     │
                                 └──────────────────────────┘
```

### Shipper Surface Integration

**Product creation/edit** — when PGA screening detects CPSC:
- Show a "CPSC Compliance" card with status (complete / incomplete / not required)
- Step-by-step form for the 7 data elements
- Suggest citation codes based on HS code
- Validate children's products have a CPSC-accepted testing lab
- Save to `catalog_registrations.cpsc`

**Shipment/order creation** — when a CPSC-flagged product is added:
- Check if product has complete CPSC certificate data
- If complete: green checkmark, data flows to broker automatically
- If incomplete: warn/block, link back to product to complete certificate
- Allow per-shipment overrides for batch-specific fields (manufacture date, test date)

**DocumentRequirements panel** — existing component:
- Add "CPSC Certificate of Compliance" as a document type
- "Generate & Attach" auto-fills from `catalog_registrations.cpsc`
- "Review Fields" shows the 7 data elements for verification

### Implementation Changes Required

| Layer | Change |
|-------|--------|
| `SourceLocation` schema | Add `address: str` field |
| `catalog_registrations` | Define typed Pydantic schema for CPSC certificate data |
| `ProductCreate`/`ProductResponse` | Expose `cpsc_certificate` as nested model |
| PGA screener output | Return specific citation codes (not just free-text requirements) |
| Declaration management | Pull `catalog_registrations.cpsc` at entry time, format for ACE PGA Message Set |
| Product Registry sync | Optional API integration to push/pull from CPSC Product Registry |
| Validation | Children's products require `testing_lab.cpsc_accepted == true` when `certificate_type == "CPC"` |
| Frontend: Product Detail | New CPSC Certificate Data panel with guided form |
| Frontend: Order creation | Certificate completeness check when adding CPSC-flagged products |

---

## 8. References

- [CPSC eFiling Portal](https://www.cpsc.gov/eFiling)
- [CPSC eFiling FAQ](https://www.cpsc.gov/FAQ/eFiling-Frequently-Asked-Questions-FAQ)
- [Federal Register Final Rule (Jan 8, 2025)](https://www.federalregister.gov/documents/2025/01/08/2024-30826/certificates-of-compliance)
- [CPSC CATAIR Implementation Guide V2.2](https://www.cpsc.gov/s3fs-public/eFiling-CATAIR-IG-V2_2.pdf)
- [Covington & Burling: CPSC Revises Requirements](https://www.cov.com/en/news-and-insights/insights/2025/01/cpsc-revises-requirements-for-certificates-of-compliance)
- [Arnold & Porter: eFiling Final Countdown](https://www.arnoldporter.com/en/perspectives/blogs/consumer-products-and-retail-navigator/2025/07/cpsc-efiling-requirement-takes-effect-one-year-from-today)
- [C.H. Robinson: CPSC eFiling Mandate](https://www.chrobinson.com/en-us/resources/blog/cpsc-efiling-mandate/)
- [Stinson LLP: Get Your Certificates Ready](https://www.stinson.com/newsroom-publications-get-your-certificates-ready-for-the-cpsc-efiling-rule-for-imported-consumer-products)
- [Federal Register: PGA Message Set Beta Pilot (2022)](https://www.federalregister.gov/documents/2022/06/10/2022-12477/electronic-filing-of-certain-certificate-of-compliance-data-announcement-of-pga-message-set-test-and)
- [Federal Register: PGA Message Set Expansion (2024)](https://www.federalregister.gov/documents/2024/06/04/2024-12194/electronic-filing-of-certificate-of-compliance-data-announcement-of-expansion-of-pga-message-set)
