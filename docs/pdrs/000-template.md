# PDR-NNN: [Title]

<!--
  AUTHORING GUIDE (delete this block before committing)
  ──────────────────────────────────────────────────────
  Copy → rename to pdr-NNN-kebab-title.md → fill every section → delete this comment block.
  NNN = next available number. Never reuse or renumber.

  PDR vs ADR:
    PDR = product decision — what the platform does, how it behaves, why.
    ADR = architecture decision — how it is implemented, naming, data patterns.
    If it governs agent behavior, platform philosophy, or UX: PDR.
    If it governs implementation mechanics, protocols, schemas: ADR.

  WHY THIS STRUCTURE:
  The XML blocks at the top are for Claude. Claude parses XML tags more reliably
  than markdown prose for structured data and performs better when context (data)
  precedes instructions. The <pdr-meta> and <pdr-quick-reference> blocks are what
  an agent reads first — without parsing the full record — to determine relevance
  and extract invariants.

  The markdown sections below are for humans. Write them for the human reader.
  Claude reads them when it needs the full reasoning behind a decision.

  INVARIANT WRITING:
  Invariants are absolute constraints that hold regardless of context, phase,
  trade-off pressure, or user request. If a constraint has exceptions, it is
  not an invariant — move it to the Consequences section instead.
  Max 5 invariants. If you need more, this PDR is too broad — split it.

  IMPLEMENTATION STATUS is critical for Claude agents.
  Without it, agents cannot tell whether to implement this decision or work
  within it. "Aspirational" means don't implement yet — it's designed but
  not scoped. "Pending" means it's scoped but not built. "Implemented" means
  it exists in the codebase right now.
-->

<pdr-meta>
  <status>Proposed</status>
  <!-- Proposed | Accepted | Deprecated | Superseded -->
  <date>YYYY-MM-DD</date>
  <last-validated>YYYY-MM-DD</last-validated>
  <!-- Update this whenever the decision is reviewed and confirmed still current. -->
  <category>Platform Philosophy</category>
  <!-- Platform Philosophy | Domain Architecture | AI/LLM Strategy |
       Governance | Operating Model | Data Strategy | Technology Strategy -->
  <amends></amends>
  <!-- PDR-NNN this modifies, if any -->
  <amended-by></amended-by>
  <!-- PDR-NNN that later modified this record — MUST be updated when that PDR merges -->
  <supersedes></supersedes>
</pdr-meta>

<pdr-quick-reference>
<!--
  Write this block for Claude. If an agent reads nothing else, it reads this.
  Data (summary, scope, implementation-status) before invariants — long context first.
-->

  <summary>
    One sentence. The decision and its primary implication for the platform.
  </summary>

  <scope>
    <!-- What this governs. Claude uses this to decide if this PDR applies to the current task. -->
    Applies to: all domains | specific domains | specific layers | [specific context]
    Excludes: [what this explicitly does NOT govern]
  </scope>

  <implementation-status>
    <!--
      Current state — critical for Claude to distinguish "implement this" from "work within this".
      Implemented:           Exists in the codebase. Location: [file/service/path]
      Partially Implemented: [what exists] / [what is still pending]
      Pending:               Designed and scoped, not yet built. Sprint: [N]
      Aspirational:          Intended direction, not yet designed or scoped.
    -->
    [status and location or sprint]
  </implementation-status>

  <invariants>
    <!--
      Absolute constraints. Hold regardless of context, phase, or trade-off pressure.
      Claude enforces these without reading the full record.
      Max 5. If a constraint has exceptions, it belongs in Consequences, not here.
    -->
    INVARIANT: [...]
    INVARIANT: [...]
  </invariants>

</pdr-quick-reference>

---

## Context

<!--
  What situation, tension, or need led to this decision?
  What would happen without a clear decision here?
  Include concrete evidence: data, sprint observations, failure modes, measurements.
  The stronger the evidence, the more durable the decision.
-->

[Context narrative]

---

## Decision

<!--
  What was decided. State it directly and completely.
  Include explicit scope boundaries: what is in, what is out.
  If the decision has phases or conditions, state them explicitly.
-->

[Decision narrative]

---

## Rationale

<!--
  Why this choice. Be specific: "we chose X over Y because Y fails under
  condition Z, as evidenced by [observation/data]."
  Alternatives MUST appear in the table below — not just in prose.
  This table is the primary protection against future re-litigation.
-->

[Rationale narrative — why this over alternatives]

| Alternative | Reason Rejected |
|---|---|
| [Alternative A] | [Specific failure mode or trade-off, not just "worse"] |
| [Alternative B] | [Specific failure mode] |

---

## Consequences

<!--
  What follows from this decision.
  Positive: what this enables or simplifies.
  Negative: what this rules out, constrains, or costs.
-->

### Positive
- [...]

### Negative
- [...]

---

## Compliance

<!--
  How is compliance with this PDR verified?
  Even product decisions need a gate.
  Examples:
    Automated: a lint rule, a CI check, a schema constraint
    Manual: sprint review checkpoint, architecture review, code review gate
    Observable: a metric or dashboard that makes violation visible

  AMENDED-BY REQUIREMENT: If this PDR is later amended by another PDR, the PR
  introducing the amending PDR MUST update <amended-by> in this file atomically.
  Code review is the enforcement gate.
-->

[Compliance mechanism]

---

## Related

<!--
  Typed relationships only. Relationship type matters as much as the reference.
  Options:
    Implemented by: ADR-NNN implements the mechanism for this product decision
    Enabled by:     this PDR requires another decision to be in place first
    Constrains:     this PDR limits what another record can specify
    Amends:         this PDR modifies another record (update <amends> field too)
    Supersedes:     this PDR replaces another record
    References:     design document section, external source
-->

- **Implemented by:** [ADR-NNN — title]
- **References:** [Nexus Fusion Platform Design, Section X.Y]
