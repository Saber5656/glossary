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
     `^t-[0-9a-f]{8}$`; `term` non-empty string; `reading?` string;
     `aliases: string[]` (default []); `kind` enum
     `domain|code|abbreviation|doc-defined`; `definition: string` (may be "");
     `definitionSource` enum `human|llm|doc`; `tags: string[]` (default [],
     each matching `^[\p{L}\p{N}][\p{L}\p{N}-]{0,31}$u`); `relatedTerms:
     string[]` (ids); `examples: {text, source}[]` (default []); `sources:
     {path, line}[]` (default []); `createdAt`/`updatedAt` `YYYY-MM-DD`
     strings; `notes: string | null` (default null).
2. `readAllTerms(glossaryDir): {terms: Term[], problems: Problem[]}`
   - Scan `terms/*.yaml` (sorted by filename, compareCodepoint); parse via
     yaml-io; schema failures collect into `problems` `{file, message}` and
     skip the file (commands decide severity; `validate` errors, `list` warns).
   - Enforce `id === basename` (mismatch ⇒ problem).
3. `writeTerm(glossaryDir, term: Term)` — serialize with keys in the §7.4
   order exactly; write via yaml-io atomic write to `terms/<id>.yaml`.
   Refuse (RuntimeError `E_ID_CONFLICT`) if file exists with different `term`
   unless `opts.overwrite` (draft/approve pass explicit intent).
4. `buildTermIndexes(terms)` →
   `{byId: Map<string,Term>, byKey: Map<string,{term: Term, via: 'term'|'alias'}>}`
   where keys come from `termKey(term.term)` and every `termKey(alias)`.
   Duplicate key across files is NOT resolved here — collect into
   `indexProblems` (validate V2 consumes).
5. `newTermFromCandidate(candidate, opts: {id?, tags?, clock})` — pure builder
   used by approve (26): maps surface→term, evidence sources (≤5), sets
   `definition` from `suggestedDefinition ?? ''`, `definitionSource` from
   `suggestedDefinitionSource ?? 'human'`, dates from clock.

## Acceptance Criteria

- [ ] Schema tests: every field's happy/violating case incl. slug rules, tag pattern, date format, strictness (extra key rejected).
- [ ] readAllTerms on a fixture dir: valid+invalid+mismatched-id files → correct terms/problems split; order deterministic.
- [ ] writeTerm output byte-golden for a JA sample (multiline definition uses `|` block); key order matches §7.4 listing.
- [ ] Indexes: term+aliases all resolve; duplicate alias across two terms lands in indexProblems.
- [ ] newTermFromCandidate: llm-suggested definition carries `definitionSource: llm`; empty suggestion ⇒ `''`+`human`.

## Validation

`npm run test`; golden file committed under the test dir.

## Dependencies

04, 08.

## Non-goals

approve/reject CLI (26), invariant aggregation (12), any mass rewrite of
terms/ (forbidden by ADR-002 §2).

## Design References

DESIGN.md §7.4, §8 (T2/T9 effects), §6 ownership table; ADR-002.
