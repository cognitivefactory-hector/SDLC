# Boundary Patterns — the nondeterministic-design contract

Shared reference for the `nondeterministic-design` skill. Distills the shell model from
"The New SDLC with Vibe Coding" (guide §3) into a classification reference: the 3-layer
shell, the deterministic-vs-scored heuristic, invariant patterns, and what to capture per
scored boundary for `eval-authoring`.

This skill DESIGNS boundaries (a doc); it writes no code and no eval datasets.

## The 3-layer shell model (apply per non-deterministic feature)

| Layer | What it is | Guarded by |
|---|---|---|
| 1. Deterministic input contract | typed inputs, schema validation, ubiquitous language | types + tests |
| 2. Non-deterministic core | the LLM/fuzzy generation | scored (evals) |
| 3. Deterministic output validation | domain invariants reject bad output | types + invariants |

Rule of the model: push the deterministic boundary inward until the non-deterministic core
is as small as possible, then validate everything crossing it.

## Deterministic-vs-scored heuristic

- DETERMINISTIC ← the assertion is expressible as `==`, a typed invariant, or a schema check.
  Guarded by types + tests.
- SCORED ← it would take a human saying "that's good enough" (quality, relevance, tone,
  faithfulness, or did-it-take-a-sane-path). Guarded by evals.
- When a typed invariant CAN capture it, it is deterministic, not scored. Only score what
  genuinely needs judgment.

## Invariant patterns (the layer-3 wall)

- Type/schema conformance (output parses into a typed object)
- Range/domain constraints (e.g. quantity > 0, currency in an allowed set)
- Cross-field consistency (e.g. total == sum of line items)
- Referential validity (referenced IDs exist)
- Required / non-empty fields
- Enum membership

## Per scored boundary, capture (for eval-authoring)

- Acceptance-criteria intent — "what good looks like," in plain language (the rubric).
- Eval kind: OUTPUT (the artifact) vs TRAJECTORY (the path). Trajectory evals require
  observability (item 7) to capture the trace — flag that dependency.

## Audit failure modes (AUDIT mode)

- COVERAGE: a non-deterministic touchpoint with no boundary entry.
- OVER-SCORING: something marked scored that a typed invariant could make deterministic →
  push it inward to a layer-3 invariant.
- UNDER-SCORING: something marked deterministic that actually needs human judgment → it
  should be scored.
- Missing layer-3 invariants for a core.

## Non-determinism detection

A feature is non-deterministic if it calls a model, or generates / summarizes / classifies /
extracts / converses over natural language. If a project has none, item 4 is N/A — report
and write nothing.
