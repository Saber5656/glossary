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
     0 = all), `--json`. `--kind` combined with `--status rejected` ⇒
     `UsageError(E_USAGE, '--kind cannot be used with --status rejected')`
     (rejected entries have no kind).
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
   - Resolution order (DESIGN §10.4): candidates key (after
     termKey-normalizing input) → curated id → curated term/alias key →
     rejected key. Not found ⇒ UsageError E_KEY_NOT_FOUND with suggestions:
     the first 3 keys (codepoint-sorted) from the same resolution universe
     whose key starts with the normalized input; empty when none match.
   - Candidate view: all fields + each source as `path:line  snippet`.
   - Curated view: full record; definition printed verbatim; `definitionSource
     llm` renders a `[LLM draft — review required]` marker line.
   - `--json`: `{status: 'candidate'|'curated'|'rejected', record: <raw>}`.
3. `status`:
   - Counts: candidates (by kind), curated (by kind; +definition-pending
     count; +llm-definition count), rejected total; `lastExtract` from
     candidates.yaml generatedAt (null if absent); tokenizer availability NOT
     probed here (extract-time info only).
   - Human output: exactly these rows in this order (label left, value
     right): `candidates` (total), one indented row per kind present,
     `curated` (total), per-kind rows, `  pending definition`,
     `  llm definitions`, `rejected`, `last extract` (ISO or `-`).
   - `--json` frozen:
     `{candidates: {total, byKind}, curated: {total, byKind, pendingDefinition, llmDefinition}, rejected: n, lastExtract: iso|null}`.
4. Store-failure behavior (B2): candidates/rejected schema failures surface
   their store errors (`E_SCHEMA_INVALID` / `E_YAML_*`, exit 1); invalid
   curated term files are SKIPPED with one warning each (from
   readAllTerms problems) — warnings ride the envelope in `--json`.
5. All three commands are strictly read-only (fs-spy test: zero writes).

## Acceptance Criteria

- [ ] Fixture-driven runCli tests for each command × human + json snapshots.
- [ ] JA alignment: list output columns align for rows mixing 支払予約 and `slo` (width function tested).
- [ ] show resolves by candidate key, curated id, alias, rejected key (4 tests); not-found suggestion rule asserted (prefix matches, sorted, ≤3, empty case); JA input normalizes (`show 「支払予約」` works).
- [ ] `--limit 0` returns all; default caps at 50 with correct footer; `--kind` with `--status rejected` exits 2 with E_USAGE.
- [ ] Broken curated file ⇒ list/status still succeed with a warning in the envelope; broken candidates.yaml ⇒ exit 1 with store error.
- [ ] Read-only fs-spy passes.

## Validation

runCli tests on a prepared temp glossary (built by store APIs in test setup).

## Dependencies

06, 09, 10, 11.

## Non-goals

Interactive TUI, fuzzy search (prefix suggestion only), pagination beyond
--limit.

## Design References

DESIGN.md §10 (command table + envelope), §7.3–7.5 (record shapes), §14
(errors/logging); issue 06 (JSON envelope convention).
