---
name: nondeterministic-design
description: Design the containment boundaries around a project's non-deterministic (LLM/fuzzy) parts — what's deterministic (typed/validated, DDD invariants) vs scored (evals), and where non-determinism is allowed. Use before implementation, after harness-setup. Triggers on "nondeterministic-design", "design the boundaries", "deterministic vs scored", "domain boundaries", or "DDD containment".
---

# Nondeterministic-design — contain the fuzzy core

Designs the deterministic shells around each non-deterministic feature and produces
`docs/domain-boundaries.md`. Doc only — no code, no eval datasets. The scored boundaries
feed `eval-authoring`; the layer-3 invariants become TDD targets. Conditional: applies only
if the project has non-deterministic parts. Read `boundary-patterns.md` for the contract.

## Step 0 — Applicability + mode detection (first)
- Project has NO non-deterministic/LLM parts (see `boundary-patterns.md` "Non-determinism
  detection") → report "item 4 N/A", write nothing, stop.
- No `docs/domain-boundaries.md` → GENERATE mode.
- Exists → AUDIT mode.

## GENERATE mode
1. Find the non-deterministic touchpoints — scan the spec/code for LLM calls or
   generate/summarize/classify/extract/converse-over-NL.
2. Per touchpoint, apply the `boundary-patterns.md` shell model: define the layer-1 input
   contract; name the layer-2 core (mark scored); list the layer-3 output invariants.
3. Classify each sub-decision deterministic vs scored via the heuristic.
4. Per scored boundary, capture acceptance-criteria intent + eval kind (output/trajectory);
   if trajectory, flag the observability dependency (item 6 needs item 7 to capture the trace).
5. Write `docs/domain-boundaries.md` from `templates/domain-boundaries.md.template` (one
   Feature block per touchpoint).
6. Summarize — note `eval-authoring` turns scored boundaries into evals; layer-3 invariants
   become TDD targets.

## AUDIT mode
1. Read existing `docs/domain-boundaries.md` + the touchpoints in spec/code.
2. Check the `boundary-patterns.md` audit failure modes: coverage, over-scoring,
   under-scoring, missing layer-3 invariants.
3. Propose diffs — advisory.
4. Refresh `docs/domain-boundaries.md`.

## Rules
- Doc only — never write code, eval datasets, or BDD/Gherkin scenarios (eval-authoring's job).
- Push the deterministic boundary inward: when a typed invariant CAN capture it, it is
  deterministic, not scored.
- Never invent fuzzy boundaries for a deterministic project — if no non-deterministic parts,
  report N/A.

## Reference files
- `boundary-patterns.md` — shell model, heuristic, invariant patterns, audit failure modes.
- `templates/domain-boundaries.md.template` — the produced design doc.
