# Agent System Internals -- Implementation Details

Deep implementation reference for the agentic assistant system in the Clearance Intelligence Engine. Covers the agent registry data structures, prompt template architecture, tool schemas, execution dispatcher, the agentic SSE loop, and the LLM translation layer.

Source files:
- `backend/app/api/agent/prompts.py` -- AgentConfig, registry, all prompt templates
- `backend/app/api/agent/tools.py` -- TOOL_SCHEMAS, execute_tool(), _dispatch()
- `backend/app/api/routes/chat.py` -- _agentic_sse_stream(), endpoint handler
- `backend/app/api/schemas/chat.py` -- ChatRequest, ChatContext, ChatChunk

---

## Table of Contents

1. [Agent Registry Implementation](#1-agent-registry-implementation)
2. [Prompt Template Structure](#2-prompt-template-structure)
3. [Tool Schema Definitions](#3-tool-schema-definitions)
4. [Execution Dispatcher](#4-execution-dispatcher)
5. [Agentic SSE Loop](#5-agentic-sse-loop)
6. [LLM Translation Layer](#6-llm-translation-layer)
7. [Serialisation Helpers](#7-serialisation-helpers)

---

## 1. Agent Registry Implementation

### Data Structure

```python
@dataclass
class AgentConfig:
    name: str                      # Human-readable display name
    prompt: str                    # Full system prompt (shared + specialist)
    tools: list[dict[str, Any]]    # Reference to TOOL_SCHEMAS (same for all agents)
```

### Registry Map

```python
AGENT_REGISTRY: dict[str, AgentConfig] = {
    "exception-resolution": AgentConfig(
        name="Exception Resolution Specialist",
        prompt=EXCEPTION_PROMPT,
        tools=TOOL_SCHEMAS,
    ),
    "education-center": AgentConfig(
        name="Trade Compliance Instructor",
        prompt=EDUCATION_PROMPT,
        tools=TOOL_SCHEMAS,
    ),
    "regulatory-intel": AgentConfig(
        name="Regulatory Impact Analyst",
        prompt=REGULATORY_PROMPT,
        tools=TOOL_SCHEMAS,
    ),
    "shipment-detail": AgentConfig(
        name="Shipment Analysis Specialist",
        prompt=SHIPMENT_PROMPT,
        tools=TOOL_SCHEMAS,
    ),
    "default": AgentConfig(
        name="Senior Operations Specialist",
        prompt=DEFAULT_PROMPT,
        tools=TOOL_SCHEMAS,
    ),
}
```

### Selection Function

```python
def get_agent_config(screen: str | None) -> AgentConfig:
    return AGENT_REGISTRY.get(screen or "default", AGENT_REGISTRY["default"])
```

Screen values map to frontend route names. Unknown values fall through to the default agent.

---

## 2. Prompt Template Structure

All prompts follow the pattern: `_SHARED_INSTRUCTIONS` + specialist addendum.

### Shared Preamble (`_SHARED_INSTRUCTIONS`)

Sets the AI's identity, critical rules, and capabilities:

1. **Identity:** AI agent in the Clearance Intelligence Engine platform
2. **Target audience:** Licensed customs brokers and compliance operators
3. **Critical rules:**
   - Always use tools before answering (no speculation)
   - When given a shipment ID in context, always call `lookup_shipment` first
   - Cite specific regulations by CFR section / USC / act name
   - Confirm write actions clearly
4. **Analysis cards:** Instructions for `[ANALYSIS_CARD]{JSON}[/ANALYSIS_CARD]` blocks
5. **Tone:** Concise but thorough; operators are professionals

### Specialist Addendum Pattern

Each specialist prompt adds:

- **Persona declaration:** Role title and scope
- **Reasoning pattern:** Ordered thinking framework (e.g., Diagnose -> Explain -> Remediate -> Action plan)
- **WHEN RESPONDING:** Numbered steps the agent should follow
- **Domain-specific reference data:** Deadlines, regulations, mitigation strategies
- **Facilitation instructions:** When and how to use write tools

### Prompt Composition

```python
EXCEPTION_PROMPT = _SHARED_INSTRUCTIONS + """
You are the **Exception Resolution Specialist**. ...
REASONING PATTERN: Diagnose -> Explain -> Remediate -> Action plan
WHEN RESPONDING:
1. FIRST: Call lookup_shipment ...
...
"""
```

---

## 3. Tool Schema Definitions

All 11 tools are defined in `TOOL_SCHEMAS` as a `list[dict]` in Anthropic tool-use format.

### Schema Structure

```python
{
    "name": "tool_name",
    "description": "Multi-sentence description of what the tool does",
    "input_schema": {
        "type": "object",
        "properties": {
            "param_name": {
                "type": "string | number | integer | array",
                "description": "What this parameter means"
            }
        },
        "required": ["param_name"]
    }
}
```

### Complete Tool Parameter Reference

**lookup_shipment**
- `shipment_id` (string, required) -- UUID of the shipment

**search_shipments**
- `status` (string, optional) -- booked, in_transit, at_customs, inspection, held, cleared, delivered
- `origin` (string, optional) -- Country code
- `destination` (string, optional) -- Country code
- `company_name` (string, optional) -- Case-insensitive partial match
- `hold_type` (string, optional) -- uflpa, entity_screening, cf28_cf29, pga, inspection
- `limit` (integer, optional) -- Default 10, max 50

**screen_entity**
- `entity_name` (string, required) -- Name to screen

**classify_product**
- `product_description` (string, required) -- Free-text description
- `origin_country` (string, optional) -- Default CN

**calculate_tariff**
- `hs_code` (string, required)
- `origin_country` (string, required)
- `destination_country` (string, optional) -- Default US
- `declared_value` (number, required)

**check_compliance**
- `hs_code` (string, required)
- `origin_country` (string, required)
- `entity_name` (string, optional)
- `product_description` (string, optional)

**identify_documents**
- `hs_code` (string, required)
- `origin_country` (string, required)
- `destination_country` (string, optional) -- Default US
- `declared_value` (number, optional)
- `transport_mode` (string, optional) -- ocean, air, truck, rail

**get_regulatory_signals**
- `category` (string, optional) -- TARIFF, COMPLIANCE, TRADE_POLICY, FTA
- `status` (string, optional) -- CONFIRMED, PROPOSED, DISCUSSED
- `country` (string, optional) -- Country code

**add_shipment_event**
- `shipment_id` (string, required)
- `event_type` (string, required) -- note, investigation, communication, analysis_finding
- `description` (string, required)
- `detail` (string, optional) -- Extended detail or full communication text

**request_shipper_documents**
- `shipment_id` (string, required)
- `documents` (array of string, required) -- e.g., ["Certificate of Origin", "Bill of Lading"]
- `deadline` (string, optional) -- e.g., "2025-02-15" or "30 days"
- `message` (string, optional) -- Message to include in request

**resolve_shipment_hold**
- `shipment_id` (string, required)
- `resolution_note` (string, required) -- Explanation of decision
- `target_status` (string, required, enum: ["cleared", "at_customs"])

---

## 4. Execution Dispatcher

### Entry Point

```python
async def execute_tool(
    tool_name: str,
    tool_input: dict[str, Any],
    session_factory: Any,
) -> str:
```

Returns a JSON string. Catches all exceptions and returns `{"error": "..."}` on failure.

### Dispatch Table

The `_dispatch()` function uses if/elif routing:

| Tool Name | Implementation |
|---|---|
| `lookup_shipment` | Direct SQLAlchemy query: `select(Shipment).where(Shipment.id == sid)` |
| `search_shipments` | Dynamic query builder with optional filters and `.order_by(created_at.desc()).limit(limit)` |
| `screen_entity` | Instantiates `DPSScreener()` and calls `.screen()` |
| `classify_product` | Calls `app.engines.e1_classification.classify()` |
| `calculate_tariff` | Calls `app.engines.e2_tariff.calculate_tariff()` |
| `check_compliance` | Calls `app.engines.e3_compliance.screen_compliance()` |
| `identify_documents` | Instantiates `DocumentEngine()` and calls `.execute()` |
| `get_regulatory_signals` | Instantiates `RegulatoryEngine()` and calls `.get_signals()` |
| `add_shipment_event` | Appends to `shipment.events` JSONB, calls `flag_modified()`, commits |
| `request_shipper_documents` | Appends `document_request` event to timeline with docs, deadline, message |
| `resolve_shipment_hold` | Validates `status == "held"`, calls `state_machine.transition()`, commits |

### Database Session Pattern

DB-backed tools use context managers:

```python
async with session_factory() as session:
    stmt = select(Shipment).where(Shipment.id == sid)
    result = await session.execute(stmt)
    shipment = result.scalar_one_or_none()
    # ... modify ...
    await session.commit()
```

### Search Shipments Dynamic Filtering

```python
stmt = select(Shipment)
if tool_input.get("status"):
    stmt = stmt.where(Shipment.status == tool_input["status"])
if tool_input.get("origin"):
    stmt = stmt.where(Shipment.origin == tool_input["origin"])
if tool_input.get("company_name"):
    stmt = stmt.where(Shipment.company_name.ilike(f"%{tool_input['company_name']}%"))
# ... etc
stmt = stmt.order_by(Shipment.created_at.desc()).limit(limit)
```

### Write Tool Event Structure

Events appended by `add_shipment_event`:

```python
{
    "timestamp": datetime.now(timezone.utc).isoformat(),
    "event_type": tool_input["event_type"],
    "description": tool_input["description"],
    "actor": "platform",
    "detail": tool_input.get("detail"),  # optional
}
```

Events appended by `request_shipper_documents`:

```python
{
    "timestamp": datetime.now(timezone.utc).isoformat(),
    "event_type": "document_request",
    "description": f"Document request: {', '.join(docs)}",
    "actor": "platform",
    "detail": {
        "documents": docs,
        "deadline": deadline,
        "message": message,
    },
}
```

---

## 5. Agentic SSE Loop

### Implementation: `_agentic_sse_stream()`

```python
async def _agentic_sse_stream(
    system_prompt: str,
    user_message: str,
    session_factory: Any,
    llm: Any,
    tools: list[dict[str, Any]],
) -> AsyncIterator[str]:
```

### Iteration Flow

```python
messages = [{"role": "user", "content": user_message}]

for _iteration in range(MAX_AGENT_ITERATIONS):  # MAX = 10
    response = await llm.complete_with_tools(
        system_prompt=system_prompt,
        messages=messages,
        tools=tools,
        temperature=0.3,
        max_tokens=4096,
    )

    content_blocks = _extract_content_blocks(response)

    has_tool_use = False
    tool_results = []

    for block in content_blocks:
        if block["type"] == "text":
            yield f'data: {json.dumps({"text": block["text"]})}\n\n'

        elif block["type"] == "tool_use":
            has_tool_use = True
            # Yield running event
            yield f'data: {json.dumps({"tool": name, "status": "running", "label": label})}\n\n'

            # Execute
            result_str = await execute_tool(name, input, session_factory)

            # Yield complete event
            yield f'data: {json.dumps({"tool": name, "status": "complete"})}\n\n'

            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block["id"],
                "content": result_str,
            })

    if not has_tool_use:
        break  # Agent produced final text response

    # Thread tool results back into the conversation
    assistant_content = _build_assistant_content(content_blocks)
    messages.append({"role": "assistant", "content": assistant_content})
    messages.append({"role": "user", "content": tool_results})

yield "data: [DONE]\n\n"
```

### Message Threading Mechanics

After each tool-using iteration, two messages are appended:

1. **Assistant message** with content blocks (both text and tool_use blocks preserved):
```python
[
    {"type": "text", "text": "Let me look up that shipment..."},
    {"type": "tool_use", "id": "toolu_abc", "name": "lookup_shipment", "input": {"shipment_id": "..."}}
]
```

2. **User message** with tool results:
```python
[
    {"type": "tool_result", "tool_use_id": "toolu_abc", "content": "{\"id\": \"...\", ...}"}
]
```

This threading format is compatible with both Anthropic and OpenAI conversation APIs.

---

## 6. LLM Translation Layer

### `_extract_content_blocks(response)`

Normalises provider-specific response formats into a uniform `list[dict]`:

**Anthropic path:**
```python
if hasattr(response, "content") and isinstance(response.content, list):
    for block in response.content:
        if block.type == "text":
            blocks.append({"type": "text", "text": block.text})
        elif block.type == "tool_use":
            blocks.append({
                "type": "tool_use",
                "id": block.id,
                "name": block.name,
                "input": block.input,
            })
```

**OpenAI path:**
```python
if hasattr(response, "choices"):
    msg = response.choices[0].message
    if msg.content:
        blocks.append({"type": "text", "text": msg.content})
    if msg.tool_calls:
        for tc in msg.tool_calls:
            blocks.append({
                "type": "tool_use",
                "id": tc.id,
                "name": tc.function.name,
                "input": json.loads(tc.function.arguments),
            })
```

### `_build_assistant_content(blocks)`

Reconstructs the content array for message threading:

```python
def _build_assistant_content(blocks):
    content = []
    for b in blocks:
        if b["type"] == "text":
            content.append({"type": "text", "text": b["text"]})
        elif b["type"] == "tool_use":
            content.append({
                "type": "tool_use",
                "id": b["id"],
                "name": b["name"],
                "input": b["input"],
            })
    return content
```

---

## 7. Serialisation Helpers

### `_serialize(obj)`

Recursively converts any object to JSON-safe primitives:

- `None`, `str`, `int`, `float`, `bool` -> pass through
- `uuid.UUID` -> `str(obj)`
- `datetime` -> `obj.isoformat()`
- `dict` -> recurse on values
- `list/tuple` -> recurse on elements
- Pydantic model -> `obj.model_dump()` -> recurse
- Fallback -> `str(obj)`

### `_shipment_to_dict(s: Shipment)`

Full detail conversion of a Shipment ORM object:

```python
{
    "id", "product", "origin", "destination", "status",
    "carrier", "tracking_number", "declared_value",
    "company_name", "hold_type", "events", "waypoints",
    "codes", "financials", "analysis", "created_at"
}
```

### `_shipment_summary(s: Shipment)`

Compact summary for search results:

```python
{
    "id", "product", "origin", "destination", "status",
    "company_name", "hold_type", "declared_value",
    "carrier", "created_at"
}
```

### Request Schema

```python
class ChatRequest(BaseModel):
    message: str = Field(..., max_length=50000)
    context: ChatContext = Field(default_factory=ChatContext)
    operator_mode: bool = False

class ChatContext(BaseModel):
    screen: str | None = None
    shipment_id: str | None = None
    product_id: str | None = None
    hs_code: str | None = None
    order_context: str | None = None
    operator_context: str | None = None
```
