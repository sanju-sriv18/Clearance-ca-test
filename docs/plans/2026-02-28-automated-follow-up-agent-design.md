# Automated Follow-Up Agent — Feature Design

> **Author:** Simon Kissler | **Co-Author:** Claude Code (Anthropic)
> **Date:** 2026-02-28
> **Status:** Design

---

## 1. Overview

The Automated Follow-Up Agent is an event-driven, per-broker AI agent that autonomously manages outbound communication with shippers and importers. Its purpose is to resolve missing documents, refine product classifications, and enforce filing deadlines — without requiring the broker to manually chase each party for each entry.

The agent operates with full autonomy but complete observability. Brokers see every action the agent takes, every reasoning step behind each decision, and can override any action after the fact. The licensed broker retains final authority; the agent is an autonomous colleague, not a replacement.

### Core Characteristics

- **Event-driven**: Reacts to domain events (checklist changes, classification results, inbound shipper responses, deadline proximity). No polling loop.
- **Self-scheduling**: The agent creates delayed triggers for itself ("follow up in 12 hours if no response"). These are stored as pending events that re-invoke the agent with the original entry context when they fire.
- **Natural language rules**: Each broker describes their follow-up preferences in natural language. The agent interprets these rules, shows its interpretation reasoning to the broker for confirmation, and applies them to determine which entries qualify.
- **Identity-configurable**: Each broker chooses whether automated messages go out under their name or as a platform assistant identity.
- **Override model**: Per-entry opt-out now; per-shipper override planned for future once robust customer handling is in place.

---

## 2. Trigger Events

Domain events that wake the agent and cause it to evaluate an entry:

| Trigger Event | Source | Agent Evaluates |
|---|---|---|
| `ChecklistUpdatedEvent` | Checklist refresh or document upload | Are there new missing documents for an enrolled entry? |
| `ClassificationProducedEvent` | E1 Classification engine | Is confidence below threshold? Are there unanswered refinement questions? |
| `DeclarationCreatedEvent` | Entry filing created | Does this entry match the broker's enrollment rules? If so, begin tracking. |
| `BrokerAssignedEvent` | Entry claimed by a broker | Same — evaluate enrollment against the broker's rules. |
| `FollowUpDueEvent` | Self-scheduled delayed trigger | Re-evaluate the entry: has the shipper responded since the last follow-up? If not, escalate or retry. |
| `InboundMessageEvent` | Shipper responds via Resolution Center | Process the response: validate documents, incorporate classification answers, update checklist. Acknowledge and nudge if items remain. |

---

## 3. Agent Actions

What the agent can do when a trigger fires:

| Action | Description |
|---|---|
| **Request documents** | Send a structured document request to the shipper with specific items needed, a deadline, and context explaining why each document is required. |
| **Ask classification questions** | Send follow-up questions to the shipper about product characteristics (material composition, intended use, physical form). Questions are translated from trade jargon into plain language the shipper can answer. |
| **Send deadline reminder** | Warn the shipper of approaching CBP response deadlines, General Order deadlines, or document submission deadlines. |
| **Process inbound response** | Validate uploaded documents against what was requested, incorporate classification answers into the product description, and update checklist state. |
| **Acknowledge and nudge** | After a partial upload (shipper provides some but not all requested items), wait a brief delay then send a thank-you confirming receipt and reminding the shipper of remaining items. |
| **Schedule delayed trigger** | Create a `FollowUpDueEvent` to fire after a specified delay (hours or days), carrying the entry context so the agent can re-evaluate when the trigger fires. |
| **Escalate to broker** | After max follow-up attempts, on anomaly, or when manual judgment is required, notify the broker with full context and the agent's reasoning trail. |
| **Log reasoning** | Every evaluation and decision is logged to the entry's follow-up audit trail with full reasoning transparency. |

### Self-Scheduling Flow

```
Event arrives → Agent evaluates entry state
  → Takes action (e.g., sends document request to shipper)
  → Schedules FollowUpDueEvent in 24 hours
  → ...24 hours later...
  → FollowUpDueEvent fires → Agent re-evaluates
    → Shipper responded? → Process response, close loop or continue
    → No response? → Send reminder, schedule another trigger
    → Max attempts reached? → Escalate to broker
```

### Acknowledge-and-Nudge Flow

When a shipper uploads a document while other items remain outstanding:

```
InboundMessageEvent (document uploaded) → Agent processes document
  → Document valid? Update checklist item to "uploaded"
  → Still missing items? Schedule acknowledgment in ~20 minutes
    (brief delay allows shipper to complete batch uploads)
  → Acknowledgment fires:
    "Thank you for the commercial invoice — it looks good.
     We still need the certificate of origin and packing list
     to complete the filing. Can you send those by March 5?"
  → Reset the main follow-up timer (shorter than normal since shipper is active)
```

This pattern treats the interaction as a human conversation: it acknowledges what the shipper has done, confirms it meets requirements, and gently reminds them of what remains.

---

## 4. Natural Language Rules and Enrollment

### Configuration Flow

1. Broker opens the Follow-Up Agent settings (accessible from the Dashboard sidebar or a dedicated configuration screen).
2. Broker describes their follow-up rules in natural language:
   - *"Follow up on all CN-origin entries where the checklist is below 80%"*
   - *"Only chase documents for entries over $50K declared value"*
   - *"Be aggressive on anything with a CF-28 deadline within 10 days"*
   - *"Don't bother with entries that are already submitted"*
   - *"Be friendly and informal with small importers"*
3. The agent interprets the rules and plays back both the structured interpretation and its reasoning:
   - **Enrollment rule**: origin_country = 'CN' AND completion_percentage < 80
     - *Reasoning: "You said 'all CN-origin entries where the checklist is below 80%'. I interpret 'CN-origin' as entries where the origin country code is CN (China), and 'checklist below 80%' as entries where the filing checklist completion percentage is strictly less than 80%. This means an entry at exactly 80% would NOT be enrolled."*
   - **Value filter**: declared_value > 50,000 USD
     - *Reasoning: "You said 'only chase documents for entries over $50K declared value'. I interpret this as a hard filter: entries at or below $50,000 USD declared value are excluded from automated follow-up entirely, regardless of other rules."*
   - **Urgency override**: CF-28 deadline < 10 days → urgency = high
   - **Exclusion**: filing_status in (submitted, accepted, released)
4. Broker confirms or refines until the rules match their intent.
5. Rules are stored as both the original natural language and the agent's structured interpretation with reasoning.

### Rule Storage

```
FollowUpRuleSet:
  broker_id: UUID
  natural_language_rules: text           # broker's original words
  structured_interpretation: JSONB       # agent's parsed rules with reasoning
  confirmed_at: timestamp                # when broker approved the interpretation
  identity_mode: "broker" | "assistant"  # whose name messages go out under
  tone_preferences: text                 # natural language tone guidance
  guardrails:                            # see Section 6
    min_interval_hours: 24
    max_attempts: 3
    quiet_hours: [22:00, 07:00]
    weekend_enabled: false
    ...
```

### Per-Entry Override

From Entry Detail, the broker can toggle automated follow-up on or off for a specific entry:
- **Disabled**: Entry is excluded from all automated follow-up regardless of whether it matches the rules.
- **Enabled (explicit)**: Entry is included in automated follow-up even if it does not match the rules.
- **Default (no override)**: Entry follows the rules.

### Future: Per-Shipper Override

Once robust shipper/customer modeling is in place, brokers will be able to set follow-up preferences at the shipper level:
- *"Never auto-follow-up with Shipper X"* (they prefer to handle everything through their own logistics team)
- *"Always follow up aggressively with Shipper Y"* (they're responsive but need reminders)
- Per-shipper tone and cadence preferences

---

## 5. Response Processing

### Document Response Processing

When a shipper uploads documents or sends a message:

1. `InboundMessageEvent` fires, waking the agent with the entry context.
2. Agent evaluates the response:
   - **Document uploaded**: Validates type against the request (is this actually a commercial invoice?). Checks basic completeness (non-empty, reasonable file size). Updates the checklist item status to "uploaded."
   - **Text response**: Extracts actionable information (e.g., "the certificate of origin is being processed by our freight forwarder, expect it Tuesday").
3. Agent updates the follow-up log with reasoning and outcome.
4. Agent re-evaluates remaining items:
   - **Still missing items**: Schedules an acknowledgment-and-nudge after the `acknowledgment_delay_minutes` (default 20 min), then resets the main follow-up timer.
   - **All items received**: Closes the follow-up loop. Notifies broker: "All requested documents received for entry X."

### Classification Refinement Loop

1. E1 produces a LOW or MEDIUM confidence classification, firing `ClassificationProducedEvent`.
2. Agent generates plain-language questions for the shipper (translated from the classification engine's technical follow-up questions):
   - Engine asks: *"Is the product classified under GRI 3(b) by essential character?"*
   - Agent sends: *"Could you tell us the primary material this product is made of? For example: mostly cotton, mostly polyester, or a mix."*
3. Shipper responds with answers.
4. Agent feeds answers back into the classification chat endpoint, enriching the product description.
5. If confidence improves to HIGH: agent accepts classification, updates entry, notifies broker.
6. If confidence still below threshold: agent sends the next round of questions (up to `classification_max_rounds` guardrail).
7. If max rounds reached without HIGH confidence: escalates to broker with full context: "Classification requires manual review. Here's what the shipper told us and where confidence stands."

### Anomaly Handling

The agent escalates to the broker rather than acting autonomously when:

- Shipper's response contradicts the request (uploaded document doesn't match what was asked for)
- Shipper disputes the classification or declared valuation
- Response suggests potential compliance issues (mentions sanctioned entities, restricted regions, or raises red flags)
- Shipper is unresponsive after maximum follow-up attempts
- Agent encounters ambiguity it cannot resolve with confidence

---

## 6. Observability and Broker Override

### Follow-Up Activity Log

Each entry gets a follow-up audit trail, visible in the Entry Detail screen alongside the existing Audit Trail Panel. Every log entry contains:

| Field | Description |
|---|---|
| **Timestamp** | When the action occurred |
| **Trigger** | What event woke the agent (e.g., "ChecklistUpdatedEvent — commercial_invoice now missing") |
| **Rule match reasoning** | Why this entry qualified, including the agent's interpretation of the broker's natural language rules and how it applied them to this entry's data |
| **Evaluation** | What the agent assessed (e.g., "3 documents missing: commercial_invoice, certificate_of_origin, packing_list. Last contact: none. Deadline: CF-28 due in 18 days.") |
| **Decision reasoning** | Why the agent chose this specific action (e.g., "First contact for this entry. Three required documents missing. No prior follow-up. Sending initial document request with 5-day deadline.") |
| **Action taken** | What the agent did (e.g., "Sent document request to shipper@example.com requesting commercial_invoice, certificate_of_origin, packing_list with deadline 2026-03-05") |
| **Scheduled next** | What delayed trigger the agent set for itself (e.g., "FollowUpDueEvent scheduled for 2026-03-01 09:00 UTC") |
| **Outcome** | Filled in when the follow-up resolves (e.g., "Shipper uploaded commercial_invoice and packing_list on 2026-02-29. Certificate of origin still missing.") |

### Broker Override Capabilities

| Override | Effect |
|---|---|
| **Disable follow-up on entry** | Agent stops all automated actions for this entry immediately |
| **Undo last action** | Mark the last outbound message as retracted; send a correction to the shipper if possible |
| **Adjust urgency** | Override the agent's urgency assessment for this entry (force high urgency or reduce to low) |
| **Edit pending message** | Before a scheduled follow-up fires, the broker can review and edit the draft message |
| **Reassign to manual** | Take over follow-up manually; agent stops but its full reasoning trail remains visible for context |

### Dashboard Integration

The broker Dashboard gets a new "Agent Activity" widget showing:

- How many entries are being actively followed up by the agent
- Recent actions taken in the last 24 hours (with one-click access to the reasoning trail)
- Pending scheduled triggers (upcoming follow-ups)
- Entries requiring escalation (max attempts reached, anomalies detected)

---

## 7. Guardrails and Configuration

### Default Guardrails

Each broker can configure these bounds. The AI agent operates within them but can tighten (never loosen) timing based on context.

| Guardrail | Default | Description |
|---|---|---|
| `min_interval_hours` | 24 | Minimum time between follow-ups for the same entry (normal cadence) |
| `acknowledgment_delay_minutes` | 20 | Delay before sending "thanks + still need" after a partial upload. Allows time for batch uploads. |
| `max_attempts` | 3 | Maximum follow-up attempts per outstanding item before escalating to broker |
| `quiet_hours_start` | 22:00 | No outbound messages after this time (broker's timezone) |
| `quiet_hours_end` | 07:00 | No outbound messages before this time |
| `weekend_enabled` | false | Whether to send follow-ups on Saturdays and Sundays |
| `escalation_notify` | true | Whether to push-notify the broker when an escalation occurs |
| `classification_max_rounds` | 3 | Maximum classification Q&A rounds with the shipper before escalating |
| `deadline_warning_days` | 7 | How many days before a deadline to start sending warnings |

### AI Cadence Overrides Within Guardrails

The agent can tighten timing based on urgency signals, but never violates the guardrail bounds:

- **CF-28 deadline < 7 days**: May follow up every 12 hours instead of 24 (but never below `min_interval_hours` if the broker sets it higher)
- **General Order deadline approaching**: Same urgency boost
- **High declared value entry**: May start follow-up sooner after enrollment
- **Shipper currently active** (just uploaded a document): Acknowledgment fires on `acknowledgment_delay_minutes` rather than waiting for the full `min_interval_hours`

### Tone Configuration

Part of the broker's natural language rules can include tone preferences. These are stored alongside the rules and applied during message generation:

- *"Be friendly and informal with small importers"*
- *"Use formal language for large corporate shippers"*
- *"Always include our phone number for questions"*
- *"Sign off with my name and license number"*

---

## 8. Data Model Summary

### New Models

**`FollowUpRuleSet`** — One per broker. Stores natural language rules, structured interpretation with reasoning, identity mode, tone preferences, and guardrail configuration.

**`FollowUpTracker`** — One per enrolled entry. Tracks the agent's state for that entry: current outstanding items, follow-up attempt count per item, last contact timestamp, next scheduled trigger, and the full reasoning log.

**`FollowUpScheduledTrigger`** — Pending delayed triggers. Each has a `fire_at` timestamp, the entry context (entry_id, broker_id, trigger_reason), and a status (pending, fired, cancelled). A lightweight scheduler checks for due triggers and dispatches `FollowUpDueEvent` events.

### Extended Models

**`EntryFiling`** — Gains a `follow_up_enabled` field (nullable boolean: null = use rules, true = force on, false = force off).

**`BrokerMessage`** — Gains an `automated` boolean flag and `follow_up_tracker_id` foreign key so messages generated by the agent are tagged and linked to the follow-up trail.

---

## 9. Integration Points

| System | Integration |
|---|---|
| **Event Bus (NATS JetStream)** | Agent subscribes to trigger events via durable JetStream consumers. Self-scheduled triggers are dispatched as `FollowUpDueEvent` on the event bus. |
| **Messaging (BrokerMessage)** | Agent uses the existing `send_broker_request` tool pattern to create outbound messages. Messages are tagged as automated. |
| **Checklist** | Agent reads `EntryChecklistState` to determine missing documents and completion percentage. |
| **Classification Chat** | Agent uses the existing `/entries/{id}/classification-chat` endpoint to feed shipper answers and get refined classifications. |
| **Filing Readiness Gates** | Agent monitors gate states to determine what's blocking a filing. |
| **LLM Service** | Agent uses the LLM for: interpreting natural language rules, generating shipper-friendly messages, translating classification questions, and processing inbound responses. |
| **Redis** | Distributed locks for trigger scheduling. Trigger storage as a sorted set (score = fire_at timestamp) for efficient due-trigger scanning. |

---

## 10. Future Considerations

- **Per-shipper profiles**: Once shipper/customer modeling is implemented, follow-up rules and tone can be configured per shipper, not just per broker.
- **Multi-channel delivery**: Currently messages go through the platform's messaging system. Future: direct email (SMTP), SMS, or EDI integration so follow-ups reach shippers on their preferred channel.
- **Learning from outcomes**: The agent could learn which follow-up strategies are most effective per shipper (faster response to emails vs. portal messages, more responsive on weekdays, etc.) and adapt its approach over time.
- **Cross-entry coordination**: If a broker has 5 entries for the same shipper, the agent could batch follow-ups into a single message rather than sending 5 separate requests.
- **Regulatory deadline intelligence**: Integration with the Regulatory Intelligence feed to proactively warn shippers about upcoming regulatory changes that affect their pending entries.
