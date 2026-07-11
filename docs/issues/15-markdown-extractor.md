# 15 — Markdown content extractor (text blocks, structure facts, positions)

## Title

`src/content/markdown.ts`: mdast-based doc content extraction with line positions

## Summary

Parse doc files (md/mdx/txt) into positioned text blocks and structural facts
per DESIGN.md §9.2, feeding extractors E1/E3/E4.

## Context

E1 needs clean Japanese prose with line numbers; E4 needs structure
(definition lists, tables, bold-lead bullets, headings). Fenced code inside
docs must be separated so prose extractors don't eat code.

## Scope

- One module + tests. `.txt` handling included (trivial path).

## Detailed Requirements

1. `extractDocContent(relPath, text): DocContent` with (exported types):
   ```ts
   type TextBlock = { path: string; text: string; startLine: number;
                      kind: 'prose'|'heading'|'code-in-doc'; depth?: 1|2|3|4|5|6 }
     // path is always the relPath argument; depth present iff kind === 'heading'
   type StructureFact =
     | { type: 'definitionList'; path: string; line: number; term: string; definition: string }
     | { type: 'table'; path: string; line: number; headers: string[]; rows: {cells: string[]; line: number}[] }
     | { type: 'boldLead'; path: string; line: number; term: string; rest: string }
     | { type: 'headingSection'; path: string; line: number; heading: string; depth: number; firstParagraph: string }
   type DocWarning = { path: string; line: number; type: 'line-truncated' }
   type DocContent = { blocks: TextBlock[]; facts: StructureFact[]; warnings: DocWarning[] }
   ```
0. Line-cap precondition (B2'): BEFORE any regex or lexical detection, split
   the raw input into physical lines and cap each at 2000 chars (record a
   DocWarning per truncation); all block/fact detection operates on capped
   lines. (mdast parsing runs on the capped text.)
2. Markdown parsing: `unified().use(remarkParse)`; walk mdast:
   - paragraph/blockquote/listItem text → `prose` blocks
     (mdast-util-to-string; keep one block per paragraph/list item;
     `startLine` from node.position).
   - heading → `heading` block with `depth` set on the block itself.
   - fenced/indented code → `code-in-doc` block (raw value) — E1 skips these;
     E3 may use them.
   - inline code stays inside its paragraph text as-is.
   - html nodes: keep RAW value as prose text (do not parse; extractors treat
     as text — hostile fixture relies on this).
3. StructureFact detection rules (shapes are fixed by the union in Req 1):
   - definitionList — Markdown doesn't have
     native def-lists in CommonMark; recognize the common pattern:
     paragraph line `X`, next line starting `: ` (remark parses as paragraph
     text with `\n: ` — detect within paragraph text by regex
     `/^(.{1,80})\n:\s+(.+)$/s`).
   - table facts — remark-parse WITHOUT gfm does not parse tables; **no new
     dependency without ADR** — detect pipe tables lexically: contiguous
     lines matching `/^\|.+\|$/` where line 2 matches `/^\|[\s:\-|]+\|$/`.
     Cell rules: split on unescaped `|` (`\|` is preserved as a literal `|`),
     trim each cell; the separator row is excluded from `rows`; a row whose
     cell count differs from the header is SKIPPED; fact `line` = header
     line; each row records its own physical `line`.
   - boldLead — list item text matching
     `/^\*\*(.{1,80}?)\*\*\s*[:：]\s*(.+)/s` (term = captured 1).
   - headingSection — heading followed by its first prose block.
4. `.txt`: whole file split into paragraphs on blank lines; all prose, line
   numbers computed.
5. (Line-cap is requirement 0 above — a precondition, not a post-step.)
6. Pure function; no filesystem access (caller reads file via scanner list);
   warnings are returned in `DocContent.warnings`, never logged here.

## Acceptance Criteria

- [ ] Fixture-driven tests on `repo-ja-mixed` docs: billing.md yields the 用語/説明 table fact with 2 rows; the bold-lead fact for 売上確定; def-list fact; fenced block emitted as code-in-doc and its text absent from prose blocks; glossary.md yields headingSection for 締め処理.
- [ ] Positions: assert exact startLine for ≥3 blocks against the fixture file.
- [ ] Hostile xss.md: html/script content appears verbatim as prose text (no crash, no interpretation).
- [ ] 2500-char line truncated to 2000 with a `line-truncated` DocWarning carrying the right path/line; table/fact detection still works on the capped text.
- [ ] mdx file parses via the same path (plain remark-parse tolerates it; jsx lines become prose/raw — assert no crash on a small mdx sample).

## Validation

`npm run test`; committed snapshot at
`src/content/__snapshots__/billing-doc-content.json` (or vitest-managed
equivalent) containing `blocks`, `facts`, `warnings` with LF-normalized text —
the stable review artifact.

## Dependencies

01, 03; fixtures 13.

## Non-goals

GFM plugin adoption (lexical table detection suffices; revisit only via ADR),
E4 pattern semantics (issue 22 consumes facts), frontmatter parsing (treated
as prose; harmless).

## Design References

DESIGN.md §9.2 (doc bullet), §9.4-E4 inputs, §13-B2' (line cap).
