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

## Install the skills on another computer

```bash
git clone https://github.com/cognitivefactory-hector/SDLC.git
cp -r SDLC/skills/* ~/.claude/skills/
```

Restart Claude Code (skills load on session start). Then say `preflight` to begin a guided
SDLC pass, or invoke any skill by name.

> The `skills/` directory is a vendored copy of `~/.claude/skills/{preflight,context-engineering,harness-setup}`.
> After editing a skill on one machine, re-copy it here and push to keep both machines in sync.
