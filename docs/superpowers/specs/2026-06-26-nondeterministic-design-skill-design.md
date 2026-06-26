# Nondeterministic-Design Skill — Design Spec

**Date:** 2026-06-26
**Status:** Approved design, ready for implementation plan
**Suite:** SDLC skill suite (skill #4 of 5)

---

## 1. Purpose

A personal Claude Code skill that designs the **containment boundaries** around a project's
non-deterministic (LLM/fuzzy) parts, per *The New SDLC with Vibe Coding* and the companion
`observability-and-testing-guide.html` (§3).

It produces what `preflight` gate item 4 checks: a domain-boundary doc stating what is
deterministic (typed/validated, DDD invariants) vs scored (evals), and where non-determinism
is allowed. It runs **before implementation** and is **conditional** — it only applies when
the project has non-deterministic parts.

## 2. Scope

- **Produces a design doc only** (`docs/domain-boundaries.md`) — no code scaffolding. The
  deterministic invariants become TDD targets at implementation; the scored boundaries
  become `eval-authoring`'s input.
- **Captures rubric intent** ("what good looks like") per scored boundary, for
  `eval-authoring` — but does NOT write datasets, scorers, or BDD/Gherkin scenarios (that is
  `eval-authoring`'s job).
- Conditional: if the project has no non-deterministic parts, the skill reports item 4 N/A
  and writes nothing.

## 3. Decisions log (from brainstorming)

| Decision | Choice |
|---|---|
| Primary output | **Boundary design doc only** (`docs/domain-boundaries.md`) |
| ND↔EA boundary | ND designs boundaries + **acceptance-criteria intent**; eval-authoring implements datasets/scorers/CI |
| Operational approach | Approach A — analysis-driven + shared reference + template, with light spec/code reading to find touchpoints |
| State model | State-aware GENERATE / AUDIT |
| Skill structure | thin `SKILL.md` + shared `boundary-patterns.md` + template |

## 4. Architecture & files

```
~/.claude/skills/nondeterministic-design/
├── SKILL.md                         # orchestrator: mode detect → find touchpoints → classify → write doc
├── boundary-patterns.md             # keystone reference: shell model, deterministic-vs-scored heuristic, invariants
└── templates/
    └── domain-boundaries.md.template # the produced design doc
```

Writes into the **target project**: only `docs/domain-boundaries.md`. Reads the spec and any
existing code to locate the LLM/fuzzy touchpoints.

## 5. The `boundary-patterns.md` reference (content contract)

### The 3-layer shell model (applied per non-deterministic feature)

| Layer | What it is | Guarded by |
|---|---|---|
| 1. Deterministic input contract | typed inputs, schema validation, ubiquitous language | types + tests |
| 2. Non-deterministic core | the LLM/fuzzy generation | **scored (evals)** |
| 3. Deterministic output validation | domain invariants reject bad output | types + invariants |

### Deterministic-vs-scored heuristic
- **Deterministic** ← the assertion is expressible as `==`, a typed invariant, or a schema
  check. Guarded by types + tests.
- **Scored** ← it would take a human saying "that's good enough" (quality, relevance, tone,
  faithfulness, or *did it take a sane path*). Guarded by evals.
- **Rule:** push the deterministic boundary inward — validate everything crossing it; only
  score what genuinely needs judgment. When a typed invariant CAN capture it, it is
  deterministic, not scored.

### Invariant patterns (layer-3 wall)
- Type/schema conformance (parses into a typed object)
- Range/domain constraints (e.g. `quantity > 0`, currency ∈ allowed set)
- Cross-field consistency (e.g. `total == Σ line items`)
- Referential validity (IDs exist), required/non-empty, enum membership

### Per scored boundary, capture (for eval-authoring)
- Acceptance-criteria intent — "what good looks like," in plain language (the rubric).
- Eval kind: **output** (the artifact) vs **trajectory** (the path). Trajectory flags the
  item 6→7 dependency (trajectory evals need observability).

## 6. Behavior — state-aware modes

Invoked by intent (e.g. "design the boundaries", "nondeterministic-design", "what's
deterministic vs scored").

### Mode detection (first)
- Project has NO non-deterministic/LLM parts → report **item 4 N/A**, write nothing, stop.
- No `docs/domain-boundaries.md` → **GENERATE**.
- Exists → **AUDIT**.

### GENERATE mode
1. Find the non-deterministic touchpoints — scan the spec/code for LLM calls or
   generate/summarize/classify/extract/converse-over-NL.
2. Per touchpoint, apply the shell model: define the layer-1 input contract; name the layer-2
   core (mark scored); list the layer-3 output invariants.
3. Classify each sub-decision deterministic vs scored via the heuristic.
4. Per scored boundary, capture acceptance-criteria intent + eval kind.
5. Write `docs/domain-boundaries.md` from `templates/domain-boundaries.md.template`.
6. Summarize — note `eval-authoring` turns scored boundaries into evals; deterministic
   invariants become TDD targets.

### AUDIT mode
1. Read existing `docs/domain-boundaries.md` + the touchpoints in spec/code.
2. Check three failure modes:
   - **Coverage:** every non-deterministic touchpoint has a boundary entry.
   - **Over-scoring:** anything marked scored that a typed invariant could make deterministic.
   - **Under-scoring:** anything marked deterministic that actually needs judgment.
   - Plus: each core has layer-3 output invariants.
3. Propose diffs — advisory.
4. Refresh `docs/domain-boundaries.md`.

## 7. Integration

- **→ `preflight`:** produces item 4's doc; running it flips item 4 to ✅ (or N/A).
- **→ `eval-authoring`:** scored boundaries + acceptance-criteria intent are its input.
- **→ TDD/implementation:** layer-3 invariants become test targets.
- **Conditional:** only applies when non-deterministic parts exist (mirrors item 4 `*`).
- **Flow placement:** `… → context-engineering → harness-setup → nondeterministic-design →
  eval-authoring → implementation`.

## 8. Testing / acceptance scenarios

Skills aren't unit-testable; acceptance = documented dry-runs:
1. **Deterministic CRUD app, no LLM** → detects no non-deterministic parts → reports item 4
   N/A, writes nothing (does NOT invent fuzzy boundaries).
2. **LLM summarizer feature** → GENERATE: identifies the summarize touchpoint → 3-layer shell
   (layer 1 = typed source text; layer 2 = the summary, scored; layer 3 = invariants like
   length bound + no banned content); scored boundary captures acceptance intent ("faithful,
   concise, no hallucinated facts") + eval kind (output). Writes `docs/domain-boundaries.md`.
3. **Over-scoring audit** → an existing doc marks "output is valid JSON" as scored → AUDIT
   flags it over-scored (a schema check makes it deterministic) and proposes a layer-3
   invariant.

Passes when: no-LLM → N/A; LLM feature → correct 3-layer classification with a scored
boundary carrying acceptance intent; AUDIT catches over/under-scoring.

## 9. Out of scope

- Code scaffolding (typed models, BDD scenario stubs) — implementation/TDD territory.
- Eval datasets, scorers, BDD/Gherkin scenarios, CI wiring — owned by `eval-authoring`.
- The last suite skill (`eval-authoring`) — its own spec → plan → build.

## 10. Related artifacts

- `~/.claude/skills/preflight/` — the gate this skill feeds (item 4).
- `observability-and-testing-guide.html` §3 — the shell model / DDD-BDD source.
- `~/.claude/skills/eval-authoring/` *(planned)* — consumes the scored boundaries.
