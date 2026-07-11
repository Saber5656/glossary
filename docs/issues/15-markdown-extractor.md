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

1. `extractDocContent(relPath, text): DocContent` with
   `DocContent = {blocks: TextBlock[], facts: StructureFact[]}`;
   `TextBlock = {text, startLine, kind: 'prose'|'heading'|'code-in-doc'}`.
2. Markdown parsing: `unified().use(remarkParse)`; walk mdast:
   - paragraph/blockquote/listItem text → `prose` blocks
     (mdast-util-to-string; keep one block per paragraph/list item;
     `startLine` from node.position).
   - heading → `heading` block `{depth}` retained in fact (see 3).
   - fenced/indented code → `code-in-doc` block (raw value) — E1 skips these;
     E3 may use them.
   - inline code stays inside its paragraph text as-is.
   - html nodes: keep RAW value as prose text (do not parse; extractors treat
     as text — hostile fixture relies on this).
3. `StructureFact` variants (all with `{path via caller, line}`):
   - `{type: 'definitionList', term, definition}` — Markdown doesn't have
     native def-lists in CommonMark; recognize the common pattern:
     paragraph line `X`, next line starting `: ` (remark parses as paragraph
     text with `\n: ` — detect within paragraph text by regex
     `/^(.{1,80})\n:\s+(.+)$/s`).
   - `{type: 'table', headers: string[], rows: string[][]}` — remark-parse
     WITHOUT gfm does not parse tables; ADD dependency decision: use
     `remark-gfm`?  **No new dependency without ADR** — instead detect pipe
     tables lexically: contiguous lines matching `/^\|.+\|$/` where line 2
     matches `/^\|[\s:-|]+\|$/`; split cells on unescaped `|`, trim.
   - `{type: 'boldLead', term, rest}` — list item text matching
     `/^\*\*(.{1,80}?)\*\*\s*[:：]\s*(.+)/s` (term = captured 1).
   - `{type: 'headingSection', heading, depth, firstParagraph}` — heading
     followed by its first prose block.
4. `.txt`: whole file split into paragraphs on blank lines; all prose, line
   numbers computed.
5. Line length guard: any single line > 2000 chars is truncated to 2000 for
   block text (B2' regex safety), with warning flag on the block.
6. Pure function; no filesystem access (caller reads file via scanner list).

## Acceptance Criteria

- [ ] Fixture-driven tests on `repo-ja-mixed` docs: billing.md yields the 用語/説明 table fact with 2 rows; the bold-lead fact for 売上確定; def-list fact; fenced block emitted as code-in-doc and its text absent from prose blocks; glossary.md yields headingSection for 締め処理.
- [ ] Positions: assert exact startLine for ≥3 blocks against the fixture file.
- [ ] Hostile xss.md: html/script content appears verbatim as prose text (no crash, no interpretation).
- [ ] 2500-char line truncated to 2000 with warning.
- [ ] mdx file parses via the same path (plain remark-parse tolerates it; jsx lines become prose/raw — assert no crash on a small mdx sample).

## Validation

`npm run test`; snapshot of DocContent for billing.md committed (readable
review artifact).

## Dependencies

01, 03; fixtures 13.

## Non-goals

GFM plugin adoption (lexical table detection suffices; revisit only via ADR),
E4 pattern semantics (issue 22 consumes facts), frontmatter parsing (treated
as prose; harmless).

## Design References

DESIGN.md §9.2 (doc bullet), §9.4-E4 inputs, §13-B2' (line cap).
