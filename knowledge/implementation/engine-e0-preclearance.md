# E0: Pre-Clearance Intelligence -- Implementation Guide

This document describes the implementation details of Engine 0, the Pre-Clearance
Intelligence Agent. E0 screens every incoming shipment booking before goods enter
transit by running an LLM-powered agentic tool-calling loop (or a deterministic
fallback) against five compliance tools.

---

## File Locations

```
backend/app/engines/e0_preclearance/
  __init__.py
  engine.py       # PreClearanceAgent class
  prompts.py      # PRECLEARANCE_AGENT_PROMPT system prompt
  tools.py        # TOOL_SCHEMAS, execute_tool(), 5 tool implementations
```

---

## Class: PreClearanceAgent

```
engine_id = "E0"
engine_name = "Pre-Clearance Intelligence"
MAX_AGENT_ITERATIONS = 10
```

### Constructor

```python
def __init__(self, llm_service: Any | None = None) -> None
```

The `llm_service` parameter is optional. When `None`, the engine falls back to
deterministic execution.

### Input Parameters (kwargs)

| Parameter             | Type    | Default | Description |
|-----------------------|---------|---------|-------------|
| `entity_name`         | `str`   | `""`    | Shipper / consignee company name |
| `origin_country`      | `str`   | `""`    | ISO 2-letter origin code |
| `destination_country` | `str`   | `"US"`  | ISO 2-letter destination code |
| `hs_code`             | `str`   | `""`    | Declared HS/HTS code |
| `product_description` | `str`   | `""`    | Product description |
| `declared_value`      | `float` | `0.0`   | Declared customs value (USD) |
| `transport_mode`      | `str`   | `""`    | air / ocean / ground |

### Output (EngineOutput.data)

```python
{
    "clearance_decision": "CLEAR" | "FLAG" | "HOLD",
    "events": [...],           # Timeline events for simulation UI
    "communication": "...",    # Shipper letter (populated for HOLD)
}
```

---

## Agentic Loop Implementation

The `_agentic_loop()` method implements the core agentic pattern:

### Step 1: Format Booking

`_format_booking(**kwargs)` constructs a structured message:
```
New shipment booking for pre-clearance review:

Shipper: {entity_name}
Origin: {origin_country}
Destination: {destination_country}
Product: {product_description}
HS Code: {hs_code}
Declared Value: ${declared_value:,.2f}
Transport Mode: {transport_mode}

Please run all screening tools and provide your assessment.
```

### Step 2: Conversation Loop (max 10 iterations)

```python
for _iteration in range(self.MAX_AGENT_ITERATIONS):
    response = await self.llm.complete_with_tools(
        system_prompt=PRECLEARANCE_AGENT_PROMPT,
        messages=messages,
        tools=TOOL_SCHEMAS,
        temperature=0.1,
        max_tokens=4096,
    )
```

For each response block:
- **tool_use blocks:** Execute the tool via `execute_tool()`, build a timeline
  event via `_tool_event()`, append the assistant content and tool result to the
  conversation.
- **text blocks:** Build an assessment event via `_assessment_event()`.

The loop terminates when the LLM emits no `tool_use` blocks (indicating it has
finished reasoning).

### Step 3: Parse Output

`_parse_agent_output(events)` scans the last `pre_clearance_assessment` event for:
- The clearance decision: looks for `HOLD`, `FLAG`, or defaults to `CLEAR`.
- The shipper communication: extracts text after the `SHIPPER NOTIFICATION:` marker.

### Step 4: Error Handling

If the agentic loop raises any exception, the engine falls back to deterministic
mode with a warning log:
```python
except Exception:
    logger.warning("preclearance_llm_failed", exc_info=True)
    result = await self._deterministic_fallback(...)
```

---

## Five Pre-Clearance Tools

All tools are deterministic functions. No LLM calls happen inside tools -- the LLM
orchestrates, tools execute.

### 1. screen_entity

**Schema input:** `entity_name` (required)

**Implementation:** Delegates to `E3.DPSScreener.screen()` for fuzzy entity
matching against OFAC SDN, BIS Entity List, UFLPA Entity List, and EU Sanctions.

**Output:** `{ matches: [...], risk_level: "CLEAR" | "REVIEW" | "HIGH" | "CRITICAL" }`

### 2. validate_hs_code

**Schema input:** `hs_code` (required), `product_description` (optional)

**Implementation:**
- Strips dots/dashes, checks digit count (6-10), validates chapter (01-99).
- Looks up chapter description from a hardcoded chapter-names dict.
- Matches against `PRODUCTS` reference data by 4-digit prefix.

**Output:** `{ valid_format: bool, issues: [...], chapter: "XX", chapter_description: "...", product_match: {...} }`

### 3. check_value_reasonableness

**Schema input:** `declared_value` (required), `product_description` (required),
`hs_code` (optional)

**Implementation:**
- Finds reference product by HS prefix or name match.
- Compares declared value against reference `value_range` tuple `(lo, hi)`.
- Thresholds: < 50% of min = `under_valued`, > 200% of max = `over_valued`.

**Output:** `{ assessment: "reasonable" | "under_valued" | "over_valued" | ..., risk_flags: [...], reference_range: {...} }`

### 4. identify_required_documents

**Schema input:** `hs_code` (required), `origin_country` (required),
`destination_country`, `declared_value`, `transport_mode`

**Implementation:**
- Universal CBP forms: 3461 (Entry), 7501 (Entry Summary), Commercial Invoice,
  Packing List.
- ISF-10+2 for ocean cargo (19 CFR Part 149).
- Customs Bond for entries > $2,500 (19 CFR Part 113).
- PGA requirements via `_get_pga_flags_sync()` which uses E3's `PGA_MAPPINGS`.
- USMCA Certificate of Origin for MX/CA corridors.

**Output:** `{ documents: [...], total_required: int, total_conditional: int }`

### 5. assess_origin_risk

**Schema input:** `origin_country` (required), `hs_code`, `product_description`

**Implementation:**
- Corridor scrutiny levels: CN=1.3, VN=1.1, IN=1.1.
- Section 301 check (CN origin, up to 25%).
- Section 232 check (chapters 72, 73, 76).
- UFLPA region risk (CN origin).
- AD/CVD exposure for common categories (CN steel, CN electronics).
- Vietnam transshipment monitoring.

**Output:** `{ origin_country, scrutiny_level, overall_risk, risk_factors: [...] }`

---

## Deterministic Fallback Logic

`_deterministic_fallback()` runs all five tools in sequence:

1. `screen_entity` -- check entity against denied party lists.
2. `validate_hs_code` -- verify HS code format and product alignment.
3. `check_value_reasonableness` -- flag value anomalies.
4. `identify_required_documents` -- determine required filings.
5. `assess_origin_risk` -- evaluate country-of-origin risk.

Decision logic:
```python
if entity risk in ("CRITICAL", "HIGH"):
    decision = "HOLD"
elif entity risk == "REVIEW":
    decision = "FLAG"
if value assessment in ("under_valued", "over_valued"):
    decision = max(decision, "FLAG")
if not valid HS format:
    decision = max(decision, "FLAG")
```

For HOLD decisions, `_template_hold_communication()` generates a professional
letter to the shipper with:
- The specific finding (list match, score).
- Regulatory basis (UFLPA, OFAC/IEEPA, BIS/EAR).
- Required actions and documentation.
- Signed as "Clearance Intelligence Engine -- Pre-Clearance Division".

---

## Event Building for Simulation Integration

E0 generates three types of timeline events for the shipment simulation UI:

### pre_clearance_tool

Built by `_tool_event()`. Each tool execution produces a human-readable description
and detail text based on tool type. Fields:

```python
{
    "timestamp": "...",           # UTC ISO format
    "event_type": "pre_clearance_tool",
    "description": "...",         # One-line summary
    "actor": "platform",
    "detail": "...",              # Multi-line explanation
    "tool": "screen_entity",     # Tool name
    "tool_result": {...},         # Serialized tool output
}
```

### pre_clearance_assessment

Built by `_assessment_event()`. Extracts RECOMMENDATION from assessment text.

### pre_clearance_communication

Built by `_communication_event()`. Captures the shipper notification letter with
direction (`to_shipper`) and channel (`email`).

---

## System Prompt (prompts.py)

The `PRECLEARANCE_AGENT_PROMPT` instructs the LLM to:
1. Review every incoming shipment booking systematically.
2. Use all five tools for each booking.
3. Provide a CLEAR/FLAG/HOLD recommendation.
4. For HOLD: draft a professional notification citing specific regulations
   (UFLPA 19 USC 1307, OFAC 31 CFR Part 501, BIS 15 CFR Parts 730-774, etc.).
5. End response with `RECOMMENDATION: CLEAR|FLAG|HOLD`.
6. For HOLD, include `SHIPPER NOTIFICATION:` followed by the full letter.

---

## How to Add a New Tool

1. **Define the schema** in `tools.py` `TOOL_SCHEMAS` list following the Anthropic
   tool schema format:
   ```python
   {
       "name": "new_tool_name",
       "description": "...",
       "input_schema": {
           "type": "object",
           "properties": { ... },
           "required": [ ... ],
       },
   }
   ```

2. **Implement the function** as a sync or async function in `tools.py`:
   ```python
   async def _new_tool(inp: dict[str, Any]) -> dict[str, Any]:
       # Implementation
       return { ... }
   ```

3. **Register in the router** by adding a branch to `execute_tool()`:
   ```python
   elif tool_name == "new_tool_name":
       return await _new_tool(tool_input)
   ```

4. **Add a description branch** in `_tool_event()` inside `engine.py` to generate
   human-readable event descriptions for the new tool.

5. **Update the deterministic fallback** in `_deterministic_fallback()` to include
   the new tool in the sequential execution order.

6. **Update the system prompt** in `prompts.py` to list and describe the new tool
   so the LLM knows to use it.
