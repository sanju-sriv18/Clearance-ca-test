# Presenter Demo Notes: Areas of Inherent Complexity

## Voiceover Guidance for When Precision Can't Be Guaranteed

---

> **Purpose:** Some aspects of customs and trade compliance are inherently complex to the point where even a perfectly built system operating within our defined scope cannot guarantee accuracy. These areas require the presenter to set context, acknowledge limitations, and frame the system's output appropriately. This document provides specific voiceover language for each such area.

> **When to use these notes:** The presenter should internalize these talking points, not read them. They're for situations where an audience member asks a pointed question or when the system displays output in one of these inherently uncertain areas. The goal is to turn limitations into credibility by showing the team understands the complexity.

---

## 1. State and Local Tax Computation

**The inherent complexity:** There are 11,000+ tax jurisdictions in the United States. State sales/use tax rates, taxable base rules, and exemptions vary dramatically. Some states tax imports differently from domestic sales. Some include shipping in the taxable base; others don't. Some include duty; most don't. There is no single federal rule.

**What the system does:** Uses a representative combined state + local rate per state (from K13) applied to customs value as the default taxable base. This is a reasonable simplification.

**What can be wrong:**
- The local rate component depends on the specific city/county, which the system doesn't determine
- Whether shipping and/or duty is included in the taxable base varies by state
- Some products are exempt from sales tax in some states (e.g., clothing in PA, food in many states)
- The system shows a "state/use tax" line item — the actual obligation depends on whether the buyer owes use tax (they usually do for imports, but enforcement varies)

**Voiceover if questioned:**

> "The state tax line is computed from a representative rate for the destination state. In reality, there are over 11,000 tax jurisdictions in the US, and the exact rate depends on the specific city and county. The taxable base also varies — some states include shipping, most exclude duty. For the demo, we use the customs value as the base, which is the most common approach. In production, the system would integrate a real-time tax API — like Avalara or Vertex — to get the precise rate for any delivery address."

**Key point:** Don't apologize for the simplification. Frame it as a known integration point that's well understood and commercially solvable.

---

## 2. Entry Type for Post-De Minimis Low-Value Parcels

**The inherent complexity:** With de minimis (Section 321) suspended as of August 29, 2025, approximately 4 million daily parcels that previously entered the US with minimal data now require full entry. But "full entry" doesn't necessarily mean "formal entry." CBP's guidance on exactly how these parcels are processed is still evolving.

**What the system does:** Applies a $2,500 threshold: above → formal entry, at or below → informal entry. This is the textbook rule that has existed for decades.

**What may be uncertain:**
- CBP has been developing new entry types and data requirements specifically for the post-de minimis world
- Type 86 entries (which were designed for Section 321 + advance data) are effectively deprecated, but some brokers may still use variations
- The exact data requirements for informal entry of parcels that previously entered under de minimis are still being clarified in CBP guidance
- Express carriers like FedEx have operational processes that may not map cleanly to the textbook entry types

**Voiceover if questioned:**

> "The system determines entry type using the $2,500 threshold — informal below, formal above. That's the standing rule. What's evolving is how CBP operationalizes this for the millions of low-value parcels that used to enter under Section 321. The entry type logic is correct in principle; the implementation details are something CBP is still refining. For a FedEx solution, this is actually a competitive advantage — FedEx has the data infrastructure to meet whatever requirements CBP settles on."

**Key point:** This is an opportunity, not a weakness. Frame it as "the system is ready for whatever CBP requires."

---

## 3. Customs Valuation Beyond Invoice Price

**The inherent complexity:** Transaction value (the price actually paid or payable) sounds simple but can involve significant adjustments under 19 USC 1401a:
- Assists (tools, dies, molds, engineering provided free or at reduced cost by the buyer to the seller)
- Royalties and license fees payable as a condition of sale
- Proceeds of subsequent resale accruing to the seller
- Buying commissions vs. selling commissions (only selling commissions are added)
- Related-party transactions where transfer pricing may not reflect arm's-length value

For Maria's $38 mug (DTC consumer sale), none of this applies — the invoice price IS the transaction value. For David's commercial auto parts imports, it could matter significantly.

**What the system does:** Accepts a declared value and uses it as-is. No valuation adjustments.

**Voiceover if questioned:**

> "The system uses the declared value as transaction value — which is correct for the vast majority of consumer and simple commercial transactions. For complex supply chains with assists, royalties, or related-party transactions, customs valuation can require adjustments under 19 USC 1401a. That's a production feature — the system would capture valuation elements and flag when adjustments might apply. For this demo, we're showing the analytical engines on straightforward transactions where the invoice price is the transaction value."

**Key point:** Acknowledge the complexity exists. Don't claim the demo handles it.

---

## 4. AD/CVD Rates Are Manufacturer-Specific

**The inherent complexity:** Anti-dumping and countervailing duty rates are not uniform. Each AD/CVD order specifies:
- Individually examined rates (for manufacturers who cooperated with the ITC investigation)
- An "all-others" rate (for cooperating manufacturers not individually examined)
- A "PRC-wide" or "country-wide" rate (for non-cooperating manufacturers — often punitively high)

The rate a specific shipment faces depends on which manufacturer produced the goods. Without knowing the manufacturer, the system can only show the range.

**What the system does:** When an active AD/CVD order is found for the HS code and origin country, the system shows: "AD/CVD order active. All-others rate: X%. Range: Y% - Z% depending on manufacturer."

**Voiceover if questioned:**

> "AD/CVD rates are manufacturer-specific — the rate depends on who produced the goods, not just what they are or where they're from. The system identifies that an active order exists and shows the range. To get the exact rate, you need the manufacturer's name mapped to the ITC's rate schedule. In production, the system would maintain that mapping and ask for the manufacturer. For the demo, we show the range and the all-others rate as a default."

**Key point:** Showing the range is the honest and correct approach. A system that shows a single rate without knowing the manufacturer is either guessing or defaulting — and the audience should see that we understand this.

---

## 5. IEEPA Tariff Rate Volatility

**The inherent complexity:** IEEPA reciprocal tariff rates are set by executive order and can change with very little notice. The rates have been modified multiple times since their introduction. A rate that is correct today may be wrong tomorrow.

**What the system does:** Looks up the rate from K5 and shows an "as of" date.

**Voiceover if questioned:**

> "The IEEPA rates are sourced from the executive orders and CBP implementation guidance as of [date]. These rates have changed multiple times — for China alone, the rate has been modified [N] times in the past year. That's why the system shows an 'as of' date on every IEEPA rate. In production, the system would monitor Federal Register notices and flag rate changes that affect your portfolio. The volatility is actually one of the strongest arguments for this platform — manually tracking IEEPA changes across thousands of SKUs is one of the most painful compliance burdens today."

**Key point:** Rate volatility is a feature of the sales pitch, not a weakness of the system. It's literally the problem the platform solves.

---

## 6. Tariff-Rate Quotas (TRQ)

**The inherent complexity:** Some products (dairy, sugar, certain agricultural goods, some steel) have tariff-rate quotas: a lower duty rate up to a quota volume, and a higher rate above the quota. Whether the in-quota rate applies depends on real-time quota fill status, which CBP tracks but does not publish in an easily consumable format.

**What the system does:** Notes when a TRQ exists and shows both rates: "Within-quota rate: X%. Over-quota rate: Y%. TRQ status: check with CBP for current quota fill." The system does not attempt to determine current quota fill.

**Voiceover if questioned:**

> "The system identifies that this product has a tariff-rate quota and shows both rates. Whether the lower rate applies depends on the current quota fill, which CBP tracks in real time but doesn't publish as an API. In production, there are commercial services that track quota fill status — we'd integrate one. For the demo, we show you both rates and flag the TRQ so you know the question exists."

---

## 7. Classification Confidence and Handling Errors Live

**The inherent complexity:** Product classification is fundamentally interpretive. Reasonable trade professionals can disagree on the correct HS code for a product. The system is reasoning against the same legal text and classification rules that human classifiers use — and like human classifiers, it can be wrong.

**What the system does:** Shows a confidence level (High/Medium/Low) with clear criteria. Shows alternatives when confidence is Medium or Low. Explains its reasoning.

**If the system gets a classification wrong during the demo:**

> "Good catch. The system classified this as [X], but you're telling me it's [Y]. That's exactly the kind of feedback that makes the system better over time. Notice that the system gave this [Medium/Low] confidence — it flagged uncertainty. The distinction between [X] and [Y] comes down to [specific technical point], and the description didn't give enough signal to resolve it. In production, this is where the human-in-the-loop workflow kicks in: the system flags uncertainty, a classification specialist resolves it, and that resolution feeds back into the system's knowledge base."

**If the system gets a classification right and the audience is surprised:**

> "The system got that right — not because it memorized a table of products and codes, but because it reasoned through the HTSUS hierarchy using the same rules a human classifier follows. You can see the reasoning chain: it identified the material, determined the chapter, applied the heading definitions, and landed on [code]. That reasoning is the value — it's auditable, it's citable, and it gets better with every product it sees."

**Key point:** Classification errors are expected, not catastrophic. The system's value is in making its reasoning transparent, not in being infallible.

---

## 8. FTA Qualification with Incomplete BOM Data

**The inherent complexity:** Rules of origin analysis requires detailed Bill of Materials data: every component, its HS code, its origin, and its value. In practice, importers often don't have complete BOM data readily available — especially for complex manufactured goods with many components.

**What the system does:** Accepts a simplified BOM and runs the analysis against it. When data is incomplete, it says so: "Based on available data, RVC appears to meet the threshold. A definitive determination requires [missing data elements]."

**Voiceover if questioned:**

> "The FTA engine runs real USMCA rules of origin analysis — tariff shift tests, regional value content calculations, the whole analytical framework. But the quality of the answer depends on the quality of the BOM data. In this demo scenario, we've provided a simplified BOM to show the analytical capability. In production, integrating with the importer's ERP or procurement system to pull real BOM data is how this becomes a decision-support tool. The analytical logic is ready; the data integration is the production engineering."

---

## 9. The "This Is Just ChatGPT" Objection

**The inherent complexity:** This is not a technical limitation — it's a perception problem. The classification engine uses an LLM. A skeptic will say "I can just ask ChatGPT to do this."

**Voiceover when this comes up (and it will):**

> "Fair question. Let me show you the difference. [If possible, have a second screen or tab ready with ChatGPT.]
>
> Ask ChatGPT to classify 'wireless Bluetooth speaker from China' — it'll give you an HS code. Maybe the right one, maybe not. Now look at what our system does:
>
> First, it classifies against the *current* HTSUS — not training data from two years ago. ChatGPT's tariff knowledge has a cutoff date. Ours is loaded from the live USITC schedule.
>
> Second, every HS code is validated against the schedule before it's displayed. ChatGPT can hallucinate an HS code that doesn't exist. Our system cannot.
>
> Third, the classification flows directly into the tariff stack, compliance screening, and landed cost computation. ChatGPT gives you a code. We give you a code, a reasoning chain, a duty calculation, a compliance assessment, and a total cost — all sourced, all auditable.
>
> Fourth, ChatGPT doesn't know about Section 301 lists, IEEPA country schedules, AD/CVD orders, or PGA jurisdiction. It might mention them in general terms. Our system checks them specifically for your product and your origin country.
>
> ChatGPT is a general-purpose AI. This is a purpose-built clearance intelligence system. That's the difference."

---

## 10. Supply Chain Traceability Limitations

**The inherent complexity:** The UFLPA risk assessment shows a tier-2 supply chain visualization. But true multi-tier supply chain traceability is, as documented in our knowledge base, one of the hardest problems in global trade. Clearance data provides tier-1 visibility (who shipped what to whom) but not tier-2+ (where did the inputs come from).

**What the system does:** The supply chain tree in the dashboard is illustrative (pre-authored data). The entity screening against the UFLPA list is live.

**Voiceover if questioned:**

> "Let me be specific about what's live and what's illustrative here. The entity screening — checking names against the UFLPA Entity List — that's live. I can type any company name and the system screens it in real time. The supply chain tree you see — the visualization of who supplies whom — that's illustrative data. In production, this would integrate with a supply chain intelligence provider. But here's the honest truth: multi-tier supply chain traceability is one of the hardest problems in global trade. Even with a provider, visibility past tier 2 is limited by the fungibility of raw materials and the opacity of sub-tier suppliers. The system is honest about that boundary."

---

## 11. Global Scope and Tier Depth Differences

**The inherent complexity:** The system operates at three tiers of depth — full depth for US, meaningful depth for EU and China, and framework level for Brazil/India. An audience member may ask about a jurisdiction where the system's coverage is less complete, or may assume the EU and China coverage is as deep as the US.

**What the system does:** Computes landed costs using jurisdiction-specific tax regime templates. The HS 6-digit classification is globally valid. National subheading extensions, compliance frameworks, and surcharge programs vary by tier.

**Voiceover if questioned about depth:**

> "The system operates at different depths for different jurisdictions. For the US, we have full depth — every surcharge program, every PGA mapping, every compliance framework. For the EU and China, we have meaningful analytical capability — MFN rates, trade remedies, VAT/import taxes, major compliance frameworks. For other markets like Brazil, we demonstrate the framework — you can see the architecture scales, but the knowledge depth is not yet at the same level as US or EU. The key architectural point is that adding depth to any jurisdiction is a knowledge-loading exercise, not an engine-building exercise. The analytical engines are the same."

**If someone asks about a jurisdiction not loaded (e.g., "What about imports into Indonesia?"):**

> "Indonesia isn't in the active knowledge base yet. But let me show you something — the product classification at HS 6 digits is the same regardless of destination. Indonesia is a WCO member, so the 6-digit code is valid there. What we'd need to load is Indonesia's national tariff schedule, any special programs, and the tax structure. That's configuration, not development. The same pattern we used to add the EU and China applies to any WCO member country."

**Key point:** The tiered approach is a strength, not a weakness. It shows deliberate prioritization and a scalable architecture.

---

## 12. Regulatory Change Signal Accuracy

**The inherent complexity:** Engine 6 shows regulatory change signals at three confidence levels: confirmed, proposed, and discussed. The "discussed" signals are inherently uncertain — they may never materialize. Even "proposed" signals may be modified significantly before enactment. The audience may question the reliability of pre-decisional signals.

**What the system does:** Clearly labels every signal with its status (CONFIRMED / PROPOSED / DISCUSSED), source, and attribution. Scenario modeling is always presented as hypothetical — "if enacted at preliminary rates." The system never presents a fuzzy signal as a current obligation.

**Voiceover if questioned about signal reliability:**

> "The system distinguishes three levels of certainty. CONFIRMED means it's enacted — we have the Federal Register citation or the Official Journal reference. PROPOSED means there's a formal proceeding — an investigation, a proposed rule, a draft regulation. DISCUSSED means someone in authority mentioned it but nothing formal has started. The value of the system isn't predicting which signals will materialize — it's letting you model the impact if they do. If you're a supply chain director and the US is talking about increasing tariffs on a product category that represents 30% of your imports, you need to know the dollar impact of that scenario even if it's uncertain. That's what the scenario modeling does."

**If someone points out a signal that didn't materialize:**

> "Exactly right — that signal didn't materialize. The system tracked it, modeled the impact, and your team was prepared. When it didn't happen, no harm done. When the next one does happen — and in this regulatory environment, something always does — you'll be prepared for that one too. The cost of modeling a scenario that doesn't materialize is near zero. The cost of being blindsided by one that does is the real risk."

---

## 13. Tax Regime Template Differences Across Jurisdictions

**The inherent complexity:** Different countries compute import taxes in structurally different ways. The US stacks duties additively. The EU computes VAT on (value + duty). Brazil's ICMS includes itself in its own tax base, requiring algebraic grossup. An audience member from one jurisdiction may not understand why the same product has a different tax computation structure in another jurisdiction.

**Voiceover if questioned about why computations look different:**

> "The tax structures are genuinely different. In the US, duty programs stack additively — MFN plus Section 301 plus IEEPA. In the EU, VAT computes on the customs value plus duty, so the VAT amount changes if the duty changes. In Brazil, it's more complex — the state tax, ICMS, includes itself in its own base, which requires a mathematical grossup. The system handles each of these through what we call tax regime templates — configurable computation patterns that define the order, the base formula, and the stacking behavior for each tax layer. Adding a new country means defining its template and loading its rates. The analytical engine is the same."

**Key point:** The tax regime template architecture is one of the strongest technical points in the demo. It demonstrates that the team understands global trade is not "US customs applied to other countries" — each jurisdiction has its own structure, and the platform is designed for that reality.

---

## 14. The Surface Switching Moments

**The inherent complexity:** The demo's most powerful recurring motif is the act of switching between the three application surfaces — Platform Intelligence (the command center), Shipper (workflow-embedded), and Buyer (checkout-embedded). These switches are the thesis of the platform made visible: the system absorbs complexity so that the people who ship and buy never have to. But the switching moments can fall flat if they're treated as a quick UI toggle rather than a deliberate reveal. They can also raise questions about whether the simpler surfaces are "real" or just "dumbed-down versions of the main screen."

**What the presenter does:** Each switch is a paced, deliberate moment with a pause before and after. The presenter narrates what's happening: "You've seen what the compliance team sees. Now let me show you what Maria sees." The audience needs a beat to absorb the contrast between the analytical complexity they just watched and the simplicity of the Shipper or Buyer surface.

**The four switching moments:**

**Scene 1 — Platform Intelligence → Shipper:** After the full product analysis (classification chain, tariff stack, compliance screening), the presenter says: "That's what the compliance team sees. But Maria — the ceramics artist in Lisbon — she doesn't see any of this." Switch. "Ready to ship. $56.81. Commercial invoice required. That's it." Pause. "All of that became this." The audience should feel the compression viscerally.

**Scene 3 — Platform Intelligence → Buyer:** After the three-column trade lane comparison showing tariff stacks for US, EU, and China, the presenter says: "That's three tax regimes. Now here's what the buyer sees." Switch to checkout mockup. Three prices. Three currencies. "Duties included." The tariff analysis vanishes; three numbers remain.

**Scene 4 — Platform Intelligence → Shipper:** After the regulatory change intelligence scene — signal board, scenario modeling, cross-jurisdiction impact — the presenter says: "That's the strategic layer. What about Maria?" Switch to a notification: "Shipping costs to the US will increase by $4.20 on March 1. No action needed." The seven-signal intelligence dashboard became one notification.

**Scene 6 — The Triple Reveal:** After the audience's product is analyzed, the presenter shows all three surfaces in sequence: Platform Intelligence ("your compliance team"), Shipper ("your logistics coordinator"), Buyer ("your customer"). This is the final image. Three surfaces, one system, one product. "That's what we're building."

**Voiceover for the "is this just hiding columns?" question:**

> "These aren't different views of the same screen — they're different applications. The Shipper surface doesn't have a 'show me the details' button that reveals the Platform Intelligence view underneath. It's designed from scratch for a different user with different needs. A logistics coordinator in a warehouse doesn't need GRI citations and confidence scores. They need: can I ship this, what does it cost, what documents do I need. The platform answers those three questions by running the full analytical stack behind the scenes. The simplicity isn't cosmetic — it's earned. In production, this panel lives inside their shipping platform. They never see our system at all."

**Voiceover for "who decides what the shipper sees?":**

> "The platform's translation layer. Every engine returns full analytical depth — classification reasoning, tariff program details, compliance screening results. The Platform Intelligence surface shows all of that. The Shipper surface runs the output through a plain-language renderer: 'Section 301 surcharge applicable' becomes 'Additional import duty applies for products from China.' The Buyer surface reduces further: just the total price and a 'duties included' badge. The decisions about what to show on each surface are product design decisions, not engine decisions. The engines don't know or care which surface will display their output."

**Key point:** The switching moments are the emotional heart of the demo. They don't prove the engines work — Scenes 1-5 do that. They prove *why the engines matter* — because they enable an experience where complexity becomes invisible to the people who ship and buy. If the presenter rushes these moments, the demo loses its thesis. Practice the pause.

---

## 15. Shipper and Buyer Surface Authenticity

**The inherent complexity:** The Shipper and Buyer surfaces are realistic stubs in the demo — the Shipper panel is standalone (not actually embedded in Shopify), and the Buyer checkout is a mockup (not actually embedded in an e-commerce site). An audience member may ask whether this is a real integration or a concept.

**Voiceover if questioned about Shipper surface reality:**

> "What you're seeing is a standalone panel that represents the integration. In production, this panel lives inside the shipper's existing platform — their Shopify dashboard, their ERP, their TMS. They never come to us; we go to them. The analytical capability behind it is real — the same engines that just showed you the twelve-line tariff stack computed the 'ready to ship, $56.81' result. What changes in production is where the panel appears, not what it computes."

**Voiceover if questioned about Buyer surface reality:**

> "The checkout price you see — $56.81 delivered, duties included — is computed live by the same tariff engine. In production, this price surfaces through an API that the e-commerce platform calls at checkout. The buyer never sees our brand, our UI, or our system. They see a price, a 'duties included' badge, and a delivery date. We provide the intelligence; the e-commerce platform provides the experience."

**Key point:** Be direct about what's real (the computation) and what's illustrative (the embedding context). This honesty follows the same pattern as the dashboard disclaimer — "the analytical results are live; the context is authored."

---

## General Framing Principles

When any question about accuracy or completeness comes up, the presenter should follow this framework:

1. **Acknowledge the complexity.** Don't dismiss the question. "That's a great question — [topic] is genuinely complex because..."
2. **Explain what the system does.** Be specific about the engine, the data source, and the logic.
3. **Be honest about the boundary.** "For this demo scope, we [approach]. In production, this would [full solution]."
4. **Turn the limitation into a value proposition.** "The fact that this is hard is exactly why the platform exists. Today, [how it's handled manually]. The system [how it improves that]."

**Never:**
- Claim the system does something it doesn't
- Hand-wave away a valid concern
- Make the audience feel their question was stupid
- Promise a capability that isn't specified
- Present a fuzzy regulatory signal as a confirmed obligation
- Rush the surface switching moments — they are the thesis of the demo
- Call the Shipper or Buyer surfaces "simplified views" — they are different applications

**Always:**
- Show you understand the real-world complexity
- Be transparent about scope and boundaries (including tier differences)
- Frame every limitation as a known engineering problem with a path to solution
- Let the working parts of the system speak for themselves
- When showing global capability, make the architecture visible — explain *why* adding jurisdictions is configuration, not development
- When switching surfaces, name the surface and the actor: "This is what Maria sees." "This is what appears at checkout." "Back to the command center."
- Let the contrast between surfaces speak — the compression from analytical depth to "ready to ship, $56.81" is more powerful than any explanation

---

*These notes should be internalized, not read during the demo. The presenter should rehearse responses to all 15 areas above and be prepared for variations of each question. The surface switching moments (Section 14) deserve dedicated rehearsal time — they carry the demo's emotional thesis. See also the [Authenticity Audit](authenticity_audit.md) for specific data points that have been verified.*
