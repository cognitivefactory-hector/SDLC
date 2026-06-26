# Preflight Readiness Gate Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `preflight` personal skill — an adaptive, advisory SDLC readiness gate that bookends the build flow (opens at brainstorm, checks before implementation).

**Architecture:** Three markdown files in `~/.claude/skills/preflight/` (Approach B): a thin `SKILL.md` orchestrator, a shared `criteria.md` contract (the suite's single source of truth), and a `report-template.md`. The skill is state-aware: no spec/plan → OPEN mode; spec+plan present → CHECK mode.

**Tech Stack:** Claude Code skills (Markdown + YAML frontmatter). No build step, no compiler. Verification is behavioral dry-runs.

## Global Constraints

- Install location: `~/.claude/skills/preflight/` (global personal skill). Copy verbatim.
- Trigger word: `preflight` (must appear in `SKILL.md` description so the skill auto-activates).
- Behavior is **advisory** — the skill produces a verdict and STOPS; it never hard-blocks or edits code.
- Verdicts: exactly `GO` / `GO-WITH-GAPS` / `NOT-READY`.
- Tiers: exactly `Prototype` / `Structured` / `Production`.
- Conditional rows 4 & 6 apply ONLY if the project has non-deterministic/LLM parts; otherwise removed, never failed.
- Source of truth for all content: `docs/superpowers/specs/2026-06-26-sdlc-preflight-readiness-gate-design.md`.
- Commits are OPTIONAL: run the commit step only if `~/.claude` is version-controlled; otherwise skip it.

---

## File Structure

| File | Responsibility |
|---|---|
| `~/.claude/skills/preflight/criteria.md` | The contract: tiers, inference signals, non-determinism detection, 7 components × evidence × tier. Read by the gate AND the other suite skills. Built first — everything references it. |
| `~/.claude/skills/preflight/report-template.md` | The CHECK-mode output shape + status icons + verdict logic. |
| `~/.claude/skills/preflight/SKILL.md` | Thin orchestration: frontmatter (trigger), mode detection, OPEN procedure, CHECK procedure, invariant rules. |

Build order: `criteria.md` → `report-template.md` → `SKILL.md` → validation. The contract comes first because `SKILL.md` references it.

---

### Task 1: Create the criteria contract (`criteria.md`)

**Files:**
- Create: `~/.claude/skills/preflight/criteria.md`

**Interfaces:**
- Produces: the canonical contract referenced by `SKILL.md` (Task 3) at path `~/.claude/skills/preflight/criteria.md`, and by the future suite skills. Section anchors relied on by `SKILL.md`: "Tier inference signals", "Non-determinism detection", "Criteria".

- [ ] **Step 1: Define done (acceptance for this task)**

This task is complete when `criteria.md` contains: the 3 tier definitions; tier-inference signals; non-determinism detection rule; and all 7 components each with tier markers, `[scriptable]`/`[judgment]` evidence, and the two invariant rules (guardrails-always-if-destructive; trajectory-evals-need-observability).

- [ ] **Step 2: Create the directory**

```bash
mkdir -p ~/.claude/skills/preflight
```

- [ ] **Step 3: Write `criteria.md` with this exact content**

```markdown
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
```

- [ ] **Step 4: Verify the file**

Run: `cat ~/.claude/skills/preflight/criteria.md | grep -E "^### [0-9]" | wc -l`
Expected: `7` (seven numbered components present).

Run: `grep -c "DEPENDENCY" ~/.claude/skills/preflight/criteria.md`
Expected: `1` (the trajectory→observability dependency is recorded).

- [ ] **Step 5: Commit (optional — only if `~/.claude` is version-controlled)**

```bash
cd ~/.claude && git add skills/preflight/criteria.md && git commit -m "feat(preflight): add SDLC readiness criteria contract"
```

---

### Task 2: Create the report template (`report-template.md`)

**Files:**
- Create: `~/.claude/skills/preflight/report-template.md`

**Interfaces:**
- Consumes: nothing.
- Produces: the output shape `SKILL.md` (Task 3) uses in CHECK mode. Relied-on tokens: status icons `✅`/`⚠️`/`❌`; verdicts `GO`/`GO-WITH-GAPS`/`NOT-READY`.

- [ ] **Step 1: Define done (acceptance for this task)**

Complete when `report-template.md` shows the header line, one-line-per-item body with a fix-on-gap arrow, the verdict line, the "To reach GO" line, the status-icon key, and the verdict logic.

- [ ] **Step 2: Write `report-template.md` with this exact content**

````markdown
# Preflight report template (CHECK mode)

Render this. Omit non-applicable conditional rows (4 & 6) entirely.

```
PREFLIGHT — <project>   ·   Tier: <tier> (confirmed)
─────────────────────────────────────────────────────────
<icon> <n> <component>      <evidence-if-present | gap-if-missing>   → <specific fix if gap>
... one line per APPLICABLE item ...
─────────────────────────────────────────────────────────
VERDICT:  <GO | GO-WITH-GAPS | NOT-READY>  — <one-line reason>
To reach GO:  <ordered specific next actions, or "nothing — you're clear">
```

## Status icons
- `✅` present
- `⚠️` recommended-gap (a `◐` item is missing)
- `❌` required-missing (a `●` item is missing)

## Verdict logic
- `GO`           → every REQUIRED (`●`) applicable item is `✅`, no recommended gaps.
- `GO-WITH-GAPS` → every required item is `✅`, but ≥1 RECOMMENDED (`◐`) item is a gap.
- `NOT-READY`    → ≥1 REQUIRED (`●`) applicable item is `❌`.

Every gap line MUST carry a specific next action after `→`, so the report reads as a
to-do list, not a scold.
````

- [ ] **Step 3: Verify the file**

Run: `grep -cE "GO-WITH-GAPS|NOT-READY" ~/.claude/skills/preflight/report-template.md`
Expected: `≥2` (verdicts documented in template + logic).

- [ ] **Step 4: Commit (optional — only if `~/.claude` is version-controlled)**

```bash
cd ~/.claude && git add skills/preflight/report-template.md && git commit -m "feat(preflight): add readiness report template"
```

---

### Task 3: Create the skill orchestrator (`SKILL.md`)

**Files:**
- Create: `~/.claude/skills/preflight/SKILL.md`

**Interfaces:**
- Consumes: `criteria.md` (Task 1) and `report-template.md` (Task 2) by relative path within the skill dir.
- Produces: the activatable skill. Frontmatter `name: preflight`; `description` contains the trigger word `preflight`.

- [ ] **Step 1: Define done (acceptance for this task)**

Complete when `SKILL.md` has valid frontmatter (`name`, `description` mentioning `preflight` and "before implementation"), a mode-detection step, an OPEN procedure that ends by invoking `superpowers:brainstorming`, a CHECK procedure that emits the report, and the invariant rules.

- [ ] **Step 2: Write `SKILL.md` with this exact content**

````markdown
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
````

- [ ] **Step 3: Verify frontmatter and triggers**

Run: `head -3 ~/.claude/skills/preflight/SKILL.md`
Expected: starts with `---`, line 2 is `name: preflight`.

Run: `grep -c "preflight" ~/.claude/skills/preflight/SKILL.md`
Expected: `≥3` (trigger word appears in description and body).

Run: `grep -c "superpowers:brainstorming" ~/.claude/skills/preflight/SKILL.md`
Expected: `1` (OPEN mode hands off to brainstorming).

- [ ] **Step 4: Commit (optional — only if `~/.claude` is version-controlled)**

```bash
cd ~/.claude && git add skills/preflight/SKILL.md && git commit -m "feat(preflight): add skill orchestrator with OPEN/CHECK modes"
```

---

### Task 4: Validate against the three acceptance scenarios

**Files:**
- No new files. Behavioral validation of the skill built in Tasks 1–3.

**Interfaces:**
- Consumes: the complete skill at `~/.claude/skills/preflight/`.

- [ ] **Step 1: Confirm the skill is discoverable**

In a NEW Claude Code session (so the skill list reloads), confirm `preflight` appears in the available skills. If not, check the frontmatter `name`/`description` from Task 3.

- [ ] **Step 2: Scenario A — throwaway prototype (expect fast GO)**

Dry-run: invoke `preflight` in CHECK mode against a tiny personal script with a spec but no LLM parts, not deployed.
Expected behavior:
- Tier inferred = `Prototype` (proposed, confirmed).
- Rows 4, 6, 7 DROP (4 & 6 non-applicable; 7 not required at Prototype).
- All remaining items are `◐` → verdict `GO` (or `GO-WITH-GAPS` only if a recommended item is genuinely absent).
If rows 4/6/7 do not drop, fix `criteria.md` non-determinism/tier logic.

- [ ] **Step 3: Scenario B — production LLM feature (expect NOT-READY until evals + guardrails)**

Dry-run: invoke `preflight` against a deployed, multi-user feature that summarizes user text (LLM), with a spec + plan but no eval suite and a destructive op lacking a guardrail.
Expected behavior:
- Tier = `Production`.
- Rows 4 & 6 ACTIVE.
- Item 3 = `❌` (destructive op, no guardrail) and item 6 = `❌` (no evals) → verdict `NOT-READY`.
- Report names specific fixes (add guardrail hook; add output + trajectory evals) and notes item 6's dependency on item 7.
If it does not flag both, fix the evidence/verdict logic.

- [ ] **Step 4: Scenario C — deterministic CRUD prod app (expect rows 4 & 6 correctly drop)**

Dry-run: invoke `preflight` against a deployed CRUD app with NO LLM parts.
Expected behavior:
- Tier = `Production`.
- Rows 4 & 6 DROP OFF (no non-deterministic parts) and are NOT counted against the verdict.
- Items 1,2,3,5,7 required; verdict reflects only those.
If rows 4/6 appear or are failed, fix the non-determinism detection.

- [ ] **Step 5: Record results and fix discrepancies inline**

For any scenario whose behavior diverged, edit `criteria.md` or `SKILL.md` to correct it, then re-run that scenario. The skill passes when all three behave as specified.

- [ ] **Step 6: Final commit (optional — only if `~/.claude` is version-controlled)**

```bash
cd ~/.claude && git add skills/preflight && git commit -m "test(preflight): validate against prototype/production-LLM/deterministic scenarios"
```

---

## Self-Review

**1. Spec coverage:**
- Adaptive + advisory, GO/GAPS/NOT-READY → Task 3 CHECK + Task 2 verdict logic. ✓
- Infer-then-confirm tier → Task 1 signals + Task 3 OPEN/CHECK steps. ✓
- Global install, Approach B (3 files) → File Structure + Tasks 1–3. ✓
- Bookend through-line, state-aware OPEN/CHECK → Task 3 Step 0 + both procedures. ✓
- Tier model + 7-item criteria matrix + conditional rows + 2 invariant rules → Task 1. ✓
- Output vs trajectory evals + dependency on observability → Task 1 item 6. ✓
- Future-script seam (`[scriptable]`/`[judgment]` tags) → Task 1 evidence tags. ✓
- Three acceptance scenarios → Task 4. ✓
- Trigger word `preflight` → Task 3 frontmatter. ✓

**2. Placeholder scan:** No TBD/TODO. All file contents are complete and verbatim. The `<…>` tokens inside `report-template.md` are intentional template fields, not plan gaps. ✓

**3. Type consistency:** File paths consistent across tasks (`~/.claude/skills/preflight/{criteria,report-template,SKILL}.md`). Verdict strings, tier names, status icons, and section anchors referenced in `SKILL.md` match those defined in `criteria.md`/`report-template.md`. ✓
