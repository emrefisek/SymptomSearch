# Redcare Symptom-to-Product Discovery

Case-study prototype for adaptive health search: route product intent directly, guide ambiguous needs, and escalate safety-sensitive queries before retrieval and ranking.

**Prototype:** https://emrefisek.github.io/SymptomSearch/

## Demo

- `ibuprofen`: direct catalogue route
- `sunburn`: low-ambiguity self-care route
- `I can't sleep`: guided clarification
- `chest pain`: safety escalation

## Files

- `symptom-discovery-prototype.html`: interactive prototype and routing trace
- `WRITEUP.md`: case narrative
- `DESIGN-DECISIONS.md`: architecture, modeling boundaries, and governance
- `EVAL-METHODOLOGY.md`: held-out offline eval and next steps

This is a case-study prototype, not a medical product. Its measured seed eval fails the safety release guardrail; it is not production-safe.
