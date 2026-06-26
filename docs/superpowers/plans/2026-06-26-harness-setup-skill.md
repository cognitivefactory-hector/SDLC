# Harness-Setup Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `harness-setup` personal skill — a state-aware, hybrid configurator for the four harness components (Tools/MCP, Sandboxes, Guardrails/Hooks, Observability) that writes safe config and checklists the credential/install steps.

**Architecture:** Three markdown files in `~/.claude/skills/harness-setup/` (same Approach-B pattern as `preflight` and `context-engineering`): a thin `SKILL.md` orchestrator, a shared `harness-components.md` contract, and a checklist template. Guardrail authoring is delegated to the `hookify` skill.

**Tech Stack:** Claude Code skills (Markdown + YAML frontmatter). No build step. Verification is behavioral dry-runs.

## Global Constraints

- Install location: `~/.claude/skills/harness-setup/`. Copy content verbatim.
- Covers 4 components: Tools/MCP, Sandboxes/execution, Guardrails/Hooks, Observability. Orchestration is OUT of scope; Instructions owned by `context-engineering`.
- Behavior is HYBRID: WRITE deterministic secret-free config; CHECKLIST credential/install/deploy steps. NEVER write secrets into files.
- Guardrail authoring is DELEGATED to the `hookify` skill (harness-setup decides what, hookify writes the hook).
- GUARDRAILS RULE (verbatim): if any destructive/irreversible op exists (deletes, payments, prod writes), the guardrail is REQUIRED regardless of tier.
- Tiers: Prototype / Structured / Production (same as `preflight`).
- Input: reads `context-engineering`'s `docs/context-map.md` (deferred Tools & Guardrails rows).
- Source of truth: `docs/superpowers/specs/2026-06-26-harness-setup-skill-design.md`.
- Commits OPTIONAL: only if `~/.claude` is version-controlled; otherwise skip.
- Fence caution: `SKILL.md` and the template start with their own first line (`---` or `#`), NOT an outer display fence. `harness-components.md` DOES contain inner ` ```json `/` ```bash ` snippet blocks — those are file content; do NOT add an additional outer fence around the whole file.

---

## File Structure

| File | Responsibility |
|---|---|
| `~/.claude/skills/harness-setup/harness-components.md` | Contract: 4 components × tier, write-safe vs checklist, dangerous-op→guardrail map, ready-to-paste config snippets, audit scoring. Built first. |
| `~/.claude/skills/harness-setup/templates/harness-checklist.md.template` | The produced doc (wired ✓ + TODOs). |
| `~/.claude/skills/harness-setup/SKILL.md` | Orchestrator: mode detect → per-component generate/audit → checklist; hookify hand-off. |

Build order: `harness-components.md` → template → `SKILL.md` → validation.

---

### Task 1: Create the contract (`harness-components.md`)

**Files:**
- Create: `~/.claude/skills/harness-setup/harness-components.md`

**Interfaces:**
- Produces the contract referenced by `SKILL.md` (Task 3). Section anchors relied on: "Components", "Dangerous-op → guardrail map", "Ready-to-paste config snippets", "Audit scoring (AUDIT mode)".

- [ ] **Step 1: Define done**

Complete when the file contains the 4-component table (with tier + write-safe + checklist columns), the GUARDRAILS RULE, the dangerous-op→guardrail map, the config snippets (Sentry/Langfuse/OTel), and the audit scoring rubric.

- [ ] **Step 2: Create the directory**

```bash
mkdir -p ~/.claude/skills/harness-setup/templates
```

- [ ] **Step 3: Write `harness-components.md` with this exact content** (the file contains inner ` ``` ` snippet blocks — reproduce them; do NOT wrap the whole file in an extra outer fence)

````markdown
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

### OpenTelemetry → Langfuse — CHECKLIST (app code, not config)

Add the OTel SDK to the app and point its exporter at Langfuse. OTel is NOT an MCP server.

## Audit scoring (AUDIT mode)

For each component, mark PRESENT / MISSING / INSUFFICIENT vs the tier:
- MISSING: component absent (no .mcp.json, no hooks, no observability).
- INSUFFICIENT: present but below tier need (e.g., a dangerous op with no covering guardrail;
  prod with no tracing/error tracking).
Always refresh `docs/harness-checklist.md`.
````

- [ ] **Step 4: Verify**

Run: `grep -cE "^\| [0-9] \| (Tools|Sandboxes|Guardrails|Observability)" ~/.claude/skills/harness-setup/harness-components.md`
Expected: `4` (four component rows).

Run: `grep -cE "GUARDRAILS RULE|Dangerous-op|Audit scoring" ~/.claude/skills/harness-setup/harness-components.md`
Expected: `3`.

Run: `grep -c "mcp.sentry.dev" ~/.claude/skills/harness-setup/harness-components.md`
Expected: `1` (Sentry snippet present).

- [ ] **Step 5: Commit (optional — only if `~/.claude` is version-controlled)**

```bash
cd ~/.claude && git add skills/harness-setup/harness-components.md && git commit -m "feat(harness-setup): add harness-components contract"
```

---

### Task 2: Create the checklist template

**Files:**
- Create: `~/.claude/skills/harness-setup/templates/harness-checklist.md.template`

**Interfaces:**
- Consumes nothing. Produces the doc `SKILL.md` (Task 3) writes into target projects as `docs/harness-checklist.md`. The `{{TOKEN}}` markers are intentional fill-in fields.

- [ ] **Step 1: Define done**

Complete when the template exists with its `{{TOKEN}}` fields and the four sections (Wired, Outstanding, Guardrails, Notes).

- [ ] **Step 2: Write `templates/harness-checklist.md.template` with this exact content** (starts with `#` on line 1 — NO outer code fence)

```markdown
# Harness Checklist — {{PROJECT_NAME}}

Tier: {{TIER}}   ·   Generated by harness-setup

## Wired ✓ (done by the skill)
{{WIRED_LIST}}

## Outstanding (you do these)
{{TODO_LIST}}

## Guardrails (via hookify)
{{GUARDRAILS_LIST}}

## Notes
- Observability: error tracking (Sentry) + tracing/cost (OTel→Langfuse). Trajectory evals
  depend on this being wired (see the eval-authoring skill).
- Re-run `harness-setup` in AUDIT mode after wiring to refresh this checklist.
```

- [ ] **Step 3: Verify**

Run: `grep -cE "Wired|Outstanding|Guardrails|Notes" ~/.claude/skills/harness-setup/templates/harness-checklist.md.template`
Expected: `≥4`.

Run: `grep -c "{{" ~/.claude/skills/harness-setup/templates/harness-checklist.md.template`
Expected: `≥4` (fill-in tokens present).

- [ ] **Step 4: Commit (optional)**

```bash
cd ~/.claude && git add skills/harness-setup/templates && git commit -m "feat(harness-setup): add harness-checklist template"
```

---

### Task 3: Create the skill orchestrator (`SKILL.md`)

**Files:**
- Create: `~/.claude/skills/harness-setup/SKILL.md`

**Interfaces:**
- Consumes `harness-components.md` (Task 1) and the template (Task 2) by relative path.
- Produces the activatable skill. Frontmatter `name: harness-setup`; `description` contains `harness-setup`.

- [ ] **Step 1: Define done**

Complete when `SKILL.md` has valid frontmatter, a mode-detection step, GENERATE and AUDIT procedures, the hybrid write/checklist rule, the GUARDRAILS RULE, and the `hookify` delegation.

- [ ] **Step 2: Write `SKILL.md` with this exact content** (file starts with `---` on line 1 — NO outer code fence)

```markdown
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
```

- [ ] **Step 3: Verify**

Run: `head -2 ~/.claude/skills/harness-setup/SKILL.md`
Expected: line 1 `---`, line 2 `name: harness-setup`.

Run: `grep -c "harness-components.md" ~/.claude/skills/harness-setup/SKILL.md`
Expected: `≥2`.

Run: `grep -c "hookify" ~/.claude/skills/harness-setup/SKILL.md`
Expected: `≥3`.

- [ ] **Step 4: Commit (optional)**

```bash
cd ~/.claude && git add skills/harness-setup/SKILL.md && git commit -m "feat(harness-setup): add skill orchestrator with GENERATE/AUDIT modes"
```

---

### Task 4: Validate against the three acceptance scenarios

**Files:**
- No new files. Behavioral validation of the skill built in Tasks 1–3.

- [ ] **Step 1: Confirm discoverability**

In a NEW Claude Code session, confirm `harness-setup` appears in the skill list. If not, check the Task 3 frontmatter.

- [ ] **Step 2: Scenario A — greenfield prototype, no dangerous ops (GENERATE)**

Dry-run in an empty/prototype repo with no destructive operations.
Expected:
- GENERATE mode; tier inferred Prototype.
- Observability dropped (— at Prototype); minimal/no required config.
- `docs/harness-checklist.md` near-empty; no guardrails invented.
If it over-configures (wires observability or invents guardrails), fix `harness-components.md` tier markers / the dangerous-op scan.

- [ ] **Step 3: Scenario B — production LLM app with a destructive op (GENERATE)**

Dry-run in a deployed multi-user app that summarizes text (LLM) and has a DB-drop path.
Expected:
- Tier Production.
- Detects the DB-drop op → invokes `hookify` to author a guardrail; verifies coverage.
- Writes the Sentry `.mcp.json` entry + enables the plugin.
- `docs/harness-checklist.md` lists outstanding TODOs: Langfuse deploy, OTel SDK, drift+feedback.
If it does not route the dangerous op to hookify or skips the Sentry write, fix the GENERATE steps.

- [ ] **Step 4: Scenario C — existing repo, `.mcp.json` present, unguarded `rm -rf` (AUDIT)**

Dry-run in a repo that already has a `.mcp.json` but a code path running `rm -rf` with no hook.
Expected:
- AUDIT mode; scans dangerous ops.
- Flags the `rm -rf` as a RULE violation (missing guardrail) → routes to `hookify`.
- Scores Observability INSUFFICIENT if the repo is prod and unwired.
- Refreshes `docs/harness-checklist.md`.
If it misses the unguarded op, fix the dangerous-op map / audit scoring in `harness-components.md`.

- [ ] **Step 5: Record results and fix inline**

For any scenario that diverged, edit `harness-components.md` or `SKILL.md`, then re-run that
scenario. Passes when all three behave as specified.

- [ ] **Step 6: Final commit (optional)**

```bash
cd ~/.claude && git add skills/harness-setup && git commit -m "test(harness-setup): validate prototype/production/audit scenarios"
```

---

## Self-Review

**1. Spec coverage:**
- 4 components, defer orchestration → Task 1 table + Task 3 rules. ✓
- Hybrid write-safe/checklist → Task 1 columns + snippets + Task 3 rules. ✓
- State-aware GENERATE/AUDIT → Task 3 Step 0 + both procedures. ✓
- hookify delegation for guardrails → Task 1 op-map + Task 3 GENERATE step 4 / AUDIT step 4 + verify (`hookify` ≥3). ✓
- GUARDRAILS RULE (any tier) → Task 1 RULE + Task 3 rules. ✓
- Reads context-engineering's context-map.md → Task 3 GENERATE step 1. ✓
- Observability per tier (Sentry write-safe, Langfuse/OTel checklist) → Task 1 row 4 + snippets + Task 3 step 4. ✓
- Produces docs/harness-checklist.md → Task 2 template + Task 3 step 5. ✓
- Three acceptance scenarios → Task 4. ✓
- Trigger phrase `harness-setup` → Task 3 frontmatter. ✓

**2. Placeholder scan:** No TBD/TODO. All file contents complete and verbatim. The `{{TOKEN}}` markers (template) and `YOUR-LANGFUSE-HOST`/`${LANGFUSE_MCP_TOKEN}` (snippet placeholders) are intentional fill-in fields, not plan gaps. ✓

**3. Type consistency:** File paths consistent across tasks. Section anchors `SKILL.md` references ("harness-components.md", the op-map, audit scoring) match headings defined in `harness-components.md` (Task 1). `hookify` delegation phrasing consistent across Task 1, Task 3, and Task 4. Component names (Tools/MCP, Sandboxes, Guardrails, Observability) identical across all files. ✓
