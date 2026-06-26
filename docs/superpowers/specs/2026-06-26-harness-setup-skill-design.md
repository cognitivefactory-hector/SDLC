# Harness-Setup Skill — Design Spec

**Date:** 2026-06-26
**Status:** Approved design, ready for implementation plan
**Suite:** SDLC skill suite (skill #3 of 5)

---

## 1. Purpose

A personal Claude Code skill that configures the **harness** — the scaffolding around the
model that turns it into a working agent — per *The New SDLC with Vibe Coding* and the
companion `observability-and-testing-guide.html`.

It satisfies two `preflight` gate items:
- **Item 3 (Harness):** Tools/MCP defined + guardrails/hooks for dangerous ops.
- **Item 7 (Observability):** error tracking + traces + cost/latency, and drift + a
  production→development feedback loop at Production tier.

## 2. Scope

Covers **four** harness components; defers a fifth (Orchestration). `context-engineering`
already owns the sixth (Instructions).

| Component | In scope? |
|---|---|
| Tools / MCP | ✅ |
| Sandboxes / execution boundary | ✅ |
| Guardrails / Hooks | ✅ (authoring delegated to `hookify`) |
| Observability | ✅ |
| Orchestration logic (sub-agent/model routing) | ❌ deferred (advanced, rarely per-project) |
| Instructions / rule files | ❌ owned by `context-engineering` |

## 3. Decisions log (from brainstorming)

| Decision | Choice |
|---|---|
| Scope | 4 core components (defer Orchestration) |
| Behavior | **Hybrid**: write the safe deterministic config; checklist the credential/install steps |
| State model | State-aware GENERATE / AUDIT |
| Guardrails impl | **Integrate `hookify`** — harness-setup decides what's needed, hookify authors the hooks |
| Skill structure | Approach A — single skill, 4 component sections, shared `harness-components.md` + checklist template |
| Input | reads `context-engineering`'s `docs/context-map.md` (deferred Tools & Guardrails rows) |

## 4. Architecture & files

```
~/.claude/skills/harness-setup/
├── SKILL.md                          # orchestrator: mode detect → per-component generate/audit → checklist
├── harness-components.md             # contract (keystone): 4 components × tier, write-safe vs checklist,
│                                     #   dangerous-op→guardrail map, ready-to-paste config snippets
└── templates/
    └── harness-checklist.md.template  # produced doc: wired ✓ + credential/install TODOs
```

Writes into the **target project**: `.mcp.json` entries, `.claude/settings.json`
permissions/hooks (hooks via `hookify`), and `docs/harness-checklist.md`. Reads
`context-engineering`'s `docs/context-map.md` as input.

## 5. The `harness-components.md` reference (content contract)

Per component: tier requirement, write-safe action, checklist items.

| # | Component | Proto | Struct | Prod | Write-safe (skill does) | Checklist (user does) |
|---|---|:--:|:--:|:--:|---|---|
| 1 | Tools / MCP | ◐ | ● | ● | `.mcp.json` entries for no-secret/OAuth servers; document tools | OAuth approval; self-hosted URLs+keys; local MCP installs |
| 2 | Sandboxes / execution | ◐ | ● | ● | `.claude/settings.json` allow/deny permissions | container/CI sandbox for prod |
| 3 | Guardrails / Hooks | ◐* | ● | ● | identify dangerous ops → hand to `hookify` | confirm each op has a covering hook |
| 4 | Observability | — | ◐ | ● | Sentry `.mcp.json` entry + enable plugin; OTel config pointers | Sentry OAuth; deploy Langfuse + add MCP; OTel SDK in app; drift + prod→dev feedback (prod) |

`*` **Guardrails RULE** (verbatim from `preflight` criteria): if any destructive/irreversible
op exists (deletes, payments, prod writes), the guardrail is REQUIRED regardless of tier.

**Dangerous-op → guardrail map:**

| Dangerous op detected | Guardrail to author (via hookify) |
|---|---|
| File deletes / `rm -rf` | pre-tool-use hook: block/confirm destructive FS ops |
| DB drop / destructive migration | pre-tool-use or pre-commit hook |
| Payments / money writes | confirmation guardrail |
| Secrets / hardcoded creds in a commit | pre-commit hook blocking secrets |
| Prod writes / deploys | confirmation or scoped-permission guardrail |

The reference also holds ready-to-paste config snippets: the Sentry `.mcp.json` entry, the
Langfuse entry (with credential placeholders), and OTel→Langfuse pointers.

## 6. Behavior — state-aware modes

Invoked by intent (e.g. "set up the harness", "harness-setup", "wire up MCP and guardrails").

### Mode detection (first)
- No harness artifacts (`.mcp.json` absent, no hooks in `settings.json`, no
  `docs/harness-checklist.md`) → **GENERATE**.
- Any present → **AUDIT**.
- Ambiguous → ask the user.

### GENERATE mode
1. **Read inputs:** `context-engineering`'s `docs/context-map.md` (deferred Tools &
   Guardrails rows) + the spec, to learn what tools and ops the project needs.
2. **Confirm tier** — infer+confirm via the same signals `preflight` uses (or reuse a tier
   already set).
3. **Scan for dangerous ops** using the dangerous-op→guardrail map.
4. **Per component**, perform the write-safe action and collect checklist items:
   - Tools/MCP: write `.mcp.json` entries for no-secret/OAuth servers; checklist credential ones.
   - Sandboxes: write `.claude/settings.json` allow/deny permissions.
   - Guardrails: for each dangerous op found, hand off to `hookify` to author the hook; verify coverage.
   - Observability (per tier): write the Sentry entry + enable plugin; checklist Langfuse/OTel/drift.
5. **Write** `docs/harness-checklist.md` (wired ✓ + outstanding TODOs).
6. **Summarize** and point to the checklist.

### AUDIT mode
1. Read existing `.mcp.json`, `.claude/settings.json` (hooks/permissions), any checklist.
2. Scan for dangerous ops; verify each has a covering guardrail (the RULE).
3. Score each component PRESENT / MISSING / INSUFFICIENT vs the tier.
4. Propose diffs — advisory; apply write-safe on confirm; route guardrail gaps to `hookify`.
5. Refresh `docs/harness-checklist.md`.

### The `hookify` hand-off
When a dangerous op needs a guardrail, `harness-setup` invokes the `hookify` skill with the
specific rule (e.g., "block `rm -rf`", "block secrets in a commit"), then verifies the hook
exists. harness-setup decides WHAT is needed; hookify AUTHORS the hook.

## 7. Integration

- **→ `preflight`:** produces what items 3 & 7 check; running it moves both toward ✅.
- **← `context-engineering`:** reads `docs/context-map.md` (deferred Tools & Guardrails rows).
- **→ `hookify`:** delegates all guardrail/hook authoring.
- **Builds on existing repo work:** the `.mcp.json` (Sentry) + `MCP_SETUP.md` already here
  are exactly what the Observability component produces; this skill generalizes that flow.
- **Flow placement:** `preflight (open) → brainstorming → context-engineering →
  harness-setup → …`.

## 8. Testing / acceptance scenarios

Skills aren't unit-testable; acceptance = documented dry-runs:
1. **Greenfield prototype, no dangerous ops** → GENERATE: writes minimal config; observability
   dropped at Prototype; checklist near-empty. Verifies it does NOT over-configure.
2. **Production LLM app with a destructive op (DB drop)** → GENERATE: detects the op → routes
   to `hookify` for a guardrail; writes the Sentry entry; checklist carries Langfuse/OTel +
   drift/feedback; `docs/harness-checklist.md` lists the TODOs.
3. **Existing repo with `.mcp.json` but an unguarded `rm -rf`** → AUDIT: flags the missing
   guardrail (RULE violation), routes to `hookify`; scores Observability INSUFFICIENT if it's
   prod and unwired.

Passes when GENERATE writes safe config + checklist and routes dangerous ops to `hookify`;
AUDIT detects unguarded dangerous ops and insufficient observability; nothing is
over-configured at Prototype.

## 9. Out of scope

- Orchestration logic (sub-agent spawning, model routing) — deferred.
- Instructions / rule files — owned by `context-engineering`.
- Authoring the hooks themselves — delegated to `hookify`.
- Deploying external infra (Langfuse server, app-code OTel SDK) — checklisted, not performed.
- The remaining two suite skills (`nondeterministic-design`, `eval-authoring`).

## 10. Related artifacts

- `~/.claude/skills/preflight/` — the gate this skill feeds (items 3 & 7).
- `~/.claude/skills/context-engineering/` — produces the map this skill consumes.
- `.mcp.json`, `MCP_SETUP.md` — existing observability wiring this skill generalizes.
- `observability-and-testing-guide.html` — companion guide (observability section).
