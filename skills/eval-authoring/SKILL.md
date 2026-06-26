---
name: eval-authoring
description: Turn the scored boundaries from nondeterministic-design into eval artifacts — datasets, DeepEval scorers (output + trajectory), and a CI step. Use after nondeterministic-design, before/around implementation. Writes eval scaffolding and checklists the install/credential steps. Triggers on "eval-authoring", "author the evals", "scaffold the eval suite", "set up DeepEval", or "write evals".
---

# Eval-authoring — scaffold the eval suite

Reads the scored boundaries from `docs/domain-boundaries.md` and scaffolds output (and where
applicable trajectory) evals using DeepEval, plus a CI step and an install checklist. Hybrid:
WRITE the eval files, CHECKLIST the install/credential steps. Conditional: applies only if
scored boundaries exist. Read `eval-patterns.md` for the contract.

## Step 0 — Applicability + mode detection (first)
- No scored boundaries (`docs/domain-boundaries.md` absent or contains none) → report
  "item 6 N/A", write nothing, stop.
- No `evals/` directory → GENERATE mode.
- Existing `evals/` → AUDIT mode.

## GENERATE mode
1. Read `docs/domain-boundaries.md` → scored boundaries + acceptance intent + eval kind.
2. Map each boundary's acceptance intent → scorer(s) via the `eval-patterns.md` scorer catalog.
3. Seed a dataset (5–20 cases) from the acceptance intent / examples.
4. Scaffold into the project from `templates/`:
   - `evals/<feature>_eval.py` from `eval_test.py.template` (output scorers; trajectory
     scorers too ONLY if the eval kind is trajectory AND observability is wired).
   - `evals/dataset.jsonl` from `dataset.jsonl.template`.
5. Trajectory dependency check: a boundary needing a trajectory eval but observability
   (item 7) not wired → CHECKLIST it; do not scaffold a broken trajectory eval.
6. Write `docs/eval-checklist.md` from `eval-checklist.md.template`.
7. Summarize — note item 6 flips ✅ once the dataset is seeded and the CI step is wired.

## AUDIT mode
1. Read existing `evals/` + `docs/domain-boundaries.md`.
2. Check the `eval-patterns.md` audit coverage checks: every scored boundary has an output
   eval; trajectory boundaries have a trajectory eval or a flagged dependency; a CI step
   exists; the dataset is more than a stub.
3. Propose diffs — advisory; scaffold on confirm.
4. Refresh `docs/eval-checklist.md`.

## Rules
- Hybrid: WRITE eval scaffolding (tests, dataset, CI step); CHECKLIST install/credentials
  (`pip install deepeval`, LM-judge API key). Do NOT run evals or install deps.
- Never scaffold a trajectory eval without the observability trace (item 7) — checklist it.
- DeepEval is the default; note promptfoo only as a fallback (do not scaffold it).
- If no scored boundaries, report item 6 N/A — do not invent evals.

## Reference files
- `eval-patterns.md` — scorer catalog, output/trajectory, dataset seeding, CI step, audit checks.
- `templates/eval_test.py.template`, `templates/dataset.jsonl.template`,
  `templates/eval-checklist.md.template` — the scaffolds.
