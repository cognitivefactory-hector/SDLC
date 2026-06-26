# Eval-Authoring Skill — Design Spec

**Date:** 2026-06-26
**Status:** Approved design, ready for implementation plan
**Suite:** SDLC skill suite (skill #5 of 5 — final)

---

## 1. Purpose

A personal Claude Code skill that turns the **scored boundaries** from
`nondeterministic-design` into real eval artifacts — datasets, scorers, and a CI step — per
*The New SDLC with Vibe Coding* and the companion `observability-and-testing-guide.html`
(§2).

It produces what `preflight` gate item 6 checks: output evals (DeepEval scorers in CI) and,
where applicable, trajectory evals (scored from observability traces), plus a dataset + a CI
eval step. Conditional: only applies when scored boundaries exist.

## 2. Scope

- **Hybrid: scaffold + checklist.** WRITES real eval scaffolding (dataset file, DeepEval test
  file(s), CI step) and CHECKLISTS the install/credential steps (`pip install deepeval`,
  LM-judge API key, wire the CI runner).
- **DeepEval default**, promptfoo (YAML, language-agnostic) noted as the fallback when Python
  isn't wanted — not scaffolded by default.
- **Both eval kinds:** output evals for every scored boundary; trajectory evals where the
  boundary's eval kind is trajectory AND observability (item 7) is wired — otherwise the
  trajectory eval's dependency is checklisted, not faked.
- Conditional: no scored boundaries → report item 6 N/A and write nothing.

## 3. Decisions log (from brainstorming)

| Decision | Choice |
|---|---|
| Behavior | **Hybrid**: scaffold real eval files + checklist install/credential steps |
| Tooling | **DeepEval default**, promptfoo as the fallback note |
| Operational approach | Approach A — scaffold-driven + shared reference + templates |
| State model | State-aware GENERATE / AUDIT |
| Input | reads `nondeterministic-design`'s `docs/domain-boundaries.md` (scored boundaries + acceptance intent + eval kind) |
| Skill structure | thin `SKILL.md` + shared `eval-patterns.md` + templates |

## 4. Architecture & files

```
~/.claude/skills/eval-authoring/
├── SKILL.md                       # orchestrator: mode detect → read boundaries → map scorers → scaffold + checklist
├── eval-patterns.md               # keystone: scorer catalog, output-vs-trajectory, dataset seeding, CI step, promptfoo fallback
└── templates/
    ├── eval_test.py.template        # DeepEval test file (output + trajectory scorers)
    ├── dataset.jsonl.template        # dataset seed
    └── eval-checklist.md.template    # install/credential/CI checklist
```

Writes into the **target project**: `evals/` (test + dataset) and `docs/eval-checklist.md`.
Reads `docs/domain-boundaries.md` as input. Produces what `preflight` item 6 checks.

## 5. The `eval-patterns.md` reference (content contract)

### Scorer catalog (acceptance-criteria kind → DeepEval metric)

| Acceptance intent kind | DeepEval metric |
|---|---|
| Faithful / grounded in source | `FaithfulnessMetric` |
| No hallucinated facts | `HallucinationMetric` |
| Relevant to the query | `AnswerRelevancyMetric` |
| Quality / tone / style / "looks right" | `GEval` (criteria from the acceptance intent) |
| RAG context use | `ContextualPrecisionMetric` / `ContextualRecallMetric` (Ragas as alt) |
| Trajectory: task done | `TaskCompletionMetric` |
| Trajectory: right tools | `ToolCorrectnessMetric` |

### Output vs trajectory mapping
- **Output eval** — scores the final artifact; needs only the output. Default for every
  scored boundary.
- **Trajectory eval** — scores the path; needs the trace from observability (item 7).
  Applies when the boundary's eval kind is `trajectory`.
- **DEPENDENCY:** if observability isn't wired, checklist the trajectory eval — never fake it
  without the trace.

### Dataset-seeding rules
- Turn each boundary's acceptance intent into 5–20 seed cases.
- An FAQ / existing examples make excellent seed data.
- Format: JSONL — `input`, optional `expected_output`/`context`, plus the rubric/criteria.
- Mark it a starter to grow from real failures.

### CI step
- `deepeval test run evals/` as a CI job, gating the build on a score threshold.

### LM-judge calibration note
- LM-as-judge needs calibrating — spot-check against human review on borderline cases before
  trusting it.

### promptfoo fallback
- If Python isn't wanted, the same boundaries map to a promptfoo YAML config (noted, not
  scaffolded by default).

## 6. Behavior — state-aware modes

Invoked by intent (e.g. "author the evals", "eval-authoring", "scaffold the eval suite").

### Mode detection + applicability (first)
- No scored boundaries (`docs/domain-boundaries.md` absent or contains none) → report
  **item 6 N/A**, write nothing, stop.
- No `evals/` → **GENERATE**.
- Existing evals → **AUDIT**.

### GENERATE mode
1. Read `docs/domain-boundaries.md` → scored boundaries + acceptance intent + eval kind.
2. Map each boundary's acceptance intent → scorer(s) via the `eval-patterns.md` catalog.
3. Seed a dataset (5–20 cases) from the acceptance intent / examples.
4. Scaffold into the project: `evals/<feature>_eval.py` (output scorers; trajectory scorers
   too if eval kind is trajectory AND observability is wired) + `evals/dataset.jsonl`.
5. Trajectory dependency check: a boundary needing trajectory eval but item 7 not wired →
   checklist it, do not scaffold a broken trajectory eval.
6. Write `docs/eval-checklist.md` from the template: `pip install deepeval`, LM-judge API
   key, wire the CI step, grow the dataset, + any trajectory→observability items.
7. Summarize.

### AUDIT mode
1. Read existing `evals/` + `docs/domain-boundaries.md`.
2. Check coverage: every scored boundary has an output eval? trajectory boundaries have
   trajectory evals (or a flagged dependency)? is there a CI step? is the dataset more than a
   stub?
3. Propose diffs — advisory; scaffold on confirm.
4. Refresh `docs/eval-checklist.md`.

## 7. Integration

- **→ `preflight`:** produces item 6; running it flips item 6 to ✅ (or N/A).
- **← `nondeterministic-design`:** reads `docs/domain-boundaries.md` (scored boundaries).
- **← `harness-setup`:** trajectory evals consume the observability traces it wired (item 7)
  — this closes the loop (ND marks trajectory-scored → HS wires the trace → EA scores it).
- **→ TDD:** complementary — ND's deterministic invariants go to TDD; its scored boundaries
  come here.
- **Conditional & flow:** applies only when scored boundaries exist; sits last:
  `… → nondeterministic-design → eval-authoring → implementation`. Final skill of the suite.

## 8. Testing / acceptance scenarios

Skills aren't unit-testable; acceptance = documented dry-runs:
1. **Deterministic project, no scored boundaries** → reports item 6 N/A, writes nothing.
2. **Summarizer with an output-eval boundary** → GENERATE: maps "faithful / concise / no
   hallucination" → `FaithfulnessMetric` + `HallucinationMetric` + `GEval`; seeds
   `evals/dataset.jsonl`; scaffolds `evals/summarizer_eval.py` + CI step; checklist covers
   `deepeval` install + LM-judge API key.
3. **Agentic boundary needing trajectory eval, observability NOT wired** → GENERATE scaffolds
   the output eval, but checklists the trajectory eval's item-7 dependency (no broken
   trajectory eval). If observability IS wired → also scaffolds
   `TaskCompletionMetric`/`ToolCorrectnessMetric`.

Passes when: no boundaries → N/A; output boundary → scorer-mapped scaffold + dataset + CI;
trajectory-without-observability → checklisted dependency, not a broken eval.

## 9. Out of scope

- Running the evals / installing deps / executing LM-judges (costs credits) — checklisted.
- promptfoo scaffolding (fallback note only).
- The deterministic invariants — those go to TDD, not here.

## 10. Related artifacts

- `~/.claude/skills/preflight/` — the gate this skill feeds (item 6).
- `~/.claude/skills/nondeterministic-design/` — produces the scored boundaries this consumes.
- `~/.claude/skills/harness-setup/` — wires the observability (item 7) trajectory evals need.
- `observability-and-testing-guide.html` §2 — tests-vs-evals, DeepEval source.
