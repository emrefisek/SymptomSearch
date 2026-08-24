# Symptom-to-Product Discovery — Case Study Write-up

**Prototype:** https://emrefisek.github.io/SymptomSearch/
**Design decisions & diagrams:** see `DESIGN-DECISIONS.md` in this repo.

---

Today, searching "I can't sleep" on Redcare returns a standard keyword results page — mostly melatonin, ranked by popularity. There's no experience for a customer who arrives with a problem rather than a product in mind. Rather than building a single "symptom search," I built **intent routing**: a deterministic cascade that recognizes different query types need fundamentally different treatment, and only applies guided discovery where it actually helps.

| Layer | Fires on | Outcome |
|---|---|---|
| 1 — Safety | Red-flag phrase (e.g. "chest pain") | Hard-stop escalation — never a product, overrides everything below |
| 2 — Catalogue | Known brand/ingredient (e.g. "ibuprofen") | Direct results, no guidance |
| 3 — Low-ambiguity self-care need | Relatively low routing ambiguity (e.g. "sunburn") | Light category route, escape hatch to guided flow |
| 4 — Long-form | Natural-language problem (e.g. "I can't sleep") | Guided flow, 1–2 clarifying questions |
| 5 — Ambiguous | Short, under-specified (e.g. "pain") | Guided flow, broader initial question |

**The decision I'm most proud of** is this layered cascade, paired with a visible **routing-trace panel** — every result shows *which layer handled the query and why*, live on screen, which turns the architecture into something demonstrable rather than only explained. Two real failures on the live Shop Apotheke site motivated it: "Herzinfarkt" (heart attack) resolves straight to products with zero escalation, and "Herzschmerzen" (heart pain — cardiac, muscular, anxiety, or reflux are all plausible) resolves *confidently* to aspirin instead of asking one question first. That principle got tested against my own build: an early safety layer used only exact-phrase matching, so real phrasing like "my chest feels really tight" fell straight through it. The fix — a second, deliberately high-recall check (risk-area word + risk modifier) that escalates even without an exact match — is now in the prototype, demoable side-by-side with the original list.

**The biggest trade-off I made** was breadth of taxonomy over depth of questioning — 52 phrases across five prototype domains, but only two clarifying questions per guided flow. These are not a complete pharmacy taxonomy. Production needs governed mappings from consumer language and need state through symptom/clinical concept, therapeutic goal and class, ingredient, formulation and eligibility constraints to the catalogue product/PZN. Two further trade-offs worth naming: *ranking* within a category is illustrative, not real ("Top match"/"Also relevant" labels with a mock rule, since there's no click data to rank against) — and I deliberately didn't tackle structured attribute extraction (form, dosage, audience, child vs. adult, or other eligibility constraints) or the commercial question of whether routing customers *away* from a purchase is net-positive versus today's baseline. That last one needs a real A/B test, not an assumption.

**The seed offline eval has been built and run** against 20 held-out queries. It exposes the key result: safety recall is 50% (5/10), sensitive-topic detection is 0% (0/4), and category-match accuracy is 50% (3/6). I would not ship this implementation. Missed safety escalation and sensitive-topic detection are release-blocking guardrails, not numbers to trade for conversion; category coverage is a quality metric to improve. The next milestone is clinically reviewed concept-level coverage, adversarial/paraphrase evaluation, and independent safety signals with deterministic override behavior, not endlessly adding keywords. A production CI/release gate and a large expert-labeled dataset are still to be built. Full method and results are in [`EVAL-METHODOLOGY.md`](https://github.com/emrefisek/SymptomSearch/blob/main/EVAL-METHODOLOGY.md).

One scoping choice worth naming: I scoped this to Germany, Redcare's largest market and one of only three where prescription medicines are sold — which justifies a fourth resolution path ("this may need a prescription"). It also grounds the prototype in HWG (Heilmittelwerbegesetz): **§12** restricts advertising referencing certain diseases (Annex A) to the public, and **§11** bans fear-inducing disease framing, which is why the escalation screen's tone is deliberately calm and factual. To be precise about weight: the primary reason the two HWG-flagged queries resolve to a referral is clinical judgment, not the statute — I applied the §12 label as a worked example on two terms, not a complete audit or legal conclusion.
