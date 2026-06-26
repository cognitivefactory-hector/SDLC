# SDLC Preflight Readiness Gate — Design Spec

**Date:** 2026-06-26
**Status:** Approved design, ready for implementation plan
**Trigger word:** `preflight`

---

## 1. Purpose

A personal Claude Code skill that acts as a **readiness gate** for the AI-driven SDLC
described in *The New SDLC with Vibe Coding* (Osmani, Saboo, Kartakis — Google, May 2026)
and the companion guide `observability-and-testing-guide.html` in this repo.

The gate lets the user check themselves — like an engineering leader — that every SDLC
component is in place **before implementation begins**, scaled to the project's stakes.

It is **adaptive** (rigor scales to stakes) and **advisory** (it produces a verdict but
never hard-blocks).

## 2. Scope

This skill is the **capstone** of a five-skill suite that encodes the new SDLC. The other
four are out of scope for *this* spec but their "done" contract is defined here (in the
criteria matrix), because the gate verifies their outputs.

**The suite (global personal skills, `~/.claude/skills/`):**

1. `context-engineering` — produces `AGENTS.md`/`CLAUDE.md` + static-vs-dynamic context map
2. `harness-setup` — configures tools/MCP, sandboxes, guardrails/hooks, **and observability**
3. `nondeterministic-design` — domain boundaries (typed/validated vs scored) + DDD invariants
4. `eval-authoring` — eval dataset + scorers wired into CI
5. `preflight` (this spec) — the readiness gate / through-line

Already covered by existing skills (not rebuilt): requirements/design → `superpowers:brainstorming`;
deterministic tests → `superpowers:test-driven-development`.

## 3. Decisions log (from brainstorming)

| Decision | Choice |
|---|---|
| Deliverable shape | A suite of phase skills (not a single skill or monolithic orchestrator) |
| Suite size | 5 skills — observability folded into `harness-setup` |
| First skill designed | `preflight` (the capstone gate) |
| Gate behavior | **Adaptive + advisory** — verdict GO / GO-WITH-GAPS / NOT-READY, user keeps the call |
| Stakes determination | **Infer, then confirm** with the user |
| Location | **Global** personal skills in `~/.claude/skills/` |
| Skill structure | **Approach B** — thin `SKILL.md` + shared `criteria.md` contract + `report-template.md` |
| Trigger word | **`preflight`** (runners-up: `greenlight`, `checkpoint`) |
| Role in flow | **Bookend through-line** — opens at brainstorm, checks before implementation |
| Mode selection | **State-aware** — no artifacts → OPEN; spec+plan present → CHECK |

## 4. Architecture & files

```
~/.claude/skills/preflight/
├── SKILL.md            # thin orchestration: mode detect → tier → check → report
├── criteria.md         # THE CONTRACT: 5 components × evidence × tier (shared by whole suite)
└── report-template.md  # the readiness-report output shape
```

- `SKILL.md` holds the *procedure* (the how); heavy per-component detail lives in
  `criteria.md` and is loaded only when the gate runs (progressive disclosure).
- `criteria.md` is the **single source of truth** for what "done" means per component per
  tier. The gate reads it to **verify**; the other four suite skills read the same file to
  know what to **produce**. It lives in the `preflight` folder as its canonical home; other
  skills reference it by path (`~/.claude/skills/preflight/criteria.md`).
- `report-template.md` keeps output consistent across runs.

## 5. Tier model

Set by infer-then-confirm. The three tiers make the paper's vibe-coding↔agentic-engineering
spectrum concrete:

- **Prototype** — throwaway, personal, weekend, no real users. Vibe coding is fine.
- **Structured** — feature in an established codebase, internal tool, real but low-stakes.
- **Production** — deployed, multi-user, or touches money / PII / auth. Full rigor.

Tier is inferred from risk signals (auth, payments, PII, public/deployed, multi-user) in the
spec/codebase, proposed to the user, and confirmed or overridden.

## 6. The criteria matrix (the contract)

Legend: `—` not required · `◐` recommended · `●` required · `*` conditional · tag = `[scriptable]`/`[judgment]`

| # | Component — evidence that it's in place | Proto | Struct | Prod | Check type |
|---|---|:--:|:--:|:--:|---|
| 1 | **Spec / intent written** (from brainstorming) | ◐ | ● | ● | `[scriptable]` spec file exists |
| 2 | **Context engineered** — `AGENTS.md`/`CLAUDE.md` + static-vs-dynamic map | ◐ | ● | ● | `[scriptable]` files exist + `[judgment]` map quality |
| 3 | **Harness configured** — tools/MCP + guardrails/hooks for dangerous ops | ◐ | ● | ● | `[scriptable]` `.mcp.json`/hooks exist + `[judgment]` coverage |
| 4 | **Non-deterministic design** `*` — domain boundaries + DDD invariants | — | ● `*` | ● `*` | `[judgment]` boundary doc present & sound |
| 5 | **Tests — deterministic** — TDD approach for the deterministic core | ◐ | ● | ● | `[judgment]` test plan/coverage |
| 6 | **Evals** `*` — **output** (what it built) **+ trajectory** (how it got there) datasets + scorers | — | ◐ `*` | ● `*` | `[scriptable]` eval files/CI step + `[judgment]` coverage |
| 7 | **Observability** — traces + cost/latency + error tracking (+ drift/feedback at prod) | — | ◐ | ● | `[scriptable]` observability MCP/instrumentation present |

**Conditional rows (`*`):** items 4 and 6 apply **only if the project has
non-deterministic / LLM components** (it generates, summarizes, classifies, calls a model,
etc.). The gate detects this first; for a pure-deterministic app these rows drop off and do
not count against the verdict.

**Evals come in two kinds (item 6), and the tooling differs by kind:**

| Kind | Verifies | Captured by | Scored by |
|---|---|---|---|
| **Output eval** | the final artifact ("what it built") | the output itself (already in hand) | DeepEval (G-Eval, faithfulness, hallucination, relevancy), in CI |
| **Trajectory eval** | the path of steps/tool calls ("how it got there") | **observability (OTel → Langfuse)** — item 7 | DeepEval (task-completion, tool-correctness, trajectory) or Langfuse LM-judge |

**Dependency: trajectory evals require item 7.** You cannot score a path you did not
capture, so when the project needs trajectory evals (Production tier), the gate treats
**observability (item 7) as a prerequisite of the trajectory half of item 6**. The MCP
servers (`sentry`, `langfuse`) are the **query interface** onto captured traces — not the
eval engine. DeepEval is the scorer for both kinds; Langfuse/OTel is the source of
trajectory data.

The gate checks item 6 as satisfied only when **both** an output-eval suite and (where
applicable) a trajectory-eval setup are present for the project's non-deterministic
components.

## 7. Two invariant rules

1. **Stakes can pull a single item up.** Any destructive/irreversible operation (deletes,
   payments, prod writes) makes **guardrails (item 3) required** regardless of tier — even
   in a prototype.
2. **Never fabricate a requirement the project doesn't have.** No LLM/fuzzy parts → rows 4
   and 6 are removed, not failed. This is what keeps the gate from nagging.

## 8. Behavior — bookend through-line, state-aware

Preflight is invoked by the word `preflight` (or `/preflight`). It detects project state and
selects a mode:

### OPEN mode (no spec/plan artifacts yet — i.e., brainstorm kickoff)
1. Detect whether the project will have non-deterministic/LLM components (from the user's
   stated idea) to know if rows 4 & 6 will apply.
2. Infer tier from risk signals → propose → user confirms/overrides.
3. Render the applicable checklist as the **agenda** for the session (the flight plan).
4. Invoke `superpowers:brainstorming` to begin the actual design work.
5. Remind the user: *"say `preflight` again before implementing to run the check."*

### CHECK mode (spec + plan present — i.e., before implementation)
1. Re-confirm tier (it may have shifted as scope clarified).
2. For each **applicable** item, gather light evidence per `criteria.md` and mark it
   `✅ present` / `⚠️ recommended-gap` / `❌ required-missing`.
3. Emit the readiness report (Section 9) and **stop**. Advisory — the user decides whether
   to proceed or close gaps.

**Workflow placement:**
`preflight (open) → brainstorming → [context-engineering, harness-setup,
nondeterministic-design, eval-authoring as the tier requires] → writing-plans →
preflight (check) → executing-plans / implementation`

## 9. Report format (`report-template.md`)

```
PREFLIGHT — <project>   ·   Tier: <tier> (confirmed)
─────────────────────────────────────────────────────────
✅ 1 Spec/intent         <evidence>
✅ 2 Context engineered  <evidence>
❌ 3 Harness guardrails  <gap>                    → <specific fix>
⚠️ 6 Evals (*applies)    <gap>                    → <specific fix>
…
─────────────────────────────────────────────────────────
VERDICT:  <GO | GO-WITH-GAPS | NOT-READY>  — <one-line reason>
To reach GO:  <ordered list of specific next actions>
```

Every gap carries a **specific next action**, so the report reads as a to-do list, not a
scold.

**Verdict logic:**
- **GO** — all *required* (`●`) applicable items are `✅`.
- **GO-WITH-GAPS** — all required items `✅`, but one or more *recommended* (`◐`) items are gaps.
- **NOT-READY** — at least one *required* (`●`) applicable item is `❌`.

## 10. Future seam — Approach C (deferred)

Each criteria item is tagged `[scriptable]` or `[judgment]`. Today all checks are
model-performed. Later, a small `preflight-check` helper script can mechanize the
`[scriptable]` checks (file existence, `.mcp.json` contents, CI eval step, guardrail hook
presence) and emit structured JSON that `SKILL.md` already consumes. This is a drop-in
upgrade requiring no redesign. Build it only once the common mechanical patterns stabilize.

## 11. Testing / acceptance scenarios

Skills aren't unit-testable; acceptance = documented dry-runs on three scenarios. The skill
passes when tier inference, conditional rows, and verdict logic behave correctly on all
three:

1. **Throwaway prototype** → minimal checklist; rows 4/6/7 drop; fast **GO**.
2. **Production LLM feature** → rows 4 & 6 active; **NOT-READY** until evals + guardrails exist.
3. **Deterministic CRUD prod app** → rows 4 & 6 **correctly drop off**; observability required;
   verdict reflects only applicable items.

## 12. Out of scope (this spec)

- The other four suite skills (`context-engineering`, `harness-setup`,
  `nondeterministic-design`, `eval-authoring`) — each gets its own spec → plan → build cycle.
  Their "done" contract is defined here in the criteria matrix.
- The `preflight-check` helper script (Section 10) — deferred.
- Hard-blocking enforcement / hooks — explicitly rejected in favor of advisory behavior.

## 13. Related artifacts in this repo

- `observability-and-testing-guide.html` — the companion guide (observability, tests vs
  evals, DDD/BDD containment, reading list).
- `sdlc-preflight-checklist.html` — the human-readable one-page view of this criteria matrix
  (also the printable personal checklist).
- `.mcp.json` + `MCP_SETUP.md` — observability MCP wiring referenced by item 7.
