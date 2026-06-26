# Context Types — the context-engineering contract

Shared reference for the `context-engineering` skill. Defines the six context types from
"The New SDLC with Vibe Coding", which four this skill OWNS vs the two it DEFERS to
`harness-setup`, and the heuristic for sorting each into static vs dynamic context.

The interview (GENERATE mode) and the audit checklist (AUDIT mode) both derive from this file.

## The six context types

| # | Type | Owner | Default placement | Lands in |
|---|---|---|---|---|
| 1 | Instructions — role, goals, operational boundaries | context-engineering | Static | AGENTS.md |
| 2 | Knowledge — docs, architecture diagrams, domain data | context-engineering | Dynamic (refs) | pointers in AGENTS.md → docs/ |
| 3 | Memory — session (short-term) + persistent (long-term) | context-engineering | Mixed | persistent facts → AGENTS.md; strategy → map |
| 4 | Examples — few-shot demos, codebase reference patterns | context-engineering | Dynamic | examples/ refs in map |
| 5 | Tools — APIs, scripts, external services | harness-setup | Dynamic | mapped only, DEFERRED |
| 6 | Guardrails — hard constraints, formatting, safety | harness-setup | Static (core) | mapped only, DEFERRED |

This skill writes content ONLY for types 1–4. Types 5 & 6 appear in the map with a
"deferred to harness-setup" note; this skill never configures them.

## Static vs dynamic heuristic

- **Static** (always loaded, every interaction; high token cost): small + stable + relevant
  to EVERY interaction + defines who the agent is. Examples: system instructions, rule files
  (AGENTS.md), core persona, core guardrails.
- **Dynamic** (loaded on demand, per task; low per-turn cost): large OR situational OR
  task-specific OR changes often. Examples: skill instructions triggered by task match, tool
  results, RAG documents, windowed session history.

Rule: too much static wastes tokens and dilutes signal; too little and the agent forgets
critical rules. WHEN UNSURE, default to dynamic; promote to static only if it is needed in
every interaction.

## Interview questions (GENERATE mode) — owned types only
- Instructions: project purpose? the agent's role? hard boundaries / "never do" rules? key
  conventions (style, naming, structure)?
- Knowledge: where do the important docs / architecture references live? (or "none yet")
- Memory: what facts must persist across sessions? what is scratch/session-only?
- Examples: any reference files or patterns the agent should imitate?
- Stability: which of the above is stable vs changes often? (drives static/dynamic)

## Audit checklist (AUDIT mode)
For each of the six types, mark: PRESENT / MISSING / MISPLACED.
- MISSING: the type has no representation (e.g., no Knowledge refs, no persistent Memory).
- MISPLACED: a large/situational item sits in static context (should be dynamic), or a
  critical always-needed rule sits only in a dynamic/rarely-loaded place (should be static).
Always (re)produce `docs/context-map.md`.
