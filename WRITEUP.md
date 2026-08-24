# Symptom-to-Product Discovery — Case Study Write-up

**Prototype:** https://emrefisek.github.io/SymptomSearch/
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

**The decision I'm most proud of** is this layered cascade, paired with a visible **routing-trace panel** — every result shows *which layer handled the query and why*, live on screen, which turns the architecture into something demonstrable rather than only explained. Two real failures on the live Shop Apotheke site motivated it: "Herzinfarkt" (heart attack) resolves straight to products with zero escalation, and "Herzschmerzen" (heart pain — cardiac, muscular, anxiety, or reflux are all plausible) resolves *confidently* to aspirin instead of asking one question first. That principle got tested against my own build: an early safety layer used only exact-phrase matching, so real phrasing like "my chest feels really tight" fell straight through it. The fix — a second, deliberately high-recall check (risk-area word + risk modifier) that escalates even without an exact match — is now in the prototype, demoable side-by-side with the original list.

**The biggest trade-off I made** was breadth of taxonomy over depth of questioning — 52 phrases across five categories, but only two clarifying questions per guided flow. Right for demonstrating the architecture, not yet deep enough to trust past a demo. Two further trade-offs worth naming: *ranking* within a category is illustrative, not real ("Top match"/"Also relevant" labels with a mock rule, since there's no click data to rank against) — and I deliberately didn't tackle attribute extraction (form, dosage, audience embedded in a query) or the commercial question of whether routing customers *away* from a purchase is net-positive versus today's baseline. That last one needs a real A/B test, not an assumption.

**What I'd do differently with more time** is build the eval pipeline before shipping, and be precise about what kind of metric each number is. Missed-escalation rate is a *guardrail* — target ~0%, never traded off for a better average elsewhere. Escalation precision, category-match precision, and resolution rate are normal *quality* metrics, expected to move as the system tunes. Conflating the two is how a safety layer quietly gets weakened for a cleaner dashboard. Full method and worked examples in `EVAL-METHODOLOGY.md`.

One scoping choice worth naming: I scoped this to Germany, Redcare's largest market and one of only three where prescription medicines are sold — which justifies a fourth resolution path ("this may need a prescription"). It also grounds the prototype in HWG (Heilmittelwerbegesetz): **§12** restricts advertising referencing certain diseases (Annex A) to the public, and **§11** bans fear-inducing disease framing, which is why the escalation screen's tone is deliberately calm and factual. To be precise about weight: the primary reason the two HWG-flagged queries resolve to a referral is clinical judgment, not the statute — I applied the §12 label as a worked example on two terms, not a complete audit or legal conclusion.
