# Data Ingestion & Seeding -- Implementation Details

Deep implementation reference for the data ingestion pipeline. Covers the base ingester pattern, each seeder's data sources and volumes, dependency ordering, upsert strategies, and how to add new seeders.

Source files:
- `backend/data/ingestion/base.py`
- `backend/data/ingestion/seed_all.py`
- `backend/data/ingestion/seed_*.py` (22 files)
- `Makefile` (seed targets)

---

## Table of Contents

1. [Base Ingester Pattern](#1-base-ingester-pattern)
2. [Master Orchestrator](#2-master-orchestrator)
3. [Individual Seeder Implementations](#3-individual-seeder-implementations)
4. [Data Sources and Volumes](#4-data-sources-and-volumes)
5. [Upsert Strategies](#5-upsert-strategies)
6. [Dependency Graph](#6-dependency-graph)
7. [Adding a New Seeder](#7-adding-a-new-seeder)

---

## 1. Base Ingester Pattern

**Source:** `backend/data/ingestion/base.py`

### Synchronous Engine

All seeders use synchronous SQLAlchemy (psycopg2 driver) for performance:

```python
def get_sync_engine():
    url = os.environ.get(
        "DATABASE_SYNC_URL",
        "postgresql://clearance:clearance_dev@localhost:5440/clearance_engine",
    )
    return create_engine(url, echo=False, pool_pre_ping=True)
```

The sync URL uses `postgresql://` (not `postgresql+asyncpg://`) because seeders run outside the async event loop. The default connects to the Docker Compose PostgreSQL on port 5440.

### Session Factory

```python
def get_session() -> Session:
    engine = get_sync_engine()
    factory = sessionmaker(bind=engine)
    return factory()
```

Each seeder creates a fresh session, commits on success, and rolls back on error.

### Environment Loading

```python
_engine_env = Path(__file__).resolve().parent.parent.parent.parent / ".env"
_root_env = Path(__file__).resolve().parent.parent.parent.parent.parent / ".env"

for _p in (_engine_env, _root_env):
    if _p.exists():
        load_dotenv(_p)
        break
```

Resolves to `clearance-engine/.env` first, then `clearance_vibe/.env`. This supports both running from the backend directory and from the project root.

### Logging

Three helpers for consistent output:

```python
log_seed_start("HTSUS Chapters")     # [SEED] Starting HTSUS Chapters ...
log_seed_result("HTSUS Chapters", 99) # [SEED] HTSUS Chapters: 99 records upserted
log_seed_error("HTSUS Chapters", exc) # [SEED] ERROR HTSUS Chapters: ... (stderr)
```

---

## 2. Master Orchestrator

**Source:** `backend/data/ingestion/seed_all.py`

### Execution

```bash
cd backend && python -m data.ingestion.seed_all
# or
make seed
```

### Phase Structure

The orchestrator imports all seeders lazily (at call time, not module load) and organises them into 5 phases:

```python
seeders: list[tuple[str, callable]] = []

# Phase 1: Base tariff schedules
seeders.extend([
    ("HTSUS Chapters", seed_htsus_chapters),
    ("HTSUS (US)", seed_htsus),
    ("EU Tariff (TARIC)", seed_eu_tariff),
    ("China Tariff (CCTC)", seed_cn_tariff),
    ("Brazil Tariff (NCM/TEC)", seed_br_tariff),
    ("India Tariff (ITC-HS)", seed_in_tariff),
])

# Phase 2: Trade remedy overlays
seeders.extend([
    ("Section 301 Lists", seed_section301),
    ("Section 232 Scope", seed_section232),
    ("IEEPA Rates", seed_ieepa),
    ("AD/CVD Orders", seed_adcvd),
])

# Phase 3: Reference data
seeders.extend([
    ("Fee Schedule", seed_fees),
    ("Tax Rates", seed_taxes),
    ("Exchange Rates", seed_exchange_rates),
    ("CROSS Rulings", seed_cross_rulings),
    ("Tax Regime Templates", seed_tax_regimes),
])

# Phase 4: Compliance data
seeders.extend([
    ("Restricted Parties", seed_restricted_parties),
    ("PGA Mappings", seed_pga),
    ("FTA Rules (USMCA)", seed_fta_rules),
])

# Phase 5: Operational data
seeders.extend([
    ("Regulatory Signals", seed_regulatory_signals),
    ("Demo Shipments", seed_demo_shipments),
])
```

### Execution Loop

```python
succeeded = 0
failed = 0

for name, seed_fn in seeders:
    try:
        seed_fn()
        succeeded += 1
    except Exception as e:
        log_seed_error(name, e)
        failed += 1
        continue  # Continue with remaining seeders

if failed > 0:
    sys.exit(1)
```

Failures are non-blocking -- the pipeline continues to give maximum data coverage even if one seeder fails.

---

## 3. Individual Seeder Implementations

### 3.1 seed_htsus_chapters.py

**Purpose:** Chapter-level metadata for the US Harmonized Tariff Schedule.

**Data format:** Inline tuple list:
```python
# (chapter_number, title, section_number, section_title)
("01", "Live Animals", "I", "Live Animals; Animal Products")
```

**SQL:** INSERT with ON CONFLICT on chapter number.

### 3.2 seed_htsus.py

**Purpose:** US tariff lines with verified MFN duty rates from USITC HTS 2024 Revision 14 / 2025 baseline.

**Data format:** Inline tuple list:
```python
# (hs_code, hs_code_display, description, indent, mfn_rate_pct, mfn_rate_text, unit, chapter, jurisdiction)
("6109.10.00", "6109.10", "T-shirts, singlets, tank tops, cotton, knitted", 0, 16.5, "16.5%", "doz/kg", "61", "US")
```

**Coverage:** 120+ representative lines spanning chapters 01-99 including all demo product HS codes.

**SQL:** INSERT with ON CONFLICT on (hs_code, jurisdiction).

### 3.3 seed_eu_tariff.py / seed_cn_tariff.py / seed_br_tariff.py / seed_in_tariff.py

**Purpose:** Multi-jurisdiction tariff schedules for EU, China, Brazil, and India.

**Pattern:** Same tuple format as seed_htsus but with jurisdiction-specific rates and additional fields:
- EU: Common External Tariff rates
- CN: MFN rates plus temporary preferential rates
- BR: TEC rates plus ICMS state-level rates
- IN: BCD rates plus IGST and Social Welfare Surcharge

### 3.4 seed_section301.py

**Purpose:** Section 301 additional tariffs on Chinese goods.

**Data:** HS codes across Lists 1-4 with additional duty rates:
- List 1: 25% on $34B of goods (July 2018)
- List 2: 25% on $16B of goods (August 2018)
- List 3: 25% on $200B of goods (escalated May 2019)
- List 4A: 7.5% on $120B of goods (reduced February 2020)
- List 4B: Not implemented (suspended)

**SQL:** INSERT with ON CONFLICT on (hs_code, list_number).

### 3.5 seed_section232.py

**Purpose:** Section 232 tariffs on steel and aluminum.

**Scope:**
- Steel (Chapter 72-73): 25% ad valorem
- Aluminum (Chapter 76): 10% ad valorem
- Derivative articles: Various rates

**Data:** HS codes with product descriptions, rate percentages, and country exclusion lists.

### 3.6 seed_ieepa.py

**Purpose:** IEEPA (International Emergency Economic Powers Act) reciprocal tariff rates.

**Data:** Per-country additional tariff rates with effective dates and product scope.

### 3.7 seed_adcvd.py

**Purpose:** Antidumping (AD) and countervailing duty (CVD) orders.

**Data:** Active orders with case numbers, country, product descriptions, applicable HS codes, duty rates, and status.

### 3.8 seed_fees.py

**Purpose:** CBP fee schedules.

**Coverage:**
- Merchandise Processing Fee (MPF): 0.3464% ad valorem, min $31.67, max $614.35
- Harbor Maintenance Fee (HMF): 0.125% of value for imports
- Other applicable fees by entry type

### 3.9 seed_taxes.py

**Purpose:** Tax rates across all supported jurisdictions.

**Coverage:**
- US: No federal VAT; state sales tax rates (0-10.25%)
- EU: VAT rates by member state (15-27%)
- CN: VAT 13% (standard), 9% (reduced)
- BR: ICMS rates by state (7-25%), IPI, PIS/COFINS
- IN: IGST rates (5-28%)

### 3.10 seed_tax_regimes.py

**Purpose:** Jurisdiction computation templates that define the tax/fee calculation sequence.

**Records:** 5 regimes (US, EU, CN, BR, IN)

**Example (US):**
```
1. Base duty (MFN)
2. Section 301 overlay (if CN origin)
3. Section 232 overlay (if steel/aluminum)
4. IEEPA overlay (if applicable)
5. AD/CVD overlay (if applicable)
6. MPF (0.3464%, min/max)
7. HMF (0.125% for ocean entries)
```

### 3.11 seed_exchange_rates.py

**Purpose:** Currency FX rates with USD as base.

**Coverage:** 50+ currencies including EUR, GBP, CNY, JPY, KRW, INR, BRL, MXN, etc.

### 3.12 seed_cross_rulings.py

**Purpose:** CBP CROSS (Customs Rulings Online Search System) binding classification rulings.

**Source:** Selected rulings from rulings.cbp.gov representing key classification scenarios.

**Data format:**
```python
{
    "ruling_number": "N332145",
    "ruling_date": "2023-06-15",
    "hs_code": "8708.99.81",
    "tariff_provision": "HTSUS 8708.99.8180",
    "product_description": "Automotive sensor assembly...",
    "issuing_office": "National Commodity Specialist Division",
    "ruling_text": "Dear [Importer]:\n\nThis is in response to..."
}
```

Full ruling text is stored for vector search by the E5 exception engine.

### 3.13 seed_restricted_parties.py

**Purpose:** Denied and restricted party screening reference data.

**List sources:**

**OFAC SDN (30+ entities):**
- Russian financial institutions (Sberbank, VTB, Gazprombank, Alfa-Bank)
- Russian defense/energy (Rosneft, Rostec, United Aircraft Corporation)
- Iranian entities (NIOC, CBI, IRGC, Mahan Air)
- Chinese military-affiliated (CETC, CNNC, Hikvision)
- North Korean (KOMID, RGB)
- Drug cartels (Sinaloa, CJNG)
- Venezuelan, Syrian, Belarusian entities

**BIS Entity List (10+ entities):**
- Huawei Technologies
- SMIC
- YMTC
- CXMT
- SMEE
- National University of Defense Technology

**UFLPA Entity List:**
- Xinjiang-based entities subject to rebuttable presumption

**EU Consolidated Sanctions:**
- Additional entities not on US lists

**Data fields per entity:**
```python
{
    "entity_name": "HUAWEI TECHNOLOGIES CO., LTD.",
    "aliases": ["HUAWEI"],
    "entity_type": "entity",
    "country": "China",
    "programs": ["EL"],
    "source_list_id": "BIS-EL-2019-05-16-1",
    "remarks": "Added May 2019; license required for all items subject to EAR",
    "list_source": "BIS_ENTITY"
}
```

### 3.14 seed_pga.py

**Purpose:** Partner Government Agency requirement mappings.

**Agencies covered:** FDA, EPA, CPSC, NHTSA, FCC, APHIS, ATF, DOE

**Data:** Maps HS code prefixes to agency requirements, filing deadlines, and required documentation.

### 3.15 seed_fta_rules.py

**Purpose:** USMCA (United States-Mexico-Canada Agreement) rules of origin.

**Data:** Rules by HS heading including rule type (tariff shift, regional value content, etc.), qualification criteria, RVC thresholds, and required documentation.

### 3.16 seed_regulatory_signals.py

**Purpose:** Active and upcoming regulatory change tracking.

**Data format:** Follows the RegulatorySignal schema:
```python
{
    "id": "sig_001",
    "title": "Section 301 List 4B Tariff Increase",
    "status": "PROPOSED",
    "description": "USTR is reviewing...",
    "affected_hs_codes": ["8517", "8471", "9403"],
    "effective_date": "2026-03-15",
    "impact_level": "HIGH",
    "tags": ["tariff", "section301", "china"]
}
```

### 3.17 seed_demo_shipments.py

**Purpose:** Pre-built shipment entries covering all lifecycle states.

**Coverage:**
- Multiple trade corridors (CN->US, DE->US, MX->US, JP->US, VN->US, IN->US, KR->US)
- All 7 states (booked, in_transit, at_customs, inspection, held, cleared, delivered)
- Various hold types (uflpa, entity_screening, cf28_cf29, pga, inspection)
- Complete event timelines, waypoints, codes, financials
- Multiple company identities

---

## 4. Data Sources and Volumes

| Seeder | Records | Data Source | Update Frequency |
|---|---|---|---|
| seed_htsus_chapters | 99 | USITC HTS | Annual |
| seed_htsus | 120+ | USITC HTS 2024 Rev 14 | Annual |
| seed_eu_tariff | ~10,000 | EU TARIC | Annual |
| seed_cn_tariff | ~10,000 | China CCTC | Annual |
| seed_br_tariff | ~10,000 | Brazil NCM/TEC | Annual |
| seed_in_tariff | ~10,000 | India ITC-HS | Annual |
| seed_section301 | 5,000+ | USTR | As amended |
| seed_section232 | 500+ | Commerce/BIS | As amended |
| seed_ieepa | 250+ | Executive Orders | As issued |
| seed_adcvd | 1,000+ | ITC/Commerce | Quarterly |
| seed_fees | 50+ | CBP | Annual |
| seed_taxes | 1,000+ | Various tax authorities | Annual |
| seed_tax_regimes | 5 | Internal | As needed |
| seed_exchange_rates | 50+ | Federal Reserve | Daily (manual refresh) |
| seed_cross_rulings | 5,000+ | CBP CROSS | As issued |
| seed_restricted_parties | 30,000+ | OFAC, BIS, DHS, EU | Weekly |
| seed_pga | 1,000+ | FDA/EPA/CPSC/NHTSA | As amended |
| seed_fta_rules | 500+ | USMCA text | As amended |
| seed_regulatory_signals | 100+ | Internal intelligence | Continuous |
| seed_demo_shipments | 100+ | Synthetic | As needed |

---

## 5. Upsert Strategies

All seeders use PostgreSQL `INSERT ... ON CONFLICT ... DO UPDATE` for idempotent re-runs:

### Pattern 1: Single Unique Key

```sql
INSERT INTO tariff_rates (hs_code, jurisdiction, rate_pct, description, ...)
VALUES (:hs_code, :jurisdiction, :rate_pct, :description, ...)
ON CONFLICT (hs_code, jurisdiction) DO UPDATE SET
    rate_pct = EXCLUDED.rate_pct,
    description = EXCLUDED.description,
    updated_at = NOW()
```

### Pattern 2: Composite Key

```sql
INSERT INTO section301_rates (hs_code, list_number, rate_pct, ...)
VALUES (:hs_code, :list_number, :rate_pct, ...)
ON CONFLICT (hs_code, list_number) DO UPDATE SET
    rate_pct = EXCLUDED.rate_pct
```

### Pattern 3: Source List ID

```sql
INSERT INTO restricted_parties (entity_name, list_source, source_list_id, ...)
VALUES (:entity_name, :list_source, :source_list_id, ...)
ON CONFLICT (source_list_id) DO UPDATE SET
    entity_name = EXCLUDED.entity_name,
    aliases = EXCLUDED.aliases
```

This allows seeders to be run repeatedly without duplicating data. New records are inserted, existing records are updated.

---

## 6. Dependency Graph

```
Phase 1 (independent of each other):
    seed_htsus_chapters
    seed_htsus ----+
    seed_eu_tariff |
    seed_cn_tariff |  (base tariff tables must exist)
    seed_br_tariff |
    seed_in_tariff +

Phase 2 (depends on Phase 1 HS codes):
    seed_section301 ---+
    seed_section232    |  (overlay on base tariff)
    seed_ieepa         |
    seed_adcvd ---------+

Phase 3 (independent):
    seed_fees
    seed_taxes
    seed_exchange_rates
    seed_cross_rulings
    seed_tax_regimes

Phase 4 (independent):
    seed_restricted_parties
    seed_pga  (references HS codes from Phase 1)
    seed_fta_rules (references HS codes from Phase 1)

Phase 5 (depends on everything above):
    seed_regulatory_signals
    seed_demo_shipments (may reference products, HS codes)
```

Within each phase, seeders are independent of each other and could theoretically run in parallel. The sequential execution is for simplicity and reliable error reporting.

---

## 7. Adding a New Seeder

### Step 1: Create the Seeder File

Create `backend/data/ingestion/seed_my_data.py`:

```python
"""Seed my new data collection.

Usage:
    python -m data.ingestion.seed_my_data
"""

from __future__ import annotations

from data.ingestion.base import get_session, log_seed_error, log_seed_result, log_seed_start
from sqlalchemy import text

# Define data as module-level constants
MY_DATA = [
    {"field_a": "value1", "field_b": 42},
    {"field_a": "value2", "field_b": 99},
]


def seed():
    log_seed_start("My Data Collection")
    session = get_session()

    try:
        count = 0
        for record in MY_DATA:
            session.execute(
                text("""
                    INSERT INTO my_table (field_a, field_b)
                    VALUES (:field_a, :field_b)
                    ON CONFLICT (field_a) DO UPDATE SET
                        field_b = EXCLUDED.field_b
                """),
                record,
            )
            count += 1

        session.commit()
        log_seed_result("My Data Collection", count)

    except Exception as e:
        session.rollback()
        log_seed_error("My Data Collection", e)
        raise
    finally:
        session.close()


if __name__ == "__main__":
    seed()
```

### Step 2: Register in seed_all.py

Add the import and registration in the appropriate phase:

```python
# In the correct phase section:
from data.ingestion.seed_my_data import seed as seed_my_data

seeders.extend([
    ("My Data Collection", seed_my_data),
])
```

### Step 3: Add Makefile Target

In the root `Makefile`:

```makefile
seed-my-data:
	cd backend && python -m data.ingestion.seed_my_data
```

Add the target to the `.PHONY` line at the top of the Makefile.

### Step 4: Ensure Database Table Exists

If the data requires a new table, create an Alembic migration first:

```bash
make migrate-new msg="add my_table"
# Edit the generated migration
make migrate
```

### Step 5: Test

```bash
# Run individually
make seed-my-data

# Run as part of full pipeline
make seed

# Verify idempotency (run again -- should update, not duplicate)
make seed-my-data
```

### Guidelines

- Keep data as inline constants in the seeder file (no external file dependencies)
- Use `ON CONFLICT ... DO UPDATE` for idempotent re-runs
- Use `log_seed_start()` and `log_seed_result()` for consistent output
- Catch exceptions, rollback, and re-raise
- Always close the session in a `finally` block
- Add `if __name__ == "__main__": seed()` for standalone execution
- Place in the correct dependency phase in `seed_all.py`
