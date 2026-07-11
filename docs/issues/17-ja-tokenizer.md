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
2. `getJaTokenizer(logger): Promise<JaTokenizer | null>`:
   - Lazy process-wide singleton; builds kuromoji tokenizer with
     `dicPath: <resolved from require.resolve('kuromoji')>/…/dict`
     (resolve robustly via `createRequire(import.meta.url)`).
   - Build failure (missing dict, init error): log ONE warning
     `ja tokenizer unavailable (<reason>) — falling back to heuristics-only
     extraction`, memoize `null`, never retry within the process.
3. `readings`: expose kuromoji reading (katakana) when present.
4. `nounRuns(tokens: JaToken[]): JaToken[][]` helper — maximal runs of
   consecutive tokens with pos ∈ {noun, prefix, suffix}, splitting on
   posDetail `数` (numeral) and `非自立`; exported for E1.
5. Input pre-split: tokenize per text block; blocks > 10_000 chars split on
   sentence boundaries (`。`/newline) before tokenizing (memory bound).
6. Determinism: same input ⇒ same tokens (kuromoji is deterministic; assert in
   test by double-run).

## Acceptance Criteria

- [ ] `tokenize('支払予約を作成する')` yields 支払/予約 as consecutive `noun` tokens and を as particle (exact assertion).
- [ ] `nounRuns` on `与信枠の上限` → runs [[与信,枠],[上限]].
- [ ] POS mapping table tested for at least one token per JaPos value.
- [ ] Failure path: point dicPath at a bogus dir via test seam → returns null, warns once, second call returns memoized null without re-warning.
- [ ] Double-run determinism on the fixture README text.
- [ ] CI matrix (issue 02) green on 22/24/26 with kuromoji installed — attach run link (resolves U1 for these versions).

## Validation

`npm run test`; paste tokenizer smoke output for one fixture sentence in PR.

## Dependencies

01, 03.

## Non-goals

lindera/Sudachi backends (ADR-003 revisit triggers), reading auto-fill into
curated terms (v2), tokenizing English text (E2 path).

## Design References

DESIGN.md §9.3; ADR-003; research/japanese-term-extraction.md; U1 (§3.4).
