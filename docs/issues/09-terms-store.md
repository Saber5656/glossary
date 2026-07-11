# 09 — Curated terms store (`terms/<id>.yaml`) and schema

## Title

`src/store/terms.ts`: curated term zod schema, read-all, write-one, id/key indexes

## Summary

Implement the human-owned curated tier per DESIGN.md §7.4: schema validation,
directory scanning, single-term writes (used by approve/draft), and lookup
indexes by id / term key / alias key.

## Context

Curated files are the product's most valuable data; the store must validate
strictly, never mass-rewrite, and expose the key indexes the lifecycle
invariants (V1–V3) and merge pipeline (23) depend on.

## Scope

- Schema + store functions + unit tests. No CLI (25/26), no validate rules
  (12) — but export the raw material they need.

## Detailed Requirements

1. Zod schema `TermFile` implementing DESIGN §7.4 exactly; `.strict()`:
   - `schemaVersion: literal(1)`; `id` must satisfy `isValidSlug` OR
     `^t-[0-9a-f]{8}$`; `term` non-empty string; `reading?` non-empty string;
     `aliases: string[]` (default [], each non-empty); `kind` enum
     `domain|code|abbreviation|doc-defined`; `definition: string` (may be "");
     `definitionSource` enum `human|llm|doc`; `tags: string[]` (default [],
     each matching `^[\p{L}\p{N}][\p{L}\p{N}-]{0,31}$u`); `relatedTerms:
     string[]` (ids, default []); `examples: {text: non-empty, source:
     non-empty}[]` (default []); `sources: {path, line}[]` (default []) where
     `path` is a non-empty relative POSIX path (no leading `/`, no `..`
     segment) and `line` a positive integer; `createdAt`/`updatedAt`
     `YYYY-MM-DD` strings; `notes: string | null` (default null).
2. `readAllTerms(glossaryDir): {terms: Term[], problems: Problem[]}`
   - `Problem = {file: string, code: 'E_YAML_INVALID'|'E_YAML_TOO_LARGE'|'E_SCHEMA_INVALID'|'E_ID_MISMATCH', message: string}`.
   - Scan `terms/*.yaml` (sorted by filename, compareCodepoint); parse via
     yaml-io (issue 04 — ALL reads/writes in this module go through it; no
     direct `yaml` import). EVERY yaml-io failure (parse, limits) and every
     zod failure becomes a `problems` entry with the matching code and the
     file is skipped (commands decide severity; `validate` errors, `list`
     warns).
   - Enforce `id === basename` (mismatch ⇒ `E_ID_MISMATCH` problem).
3. `writeTerm(glossaryDir, term: Term, opts?: {overwrite?: boolean})` —
   serialize with keys in the §7.4 order exactly; write via yaml-io atomic
   write to `terms/<id>.yaml`. Default: refuse with RuntimeError
   `E_ID_CONFLICT` if the file exists — regardless of content (human-owned
   tier). Replacement requires `opts.overwrite === true` (used by
   `draft --curated`, issue 37).
4. `buildTermIndexes(terms)` →
   `{byId: Map<string,Term>, byKey: Map<string,{term: Term, via: 'term'|'alias'}>, indexProblems: {key: string, firstId: string, secondId: string, via: 'term'|'alias'}[]}`
   where keys come from `termKey(term.term)` and every `termKey(alias)`.
   On duplicate keys, `byKey` keeps the FIRST entry (input order = sorted
   filenames) and each further collision appends an `indexProblems` record
   (validate V2 consumes; no resolution here).
5. `newTermFromCandidate(input, opts: {id?, tags?, clock})` — pure builder
   used by approve (26). To avoid a dependency on issue 10's module, the
   input is a LOCAL structural type defined here:
   `{surface: string, surfaces: string[], kind: Kind, sources: {path, line}[], suggestedDefinition: string|null, suggestedDefinitionSource: 'doc'|'llm'|null}`.
   Maps surface→term, `aliases` = surfaces − surface, evidence sources (≤5),
   `definition` = suggestedDefinition ?? '', `definitionSource` =
   suggestedDefinitionSource ?? 'human', dates from clock.

## Acceptance Criteria

- [ ] Schema tests: every field's happy/violating case incl. slug rules, tag pattern, date format, strictness (extra key rejected), absolute/`..` source paths rejected, zero/negative line rejected.
- [ ] readAllTerms on a fixture dir: valid + zod-invalid + oversized + yaml-broken + mismatched-id files → each lands in `problems` with the correct code; order deterministic.
- [ ] writeTerm: byte-golden for a JA sample (multiline definition uses `|` block, §7.4 key order); existing file refused with `E_ID_CONFLICT`; `overwrite: true` replaces it.
- [ ] Indexes: term+aliases all resolve; duplicate alias across two terms → `byKey` keeps first, `indexProblems` records `{key, firstId, secondId, via}` exactly.
- [ ] newTermFromCandidate: llm-suggested definition carries `definitionSource: llm`; empty suggestion ⇒ `''`+`human`; aliases exclude the surface.

## Validation

`npm run test`; golden file committed under the test dir.

## Dependencies

04, 08.

## Non-goals

approve/reject CLI (26), invariant aggregation (12), any mass rewrite of
terms/ (forbidden by ADR-002 §2).

## Design References

DESIGN.md §7.4, §8 (T2/T9 effects), §6 ownership table; ADR-002.
