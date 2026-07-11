# 22 — Extractor E4: definitions already present in docs

## Title

`src/extract/e4-doc-definitions.ts`: harvest existing definition sentences/structures

## Summary

Implement extractor E4 per DESIGN.md §9.4-E4: mine explicit definitions from
structure facts (15) and JA/EN definition sentences, emitting raws that carry
`definition` + `definitionKind`; merge (23) converts the best one per key
into `suggestedDefinition` with source `doc`.

## Context

R1 includes "definitions already written in docs" — the highest-precision
extractor and the seed for suggestedDefinition. A single explicit definition
is significant even at one occurrence (exempt from minOccurrences, §9.6).

## Scope

- E4 module + tests.

## Detailed Requirements

1. Signature:
   `extractDocDefinitions(inputs: {blocks: TextBlock[], facts: StructureFact[]}): RawCandidateWithDef[]`
   — issue 15's exported types. Sentence regexes (sources 4–5) run ONLY on
   `kind: 'prose'` blocks; facts drive sources 1–3 and 6.
   `RawCandidateWithDef = RawCandidate & {definition: string, definitionKind: 'definitionList'|'table'|'boldLead'|'jaSentence'|'enSentence'|'headingSection'}`
   (kind `doc-defined`). definitionKind priority = that listed order (DESIGN
   §9.4/§9.6). Line cap: every line truncated to 2000 chars before ANY E4
   regex (B2', defense in depth).
2. Sources (each finding tags its definitionKind; ALL findings emit — merge
   (23) selects the highest-priority definitionKind per key):
   1. **definitionList fact** → term=fact.term, definition=fact.definition.
   2. **table fact** — headers compared after NFKC + trim + Latin lowercase:
      header[0] ∈ {用語, term, 名称} AND header[1] ∈ {説明, 定義,
      description, definition}. Rows must have ≥ 2 cells (only cells 0/1
      used); rows whose cleaned term OR definition is empty are dropped;
      source line = the ROW's own line (15's table fact records per-row
      lines). Tables with other headers ignored.
   3. **boldLead fact** → term/rest.
   4. **JA sentence patterns** over prose blocks (linear regexes, applied per
      sentence after splitting on `。`):
      - `「?([^「」\n]{2,40})」?とは、?(.{8,300}?)(?:である|です|を指す|のこと)?$`
        — capture term + definition body. Require the definition part ≥ 8
        chars to avoid fragments.
      Sentence splitting (normative): split block text on `。` (delimiter
      stays with the left sentence) plus block end;
      `source.line = block.startLine + count of '\n' before the sentence's
      start offset`.
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

- [ ] repo-ja-mixed golden rows — assert exact (surface, definitionKind, path) tuples and that definition text starts with the fixture wording: 支払予約→jaSentence(README.md); 与信枠→table(docs/billing.md); 締め処理→table(docs/billing.md) AND →headingSection(docs/glossary.md) (both emitted; 23 selects table by priority); 売上確定→boldLead(docs/billing.md).
- [ ] Non-matching table (other headers) ignored.
- [ ] とは-pattern: fragment shorter than 8 chars rejected; 300-char body captured; sentence without 。 terminator still matched at block end.
- [ ] Hostile xss.md strings pass through as raw text in definitions (no crash) — escaping is downstream's job.
- [ ] Determinism double-run.
- [ ] Adversarial: a 5000-char prose line is capped at 2000 before regexes; all E4 patterns complete < 100 ms on worst-case 2000-char inputs (repeated 「, repeated とは).

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
