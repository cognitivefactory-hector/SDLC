# Nondeterministic-Design Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `nondeterministic-design` personal skill — a state-aware designer that produces `docs/domain-boundaries.md` classifying what's deterministic (typed/validated, DDD invariants) vs scored (evals) per non-deterministic feature.

**Architecture:** Three markdown files in `~/.claude/skills/nondeterministic-design/` (same Approach-B pattern as the prior suite skills): a thin `SKILL.md` orchestrator, a shared `boundary-patterns.md` contract, and a doc template. Doc only — no code, no eval datasets.

**Tech Stack:** Claude Code skills (Markdown + YAML frontmatter). No build step. Verification is behavioral dry-runs.

## Global Constraints

- Install location: `~/.claude/skills/nondeterministic-design/`. Copy content verbatim.
- Produces a DESIGN DOC ONLY (`docs/domain-boundaries.md`) — never code, eval datasets, or BDD/Gherkin scenarios.
- ND↔EA boundary: ND captures acceptance-criteria INTENT per scored boundary; `eval-authoring` implements datasets/scorers/CI.
- Conditional: if the project has NO non-deterministic/LLM parts, report "item 4 N/A" and write nothing.
- 3-layer shell model: (1) deterministic input contract, (2) non-deterministic core (scored), (3) deterministic output validation (invariants).
- Heuristic: when a typed invariant CAN capture it, it is deterministic, not scored. Only score what genuinely needs judgment.
- Source of truth: `docs/superpowers/specs/2026-06-26-nondeterministic-design-skill-design.md`.
- Commits OPTIONAL: only if `~/.claude` is version-controlled; otherwise skip.
- Fence caution: `SKILL.md` starts with `---` on line 1; the reference and template start with their `#` heading on line 1. NO outer display fence around any file.

---

## File Structure

| File | Responsibility |
|---|---|
| `~/.claude/skills/nondeterministic-design/boundary-patterns.md` | Contract: 3-layer shell model, deterministic-vs-scored heuristic, invariant patterns, audit failure modes, non-determinism detection. Built first. |
| `~/.claude/skills/nondeterministic-design/templates/domain-boundaries.md.template` | The produced design doc (per-feature shell + scored boundary + invariants). |
| `~/.claude/skills/nondeterministic-design/SKILL.md` | Orchestrator: applicability/mode detect → find touchpoints → classify → write doc. |

Build order: `boundary-patterns.md` → template → `SKILL.md` → validation.

---

### Task 1: Create the contract (`boundary-patterns.md`)

**Files:**
- Create: `~/.claude/skills/nondeterministic-design/boundary-patterns.md`

**Interfaces:**
- Produces the contract referenced by `SKILL.md` (Task 3). Section anchors relied on: "The 3-layer shell model", "Deterministic-vs-scored heuristic", "Invariant patterns", "Audit failure modes", "Non-determinism detection".

- [ ] **Step 1: Define done**

Complete when the file contains the 3-layer shell table, the deterministic-vs-scored heuristic, the invariant patterns list, the per-scored-boundary capture (acceptance intent + eval kind), the audit failure modes, and the non-determinism detection rule.

- [ ] **Step 2: Create the directory**

```bash
mkdir -p ~/.claude/skills/nondeterministic-design/templates
```

- [ ] **Step 3: Write `boundary-patterns.md` with this exact content**

```markdown
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
```

- [ ] **Step 4: Verify**

Run: `grep -cE "shell model|Deterministic-vs-scored|Invariant patterns|Audit failure modes|Non-determinism detection" ~/.claude/skills/nondeterministic-design/boundary-patterns.md`
Expected: `5`.

Run: `grep -cE "OVER-SCORING|UNDER-SCORING|COVERAGE" ~/.claude/skills/nondeterministic-design/boundary-patterns.md`
Expected: `3`.

- [ ] **Step 5: Commit (optional — only if `~/.claude` is version-controlled)**

```bash
cd ~/.claude && git add skills/nondeterministic-design/boundary-patterns.md && git commit -m "feat(nondeterministic-design): add boundary-patterns contract"
```

---

### Task 2: Create the doc template

**Files:**
- Create: `~/.claude/skills/nondeterministic-design/templates/domain-boundaries.md.template`

**Interfaces:**
- Consumes nothing. Produces the doc `SKILL.md` (Task 3) writes into target projects as `docs/domain-boundaries.md`. The `{{TOKEN}}` markers are intentional fill-in fields.

- [ ] **Step 1: Define done**

Complete when the template exists with its `{{TOKEN}}` fields, the per-feature 3-layer table, the scored-boundary acceptance section, and the layer-3 invariants section.

- [ ] **Step 2: Write `templates/domain-boundaries.md.template` with this exact content** (starts with `#` on line 1 — NO outer code fence)

```markdown
# Domain Boundaries — {{PROJECT_NAME}}

What is deterministic (typed/validated) vs scored (evals), and where non-determinism is
allowed. Produced by nondeterministic-design; feeds eval-authoring (scored items) and TDD
(layer-3 invariants).

## Feature: {{FEATURE_NAME}}

| Layer | Decision | Deterministic / Scored |
|---|---|---|
| 1. Input contract | {{INPUT_CONTRACT}} | Deterministic |
| 2. Core (generation) | {{CORE_DESCRIPTION}} | Scored |
| 3. Output validation | {{OUTPUT_INVARIANTS}} | Deterministic |

### Scored boundary — acceptance criteria (for eval-authoring)
- What good looks like: {{ACCEPTANCE_INTENT}}
- Eval kind: {{EVAL_KIND}}   (output | trajectory; trajectory needs observability / item 7)

### Layer-3 invariants (for TDD)
{{INVARIANTS_LIST}}

---
(Repeat the "Feature" block per non-deterministic touchpoint.)
```

- [ ] **Step 3: Verify**

Run: `grep -cE "Input contract|Core \(generation\)|Output validation|acceptance criteria|invariants" ~/.claude/skills/nondeterministic-design/templates/domain-boundaries.md.template`
Expected: `≥5`.

Run: `grep -c "{{" ~/.claude/skills/nondeterministic-design/templates/domain-boundaries.md.template`
Expected: `≥6` (fill-in tokens present).

- [ ] **Step 4: Commit (optional)**

```bash
cd ~/.claude && git add skills/nondeterministic-design/templates && git commit -m "feat(nondeterministic-design): add domain-boundaries template"
```

---

### Task 3: Create the skill orchestrator (`SKILL.md`)

**Files:**
- Create: `~/.claude/skills/nondeterministic-design/SKILL.md`

**Interfaces:**
- Consumes `boundary-patterns.md` (Task 1) and the template (Task 2) by relative path.
- Produces the activatable skill. Frontmatter `name: nondeterministic-design`; `description` contains `nondeterministic-design`.

- [ ] **Step 1: Define done**

Complete when `SKILL.md` has valid frontmatter, the applicability/N-A + mode-detection step, GENERATE and AUDIT procedures, and the rules (doc only; push boundary inward; never invent fuzzy boundaries).

- [ ] **Step 2: Write `SKILL.md` with this exact content** (file starts with `---` on line 1 — NO outer code fence)

```markdown
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
4. Per scored boundary, capture acceptance-criteria intent + eval kind (output/trajectory).
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
```

- [ ] **Step 3: Verify**

Run: `head -2 ~/.claude/skills/nondeterministic-design/SKILL.md`
Expected: line 1 `---`, line 2 `name: nondeterministic-design`.

Run: `grep -c "boundary-patterns.md" ~/.claude/skills/nondeterministic-design/SKILL.md`
Expected: `≥3`.

Run: `grep -c "N/A" ~/.claude/skills/nondeterministic-design/SKILL.md`
Expected: `≥2`.

- [ ] **Step 4: Commit (optional)**

```bash
cd ~/.claude && git add skills/nondeterministic-design/SKILL.md && git commit -m "feat(nondeterministic-design): add skill orchestrator with GENERATE/AUDIT modes"
```

---

### Task 4: Validate against the three acceptance scenarios

**Files:**
- No new files. Behavioral validation of the skill built in Tasks 1–3.

- [ ] **Step 1: Confirm discoverability**

In a NEW Claude Code session, confirm `nondeterministic-design` appears in the skill list. If not, check the Task 3 frontmatter.

- [ ] **Step 2: Scenario A — deterministic CRUD app, no LLM (N/A)**

Dry-run in a repo with no LLM/fuzzy parts.
Expected:
- Step 0 detects no non-deterministic touchpoints → reports "item 4 N/A" and writes NOTHING.
If it invents fuzzy boundaries or writes a doc, fix the non-determinism detection / Step 0.

- [ ] **Step 3: Scenario B — LLM summarizer feature (GENERATE)**

Dry-run in a repo/spec with a feature that summarizes user text via an LLM.
Expected:
- GENERATE mode; identifies the summarize touchpoint.
- 3-layer shell: layer 1 = typed source text (deterministic); layer 2 = the summary (scored);
  layer 3 = invariants (e.g. length bound, no banned content).
- Scored boundary captures acceptance intent ("faithful, concise, no hallucinated facts") +
  eval kind (output).
- Writes `docs/domain-boundaries.md` with a Feature block.
If it misses the 3-layer classification or omits acceptance intent, fix the GENERATE steps.

- [ ] **Step 4: Scenario C — over-scoring audit (AUDIT)**

Dry-run in a repo whose `docs/domain-boundaries.md` marks "output is valid JSON" as scored.
Expected:
- AUDIT mode; flags "output is valid JSON" as OVER-SCORING (a schema check makes it
  deterministic) and proposes moving it to a layer-3 invariant.
If it does not catch the over-scoring, fix the audit failure modes in `boundary-patterns.md`.

- [ ] **Step 5: Record results and fix inline**

For any scenario that diverged, edit `boundary-patterns.md` or `SKILL.md`, then re-run that
scenario. Passes when all three behave as specified.

- [ ] **Step 6: Final commit (optional)**

```bash
cd ~/.claude && git add skills/nondeterministic-design && git commit -m "test(nondeterministic-design): validate N/A, summarizer, over-scoring scenarios"
```

---

## Self-Review

**1. Spec coverage:**
- Doc-only output (`docs/domain-boundaries.md`) → Task 2 template + Task 3 GENERATE step 5 + Rules. ✓
- 3-layer shell model → Task 1 table + Task 3 GENERATE step 2. ✓
- Deterministic-vs-scored heuristic (push inward) → Task 1 heuristic + Task 3 rules. ✓
- Invariant patterns (layer 3) → Task 1 list + template layer-3 section. ✓
- Per scored boundary: acceptance intent + eval kind → Task 1 + Task 2 template + Task 3 step 4. ✓
- ND↔EA boundary (intent only, no datasets/scenarios) → Task 1 header + Task 3 rules. ✓
- Conditional N/A when no non-deterministic parts → Task 1 detection + Task 3 Step 0 + Task 4 Scenario A. ✓
- State-aware GENERATE/AUDIT → Task 3 Step 0 + both procedures. ✓
- AUDIT over/under-scoring + coverage → Task 1 failure modes + Task 3 AUDIT step 2 + Task 4 Scenario C. ✓
- Three acceptance scenarios → Task 4. ✓
- Trigger phrase `nondeterministic-design` → Task 3 frontmatter. ✓

**2. Placeholder scan:** No TBD/TODO. All file contents complete and verbatim. The `{{TOKEN}}` markers in the template are intentional fill-in fields, not plan gaps. ✓

**3. Type consistency:** File paths consistent across tasks. Section anchors `SKILL.md` references ("Non-determinism detection", the shell model, audit failure modes) match the headings defined in `boundary-patterns.md` (Task 1). Layer names (input contract / core / output validation) and terms (deterministic/scored, over-scoring/under-scoring, output/trajectory) identical across all files. ✓
