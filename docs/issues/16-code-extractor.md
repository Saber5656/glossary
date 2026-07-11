# 16 — Code content extractor (comments, strings, declared identifiers)

## Title

`src/content/code.ts`: lexical extraction of comments, string literals, and declared identifiers per language family

## Summary

Implement DESIGN.md §9.2 code handling: a line-oriented lexical pass (NO AST)
producing comments, bounded string literals (recorded for diagnostics only in
v1), and declared identifiers with `declKind` — consumed by E2 (identifiers)
and E3 (comments + identifier acronym tokens) per DESIGN §9.4.

## Context

v1 is deliberately regex/lexical (non-goal: tree-sitter). The per-family
patterns below are the frozen contract — implementation agents must not
invent additional language support.

## Scope

- One module + language-family pattern tables + tests.

## Detailed Requirements

1. `extractCodeContent(relPath, text): CodeContent` (exported types; every
   record carries `path: relPath`):
   ```ts
   type CodeComment = { path: string; text: string; line: number }
   type CodeString  = { path: string; text: string; line: number }
   type DeclaredIdent = { path: string; name: string; line: number;
                          declKind: string; exported: boolean }
   type CodeWarning = { code: 'W_LINE_TRUNCATED'; path: string; line: number;
                        originalLength: number; cappedLength: number }
   type CodeContent = { comments: CodeComment[]; strings: CodeString[];
                        identifiers: DeclaredIdent[]; warnings: CodeWarning[] }
   ```
   Record granularity: line comments — one record per comment line (`line` =
   that line, `//`/`#` markers stripped, leading space trimmed); block
   comments — ONE record per block (`line` = start line, delimiters stripped,
   internal newlines preserved); Python docstrings — one record per
   docstring (`line` = opening-quotes line, quotes stripped). Strings are
   recorded for diagnostics/future use only — in v1, E2 consumes identifiers
   and E3 consumes comments+identifiers (DESIGN §9.4).
2. Family by extension:
   - `ts|tsx|js|jsx|mjs|cjs|mts|cts` → tsjs
   - `py` → python; `go` → go; `java|kt|kts` → jvm; `rb` → ruby;
     `rs` → rust; anything else → generic — comments only (`//`, `#`,
     `/* */` best-effort), ZERO strings and ZERO identifiers.
3. Lexical pass (single forward scan per file, line-aware):
   - Handle line comments, block comments, and single/double/backtick strings
     enough to not extract identifiers from inside them. Python docstrings
     count as comments; exactly these positions: the first statement of the
     file (module docstring) and the first statement after a `def`/`class`
     header line, allowing intervening blank lines only. Other triple-quoted
     strings are ordinary strings.
   - String literals: record only length 3–120 chars, after trimming;
     skip strings that look like paths/URLs (`^[./]|^https?:`).
   - Line length cap 2000 (truncate, warn) before regexes (B2').
4. Declaration patterns per family (declKind in parentheses) — applied only on
   code lines outside comments/strings; identifier = `[A-Za-z_$][A-Za-z0-9_$]*`:
   - tsjs: `class X`(class), `interface X`(interface), `type X =`(type),
     `enum X`(enum), `function X(`(function),
     `(?:const|let|var) X (?::[^=]+)?=`(const),
     method shorthand at line start `X(...) {` EXCLUDED (too noisy — do not
     implement). `exported` = line contains `export ` before the keyword —
     this covers `export default function X`, `export const X`, `export
     abstract class X`. Decorator/annotation lines above the declaration are
     NOT considered (documented limitation).
   - python: `def x(`(function), `class X`(class); `exported` = name not
     starting `_`.
   - go: `func X(`/`func (r T) X(`(function/method), `type X`(type);
     `exported` = first rune uppercase.
   - jvm: `class X`/`interface X`/`enum X`/`record X`; methods EXCLUDED;
     `exported` = the same line contains the word `public` before the keyword
     (`public final class X` counts; annotations on previous lines are not
     considered — documented limitation).
   - ruby: `def x`(function), `class X`/`module X`; exported=true.
   - rust: `fn x(`, `struct X`, `enum X`, `trait X`, `type X`;
     `exported` = line starts with `pub` followed by a space or `(` —
     `pub(crate)`/`pub(super)` count as exported.
5. All patterns MUST be linear-time (no nested unbounded quantifiers); add a
   comment-table in code listing each regex with a justification note.
6. Pure function; deterministic output order = source order.

## Acceptance Criteria

- [ ] Fixture tests (`repo-ja-mixed/src/**`): finds PaymentReservation(class, exported), CreditLimit(interface, exported), reservePayment(function, exported), AUTH_TIMEOUT_MS(const, exported), create_payment_reservation(python function, exported); JA comments captured with correct lines.
- [ ] Identifiers inside strings/comments NOT extracted (test with a comment mentioning `class FakeClass`).
- [ ] Docstring in routes.py lands in comments including the `SLO (Service Level Objective)` line.
- [ ] Generic family: a `.sh` file yields comments only, zero identifiers.
- [ ] Adversarial: 2000+-char line truncated with a W_LINE_TRUNCATED warning record; unterminated string/block comment does not hang or throw (scan ends at EOF).
- [ ] Performance: a deterministic synthetic 1 MiB TS file (repeated realistic declarations, committed generator inline in the test) processes within 3 s under vitest (CI-safe bound; local target < 1 s is informational only).

## Validation

`npm run test` incl. the perf bound (vitest, generous CI margin ×3).

## Dependencies

01, 03; fixtures 13.

## Non-goals

AST parsing (v2 tree-sitter), route-string/table-name mining (strings are
recorded; E2 uses identifiers only in v1), languages beyond the table.

## Design References

DESIGN.md §9.2 (code bullet), §3.2 (no AST), §9.4-E2 inputs, §13-B2'.
