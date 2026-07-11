# 08 — Normalization and term identity module

## Title

`src/store/normalize.ts`: `normalizeSurface`, `termKey`, `termId`, slug validation

## Summary

Implement the single term-identity contract of DESIGN.md §7.1. Every component
that compares, deduplicates, or files terms MUST import from this module.

## Context

Term identity spans candidates, curated terms/aliases, and the rejected list
(state machine §8, invariant V1). A second, divergent implementation anywhere
would silently corrupt lifecycle behavior — hence one module, exhaustively
tested, frozen early.

## Scope

- The module + unit tests. No store logic.

## Detailed Requirements

1. `normalizeSurface(s: string): string` — exact pipeline:
   1. Unicode NFKC (`s.normalize('NFKC')`).
   2. Replace all Unicode whitespace runs (`/\s+/gu`) with one ASCII space;
      trim.
   3. Lowercase via Unicode `toLowerCase()` — locale-independent, no locale
      argument; Japanese is unaffected, other cased scripts also lower
      (this IS the frozen contract; DESIGN §7.1 step 3 states the same).
   4. Strip ONE balanced surrounding pair if the whole string is wrapped by
      it: `「」`, `『』`, `()`, `（）`, `""`, `''`. (Loop once, not recursive.)
   5. Return result (may be empty string — callers treat empty as invalid).
2. `termKey(surface: string): string` — `normalizeSurface` then replace each
   space with `-`. Property: `termKey(termKey(x)) === termKey(x)`.
3. `termId(key: string): string` — `'t-' + sha256hex(utf8(key)).slice(0, 8)`
   (node:crypto).
4. `isValidSlug(s: string): boolean` — `/^[a-z0-9][a-z0-9-]{0,63}$/`.
5. `compareCodepoint(a: string, b: string): -1 | 0 | 1` — compares the
   strings as sequences of Unicode code points (`Array.from(s)`, numeric
   `codePointAt(0)` comparison, first difference decides); when one sequence
   is a prefix of the other, the shorter sorts first; identical ⇒ 0. NOT
   locale collation and NOT UTF-16 code-unit order (they differ for
   supplementary-plane chars). Determinism per DESIGN §10.9; exported for all
   sorted outputs.
6. Document in the module header: examples table (from DESIGN §7.1) —
   `支払予約` → key `支払予約`; `Payment  Reservation` → `payment-reservation`;
   `ＳＬＯ` → `slo`; `「与信枠」` → `与信枠`.

## Acceptance Criteria

- [ ] Table-driven tests covering: NFKC width folding (full-width ASCII, half-width kana → full-width), whitespace collapse (tabs, ideographic space U+3000), Latin lowering, each bracket pair, non-wrapping brackets untouched (`支払(仮)` keeps inner parens), empty result, idempotence property (fuzz 200 random strings: `termKey∘termKey = termKey`).
- [ ] `termId('支払予約') === 't-50db70e3'` (precomputed constant; sha256 of the UTF-8 NFKC key, first 8 hex).
- [ ] `compareCodepoint` pairwise assertions: `('a','ん') < 0`, `('ん','支') < 0`, `('支','a') > 0`, `('abc','abc') === 0`, `('ab','abc') < 0` (prefix → shorter first), and the supplementary-plane discriminator `compareCodepoint('ｚ','𠮷') < 0` even though `'ｚ' > '𠮷'` under default UTF-16 string comparison.
- [ ] Module has zero imports besides `node:crypto`.

## Validation

`npm run test`. Include the examples table as executed assertions, not prose.

## Dependencies

01, 03 (error types only if needed — otherwise none).

## Non-goals

Romaji transliteration, locale collation, reading (よみ) derivation.

## Design References

DESIGN.md §7.1; §8 invariant V1; §10.9 (sort determinism); ADR-002 §4.
