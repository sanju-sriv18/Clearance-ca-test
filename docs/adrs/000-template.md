# ADR-NNN: [Title]

<!--
  AUTHORING GUIDE (delete this block before committing)
  ──────────────────────────────────────────────────────
  Copy → rename to NNN-kebab-title.md → fill every section → delete this comment block.
  NNN = next available number. Never reuse or renumber.

  WHY THIS STRUCTURE:
  The XML blocks at the top are for Claude. Claude parses XML tags more reliably
  than markdown prose for structured data and performs better when context (data)
  precedes instructions. The <adr-meta> and <adr-quick-reference> blocks are what
  an agent reads first to determine relevance and extract constraints — without
  needing to parse the full record.

  The markdown sections below are for humans: Context, Decision, Alternatives,
  Consequences, Compliance, Related. Write these for the human reader.
  Claude will read them only when it needs the full reasoning.

  CONSTRAINT WRITING:
  Use MUST / MUST NOT / NEVER / ALWAYS. One rule per line. Max 7 constraints.
  If you need more than 7, this ADR is too broad — split it.
  Do NOT write constraints as prose paragraphs — they won't be reliably extracted.

  ALTERNATIVES CONSIDERED is REQUIRED. It prevents future agents from
  re-proposing solutions that were already evaluated and rejected.

  AMENDED-BY DISCIPLINE:
  When another ADR amends this one, you MUST update <amended-by> in this file
  as part of the PR that introduces the amending ADR. Both files must be updated
  atomically — lint-decisions will fail if <amended-by> references a non-existent ADR.
-->

<adr-meta>
  <status>Proposed</status>
  <!-- Proposed | Accepted | Deprecated | Superseded -->
  <date>YYYY-MM-DD</date>
  <last-validated>YYYY-MM-DD</last-validated>
  <!-- Update this whenever the decision is reviewed and confirmed still current.
       Should be updated on every PR that touches this ADR's domain. -->
  <deciders>Platform Architecture Team</deciders>
  <triggered-by>N/A</triggered-by>
  <!-- what forced this decision: sprint, finding, security issue, etc. -->
  <supersedes></supersedes>
  <!-- ADR-NNN this replaces, if any -->
  <amended-by></amended-by>
  <!-- ADR-NNN that later modified this record — MUST be updated when that ADR merges -->
</adr-meta>

<adr-quick-reference>
<!--
  Write this block for Claude. If an agent reads nothing else, it reads this.
  Place the most constraining information here. Long context first.
-->

  <summary>
    One sentence. What was decided and why it matters to the platform.
  </summary>

  <scope>
    <!-- Be specific. Claude uses this to decide whether this ADR applies to the current task. -->
    Applies to: ALL services | Go domain services only | Experience Layer only | [specific context]
    Excludes: [what this explicitly does NOT govern — be precise]
  </scope>

  <constraints>
    <!-- Hard rules. MUST / MUST NOT / NEVER / ALWAYS. Non-negotiable. -->
    <!-- Claude enforces these without reading the full record. Max 7. -->
    MUST: [...]
    MUST NOT: [...]
    NEVER: [...]
    ALWAYS: [...]
  </constraints>

</adr-quick-reference>

---

## Context

<!--
  What situation, problem, or inconsistency required a decision?
  What would happen if no decision were made?
  Assume the reader has zero prior context. Cite evidence where it exists.
-->

[Context narrative]

---

## Decision

<!--
  What was decided. Precise and complete.
  Do not justify here — justification belongs in Rationale / Consequences.
  If the decision has sub-parts, use ### subsections.
-->

[Decision narrative]

---

## Alternatives Considered

<!--
  REQUIRED. Every alternative that was seriously evaluated.
  "We didn't consider alternatives" is not acceptable.
  This section is the primary protection against future agents re-proposing
  rejected approaches. Be specific about WHY each was rejected.
-->

| Alternative | Why Rejected |
|---|---|
| [Alternative A] | [Specific failure mode, not just "worse than chosen"] |
| [Alternative B] | [Specific failure mode] |

---

## Consequences

<!--
  What follows from this decision. Be honest about trade-offs.
  Positive: what this enables.
  Negative: what this constrains or costs.
  Neutral: what changes but is neither better nor worse.
-->

### Positive
- [...]

### Negative
- [...]

### Neutral
- [...]

---

## Compliance

<!--
  How is compliance with this ADR verified?
  Automated: schema registry, CI gate, lint rule, test, pre-commit hook
  Manual: code review, architecture review gate
  Observable: metric, alert, dashboard
  If no automated enforcement exists, state why and what the manual gate is.

  AMENDED-BY REQUIREMENT: If this ADR is later amended by another ADR, the PR
  introducing the amending ADR MUST update <amended-by> in this file. Both files
  are updated atomically. Code review is the gate — reviewers must reject PRs
  that introduce an amending ADR without updating the original.
-->

[Compliance mechanism]

---

## Related

<!--
  Typed relationships only. Relationship type is as important as the reference.
  Options:
    Implements:     this ADR implements a product-level decision (PDR-NNN)
    Implemented by: another record implements this one
    Enabled by:     this ADR depends on another being in place first
    Constrains:     this ADR limits what another record can specify
    Amends:         this ADR modifies another record
    Supersedes:     this ADR replaces another record
    References:     design document section, external source
-->

- **Implements:** [PDR-NNN — title]
- **References:** [Nexus Fusion Platform Design, Section X.Y]
