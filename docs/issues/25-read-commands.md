# 25 — `glossary list` / `show` / `status`

## Title

Read-only inspection commands: list, show, status

## Summary

Implement the three read-only commands per DESIGN.md §10.3–10.5 over the three
stores, with stable human tables and frozen `--json` shapes.

## Context

Curation UX depends on being able to see candidates with evidence before
approving; these commands are the primary human interface to the data.

## Scope

- `src/cli/commands/{list,show,status}.ts` + tests.

## Detailed Requirements

1. `list`:
   - Flags: `--status candidate|curated|rejected` (default `candidate`),
     `--kind domain|code|abbreviation|doc-defined`, `--limit N` (default 50,
     0 = all), `--json`.
   - candidate rows: `KEY  KIND  SCORE  OCC  SUGGESTED?` (SUGGESTED? = `doc`/
     `llm`/`-`), ordered as stored (score desc). curated rows:
     `ID  TERM  KIND  TAGS  DEF?` ordered by id. rejected rows:
     `KEY  REASON  DATE` ordered by key.
   - Human output: aligned columns (pad by display width — use simple
     2-cells-per-CJK-char width function; no dependency), header line, footer
     `N shown / M total`.
   - `--json` data: `{status, total, items: [...]}` where items are the raw
     store records (candidate/term/rejected schemas).
2. `show <key-or-id>`:
   - Resolution order: candidates key (after termKey-normalizing input) →
     curated id → curated term/alias key → rejected key. Not found ⇒
     UsageError E_KEY_NOT_FOUND listing the closest 3 keys by prefix match.
   - Candidate view: all fields + each source as `path:line  snippet`.
   - Curated view: full record; definition printed verbatim; `definitionSource
     llm` renders a `[LLM draft — review required]` marker line.
   - `--json`: `{status: 'candidate'|'curated'|'rejected', record: <raw>}`.
3. `status`:
   - Counts: candidates (by kind), curated (by kind; +definition-pending
     count; +llm-definition count), rejected total; `lastExtract` from
     candidates.yaml generatedAt (null if absent); tokenizer availability NOT
     probed here (extract-time info only).
   - Human: small fixed table; `--json` frozen:
     `{candidates: {total, byKind}, curated: {total, byKind, pendingDefinition, llmDefinition}, rejected: n, lastExtract: iso|null}`.
4. All three commands are strictly read-only (fs-spy test: zero writes).

## Acceptance Criteria

- [ ] Fixture-driven runCli tests for each command × human + json snapshots.
- [ ] JA alignment: list output columns align for rows mixing 支払予約 and `slo` (width function tested).
- [ ] show resolves by candidate key, curated id, alias, rejected key (4 tests); not-found suggests prefix matches; JA input normalizes (`show 「支払予約」` works).
- [ ] `--limit 0` returns all; default caps at 50 with correct footer.
- [ ] Read-only fs-spy passes.

## Validation

runCli tests on a prepared temp glossary (built by store APIs in test setup).

## Dependencies

06, 09, 10, 11.

## Non-goals

Interactive TUI, fuzzy search (prefix suggestion only), pagination beyond
--limit.

## Design References

DESIGN.md §10.3–10.5, §7.3–7.5 (record shapes), §14 (output conventions).
