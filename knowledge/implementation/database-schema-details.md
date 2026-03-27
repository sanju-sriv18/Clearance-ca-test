# Database Schema Implementation Details

Implementation-focused reference for the Clearance Intelligence Engine database.
This document is intended for developers working directly with the schema, writing
queries, adding new tables, or troubleshooting data issues.

**ORM Source**: `backend/app/knowledge/models/`
**Migration Source**: `backend/alembic/versions/`
**Database**: PostgreSQL 16 via asyncpg
**Pool**: pool_size=10, max_overflow=20, pool_pre_ping=True

---

## Table of Contents

1. [Complete Column Reference](#complete-column-reference)
2. [JSONB Schemas with Examples](#jsonb-schemas-with-examples)
3. [Relationships Diagram](#relationships-diagram)
4. [Fuzzy Matching with pg_trgm](#fuzzy-matching-with-pg_trgm)
5. [Query Patterns Used in Engines](#query-patterns-used-in-engines)
6. [Adding New Tables and Columns](#adding-new-tables-and-columns)
7. [Data Seeding](#data-seeding)

---

## Complete Column Reference

### Tariff Tables

#### `htsus_chapters`

| Column | SQLAlchemy Type | PG Type | Nullable | Default | Index | Notes |
|--------|----------------|---------|----------|---------|-------|-------|
| id | Integer | serial | No | autoincrement | PK | |
| chapter_number | String(2) | varchar(2) | No | -- | B-tree, Unique | "01"-"99" |
| title | String(500) | varchar(500) | No | -- | -- | |
| section_number | String(10) | varchar(10) | No | -- | -- | Roman numeral |
| section_title | String(500) | varchar(500) | No | -- | -- | |
| notes_text | Text | text | Yes | NULL | -- | Chapter notes |
| jurisdiction | String(10) | varchar(10) | No | "US" | -- | Server default |
| created_at | DateTime(tz) | timestamptz | No | now() | -- | |
| updated_at | DateTime(tz) | timestamptz | No | now() | -- | onupdate |

#### `htsus_headings`

| Column | SQLAlchemy Type | PG Type | Nullable | Default | Index | Notes |
|--------|----------------|---------|----------|---------|-------|-------|
| id | Integer | serial | No | autoincrement | PK | |
| hs_code | String(15) | varchar(15) | No | -- | B-tree | Part of unique constraint |
| hs_code_display | String(20) | varchar(20) | No | -- | -- | Formatted display |
| description | Text | text | No | -- | -- | |
| indent_level | Integer | integer | No | 0 | -- | Server default "0" |
| parent_code | String(15) | varchar(15) | Yes | NULL | -- | Tree navigation |
| mfn_rate_pct | Float | double precision | Yes | NULL | -- | Ad valorem % |
| mfn_rate_specific | String(100) | varchar(100) | Yes | NULL | -- | Specific rate text |
| mfn_rate_text | String(500) | varchar(500) | Yes | NULL | -- | Full rate text |
| general_rate_text | String(500) | varchar(500) | Yes | NULL | -- | Column 1 rate |
| special_rate_text | Text | text | Yes | NULL | -- | GSP/FTA rates |
| unit_of_quantity | String(50) | varchar(50) | Yes | NULL | -- | Statistical UOM |
| chapter_number | String(2) | varchar(2) | No | -- | B-tree | |
| jurisdiction | String(5) | varchar(5) | No | -- | B-tree | US/EU/CN/BR/IN |
| source | String(50) | varchar(50) | No | -- | -- | Data source |
| effective_date | Date | date | Yes | NULL | -- | |
| created_at | DateTime(tz) | timestamptz | No | now() | -- | |
| updated_at | DateTime(tz) | timestamptz | No | now() | -- | onupdate |

**Composite indexes**: `(hs_code, jurisdiction)` B-tree
**Unique constraint**: `uq_htsus_headings_hs_code_jurisdiction` on `(hs_code, jurisdiction)`

#### `section_301_lists`

| Column | SQLAlchemy Type | PG Type | Nullable | Default | Index | Notes |
|--------|----------------|---------|----------|---------|-------|-------|
| id | Integer | serial | No | autoincrement | PK | |
| hs_code | String(15) | varchar(15) | No | -- | B-tree | |
| list_number | String(10) | varchar(10) | No | -- | B-tree | "1","2","3","4a","4b" |
| rate_pct | Float | double precision | No | -- | -- | |
| effective_date | Date | date | No | -- | -- | |
| exclusion | Boolean | boolean | No | false | -- | |
| federal_register_citation | String(200) | varchar(200) | Yes | NULL | -- | |
| notes | Text | text | Yes | NULL | -- | |
| created_at | DateTime(tz) | timestamptz | No | now() | -- | |
| updated_at | DateTime(tz) | timestamptz | No | now() | -- | |

#### `section_232_scope`

| Column | SQLAlchemy Type | PG Type | Nullable | Default | Index | Notes |
|--------|----------------|---------|----------|---------|-------|-------|
| id | Integer | serial | No | autoincrement | PK | |
| hs_code | String(15) | varchar(15) | No | -- | B-tree | |
| product_category | String(20) | varchar(20) | No | -- | B-tree | STEEL/ALUMINUM |
| rate_pct | Float | double precision | No | -- | -- | |
| effective_date | Date | date | No | -- | -- | |
| country_exemptions | JSONB | jsonb | Yes | NULL | -- | See JSONB section |
| federal_register_citation | String(200) | varchar(200) | Yes | NULL | -- | |
| created_at | DateTime(tz) | timestamptz | No | now() | -- | |
| updated_at | DateTime(tz) | timestamptz | No | now() | -- | |

#### `ieepa_rates`

| Column | SQLAlchemy Type | PG Type | Nullable | Default | Index | Notes |
|--------|----------------|---------|----------|---------|-------|-------|
| id | Integer | serial | No | autoincrement | PK | |
| country_code | String(2) | varchar(2) | No | -- | B-tree | ISO alpha-2 |
| country_name | String(100) | varchar(100) | No | -- | -- | |
| rate_pct | Float | double precision | No | -- | -- | |
| effective_date | Date | date | No | -- | -- | |
| executive_order | String(100) | varchar(100) | Yes | NULL | -- | |
| notes | Text | text | Yes | NULL | -- | |
| created_at | DateTime(tz) | timestamptz | No | now() | -- | |
| updated_at | DateTime(tz) | timestamptz | No | now() | -- | |

#### `adcvd_orders`

| Column | SQLAlchemy Type | PG Type | Nullable | Default | Index | Notes |
|--------|----------------|---------|----------|---------|-------|-------|
| id | Integer | serial | No | autoincrement | PK | |
| case_number | String(50) | varchar(50) | No | -- | Unique | e.g. "A-570-967" |
| country | String(100) | varchar(100) | No | -- | B-tree | |
| product_description | Text | text | No | -- | -- | |
| hs_codes | JSONB | jsonb | Yes | NULL | GIN | Array of strings |
| ad_rate_pct | Float | double precision | Yes | NULL | -- | |
| cvd_rate_pct | Float | double precision | Yes | NULL | -- | |
| effective_date | Date | date | No | -- | -- | |
| status | String(20) | varchar(20) | No | "ACTIVE" | B-tree | |
| petitioner | String(300) | varchar(300) | Yes | NULL | -- | |
| federal_register_citation | String(200) | varchar(200) | Yes | NULL | -- | |
| created_at | DateTime(tz) | timestamptz | No | now() | -- | |
| updated_at | DateTime(tz) | timestamptz | No | now() | -- | |

### Compliance Tables

#### `restricted_parties`

| Column | SQLAlchemy Type | PG Type | Nullable | Default | Index | Notes |
|--------|----------------|---------|----------|---------|-------|-------|
| id | Integer | serial | No | autoincrement | PK | |
| entity_name | String(500) | varchar(500) | No | -- | GIN (trgm) | Fuzzy matching |
| aliases | JSONB | jsonb | Yes | NULL | -- | String array |
| list_source | String(30) | varchar(30) | No | -- | B-tree | OFAC_SDN, BIS_ENTITY, etc. |
| entity_type | String(20) | varchar(20) | No | -- | B-tree | entity/individual/vessel |
| country | String(100) | varchar(100) | Yes | NULL | B-tree | |
| addresses | JSONB | jsonb | Yes | NULL | -- | Array of address objects |
| identification | JSONB | jsonb | Yes | NULL | -- | ID documents |
| programs | JSONB | jsonb | Yes | NULL | -- | Sanctions program codes |
| remarks | Text | text | Yes | NULL | -- | |
| effective_date | Date | date | Yes | NULL | -- | |
| source_list_id | String(100) | varchar(100) | Yes | NULL | -- | Original list ID |
| created_at | DateTime(tz) | timestamptz | No | now() | -- | |
| updated_at | DateTime(tz) | timestamptz | No | now() | -- | |

#### `pga_mappings`

| Column | SQLAlchemy Type | PG Type | Nullable | Default | Index | Notes |
|--------|----------------|---------|----------|---------|-------|-------|
| id | Integer | serial | No | autoincrement | PK | |
| agency_code | String(10) | varchar(10) | No | -- | B-tree | FDA, APHIS, etc. |
| agency_name | String(100) | varchar(100) | No | -- | -- | |
| hs_code_start | String(15) | varchar(15) | No | -- | Composite | Range start |
| hs_code_end | String(15) | varchar(15) | No | -- | Composite | Range end |
| description | Text | text | No | -- | -- | |
| requirements | JSONB | jsonb | No | -- | -- | Structured requirements |
| jurisdiction | String(10) | varchar(10) | No | "US" | -- | |
| severity | String(20) | varchar(20) | No | "REQUIRED" | -- | |
| created_at | DateTime(tz) | timestamptz | No | now() | -- | |
| updated_at | DateTime(tz) | timestamptz | No | now() | -- | |

**Composite index**: `(hs_code_start, hs_code_end)`

#### `fta_rules`

| Column | SQLAlchemy Type | PG Type | Nullable | Default | Index | Notes |
|--------|----------------|---------|----------|---------|-------|-------|
| id | Integer | serial | No | autoincrement | PK | |
| agreement | String(30) | varchar(30) | No | -- | B-tree | USMCA, KORUS, etc. |
| chapter_range_start | String(4) | varchar(4) | No | -- | Composite | |
| chapter_range_end | String(4) | varchar(4) | No | -- | Composite | |
| product_category | String(200) | varchar(200) | No | -- | -- | |
| rule_type | String(30) | varchar(30) | No | -- | -- | tariff_shift/rvc/hybrid |
| tariff_shift_requirement | Text | text | Yes | NULL | -- | CC/CTH/CTSH |
| rvc_threshold_pct | Float | double precision | Yes | NULL | -- | |
| rvc_method | String(30) | varchar(30) | Yes | NULL | -- | transaction_value/net_cost |
| additional_requirements | Text | text | Yes | NULL | -- | |
| documentation | JSONB | jsonb | Yes | NULL | -- | Required docs |
| created_at | DateTime(tz) | timestamptz | No | now() | -- | |
| updated_at | DateTime(tz) | timestamptz | No | now() | -- | |

**Composite index**: `(chapter_range_start, chapter_range_end)`

### Reference Tables

#### `cross_rulings`

| Column | SQLAlchemy Type | PG Type | Nullable | Default | Index | Notes |
|--------|----------------|---------|----------|---------|-------|-------|
| id | Integer | serial | No | autoincrement | PK | |
| ruling_number | String(30) | varchar(30) | No | -- | B-tree, Unique | "N123456" |
| ruling_date | Date | date | No | -- | -- | |
| product_description | Text | text | No | -- | -- | |
| hs_code | String(15) | varchar(15) | No | -- | B-tree | |
| tariff_provision | String(200) | varchar(200) | Yes | NULL | -- | |
| ruling_text | Text | text | No | -- | -- | Full text in PG, truncated in Qdrant |
| issuing_office | String(100) | varchar(100) | Yes | NULL | -- | |
| status | String(20) | varchar(20) | No | "ACTIVE" | B-tree | |
| vector_id | String(64) | varchar(64) | Yes | NULL | -- | Qdrant point ID |
| created_at | DateTime(tz) | timestamptz | No | now() | -- | |
| updated_at | DateTime(tz) | timestamptz | No | now() | -- | |

#### `tax_rates`

| Column | SQLAlchemy Type | PG Type | Nullable | Default | Index | Notes |
|--------|----------------|---------|----------|---------|-------|-------|
| id | Integer | serial | No | autoincrement | PK | |
| jurisdiction | String(20) | varchar(20) | No | -- | B-tree | |
| tax_type | String(30) | varchar(30) | No | -- | B-tree | VAT/ICMS/IGST/etc. |
| rate_pct | Float | double precision | No | -- | -- | |
| description | String(300) | varchar(300) | No | -- | -- | |
| applies_to | String(50) | varchar(50) | No | -- | -- | |
| effective_date | Date | date | No | -- | -- | |
| notes | Text | text | Yes | NULL | -- | |
| created_at | DateTime(tz) | timestamptz | No | now() | -- | |
| updated_at | DateTime(tz) | timestamptz | No | now() | -- | |

**Composite index**: `(jurisdiction, tax_type)`

#### `fee_schedule`

| Column | SQLAlchemy Type | PG Type | Nullable | Default | Index | Notes |
|--------|----------------|---------|----------|---------|-------|-------|
| id | Integer | serial | No | autoincrement | PK | |
| fee_type | String(10) | varchar(10) | No | -- | B-tree | MPF/HMF |
| entry_type | String(20) | varchar(20) | No | -- | -- | FORMAL/INFORMAL |
| rate_pct | Float | double precision | Yes | NULL | -- | |
| min_amount | Float | double precision | Yes | NULL | -- | Floor |
| max_amount | Float | double precision | Yes | NULL | -- | Ceiling |
| flat_amount | Float | double precision | Yes | NULL | -- | |
| fiscal_year | Integer | integer | No | -- | B-tree | |
| effective_date | Date | date | No | -- | -- | |
| notes | Text | text | Yes | NULL | -- | |
| created_at | DateTime(tz) | timestamptz | No | now() | -- | |
| updated_at | DateTime(tz) | timestamptz | No | now() | -- | |

#### `exchange_rates`

| Column | SQLAlchemy Type | PG Type | Nullable | Default | Index | Notes |
|--------|----------------|---------|----------|---------|-------|-------|
| id | Integer | serial | No | autoincrement | PK | |
| currency_code | String(3) | varchar(3) | No | -- | B-tree, Unique | ISO 4217 |
| currency_name | String(50) | varchar(50) | No | -- | -- | |
| rate_to_usd | Float | double precision | No | -- | -- | Multiply to get USD |
| rate_from_usd | Float | double precision | No | -- | -- | Multiply USD to get foreign |
| source | String(50) | varchar(50) | No | -- | -- | |
| effective_date | Date | date | No | -- | -- | |
| created_at | DateTime(tz) | timestamptz | No | now() | -- | |
| updated_at | DateTime(tz) | timestamptz | No | now() | -- | |

#### `tax_regime_templates`

| Column | SQLAlchemy Type | PG Type | Nullable | Default | Index | Notes |
|--------|----------------|---------|----------|---------|-------|-------|
| id | Integer | serial | No | autoincrement | PK | |
| jurisdiction | String(10) | varchar(10) | No | -- | B-tree, Unique | US/EU/CN/BR/IN |
| pattern | String(20) | varchar(20) | No | -- | -- | additive/cascading/etc. |
| template | JSONB | jsonb | No | -- | -- | Computation layers |
| description | Text | text | No | -- | -- | |
| notes | Text | text | Yes | NULL | -- | |
| created_at | DateTime(tz) | timestamptz | No | now() | -- | |
| updated_at | DateTime(tz) | timestamptz | No | now() | -- | |

### Operational Tables

#### `regulatory_signals`

| Column | SQLAlchemy Type | PG Type | Nullable | Default | Index | Notes |
|--------|----------------|---------|----------|---------|-------|-------|
| id | Integer | serial | No | autoincrement | PK | |
| title | String(500) | varchar(500) | No | -- | -- | |
| status | String(20) | varchar(20) | No | -- | B-tree | PROPOSED/FINAL/EFFECTIVE |
| description | Text | text | No | -- | -- | |
| affected_hs_codes | JSONB | jsonb | No | -- | GIN | String array |
| affected_countries | JSONB | jsonb | Yes | NULL | -- | String array |
| effective_date | Date | date | Yes | NULL | B-tree | |
| source | String(200) | varchar(200) | No | -- | -- | |
| source_url | String(500) | varchar(500) | Yes | NULL | -- | |
| impact_level | String(10) | varchar(10) | No | -- | B-tree | HIGH/MEDIUM/LOW |
| category | String(30) | varchar(30) | No | -- | B-tree | |
| created_at | DateTime(tz) | timestamptz | No | now() | -- | No updated_at (append-only) |

#### `demo_shipments`

| Column | SQLAlchemy Type | PG Type | Nullable | Default | Index | Notes |
|--------|----------------|---------|----------|---------|-------|-------|
| id | Integer | serial | No | autoincrement | PK | |
| company_id | String(50) | varchar(50) | No | -- | B-tree | |
| company_name | String(200) | varchar(200) | No | -- | -- | |
| entry_number | String(30) | varchar(30) | No | -- | Unique | |
| entry_date | Date | date | No | -- | B-tree | |
| product_description | Text | text | No | -- | -- | |
| hs_code | String(15) | varchar(15) | No | -- | B-tree | |
| origin_country | String(2) | varchar(2) | No | -- | B-tree | |
| destination_country | String(2) | varchar(2) | No | "US" | -- | |
| declared_value | Float | double precision | No | -- | -- | |
| currency | String(3) | varchar(3) | No | "USD" | -- | |
| quantity | Integer | integer | Yes | NULL | -- | |
| unit_of_measure | String(20) | varchar(20) | Yes | NULL | -- | |
| entry_type | String(20) | varchar(20) | No | -- | -- | |
| status | String(20) | varchar(20) | No | "PENDING" | B-tree | |
| compliance_flags | JSONB | jsonb | Yes | NULL | -- | |
| tariff_total | Float | double precision | Yes | NULL | -- | |
| notes | Text | text | Yes | NULL | -- | |
| created_at | DateTime(tz) | timestamptz | No | now() | -- | |
| updated_at | DateTime(tz) | timestamptz | No | now() | -- | |

#### `products`

| Column | SQLAlchemy Type | PG Type | Nullable | Default | Index | Notes |
|--------|----------------|---------|----------|---------|-------|-------|
| id | UUID | uuid | No | uuid4() | PK | |
| name | String(300) | varchar(300) | No | -- | B-tree | |
| description | Text | text | Yes | NULL | -- | |
| hs_code | String(15) | varchar(15) | Yes | NULL | B-tree | Primary HS code |
| hs_codes | JSONB | jsonb | Yes | NULL | -- | Multi-jurisdiction |
| category | String(100) | varchar(100) | Yes | NULL | B-tree | |
| source_locations | JSONB | jsonb | Yes | NULL | -- | |
| unit_value | Float | double precision | Yes | NULL | -- | |
| currency | String(3) | varchar(3) | No | "USD" | -- | |
| weight_kg | Float | double precision | Yes | NULL | -- | |
| cached_analysis | JSONB | jsonb | Yes | NULL | -- | |
| created_at | DateTime(tz) | timestamptz | No | now() | -- | |
| updated_at | DateTime(tz) | timestamptz | No | now() | -- | |

#### `orders`

| Column | SQLAlchemy Type | PG Type | Nullable | Default | Index | Notes |
|--------|----------------|---------|----------|---------|-------|-------|
| id | UUID | uuid | No | uuid4() | PK | |
| status | String(20) | varchar(20) | No | "draft" | B-tree | |
| origin | String(3) | varchar(3) | No | "" | -- | |
| destination | String(3) | varchar(3) | No | "US" | -- | |
| transit_points | JSONB | jsonb | Yes | NULL | -- | |
| analysis | JSONB | jsonb | Yes | NULL | -- | Engine results |
| document_status | JSONB | jsonb | Yes | NULL | -- | Added in 004 |
| shipment_id | UUID (FK) | uuid | Yes | NULL | -- | FK to shipments.id |
| created_at | DateTime(tz) | timestamptz | No | now() | B-tree | |
| updated_at | DateTime(tz) | timestamptz | No | now() | -- | |

**FK**: `orders.shipment_id` -> `shipments.id` (SET NULL on delete)

#### `order_line_items`

| Column | SQLAlchemy Type | PG Type | Nullable | Default | Index | Notes |
|--------|----------------|---------|----------|---------|-------|-------|
| id | Integer | serial | No | autoincrement | PK | |
| order_id | UUID (FK) | uuid | No | -- | B-tree | CASCADE on delete |
| product_id | UUID | uuid | No | -- | B-tree | Logical ref, no FK |
| product_name | String(300) | varchar(300) | No | -- | -- | Denormalized |
| quantity | Integer | integer | No | 1 | -- | |
| source_country | String(3) | varchar(3) | No | "" | -- | |
| source_facility | String(300) | varchar(300) | No | "" | -- | |
| unit_value | Float | double precision | No | 0 | -- | |
| currency | String(3) | varchar(3) | No | "USD" | -- | |
| line_total | Float | double precision | No | 0 | -- | |
| hs_code | String(15) | varchar(15) | Yes | NULL | -- | |
| duty_amount | Float | double precision | Yes | NULL | -- | |
| compliance_status | String(20) | varchar(20) | Yes | NULL | -- | |
| tariff_breakdown | JSONB | jsonb | Yes | NULL | -- | |

**FK**: `order_line_items.order_id` -> `orders.id` (CASCADE on delete)

#### `shipments`

| Column | SQLAlchemy Type | PG Type | Nullable | Default | Index | Notes |
|--------|----------------|---------|----------|---------|-------|-------|
| id | UUID | uuid | No | uuid4() | PK | |
| product | String(500) | varchar(500) | No | -- | -- | Denormalized |
| product_id | UUID | uuid | Yes | NULL | -- | Logical ref |
| order_id | UUID (FK) | uuid | Yes | NULL | Unique B-tree | SET NULL, one-to-one |
| origin | String(3) | varchar(3) | No | -- | B-tree | |
| destination | String(3) | varchar(3) | No | -- | B-tree | |
| status | String(20) | varchar(20) | No | "in_transit" | B-tree | |
| carrier | String(200) | varchar(200) | No | "" | -- | |
| tracking_number | String(100) | varchar(100) | Yes | NULL | -- | |
| declared_value | Float | double precision | No | 0 | -- | |
| events | JSONB | jsonb | Yes | NULL | -- | |
| waypoints | JSONB | jsonb | Yes | NULL | -- | |
| codes | JSONB | jsonb | Yes | NULL | -- | |
| financials | JSONB | jsonb | Yes | NULL | -- | |
| analysis | JSONB | jsonb | Yes | NULL | -- | |
| company_name | String(200) | varchar(200) | No | "Apex Trading Co." | B-tree | Added in 007 |
| hold_type | String(50) | varchar(50) | Yes | NULL | -- | Added in 008 |
| created_at | DateTime(tz) | timestamptz | No | now() | B-tree | |
| updated_at | DateTime(tz) | timestamptz | No | now() | -- | |

**FK**: `shipments.order_id` -> `orders.id` (SET NULL on delete, unique)

#### `documents`

| Column | SQLAlchemy Type | PG Type | Nullable | Default | Index | Notes |
|--------|----------------|---------|----------|---------|-------|-------|
| id | UUID | uuid | No | uuid4() | PK | |
| order_id | UUID (FK) | uuid | Yes | NULL | B-tree | CASCADE, nullable since 006 |
| shipment_id | UUID (FK) | uuid | Yes | NULL | B-tree | CASCADE, added in 006 |
| document_type | String(100) | varchar(100) | No | -- | -- | |
| filename | String(300) | varchar(300) | No | -- | -- | |
| content_base64 | Text | text | No | -- | -- | Base64-encoded content |
| content_type | String(100) | varchar(100) | No | "application/pdf" | -- | |
| created_at | DateTime(tz) | timestamptz | No | now() | -- | |
| updated_at | DateTime(tz) | timestamptz | No | now() | -- | |

---

## JSONB Schemas with Examples

### `section_232_scope.country_exemptions`

```json
{
  "countries": ["AU", "CA", "MX"],
  "notes": "Exempted under quota arrangements effective 2022-04-01"
}
```

### `adcvd_orders.hs_codes`

Simple array of HS code strings covered by the AD/CVD order:

```json
["7304.19.10", "7304.19.50", "7304.31.60", "7304.39.00"]
```

### `restricted_parties.aliases`

Simple array of known alias strings:

```json
["SBERBANK", "PAO SBERBANK"]
```

### `restricted_parties.addresses`

Array of address objects:

```json
[
  {
    "street": "19 Vavilova St.",
    "city": "Moscow",
    "country": "Russia",
    "postal_code": "117997"
  }
]
```

### `restricted_parties.identification`

Object with ID document types as keys:

```json
{
  "registration_number": "1027700132195",
  "tax_id": "7707083893"
}
```

### `restricted_parties.programs`

Array of sanctions program codes from the source list:

```json
["RUSSIA-EO14024", "UKRAINE-EO13662"]
```

### `pga_mappings.requirements`

Structured requirements object with boolean flags and string arrays:

```json
{
  "prior_notice": true,
  "food_facility_registration": true,
  "fda_product_code": true,
  "documentation": [
    "FDA Prior Notice",
    "Food Facility Registration",
    "Commercial Invoice"
  ],
  "ace_filing": "PGA message set required in ACE"
}
```

For medical devices:

```json
{
  "device_listing": true,
  "establishment_registration": true,
  "premarket_clearance": "510(k) or PMA as applicable",
  "documentation": [
    "510(k) Clearance Letter",
    "Device Listing",
    "MDR Registration"
  ],
  "ace_filing": "PGA message set required in ACE"
}
```

### `fta_rules.documentation`

```json
{
  "certificate_of_origin": true,
  "form_type": "USMCA Certificate of Origin",
  "required_fields": [
    "producer",
    "exporter",
    "importer",
    "hs_code",
    "origin_criterion"
  ]
}
```

### `tax_regime_templates.template`

US additive regime example:

```json
{
  "layers": [
    {"name": "MFN Duty", "code": "MFN", "base": "CIF", "type": "ad_valorem"},
    {"name": "Section 301", "code": "301", "base": "CIF", "type": "ad_valorem", "condition": "origin=CN"},
    {"name": "Section 232", "code": "232", "base": "CIF", "type": "ad_valorem", "condition": "product_category in [STEEL, ALUMINUM]"},
    {"name": "IEEPA", "code": "IEEPA", "base": "CIF", "type": "ad_valorem", "condition": "country_specific"},
    {"name": "AD/CVD", "code": "ADCVD", "base": "CIF", "type": "ad_valorem", "condition": "case_specific"},
    {"name": "MPF", "code": "MPF", "base": "CIF", "type": "ad_valorem", "min": 31.67, "max": 614.35, "rate": 0.3464},
    {"name": "HMF", "code": "HMF", "base": "CIF", "type": "ad_valorem", "rate": 0.125}
  ],
  "computation": "additive",
  "formula": "landed = CIF + sum(all_layers)"
}
```

Layer object fields:
- `name` (string): Human-readable layer name
- `code` (string): Short code used in calculations
- `base` (string): What the rate applies to ("CIF", "CIF + MFN + ADD")
- `type` (string): "ad_valorem" or "specific"
- `condition` (string, optional): When this layer applies
- `rate` (float, optional): Default rate if not looked up
- `min` (float, optional): Minimum amount (floor)
- `max` (float, optional): Maximum amount (ceiling)

### `regulatory_signals.affected_hs_codes`

```json
["8471", "8542", "8541.40"]
```

### `regulatory_signals.affected_countries`

```json
["CN", "VN", "TH"]
```

### `products.hs_codes`

Multi-jurisdiction HS code mapping:

```json
{
  "US": "8471.30.01",
  "EU": "8471.30.00",
  "CN": "8471.3000"
}
```

### `products.source_locations`

```json
[
  {"country": "CN", "facility": "Shenzhen Factory A", "percentage": 60},
  {"country": "VN", "facility": "Ho Chi Minh Plant B", "percentage": 40}
]
```

### `shipments.events`

Ordered array of lifecycle events:

```json
[
  {"timestamp": "2026-01-15T08:00:00Z", "type": "departed", "location": "Shanghai", "details": "Vessel departed port"},
  {"timestamp": "2026-01-20T14:30:00Z", "type": "in_transit", "location": "Pacific Ocean", "details": "ETA Long Beach Jan 28"},
  {"timestamp": "2026-01-28T09:00:00Z", "type": "arrived", "location": "Long Beach", "details": "Arrived at port"},
  {"timestamp": "2026-01-28T12:00:00Z", "type": "customs_hold", "location": "Long Beach", "details": "Held for inspection"}
]
```

### `shipments.financials`

```json
{
  "declared_value": 10000.00,
  "currency": "USD",
  "total_duty": 4531.67,
  "mfn_duty": 0.00,
  "section_301": 2500.00,
  "ieepa": 2000.00,
  "mpf": 31.67,
  "hmf": 12.50,
  "effective_rate_pct": 45.3
}
```

### `orders.document_status`

```json
{
  "commercial_invoice": "done",
  "packing_list": "done",
  "bill_of_lading": "pending",
  "certificate_of_origin": "skipped"
}
```

Values: `"done"`, `"pending"`, `"skipped"`

### `order_line_items.tariff_breakdown`

```json
{
  "mfn_rate_pct": 0.0,
  "section_301_pct": 25.0,
  "ieepa_pct": 20.0,
  "mpf": 31.67,
  "hmf": 12.50,
  "total_duty": 4531.67,
  "effective_rate_pct": 45.3
}
```

---

## Relationships Diagram

```
  products (UUID PK)
      |
      | (logical ref via product_id, no FK)
      |
  order_line_items ----FK(CASCADE)----> orders (UUID PK)
                                            |         |
                                            |         | FK(SET NULL)
                                            |         v
                                            |     shipments (UUID PK)
                                            |         |
                                            |         | FK(SET NULL, unique)
                                            +---------+
                                            |         |
                                       FK(CASCADE)  FK(CASCADE)
                                            |         |
                                            v         v
                                         documents (UUID PK)
```

**Bidirectional order-shipment link**:
- `orders.shipment_id` -> `shipments.id` (SET NULL on delete)
- `shipments.order_id` -> `orders.id` (SET NULL on delete, unique)
- This creates a soft one-to-one relationship where either side can be deleted independently.

---

## Fuzzy Matching with pg_trgm

The `restricted_parties` table uses `pg_trgm` for fuzzy name matching in the compliance
screening engine (E3).

### Index Definition

```python
Index(
    "ix_restricted_parties_entity_name_trgm",
    "entity_name",
    postgresql_using="gin",
    postgresql_ops={"entity_name": "gin_trgm_ops"},
)
```

### How it works

The GIN index stores trigrams (3-character substrings) of each `entity_name`. When a query
uses `similarity()` or the `%` operator, PostgreSQL can use this index to efficiently find
matches without a full table scan.

### Usage patterns

**Similarity function** (returns a score from 0.0 to 1.0):

```sql
SELECT entity_name, list_source, similarity(entity_name, 'HUAWEI TECH') AS score
FROM restricted_parties
WHERE similarity(entity_name, 'HUAWEI TECH') > 0.3
ORDER BY score DESC
LIMIT 10;
```

**Trigram operator** (boolean, uses default threshold of 0.3):

```sql
SELECT * FROM restricted_parties
WHERE entity_name % 'SMIC SEMICONDUCTOR';
```

**Setting similarity threshold** (session-level):

```sql
SET pg_trgm.similarity_threshold = 0.4;
SELECT * FROM restricted_parties
WHERE entity_name % 'ROSNEFT OIL';
```

### Performance considerations

- The GIN index supports efficient lookups even with 10,000+ entities.
- Trigram matching handles common screening challenges: transliterations, abbreviations,
  spacing variations, and minor typos.
- The compliance engine also checks the `aliases` JSONB array after initial trigram
  matching on `entity_name`.

---

## Query Patterns Used in Engines

### Engine E1 (Classification) -- HTSUS Lookup

```python
# Look up tariff headings by HS code prefix and jurisdiction
stmt = (
    select(HTSUSHeading)
    .where(HTSUSHeading.hs_code.startswith(hs_prefix))
    .where(HTSUSHeading.jurisdiction == "US")
    .order_by(HTSUSHeading.hs_code)
)
```

### Engine E2 (Tariff) -- Multi-program Rate Stacking

```python
# Get base MFN rate
heading = await session.execute(
    select(HTSUSHeading)
    .where(HTSUSHeading.hs_code == hs_code)
    .where(HTSUSHeading.jurisdiction == destination)
)

# Check Section 301 applicability
s301 = await session.execute(
    select(Section301List)
    .where(Section301List.hs_code == hs_code)
    .where(Section301List.exclusion == False)
)

# Check Section 232 scope
s232 = await session.execute(
    select(Section232Scope)
    .where(Section232Scope.hs_code == hs_code)
)

# Check IEEPA rates for origin country
ieepa = await session.execute(
    select(IEEPARate)
    .where(IEEPARate.country_code == origin_country)
)

# Check AD/CVD orders (uses GIN index on hs_codes JSONB)
adcvd = await session.execute(
    select(ADCVDOrder)
    .where(ADCVDOrder.hs_codes.contains([hs_code]))
    .where(ADCVDOrder.country == origin_country)
    .where(ADCVDOrder.status == "ACTIVE")
)
```

### Engine E3 (Compliance) -- Restricted Party Screening

```python
# Fuzzy match using pg_trgm similarity
stmt = text("""
    SELECT entity_name, list_source, country,
           similarity(entity_name, :name) AS score
    FROM restricted_parties
    WHERE similarity(entity_name, :name) > :threshold
    ORDER BY score DESC
    LIMIT :limit
""")
result = await session.execute(
    stmt, {"name": entity_name, "threshold": 0.3, "limit": 10}
)
```

### Engine E3 (Compliance) -- PGA Range Query

```python
# Find PGA requirements for an HS code
stmt = (
    select(PGAMapping)
    .where(PGAMapping.hs_code_start <= hs_code)
    .where(PGAMapping.hs_code_end >= hs_code)
)
```

### Engine E4 (FTA) -- Rules of Origin Lookup

```python
# Find applicable FTA rules for a chapter
chapter = hs_code[:2]
stmt = (
    select(FTARule)
    .where(FTARule.agreement == "USMCA")
    .where(FTARule.chapter_range_start <= chapter)
    .where(FTARule.chapter_range_end >= chapter)
)
```

### Engine E2 (Tariff) -- Tax Regime Template

```python
# Load computation template for a jurisdiction
stmt = (
    select(TaxRegimeTemplate)
    .where(TaxRegimeTemplate.jurisdiction == destination)
)
```

### Dashboard -- Shipment Aggregation

```python
# Count shipments by status
stmt = (
    select(Shipment.status, func.count(Shipment.id))
    .group_by(Shipment.status)
)

# Shipments with holds
stmt = (
    select(Shipment)
    .where(Shipment.hold_type.isnot(None))
    .where(Shipment.company_name == company_name)
)
```

---

## Adding New Tables and Columns

### Adding a new column to an existing table

1. **Update the ORM model** in the appropriate file under `backend/app/knowledge/models/`:

```python
# Example: adding a new column to Shipment
class Shipment(TimestampMixin, Base):
    # ... existing columns ...
    new_column: Mapped[Optional[str]] = mapped_column(String(100), nullable=True)
```

2. **Create an Alembic migration**:

```bash
cd backend
alembic revision --autogenerate -m "add_new_column_to_shipments"
```

3. **Review the generated migration** in `alembic/versions/` and adjust if needed.

4. **Apply the migration**:

```bash
alembic upgrade head
```

### Adding a new table

1. **Create the model** in the appropriate domain file (tariff.py, compliance.py,
   reference.py, or operational.py), or create a new file.

2. **Import the model** in the package init or ensure Alembic's `env.py` discovers it.

3. **Generate and apply the migration** as above.

### Naming conventions

- Table names: `snake_case` plural (e.g. `restricted_parties`, `fee_schedule`)
- Column names: `snake_case` (e.g. `hs_code`, `rate_pct`)
- Index names: `ix_{table}_{column}` (e.g. `ix_htsus_headings_hs_code`)
- FK constraint names: `fk_{source_table}_{column}` (e.g. `fk_shipments_order_id`)
- Unique constraint names: `uq_{table}_{columns}` (e.g. `uq_htsus_headings_hs_code_jurisdiction`)

### JSONB column guidelines

- Use `JSONB` (not `JSON`) for all structured data -- supports indexing and containment queries.
- Document the expected schema in this file and in the model docstring.
- Use GIN indexes on JSONB columns that participate in `@>` containment queries.
- Prefer simple structures (arrays of strings, flat objects) over deeply nested JSON.

---

## Data Seeding

All seed scripts are in `backend/data/ingestion/`. Run them via:

```bash
cd backend
python -m data.ingestion.seed_all          # Seed everything
python -m data.ingestion.seed_htsus_chapters
python -m data.ingestion.seed_section301
python -m data.ingestion.seed_section232
python -m data.ingestion.seed_ieepa
python -m data.ingestion.seed_adcvd
python -m data.ingestion.seed_restricted_parties
python -m data.ingestion.seed_pga
python -m data.ingestion.seed_fta_rules
python -m data.ingestion.seed_cross_rulings
python -m data.ingestion.seed_taxes
python -m data.ingestion.seed_fees
python -m data.ingestion.seed_exchange_rates
python -m data.ingestion.seed_tax_regimes
python -m data.ingestion.seed_regulatory_signals
python -m data.ingestion.seed_demo_shipments
python -m data.ingestion.seed_eu_tariff
python -m data.ingestion.seed_cn_tariff
python -m data.ingestion.seed_br_tariff
python -m data.ingestion.seed_in_tariff
```

Seed scripts use `ON CONFLICT DO NOTHING` or upsert patterns to be idempotent.
