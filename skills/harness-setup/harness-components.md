# Harness Components — the harness-setup contract

Shared reference for the `harness-setup` skill. Defines the four harness components this
skill configures, what each needs per tier, which parts are WRITE-SAFE (the skill writes
them) vs CHECKLIST (the user does them), the dangerous-op→guardrail map, and ready-to-paste
config snippets.

`context-engineering` owns Instructions; Orchestration logic is deferred.

Tiers (same as preflight): Prototype / Structured / Production.
Legend: — not required · ◐ recommended · ● required.

## Components

| # | Component | Proto | Struct | Prod | Write-safe (skill does) | Checklist (user does) |
|---|---|---|---|---|---|---|
| 1 | Tools / MCP | ◐ | ● | ● | .mcp.json entries for no-secret/OAuth servers; document tools | OAuth approval; self-hosted URLs+keys; local MCP installs |
| 2 | Sandboxes / execution | ◐ | ● | ● | .claude/settings.json allow/deny permissions | container/CI sandbox for prod |
| 3 | Guardrails / Hooks | ◐* | ● | ● | identify dangerous ops → hand to hookify | confirm each op has a covering hook |
| 4 | Observability | — | ◐ | ● | Sentry .mcp.json entry + enable plugin; OTel config pointers | Sentry OAuth; deploy Langfuse + add MCP; OTel SDK in app; drift + prod→dev feedback (prod) |

`*` GUARDRAILS RULE: if any destructive/irreversible op exists (deletes, payments, prod
writes), the guardrail is REQUIRED regardless of tier.

## Dangerous-op → guardrail map

| Dangerous op detected | Guardrail to author (via hookify) |
|---|---|
| File deletes / `rm -rf` | pre-tool-use hook: block/confirm destructive FS ops |
| DB drop / destructive migration | pre-tool-use or pre-commit hook |
| Payments / money writes | confirmation guardrail |
| Secrets / hardcoded creds in a commit | pre-commit hook blocking secrets |
| Prod writes / deploys | confirmation or scoped-permission guardrail |

## Ready-to-paste config snippets

### Sentry MCP (hosted, OAuth) — WRITE-SAFE into .mcp.json

```json
{
  "mcpServers": {
    "sentry": { "type": "http", "url": "https://mcp.sentry.dev/mcp" }
  }
}
```

Then checklist: complete OAuth on first connect.

### Langfuse MCP (self-hosted) — CHECKLIST (needs a deployed instance + keys)

```bash
# token = base64 of "public_key:secret_key"
claude mcp add --transport http langfuse \
  https://YOUR-LANGFUSE-HOST/api/public/mcp \
  --header "Authorization: Basic ${LANGFUSE_MCP_TOKEN}"
```

### OpenTelemetry → Langfuse — CHECKLIST only (app code required, not writable config)

Add the OTel SDK to the app and point its exporter at Langfuse. OTel is NOT an MCP server,
and the skill cannot write it — it is always a checklist item, never a write-safe action.

## Audit scoring (AUDIT mode)

For each component, mark PRESENT / MISSING / INSUFFICIENT vs the tier:
- MISSING: component absent (no .mcp.json, no hooks, no observability).
- INSUFFICIENT: present but below tier need (e.g., a dangerous op with no covering guardrail;
  prod with no tracing/error tracking).
Always refresh `docs/harness-checklist.md`.
