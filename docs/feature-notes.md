# Feature Notes & Findings

> Observations, gaps, and improvement opportunities discovered during development.

---

## FN-001: Broker Lacks Description Quality Improvement for Low-Confidence HS Classification

**Date:** 2026-02-13
**Status:** Open — Not Yet Implemented
**Priority:** High
**Surfaces Affected:** Broker (EntryDetail)

### Finding

When the AI classification engine assigns an HS code with LOW confidence, the broker currently has two options:

1. **Verify Classification** — Re-runs the E1 classifier against the *same* product description. If the underlying description is poor or ambiguous, this produces the same low-confidence result.
2. **Reclassify to [suggested code]** — UI stub only. Displays a toast notification but does NOT persist the reclassified HS code to the database.

Meanwhile, the **Shipper** and **Platform** surfaces already have a working `DescriptionQualityPanel` component that:
- Shows a quality score with missing attributes highlighted
- Offers "Help me improve it" with AI-generated follow-up questions (e.g., "What is the primary material?", "What is the intended use?")
- Collects answers and generates an enriched product description
- Re-runs classification against the improved description, yielding higher confidence

### Gap

The broker surface (`EntryDetail.tsx`) does not integrate `DescriptionQualityPanel`. Brokers see LOW confidence but have no tool to iteratively improve the product description that drives classification.

### Existing Components to Reuse

| Component | Location | Status |
|-----------|----------|--------|
| `DescriptionQualityPanel` | `frontend/src/components/DescriptionQualityPanel.tsx` | Working — used in Shipper/Platform |
| `analyzeDescriptionQuality()` | `frontend/src/api/client.ts` | Working API client function |
| `POST /api/description-quality` | Backend route | Working endpoint |
| Follow-up questions UI | `frontend/src/surfaces/shipper/screens/AddProduct.tsx:148-180` | Pattern to replicate |

### Recommended Implementation

1. Add `DescriptionQualityPanel` to `EntryDetail.tsx` — show it in the Classification section when confidence is not HIGH
2. Wire the "Help me improve it" flow to collect broker answers to follow-up questions
3. On improved description, re-run E1 classifier and update the entry's HS classification
4. Fix the "Reclassify to X" button to actually persist the new HS code via an API call

### Impact

Without this, brokers must manually research HS codes for low-confidence classifications or accept potentially incorrect codes — defeating the purpose of the AI classification engine.

---

## FN-002: Reclassify Button is a UI Stub

**Date:** 2026-02-13
**Status:** Open — Not Yet Implemented
**Priority:** Medium
**Surfaces Affected:** Broker (EntryDetail)
**Related:** FN-001

### Finding

The "Reclassify to [HS code]" button in the broker's Classification Verification panel (`EntryDetail.tsx:1636`) only shows a toast notification. It does not call any API endpoint to persist the reclassified HS code.

### Location

```
EntryDetail.tsx:1636 — onClick handler shows toast only
```

### Recommended Fix

Wire the button to call an API endpoint (e.g., `POST /api/broker/entries/{id}/reclassify`) that updates the entry's HS code, recalculates duties, and emits a `ClassificationUpdated` domain event.
