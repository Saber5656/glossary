# 29 — Shared search tokenizer (CJK bigram + Latin words)

## Title

`src/tokenize/search.ts`: one tokenizer for index build (Node) and query (browser)

## Summary

Implement the search tokenization strategy of
docs/research/client-side-search.md §3 as a single dependency-free module
compiled into both the site builder and the browser bundle.

## Context

Index/query agreement is the correctness condition for client-side search
(ADR-004). The module must be isomorphic (no Node APIs) so esbuild can include
it in the client.

## Scope

- One pure module + tests. Consumed by 30 (index build) and 32 (client).

## Detailed Requirements

1. `searchTokenize(text: string): string[]` pipeline (ORDER MATTERS —
   identifier splitting sees the pre-lowercase original):
   0. Input cap: slice input to its first 10_000 chars before ANY work
      (browser-side DoS guard).
   1. NFKC normalize (case preserved at this point).
   2. Segment into script runs by class: `latin` = `[A-Za-z0-9]+`;
      `katakana` = `[\p{Script=Katakana}ー]+`; `han` = `\p{Script=Han}+`;
      `hiragana` = `\p{Script=Hiragana}+`; every other char is a separator.
   3. Latin runs: compute `subs = splitIdentifier(run)` (issue 18 — pure,
      isomorphic, sees the original casing). Emit the lowercased whole run
      when `subs.length > 1`, then every sub-token. A single-token run emits
      just that token.
   4. katakana runs: emit the whole run FIRST, then its character bigrams
      (a length-1/2 run's bigram set may equal the run — dedup handles it).
      han / hiragana runs: emit character bigrams (length-1 run → unigram).
   5. Deduplicate preserving first-occurrence order; cap output at 512
      tokens.
   Frozen examples (each an exact-array test):
   - `PaymentReservation` → `['paymentreservation','payment','reservation']`
   - `AUTH_TIMEOUT_MS` → `['auth','timeout','ms']`
   - `utf8Decoder` → `['utf8decoder','utf','8','decoder']`
   - `支払予約API` → `['支払','払予','予約','api']`
   - `オーソリ予約` → `['オーソリ','オー','ーソ','ソリ','予約']`
   - `予約する` → `['予約','する']`
   - `ＳＬＯとは` → `['slo','とは']`
2. Isomorphism: no imports besides 18's module; no `node:*`; works under
   `es2023` browser target (Unicode property escapes OK — baseline: Node
   ≥22, evergreen browsers 2024+).
3. Exports (single source for 30/32):
   ```ts
   export const SEARCH_FIELD_WEIGHTS =
     { term: 3, aliasesText: 2, reading: 2, definition: 1, tagsText: 1 }
   export function searchTokenize(text: string): string[]
   export function miniSearchOptions(): {
     tokenize: typeof searchTokenize,
     searchOptions: { prefix: true, boost: typeof SEARCH_FIELD_WEIGHTS, combineWith: 'AND' }
   }
   ```
   Issue 30 adds `fields`/`storeFields`/`idField` when CONSTRUCTING
   MiniSearch — they are index-shape concerns, not tokenizer concerns.

## Acceptance Criteria

- [ ] Every frozen example above asserted as an exact array (7 tests).
- [ ] `searchTokenize('支払予約')` → `['支払','払予','予約']`; 1-char CJK input → unigram; empty/whitespace → `[]`.
- [ ] Dedup, 512-token cap, and the 10_000-char input cap tested (a 100k-char input completes < 50 ms and is truncated).
- [ ] Module import graph contains no node builtins (unit assertion via source scan).

## Validation

`npm run test`; quick browser check happens in 32.

## Dependencies

01, 08, 18 (identifier splitting — this module is NOT dependency-free; it is
node-builtin-free).

## Non-goals

Morphological search tokens (v2 via Intl.Segmenter — research §5), stemming,
synonym expansion at query time.

## Design References

ADR-004; research/client-side-search.md §3; DESIGN.md §9.3, §11.3.
