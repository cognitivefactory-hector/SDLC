# Eval Patterns — the eval-authoring contract

Shared reference for the `eval-authoring` skill. Maps scored boundaries (from
nondeterministic-design's `docs/domain-boundaries.md`) to scorers, and defines dataset
seeding, the CI step, and the output-vs-trajectory split.

DeepEval is the default tool; promptfoo is the fallback (noted, not scaffolded).

## Scorer catalog (acceptance-criteria kind → DeepEval metric)

| Acceptance intent kind | DeepEval metric |
|---|---|
| Faithful / grounded in source | FaithfulnessMetric |
| No hallucinated facts | HallucinationMetric |
| Relevant to the query | AnswerRelevancyMetric |
| Quality / tone / style / "looks right" | GEval (criteria from the acceptance intent) |
| RAG context use | ContextualPrecisionMetric / ContextualRecallMetric (Ragas as alt) |
| Trajectory: task done | TaskCompletionMetric |
| Trajectory: right tools | ToolCorrectnessMetric |

## Output vs trajectory

- OUTPUT eval — scores the final artifact; needs only the output. Default for every scored
  boundary.
- TRAJECTORY eval — scores the path; needs the trace from observability (item 7). Applies
  when the boundary's eval kind is `trajectory`.
- DEPENDENCY: if observability is not wired, CHECKLIST the trajectory eval — never scaffold a
  trajectory eval without the trace.

## Dataset seeding

- Turn each boundary's acceptance intent into 5–20 seed cases.
- An FAQ / existing examples make excellent seed data.
- Format: JSONL — `input`, optional `expected_output`/`context`, plus the rubric/criteria.
- Mark it a starter to grow from real failures.

## CI step

Run the eval suite in CI and gate on a threshold:

```yaml
- name: LLM evals
  run: deepeval test run evals/
```

## LM-judge calibration

LM-as-judge needs calibrating — spot-check its scores against human review on borderline
cases before trusting it.

## promptfoo fallback

If Python is not wanted, the same boundaries map to a promptfoo YAML config
(`promptfooconfig.yaml`) with `assert` blocks. Noted only; not scaffolded by default.

## Audit coverage checks (AUDIT mode)

- Every scored boundary has an OUTPUT eval.
- Trajectory boundaries have a trajectory eval OR a flagged item-7 dependency.
- A CI eval step exists.
- The dataset is more than a stub.
