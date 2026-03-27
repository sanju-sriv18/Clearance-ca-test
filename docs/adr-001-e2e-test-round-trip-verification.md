# ADR-001: E2E Tests Must Verify Round-Trip State Changes

**Date:** 2026-02-14
**Status:** Accepted
**Context:** Broker filing flow E2E tests (Sprint 3.5)

---

## Decision

Every E2E test that exercises a user action must verify the **actual state change** through the full round-trip, not just that a UI element appeared.

---

## What Happened

Across two consecutive fix cycles (`d527991`, `f74e3c5`), we discovered that our E2E tests were passing while the features they covered were broken:

1. **Request Documents draft listed wrong documents.** The draft communication endpoint was reading a stale `checklist_state` JSONB column instead of the computed checklist. The E2E test only checked "is there a draft preview visible?" — it never verified the draft body contained the *actually* missing documents. The test passed. The feature was broken.

2. **Bond type selector sent display labels instead of API values.** The frontend sent `"Single Transaction"` instead of `"single_transaction"`. The E2E test only checked "are the three buttons visible?" — it never clicked one and verified the bond status changed to `verified`. The test passed. The feature was broken.

3. **Reclassify button showed a toast but never called the API.** The `onClick` handler called `toast.info()` and nothing else. A visibility-only test would have passed — the button was there, it was clickable. But it did nothing.

In every case, a test that verified the **outcome** would have caught the bug immediately.

---

## The Pattern

A correct E2E test for a user action follows this structure:

```
1. Find data in the right state       (API query, not hardcoded IDs)
2. Perform the action                  (click, submit)
3. Wait for the system response        (toast, API response, spinner clear)
4. Verify old UI state is gone         (selector disappears, panel clears)
5. Verify new UI state appeared        (green checkmark, updated text)
6. Re-fetch via API and assert state   (DB actually changed)
```

### Concrete example: Bond type selector

```typescript
// BAD — visibility only (what we had before)
await expect(page.locator("button", { hasText: "Continuous" })).toBeVisible();
// Test passes. Feature is broken. Nobody knows.

// GOOD — full round-trip (what we have now)
await continuousBtn.click();
await expect(page.locator("text=/Bond type set/i")).toBeVisible({ timeout: 10000 });
await expect(page.locator("text=/Select bond type/i")).toBeHidden({ timeout: 15000 });
await expect(bondRow.locator(".text-green-500")).toBeVisible({ timeout: 5000 });
```

### Concrete example: Accept classification

```typescript
// BAD — shallow API ping
const resp = await page.request.post(`${API}/api/broker/entries/${id}/accept-classification`, {
  data: { hs_code: "8471.50.00" },
});
expect(resp.ok()).toBe(true);
// Test passes. But did the DB actually update? Did the checklist refresh?

// GOOD — verify the state persisted
const resp = await page.request.post(/* ... */);
expect(resp.ok()).toBe(true);
const afterDetail = await page.request.get(`${API}/api/broker/entries/${id}`).then(r => r.json());
expect(afterDetail.hs_code).toBe("8471.50.00");
const confCheck = afterDetail.checklist.items
  .find(i => i.key === "hts_classification")
  .validation_checks.find(vc => vc.check === "classification_confidence");
expect(confCheck.detail).toBe("HIGH");
```

### Concrete example: Draft communication content

```typescript
// BAD — "is there a draft?"
const draftBody = page.locator("pre").first();
expect(await draftBody.isVisible()).toBe(true);

// GOOD — "does the draft list the right documents?"
const missing = checklist.items.filter(i => docKeys.has(i.key) && i.status === "missing");
const draft = await draftResp.json();
for (const label of missing.map(i => i.label)) {
  expect(draft.body).toContain(label);
}
const verified = checklist.items.filter(i => docKeys.has(i.key) && i.status === "verified");
for (const label of verified.map(i => i.label)) {
  expect(draft.body).not.toContain(label);
}
```

---

## Rules Derived

1. **No visibility-only tests for actions.** If a test clicks a button, it must verify what the button *did*, not just that the button *existed*.

2. **Re-fetch after mutation.** After any API call that changes state, GET the resource again and assert the new values. The API response saying `"status": "updated"` is necessary but not sufficient.

3. **Verify both positive and negative.** The draft communication test checks that missing docs *are* listed AND that verified docs are *not* listed. The bond test checks the selector *disappears* AND the green checkmark *appears*.

4. **Match UI to API data.** When the UI displays counts or labels, fetch the same data via API and verify the numbers match. Don't just check "is there a progress message" — check the message contains the correct count.

5. **Find test data by state, not by position.** The simulation constantly shuffles entries. Never use `queue[0]` and assume it's in the right state. Query the API, filter for the state you need, skip if not found.

---

## Consequences

- Tests take slightly longer to write (re-fetch + assertion adds ~5 lines per test)
- Tests catch real bugs that visibility-only tests miss
- Tests are resilient to UI refactors (asserting on data, not CSS classes, where possible)
- When a test fails, the failure message tells you *what's wrong* ("expected hs_code to be 8471.50.00, got 8471.30.01") instead of "element not visible"
