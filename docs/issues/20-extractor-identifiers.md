# 20 — Extractor E2: code identifiers and API names

## Title

`src/extract/e2-identifiers.ts`: exported-declaration phrases with cross-file scoring

## Summary

Implement extractor E2 per DESIGN.md §9.4-E2: turn declared identifiers (16)
into human-readable phrase candidates, scored by spread and frequency, with a
noise stoplist.

## Context

Identifier terms anchor the glossary to the codebase (R1). The signal is
"declared once, referenced widely" — spread across files beats raw counts.

## Scope

- E2 module + built-in EN stoplist + tests.

## Detailed Requirements

1. Signature:
   `extractIdentifiers(inputs: {identifiers: DeclaredIdent[], fileTexts: Map<string, string>}, cfg: {minOccurrences: number, stopwords: Set<string>}): RawCandidate[]`
   — `DeclaredIdent` is issue 16's exported type (already carries `path`);
   `fileTexts` maps relPath → full text of every scanned CODE file (the
   pipeline, issue 24, provides it).
2. Candidate set: declarations with `exported === true`, name length ≥ 4,
   split tokens ≥ 2 OR declKind ∈ {class, interface, type, enum} (single-word
   class names like `Logger` allowed but face the stoplist).
3. Surface: `identifierPhrase(name)` (18) — e.g. `payment reservation`.
   Keep `name` as an additional surface variant (merge in 23 handles both via
   `surfaces`; emit RawCandidate.surface = phrase; put raw name into snippet
   context: snippet = declaration line text).
4. Occurrence counting: count REFERENCES of the raw identifier across all
   scanned code files. Scan PER LINE, each line capped at 2000 chars first
   (B2'), with identifier-boundary matching:
   `new RegExp('(^|[^A-Za-z0-9_$])' + escapeRegExp(name) + '(?=$|[^A-Za-z0-9_$])', 'g')`
   — `\b` is wrong for names with `$`. Comments included; the declaration
   line counts once.
5. Filters:
   - occurrences (references) < cfg.minOccurrences (default 3, §7.2) ⇒ drop.
   - test-file declarations dropped: path matches
     `/(^|\/)(test|tests|__tests__|spec)\//` or filename
     `*.test.*`/`*.spec.*`.
   - Stoplist on ANY single-token candidate and on phrases whose every token
     is stoplisted: builtin `src/extract/stopwords-en.ts` (≥ 60 entries:
     util, utils, helper, helpers, data, info, item, items, list, map, get,
     set, index, main, common, base, core, config, options, params, value,
     values, type, types, error, errors, handler, manager, service, client,
     server, request, response, result, results, test, mock, temp, tmp, node,
     app, application, default, new, old, name, id, key, string, number, …).
   - Digit-only tokens removed from phrase; if nothing remains, drop.
   - One-letter parts (§9.4-E2): drop any candidate whose phrase contains a
     single-letter alphabetic token after splitting (`XPayment` → [x,
     payment] ⇒ dropped).
6. Score: `distinctFiles * ln(1 + references)` rounded to 2 decimals, where
   distinctFiles = number of files containing a reference.
7. Emit one RawCandidate (kind `code`) per DECLARATION site (path/line of the
   declaration; snippet = trimmed declaration line ≤ 200 chars). Multiple
   declarations of the same phrase (overloads/re-exports) each emit.
8. Determinism: sort outputs by (surface, path, line).

## Acceptance Criteria

- [ ] repo-ja-mixed: emits `payment reservation` (class, ≥3 refs), `credit limit`, `reserve payment`, `auth timeout ms`, `create payment reservation` (python); does NOT emit `format date`/`logger` (stoplist/threshold per fixture design).
- [ ] Reference counting: crafted fixture — identifier referenced in 3 files ⇒ distinctFiles=3 asserted in score.
- [ ] Test-file declaration excluded (add `src/x.test.ts` decl in test fixture inline).
- [ ] Boundary matching: `$foo`, `foo$`, `a$b` counted literally at line starts/ends and mid-line; `PaymentReservationX` does NOT count as a reference of `PaymentReservation`.
- [ ] One-letter-part rule: `XPayment`-style declaration dropped.
- [ ] Adversarial: a 5000-char single-line file is capped before matching; counting completes < 100 ms.
- [ ] Determinism double-run.

## Validation

Unit tests incl. a committed vitest snapshot of the fixture top-10 (sorted
score desc, key asc).

## Dependencies

08, 16, 18; fixtures 13.

## Non-goals

Route-string/table-name mining from string literals (v2), non-exported/local
identifiers, rename tracking.

## Design References

DESIGN.md §9.4-E2; §7.2 (identifiers.minOccurrences); §13-B2'.
