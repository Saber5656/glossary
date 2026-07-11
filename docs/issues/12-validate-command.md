# 12 — `glossary validate`: invariants V1–V5

## Title

`glossary validate`: cross-store consistency checks with error/warning findings

## Summary

Implement the `validate` command running the invariants of DESIGN.md §8 over
config + all three stores, reporting findings, exit 3 on errors.

## Context

`validate` is the safety net for the two-tier model (and runs in users' CI).
It must catch exactly the states the design declares illegal — no more (avoid
false alarms), no less.

## Scope

- `src/cli/commands/validate.ts` + `src/store/invariants.ts` (pure rule
  functions) + tests.

## Detailed Requirements

1. Load config (fail ⇒ its own error path already), then read all stores
   collecting their `problems` first (schema-level findings).
2. Rule functions, each returning findings `{level: 'error'|'warning', code,
   message, where}`:
   - **V1 exclusivity** (error, `V_KEY_OVERLAP`): a key present in ≥2 of
     {candidate keys} / {curated term+alias keys} / {rejected keys}. Message
     names the key and both locations.
   - **V2 uniqueness** (error, `V_DUP_ID` / `V_DUP_KEY`): duplicate curated
     ids (filename collisions can't happen; id-field duplicates can via
     hand-edit) and duplicate term/alias keys across curated files (from 09's
     indexProblems).
   - **V3 references** (error, `V_BAD_REF`): `relatedTerms` id not in byId.
   - **V4 schema** (error, `V_SCHEMA`): any store `problems` entry, plus
     schemaVersion ≠ 1 anywhere.
   - **V5 advisories** (warning): `V_EMPTY_DEF` curated definition == "";
     `V_LLM_DEF` definitionSource == 'llm'; `V_STALE` — only when a drift
     report exists is drift computed by extract (24), so here instead:
     curated `sources` entries whose `path` no longer exists in the repo
     (cheap existence check, no scan).
3. Output: human = findings grouped errors-then-warnings, each
   `«level» code  message (where)`, then summary line
   `validate: E errors, W warnings`. `--json` data:
   `{errors: Finding[], warnings: Finding[]}`.
4. Exit code: 3 if ≥1 error; 0 otherwise (warnings alone don't fail); design
   §10 exit table.
5. `--strict` flag: warnings also cause exit 3 (for CI users who want it).

## Acceptance Criteria

- [ ] Fixture-driven tests produce each finding code at least once and assert exact codes/paths (build tiny stores in temp dirs).
- [ ] Clean fixture ⇒ exit 0, `0 errors, 0 warnings`.
- [ ] V1 test covers all three pairings (cand∩curated via alias, cand∩rejected, curated∩rejected).
- [ ] `--strict` flips warning-only run to exit 3.
- [ ] `--json` snapshot frozen.

## Validation

runCli integration tests; run against `fixtures/repo-ja-mixed` glossary once
issue 13 lands (green).

## Dependencies

06, 09, 10, 11.

## Non-goals

Auto-fixing; drift-vs-repo-content analysis (24's report); prose linting (v2).

## Design References

DESIGN.md §8 (invariants), §10.8, §13-B2 (schema findings as errors).
