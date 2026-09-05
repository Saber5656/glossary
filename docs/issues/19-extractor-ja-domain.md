# 19 — Extractor E1: Japanese domain terms with FLR scoring

## Title

`src/extract/e1-ja-domain.ts`: compound-noun candidates + heuristics + FLR score

## Summary

Implement extractor E1 per DESIGN.md §9.4-E1 and the FLR method from
docs/research/japanese-term-extraction.md §4, over doc text blocks (prose and
headings). Code files contribute nothing to E1 in v1 — comments are E3 input
(DESIGN §9.4).

## Context

This is the highest-value extractor (R1 primary target: domain/business
terms). Precision defaults here are known unknown U2 — thresholds are config
knobs so the owner's validation can tune without code change.

## Scope

- E1 module + built-in JA stopword list + tests. Input: `TextBlock[]` (15;
  carries `path`) + `JaTokenizer` (17).

## Detailed Requirements

1. Signature:
   `extractJaDomain(inputs: {blocks: TextBlock[]}, tokenizer: JaTokenizer | null, cfg: {minScore: number, stopwords: Set<string>}): RawCandidate[]`
   — `TextBlock` is issue 15's exported type (carries `path`). Process
   `kind: 'prose' | 'heading'` blocks; SKIP `code-in-doc`.
   `RawCandidate = {surface, kind: 'domain', score, source: {path, line}, snippet}`
   with `line` = block.startLine + newline offset of the occurrence within
   the block. One RawCandidate PER OCCURRENCE; global minOccurrences and all
   cross-extractor thresholds are issue 23's job — E1 applies ONLY its own
   `minScore` at the term level before emitting (step 5).
2. Candidate generation (tokenizer path):
   - For each prose/heading block, tokenize; build noun runs via
     `nounRuns` (17).
   - From each run of length L, emit the FULL run surface only (maximal
     match; sub-spans are counted for FLR connectivity but not emitted as
     candidates in v1 — keeps noise down).
   - Skip candidates: length < 2 chars; pure ASCII (E2's job); contains only
     hiragana; matches stopwords (after termKey normalization); posDetail 数
     runs already split (17).
3. Heuristic generation (always, tokenizer or not):
   - Katakana runs `[\p{Script=Katakana}ー]{3,}`.
   - Kanji runs `\p{Script=Han}{2,}` (only when tokenizer is null — otherwise
     noun runs cover them).
   - Quoted phrases `「([^」\n]{2,30})」`.
   - Occurrence dedup (normative): before counting and emission, collapse
     duplicates by `(termKey(surface), path, line)` — a surface found at the
     same location by both the tokenizer path and a heuristic counts ONCE.
   - Line cap (B2', defense in depth): truncate every line to 2000 chars
     before ANY E1 regex, regardless of upstream capping.
4. FLR scoring (research doc formula), computed over the whole corpus of this
   run:
   - Token unit = tokenizer tokens of each candidate (heuristic candidates:
     treat the whole surface as one token, L=1).
   - `FL(n)`/`FR(n)` = distinct left/right neighbor tokens within noun runs.
   - `score(term) = f(term) * (Π (FL(ni)+1)(FR(ni)+1))^(1/(2L))`, where
     `f` = occurrence count of the exact surface (after normalizeSurface).
   - Round to 2 decimals half-up. Worked example (MUST be a unit test):
     corpus `支払予約 支払予約 支払方法 予約確認` →
     f(支払予約)=2, 支払: FL=0,FR=2(予約,方法), 予約: FL=1(支払),FR=1(確認)
     ⇒ score = 2 × ((1·3)(2·2))^(1/4) = 2 × 12^0.25 ≈ 3.72.
5. Emission: for each distinct term with `score ≥ cfg.minScore` AND
   occurrences ≥ 1 (global minOccurrences applied in 23), emit one
   RawCandidate per occurrence location (path, line, snippet = the containing
   sentence/line trimmed ≤ 200 chars).
6. Built-in stopword list `src/extract/stopwords-ja.ts`: ≥ 80 entries of
   generic nouns (システム, データ, 情報, 処理, 場合, 方法, 内容, 結果, 対応,
   確認, 設定, 環境, 機能, 画面, 一覧, 管理, 状態, 利用, 使用, 実行, 作成,
   更新, 削除, 追加, 取得, 登録, ファイル, ディレクトリ, ユーザー, サーバー,
   エラー, ログ, テスト, コード, 実装, 開発, 仕様, 設計, 課題, 問題, …).
   User stopwords file (config.extract.stopwordsPath) merges in (loaded by 24,
   passed as the Set).
7. Determinism: iteration over Maps in insertion order of sorted inputs;
   output sorted by (surface, path, line).

## Acceptance Criteria

- [ ] Worked FLR example passes with the exact value 3.72.
- [ ] On repo-ja-mixed corpus: 支払予約, 与信枠, オーソリゼーション, 締め処理, 売上確定 all emitted; the stopword surfaces システム, データ, 情報, 処理, 確認 are absent (after termKey normalization); `無視用語` absent (never scanned); code-in-doc content absent.
- [ ] Tokenizer-null mode: katakana + kanji-run + quoted heuristics still yield 支払予約 and オーソリゼーション (scores from heuristic path).
- [ ] Dedup: a katakana term found by BOTH tokenizer and heuristic at the same (path, line) counts one occurrence in FLR's f and emits one RawCandidate.
- [ ] Adversarial: a synthetic 5000-char line is capped to 2000 before regexes and the full pattern set completes < 100 ms on worst-case inputs (repeated 「, repeated katakana).
- [ ] Determinism double-run test.

## Validation

Unit tests + a committed vitest snapshot of the fixture corpus top-20
(surface, score, occurrences, first source), sorted by (score desc, surface
asc via compareCodepoint) — the deterministic ranking artifact.

## Dependencies

08, 15, 17; fixtures 13.

## Non-goals

Sub-span candidate emission, synonym clustering (v2/LLM), English domain
phrases (E2 covers identifier-derived ones).

## Design References

DESIGN.md §9.4-E1, §9.5; research/japanese-term-extraction.md §4;
U2 (§3.4); §13-B2' (regex discipline).
