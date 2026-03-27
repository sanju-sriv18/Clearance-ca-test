# E3: Compliance Screening -- Implementation Guide

This document describes the implementation details of Engine 3, the Compliance
Screening engine. E3 orchestrates three sub-engines (PGA, DPS, UFLPA) in parallel
and produces a unified compliance status for each shipment.

---

## File Locations

```
backend/app/engines/e3_compliance/
  __init__.py
  engine.py        # ComplianceEngine orchestrator
  pga.py           # PGAScreener + PGA_MAPPINGS reference data
  dps.py           # DPSScreener + RESTRICTED_PARTIES reference data
  uflpa.py         # UFLPAScreener + high-risk sectors + entities
  fuzzy_match.py   # normalize_entity(), multi_score() utilities
```

---

## ComplianceEngine Orchestrator

### Class: ComplianceEngine

```
engine_id = "E3"
engine_name = "Compliance Screening"
```

### Constructor

```python
def __init__(self, db_session: Any | None = None) -> None:
    self.pga = PGAScreener(db_session)
    self.dps = DPSScreener(db_session)
    self.uflpa = UFLPAScreener(db_session)
```

### Execute Method

```python
async def execute(
    self,
    hs_code: str = "",
    origin_country: str = "",
    entity_name: str = "",
    product_description: str = "",
    destination_country: str = "US",
    **kwargs: Any,
) -> EngineOutput
```

### Parallel Execution Strategy

All three sub-engines run concurrently via `asyncio.gather` with
`return_exceptions=True`:

```python
pga_result, dps_result, uflpa_result = await asyncio.gather(
    pga_coro, dps_coro, uflpa_coro,
    return_exceptions=True,
)
```

When input is absent (e.g., no `hs_code` for PGA, no `entity_name` for DPS), the
engine substitutes `asyncio.sleep(0, result=default)` to avoid unnecessary work.

Individual sub-engine failures are captured as `BaseException` instances and
converted to safe defaults with the error appended to the output `errors` list.
The overall status is set to `"partial"` when any sub-engine fails.

### Output Structure

```python
{
    "overall_status": "HOLD" | "REVIEW" | "PGA_REQUIRED" | "CLEAR",
    "pga_flags": [...],         # List of agency requirement dicts
    "dps_result": {...},        # DPS screening result dict
    "uflpa_risk": {...},        # UFLPA assessment dict
}
```

### Status Precedence

`_determine_status()` applies strict precedence:

```
HOLD > REVIEW > PGA_REQUIRED > CLEAR
```

| Condition | Status |
|-----------|--------|
| DPS risk_level not in (CLEAR, ERROR) | HOLD |
| UFLPA risk_level in (HIGH, MEDIUM) | REVIEW |
| PGA flags present | PGA_REQUIRED |
| None of the above | CLEAR |

---

## PGA Screening (pga.py)

### PGA_MAPPINGS Reference Data

A list of approximately 20 mapping entries covering major US Partner Government
Agencies. Each entry:

```python
{
    "agency": "FDA",
    "agency_name": "Food and Drug Administration",
    "hs_start": "0100",       # 4-digit range start (string)
    "hs_end": "2499",         # 4-digit range end (string)
    "description": "Food products",
    "requirements": ["Prior Notice", "Food Facility Registration"],
    "severity": "REQUIRED" | "CONDITIONAL",
    "citation": "21 CFR Part 1 Subpart I; FD&C Act S.801",
}
```

### Agencies Covered

| Agency | Coverage | Key HS Ranges |
|--------|----------|---------------|
| FDA    | Food, pharma, cosmetics, medical devices | 0100-2499, 3001-3006, 3301-3307, 9018-9022 |
| CPSC   | Toys, children's products, furniture | 9503-9505, 9401-9404, 6111-6112 |
| EPA    | Chemicals (TSCA), pesticides (FIFRA), engines | 2801-2853, 3808, 8407-8408 |
| USDA   | Plants, flowers, meat | 0601-0604, 0201-0210 |
| FWS    | Wildlife, coral, fur | 0508, 4301-4304 |
| TTB    | Alcoholic beverages, tobacco | 2203-2208, 2401-2403 |
| NHTSA  | Motor vehicles | 8701-8711 |
| ATF    | Firearms and ammunition | 9301-9307 |
| DEA    | Controlled substance precursors | 2939 |
| FCC    | Telecom equipment, radio/TV | 8517-8518, 8525-8529 |

### PGAScreener.screen() Algorithm

1. If `destination_country` is not `"US"`, return empty list (US-only).
2. Normalize HS code to first 4 digits, left-padded with zeros.
3. Iterate `PGA_MAPPINGS`: if `hs_start <= normalized <= hs_end`, add the mapping
   to the flags list.
4. Return list of flag dicts with: `agency`, `agency_name`, `description`,
   `requirements`, `severity`, `citation`, `matched_range`.

---

## DPS Fuzzy Matching (dps.py)

### RESTRICTED_PARTIES Reference Data

Approximately 50 entities across four consolidated lists:

| List          | Source     | Count (approx.) |
|---------------|------------|------------------|
| SDN           | OFAC Treasury | 20 |
| BIS_ENTITY    | Commerce   | 15 |
| UFLPA         | DHS        | 12 |
| EU_SANCTIONS  | EU Council | 10 |

Each entry:
```python
{
    "name": "HUAWEI TECHNOLOGIES CO LTD",
    "list": "SDN",
    "type": "ENTITY",
    "country": "CN",
    "aliases": ["HUAWEI", "HUAWEI TECH", "HUAWEI TECHNOLOGIES"],
}
```

### DPSScreener Class

Constructor builds a flat searchable index: `(UPPER_NAME, party_index)` tuples
covering both canonical names and all aliases.

### Three-Stage Screening Pipeline

**Stage 1 -- Exact Match:**
Direct string equality (`query == name`) against all uppercased canonical names and
aliases. O(n) scan. Match type: `EXACT`, score: 100.

**Stage 2 -- Token-Sort Fuzzy Match:**
```python
process.extract(query, name_list, scorer=fuzz.token_sort_ratio, limit=10)
```
Uses `rapidfuzz` library. Catches reordered or partially matched names. Filters by
score >= `MEDIUM_THRESHOLD` (70).

Match types by score:
- >= 95: `EXACT`
- >= 85: `HIGH`
- >= 70: `POSSIBLE`

**Stage 3 -- Partial-Ratio Fallback:**
Only runs if stages 1 and 2 return no matches. Uses `fuzz.partial_ratio` for
substring matching. Score threshold: >= 95 (`EXACT_THRESHOLD`). Match type:
`PARTIAL`.

### Scoring Thresholds

| Threshold          | Value | Match Type |
|--------------------|-------|------------|
| `EXACT_THRESHOLD`  | 95    | EXACT or PARTIAL |
| `HIGH_THRESHOLD`   | 85    | HIGH |
| `MEDIUM_THRESHOLD` | 70    | POSSIBLE |

### Risk Assessment

`_assess_risk(matches)` derives risk level from top matches:

| Condition | Risk Level |
|-----------|------------|
| Any EXACT match | CRITICAL |
| Any HIGH match | HIGH |
| Any matches at all | REVIEW |
| No matches | CLEAR |

### Output Structure

```python
{
    "matches": [                # Top 5, sorted by score descending
        {
            "entity_name": "HUAWEI TECHNOLOGIES CO LTD",
            "list_source": "SDN",
            "entity_type": "ENTITY",
            "country": "CN",
            "match_type": "EXACT",
            "match_score": 100.0,
            "matched_on": "HUAWEI TECHNOLOGIES CO LTD",
        },
    ],
    "risk_level": "CRITICAL",
    "screened_name": "Huawei Technologies",
    "lists_checked": ["SDN", "BIS_ENTITY", "UFLPA", "EU_SANCTIONS"],
    "total_entries_checked": 50,
}
```

---

## Fuzzy Match Utilities (fuzzy_match.py)

### normalize_entity(name)

Normalization pipeline:
1. Uppercase and strip whitespace.
2. Remove common legal suffixes: LTD, LLC, INC, CORP, CO, SA, AG, GMBH, PLC, NV,
   BV, PTY, OOO, JSC, PJSC, OAO, ZAO, SRL, SARL, KK, SPA.
3. Strip punctuation (commas, dots, hyphens to spaces).
4. Collapse consecutive whitespace.

### multi_score(query, candidate)

Computes four `rapidfuzz` scores after normalization:
- `ratio` -- simple Levenshtein similarity.
- `partial_ratio` -- best partial substring match.
- `token_sort` -- tokens sorted alphabetically then scored.
- `token_set` -- intersection/remainder token decomposition.
- `best` -- maximum of all four.

Returns a dict mapping score names to float percentages (0-100).

---

## UFLPA Risk Assessment (uflpa.py)

### Four Risk Factors

**Factor 1: Origin Country (China)**
- Non-CN origin: immediately return CLEAR with score 0.
- CN origin: +2 points, "China origin" factor added.

**Factor 2: Sector Risk**
Six high-risk sectors with HS prefix matching:

| Sector               | HS Prefixes | Risk Factor |
|----------------------|-------------|-------------|
| COTTON_TEXTILES      | 52xx, 6104, 6106, 6109, 6110, 6203, 6204 | Xinjiang ~85% of China's cotton |
| POLYSILICON_SOLAR    | 2804, 3818, 8541, 8501 | Xinjiang ~35% of global polysilicon |
| TOMATO_PRODUCTS      | 0702, 2002, 2103 | Major Xinjiang processing |
| HUMAN_HAIR           | 6703, 6704 | CBP detained shipments |
| PVC_CHEMICALS        | 3904, 2815 | Zhongtai Chemical (UFLPA entity) |
| ELECTRONICS_COMPONENTS | 8542, 8541 | Supply chain forced labor links |

Each sector match: +3 points (one match per sector).

**Factor 3: Entity Match**
Check entity name and product description against 14 known UFLPA entity keywords
(HOSHINE SILICON, XPCC, BINGTUAN, XINJIANG JUNGGAR, LOPTOP COUNTY, ZHONGTAI
CHEMICAL, NONGFU SPRING, HETIAN HAOLIN, KASHGAR HUAFU, XINJIANG PRODUCTION, AKSU
HUAFU, CHANGJI ESQUEL, HEFEI BITLAND, HEFEI MEILING).

First entity match: +5 points.

**Factor 4: Region References**
Check entity name and product description for keywords: XINJIANG, XUAR, UYGHUR.

First region match: +4 points.

### Score Mapping

| Score Range | Risk Level | Enforcement Priority | Recommendation |
|-------------|------------|----------------------|----------------|
| 0-1         | CLEAR      | NONE                 | No risk factors |
| 2-4         | LOW        | MONITOR              | Baseline monitoring |
| 5-7         | MEDIUM     | POSSIBLE_REVIEW      | Maintain traceability documentation |
| 8+          | HIGH       | LIKELY_DETENTION      | Prepare due diligence, consider alternative sourcing |

### Output Structure

```python
{
    "risk_level": "HIGH",
    "risk_score": 10,
    "factors": [
        {"factor": "China origin", "severity": "MODERATE", "description": "..."},
        {"factor": "High-risk sector: Cotton...", "severity": "HIGH", ...},
        {"factor": "UFLPA entity match: KASHGAR HUAFU", "severity": "CRITICAL", ...},
    ],
    "matched_sectors": ["COTTON_TEXTILES"],
    "enforcement_priority": "LIKELY_DETENTION",
    "recommendation": "HIGH RISK -- Likely to be detained...",
    "citation": "19 USC S.6901 et seq.; UFLPA Strategy (DHS, June 2022)",
}
```

---

## How to Add New Compliance Lists

### Adding a New Restricted Party List (DPS)

1. Add entries to `RESTRICTED_PARTIES` in `dps.py`:
   ```python
   {
       "name": "NEW ENTITY NAME",
       "list": "NEW_LIST_ID",
       "type": "ENTITY",
       "country": "XX",
       "aliases": ["ALIAS1", "ALIAS2"],
   },
   ```

2. Add the list ID to `_LISTS_CHECKED`:
   ```python
   _LISTS_CHECKED = ["SDN", "BIS_ENTITY", "UFLPA", "EU_SANCTIONS", "NEW_LIST_ID"]
   ```

3. The `DPSScreener` constructor automatically rebuilds its searchable index on
   initialization, so no additional registration is needed.

### Adding New PGA Agency Requirements

Add entries to `PGA_MAPPINGS` in `pga.py`:
```python
{
    "agency": "NEW_AGENCY",
    "agency_name": "Full Agency Name",
    "hs_start": "XXXX",
    "hs_end": "XXXX",
    "description": "Product category",
    "requirements": ["Requirement 1", "Requirement 2"],
    "severity": "REQUIRED",
    "citation": "Legal citation",
},
```

### Adding New UFLPA Sectors

Add entries to `UFLPA_HIGH_RISK_SECTORS` in `uflpa.py`:
```python
"NEW_SECTOR_ID": {
    "hs_prefixes": ["XXXX", "YY"],
    "description": "Sector description",
    "risk_factor": "Explanation of forced labor risk",
},
```

### Adding New UFLPA Entities

Add keywords to `UFLPA_ENTITIES` in `uflpa.py`:
```python
UFLPA_ENTITIES.append("NEW ENTITY NAME")
```

Also add the full entry to `RESTRICTED_PARTIES` in `dps.py` with
`"list": "UFLPA"` for cross-referencing by the DPS screener.

### Adding New UFLPA Region Keywords

Add to `UFLPA_HIGH_RISK_REGIONS` in `uflpa.py`:
```python
UFLPA_HIGH_RISK_REGIONS.append("NEW_REGION_KEYWORD")
```
