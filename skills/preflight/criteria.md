# Preflight Criteria — the SDLC readiness contract

Single source of truth for what "done" means per SDLC component, per tier.
`preflight` reads this to VERIFY. The other suite skills (context-engineering,
harness-setup, nondeterministic-design, eval-authoring) read it to know what to PRODUCE.

Legend: `—` not required · `◐` recommended · `●` required · `*` conditional

## Tiers
- **Prototype** — throwaway, personal, weekend, no real users. Vibe coding is fine.
- **Structured** — feature in an established codebase, internal tool, real but low-stakes.
- **Production** — deployed, multi-user, or touches money / PII / auth. Full rigor.

## Tier inference signals
- **Production** if ANY: handles money/payments, PII, auth/credentials, public or deployed,
  multi-user.
- **Structured** if: committed to a real/existing codebase but none of the above.
- **Prototype** otherwise (scratch, personal, throwaway).
Always PROPOSE the inferred tier and ask the user to confirm or override.

## Non-determinism detection
Rows 4 and 6 apply ONLY if the project has non-deterministic / LLM parts: it calls a model,
or generates / summarizes / classifies / extracts / converses over natural language.
If it has none, REMOVE rows 4 and 6 — do not fail them.

## Criteria

### 1. Spec / intent written — Proto ◐ · Struct ● · Prod ●
- [scriptable] A spec/design doc exists (e.g. `docs/superpowers/specs/*.md`) OR a clearly
  written intent for the task — not a one-line vibe.

### 2. Context engineered — Proto ◐ · Struct ● · Prod ●
- [scriptable] `AGENTS.md` or `CLAUDE.md` present at repo root.
- [judgment] A static-vs-dynamic context map exists (what's always loaded vs retrieved on
  demand). Produced by the `context-engineering` skill.

### 3. Harness configured — Proto ◐ · Struct ● · Prod ●
- [scriptable] Tools/MCP defined (`.mcp.json` or documented) and a guardrail/hook exists for
  any dangerous op.
- [judgment] Guardrail coverage matches the destructive ops present.
- RULE: if any destructive/irreversible op exists (deletes, payments, prod writes), the
  guardrail is REQUIRED regardless of tier.

### 4. Non-deterministic design * — Proto — · Struct ● · Prod ●
- [judgment] A domain-boundary doc states what is deterministic (typed/validated, DDD
  invariants) vs scored (evals), and where non-determinism is allowed. Produced by the
  `nondeterministic-design` skill.

### 5. Tests — deterministic — Proto ◐ · Struct ● · Prod ●
- [judgment] A TDD approach / test plan covers the deterministic core
  (superpowers:test-driven-development).

### 6. Evals * — Proto — · Struct ◐ · Prod ●
Two kinds:
- **Output eval** (what it built): DeepEval scorers (G-Eval, faithfulness, hallucination,
  relevancy) in CI.
- **Trajectory eval** (how it got there): the trace is captured by observability (item 7),
  then scored by DeepEval (task-completion, tool-correctness) or Langfuse LM-judge.
- [scriptable] An eval dataset + a CI eval step exist.
- [judgment] Coverage spans the non-deterministic components.
- DEPENDENCY: the trajectory half REQUIRES item 7 (you cannot score a path you did not
  capture). Item 6 is satisfied only when output evals AND (where applicable) a trajectory
  setup exist.

### 7. Observability — Proto — · Struct ◐ · Prod ●
- [scriptable] Error tracking (e.g. `sentry` MCP/SDK) and tracing + cost/latency
  (OpenTelemetry → Langfuse) wired.
- [judgment] At Production: drift detection + a production→development feedback loop.

## Invariant rules (never break)
1. Stakes can pull a single item up: any destructive/irreversible op makes item 3 REQUIRED
   at any tier.
2. Never fabricate a requirement the project doesn't have: no LLM/fuzzy parts → rows 4 & 6
   are removed, not failed.
