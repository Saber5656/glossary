# 19 — Extractor E1: Japanese domain terms with FLR scoring

## Title

`src/extract/e1-ja-domain.ts`: compound-noun candidates + heuristics + FLR score

## Summary

Implement extractor E1 per DESIGN.md §9.4-E1 and the FLR method from
docs/research/japanese-term-extraction.md §4, over doc prose blocks and JA
code comments.

## Context

This is the highest-value extractor (R1 primary target: domain/business
terms). Precision defaults here are known unknown U2 — thresholds are config
knobs so the owner's validation can tune without code change.

## Scope

- E1 module + built-in JA stopword list + tests. Input: `TextBlock[]` (15) +
  comments (16) + `JaTokenizer` (17).

## Detailed Requirements

1. Signature:
   `extractJaDomain(inputs: {blocks: PositionedText[]}, tokenizer: JaTokenizer | null, cfg: {minScore, minOccurrences, stopwords: Set<string>}): RawCandidate[]`
   where `PositionedText = {text, path, line}` and
   `RawCandidate = {surface, kind: 'domain', score, source: {path, line}, snippet}`
   (one RawCandidate PER OCCURRENCE; merge/thresholding happens in 23 — but E1
   applies its own `minScore` on the term level before emitting, see 5).
2. Candidate generation (tokenizer path):
   - For each prose block (skip `code-in-doc`), tokenize; build noun runs via
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
- [ ] On repo-ja-mixed corpus: 支払予約, 与信枠, オーソリゼーション, 締め処理, 売上確定 all emitted; システム-like stopwords absent; `無視用語` absent (never scanned); code-in-doc content absent.
- [ ] Tokenizer-null mode: katakana + kanji-run + quoted heuristics still yield 支払予約 and オーソリゼーション (scores from heuristic path).
- [ ] All regexes documented linear-time; 2000-char-line inputs safe (upstream capped, re-assert here on a synthetic input).
- [ ] Determinism double-run test.

## Validation

Unit tests + a printed top-20 candidate table for the fixture corpus attached
to the PR (human sanity check of ranking).

## Dependencies

08, 15, 17; fixtures 13.

## Non-goals

Sub-span candidate emission, synonym clustering (v2/LLM), English domain
phrases (E2 covers identifier-derived ones).

## Design References

DESIGN.md §9.4-E1, §9.5; research/japanese-term-extraction.md §4;
U2 (§3.4); §13-B2' (regex discipline).
