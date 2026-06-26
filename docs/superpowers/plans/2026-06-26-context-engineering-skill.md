# Context-Engineering Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `context-engineering` personal skill — a state-aware generator/auditor that produces `AGENTS.md` + a `CLAUDE.md` pointer + a versioned static-vs-dynamic context map.

**Architecture:** Four markdown files in `~/.claude/skills/context-engineering/` (same Approach-B pattern as `preflight`): a thin `SKILL.md` orchestrator, a shared `context-types.md` contract, and two fill-in templates. The skill writes its artifacts into the *target project*, not the skill dir.

**Tech Stack:** Claude Code skills (Markdown + YAML frontmatter). No build step. Verification is behavioral dry-runs.

## Global Constraints

- Install location: `~/.claude/skills/context-engineering/`. Copy content verbatim.
- The skill OWNS four context types (Instructions, Knowledge, Memory, Examples) and DEFERS two (Tools, Guardrails) to `harness-setup` — but MAPS all six.
- Rule file: `AGENTS.md` canonical; `CLAUDE.md` is a one-line pointer containing exactly `@AGENTS.md`.
- Context map lands at `docs/context-map.md` in the target project.
- Static-vs-dynamic default: when unsure, DYNAMIC; promote to static only if needed every interaction.
- Source of truth: `docs/superpowers/specs/2026-06-26-context-engineering-skill-design.md`.
- Commits OPTIONAL: only if `~/.claude` is version-controlled; otherwise skip.
- Fence caution (learned on `preflight`): file content must NOT include an outer display code-fence. A `SKILL.md` starts with `---` on line 1. Template/reference files start with their `#` heading on line 1.

---

## File Structure

| File | Responsibility |
|---|---|
| `~/.claude/skills/context-engineering/context-types.md` | The contract: 6 types, ownership, static/dynamic heuristic, interview questions, audit checklist. Built first — SKILL.md and the audit/interview derive from it. |
| `~/.claude/skills/context-engineering/templates/AGENTS.md.template` | Fill-in canonical instructions file. |
| `~/.claude/skills/context-engineering/templates/context-map.md.template` | Fill-in static-vs-dynamic map. |
| `~/.claude/skills/context-engineering/SKILL.md` | Thin orchestrator: mode detect → interview/audit → write artifacts. |

Build order: `context-types.md` → templates → `SKILL.md` → validation.

---

### Task 1: Create the contract (`context-types.md`)

**Files:**
- Create: `~/.claude/skills/context-engineering/context-types.md`

**Interfaces:**
- Produces the contract referenced by `SKILL.md` (Task 3). Section anchors relied on: "The six context types", "Static vs dynamic heuristic", "Interview questions (GENERATE mode)", "Audit checklist (AUDIT mode)".

- [ ] **Step 1: Define done**

Complete when the file contains the 6-type table (with owner + placement + lands-in), the static/dynamic heuristic, the GENERATE interview questions, and the AUDIT checklist.

- [ ] **Step 2: Create the directory**

```bash
mkdir -p ~/.claude/skills/context-engineering/templates
```

- [ ] **Step 3: Write `context-types.md` with this exact content**

```markdown
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
```

- [ ] **Step 4: Verify**

Run: `grep -cE "^\| [0-9] \|" ~/.claude/skills/context-engineering/context-types.md`
Expected: `6` (six type rows).

Run: `grep -cE "Interview questions|Audit checklist|Static vs dynamic" ~/.claude/skills/context-engineering/context-types.md`
Expected: `3`.

- [ ] **Step 5: Commit (optional — only if `~/.claude` is version-controlled)**

```bash
cd ~/.claude && git add skills/context-engineering/context-types.md && git commit -m "feat(context-engineering): add context-types contract"
```

---

### Task 2: Create the two templates

**Files:**
- Create: `~/.claude/skills/context-engineering/templates/AGENTS.md.template`
- Create: `~/.claude/skills/context-engineering/templates/context-map.md.template`

**Interfaces:**
- Consumes nothing. Produces the two fill-in files `SKILL.md` (Task 3) writes into target projects. The `{{TOKEN}}` markers are intentional fill-in fields.

- [ ] **Step 1: Define done**

Complete when both template files exist with their `{{TOKEN}}` fields and the Tools/Guardrails "deferred to harness-setup" note.

- [ ] **Step 2: Write `templates/AGENTS.md.template` with this exact content**

```markdown
# {{PROJECT_NAME}} — Agent Instructions

> Canonical instructions for AI agents in this repo. Claude Code reads this via `CLAUDE.md`,
> which imports it with `@AGENTS.md`.

## Purpose
{{ONE_PARAGRAPH_PURPOSE}}

## Your role
{{AGENT_ROLE}}

## Hard boundaries (never do)
{{BOUNDARIES_LIST}}

## Conventions
{{CONVENTIONS_LIST}}

## Persistent project facts (static memory)
{{PERSISTENT_FACTS}}

## Knowledge — where to look (dynamic, load on demand)
{{KNOWLEDGE_REFS}}

## Examples / reference patterns (dynamic)
{{EXAMPLE_REFS}}

## Tools & Guardrails
Configured by the `harness-setup` skill — see `docs/context-map.md` for the map.
```

- [ ] **Step 3: Write `templates/context-map.md.template` with this exact content**

```markdown
# Context Map — {{PROJECT_NAME}}

The static-vs-dynamic context plan. A first-class, versioned decision — review it like config.

Legend: **Static** = always loaded (high token cost). **Dynamic** = loaded on demand (low per-turn cost).

| Type | Item | Static / Dynamic | Location | Owner |
|---|---|---|---|---|
| Instructions | {{INSTRUCTIONS_ITEM}} | Static | AGENTS.md | context-engineering |
| Knowledge | {{KNOWLEDGE_ITEM}} | Dynamic | {{KNOWLEDGE_PATH}} | context-engineering |
| Memory (persistent) | {{MEMORY_PERSISTENT}} | Static | AGENTS.md | context-engineering |
| Memory (session) | {{MEMORY_SESSION}} | Dynamic | session | context-engineering |
| Examples | {{EXAMPLES_ITEM}} | Dynamic | {{EXAMPLES_PATH}} | context-engineering |
| Tools | (deferred) | Dynamic | (deferred) | harness-setup |
| Guardrails | (deferred) | Static | (deferred) | harness-setup |

## Notes
- When unsure, default to dynamic; promote to static only if needed every interaction.
- The Tools & Guardrails rows are placeholders until `harness-setup` fills them.
```

- [ ] **Step 4: Verify**

Run: `ls ~/.claude/skills/context-engineering/templates/`
Expected: `AGENTS.md.template  context-map.md.template`.

Run: `grep -l "deferred to harness-setup\|harness-setup" ~/.claude/skills/context-engineering/templates/*.template | wc -l`
Expected: `2` (both templates reference the harness-setup hand-off).

- [ ] **Step 5: Commit (optional)**

```bash
cd ~/.claude && git add skills/context-engineering/templates && git commit -m "feat(context-engineering): add AGENTS.md and context-map templates"
```

---

### Task 3: Create the skill orchestrator (`SKILL.md`)

**Files:**
- Create: `~/.claude/skills/context-engineering/SKILL.md`

**Interfaces:**
- Consumes `context-types.md` (Task 1) and the two templates (Task 2) by relative path.
- Produces the activatable skill. Frontmatter `name: context-engineering`; `description` contains the trigger phrase `context-engineering`.

- [ ] **Step 1: Define done**

Complete when `SKILL.md` has valid frontmatter, a mode-detection step, GENERATE and AUDIT procedures, the exact `@AGENTS.md` pointer instruction, and the rules (own types 1–4 only; default dynamic).

- [ ] **Step 2: Write `SKILL.md` with this exact content** (file starts with `---` on line 1 — NO outer code fence)

```markdown
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
```

- [ ] **Step 3: Verify**

Run: `head -2 ~/.claude/skills/context-engineering/SKILL.md`
Expected: line 1 `---`, line 2 `name: context-engineering`.

Run: `grep -c "context-types.md" ~/.claude/skills/context-engineering/SKILL.md`
Expected: `≥2`.

Run: `grep -c "@AGENTS.md" ~/.claude/skills/context-engineering/SKILL.md`
Expected: `≥1`.

- [ ] **Step 4: Commit (optional)**

```bash
cd ~/.claude && git add skills/context-engineering/SKILL.md && git commit -m "feat(context-engineering): add skill orchestrator with GENERATE/AUDIT modes"
```

---

### Task 4: Validate against the three acceptance scenarios

**Files:**
- No new files. Behavioral validation of the skill built in Tasks 1–3.

- [ ] **Step 1: Confirm discoverability**

In a NEW Claude Code session, confirm `context-engineering` appears in the skill list. If not, check the Task 3 frontmatter.

- [ ] **Step 2: Scenario A — greenfield empty repo (GENERATE)**

Dry-run in an empty repo with no `AGENTS.md`/`CLAUDE.md`.
Expected:
- Enters GENERATE mode; runs the interview (only for what it can't infer).
- Writes `AGENTS.md`, `CLAUDE.md` containing exactly `@AGENTS.md`, and `docs/context-map.md`.
- Map sorts items static/dynamic; Tools & Guardrails rows marked `(deferred)`.
If it does not produce all three artifacts or duplicates AGENTS.md into CLAUDE.md, fix `SKILL.md` GENERATE steps.

- [ ] **Step 3: Scenario B — existing repo with a `CLAUDE.md` (AUDIT)**

Dry-run in a repo that already has a `CLAUDE.md` but no context map.
Expected:
- Enters AUDIT mode; scores the six types PRESENT/MISSING/MISPLACED.
- Flags missing types (e.g., no Knowledge refs, no map) and proposes diffs.
- Produces `docs/context-map.md`.
If it regenerates from scratch instead of auditing, fix the mode-detection step.

- [ ] **Step 4: Scenario C — misplaced context (AUDIT)**

Dry-run in a repo whose `CLAUDE.md` contains a large API reference pasted inline (static).
Expected:
- AUDIT flags that item as MISPLACED ("should be dynamic") and proposes moving it to a
  referenced doc with a pointer.
If it does not flag the misplacement, fix the audit checklist wording in `context-types.md`.

- [ ] **Step 5: Record results and fix inline**

For any scenario that diverged, edit `context-types.md` or `SKILL.md`, then re-run that
scenario. Passes when all three behave as specified.

- [ ] **Step 6: Final commit (optional)**

```bash
cd ~/.claude && git add skills/context-engineering && git commit -m "test(context-engineering): validate greenfield/audit/misplaced scenarios"
```

---

## Self-Review

**1. Spec coverage:**
- State-aware GENERATE/AUDIT → Task 3 Step 0 + both procedures. ✓
- Own 4 / map 6, defer Tools & Guardrails → Task 1 table + Task 2 deferred rows + Task 3 rules. ✓
- AGENTS.md canonical + CLAUDE.md `@AGENTS.md` pointer → Task 2 template + Task 3 GENERATE step 4 + verify. ✓
- Interview-driven + templates + shared reference → Task 1 (interview/checklist) + Task 2 (templates) + Task 3 (orchestrator). ✓
- Static/dynamic heuristic, default-dynamic → Task 1 heuristic + Task 3 rules. ✓
- Map at `docs/context-map.md` → Task 3 GENERATE step 4. ✓
- Three acceptance scenarios → Task 4. ✓
- Trigger phrase `context-engineering` → Task 3 frontmatter. ✓

**2. Placeholder scan:** No TBD/TODO. All file contents complete and verbatim. The `{{TOKEN}}` markers in the templates are intentional fill-in fields, not plan gaps. ✓

**3. Type consistency:** File paths consistent across tasks. Section anchors that `SKILL.md` references ("Interview questions", audit checklist, heuristic) match the headings defined in `context-types.md` (Task 1). The `@AGENTS.md` pointer string is identical in Task 2 template note, Task 3 instruction, and Task 4 verification. ✓
