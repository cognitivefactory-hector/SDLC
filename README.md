# SDLC — Agentic Engineering Skill Suite

A Claude Code skill suite and companion materials implementing the AI-driven software
development life cycle from *The New SDLC with Vibe Coding* (Google, 2026).

## Contents

- `skills/` — the Claude Code personal skills (the suite)
- `docs/superpowers/specs/` & `docs/superpowers/plans/` — design specs and implementation plans
- `observability-and-testing-guide.html` — companion guide (observability layer, tests vs evals, DDD/BDD non-deterministic design)
- `sdlc-preflight-checklist.html` — one-page personal readiness checklist
- `.mcp.json`, `MCP_SETUP.md` — observability MCP wiring (Sentry + Langfuse notes)

## The skill suite

| Skill | Purpose | Trigger |
|---|---|---|
| `preflight` | SDLC readiness gate — bookends the build (opens at brainstorm, checks before implementation) | say `preflight` |
| `context-engineering` | Generates/audits `AGENTS.md`/`CLAUDE.md` + a static-vs-dynamic context map | say `context-engineering` |
| `harness-setup` | Configures Tools/MCP, sandboxes, guardrails (via `hookify`), observability | say `harness-setup` |
| `nondeterministic-design` | DDD/BDD containment — domain boundaries, what's typed vs scored (eval targets) | say `nondeterministic-design` |
| `eval-authoring` | Scaffolds output + trajectory evals (DeepEval) + dataset + CI step from scored boundaries | say `eval-authoring` |

## Walkthrough — a full SDLC pass

The suite is a **through-line**: `preflight` bookends the work, and the four phase skills run
in between, each producing an artifact the next one consumes and the gate checks. Here's a
complete pass on a hypothetical feature — *"summarize a customer email and draft a reply."*

**1. Kick off — say `preflight`.**
With no spec yet, `preflight` opens: it infers the **tier** (this touches customer
PII + is user-facing → *Production*), lays the readiness checklist out as the session agenda,
and hands off to brainstorming. You now know up front what "done" requires.

**2. Brainstorm + plan** (your normal `brainstorming` → `writing-plans` flow) to produce the spec.

**3. `context-engineering`.**
No `AGENTS.md` yet → GENERATE. It interviews you, then writes `AGENTS.md` (canonical
instructions), a `CLAUDE.md` pointer (`@AGENTS.md`), and `docs/context-map.md` sorting each
context type static vs dynamic. Tools & Guardrails are mapped but deferred. → flips
`preflight` **item 2** ✅.

**4. `harness-setup`.**
Reads that context map's deferred rows. Detects the dangerous op (sending email / writing
customer data) → routes to `hookify` to author a guardrail. Writes the Sentry `.mcp.json`
entry. Since it's Production, it checklists Langfuse + OTel tracing in
`docs/harness-checklist.md`. → flips **items 3 & 7**.

**5. `nondeterministic-design`.**
The summarize + draft steps are LLM calls → GENERATE. It applies the 3-layer shell per
feature: typed input contract → the fuzzy *core* (marked **scored**) → output invariants
(e.g. "reply addresses the refund," "no delivery date"). The scored boundaries carry
acceptance intent ("faithful, polite, no hallucinated facts") into `docs/domain-boundaries.md`.
→ flips **item 4**.

**6. `eval-authoring`.**
Reads those scored boundaries → maps the intent to DeepEval scorers (`FaithfulnessMetric`,
`GEval`), seeds `evals/dataset.jsonl`, scaffolds `evals/*_eval.py` + a CI step, and writes
`docs/eval-checklist.md` (`pip install deepeval`, API key, wire CI). Because a trajectory
boundary exists and observability was checklisted (not yet wired), it **checklists** the
trajectory eval's dependency rather than scaffolding a broken one. → flips **item 6**.

**7. Verify — say `preflight` again.**
With spec + plan present, `preflight` now *checks*: it walks the criteria, finds every
artifact in place, and returns a verdict — **GO**, **GO-WITH-GAPS** (e.g. observability still
checklisted), or **NOT-READY**. You decide whether to implement or close gaps first.

**8. Implement** with TDD (the layer-3 invariants from step 5 are your deterministic tests).

> On a throwaway prototype, the same pass is much lighter: `preflight` sets tier *Prototype*,
> the conditional rows (non-deterministic design, evals, observability) drop off, and most
> items are recommended rather than required. The skill is knowing where to draw the line —
> the suite scales the rigor to the stakes for you.

## Install the skills on another computer

```bash
git clone https://github.com/cognitivefactory-hector/SDLC.git
cp -r SDLC/skills/* ~/.claude/skills/
```

Restart Claude Code (skills load on session start). Then say `preflight` to begin a guided
SDLC pass, or invoke any skill by name.

> The `skills/` directory is a vendored copy of all five skills under `~/.claude/skills/`
> (`preflight`, `context-engineering`, `harness-setup`, `nondeterministic-design`, `eval-authoring`).
> After editing a skill on one machine, re-copy it here and push to keep both machines in sync.
