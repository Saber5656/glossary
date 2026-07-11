# 22 — Extractor E4: definitions already present in docs

## Title

`src/extract/e4-doc-definitions.ts`: harvest existing definition sentences/structures

## Summary

Implement extractor E4 per DESIGN.md §9.4-E4: mine explicit definitions from
structure facts (15) and JA/EN definition sentences, emitting candidates with
`suggestedDefinition` (source `doc`).

## Context

R1 includes "definitions already written in docs" — the highest-precision
extractor and the seed for suggestedDefinition. A single explicit definition
is significant even at one occurrence (exempt from minOccurrences, §9.6).

## Scope

- E4 module + tests.

## Detailed Requirements

1. Signature:
   `extractDocDefinitions(inputs: {blocks: PositionedText[], facts: StructureFact[]}): RawCandidateWithDef[]`
   where RawCandidateWithDef = RawCandidate + `{definition: string}`
   (kind `doc-defined`).
2. Sources, in priority order (first hit per key wins; later duplicates still
   emit occurrences but merge keeps the first definition by this order):
   1. **definitionList fact** → term=fact.term, definition=fact.definition.
   2. **table fact** whose headers match: header[0] ∈ {用語, term, Term, 名称}
      AND header[1] ∈ {説明, 定義, description, Description, definition,
      Definition} → each row: term=cell[0], definition=cell[1]. Tables with
      other headers ignored.
   3. **boldLead fact** → term/rest.
   4. **JA sentence patterns** over prose blocks (linear regexes, applied per
      sentence after splitting on `。`):
      - `「?([^「」\n]{2,40})」?とは、?(.{8,300}?)(?:である|です|を指す|のこと)?$`
        — capture term + definition body. Require the definition part ≥ 8 chars
        to avoid fragments.
   5. **EN sentence patterns**: `^([A-Z][A-Za-z0-9 -]{2,60}) (?:is|are) defined as (.{8,300})$`
      and `^([A-Z][A-Za-z0-9 -]{2,60}) refers to (.{8,300})$`.
   6. **headingSection fact** where file path matches `/(glossary|用語)/i` →
      term=heading text, definition=firstParagraph.
3. Cleaning: definitions trimmed, internal newlines collapsed to spaces,
   truncated at 500 chars on a char boundary with `…` appended; control chars
   stripped. Terms pass normalizeSurface; empty/`> 80 chars` terms dropped.
4. Emit one candidate per finding: source={path,line of the fact/sentence},
   snippet = first 200 chars of definition, definition = full cleaned text.
5. No numeric scoring: E4 sets `score = 5.0` flat (merge in 23 takes max and
   adds multi-extractor bonus; doc-defined kind priority already dominates).
6. Determinism: sorted output (surface, path, line).

## Acceptance Criteria

- [ ] repo-ja-mixed yields: 支払予約 (とは-sentence, definition text exact), 与信枠+締め処理 (table rows), 売上確定 (boldLead), 締め処理 headingSection ALSO found in glossary.md (duplicate key — both emitted; priority order test asserts table definition wins later in 23's test, here both present).
- [ ] Non-matching table (other headers) ignored.
- [ ] とは-pattern: fragment shorter than 8 chars rejected; 300-char body captured; sentence without 。 terminator still matched at block end.
- [ ] Hostile xss.md strings pass through as raw text in definitions (no crash) — escaping is downstream's job.
- [ ] Determinism double-run; regex linearity comments present.

## Validation

Unit tests; PR includes extracted (term, definition) table from fixtures.

## Dependencies

08, 15; fixtures 13.

## Non-goals

Cross-file definition merging policy (23), Markdown rendering of definitions
(site treats them as plain text), English glossary-file heading harvest beyond
the path regex.

## Design References

DESIGN.md §9.4-E4, §9.6 (min-occurrence exemption, definition priority).
