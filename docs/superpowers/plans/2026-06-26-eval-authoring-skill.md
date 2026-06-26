# Eval-Authoring Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `eval-authoring` personal skill — a state-aware, hybrid scaffolder that turns `nondeterministic-design`'s scored boundaries into DeepEval output/trajectory evals, a seed dataset, and a CI step, plus an install checklist.

**Architecture:** Five markdown/template files in `~/.claude/skills/eval-authoring/` (same Approach-B pattern as the prior suite skills): a thin `SKILL.md` orchestrator, a shared `eval-patterns.md` contract, and three templates. Hybrid: writes eval scaffolding, checklists install/credential steps.

**Tech Stack:** Claude Code skills (Markdown + YAML frontmatter); the scaffolded evals target DeepEval (Python). No build step for the skill. Verification is behavioral dry-runs.

## Global Constraints

- Install location: `~/.claude/skills/eval-authoring/`. Copy content verbatim.
- Behavior is HYBRID: WRITE eval scaffolding (DeepEval tests, dataset, CI step); CHECKLIST install/credential steps (`pip install deepeval`, LM-judge API key). Do NOT run evals or install deps.
- DeepEval is the DEFAULT tool; promptfoo is the fallback (noted, never scaffolded).
- Two eval kinds: OUTPUT for every scored boundary; TRAJECTORY only when the boundary's eval kind is trajectory AND observability (item 7) is wired — otherwise CHECKLIST the dependency, never scaffold a broken trajectory eval.
- Conditional: no scored boundaries → report "item 6 N/A" and write nothing.
- Input: reads `nondeterministic-design`'s `docs/domain-boundaries.md` (scored boundaries + acceptance intent + eval kind).
- Source of truth: `docs/superpowers/specs/2026-06-26-eval-authoring-skill-design.md`.
- Commits OPTIONAL: only if `~/.claude` is version-controlled; otherwise skip.
- Fence caution: `SKILL.md` starts with `---` on line 1; `eval-patterns.md` and the markdown template start with their `#` heading; `eval_test.py.template` starts with its first Python line; `dataset.jsonl.template` starts with its first JSON line. NO outer display fence around any file. `eval-patterns.md` DOES contain an inner ` ```yaml ` block (the CI snippet) — that is file content; do not add an extra outer fence.

---

## File Structure

| File | Responsibility |
|---|---|
| `~/.claude/skills/eval-authoring/eval-patterns.md` | Contract: scorer catalog, output-vs-trajectory, dataset seeding, CI step, audit checks, promptfoo fallback. Built first. |
| `~/.claude/skills/eval-authoring/templates/eval_test.py.template` | DeepEval test scaffold (output scorers; trajectory commented). |
| `~/.claude/skills/eval-authoring/templates/dataset.jsonl.template` | JSONL seed dataset. |
| `~/.claude/skills/eval-authoring/templates/eval-checklist.md.template` | Install/credential/CI checklist. |
| `~/.claude/skills/eval-authoring/SKILL.md` | Orchestrator: applicability/mode detect → read boundaries → map scorers → scaffold + checklist. |

Build order: `eval-patterns.md` → templates → `SKILL.md` → validation.

---

### Task 1: Create the contract (`eval-patterns.md`)

**Files:**
- Create: `~/.claude/skills/eval-authoring/eval-patterns.md`

**Interfaces:**
- Produces the contract referenced by `SKILL.md` (Task 3). Section anchors relied on: "Scorer catalog", "Output vs trajectory", "Dataset seeding", "CI step", "Audit coverage checks".

- [ ] **Step 1: Define done**

Complete when the file contains the scorer catalog table, the output-vs-trajectory split with the item-7 dependency, dataset-seeding rules, the CI step snippet, the LM-judge calibration note, the promptfoo fallback, and the audit coverage checks.

- [ ] **Step 2: Create the directory**

```bash
mkdir -p ~/.claude/skills/eval-authoring/templates
```

- [ ] **Step 3: Write `eval-patterns.md` with this exact content** (the file contains an inner ` ```yaml ` block — reproduce it; do NOT wrap the whole file in an extra outer fence)

````markdown
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
````

- [ ] **Step 4: Verify**

Run: `grep -cE "Scorer catalog|Output vs trajectory|Dataset seeding|CI step|Audit coverage checks" ~/.claude/skills/eval-authoring/eval-patterns.md`
Expected: `5`.

Run: `grep -cE "FaithfulnessMetric|HallucinationMetric|GEval|TaskCompletionMetric|ToolCorrectnessMetric" ~/.claude/skills/eval-authoring/eval-patterns.md`
Expected: `≥5`.

Run: `grep -c "deepeval test run evals/" ~/.claude/skills/eval-authoring/eval-patterns.md`
Expected: `1`.

- [ ] **Step 5: Commit (optional — only if `~/.claude` is version-controlled)**

```bash
cd ~/.claude && git add skills/eval-authoring/eval-patterns.md && git commit -m "feat(eval-authoring): add eval-patterns contract"
```

---

### Task 2: Create the three templates

**Files:**
- Create: `~/.claude/skills/eval-authoring/templates/eval_test.py.template`
- Create: `~/.claude/skills/eval-authoring/templates/dataset.jsonl.template`
- Create: `~/.claude/skills/eval-authoring/templates/eval-checklist.md.template`

**Interfaces:**
- Consumes nothing. Produces the three scaffolds `SKILL.md` (Task 3) writes into target projects. The `{{TOKEN}}` markers are intentional fill-in fields.

- [ ] **Step 1: Define done**

Complete when all three template files exist with their `{{TOKEN}}` fields and the content below.

- [ ] **Step 2: Write `templates/eval_test.py.template` with this exact content** (raw Python — file starts with the `"""` docstring on line 1, NO outer code fence)

```python
"""DeepEval suite for {{FEATURE_NAME}} — scaffolded by eval-authoring.

Run: deepeval test run evals/
Grow the dataset in dataset.jsonl from real failures.
"""
import json
import os

import pytest
from deepeval import assert_test
from deepeval.metrics import GEval
from deepeval.test_case import LLMTestCase, LLMTestCaseParams

DATASET = os.path.join(os.path.dirname(__file__), "dataset.jsonl")


def load_cases():
    with open(DATASET) as f:
        return [json.loads(line) for line in f if line.strip()]


# Output eval — criteria come from the scored boundary's acceptance intent.
QUALITY = GEval(
    name="{{FEATURE_NAME}} quality",
    criteria="{{ACCEPTANCE_CRITERIA}}",
    evaluation_params=[LLMTestCaseParams.ACTUAL_OUTPUT],
    threshold=0.8,
)


@pytest.mark.parametrize("case", load_cases())
def test_{{FEATURE_SLUG}}_output(case):
    # Replace run_feature(...) with a call to your system under test.
    output = run_feature(case["input"])
    tc = LLMTestCase(input=case["input"], actual_output=output)
    assert_test(tc, [QUALITY])


# Trajectory eval (only if eval kind is trajectory AND observability is wired):
#   from deepeval.metrics import TaskCompletionMetric, ToolCorrectnessMetric
#   build the LLMTestCase from the captured trace (tools_called / steps), then assert_test.
```

- [ ] **Step 3: Write `templates/dataset.jsonl.template` with this exact content** (raw JSONL — one JSON object per line, NO code fence)

```
{"input": "{{EXAMPLE_INPUT}}", "expected_output": null, "criteria": "{{ACCEPTANCE_CRITERIA}}"}
{"input": "{{ANOTHER_INPUT}}", "expected_output": null, "criteria": "{{ACCEPTANCE_CRITERIA}}"}
```

- [ ] **Step 4: Write `templates/eval-checklist.md.template` with this exact content** (starts with `#` on line 1, NO outer code fence)

```markdown
# Eval Checklist — {{PROJECT_NAME}}

Scaffolded by eval-authoring. Output evals are in `evals/`; complete these to run them.

## Install & run
- [ ] `pip install deepeval`
- [ ] Set the LM-judge API key (e.g. `OPENAI_API_KEY` or your provider's) for GEval/Faithfulness
- [ ] Seed `evals/dataset.jsonl` with 5–20 real cases (grow from failures)
- [ ] Replace `run_feature(...)` in the eval test with your system under test
- [ ] Run: `deepeval test run evals/`

## CI
- [ ] Add the eval job step (`deepeval test run evals/`) and gate the build on the threshold

## Trajectory evals (only if applicable)
{{TRAJECTORY_ITEMS}}

## Notes
- Calibrate LM-as-judge against human review on borderline cases before trusting it.
- Re-run `eval-authoring` in AUDIT mode after adding cases to refresh this checklist.
```

- [ ] **Step 5: Verify**

Run: `ls ~/.claude/skills/eval-authoring/templates/`
Expected: `dataset.jsonl.template  eval-checklist.md.template  eval_test.py.template`.

Run: `grep -c "deepeval" ~/.claude/skills/eval-authoring/templates/eval_test.py.template`
Expected: `≥3` (imports + docstring).

Run: `head -1 ~/.claude/skills/eval-authoring/templates/dataset.jsonl.template | python3 -c "import sys,json; json.loads(sys.stdin.read())" && echo VALID_JSON`
Expected: `VALID_JSON` (the first seed line parses as JSON).

- [ ] **Step 6: Commit (optional)**

```bash
cd ~/.claude && git add skills/eval-authoring/templates && git commit -m "feat(eval-authoring): add eval test, dataset, and checklist templates"
```

---

### Task 3: Create the skill orchestrator (`SKILL.md`)

**Files:**
- Create: `~/.claude/skills/eval-authoring/SKILL.md`

**Interfaces:**
- Consumes `eval-patterns.md` (Task 1) and the three templates (Task 2) by relative path.
- Produces the activatable skill. Frontmatter `name: eval-authoring`; `description` contains `eval-authoring`.

- [ ] **Step 1: Define done**

Complete when `SKILL.md` has valid frontmatter, the applicability/N-A + mode-detection step, GENERATE and AUDIT procedures, and the rules (hybrid write/checklist; no broken trajectory eval; DeepEval default; N/A when no boundaries).

- [ ] **Step 2: Write `SKILL.md` with this exact content** (file starts with `---` on line 1 — NO outer code fence)

```markdown
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
```

- [ ] **Step 3: Verify**

Run: `head -2 ~/.claude/skills/eval-authoring/SKILL.md`
Expected: line 1 `---`, line 2 `name: eval-authoring`.

Run: `grep -c "eval-patterns.md" ~/.claude/skills/eval-authoring/SKILL.md`
Expected: `≥2`.

Run: `grep -c "item 6 N/A" ~/.claude/skills/eval-authoring/SKILL.md`
Expected: `≥1`.

- [ ] **Step 4: Commit (optional)**

```bash
cd ~/.claude && git add skills/eval-authoring/SKILL.md && git commit -m "feat(eval-authoring): add skill orchestrator with GENERATE/AUDIT modes"
```

---

### Task 4: Validate against the three acceptance scenarios

**Files:**
- No new files. Behavioral validation of the skill built in Tasks 1–3.

- [ ] **Step 1: Confirm discoverability**

In a NEW Claude Code session, confirm `eval-authoring` appears in the skill list. If not, check the Task 3 frontmatter.

- [ ] **Step 2: Scenario A — deterministic project, no scored boundaries (N/A)**

Dry-run in a repo with no `docs/domain-boundaries.md` (or one with no scored boundaries).
Expected:
- Step 0 detects no scored boundaries → reports "item 6 N/A" and writes NOTHING.
If it scaffolds evals anyway, fix the applicability check in Step 0 / SKILL.md.

- [ ] **Step 3: Scenario B — summarizer with an output-eval boundary (GENERATE)**

Dry-run where `docs/domain-boundaries.md` has a summarizer boundary scored output with
acceptance intent "faithful, concise, no hallucinated facts".
Expected:
- GENERATE mode; maps the intent to scorers (Faithfulness/Hallucination/GEval).
- Scaffolds `evals/summarizer_eval.py` (output scorers) + `evals/dataset.jsonl` (seed cases).
- Writes `docs/eval-checklist.md` covering `pip install deepeval` + LM-judge API key + CI step.
If it omits the dataset, CI step, or checklist, fix the GENERATE steps.

- [ ] **Step 4: Scenario C — trajectory boundary, observability NOT wired (GENERATE)**

Dry-run where a boundary's eval kind is `trajectory` but no observability (item 7) is wired.
Expected:
- Scaffolds the OUTPUT eval normally.
- For the trajectory eval, CHECKLISTS the item-7 dependency in `docs/eval-checklist.md` and
  does NOT scaffold a broken trajectory eval.
If it scaffolds a trajectory eval without the trace, fix the trajectory dependency check.

- [ ] **Step 5: Record results and fix inline**

For any scenario that diverged, edit `eval-patterns.md` or `SKILL.md`, then re-run that
scenario. Passes when all three behave as specified.

- [ ] **Step 6: Final commit (optional)**

```bash
cd ~/.claude && git add skills/eval-authoring && git commit -m "test(eval-authoring): validate N/A, output, trajectory-dependency scenarios"
```

---

## Self-Review

**1. Spec coverage:**
- Hybrid scaffold + checklist → Task 1 (CI step) + Task 2 (templates) + Task 3 rules. ✓
- DeepEval default, promptfoo fallback → Task 1 scorer catalog + promptfoo section + Task 3 rules. ✓
- Output + trajectory eval kinds, trajectory→observability dependency → Task 1 output-vs-trajectory + Task 3 GENERATE step 5 + Task 4 Scenario C. ✓
- Reads domain-boundaries.md (scored boundaries) → Task 3 GENERATE step 1. ✓
- Scorer catalog (acceptance intent → metric) → Task 1 table + Task 3 GENERATE step 2. ✓
- Dataset seeding + CI step → Task 1 + Task 2 templates + Task 3 step 4/6. ✓
- Produces evals/ + docs/eval-checklist.md → Task 2 + Task 3 steps 4/6. ✓
- Conditional N/A when no scored boundaries → Task 3 Step 0 + Task 4 Scenario A. ✓
- State-aware GENERATE/AUDIT → Task 3 Step 0 + both procedures. ✓
- AUDIT coverage checks → Task 1 audit checks + Task 3 AUDIT step 2. ✓
- Three acceptance scenarios → Task 4. ✓
- Trigger phrase `eval-authoring` → Task 3 frontmatter. ✓

**2. Placeholder scan:** No TBD/TODO as plan gaps. All file contents complete and verbatim. The `{{TOKEN}}` markers (templates), the `run_feature(...)` scaffold call, and `{{TRAJECTORY_ITEMS}}` are intentional fill-in fields, not plan gaps. ✓

**3. Type consistency:** File paths consistent across tasks. Section anchors `SKILL.md` references ("eval-patterns.md" scorer catalog, audit coverage checks) match the headings in `eval-patterns.md` (Task 1). DeepEval metric names (`GEval`, `FaithfulnessMetric`, `HallucinationMetric`, `TaskCompletionMetric`, `ToolCorrectnessMetric`) consistent between Task 1 catalog and Task 2 template. `evals/` output path and `docs/eval-checklist.md` consistent across Task 2, Task 3, and Task 4. ✓
