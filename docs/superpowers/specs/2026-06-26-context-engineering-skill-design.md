# Context-Engineering Skill — Design Spec

**Date:** 2026-06-26
**Status:** Approved design, ready for implementation plan
**Suite:** SDLC skill suite (skill #2 of 5)

---

## 1. Purpose

A personal Claude Code skill that engineers a project's **context** — the rich, structured
information an agent needs to do good work — per *The New SDLC with Vibe Coding* and the
companion `observability-and-testing-guide.html`.

It produces the two artifacts the `preflight` gate checks for item 2:
1. `AGENTS.md` (canonical instructions) + a `CLAUDE.md` pointer.
2. A versioned static-vs-dynamic **context map** at `docs/context-map.md`.

## 2. Scope

The paper defines six context types. This skill **owns four and defers two** (to
`harness-setup`), but **maps all six** for the holistic view.

| # | Context type | Owner |
|---|---|---|
| 1 | Instructions — role, goals, boundaries | **context-engineering** |
| 2 | Knowledge — docs, architecture, domain data | **context-engineering** |
| 3 | Memory — session (short) + persistent (long) | **context-engineering** |
| 4 | Examples — few-shot demos, reference patterns | **context-engineering** |
| 5 | Tools — APIs, scripts, services | `harness-setup` (mapped only) |
| 6 | Guardrails — hard constraints, safety | `harness-setup` (mapped only) |

It writes only into the target project (`AGENTS.md`, `CLAUDE.md`, `docs/context-map.md`).
It never configures Tools/Guardrails — it only references them in the map.

## 3. Decisions log (from brainstorming)

| Decision | Choice |
|---|---|
| Core behavior | **State-aware**: GENERATE if no rule file exists; AUDIT if one does |
| Context scope | **Own 4, map 6** (defer Tools & Guardrails to harness-setup) |
| Rule file | **`AGENTS.md` canonical + `CLAUDE.md` pointer** (portable + native) |
| Pointer mechanism | `CLAUDE.md` contains `@AGENTS.md` (Claude Code native import) |
| Operational approach | **Interview-driven + templates + shared reference**, with light codebase pre-read to pre-fill |
| Map location | `docs/context-map.md` (versioned, first-class, per the paper) |
| Skill structure | Thin `SKILL.md` + shared `context-types.md` + `templates/` |

## 4. Architecture & files

```
~/.claude/skills/context-engineering/
├── SKILL.md                     # orchestrator: mode detect → interview/audit → generate
├── context-types.md             # shared reference: 6 types, ownership, static/dynamic heuristic
└── templates/
    ├── AGENTS.md.template        # canonical instructions file
    └── context-map.md.template   # the static-vs-dynamic map
```

- `SKILL.md` is thin procedure; the type definitions and heuristics live in `context-types.md`
  (loaded when the skill runs — progressive disclosure).
- `context-types.md` is the keystone, parallel to `preflight`'s `criteria.md`. Both the
  interview questions and the audit checklist derive from it.
- Templates are opinionated starting points written into the **target project**, not the
  skill dir.

## 5. The `context-types.md` reference (content contract)

Defines, for each of the six types: what it is, owner, default static/dynamic placement,
and where the skill writes it.

| # | Type | Owner | Default placement | Lands in |
|---|---|---|---|---|
| 1 | Instructions | CE | **Static** | `AGENTS.md` |
| 2 | Knowledge | CE | **Dynamic** (refs) | pointers in `AGENTS.md` → `docs/` |
| 3 | Memory | CE | **Mixed** | persistent facts → `AGENTS.md`; strategy in map |
| 4 | Examples | CE | **Dynamic** | `examples/` refs in map |
| 5 | Tools | harness-setup | Dynamic | mapped only, deferred |
| 6 | Guardrails | harness-setup | Static (core) | mapped only, deferred |

**Static-vs-dynamic heuristic** (applied to every item):
- **Static** (always loaded, high token cost) ← small + stable + relevant to *every*
  interaction + defines who the agent is.
- **Dynamic** (loaded on demand, low per-turn cost) ← large *or* situational *or*
  task-specific *or* changes often.
- Too much static wastes tokens and dilutes signal; too little and the agent forgets
  critical rules. **When unsure, default to dynamic; promote to static only if needed every
  time.**

## 6. Behavior — state-aware modes

Invoked by intent (e.g. "engineer the context", "set up AGENTS.md", "context-engineering").

### Mode detection (first)
- No `AGENTS.md` and no `CLAUDE.md` at repo root → **GENERATE**.
- Either exists → **AUDIT**.
- Ambiguous → ask the user.

### GENERATE mode
1. **Pre-read the repo** — languages, frameworks, dir layout, existing docs/tests, package
   manifests — to pre-fill answers.
2. **Short interview**, asking only what could not be inferred, across the 4 owned types:
   - Instructions: purpose, role, hard boundaries/don'ts, key conventions.
   - Knowledge: where important docs/architecture live (or "none yet").
   - Memory: persistent facts that must survive sessions; scratch/session strategy.
   - Examples: reference patterns/files to point at.
   - Stability: what's stable vs what changes often (drives static/dynamic sorting).
3. **Sort** each item static/dynamic via the `context-types.md` heuristic.
4. **Write** `AGENTS.md` (from template), the `CLAUDE.md` pointer (`@AGENTS.md`), and
   `docs/context-map.md` (from template).
5. **Summarize**; remind the user that `harness-setup` handles Tools & Guardrails next.

### AUDIT mode
1. Read existing `AGENTS.md`/`CLAUDE.md` and any context map.
2. Score against the six-type checklist: which types are present / missing / **misplaced**
   (e.g., a large doc jammed into always-loaded static).
3. Propose specific diffs — advisory; apply on the user's confirm.
4. Produce or refresh `docs/context-map.md`.

### The `CLAUDE.md` pointer
A one-line file containing `@AGENTS.md` — Claude Code's native import — so the canonical
instructions load automatically with no duplication or symlink fragility.

## 7. Integration

- **→ `preflight`:** produces exactly what item 2 checks; running CE flips item 2 to ✅.
- **→ `harness-setup`:** CE defers Tools & Guardrails but maps them; that map is the input
  `harness-setup` reads. No overlap.
- **Flow placement:** `preflight (open) → brainstorming → context-engineering →
  harness-setup → …`. CE runs early, right after the spec exists.
- **Reference consistency:** `context-types.md` and `preflight`'s `criteria.md` agree —
  item 2's "static-vs-dynamic context map" is precisely CE's output artifact.

## 8. Testing / acceptance scenarios

Skills aren't unit-testable; acceptance = documented dry-runs:
1. **Greenfield empty repo** → GENERATE: interview runs; produces `AGENTS.md` +
   `CLAUDE.md` (`@AGENTS.md`) + `docs/context-map.md`; map sorts items static/dynamic;
   Tools & Guardrails marked *deferred*.
2. **Existing repo with a `CLAUDE.md`** → AUDIT: reads it, flags missing types (e.g., no
   Knowledge refs, no map), proposes diffs, writes `context-map.md`.
3. **Misplaced-context repo** (a giant API doc pasted into static `CLAUDE.md`) → AUDIT flags
   it as "should be dynamic" and proposes moving it to a referenced doc.

Passes when GENERATE produces all three artifacts correctly, AUDIT detects missing/misplaced
types, and Tools/Guardrails are deferred (never generated here).

## 9. Out of scope

- Tools & Guardrails configuration — owned by `harness-setup`.
- RAG pipeline implementation, memory backends — CE defines the *strategy/map*, not the
  infrastructure.
- The other three suite skills (`harness-setup`, `nondeterministic-design`,
  `eval-authoring`) — each gets its own spec → plan → build.

## 10. Related artifacts

- `~/.claude/skills/preflight/` — the gate this skill feeds (item 2).
- `docs/superpowers/specs/2026-06-26-sdlc-preflight-readiness-gate-design.md` — the suite
  capstone spec.
- `observability-and-testing-guide.html` — companion guide (context engineering section).
