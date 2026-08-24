# Eval Methodology — Symptom-to-Product Discovery

This document exists to make one concept concrete: **what "eval" actually means for a search/routing system, how you build one, and what it looks like when you run it against something real.** It uses this project's own routing engine as the worked example throughout — every number below was computed by actually running 20 held-out queries through `symptom-discovery-prototype.html`'s real `route()` function, not invented for illustration. That matters: made-up example numbers teach you the shape of the idea; real numbers on your own system teach you where it's actually wrong, which is the entire point of doing an eval in the first place.

---

## 1. What "eval" means here

"Eval" (evaluation) is the practice of measuring how a system behaves against a set of *known-correct answers*, so you have evidence instead of an impression. It is different from a unit test in one important way: a unit test checks that code does what the code is supposed to do (deterministic, code-level correctness). An eval checks that the system does what the *product* is supposed to do against realistic inputs — including inputs the code was never explicitly written to handle. A routing cascade can pass every unit test and still fail its eval, because unit tests check the rules you wrote, and eval checks the queries real people actually type.

The three ingredients of any eval:
1. **A golden set** — a collection of inputs paired with the correct expected output, usually built by a human who applies judgment.
2. **A run** — the system under test processes every input in the golden set.
3. **A scoring step** — comparing actual output to expected output and computing metrics.

That's it structurally. The craft is in *what* you measure and *how you label the golden set well* — both covered below.

## 2. Two kinds of metrics: guardrail vs. quality

This is the single most important distinction in this document, because getting it wrong is how a safety mechanism quietly degrades over time.

| | **Guardrail metric** | **Quality metric** |
|---|---|---|
| What it answers | "Did something unacceptable happen?" | "How good is the system overall?" |
| Target | As close to the extreme as measurable (e.g. 0% miss rate) | A reasonable, improvable number |
| Trade-off behavior | **Never traded off.** A guardrail regression blocks a release regardless of what else improved. | **Expected to trade off** against other quality metrics as the system is tuned. |
| Example in this project | Missed-escalation rate — a red-flag query that should have escalated but didn't | Category-match precision — how often a resolved category is the *right* category |

The failure mode this distinction prevents: a team ships a change that improves average category-match precision by 5%, and nobody notices it also raised the missed-escalation rate from 0% to 2%, because both numbers lived on the same dashboard with the same visual weight. A guardrail metric needs a different review process — ideally a release-blocking check, not a chart someone glances at.

## 3. What a golden set actually is, and how this one was built

A golden set is a table of `(input, expected_output, rationale)` rows, hand-labeled by someone applying judgment about what the *correct* behavior is — not what the current system happens to do.

For this project, the golden set below was built by deliberately **not** reusing any of the 52 phrases already hardcoded into the app's own taxonomy. Testing a system against its own training examples tells you the system can reproduce itself — it tells you nothing about generalization. Instead, each row is a natural paraphrase of a risk category or symptom category the taxonomy already claims to handle, written the way a real person might actually type it. This is the same principle as a held-out test set in ML: the value of an eval comes specifically from inputs the system-builder didn't have in mind when writing the rules.

Three groups, 20 rows total:

**Group 1 — Safety (10 rows).** Natural rephrasings of the same clinical risks the exact red-flag list and near-miss heuristic are supposed to catch.

| ID | Query | Expected |
|---|---|---|
| S1 | "my chest feels really tight and it's scary" | escalate |
| S2 | "i can't seem to catch my breath at all" | escalate |
| S3 | "there's a lot of pressure in my chest right now" | escalate |
| S4 | "sudden chest tightness that won't go away" | escalate |
| S5 | "i feel like i'm having a heart attack" | escalate |
| S6 | "i feel like i might pass out any second" | escalate |
| S7 | "my left arm feels numb and my chest hurts" | escalate |
| S8 | "sudden numbness down my left arm" | escalate |
| S9 | "coughing up a little blood this morning" | escalate |
| S10 | "i think i'm going to lose consciousness" | escalate |

**Group 2 — HWG-adjacent (4 rows).** Natural rephrasings of the two clinically/legally sensitive taxonomy terms, which should never silently resolve to a confident OTC product.

| ID | Query | Expected |
|---|---|---|
| H1 | "i think this spot on my skin might be cancer" | referral / flagged, not a confident OTC product |
| H2 | "there's a dark spot that's gotten bigger" | referral / flagged, not a confident OTC product |
| H3 | "i drink every day and can't stop myself" | referral / flagged, not a confident OTC product |
| H4 | "i just can't stop drinking no matter what" | referral / flagged, not a confident OTC product |

**Group 3 — Category / product-discovery (6 rows).** Synonym coverage for symptom buckets the taxonomy already has categories for.

| ID | Query | Expected |
|---|---|---|
| C1 | "paracetamol 500mg tablets" | direct match, Pain & Fever Relief |
| C2 | "my hay fever has been brutal this week" | category-light, Allergy & Hay Fever |
| C3 | "can't fall asleep no matter what i try" | sleep bucket |
| C4 | "my stomach has been upset since lunch" | digestive bucket |
| C5 | "i twisted my ankle and now it's swollen" | joint bucket |
| C6 | "my baby has a rash on her cheeks" | skin bucket |

## 4. Running it, and the actual results

Each query was run through the app's real `route()` function in a headless browser (no UI clicking — the routing decision is deterministic and available directly). Actual outcomes, unedited:

**Group 1 — Safety.** 5 of 10 escalated correctly; 5 fell through to guided flow with zero warning:

| ID | Caught? | Why |
|---|---|---|
| S1 | ✅ | near-miss: "chest" + "tight" |
| S2 | ✅ | near-miss: "breath" + "can't" |
| S3 | ✅ | near-miss: "chest" + "pressure" |
| S4 | ✅ | near-miss: "chest" + "tight" |
| S5 | ❌ **missed** | "heart" is a risk noun, but "attack" isn't a listed modifier |
| S6 | ❌ **missed** | "pass out" is a listed modifier, but no risk noun is present in the sentence |
| S7 | ❌ **missed** | two risk nouns ("numb", "chest") co-occur, but the heuristic requires noun **+** modifier, not noun + noun |
| S8 | ✅ | near-miss: "numb" + "sudden" |
| S9 | ❌ **missed** | "blood" present, but no listed modifier — "coughing up **a little** blood" breaks the exact-phrase match, and no near-miss modifier applies |
| S10 | ❌ **missed** | "conscious" (via "consciousness") present, but "lose" isn't a listed modifier |

**Safety recall = 5/10 = 50%.**

**Group 2 — HWG-adjacent.** 0 of 4 were flagged in any way:

| ID | Flagged? | What actually happened |
|---|---|---|
| H1 | ❌ | Landed in the "skin" bucket by lucky coincidence (the single word "skin" matched the ambiguous-fallback list) — then proceeds through guided flow toward a confident skin-cream recommendation, with **zero indication** this might be cancer-adjacent |
| H2 | ❌ | No match at all; generic fallback |
| H3 | ❌ | No match at all; generic fallback |
| H4 | ❌ | Doesn't contain the exact phrase "i can't stop drinking" as a substring, so the existing HWG flag never fires |

**HWG-adjacent detection recall = 0/4 = 0%.**

**Group 3 — Category / product-discovery.** 3 of 6 landed in the intended bucket:

| ID | Correct? | What actually happened |
|---|---|---|
| C1 | ✅ | Catalogue substring match works regardless of surrounding text ("500mg tablets" doesn't break it) |
| C2 | ✅ | Taxonomy substring match works the same way |
| C3 | ❌ | "can't fall asleep" doesn't contain the exact phrase "i can't sleep" → generic fallback instead of the sleep bucket |
| C4 | ✅ | Matched via the single-word ambiguous-fallback entry "stomach" |
| C5 | ❌ | No taxonomy entry for injury/joint-adjacent phrasing at all → generic fallback |
| C6 | ❌ | No taxonomy entry for rash/skin phrasing outside the two hardcoded examples, and no age/audience handling → generic fallback |

**Category-match accuracy = 3/6 = 50%.**

## 5. What these numbers actually mean

**The safety number is the one that matters most, and it fails.** A guardrail target of near-zero misses is not met by 50% recall on held-out adversarial phrasing. The near-miss heuristic improved several phrasings, but the eval proves the underlying approach is not production-ready. The next direction is not endless keyword patching: use a clinically reviewed safety ontology, high-recall semantic/concept detection, paraphrase and adversarial coverage, and potentially multiple independent safety signals. Any probabilistic component must remain subject to deterministic override/fail-safe behavior and clinical release review.

**The sensitive-topic number is the sharpest finding in this whole exercise.** Zero of four. A query about a possibly cancerous skin spot can land in the skin bucket and progress toward a cream recommendation with no warning. This is a reproducible gap, not a hypothetical concern. It requires its own clinically and legally reviewed concept/referral strategy, evaluated separately from emergency safety; the two should not be treated as one keyword problem or as a legal conclusion already reached.

**The category-match number is a normal quality metric, not a guardrail** — 50% is a real gap (taxonomy synonym coverage is thin), but the correct response is prioritized backlog work, not a release block. This is also where the ML section's proposal (embeddings/semantic matching once lexical rules run out, per `DESIGN-DECISIONS.md` §3) earns its place: C3, C5, and C6 are precisely the long-tail phrasing case that hand-authored lexical rules structurally can't scale to cover one string at a time.

## 6. Scaling this beyond a 20-row golden set

The process above works at any scale; only the sourcing of the golden-set rows changes:

1. **Seed set (what this document used):** hand-authored by a PM/domain expert, deliberately held out from the system's own taxonomy. Good for pre-launch validation and catching structural gaps like the HWG one above.
2. **Sampled production queries (once there's real traffic):** pull a stratified random sample of real anonymized queries — stratified by query length and by which cascade layer currently handles them, so rare-but-important cases (like red-flag-adjacent queries, which should be a tiny fraction of total volume) aren't drowned out by common ones in a pure random sample.
3. **Human labeling with a rubric:** each sampled query gets labeled by a person following a written rubric (exactly the "expected" column above, but for real queries). For anything safety- or HWG-adjacent, use **two independent labelers** and measure inter-annotator agreement — if two people label the same ambiguous query differently, that disagreement is itself useful signal about where the taxonomy's boundaries are unclear.
4. **Continuous growth:** every production incident, every escalation a human reviewer disagrees with, and every support ticket that traces back to a routing miss becomes a new golden-set row. The golden set should only ever grow, and a query that was mislabeled once should never be able to silently regress without the eval catching it.

Minimal pseudocode for the scoring step once you have real predictions logged, matching exactly what §5 above did by hand:

```
for each row in golden_set:
    prediction = system.route(row.query)
    row.correct = matches(prediction, row.expected)

for each guardrail_group (e.g. "safety"):
    recall = count(correct in group) / count(group)
    assert recall >= guardrail_threshold   # blocks release if this fails

for each quality_group (e.g. "category"):
    precision = count(correct in group) / count(predicted_as_group)
    recall    = count(correct in group) / count(expected_as_group)
    # tracked on a dashboard, reviewed regularly, allowed to trade off
```

## 7. Offline eval vs. online eval — and where they connect

Everything above is **offline eval**: scoring against a fixed golden set before anything ships, ideally as a release gate (this is the "Eval pipeline... validates before release" box in `DESIGN-DECISIONS.md` §9's production-architecture diagram). It answers "is the routing logic correct?"

It cannot answer a different, equally important question: **is this feature good for the business?** That requires **online eval** — real users, real traffic, and an A/B test against the current keyword-search baseline. Conversion delta, guided-flow abandonment, and escalation precision require behavioral data. Return behavior 7–14 days after referral may be a useful signal, but is not proof of clinical correctness. The two evaluation modes are complementary: a feature can pass offline guardrails and still be commercially poor, or appear commercially attractive while retaining a golden-set-detectable safety defect. Online testing follows, rather than replaces, offline safety gates.

## 8. Summary table — what to track, and what kind of number each one is

| Metric | Type | Target / behavior | Source in this project |
|---|---|---|---|
| Missed-escalation rate (safety) | Guardrail | ~0%, release-blocking | §5 above: currently 50% on held-out set — fails |
| HWG-adjacent-topic miss rate | Guardrail (proposed) | ~0%, release-blocking | §5 above: currently 0% recall — fails badly, needs its own near-miss net |
| Escalation precision | Quality | Improvable, monitored so over-triggering doesn't go unchecked | Not computed here (needs real traffic volume) |
| Category-match precision/recall | Quality | Improvable | §5 above: 50% on held-out set |
| Guided-flow abandonment | Quality | Improvable | Needs real traffic |
| Conversion delta vs. baseline | Commercial (online only) | Positive, statistically significant | Needs an A/B test |
| Return-within-14-days after referral/escalation | Commercial (online only) | Interpret as a behavioral signal, not clinical proof | Needs real traffic |
