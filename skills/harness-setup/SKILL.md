---
name: harness-setup
description: Configure a project's agent harness — Tools/MCP, sandboxes/permissions, guardrails/hooks, and observability. Use after context-engineering, before implementation. Writes safe config and checklists the credential/install steps. Triggers on "harness-setup", "set up the harness", "wire up MCP", "configure guardrails", or "set up observability".
---

# Harness-setup — configure the agent harness

Configures four harness components — Tools/MCP, Sandboxes/execution, Guardrails/Hooks,
Observability — using a hybrid model: WRITE the safe deterministic config, CHECKLIST the
credential/install steps. Guardrail authoring is delegated to the `hookify` skill.
Orchestration logic is out of scope; Instructions are owned by `context-engineering`.
Read `harness-components.md` for the contract.

## Step 0 — Mode detection (first)
- No harness artifacts (`.mcp.json` absent, no hooks in `.claude/settings.json`, no
  `docs/harness-checklist.md`) → GENERATE mode.
- Any present → AUDIT mode.
- Ambiguous → ask the user.

## GENERATE mode
1. Read inputs: `context-engineering`'s `docs/context-map.md` (deferred Tools & Guardrails
   rows) and the project spec, to learn what tools and ops the project needs.
2. Confirm tier (infer + confirm via the same signals preflight uses, or reuse a set tier).
3. Scan the spec/codebase for dangerous ops using the `harness-components.md` dangerous-op map.
4. For each component, do the write-safe action and collect checklist items per
   `harness-components.md`:
   - Tools/MCP: write `.mcp.json` entries for no-secret/OAuth servers (e.g. the Sentry
     snippet); checklist credential ones.
   - Sandboxes: write `.claude/settings.json` allow/deny permissions.
   - Guardrails: for each dangerous op, invoke the `hookify` skill to author the hook, then
     verify coverage.
   - Observability (per tier): write the Sentry entry + enable the plugin; checklist Langfuse
     deploy / OTel SDK / drift+feedback.
5. Write `docs/harness-checklist.md` from `templates/harness-checklist.md.template`.
6. Summarize and point the user to the checklist.

## AUDIT mode
1. Read existing `.mcp.json`, `.claude/settings.json` (hooks/permissions), and any
   `docs/harness-checklist.md`.
2. Scan for dangerous ops; verify each has a covering guardrail (the RULE).
3. Score each component PRESENT / MISSING / INSUFFICIENT vs the tier per `harness-components.md`.
4. Propose diffs — advisory; apply write-safe changes on the user's confirm; route guardrail
   gaps to `hookify`.
5. Refresh `docs/harness-checklist.md`.

## Rules
- Hybrid: WRITE only deterministic, secret-free config; CHECKLIST anything needing
  credentials, installs, or external deploys. NEVER write secrets into files.
- GUARDRAILS RULE: any destructive/irreversible op (deletes, payments, prod writes) REQUIRES
  a guardrail regardless of tier.
- Delegate hook authoring to `hookify` — harness-setup decides WHAT, hookify writes the hook.
- Never configure Orchestration or Instructions (out of scope / owned by other skills).

## Reference files
- `harness-components.md` — the 4 components, tiers, write-safe/checklist, op→guardrail map, snippets.
- `templates/harness-checklist.md.template` — the produced checklist doc.
