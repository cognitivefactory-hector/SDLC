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
- `GO`           → every REQUIRED (`●`) applicable item is `✅`, and no `◐` item is a gap.
- `GO-WITH-GAPS` → every required item is `✅`, but ≥1 RECOMMENDED (`◐`) item is a gap.
- `NOT-READY`    → ≥1 REQUIRED (`●`) applicable item is `❌`.

Every gap line MUST carry a specific next action after `→`, so the report reads as a
to-do list, not a scold.
