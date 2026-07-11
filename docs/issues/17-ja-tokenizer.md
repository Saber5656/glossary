# 17 — Japanese tokenizer wrapper (kuromoji) with graceful fallback

## Title

`src/tokenize/ja.ts`: `JaTokenizer` interface, kuromoji-backed default, degraded mode

## Summary

Wrap `kuromoji` behind the `JaTokenizer` interface per DESIGN.md §9.3 and
ADR-003: lazy singleton, POS-mapped tokens, and a documented failure path that
lets extraction continue heuristics-only.

## Context

kuromoji is unmaintained (known unknown U1) — the wrapper is the isolation
layer that keeps that risk contained to one file. E1 (issue 19) is the only
consumer.

## Scope

- Interface + implementation + noun-run helper + tests. Includes the U1
  spike: verify install/load on Node 22/24/26 in CI matrix (already runs).

## Detailed Requirements

1. Types:
   ```ts
   interface JaToken { surface: string; pos: JaPos; posDetail: string; reading?: string }
   type JaPos = 'noun' | 'prefix' | 'suffix' | 'verb' | 'adjective' | 'particle' | 'symbol' | 'other'
   interface JaTokenizer { tokenize(text: string): JaToken[] }
   ```
   POS mapping from kuromoji `pos`/`pos_detail_1` (both are Japanese strings):
   `名詞→noun`, `接頭詞→prefix`, `接尾→suffix` (名詞,接尾 → suffix),
   `動詞→verb`, `形容詞→adjective`, `助詞/助動詞→particle`, `記号→symbol`,
   else `other`. Keep raw `pos_detail_1` in `posDetail` (E1 filters 数/非自立/
   代名詞).
2. `getJaTokenizer(logger: Pick<Logger, 'warn'>, opts?: {dicPathOverride?: string}): Promise<JaTokenizer | null>`:
   - Lazy process-wide singleton. Dictionary path resolution (exact):
     `createRequire(import.meta.url)` →
     `require.resolve('kuromoji/package.json')` → `join(dirname(that),
     'dict')`; check the directory exists BEFORE building; `dicPathOverride`
     (test seam) replaces it.
   - Build failure (missing dict, init error): warn ONCE with stable code
     prefix `W_JA_TOKENIZER_UNAVAILABLE: <error message only, no stack>` —
     `falling back to heuristics-only extraction`; memoize `null`, never
     retry within the process.
   - `resetJaTokenizerForTests()` exported from a test-helper entry (clears
     the memo; not part of the production API surface).
3. `readings`: expose kuromoji reading (katakana) when present.
4. `nounRuns(tokens: JaToken[]): JaToken[][]` helper — maximal runs of
   consecutive tokens with pos ∈ {noun, prefix, suffix}, splitting on
   posDetail `数` (numeral) and `非自立`; exported for E1.
5. Input pre-split (internal to `tokenize()`): inputs ≤ 10_000 chars pass
   through whole. Longer inputs split into chunks at the LAST `。` or newline
   before each 10_000-char boundary; a segment with neither delimiter is
   hard-split at exactly 10_000. Chunks tokenize independently and the token
   arrays concatenate in order — callers never see the splitting.
6. Determinism: same input ⇒ same tokens (kuromoji is deterministic; assert in
   test by double-run).

## Acceptance Criteria

- [ ] `tokenize('支払予約を作成する')` yields 支払/予約 as consecutive `noun` tokens and を as particle (exact assertion).
- [ ] `nounRuns` on `与信枠の上限` → runs [[与信,枠],[上限]].
- [ ] POS mapping table test over the fixture sentence `ご担当者が高い安全性を確認する。` plus `ああ、` asserting mapped JaPos per token: ご→prefix, 担当→noun, 者→suffix, が→particle, 高い→adjective, 安全→noun, 性→suffix, を→particle, 確認→noun, する→verb, 。→symbol, ああ→other. (Assert the MAPPED enum, not raw kuromoji strings — dictionary detail may vary.)
- [ ] Failure path: `dicPathOverride` at a bogus dir → returns null, warns once with `W_JA_TOKENIZER_UNAVAILABLE`, second call returns memoized null without re-warning; `resetJaTokenizerForTests()` restores a working build.
- [ ] Long-input test: a 25_000-char JA text tokenizes; token concatenation equals tokenizing the three chunks separately.
- [ ] Double-run determinism on the fixture README text.
- [ ] Test suite green on Node 22/24/26 via the standard CI matrix (issue 02 — infrastructural, not a dependency edge); attach the run link in the PR (resolves U1 for these versions).

## Validation

`npm run test`; paste tokenizer smoke output for one fixture sentence in PR.

## Dependencies

01, 03.

## Non-goals

lindera/Sudachi backends (ADR-003 revisit triggers), reading auto-fill into
curated terms (v2), tokenizing English text (E2 path). IPADIC/kuromoji
license notices are issue 40's NOTICE.md deliverable (cross-referenced
there), not this issue's.

## Design References

DESIGN.md §9.3; ADR-003; research/japanese-term-extraction.md; U1 (§3.4).
