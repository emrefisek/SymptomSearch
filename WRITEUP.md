# Symptom-to-Product Discovery — Case Study Write-up

**Prototype:** `symptom-discovery-prototype.html` in this repo — [add the GitHub Pages URL here once deployed]
**Design decisions & diagrams:** see `DESIGN-DECISIONS.md` in this repo.

---

Today, searching "I can't sleep" on Redcare returns a standard keyword results page — mostly melatonin, ranked by popularity. There's no experience for a customer who arrives with a problem rather than a product in mind. Rather than building a single "symptom search," I built **intent routing**: a deterministic cascade that recognizes different query types need fundamentally different treatment, and only applies guided discovery where it actually helps.

| Layer | Fires on | Outcome |
|---|---|---|
| 1 — Safety | Red-flag phrase (e.g. "chest pain") | Hard-stop escalation — never a product, overrides everything below |
| 2 — Catalogue | Known brand/ingredient (e.g. "ibuprofen") | Direct results, no guidance |
| 3 — Short symptom | Well-known single-cause term (e.g. "sunburn") | Light category route, escape hatch to guided flow |
| 4 — Long-form | Natural-language problem (e.g. "I can't sleep") | Guided flow, 1–2 clarifying questions |
| 5 — Ambiguous | Short, under-specified (e.g. "pain") | Guided flow, broader initial question |

**The decision I'm most proud of** is this layered cascade, paired with a visible **routing-trace panel**. The cascade runs five layers, cheapest and safest first — a hard-coded safety check that overrides everything, a catalogue/direct match for known products, a light-touch route for well-known single-cause symptoms, and only then a guided flow (1–2 questions) for genuinely ambiguous, natural-language queries. What makes it more than an architecture diagram is that every result shows *which layer handled the query and why*, in plain language, live on screen. Two real failures I found on the live Shop Apotheke site motivated this: "Herzinfarkt" (heart attack) resolves straight to products with zero escalation, and "Herzschmerzen" (heart pain — could be cardiac, muscular, anxiety, or reflux) resolves *confidently* to aspirin instead of asking one clarifying question first. Confident wrong answers are worse than a moment of friction, and the cascade is built specifically to make that trade-off correctly.

**The biggest trade-off I made** was breadth of taxonomy over depth of guided questioning. With limited time, I chose to cover more symptom categories shallowly (52 phrases across red-flag, catalogue, short-symptom, long-form, and ambiguous queries) rather than build a deeply branching question tree for a handful of them. That's the right call for demonstrating the *architecture* — the cascade, the safety boundary, the Rx-routing path enabled by scoping to Germany — but it means any single guided flow is fairly shallow: two questions, then resolution. A production version would need materially deeper questioning within at least the highest-volume categories before I'd trust it past a demo. One thing deliberately out of scope: this prototype routes *intent* but doesn't parse structured attributes embedded in a query — form (Tabletten vs. Saft vs. Kautabletten), dosage, or audience. That's a separate, smaller problem — a closed pharmaceutical-form vocabulary is a good fit for simple dictionary lookup — but it's a real gap between this prototype and a real query-understanding layer.

**What I'd do differently with more time** is build the eval pipeline before shipping, not after. Right now the taxonomy is hand-authored — from intuition plus real evidence I pulled from the live site — not validated against actual customer query distribution. I'd want a golden set of labeled query→cascade-layer pairs, scored against real (anonymized) query logs, to know where the lexical rules actually break down before deciding where to spend on embeddings or LLM-assisted extraction. Right now I'm guessing where the long tail is; I'd rather measure it.

One scoping choice worth naming explicitly: I scoped this to Germany, Redcare's largest market and one of only three where prescription medicines are sold — which is what justifies a fourth resolution path ("this may need a prescription") beyond direct/category/escalation. It also grounds the prototype in Germany's HWG (Heilmittelwerbegesetz), the law regulating advertising of medicinal products:

- **HWG §12** restricts advertising that references certain diseases (per Annex A) to the general public. I'd want legal to confirm exactly which taxonomy terms fall under this — I applied the §12 label to two terms in this prototype as a worked example, not as a complete audit.
- **HWG §11** prohibits fear-inducing disease representations. The safety-escalation screen's calm, factual tone isn't just good UX — it's aligned with this legal constraint on how disease-related content can be presented to consumers in Germany.
