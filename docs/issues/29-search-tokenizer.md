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

1. `searchTokenize(text: string): string[]` pipeline:
   1. NFKC normalize; lowercase (full string — JA unaffected).
   2. Segment into script runs: `latin` (`[a-z0-9]+` after lowering, including
      digits), `cjk` (`\p{Script=Han}|\p{Script=Hiragana}|\p{Script=Katakana}|ー`),
      other chars are separators.
   3. Latin runs: emit the run itself; additionally, if the ORIGINAL text at
      that position was a mixed-case/underscore identifier, callers pre-split —
      here simply also emit `splitIdentifier`-style sub-tokens when the run
      contains digits attached to letters (`utf8` → `utf8`, `utf`, `8`).
      (Import `splitIdentifier` from 18 — allowed: it is pure/isomorphic.)
   4. CJK runs: emit character bigrams (sliding window, step 1); a run of
      length 1 emits the unigram; **katakana runs additionally emit the whole
      run** as one token (research §3).
   5. Deduplicate preserving first-occurrence order.
2. Token cap: at most 512 tokens per input (defensive; longer inputs
   truncated with no error).
3. Isomorphism: no imports besides 18's module; no `node:*`; works under
   `es2023` browser target (regex `u`/`v` flags OK; Unicode property escapes
   OK in all target browsers/Node — document baseline: Node ≥22, evergreen
   browsers 2024+).
4. Export also `SEARCH_FIELD_WEIGHTS = {term: 3, aliases: 2, reading: 2,
   definition: 1, tags: 1}` (single source for 30/32) and
   `searchOptions()` returning the MiniSearch options object
   (`{tokenize: searchTokenize, searchOptions: {prefix: true, boost: SEARCH_FIELD_WEIGHTS, combineWith: 'AND'}}`)
   so builder and client cannot diverge.

## Acceptance Criteria

- [ ] `searchTokenize('支払予約')` → ['支払','払予','予約'] (order preserved).
- [ ] `searchTokenize('オーソリ')` → ['オーソリ','オー','ーソ','ソリ'] (whole-run token first or documented order — freeze it).
- [ ] `searchTokenize('Payment Reservation')` → ['payment','reservation'].
- [ ] `searchTokenize('ＳＬＯとは')` → NFKC folds to ['slo','とは'-bigrams…] (assert 'slo' present).
- [ ] Mixed `支払予約API` yields both CJK bigrams and 'api'.
- [ ] 1-char CJK input → unigram; empty/whitespace → [].
- [ ] Dedup and 512-cap tested; module import graph contains no node builtins (lint rule or unit assertion via source scan).

## Validation

`npm run test`; quick browser check happens in 32.

## Dependencies

01, 08 (normalize reuse if applicable), 18.

## Non-goals

Morphological search tokens (v2 via Intl.Segmenter — research §5), stemming,
synonym expansion at query time.

## Design References

ADR-004; research/client-side-search.md §3; DESIGN.md §9.3, §11.3.
