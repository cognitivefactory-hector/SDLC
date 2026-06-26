---
name: context-engineering
description: Engineer a project's context for AI agents. Use to set up or audit AGENTS.md/CLAUDE.md and a static-vs-dynamic context map, early in a project (after the spec, before harness-setup). Triggers on "context-engineering", "engineer the context", "set up AGENTS.md", "audit my CLAUDE.md", or "context map".
---

# Context-engineering — engineer the agent's context

Sets up (or audits) the four context types this skill owns — Instructions, Knowledge,
Memory, Examples — and produces a static-vs-dynamic context map covering all six. Tools and
Guardrails are deferred to `harness-setup`. Read `context-types.md` for the contract.

## Step 0 — Mode detection (first)
- No `AGENTS.md` and no `CLAUDE.md` at repo root → GENERATE mode.
- Either exists → AUDIT mode.
- Ambiguous → ask the user.

## GENERATE mode
1. Pre-read the repo (languages, frameworks, dir layout, existing docs/tests, package
   manifests) to pre-fill answers.
2. Run the interview from `context-types.md` ("Interview questions"), asking ONLY what you
   could not infer.
3. Sort each item static vs dynamic using the `context-types.md` heuristic.
4. Write, into the TARGET PROJECT:
   - `AGENTS.md` from `templates/AGENTS.md.template`.
   - `CLAUDE.md` containing exactly one line: `@AGENTS.md` (Claude Code native import).
   - `docs/context-map.md` from `templates/context-map.md.template`.
5. Summarize; tell the user `harness-setup` handles Tools & Guardrails next.

## AUDIT mode
1. Read existing `AGENTS.md`/`CLAUDE.md` and any `docs/context-map.md`.
2. Score each of the six types PRESENT / MISSING / MISPLACED per the `context-types.md`
   audit checklist.
3. Propose specific diffs — advisory; apply on the user's confirm.
4. Produce or refresh `docs/context-map.md`.

## Rules
- Write content only for types 1–4; never configure Tools/Guardrails (types 5–6).
- The `CLAUDE.md` pointer is exactly `@AGENTS.md` — do not duplicate AGENTS.md content.
- When unsure about placement, default to dynamic.

## Reference files
- `context-types.md` — the six types, ownership, heuristic, interview, audit checklist.
- `templates/AGENTS.md.template`, `templates/context-map.md.template` — fill-in starting points.
