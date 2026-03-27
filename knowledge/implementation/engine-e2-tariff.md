# E2: Global Tariff and Landed Cost -- Implementation Guide

This document describes the implementation details of Engine 2, the Global Tariff
and Landed Cost engine. E2 is entirely deterministic (no LLM calls, no randomness)
and computes duty, tax, fee, and landed cost breakdowns for five jurisdictions.

---

## File Locations

```
backend/app/engines/e2_tariff/
  __init__.py
  engine.py        # TariffEngine dispatcher
  tax_regime.py    # TaxRegimeEngine abstract base, TariffLineItem, TariffResult
  landed_cost.py   # Formatting utilities and comparison functions
  regimes/
    __init__.py
    us.py          # USTaxRegime (Pattern A)
    eu.py          # EUTaxRegime (Pattern A)
    cn.py          # ChinaTaxRegime (Pattern A with grossup sub-step)
    br.py          # BrazilTaxRegime (Pattern B)
    india.py       # IndiaTaxRegime (Pattern A)
  programs/
    __init__.py    # Trade programs (FTA, GSP) -- placeholder
```

---

## TariffEngine Dispatcher Pattern

### Class: TariffEngine

```
engine_id = "E2"
engine_name = "Global Tariff & Landed Cost"
```

### Regime Resolution

`_get_regime(destination)` resolves the appropriate `TaxRegimeEngine` subclass:

1. Normalize destination to uppercase.
2. If the destination is one of the 27 EU member state codes, map to `"EU"`.
3. Look up in `REGIME_MAP`:
   ```python
   REGIME_MAP = {
       "US": USTaxRegime,
       "EU": EUTaxRegime,
       "CN": ChinaTaxRegime,
       "BR": BrazilTaxRegime,
       "IN": IndiaTaxRegime,
   }
   ```
4. Cache the instantiated regime for reuse.
5. Raise `ValueError` if destination is unsupported.

### EU Member State Routing

All 27 EU member states are in the `EU_MEMBERS` frozenset:
```
AT, BE, BG, CY, CZ, DE, DK, EE, ES, FI, FR, GR, HR, HU, IE, IT,
LT, LU, LV, MT, NL, PL, PT, RO, SE, SI, SK
```

The original destination code is preserved and passed through to the EU regime for
VAT rate differentiation by member state.

### Execute Method

```python
async def execute(
    self,
    hs_code: str = "",
    origin_country: str = "",
    destination_country: str = "US",
    declared_value: float = 0.0,
    currency: str = "USD",
    entry_type: str = "FORMAL",
    **kwargs: Any,
) -> EngineOutput
```

Error handling:
- `ValueError` (unsupported destination): logged as warning, returns error status.
- Other exceptions: logged with full traceback, returns error status.

---

## TaxRegimeEngine Abstract Base

**File:** `tax_regime.py`

### Class Attributes

| Attribute      | Description |
|----------------|-------------|
| `jurisdiction` | Country/region code (`"US"`, `"EU"`, `"CN"`, `"BR"`, `"IN"`) |
| `pattern`      | `"ADDITIVE"` or `"CASCADING"` |

### Abstract Method

```python
async def compute(
    self,
    hs_code, origin_country, destination_country,
    declared_value, currency="USD", entry_type="FORMAL",
) -> dict[str, Any]
```

Returns `TariffResult.to_dict()`.

### TariffLineItem Dataclass

Each line item in the tariff stack:

| Field        | Type    | Description |
|--------------|---------|-------------|
| `program`    | `str`   | Short program name (e.g., "MFN Duty", "Section 301") |
| `description`| `str`   | Human-readable explanation |
| `rate`       | `str`   | Display-formatted rate (e.g., "9.8%", "Free") |
| `rate_pct`   | `float` | Numeric percentage rate |
| `amount`     | `float` | Computed monetary amount |
| `citation`   | `str`   | Legal/regulatory citation |
| `category`   | `str`   | `"DUTY"`, `"SURCHARGE"`, `"TAX"`, or `"FEE"` |
| `base_value` | `float` | Value this charge was computed against |
| `cumulative` | `float` | Running total through this item |

### TariffResult Dataclass

Aggregates all line items:

| Field               | Description |
|---------------------|-------------|
| `hs_code`           | HS code used for computation |
| `origin_country`    | ISO origin |
| `destination_country` | ISO destination |
| `declared_value`    | CIF value |
| `currency`          | ISO currency |
| `entry_type`        | FORMAL or INFORMAL |
| `line_items`        | Ordered list of `TariffLineItem` |
| `total_duty`        | Sum of DUTY + SURCHARGE amounts |
| `total_taxes`       | Sum of TAX amounts |
| `total_fees`        | Sum of FEE amounts |
| `landed_cost`       | declared_value + total_duty + total_taxes + total_fees |
| `effective_rate`    | (landed_cost - declared_value) / declared_value |
| `warnings`          | Non-fatal informational messages |

`to_dict()` serializes with monetary values rounded to 2dp and effective rate to
4dp.

---

## US Regime (regimes/us.py) -- Pattern A (Additive)

### Computation Order

**Step 1: MFN Duty**
- Look up base rate in `MFN_RATES` by progressively shortening the HS code
  (10-digit, 8, 7, 6, 4-digit prefixes).
- Falls back to database lookup via `_lookup_mfn_db()` when `db_session` is
  available (queries `htsus_headings` table).
- Applied on CIF (declared) value.

**Step 2: Section 301 (China origin only)**
- Check HS code against `SECTION_301_CODES` set at progressively shorter prefixes.
- Rate: 25% on covered codes.
- Database-backed: `_check_301_db()` queries `section_301_lists` table.
- Applied on CIF value (not on top of MFN duty).

**Step 3: Section 232 (Steel/Aluminum)**
- Chapters 72-73 (steel): 25%.
- Chapter 76 (aluminum): 10%.
- Database-backed: `_check_232_db()` queries `section_232_scope` table.
- Applied on CIF value.

**Step 4: IEEPA**
- Country-level surcharge: CN = 145%.
- Database-backed: `_lookup_ieepa_db()` queries `ieepa_rates` table.
- Applied on CIF value.

**Step 5: AD/CVD** -- placeholder. Product-and-country specific, requires database
lookup against `adcvd_orders` table.

**Step 6: Merchandise Processing Fee (MPF)**
- Formal entry (value > $2,500): 0.3464% of CIF, clamped between $33.58 and
  $651.50.
- Informal entry (value <= $2,500): flat $2.69.
- Citation: 19 USC 58c.

**Step 7: Harbor Maintenance Fee (HMF)**
- 0.125% of CIF on ocean-borne cargo.
- Citation: 26 USC 4461.

**Important:** All surcharges (301, 232, IEEPA) assess on declared customs value.
They do NOT compound on each other.

### MFN Rate Table (Golden Path)

Approximately 25 hardcoded MFN rates covering demo products. Examples:
- `6912.00.48` (ceramics): 9.8%
- `8518.21` (speakers): 0%
- `6104.63` (women's trousers): 28.6%
- `8703.23` (automobiles): 2.5%
- `6110.30` (sweaters): 32%

### Database Query Patterns

All database lookups use SQLAlchemy `text()` with parameterized queries. The
progressive prefix matching pattern is consistent:
```sql
SELECT mfn_rate_pct, description FROM htsus_headings
WHERE hs_code = :code AND jurisdiction = 'US' LIMIT 1
```
Tried for each prefix length (10, 8, 7, 6, 4), falling back to hardcoded data.

---

## EU Regime (regimes/eu.py) -- Pattern A

### Computation Order

**Step 1: MFN Duty**
- Common External Tariff from TARIC schedule.
- Same progressive prefix lookup pattern as US.
- Applied on CIF value.

**Step 2: VAT**
- Base = CIF + duty (duty-inclusive).
- Rate varies by member state (17-27%).
- Default for generic "EU" destination: 20%.
- Citation: Council Directive 2006/112/EC.

### VAT Rate Table

27 member states with rates ranging from 17% (Luxembourg) to 27% (Hungary).

---

## China Regime (regimes/cn.py) -- Pattern A + Grossup Sub-Step

### Computation Order

**Step 1: MFN Duty** -- China Customs Tariff Commission rates on CIF.

**Step 2: Retaliatory Tariff** -- 125% on US-origin goods (April 2025).

**Step 3: Consumption Tax (select categories)**
- Applies to luxury/excise categories by heading:
  - Automobiles (8703): 5%
  - Cosmetics (3304): 15%
  - Wine (2204): 10%
  - Tobacco (2402): 56%
- **Grossup formula:**
  ```
  base = (CIF + all duties) / (1 - consumption_rate / 100)
  tax = base * consumption_rate / 100
  ```
  This is a mini Pattern-B step within an otherwise additive regime.

**Step 4: VAT** -- 13% standard rate on (CIF + duty + consumption tax).

---

## Brazil Regime (regimes/br.py) -- Pattern B (Cascading)

### Computation Order

**Step 1: II (Imposto de Importacao)** -- Import duty on CIF.

**Step 2: IPI (Imposto sobre Produtos Industrializados)** -- Excise tax on
(CIF + II). Rates by heading (0-25%).

**Step 3: PIS-Importacao** -- 2.1% on CIF (fixed, Law 10.865/2004).

**Step 4: COFINS-Importacao** -- 9.65% on CIF (fixed, Law 10.865/2004).

**Step 5: ICMS (Grossup Solver)**
ICMS includes itself in its own base. This is the defining feature of Pattern B.
```
base_icms = (CIF + II + IPI + PIS + COFINS) / (1 - icms_rate / 100)
ICMS = base_icms * icms_rate / 100
```
- Rates by state: SP=18%, RJ=20%, MG=18%, etc. Default: 17%.
- Citation: Lei Complementar 87/1996 (Kandir Law).
- Safety check: rate must be < 100% for grossup validity.

---

## India Regime (regimes/india.py) -- Pattern A

### Computation Order

**Step 1: BCD (Basic Customs Duty)** -- First Schedule, Customs Tariff Act 1975.
Applied on CIF. Ranges from 0% (ITA-bound laptops) to 150% (wine).

**Step 2: SWS (Social Welfare Surcharge)** -- 10% of the BCD **amount** (not CIF).
Citation: Finance Act 2018 section 110.

**Step 3: IGST (Integrated Goods and Services Tax)** -- Applied on
(CIF + BCD + SWS).
- Standard rate: 18%.
- Luxury rate: 28% (motor vehicles chapter 87, tobacco chapter 24, aerated
  beverages heading 2202).
- Citation: IGST Act 2017 section 5.

---

## Landed Cost Calculator (landed_cost.py)

Three formatting functions, all pure/deterministic:

### format_platform_detail(tariff_result)
Full breakdown with `duty_breakdown`, `tax_breakdown`, `fee_breakdown` sub-sections
and a `summary` block including effective rate display.

### format_shipper_summary(tariff_result)
Category totals, active programme list, and effective rate.

### format_buyer_price(tariff_result)
Minimal: declared value, total import charges, landed cost.

### compare_landed_costs(results)
Compares multiple tariff results: sorted by landed cost, identifies cheapest and
most expensive, calculates spread.

---

## How to Add a New Jurisdiction

1. **Create a regime file** at `regimes/xx.py` (where `xx` is the ISO country code
   in lowercase).

2. **Subclass TaxRegimeEngine:**
   ```python
   class XXTaxRegime(TaxRegimeEngine):
       jurisdiction: str = "XX"
       pattern: str = "ADDITIVE"  # or "CASCADING"

       async def compute(self, hs_code, origin_country, destination_country,
                         declared_value, currency="USD", entry_type="FORMAL"):
           # Build line_items list
           # Compute totals
           # Return TariffResult(...).to_dict()
   ```

3. **Add rate tables** as class-level dicts (same format as `MFN_RATES` in other
   regimes).

4. **Register in engine.py:**
   ```python
   from app.engines.e2_tariff.regimes.xx import XXTaxRegime
   REGIME_MAP["XX"] = XXTaxRegime
   ```

5. **Add database queries** if the regime should support live rate lookups
   (follow the `_lookup_mfn_db()` pattern in `us.py`).

6. **Update EU_MEMBERS** if the country is an EU member state that should route
   to the EU regime.

### Pattern Selection Guide

- **Pattern A (Additive):** Use when each tax/duty computes from a well-defined
  base (CIF, CIF + duty, etc.) and does not reference itself. Most jurisdictions.
- **Pattern B (Cascading):** Use when any tax includes itself in its own base
  (requires algebraic grossup). Currently only Brazil ICMS.
- **Hybrid:** Pattern A with one grossup sub-step. Currently China (consumption
  tax grossup within an otherwise additive regime).
