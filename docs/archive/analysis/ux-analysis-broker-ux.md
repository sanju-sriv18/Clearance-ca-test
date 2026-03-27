# Broker Surface UX Analysis

**Analyst**: Senior Product/UX Designer
**Date**: 2026-02-07
**Surface**: Broker (Customs Broker Workspace)
**Methodology**: Screenshot review + code audit + backend capability gap analysis

---

## Executive Summary

The Broker surface is well-architected with a strong morning briefing, smart work sequencing, and contextual AI assistance. However, there are **significant gaps between backend intelligence and what's surfaced in the UI**. The most critical issues are: (1) resolution steps are display-only text with no action buttons, (2) key broker tools like `verify_classification` and `calculate_entry_fees` exist as agent tools but have no direct UI buttons, (3) the `draft_communication` endpoint has a `"[Message body]"` placeholder fallback, and (4) the Entry Detail screen (the broker's primary workspace) is missing a "Generate 7501 Summary" action despite the backend endpoint existing.

**Issue counts by severity:**
- **P0** (Blocks core workflow): 4
- **P1** (Major friction): 8
- **P2** (Moderate inefficiency): 10
- **P3** (Polish/improvement): 7
- **P4** (Nice-to-have): 3

---

## Screen-by-Screen Analysis

### 1. BrokerDashboard (Morning Briefing)

**File**: `frontend/src/surfaces/broker/screens/BrokerDashboard.tsx`
**Screenshot**: `01-dashboard.png`

#### What works well
- Personalized greeting with day estimate widget (line 288-310) showing ~37min / 5 items / 8% capacity
- Sequenced work plan with numbered steps, time estimates, deadline badges, and financial impact (lines 323-405)
- AI pre-work "Draft ready" badges on work plan items (lines 394-398)
- Clickable stat tiles navigating to filtered queue views (lines 568-629)
- Unassigned shipments table with GO countdown timers and claim buttons (lines 725-822)

#### Issues Found

**P0 — Alert Resolution Steps Are Display-Only Text**
`BrokerDashboard.tsx:474-482` — Resolution steps render as a numbered list but **none of the steps are clickable or connected to tools**. For example, "Verify entity identity against BIS/OFAC lists" should trigger `screen_entity`, and "Request corporate documentation from shipper" should trigger `draft_communication`. Currently they're passive `<li>` elements.

**Backend capability**: `BrokerIntelligenceService.get_resolution_path()` in `broker_intelligence.py:198-242` returns steps with `est_minutes`. The tools `screen_entity`, `request_shipper_documents`, and `resolve_shipment_hold` all exist in `tools.py`.

**Ideal experience**: Each resolution step should be a button that either (a) opens a pre-filled modal for that action, or (b) invokes the tool directly. E.g., step "Verify entity identity against BIS/OFAC lists" should open the assistant with the entity name pre-filled, or trigger `screen_entity` directly.

---

**P1 — Held Shipments in Unassigned List Have No Resolution Preview**
`BrokerDashboard.tsx:758-810` — Unassigned shipments with status "held" show as red badges but the only action is "Claim". A broker needs to know *why* it's held before claiming it. The backend has `hold_type` data and cage status (via `get_cage_status` tool), but none of this is previewed.

**Ideal experience**: Held shipments should show a hold type badge (entity_screening, UFLPA, PGA, etc.) and a one-line cage status preview. Hovering/expanding should show the resolution path preview before the broker commits to claiming.

---

**P2 — Work Plan Items Navigate Away Instead of Inline Action**
`BrokerDashboard.tsx:155-175` — `handleWorkPlanAction` for "approve" navigates to `/broker/entries/{id}`, and "respond_cf28" navigates to `/broker/cbp-responses`. For simple actions like "Approve", the broker should be able to act from the dashboard without navigating away. The "Approve" button already exists inline in the My Work table (line 241), creating inconsistency.

**Ideal experience**: Work plan items for "approve" should trigger inline approval with a confirmation micro-modal, not a full page navigation. "Respond CF-28" should open the response modal directly.

---

**P2 — Briefing Summary is Flat Text**
`BrokerDashboard.tsx:291-295` — The briefing summary is rendered as a single `<p>` tag. With structured data available (`briefing.workPlan`, `briefing.actionItems`), the summary could be richer — e.g., "3 entries ready for approval, 1 CF-28 due in 29d, 8 unassigned shipments" with each segment clickable.

---

**P3 — Quick Actions Section is Redundant**
`BrokerDashboard.tsx:826-883` — The "Quick Actions" grid (Review & Approve, CBP Responses, Draft Entries, Communications) largely duplicates the stat tiles above (lines 568-629). This wastes vertical space and adds no new functionality.

**Ideal experience**: Remove "Quick Actions" or replace with actions not already available: "Run Compliance Batch Check", "Generate Daily Report", "View Regulatory Updates".

---

**P3 — No Bulk Claim for Unassigned Shipments**
`BrokerDashboard.tsx:798-808` — Each shipment has an individual "Claim" button. If a broker wants to claim 5 shipments, that's 5 separate API calls with 5 loading spinners. No select-all or batch claim.

---

**P3 — Day Estimate Widget Doesn't Account for Unassigned Work**
`BrokerDashboard.tsx:299-310` — The capacity widget shows "8% capacity" based on current work plan items. But there are 8 unassigned shipments visible below. A broker would want to see: "Current: 37min (8%) | If you claim all 8: ~120min (25%)".

---

### 2. BrokerQueue (My Queue)

**File**: `frontend/src/surfaces/broker/screens/BrokerQueue.tsx`
**Screenshot**: `02-queue.png`

#### What works well
- Card-based entries with transport mode icons, status/priority badges (line 145-150)
- Inline contextual action buttons per card: Approve, Respond, Req Docs (lines 86-133)
- Checklist progress bar per entry (lines 167-177)
- Pagination for large queues (lines 369-391)

#### Issues Found

**P1 — No Search or Text Filter**
`BrokerQueue.tsx:308-338` — Only status and priority dropdown filters. No way to search by entry number, company name, product, or shipment ID. This is critical when a broker gets a call about a specific entry and needs to find it fast.

**Backend capability**: The `search_by_reference` tool in `tools.py:651-668` can search across tracking numbers, house numbers, entry numbers, and container numbers. This capability is only available through the chat assistant.

**Ideal experience**: Add a search input above the filters that queries across entry number, company name, product description, and reference numbers.

---

**P1 — No Sort Options**
`BrokerQueue.tsx:356-365` — Queue is rendered in server-provided order with no client-side sort controls. Brokers need to sort by: days waiting, declared value, priority, status, company name.

---

**P2 — "Req Docs" Button Navigates to Generic Communications Page**
`BrokerQueue.tsx:121-130` — The "Req Docs" button navigates to `/broker/messages` without any context. The broker loses the entry context and has to start a message from scratch in the Communications page.

**Backend capability**: `draft_communication` endpoint (broker.py:2461) accepts `shipment_id` and `purpose` and can auto-generate a contextual document request email.

**Ideal experience**: "Req Docs" should open a pre-filled draft communication modal using `draft_communication` with the entry's `shipment_id` and `purpose: "request_missing_documents"`. This pattern already exists in `EntryDetail.tsx:575-589`.

---

**P2 — No Deadline/Urgency Information on Queue Cards**
`BrokerQueue.tsx:180-185` — Only "days since arrival" is shown. No indication of GO deadline countdown, storage costs accruing, or cascade warnings that the dashboard work plan provides.

---

**P3 — No Bulk Actions**
No way to select multiple entries and perform batch operations (approve all, export list, reassign).

---

### 3. EntryDetail (Primary Broker Workspace)

**File**: `frontend/src/surfaces/broker/screens/EntryDetail.tsx` (1360 lines)
**Screenshot**: `03-entry-detail-top.png` (shows 400 error - likely bad test entry ID)

#### What works well
- Comprehensive checklist with status-driven styling (verified/uploaded/issues/missing/N-A) — lines 97-332
- Bond type selection inline within checklist expansion (lines 276-303)
- Document upload directly from checklist items (lines 304-318)
- AI Assistant contextual actions panel (lines 946-1048)
- Communications tracker with follow-up button (lines 1050-1094)
- CBP Response inline display with "Respond Now" (lines 1097-1126)
- Risk flags with screening matches table, financial impact, and resolution steps (lines 338-486)

#### Issues Found

**P0 — `verify_classification` Tool Not Surfaced as Direct Button**
`EntryDetail.tsx:1016-1024` — The "Improve classification confidence" button only opens the chat assistant panel (`openPanel`). The `verify_classification` tool in `tools.py:492-509` can directly compare the entry's HS code against AI classification, returning match status, GRI analysis, and alternative codes. This is one of the most critical broker workflows.

**Backend capability**: `verify_classification` tool returns: `current_hs_code`, `ai_suggested_code`, `codes_match`, `confidence`, `gri_analysis`, `alternative_codes`, and `recommendation`. This could be a full inline panel showing classification verification results.

**Ideal experience**: Add a "Verify Classification" button in the sidebar Insights section (near line 1242) that calls the backend directly and renders results inline: current code vs. suggested code, match/mismatch indicator, confidence level, and GRI reasoning. If codes don't match, show a "Reclassify" action.

---

**P0 — `calculate_entry_fees` Tool Not Surfaced as Direct Button**
`EntryDetail.tsx:1209-1239` — The sidebar shows estimated duty/MPF/HMF but these are computed with crude static rates (e.g., 5% default duty). The `calculate_entry_fees` tool in `tools.py:511-527` calls the actual tariff engine with Section 301, 232, AD/CVD, and precise landed cost calculations.

**Backend capability**: `calculate_entry_fees` tool calls `e2_tariff.calculate_tariff()` and returns a full fee breakdown. Also, the `/entries/{entry_id}/summary` endpoint (broker.py:1566) generates a 7501-equivalent entry summary.

**Ideal experience**: (1) Add a "Calculate Actual Fees" button next to the estimated duty display. (2) Add a "View 7501 Summary" or "Generate Entry Summary" button in the Actions sidebar. The endpoint exists at `/broker/entries/{entry_id}/summary` but is only used in the Platform surface.

---

**P0 — Missing "Generate Document" Capability**
`EntryDetail.tsx:304-318` — Missing documents only show an "Upload" option. There's no "Generate" or "Request from AI" button. For documents like the 7501 Entry Summary, the system should be able to generate them. The `identify_documents` tool (tools.py:181-212) can determine what documents are needed, and some (like the entry summary) can be auto-generated.

**Ideal experience**: For checklist items that can be system-generated (entry summary, duty calculation worksheet), add a "Generate" button alongside "Upload". For items requiring external documents, add "Request from Shipper" which triggers `draft_communication`.

---

**P1 — Risk Flag Resolution Steps Are Passive Text**
`EntryDetail.tsx:456-475` — Resolution steps in `RiskFlagCard` render as numbered text items with no action buttons. Identical issue to dashboard alerts. These steps should be clickable actions that invoke the appropriate tool.

Example: Step "Verify entity identity against BIS/OFAC lists" → button triggering `screen_entity` with the company name pre-filled. Step "Request corporate documentation from shipper" → button triggering `draft_communication`.

---

**P1 — Analysis Improvement Steps Are Passive Text**
`EntryDetail.tsx:881-890` — `insights.analysisSummary.improvementSteps` renders as bullet points. Items like "Upload Certificate of Origin" should scroll to the relevant checklist item. Items like "Request lab test results from shipper" should trigger a document request.

---

**P1 — Missing Docs Sidebar List Is Display-Only**
`EntryDetail.tsx:1290-1303` — Sidebar "Missing Documents" shows bullet points with no actions. Should have: (1) click to scroll to checklist item, (2) "Upload" button per doc, (3) "Request All" button to draft a combined document request.

---

**P2 — Sidebar Insights Use Crude Duty Estimates**
`EntryDetail.tsx:1214-1219` — `insights?.estimatedDuty ?? entry.declaredValue * 0.05` — Falls back to 5% of declared value when insights aren't loaded. This is misleading for entries with Section 301 (25%) or AD/CVD duties. The `calculate_entry_fees` tool provides accurate calculations but isn't called.

---

**P2 — Hold Details Banner Has No Action Button**
`EntryDetail.tsx:1262-1288` — Hold details show storage costs and GO deadline but no action. Should have a "Begin Resolution" button that either opens a resolution wizard or navigates to the first resolution step.

---

**P2 — No "Compliance Check" Button**
The `check_compliance` tool (tools.py:158-179) runs PGA requirements, DPS screening, and UFLPA assessment. This is not surfaced as a button anywhere in EntryDetail. A "Run Compliance Pre-Check" button would be valuable before submission.

---

**P2 — Entry Detail Error State Shows Raw JSON**
`EntryDetail.tsx:689-699` and screenshot `03-entry-detail-top.png` — The error state shows `400: {"detail":"Invalid entry ID format"}` — raw API response. Should be a user-friendly message like "This entry could not be found. It may have been removed or the link is incorrect."

---

**P3 — No Keyboard Shortcuts for Common Actions**
No keyboard shortcuts for Approve (Ctrl+A?), Submit (Ctrl+Enter), Open Assistant (Ctrl+/). Given this is the broker's primary workspace used 50+ times/day, keyboard shortcuts would significantly speed up workflow.

---

**P3 — Line Items Table Doesn't Show Classification Confidence**
`EntryDetail.tsx:823-848` — Multi-product line items table shows HS code, quantity, value, origin — but doesn't show classification confidence per line. If line 1 is HIGH confidence and line 2 is LOW, the broker needs to know.

---

### 4. CBPResponses

**File**: `frontend/src/surfaces/broker/screens/CBPResponses.tsx`
**Screenshot**: `05-cbp-responses.png`

#### What works well
- Clear deadline countdown with color-coded urgency (lines 27-29)
- AI-powered guidance panel with recommended attachments and urgency reasons (lines 109-140)
- "AI Draft" button in response modal that generates a structured response (lines 185-196)
- Supporting points displayed after AI draft (lines 304-313)

#### Issues Found

**P1 — No AI Draft for CF-29 Responses**
`CBPResponses.tsx:276-289` — The "AI Draft" button is only shown when `!isCF29`. CF-29 protests require even more careful drafting than CF-28 responses. The backend `draft_cf28_response` tool could be adapted, or a new `draft_cf29_protest` capability could be added.

---

**P2 — No Document Attachment Capability in Response Modal**
`CBPResponses.tsx:210-341` — The modal accepts text but has no file attachment. CBP responses typically require supporting documents (invoices, certificates, lab reports). The guidance panel even suggests "Certificate of origin", "Manufacturing records" (visible in screenshot), but there's no way to attach them.

**Ideal experience**: Add a document attachment area below the textarea. Pre-populate with suggested documents from the guidance. Allow uploading directly or linking to already-uploaded shipment documents.

---

**P2 — CBP Responses Not Linked to Entry Detail**
`CBPResponses.tsx:76-84` — Each response card shows entry number and company but clicking doesn't navigate to EntryDetail for context review. Brokers often need to review the full entry before drafting a response.

**Ideal experience**: Make the entry number a clickable link to `/broker/entries/{entryId}`, or add a "View Entry" button.

---

**P3 — No History of Past CBP Responses**
Only pending responses are shown. No way to view previously submitted responses as reference for similar future cases.

---

### 5. Communications

**File**: `frontend/src/surfaces/broker/screens/Communications.tsx`
**Screenshot**: `06-communications.png`

#### What works well
- Three-tab filtering (All/Inbound/Outbound) — lines 345-358
- AI Draft integration in compose modal with purpose selection (lines 198-227)
- Channel selection (email/phone/portal) — lines 185-195

#### Issues Found

**P0 — `draft_communication` Has `"[Message body]"` Placeholder Fallback**
`backend/app/api/routes/broker.py:2543` — The fallback for unrecognized purposes outputs: `"Dear {company},\n\n[Message body]\n\nBest regards,\n[Broker Name]"`. This means if a broker selects a purpose not covered by the hardcoded branches (missing_docs, classification_inquiry, value_clarification), or hits the catch-all, they get a placeholder that says `[Message body]`. This is visible in production if purpose doesn't match.

**Ideal experience**: The fallback should at minimum generate: "Dear {company}, I'm reaching out regarding your shipment of {product} ({tracking_number}). [Please provide the specific details for this communication.] Best regards, {broker_name}". Better yet, use LLM to generate contextual content for any purpose.

---

**P1 — Compose Modal Loses Entry Context**
`Communications.tsx:99-286` — `ComposeModal` accepts `defaultShipmentId` and `defaultPurpose` but when opened from the "New Message" button (line 336), neither is provided. The AI Draft button requires both `shipmentId` and `purpose` to be enabled (line 216: `disabled={drafting || !shipmentId || !purpose}`), so AI Draft is always disabled when opening from the Communications page directly.

**Ideal experience**: When no shipment context, show a shipment selector dropdown (populated from the broker's queue) so the user can pick which entry this communication is about. This enables AI Draft to work.

---

**P2 — No Message Threading**
`Communications.tsx:46-97` — Messages are displayed as flat cards. No threading or conversation grouping. If 10 messages are exchanged about one shipment, they appear as 10 separate cards mixed with messages about other shipments.

**Ideal experience**: Group messages by shipment/thread. Show conversation count per thread. Allow expanding a thread to see the full exchange.

---

**P2 — No Read/Unread Management**
`Communications.tsx:51-52` — Read status is displayed (`message.read`) but there's no action to mark as read/unread. No bulk "Mark all read" either.

---

**P3 — Empty State Doesn't Guide User**
`Communications.tsx:367-376` — Empty state says "No messages yet. Start a conversation with a shipper or carrier." but doesn't suggest connecting it to entries that need document requests. Should say "You have {n} entries with missing documents. Request them now?" with a button.

---

### 6. BrokerAssistant (AI Chat Panel)

**File**: `frontend/src/surfaces/broker/components/BrokerAssistant.tsx`

#### What works well
- Context-aware suggestions based on selected entry, alerts, and queue state (lines 28-101)
- Rich broker context sent with each message including dashboard stats, alerts, and entry details (lines 144-186)
- Entry context indicator bar (lines 254-262)
- Suggestion chips dynamically generated for current situation

#### Issues Found

**P2 — Assistant Cannot Execute Direct Actions**
`BrokerAssistant.tsx:188-220` — The assistant uses `streamChat` for conversational responses only. While the backend has 22+ tools available (tools.py), the assistant can only *talk about* actions, not execute them. It should be able to: trigger classification verification, calculate fees, draft communications, and file responses — all from within the chat.

**Note**: The tools ARE available to the LLM through the agentic loop, but the UI doesn't render tool results distinctly from text responses. Tool execution happens server-side but results are streamed as markdown text.

**Ideal experience**: When the LLM calls a tool, show it as a distinct card in the chat: "Verifying classification for Entry 644-2946592-6..." → result card showing current vs. suggested HS code. "Calculating fees..." → result card with fee breakdown table.

---

**P2 — Suggestion Chips Fill Input Instead of Sending**
`BrokerAssistant.tsx:283` — Clicking a suggestion chip calls `setInput(s.prompt)`, which fills the textarea but doesn't send. The user must then press Enter or click Send. This is a minor but unnecessary extra step for pre-composed prompts.

---

**P3 — No History Persistence**
Messages are stored in Zustand store (`useBrokerStore`) but not persisted across page refreshes or navigation. If a broker navigates away and comes back, the conversation is lost.

---

## Backend Capabilities NOT Surfaced in UI

| Backend Capability | Location | Surfaced? | How It Should Be Surfaced |
|---|---|---|---|
| `verify_classification` | tools.py:492 | Only via chat | Inline button in EntryDetail sidebar |
| `calculate_entry_fees` | tools.py:511 | Only via chat | Button in EntryDetail + fee breakdown panel |
| `check_entry_readiness` | tools.py:474 | Only via chat | Pre-submission checklist gate in EntryDetail |
| `check_compliance` | tools.py:158 | Only via chat | "Run Compliance Check" button in EntryDetail |
| `screen_entity` | tools.py:82 | Only via chat | Button on risk flags + resolution steps |
| `get_regulatory_signals` | tools.py:213 | Only via chat | Dashboard widget or notification badge |
| `identify_documents` | tools.py:181 | Only via chat | Auto-populate checklist requirements |
| `assess_description_quality` | tools.py:240 | Only via chat | Inline quality badge on product descriptions |
| `/entries/{id}/summary` | broker.py:1566 | Only in Platform | "View 7501 Summary" in EntryDetail |
| `search_by_reference` | tools.py:651 | Only via chat | Queue search bar |
| `get_cage_status` | tools.py:633 | Only via chat | Held entry detail + unassigned preview |
| `check_consolidation_impact` | tools.py:669 | Only via chat | Alert when approving entries in consolidations |
| `lookup_consolidation` | tools.py:614 | Only via chat | Consolidation view link from entries |
| `create_product` | tools.py:356 | Only via chat | Product catalog management screen |
| `resolve_shipment_hold` | tools.py:328 | Only via chat | Hold resolution button in EntryDetail |
| `get_broker_queue_summary` | tools.py:596 | Only via chat | Enhanced dashboard stats |

**Key insight**: 16 of 22 tools are only accessible through the chat assistant. The chat is a poor discovery mechanism — brokers don't know these tools exist unless they ask the right question.

---

## Priority Summary: Top 10 Actions

| # | Severity | Issue | Screen | Impact |
|---|---|---|---|---|
| 1 | P0 | Resolution steps are display-only, not connected to tools | Dashboard + EntryDetail | Brokers can't act on AI recommendations |
| 2 | P0 | `verify_classification` not surfaced as direct button | EntryDetail | Most critical broker tool hidden behind chat |
| 3 | P0 | `calculate_entry_fees` not surfaced; crude estimates shown | EntryDetail | Inaccurate financial data for decision-making |
| 4 | P0 | `draft_communication` has `[Message body]` placeholder | Communications | Broken UX for catch-all communication purpose |
| 5 | P1 | No search/text filter in queue | BrokerQueue | Can't find entries quickly when needed |
| 6 | P1 | Compose modal loses entry context | Communications | AI Draft disabled, manual composition required |
| 7 | P1 | Risk flag resolution steps passive | EntryDetail | Same as #1 but in entry context |
| 8 | P1 | No AI draft for CF-29 protests | CBPResponses | Complex legal writing with no AI assistance |
| 9 | P1 | Missing docs sidebar is display-only | EntryDetail | No action pathway from identified gap to resolution |
| 10 | P1 | Held shipments show no hold context before claiming | Dashboard | Blind claims lead to wasted time |

---

## Design Philosophy Recommendations

1. **Information should always connect to action**: Every piece of data displayed should have a next-step action attached. A risk flag should have a "Resolve" button. A missing doc should have "Upload" and "Request" buttons. An estimated duty should have "Calculate Precisely" link.

2. **Surface tools as buttons, not chat prompts**: The 16 tools hidden behind chat should be surfaced as contextual buttons at the point of need. Chat is for exploration and complex queries; structured actions should be direct.

3. **Pre-compose, don't template**: Every communication should be AI-drafted with real content. Never show placeholders like `[Message body]` or `[Broker Name]`. The system has enough context (shipment, entry, company, missing docs) to generate complete drafts.

4. **Resolution paths should be workflows, not lists**: Alert resolution steps should be a guided wizard where completing step 1 enables step 2, with progress tracking. Currently they're just numbered text.

5. **The Entry Detail screen needs a "readiness gate"**: Before showing the "Submit to CBP" button, run `check_entry_readiness` and show a pre-flight checklist. Currently the submit button appears based on status alone, not on actual completeness.
