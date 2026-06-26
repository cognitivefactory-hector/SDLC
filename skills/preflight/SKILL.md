---
name: preflight
description: SDLC readiness gate and through-line for AI-driven development. Use at the VERY START of any non-trivial build/feature/project (kickoff, before or instead of brainstorming) to set the stakes tier and lay out the readiness checklist as the session agenda, AND AGAIN before implementation to verify everything is in place. Triggers on "preflight", "/preflight", "run the gate", "are we ready to build/implement", or starting any substantial coding project.
---

# Preflight — SDLC readiness gate

Adaptive, advisory readiness gate. Scales rigor to project stakes; produces a verdict
(GO / GO-WITH-GAPS / NOT-READY) but NEVER hard-blocks and never edits code. It bookends the
build flow. Read `criteria.md` (the contract) when running either mode.

## Step 0 — Mode detection (do this first)
Inspect project state:
- If NO spec/plan artifacts exist yet (`docs/superpowers/specs/` and `docs/superpowers/plans/`
  are absent or empty) → **OPEN mode**.
- If a spec AND a plan exist → **CHECK mode**.
- If ambiguous, ask the user which mode they want.

## OPEN mode — brainstorm kickoff (set the flight plan)
1. From the user's stated idea, decide whether the project will have non-deterministic/LLM
   parts (see `criteria.md` → "Non-determinism detection"). This decides whether rows 4 & 6
   will apply.
2. Infer the tier from `criteria.md` → "Tier inference signals". PROPOSE it and ask the user
   to confirm or override.
3. Render the APPLICABLE checklist (drop non-applicable conditional rows) as the agenda for
   the session, marking each item required (`●`) or recommended (`◐`) at the confirmed tier.
4. Invoke the `superpowers:brainstorming` skill to begin the design work.
5. Tell the user: "Say `preflight` again before implementing to run the check."

## CHECK mode — before implementation (the verdict)
1. Re-confirm the tier (scope may have shifted) using `criteria.md`.
2. Detect non-determinism; include rows 4 & 6 only if applicable.
3. For each applicable item, gather evidence per `criteria.md` and mark `✅` / `⚠️` / `❌`:
   - `[scriptable]` checks: inspect files (`AGENTS.md`/`CLAUDE.md`, `.mcp.json`, CI config,
     eval dataset, hooks).
   - `[judgment]` checks: read the relevant doc/code and assess.
4. Render the report using `report-template.md`, with a specific fix after `→` for every gap.
5. STOP. This is advisory — the user decides whether to proceed or close gaps.

## Invariant rules (never break)
- Guardrails (item 3) are REQUIRED if any destructive/irreversible op exists, at ANY tier.
- Never fabricate a requirement the project doesn't have — no LLM parts → no rows 4 & 6.
- Trajectory evals (item 6) require observability (item 7); flag the dependency in the report.

## Reference files
- `criteria.md` — the per-component, per-tier contract (read for both modes).
- `report-template.md` — the CHECK-mode output shape and verdict logic.
